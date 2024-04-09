---
title: 动手实现一个Go的协程池
author: siegelion
date: 2022/03/04 15:15
tags: [Go,后端]
categories: [笔记]
---

## 前言

前几天听学长们回忆面试的经历，其他的内容大多都不记得了，只记得他们提到面试官问了**线程池**的相关概念，线程池对我来说不算熟悉也不算陌生。

按照我的理解，线程池主要应用在多线程服务器中，是为了解决存在大量请求时，为了使每个请求可以被迅速处理，而为每一个请求创建了一个处理线程，但每个处理请求本身所需要消耗的时间是很短暂的，在处理请求的任务完成后，运行时会销毁线程、进行GC等一系列操作，导致在处理请求的任务中，创建线程、销毁线程所需的时间和内存的开销在整个任务中所占的比例过高。并且对创建线程的数量不加限制容易引起系统资源耗尽的风险。此时可以利用池化的思想，也即存在一个线程的池子，池子中的线程数量是固定的，每当处理一个请求需要一个线程时便从池子中获取一个，任务处理完毕后线程空闲，便重新放回池子中，等待重新复用。

除了线程池外，还有一些类似的概念，例如内存池、连接池、协程池等，都是在如下的场景中进行使用的：

- 单个任务处理时间比较短
- 需要处理的任务数量很大

Go语言中原生提供了一种轻量级的线程实现：协程，虽然协程相较于线程来说更加轻量级导致创建与销毁时消耗的资源更少，但是不可避免的在处理任务的数量过大时一样会导致以上提到的诸多问题。因此本文动手实现了一个协程池，本文的实现参考了[100 行写一个 go 的协程池 (任务池)](http://www.qinblog.net/Article/article/19.html)。

## 实现

### 任务定义

协程池目的本身是为了降低处理大量任务时的资源消耗，因此我们首先对任务进行定义，我们将每个任务抽象为一个函数的执行，因此每个任务主要的内容就是执行函数以及函数参数：

```go
type Task struct {
	Params  []interface{}
	Handler func(params ...interface{})
}
```

### 协程池定义

在定义了任务之后，我们对协程池的数据结构进行定义：

```go
type GoroutinesPool struct {
	capacity  uint64 // 容量
	workerNum uint64 // 目前的协程数
	status    uint64 // 状态

	taskChan chan *Task //任务池

	sync.Mutex // 锁
}
```

协程池的状态进行如下定义，此处暂时只定义两种状态：运行和中止：

```go
const (
	RUNNING uint64 = iota
	STOP
)
```

#### 获取协程池容量

协程池的容量在初始化后不会发生变化，因此读取协程池的容量不涉及到并发读取的冲突问题，所以可以直接读取。

```go
func (p *GoroutinesPool) getPoolCapacity() uint64 {
	return p.capacity
}
```

#### 获取已经启动协程数

但启动的协程数会发生变化，因此需要考虑

```go
func (p *GoroutinesPool) getPoolWorkerNum() uint64 {
	return atomic.LoadUint64(&p.workerNum)
}
```

#### 修改已经启动协程数

由于在Go语言中就连最简单的赋值操作实际上都不是原子操作，在底层实现时是先向内存的低32位进行赋值，然后再向高32位赋值。因此在Go语言的代码中，就算是简单的读取和赋值，在高并发的情况下也无法保证数据的一致性。因此我们采用`atomic`包中的函数来保证我们的操作是具备原子性的，避免在改变变量值的 同时有另外一个协程读取了这个还没赋值完成的变量。

##### 增加

```go
func (p *GoroutinesPool) addNewWorker() {
	atomic.AddUint64(&p.workerNum, 1)
}
```

##### 减少

```go
func (p *GoroutinesPool) decWorker() {
	atomic.AddUint64(&p.workerNum, ^uint64(0))
}
```

#### 获取协程池状态

读取协程池的状态时同上，使用`atomic`保证原子性。

```go
func (p *GoroutinesPool) getPoolStatus() uint64 {
	return atomic.LoadUint64(&p.status)
}
```

#### 设置协程池状态

设置协程池的状态时，由于涉及的逻辑判断较多，无法使用单条语句保证原子性，因此此处我们使用锁的方式来保证数据一致性。

```go
func (p *GoroutinesPool) setPoolStatus(status uint64) bool {
	defer p.Unlock()
	p.Lock()
	if p.status == status {
		return false
	} else {
		p.status = status
		return true
	}
}
```

#### 启动一个新的任务

启动一个新的任务需要先获取协程池的锁，获取到后判断协程池的状态，若没有处于关闭状态，并且容量充足，则启动一个新的协程并向任务队列中写入新的任务。

```go
func (p *GoroutinesPool) NewTask(t *Task) error {
	defer p.Unlock()
	p.Lock()

	if p.getPoolStatus() == RUNNING {
		if p.getPoolCapacity() > p.getPoolWorkerNum() {
			p.newTaskGoroutine()
		}
		p.taskChan <- t
		return nil
	} else {
		return errors.New("goroutines pool has already closed")
	}
}
```

##### 启动新的协程

启动新的协程时，需要先将协程池中启动的协程数增加，然后启动一个协程，协程中主要由一个循环监听任务队列的循环组成，一旦监听到任务队列中存在未处理的任务，便取出任务进行执行。

由于执行任务的协程若在执行任务的过程中`panic`，会导致整个进程的崩溃，因此我们需要在每个协程执行时加入`recover`进行兜底。

```go
func (p *GoroutinesPool) newTaskGoroutine() {
	p.addNewWorker()

	go func() {
		defer func() {
			p.decWorker()
			if r := recover(); r != nil {
				log.Printf("worker %s has panic\n", r)
			}
		}()

		for {
			select {
			case task, ok := <-p.taskChan:
				if !ok {
					return
				}
				task.Handler(task.Params...)
			}
		}
	}()
}
```

#### 关闭协程池

```go
func (p *GoroutinesPool) ClosePool() {
	p.setPoolStatus(STOP)
	for len(p.taskChan) > 0 {
		time.Sleep(time.Second * 60)
	}
	close(p.taskChan)
}
```


---
title: 浅析 Go 语言中的锁
author: siegelion
tags: [Go]
categories: [笔记]
date: 2023-03-14 17:24
---

# 锁

## 互斥锁与自旋锁

- 互斥锁：互斥锁又被称为无忙等待锁，当获取不到锁的时候，线程会被阻塞，等待锁空闲后被唤醒，线程被阻塞是由操作系统实现的，这个过程涉及到由用户态陷入内核态，涉及线程的切换。
- 自旋锁：自旋锁又被称为忙等待锁，获取不到锁的时候，一直占用 CPU 进行自旋检测锁的状态。

> 自旋锁和互斥锁的选择也是一个值得考虑的问题。
> 如果一个资源被占用一次的时间过短，那么自旋等待的代价要低于加锁后进行线程切换的代价，这时最好选用自旋锁。反之，使用互斥锁让线程阻塞并让出 CPU 是更好的选择。

## 读写锁

对于一些操作由于他们不会修改数据，因此这种对于这类操作可以并行执行，但同时我们也要保证修改数据的操作不能与其他操作并行执行，这就需要更加细粒度的锁，进入引出了读写锁。
读写锁还可以继续细分为：读优先锁和写优先锁，这二者都无法做到兼顾到写者和读者不会出现饥饿的情况，因此更好的办法是使用一个公平读写锁，按照 FIFO 的方式处理读写操作。

## 悲观锁与乐观锁

**悲观锁与乐观锁**

- 悲观锁：假定发生冲突的概率会很高，因此读写数据前需要加锁，保证读写数据期间，数据不会被其他人修改，上文提到的互斥锁、自旋锁、读写锁都属于悲观锁。
- 乐观锁：假定发生冲突的概率较低，先修改完共享资源，再验证这段时间内有没有发生冲突，如果没有其他线程在修改资源则操作成功，如果发现有其他线程已经修改过这个资源，就放弃本次操作。

> 对于悲观锁和乐观锁的选择问题，只有在冲突概率非常低，且加锁成本非常高的场景时，才考虑使用乐观锁。

# 基本原语


Go 语言中提供了两种类型的锁：互斥锁 `Mutex` 与读写锁 `RWMutex`

## Mutex

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230730112203.png)


### 初版

Mutex 初版的实现比较简单，通过设置一个 key 字段上。key 的含义比较简单，就是一个标志位，等于 0 表示锁未被持有，1 表示被某个 goroutine 持有，等于 n 表示还有 n-1 个等待者。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230730113300.png)


#### 加锁

- 加锁（Lock）的过程首先是给 key 加 1：
	- 如果 key 返回 1，则表示当前 goroutine 占有了这把锁，其它 goroutine 只能做候选者。
	- 如果 key 返回 n（n > 1），这说明当前有其它 gorutine 正在占用这把锁，所以接下来需要通过信号量机制将当前 goroutine 挂起，加到等待队列，进入阻塞状态。

#### 解锁

- 解锁（Unlock）的过程是给 key 减 1：
	- 如果 key 返回 0，表示当前没有其它 goroutine 在等待，可以直接返回。
	- 如果 key 返回 n (n > 0)，说明还有其它 goroutine 在等待，因此需要通过信号量机制将等待队列中的其它 goroutine 唤醒。

#### 问题

初版 Mutex 在实现的时候，有两个问题：
- Unlock 调用无限制。
- goroutine 唤醒机制性能低下。

> Mutex ==本身并没有包含当前 goroutine 的任何信息==，因此 Unlock 方法能被任意的 goroutine 调用。这样会导致一个问题，如果某个 goroutine 不按套路来，随便调用 Unlock 函数，让标志位 key 清零，那么数据竞争的问题还是会出现。Mutex 的这个特性一直保留至今。因此使用 Mutex 的时候，一定要遵循 “==谁加锁，谁解锁==” 的原则。

### 第二版

这一版 Mutex 的核心特点是一个 goroutinue 被唤醒后，不是立即执行任务，而是仍然和其他 goroutine 重复一遍和抢占锁的流程，这样新来的 goroutine 就有机会获取到锁，这就是所谓的**给新人机会**。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230730113311.png)


#### 问题

虽然这一版改进了第一版中只能按照 FIFO 方式进行抢锁，导致的后续的新来的锁等待时间过长的问题。但引入了新的问题：
- 在加锁解锁的过程中，涉及到 goroutine 阻塞和唤醒的过程，系统调用涉及到不小的系统开销。如果 Lock 和 Unlock 之间的代码耗时很短，那么让新来的 goroutine 或者是醒着的 goroutine 抢占锁失败后，不立即睡眠，而是再尝试几次，说不定就能拿到锁了，尝试一定的次数之后，再进行原有的逻辑。

### 第三版

这一版和第二版区别不大，改动是如果被唤醒的协程或者是新来的协程没有抢到锁，就会通过自旋的方式尝试检查锁是否被释放。尝试了一定的次数后，会再继续原有的逻辑。这里的自旋是指循环尝试。

#### 问题

第三版在性能优化的道路上又前进了一步，几乎走到头了。但是有一种极端情况，新来的协程每次都能抢占到锁，那么等待中的协程就会一直处于等待之中，这就是所谓饥饿问题。

### 第四版

Go 语言的 `sync.Mutex` 由两个字段 `state` 和 `sema` 组成。其中 `state` 表示当前互斥锁的状态，而 `sema` 是用于控制锁状态的信号量。

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

#### state

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230221205514.png)

在默认情况下，互斥锁的所有状态位都是 0，`int32` 中的不同位分别表示了不同的状态：

- `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；
- `mutexStarving` — 当前的互斥锁进入饥饿状态；
- `mutexWoken` — 表示从正常模式被从唤醒；
- `mutexLocked` — 表示互斥锁的锁定状态；

#### 正常模式与饥饿模式

**正常模式**

锁的等待者会按照先进先出的顺序获取锁。但是刚被唤起的 Goroutine 与新创建的 Goroutine 竞争时，大概率会获取不到锁，为了减少这种情况的出现，一旦 Goroutine 超过 1 ms 没有获取到锁，它就会将当前互斥锁切换饥饿模式。

> 因为锁被解锁的时候队头的 Goroutine 可能被阻塞了，这时候如果有一个正在获取锁的 Goroutine，为了提高吞吐量自然会交给这个正在运行且在索要锁的 Goroutine。

**饥饿模式**

互斥锁会直接交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1 ms，那么当前的互斥锁就会切换回正常模式。

> 与正常模式不同的是，饥饿模式下锁会一直持有直到第一个等待者准备好获取锁。

**评价**

与饥饿模式相比，正常模式下的互斥锁能够提供 **更好的性能**，饥饿模式的能 **避免** Goroutine 由于陷入等待无法获取锁而造成的 **高尾延时**。

#### 加锁与解锁

> CAS ： `atomic.CompareAndSwapInt32(addr, old, new) bool` 方法，这个方法会先比较传入的地址的值是否是 old，如果是的话就尝试赋新值，如果不是的话就直接返回 `false`，保证数据只会被修改一次。

我们在这一节中将分别介绍互斥锁的加锁和解锁过程，它们分别使用 `sync.Mutex.Lock` 和 `sync.Mutex.Unlock` 方法。

##### 加锁

互斥锁的加锁是靠 `sync.Mutex.Lock` 完成的：

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}
```

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230314172939.png)

**快速加锁**

利用 CAS 函数，判断锁的状态是否处于初始状态，如果是则直接对锁上锁。

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
}
```

**慢速加锁**

如果互斥锁的状态不是 0（初始状态） 时就会调用 `sync.Mutex.lockSlow` 尝试通过自旋（Spinnig）等方式等待锁的释放，该方法的主体是一个非常大 for 循环，这里将它分成几个部分介绍获取锁的过程：
1. 判断当前 Goroutine 能否进入自旋。
2. 通过自旋等待互斥锁的释放：一旦当前 Goroutine 能够进入自旋就会执行 30 次的 `PAUSE` 指令，该指令只会占用 CPU 并消耗 CPU 时间。处理了自旋相关的特殊逻辑之后，互斥锁会根据上下文计算当前互斥锁最新的状态。如果最终没有通过获得锁，不断尝试获取锁并陷入休眠等待信号量的释放，一旦当前 Goroutine 可以获取信号量，它就会立刻返回。

**如何判断当前 Goroutine 能否进入自旋？**

1. 互斥锁只有在正常模式才能进入自旋；
2. `runtime.sync_runtime_canSpin` 需要返回 `true`：
   1. 运行在多 CPU 的机器上；
   2. 当前 Goroutine 为了获取该锁进入自旋的次数小于四次；
   3. 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

##### 解锁

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230314173046.png)

互斥锁的解锁过程相对简单，该过程会先使用 `sync/atomic.AddInt32` 函数快速解锁。
1. 如果该函数返回的新状态等于 0（回到了初始状态），当前 Goroutine 就成功解锁了互斥锁。
2. 如果该函数返回的新状态不等于 0，这段代码会调用 `sync.Mutex.unlockSlow` 开始慢速解锁。
	1. 会先校验锁状态的合法性：
		1. 如果当前互斥锁已经被解锁过了会直接抛出异常 `sync: unlock of unlocked mutex` 中止当前程序。
	2. 在正常模式下，上述代码会使用如下所示的处理过程：
		1. 如果互斥锁不存在等待者或者互斥锁的 `mutexLocked`、`mutexStarving`、`mutexWoken` 状态不都为 0，那么当前方法可以直接返回，不需要唤醒其他等待者；
		2. 如果互斥锁存在等待者，唤醒等待者，等待者按照正常模式的抢锁方法竞争锁。
	3. 在饥饿模式下，将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，在这时互斥锁还不会退出饥饿状态，通过判断条件确认是否退出饥饿模式。

## RWMutex

读写互斥锁 `sync. RWMutex` 是细粒度的互斥锁，它不限制资源的并发读，但是读写、写写操作无法并行执行。

`sync. RWMutex` 中总共包含以下 5 个字段：

```go
type RWMutex struct {
	w           Mutex
	writerSem   uint 32
	readerSem   uint 32
	readerCount int 32
	readerWait  int 32
}
```

- `w`： 复用互斥锁提供的能力；
- `writerSem` 和 `readerSem` ： 信号量，分别用于写等待读和读等待写：
- `readerCount`： 存储了当前正在执行的读操作数量；
- `readerWait` ：表示当写操作被阻塞时等待的读操作个数；

### 写锁

#### 加锁

```go
func (rw *RWMutex) Lock () {
    // 首先解决其他 writer 竞争问题
    rw.w.Lock ()
    // 反转 readerCount，告诉 reader 有 writer 竞争锁
    r := atomic.AddInt32 (&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有 reader 持有锁，那么需要等待
    if r != 0 && atomic.AddInt32 (&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex (&rw.writerSem, false, 0)
    }
}
```

首先调用互斥锁的 lock，获取到互斥锁之后：
1.  `atomic.AddInt32(&rw. readerCount, -rwmutexMaxReaders)`  调用这个函数阻塞后续的读操作。
2. 如果计算之后当前仍然有其他 Goroutine 持有读锁，那么就调用 `runtime_SemacquireMutex`  休眠当前的 Goroutine 等待所有的读操作完成。

#### 解锁

```go
func (rw *RWMutex) Unlock () {
    // 告诉 reader 没有活跃的 writer 了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    
    // 唤醒阻塞的 reader 们
    for i := 0; i < int (r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放内部的互斥锁
    rw.w.Unlock ()
}
```

解锁的操作，会先调用 `atomic.AddInt32(&rw. readerCount, rwmutexMaxReaders)`  将恢复之前写入的负数，然后根据当前有多少个读操作在等待，循环唤醒

### 读锁

#### 加锁

```go
func (rw *RWMutex) RLock () {
    if atomic.AddInt32(&rw. readerCount, 1) < 0 {
        // rw.readerCount 是负值的时候，意味着此时有 writer 等待请求锁，因为 writer 优先级高，所以把后来的 reader 阻塞休眠
        runtime_SemacquireMutex (&rw. readerSem, false, 0)
    }
}
```

首先是读锁， `atomic.AddInt32(&rw.readerCount, 1)`  调用这个原子方法，对当前在读的数量加一，如果返回负数，那么说明当前有其他写锁，这时候就调用 `runtime_SemacquireMutex`  休眠 Goroutine 等待被唤醒。

#### 解锁

```go
func (rw *RWMutex) RUnlock () {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow (r) // 有等待的 writer
    }
}
func (rw *RWMutex) rUnlockSlow (r int 32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个 reader 了，writer 终于有机会获得锁了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

解锁的时候对正在读的操作减一，如果返回值小于 0 那么说明当前有在写的操作，这个时候调用 `rUnlockSlow`  进入慢速通道。慢速解锁时，会先对 `readerWait` 减一，当该值为 0 说明没有在读的操作了，可以唤醒等待的写 Goroutine。

### 总结

Go 中的读写锁，写写冲突是通过封装在读写锁里的互斥锁解决的。读写冲突，是通过 `readerCount` 和 `readerWait` 这两个变量解决的。
- 写者获取锁：
	1. 写者获取到互斥锁后会将 `readerCount` 加负数最大值，并将原 `readerCount` 记录在 `readerWait` 中。
	2. 读者想要获取锁时，对 `readerCount` 加 1，发现为负则阻塞。
	3. 写者退出时，对 `readerCount` 取反，依次唤醒读者。
- 读者获取锁：
	1. 读者会对 `readerCount` 加 1。
	2. 写者这个时候会将 `readerCount` 加负数最大值，然后再取反根据取反值是否为 0，判断是否要阻塞。
	3. 但这个时候如果再有读者进来，对 `readerCount` 加 1，肯定发现为负则阻塞，因此即使现在是前面的读者持有锁，后面的读者也无法运行。
	4. 因此写者的优先级会高于读者，读者退出之后会优先唤醒写者，而后释放互斥锁，因此先被唤醒的写者会阻塞在他们之前的读者。

#### 例子

- 初始：`readerCount`：0，`readerWait`：0。
- 读者 1：当有一个读者便会将 `readerCount` 变量加一，`readerCount`：1。
- 写者 1：如果这时候出现了一个写者，便会将 `readerCount` 的值，保存在 `readerWait` 中，然后将 `readerCount` 赋一个最大读者数量的负值，`readerCount`：-10000，`readerWait`：1
- 写者 2：这时候如果再出现一个写者，由于先前的写者 1 获取了互斥锁，因此写者 2 会直接阻塞。
- 读者 2：这样后续的再出现新的读者，后续的读者 `readerCount` 变量加一后这个变量也是负的，读者就知道存在一个写者，读者便会阻塞自己，`readerCount`：-9999，`readerWait`：1
- 读者 1 完成：读者 1 完成自己的操作之后，将 `readerCount` 减一，但发现这时候的值是负的，就会知道存在等待的写者，进入慢速解锁模式，将 `readerWait` 减一，减一后如果为 0 则唤醒写者 1。


# 参考

- [Golang Mutex 的实现原理]( https://nxw.name/2021/golang-mutexde-shi-xian-yuan-li-1ef30cc7 )
- [Go 中的 Mutex 设计原理详解（三）](https://zhuanlan.zhihu.com/p/342706674)
- [Go 中的 Mutex 设计原理详解（四）](https://zhuanlan.zhihu.com/p/344977623)

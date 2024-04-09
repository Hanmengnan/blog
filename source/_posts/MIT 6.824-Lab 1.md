---
title: MIT 6.824-Lab1
author: siegelion
date: 2021/9/24 9:15
tags: [Go,分布式]
categories: [笔记]
---

# 前言

早在保研时，对分布式感兴趣的我想要涉猎一些这个领域的知识，于是在网络上搜寻相关的学习资料，就发现了很多前辈都推荐学习者阅读一下**谷歌的三大论文**——**MapReduce**、**GFS**、**BigTable**，此外还推荐了一个课程——[MIT 6.824](http://nil.csail.mit.edu/6.824/2020/schedule.html)，这是一个关于分布式系统的课程，授课的教师是大名鼎鼎的蠕虫病毒的发明者——[Robert Morris](https://zh.wikipedia.org/wiki/%E7%BD%97%E4%BC%AF%E7%89%B9%C2%B7%E6%B3%B0%E6%BD%98%C2%B7%E8%8E%AB%E9%87%8C%E6%96%AF)，听这样的大牛讲课本身就是一种享受，并且在听了两节课之后，觉得老师讲的确实透彻且深刻，并且在如今2020年，老师上课时大部分的讲解都是使用板书完成的，这点显得尤为难得，以上这些让我这听惯了国内PPT课堂的学生耳目一新。

![](https://i.loli.net/2021/09/23/SaHTu8Y9Be3Eg2t.png)

# 实验

在听了两节课之后，我开始动手进行本课的第一个实验——实现一个简单的**MapReduce**系统。由于我事先阅读过MapReduce论文，加之第一个实验较为简单，因此我并没有倒在**lab1**面前，但前前后后还是花了中秋节的三天时间才搞定这个实验，并通过了全部8个测试点。

简单描述一下该实验的要求：

- 实验中要求实现的系统包含两部分：`worker`和`coordinator`，`worker`负责完成任务，`coordinator`负责将任务分配给`worker`。会存在多个`worker`同时工作来模拟并行，但`coordinator`只存在一个。通常来说分布式系统中的每个`worker`应该存在于多台主机之上，并通过网络`RPC`进行通信，但由于是实验，因此没有多机进行模拟，所以将`worker`们都运行在一台主机上，依然通过`RPC`通信，但使用的是`UNIX SOCK`的方式。
- 系统的输入是一系列文件，仿照`Map-Reduce`论文中的思路，进行单词统计等一系列操作。实验需要通过的功能点主要有以下8个测试点：
    - word count（单词计数）
    - indexer（单词来自文件统计）
    - map parallelism （map 任务并行测试）
    - reduce parallelism（reduce 任务并行测试）
    - job count （任务计数）
    - early exit （是否有`worker`在全部任务完成前退出）
    - crash （`worker`崩溃后任务可以正常完成）
- [详细要求](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)见网页。

## 文件目录

- `main`：主文件夹

    - `mrcoordinator.go`：coordinator 的启动文件

    - `mrworker.go`：worker的启动文件

    - `mrsequential.go`：实验串行模拟版本

        > 注：以上文件需要在不同的终端启动

    - `test-mr.sh`：测试实验是否通过的`shell`脚本

- `mr`：worker 和 coordinator 具体实现的文件夹

    - `coordinator.go`
    - `worker.go`
    - `rpc.go`：worker 和 coordinator 通信的实现

- `mrapps`：每对Map-Reduce操作的具体实现文件夹

    - ......

## 尝试

### version 1

一开始我抱着有些畏惧的心态去完成这个实验，目标是实现**单词计数**的功能，由于整个系统的输入是一系列文件，所以我简单地使`worker`向`coordinator`获取文件的列表，然后为每一文件单独开启一个协程用于`map`任务，每个协程将其结果写入中间文件，由于系统规定了`reduce`任务的数量，因此在`map`任务结束后，启用规定数量的协程去执行`reduce`任务即可。

使用这个方式，通过了单词计数的测试没有问题，但是在`indexer`的测试却行不通了，由于系统是通过`test-mr.sh`进行测试的，因此我开始浏览`shell`脚本的内容。发现了在测试脚本中，会启动多个`worker`来完成`map`任务，也就是说是多个`worker`共同完成任务，这与我一开始认为的一个`worker`通过开启多个协程完成任务的想法相悖，所以代码需要进行修改。

(1个`worker`同时启动`N`个任务)

### version 2

从上文，我明白了应该是多个进程共同完成任务。因此将实现思路改为，每个`worker`通过`RPC`向`coordinator`获取任务（文件），获取到后开启协程去完成每个任务。 通过这个改动，我通过了`indexer`测试。但是在`parallelism`测试中却`Fail`了。这意味着程序的实现还是错误的。

(`m`个`worker`同时启动`N/m`个任务)

### version 3

接着我分析了`parallelism`测试中的测试原则，测试原则为`worker`在`map`阶段通过创建一个文件名为`mr-worker`+`PID`的文件，在`map`结束后删除该文件，通过统计存在几个特定文件名的数量，来判断同时运行的`worker`数量。由于在`version 2`版本中，一个`worker`启动了多个协程来执行`map`操作，每个协程在执行时会同时操作一个文件，所以会出现重复删除一个文件的情况，导致程序崩溃。

因此，需要将一个`worker`启动多个协程的方式改掉，改为每个`worker`每次只执行一个`map`任务。这次可以通过测试了。

### version 4

最后难住我的测试点是崩溃测试，在`worker`崩溃后系统能否依然正确执行，得出正确的结果，一个完备的系统设计无疑是应该具备容错的能力的。

`worker`执行前会向`coordinator`获取相应的任务，获取任务后，若在执行过程中发生崩溃，那意味着该任务没有被成功执行，需要`coordinator`检测出失败的任务，然后将任务重新分配给`worker`。

这里我采用了在`coordinator`中引入协程同时记录任务执行状态的记录的方式，在`coordinator`将任务分配给`worker`后，启动协程进行倒计时，若在规定的时间内任务并没有执行完成，那么即认为任务失败，`coordinator`会重新分配任务。



## 逻辑

### worker

```go
package mr

import (
	"encoding/json"
	"fmt"
	"hash/fnv"
	"io/ioutil"
	"log"
	"net/rpc"
	"os"
	"sort"
	"time"
)

//
// Map functions return a slice of KeyValue.
//
type KeyValue struct {
	Key   string
	Value string
}

type ByKey []KeyValue

func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }

//
// use ihash(key) % NReduce to choose the reduce
// task number for each KeyValue emitted by Map.
//
func ihash(key string) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32() & 0x7fffffff)
}

//
// main/mrworker.go calls this function.
//
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {

	// map 阶段
	mapJobNum := getMapNum()
	// map任务数，同时也是文件数
	for {
		mapFile, mapJobID := getMapJob()
		// map任务所对应的文件名以及任务ID
		if mapFile != "" {
			// 代表还有任务待分配
			file, err := os.Open(mapFile)
			if err != nil {
				log.Fatalf("cannot open %v", mapFile)
			}
			content, err := ioutil.ReadAll(file)
			// 读取文件内容
			if err != nil {
				log.Fatalf("cannot read %v", mapFile)
			}
			file.Close()
			kva := mapf(mapFile, string(content))
			// map操作
			shuffle(mapJobID, kva)
			// 将map任务的结果写入临时文件
			getMapJobDone("single", mapFile)
			// 通知调度器一个map任务已完成

		} else {
			if getMapJobDone("all", "") {
				//判断是否全部map任务已结束
				break
			}
			time.Sleep(time.Second)
		}
	}

	// reduce 阶段
	for {
		reduceJobID := getReduceJob()
		// 获取reduce任务ID
		if reduceJobID != -1 {
			kva := make([]KeyValue, 0)
			for j := 0; j < mapJobNum; j++ {
				fileName := fmt.Sprintf("mr-%d-%d", j, reduceJobID)
				// 该reduce任务对应的中间文件名
				file, err := os.Open(fileName)
				if err != nil {
					log.Fatalf("cannot open %v", fileName)
				}
				dec := json.NewDecoder(file)
				// 读取文件内容
				for {
					var kv KeyValue
					if err := dec.Decode(&kv); err != nil {
						break
					}
					kva = append(kva, kv)
					// 保存结果
				}
			}
			storeReduceRes(reduceJobID, kva, reducef)
			// reduce 操作
			getReduceJobDone(reduceJobID)
			// 通知调度器一个reduce任务已完成
		} else {
			if getMROver() {
				// 判断是否全部任务已完成
				break
			}
		}
	}
}

//
//获取map任务数量
//
func getMapNum() int {
	args := MapNumArgs{}
	reply := MapNumReply{}
	call("Coordinator.MapJobNum", &args, &reply)
	return reply.Num

}

//
// 获取新的map任务
//
func getMapJob() (string, int) {
	args := MapJobArgs{}
	reply := MapJobReply{}
	call("Coordinator.MapJob", &args, &reply)
	return reply.File, reply.MapJobID
}

//
// 通知 coordinator map任务已经完成
//
// content:
// `single`: 通知单个任务已经完成 \ `all`: 询问全部任务是否已经完成
//
func getMapJobDone(content, fileName string) bool {

	args := MapJobDoneArgs{Content: content, FileName: fileName}
	reply := MapJobDoneReply{}
	call("Coordinator.MapJobDone", &args, &reply)
	return reply.Done
}

// 
// 将map的结果保存至中间文件
//
func shuffle(mapJobID int, intermediate []KeyValue) {
	interFiles := make([]*os.File, 10)
	for i := 0; i < 10; i++ {
		interFileName := fmt.Sprintf("mr-%d-%d", mapJobID, i)
		interFiles[i], _ = os.Create(interFileName)
		defer interFiles[i].Close()
	}
	for _, kv := range intermediate {
		reduceJobID := ihash(kv.Key) % 10
		// 确定对应的reduce任务
		enc := json.NewEncoder(interFiles[reduceJobID])
		enc.Encode(&kv)
	}
}

// 
// 获取reduce
//
func getReduceJob() int {
	args := ReduceArgs{}
	reply := ReduceReply{}
	call("Coordinator.ReduceJob", &args, &reply)
	return reply.ReduceJobID
}

//
// 将reduce任务执行结果保存在最终的文件中
//
func storeReduceRes(reduceJobID int, intermediate []KeyValue, reducef func(string, []string) string) {
	fileName := fmt.Sprintf("mr-out-%d", reduceJobID)
	ofile, _ := os.Create(fileName)
	defer ofile.Close()
	sort.Sort(ByKey(intermediate))
	i := 0
	for i < len(intermediate) {
		j := i + 1
		for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {
			j++
		}
		values := []string{}
		for k := i; k < j; k++ {
			values = append(values, intermediate[k].Value)
		}
		output := reducef(intermediate[i].Key, values)
		// reduce 操作

		fmt.Fprintf(ofile, "%v %v\n", intermediate[i].Key, output)
		i = j
	}
}

//
// 通知某个reduce任务完成
//
func getReduceJobDone(jobID int) {
	args := ReduceJobDoneArgs{JobID: jobID}
	reply := ReduceJobDoneReply{}
	call("Coordinator.ReduceJobDone", &args, &reply)
}

//
// 询问是否全部map-reduce任务完成
//
func getMROver() bool {
	args := MROverArgs{}
	reply := MROverReply{}
	call("Coordinator.MROver", &args, &reply)
	return reply.Done
}

//
// send an RPC request to the coordinator, wait for the response.
// usually returns true.
// returns false if something goes wrong.
//
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := coordinatorSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer c.Close()

	err = c.Call(rpcname, args, reply)

	return err == nil
}

```

### coordinator

```go
package mr

import (
	"log"
	"net"
	"net/http"
	"net/rpc"
	"os"
	"sync"
	"time"
)

type Coordinator struct {
	MapJobs            chan string     // map任务管道
	MapJobID           chan int        // map任务id
	MapJobCount        int             // map任务数
	MapJobState        map[string]bool // map任务状态`map`
	MapJobStateLock    sync.Mutex      // map任务状态锁
	ReduceJobs         chan int        // reduce任务管道
	ReduceJobState     map[int]bool    // reduce任务状态map
	ReduceJobStateLock sync.Mutex      // reduce任务状态锁
	Over               bool            // mr任务结束标志
	OverLock           sync.Mutex      // mr任务状态锁
}

func (c *Coordinator) MapJobNum(args *MapNumArgs, reply *MapNumReply) error {
	reply.Num = c.MapJobCount
	return nil
}

func (c *Coordinator) MapJob(args *MapJobArgs, reply *MapJobReply) error {
	select {
	case reply.File = <-c.MapJobs:
		reply.MapJobID = <-c.MapJobID
		go func(jobFile string, jobID int) {
			//开启一个协程等待10s判断任务是否完成
			time.Sleep(time.Second * 10)
			c.MapJobStateLock.Lock()
			if !c.MapJobState[jobFile] {
				c.MapJobs <- jobFile
				c.MapJobID <- jobID
			}
			c.MapJobStateLock.Unlock()
		}(reply.File, reply.MapJobID)
	default:
		reply.MapJobID = 0
		reply.File = ""
	}
	return nil
}

func (c *Coordinator) MapJobDone(args *MapJobDoneArgs, reply *MapJobDoneReply) error {
	if args.Content == "single" {
		// 通知单个任务结束
		c.MapJobStateLock.Lock()
		c.MapJobState[args.FileName] = true
		c.MapJobStateLock.Unlock()
	} else if args.Content == "all" {
		// 查询全部任务是否结束
		reply.Done = true
		c.MapJobStateLock.Lock()
		for _, value := range c.MapJobState {
			// 存在未完成map任务
			if !value {
				reply.Done = false
				break
			}
		}
		c.MapJobStateLock.Unlock()
	}
	return nil
}

func (c *Coordinator) ReduceJob(args *ReduceArgs, reply *ReduceReply) error {
	select {
	case reply.ReduceJobID = <-c.ReduceJobs:
		go func(jobID int) { 
			//同map
			time.Sleep(time.Second * 10)
			c.ReduceJobStateLock.Lock()
			if !c.ReduceJobState[jobID] {
				c.ReduceJobs <- jobID
			}
			c.ReduceJobStateLock.Unlock()
		}(reply.ReduceJobID)
	default:
		reply.ReduceJobID = -1
	}
	return nil
}

func (c *Coordinator) ReduceJobDone(args *ReduceJobDoneArgs, reply *ReduceJobDoneReply) error {
	c.ReduceJobStateLock.Lock()
	c.ReduceJobState[args.JobID] = true
	c.ReduceJobStateLock.Unlock()
	return nil
}

func (c *Coordinator) MROver(args *MROverArgs, reply *MROverReply) error {
	c.OverLock.Lock()
	reply.Done = c.Over
	c.OverLock.Unlock()
	return nil
}

//
// start a thread that listens for RPCs from worker.go
//
func (c *Coordinator) server() {
	rpc.Register(c)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := coordinatorSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go http.Serve(l, nil)
}

//
// main/mrcoordinator.go calls Done() periodically to find out
// if the entire job has finished.
//
func (c *Coordinator) Done() bool {
	flag := true
	c.ReduceJobStateLock.Lock()
	for _, value := range c.ReduceJobState {
		if !value {
			flag = false
		}
	}
	c.ReduceJobStateLock.Unlock()
	c.OverLock.Lock()
	c.Over = flag
	c.OverLock.Unlock()
	return c.Over

}

//
// create a Coordinator.
// main/mrcoordinator.go calls this function.
// nReduce is the number of reduce tasks to use.
//
func MakeCoordinator(files []string, nReduce int) *Coordinator {
	c := Coordinator{}

	c.MapJobs = make(chan string, len(files))
	c.MapJobID = make(chan int, len(files))
	c.MapJobCount = len(files)
	c.MapJobState = make(map[string]bool)
	for index, file := range files {
		c.MapJobs <- file
		c.MapJobID <- index
		c.MapJobState[file] = false
	}
	c.ReduceJobs = make(chan int, nReduce)
	c.ReduceJobState = make(map[int]bool)
	for i := 0; i < nReduce; i++ {
		c.ReduceJobs <- i
		c.ReduceJobState[i] = false
	}
	// 初始化 coordinator

	c.server()
	return &c
}
```

### rpc

```go
package mr

//
// RPC definitions.
//
// remember to capitalize all names.
//

import (
	"os"
	"strconv"
)

//
// example to show how to declare the arguments
// and reply for an RPC.
//

type MapNumArgs struct {
}
type MapNumReply struct {
	Num int
}

type MapJobArgs struct {
	Content string
}

type MapJobReply struct {
	File     string
	MapJobID int
}

type MapJobDoneArgs struct {
	Content  string
	FileName string
}
type MapJobDoneReply struct {
	Done bool
}
type ReduceArgs struct {
	Content string
}

type ReduceReply struct {
	ReduceJobID int
}

type ReduceJobDoneArgs struct {
	JobID int
}
type ReduceJobDoneReply struct {
}

type MROverArgs struct {
}

type MROverReply struct {
	Done bool
}

// Add your RPC definitions here.

// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the coordinator.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func coordinatorSock() string {
	s := "/var/tmp/824-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}

```

## 源码

[lab1 实现源码](https://github.com/Hanmengnan/MIT-6.824/tree/master/lab1)

## 结果

![image-20210922193634059](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210922193634059.png)


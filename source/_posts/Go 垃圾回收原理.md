---
title: 浅析 Go 垃圾回收
author: siegelion
tags: [Go]
categories: [笔记]
date created: 2023-03-20 15:32
date updated: 2023-08-01 15:32
---

# 垃圾回收

# 垃圾回收器种类

GC 实现方式包括：

- 追踪式，分为多种不同类型，例如：
  - 标记清扫：从根对象出发，将确定存活的对象进行标记，并清扫可以回收的对象。
  - 标记 **整理**：为了解决内存碎片问题而提出，在标记过程中，将对象尽可能整理到一块连续的内存上。
  - 增量式：将标记与清扫的过程分批执行，每次执行很小的部分，从而增量的推进垃圾回收，达到近似实时、几乎无停顿的目的。
  - 增量整理：在增量式的基础上，增加对对象的 **整理** 过程。
  - 分代式：将对象根据存活时间的长短进行分类，存活时间小于某个值的为年轻代，存活时间大于某个值的为老年代，永远不会参与回收的对象为永久代。并根据分代假设（如果一个对象存活时间不长则倾向于被回收，如果一个对象已经存活很长时间则倾向于存活更长时间）对对象进行回收。
- 引用计数：根据对象自身的引用计数来回收，当引用计数归零时立即回收。

# Go 的垃圾回收机制

Go 的 GC 目前使用的是无分代（对象没有代际之分）、不整理（回收过程中不对对象进行移动与整理）、并发（与用户代码并发执行）的三色标记清扫算法。原因 [1] 在于：

1. 不整理：Go 使用的是基于 tcmalloc 的现代内存分配算法，基本没有内存碎片问题，对对象进行整理不会带来实质性的性能提升。
2. 不分代：但 Go 的编译器会通过 **逃逸分析** 将大部分新生对象存储在栈上（栈直接被回收），只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。因此在 Go 中性能提升不大，并且 Go 设计团队的关注点并不在这。

## 三色标记法

理解 **三色标记法** 的关键是理解对象的 **三色抽象** 以及 **波面（wavefront）推进** 这两个概念。

从垃圾回收器的视角来看，三色抽象规定了三种不同类型的对象，并用不同的颜色相称：

- 白色对象（可能死亡）：未被回收器访问到的对象。在回收开始阶段，所有对象均为白色，当回收结束后，白色对象均不可达。
- 灰色对象（波面）：已被回收器访问到的对象，但回收器需要对其中的一个或多个指针进行扫描，因为他们可能还指向白色对象。
- 黑色对象（确定存活）：已被回收器访问到的对象，其中所有字段都已被扫描，黑色对象中任何一个指针都不可能直接指向白色对象。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230228123631.png)

## STW

垃圾回收的过程中涉及两种流程：赋值器和染色器，为了防止在染色的同时，某些对象之间的关系发生改变，进而导致垃圾回收发生错误，需要进行 STW。STW 可以是 Stop The World 的缩写，也可以是 Start The World 的缩写。通常意义上指的是从 Stop The World 到 Start The World 这一段时间间隔。垃圾回收过程中为了保证准确性、防止无止境的内存增长等问题而不可避免的需要停止赋值器进一步操作对象图以完成垃圾回收。这一动作发生时这一段时间间隔，即万物静止。

## 并发标记清楚法

传统的垃圾收集算法会在垃圾收集的执行期间暂停应用程序，一旦触发垃圾收集，垃圾收集器会抢占 CPU 的使用权占据大量的计算资源以完成标记和清除工作，然而很多追求实时的应用程序无法接受长时间的 STW。

并发（Concurrent）的垃圾收集不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间，通过开启读写屏障、**利用多核优势与用户程序并行执行**，并发垃圾收集器确实能够减少垃圾收集对应用程序的影响：

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731202430.png)

但并发标记清除中面临的一个根本问题就是如何保证标记与清除过程的正确性。所以需要引入屏障技术保证染色器和赋值器可以正确工作。

## 屏障机制

写屏障是一个在并发垃圾回收器中才会出现的概念，垃圾回收器的正确性体现在：**不应出现对象的丢失，也不应错误的回收还不需要回收的对象。**

可以证明，当以下两个条件同时满足时会破坏垃圾回收器的正确性：

- **条件 1**: 赋值器修改对象图，导致某一黑色对象引用白色对象；
- **条件 2**: 从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏。

只要能够避免其中任何一个条件，则不会出现对象丢失的情况，因为：

- 如果条件 1 被避免，则所有白色对象均被灰色对象引用，没有白色对象会被遗漏；
- 如果条件 2 被避免，即便白色对象的指针被写入到黑色对象中，但从灰色对象出发，总存在一条没有访问过的路径，从而找到到达白色对象的路径，白色对象最终不会被遗漏。

- 强三色不变性：条件 1，条件 2 都不能出现。
- 弱三色不变性：可以允许条件 1 出现。

有两种非常经典的写屏障：Dijkstra 插入屏障和 Yuasa 删除屏障。

#### 插入屏障（Dijkstra）- 灰色赋值器

```go
// 灰色赋值器 Dijkstra 插入屏障  
func DijkstraWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {  
    shade(ptr) //先将新下游对象 ptr 标记为灰色  
    *slot = ptr  
}  
  
//说明：  
添加下游对象(当前下游对象slot, 新下游对象ptr) {     
  //step 1  
  标记灰色(新下游对象ptr)     
    
  //step 2  
  当前下游对象slot = 新下游对象ptr                      
}  
  
//场景：  
A.添加下游对象(nil, B)   //A 之前没有下游， 新添加一个下游对象B， B被标记为灰色  
A.添加下游对象(C, B)     //A 将下游对象C 更换为B，  B被标记为灰色
```

Dijkstra 插入屏障的基本思想是避免满足条件 1 以保证垃圾回收的正确性。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203107.png)


![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203118.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203127.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203150.png)


![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203205.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203217.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203227.png)


![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203244.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203254.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731203303.png)

##### 缺点
- 由于 Dijkstra 插入屏障的“保守”，在一次回收过程中可能会残留一部分对象没有回收成功，只有在下一个回收过程中才会被回收。
- 在标记阶段中，每次进行**指针赋值操作**时，都需要**引入写屏障**，这无疑会增加大量性能开销。
- 为了避免造成性能问题，`Go` 团队在最终实现时，**没有为所有栈上的指针写操作，启用写屏障**，但需要标记终止阶段 STW 时对这些栈进行**重新扫描**。


1. **为何需要在一遍标记清扫后进行 STW 然后扫描一次栈？**
	- 个人理解是：因为，由于没有对栈应用写屏障，导致如果存在栈上对象引用了白色的堆上对象，如果不 STW 重新扫描，会导致堆上正在使用的对象被错误回收。
2. **堆上的对象为什么会存在这个白色对象？**
	- 个人理解是：对象不会凭空产生，这个对象是被其他对象创建的，和其它对象有引用关系，但在染色器尚未覆盖到的情况下，父对象便删除和它的引用，由于目前是纯插入屏障，没有删除屏障，因此会导致这个白色对象的产生。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230801103457.png)



#### 删除屏障 （Yuasa）- 黑色赋值器

```go
// 黑色赋值器 Yuasa 屏障  
func YuasaWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {  
shade(*slot) 先将*slot标记为灰色  
*slot = ptr  
}  
  
//说明：  
添加下游对象(当前下游对象slot， 新下游对象ptr) {  
//step 1  
if (当前下游对象slot是灰色 || 当前下游对象slot是白色) {  
标记灰色(当前下游对象slot) //slot为被删除对象， 标记为灰色  
}  
//step 2  
当前下游对象slot = 新下游对象ptr  
}  
  
//场景  
A.添加下游对象(B, nil) //A对象，删除B对象的引用。B被A删除，被标记为灰(如果B之前为白)  
A.添加下游对象(B, C) //A对象，更换下游B变成C。B被A删除，被标记为灰(如果B之前为白)
```

另一种比较经典的写屏障是黑色赋值器的 Yuasa 删除屏障，其基本思想是避免满足条件 2。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204602.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204611.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204707.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204715.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204727.png)

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230731204746.png)

##### 缺点
1. 删除写屏障也叫基于快照的写屏障方案，必须在起始时，STW 扫描整个栈（注意了，是所有的 goroutine 栈），保证**所有堆上在用的对象**都处于灰色保护下，保证的是弱三色不变式。


1. **为何需要在初始状态时扫描整个栈？**
	-  堆内存在被栈对象引用的白色堆对象，无法解决堆内黑色对象引用白色对象的问题。

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230801104501.png)


### 混合屏障

实际上 Go 中并没有真正实现过删除写屏障，Go 在 `v1.8` 引入了混合屏障，混合屏障将插入屏障和删除屏障结合了起来。

```go
// 添加下游对象的函数, 当前下游对象slot, 新下游对象ptr
func HybridWritePointerSimple(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    // 1) 将被删除的下游对象标记为灰色
    shade(*slot)
    // 2) 将新下游对象标记为灰色
    shade(ptr)
    // 3) 当前下游对象slot = 新下游对象ptr
    *slot = ptr
}
```

混合写屏障逻辑如下：
- `GC` 开始时只需将当前栈上所有对象标记为黑色，无须 `STW`。
- `GC` 期间在栈上创建的新对象均标记为黑色。
- 将被删除的下游对象标记为灰色。
- 将被添加的下游对象标记为灰色。


##### 优点
1. 由于结合了删除屏障和插入屏障，导致混合屏障没有了插入屏障扫描一次后 STW 的问题。
2. 由于结合了删除屏障和插入屏障，导致在初始时可以不初始化整个栈，而是只初始化一个 goroutine 的栈。


**优化 1**

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230801105336.png)

**优化 2**

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230801105439.png)

# 参考

- https://liangyaopei.github.io/2021/01/02/golang-gc-intro/
- https://colobu.com/2022/07/16/A-Guide-to-the-Go-Garbage-Collector/
- https://zhuanlan.zhihu.com/p/297177002
- https://liqingqiya.github.io/golang/gc/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/%E5%86%99%E5%B1%8F%E9%9A%9C/2020/07/24/gc5.html
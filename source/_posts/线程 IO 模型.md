---
title: 线程 I/O 模型
author: siegelion
tags:  [操作系统]
categories: [笔记]
date created: 2022-11-13 12:55
date updated: 2022-11-13 14:10
---

UNIX 网络编程中共提到了 5 种 I/O 模型，分别为阻塞 I/O、非阻塞 I/O、I/O 多路复用、信号驱动 I/O、异步 I/O，前四种都属于同步 I/O。

因为实际上 I/O 操作实际上分为两个阶段：将数据从 I/O 设备复制到内核区，将数据从内核区复制到用户区。前四种模型无法做到在两个阶段都不需要阻塞等待，他们都需要在第二个阶段阻塞等待数据从内核中复制到用户区，只有异步 I/O 模型可以做到无阻塞因此才被成为异步。

# 阻塞 I/O

最简单的一种 I/O 模型，当用户线程发出 I/O 请求之后，内核会去查看数据是否就绪，如果没有就绪就会等待数据就绪，而用户线程就会处于阻塞状态，用户线程交出 CPU。当数据就绪之后，内核会将数据拷贝到用户线程，并返回结果给用户线程，用户线程才解除阻塞状态。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20221111153558.png)

# 非阻塞 I/O

当用户线程发起一个 I/O 操作后，并不需要等待，而是马上就得到了一个结果。如果结果指示错误，它就知道数据还没有准备好，于是它可以再次发送 I/O 操作。一旦内核中的数据准备好了，并且又再次收到了用户线程的请求，那么它马上就将数据拷贝到了用户线程，然后返回。

在非阻塞 I/O 模型中，用户线程需要不断地询问内核数据是否就绪，也就说非阻塞 I/O 不会交出 CPU，而会一直占用 CPU。过程中会反复执行系统调用询问数据是否已经可以读取，这样会导致 CPU 占用率非常高。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20221111154019.png)

# I/O 多路复用

## Select

select 是操作系统提供的系统调用函数，select 用来等待文件描述词（普通文件、终端、伪终端、管道、FIFO、套接字及其他类型的字符型）状态的改变。是一个轮循函数，循环询问文件节点，可设置超时时间，超时时间到了就跳过代码继续往下执行。

通过 select，我们可以把一个文件描述符的数组发给操作系统， 让操作系统去遍历，确定哪个文件描述符可以读写， 然后告诉我们去处理：

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/737b74ad42211c34b6af242e74528726.gif)

**原理**

select 中描述符集合 fd_set 通过 bitmap 的方式进行存储。

> bitmap 的基本思想就是用一个 bit 位来标记某个元素是否存在，由于采用了 bit 为单位来存储数据，因此可以大大节省存储空间。
>
> 例如利用一个 32bit 的整数的每一个 bit 便可以表示 0~31 范围内的哪些数字存在哪些数字不存在。

fd_set 则是一个长度确定 unsigned long 数组，一共组成一个 1024bit 的连续空间，每一 bit 可以对应一个描述符 fd，因此一共可以对应 1024 个描述符。

```c
#define FD_SETSIZE 1024 
#define NFDBITS (8 * sizeof(unsigned long)) 
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS) 

// 数据结构 (bitmap) 
typedef struct { 
	unsigned long fds_bits[__FDSET_LONGS]; 
} fd_set;

```

**相关函数**

```c
int select(
	int maxfdp ,
	fd_set *readset , 
	fd_set *writeset , 
	fd_set *exceptset ,
	struct timeval *timeout
);
```

**参数**

- maxfdp：被监听的文件描述符的最大值，它比所有文件描述符集合中的文件描述符的最大值大 1，因为文件描述符是从 0 开始计数的；
- readfds、writefds、exceptset：分别指向可读、可写和异常等事件对应的描述符集合。
- timeout：用于设置 select 函数的超时时间，即告诉内核 select 等待多长时间之后就放弃等待。timeout == NULL 表示等待无限长的时间，timeout == 0，select 立即返回

```c
int FD_ZERO(int fd, fd_set *fdset); //一个 fd_set类型变量的所有位都设为 0 
int FD_CLR(int fd, fd_set *fdset); //清除某个位时可以使用 
int FD_SET(int fd, fd_set *fd_set); //设置变量的某个位置位 
int FD_ISSET(int fd, fd_set *fdset); //测试某个位是否被置位
```

**例如**

设置 fd_set 长度为 8，则对应 8 个 fd。

1. 执行 `FD_ZERO(&set);` 则 fd_set 用位表示是 00000000。
2. 用户传关注的文件描述符 `fd＝5,` 执行 `FD_SET(fd,&set);` 后 fd_set 变为 00010000(第 5 位为 1)。
3. 若再加入 `fd＝2，fd=1`，则 fd_set 变为 00010011。
4. 执行 `select(6,&set,0,0,0)` 阻塞等待。
5. 若 `fd=1,fd=2` 上都发生可读事件，则 select 返回，此时 set 变为 00000011。没有事件发生的 fd=5 被清空。

**评价**

1. select 本质上是设置与判断 bitmap 每一位上的值来判断对应描述符是否可以执行下一步的操作的。因为 bitmap 的大小是确定的，因此单个进程所打开的 fd 的数量是有限制的。
2. 每次调用 select，都需要把 fd 集合从用户态拷贝到内核态。
3. select 是采用轮询的方式判断描述符对应的硬件是否已经就绪，因此效率太低。

## Poll

select 和 poll 系统调用的本质一样，poll 的机制与 select 类似，与 select 在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是 poll 没有最大文件描述符数量的限制（ 基于链表来存储的，但是数量过大后性能也是会下降）。

poll 和 select 同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);


struct pollfd{ 
	int fd; //文件描述符 
	short events; //等待的事件 
	short revents; //实际发生的事件 
};
```

**参数**

- fds ： 指向一个 pollfd 数组的指针，用于指定测试某个给定的 fd 的条件。
- nfds ： 指定 fds 数组元素个数 ，相当于要检查多少个文件描述符。
- timeout：指定等待的毫秒数，无论 I/O 是否准备好，poll() 都会返回。

> 在用户态时描述符是通过数组方式存储的，在复制进内核后会被转换为链表的形式存储，因此没有存储大小的限制。（[关于poll的疑问？数组还是链表？](https://www.doudaxia.club/index.php/archives/97/)）
>
> 每个结构体的 events 域是由用户来设置，告诉内核我们关注的是什么，而 revents 域是返回时内核设置的，以说明对该描述符发生了什么事件。

**返回值**

成功时，poll() 返回结构体中 revents 域不为 0 的文件描述符个数；如果在超时前没有任何事件发生，poll() 返回 0。

## ePoll

epoll 没有对描述符数目的限制，**它所支持的文件描述符上限是整个系统最大可以打开的文件数目**，例如，在 1GB 内存的机器上，这个限制大概为 10 万左右。

epoll 只有 `epoll_create`，`epoll_ctl` 和 `epoll_wait` 这三个系统调用。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/fb2a1a9fba7fd722c2e6ce1ae8ba2ee1.gif)

当某一进程调用 epoll_create 方法时，Linux 内核会创建一个 eventpoll 结构体，这个结构体也即是 epoll 对象的一个句柄，会通过 `epoll_create` 函数返回。

```c
int epoll_create(int size); // 返回值就是 epoll 对象的句柄
```

```c
struct eventpoll { 
	// 监听列表：红黑树的根节点，这颗树中存储着所有添加到 epoll 中的需要监控的事件
	struct rb_root rbr; 
	// 就绪列表：双链表中则存放着将要通过 epoll_wait 返回给用户的满足条件的事件
	struct list_head rdlist; 
};
```

每一个 epoll 对象都有一个对应的 eventpoll 结构体，用于存放通过 `epoll_ctl` 方法向 epoll 对象中添加进来的事件，这些事件都会挂载在红黑树中。

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
```

而所有添加到 epoll 中的事件都会与设备驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫 `ep_poll_callback`，它会将发生的事件添加到 rdlist 双链表中。

**参数**

- epfd：如何确定将事件加入那个 epoll 对象呢？就是通过上文提到的句柄实现的。
- op：操作类型
  - EPOLL_CTL_ADD：注册新的 fd 到 epfd 中
  - EPOLL_CTL_MOD：修改已经注册的 fd 的监听事件
  - EPOLL_CTL_DEL：从 epfd 中删除一个 fd
- fd：需要注册监视对象文件描述符。
- event：告诉内核需要监听什么事件。

`epoll_wait` 的作用相当于 `select`，当调用 `epoll_wait` 检查是否有事件发生时，只需要检查 eventpoll 对象中的 rdlist 双链表中是否有事件元素即可。如果 rdlist 不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户。

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**参数**

- epfd：epoll_create 函数的返回值，epoll 对象的句柄
- events：是分配好的 epoll_event 结构体数组，epoll 将会把发生的事件赋值到 events 数组中
- maxevents：告诉内核这个 events 数组有多大，这个 maxevents 的值不能大于创建 `epoll_create` 时的 size。
- timeout：是超时时间，如果函数调用成功，则返回对应 I/O 上已准备好的文件描述符数目，如果返回 0 则表示已经超时。

# 信号驱动 I/O

在信号驱动 I/O 模型中，当用户线程发起一个 I/O 请求操作，会给对应的 I/O 操作注册一个信号函数，然后用户线程会继续执行，当内核数据就绪时会发送一个信号给用户线程，用户线程接收到信号之后，便在信号函数中调用中断程序，中断原本正在执行的操作，转而进行实际的 I/O 请求操作。

信号驱动 I/O 的前半部分是异步进行的，不需要阻塞等待 I/O 设备数据的写入，也不需要通过轮询的方式判断数据是否就绪，但将数据从内核种复制出来的过程，依然需要中断原本的操作，转而去读取数据，这部分的操作导致这种模型依然不是全异步的。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20221111154708.png)

# 异步 I/O

**Reactor**

在 Reactor 模式中，会先对每个 client 注册感兴趣的事件，然后有一个线程专门去轮询每个 client 是否有事件发生，当有事件发生时，便处理每个事件，当所有事件处理完之后，便再转去继续轮询。从这里可以看出，**多路复用 I/O 就是采用 Reactor 模式。**

**Proactor**

在 Proactor 模式中：当检测到有事件发生时，会新起一个异步操作，然后交由 **内核线程** 去处理，当内核线程完成 I/O 操作之后，发送一个通知告知操作已完成；可以得知，**异步 I/O 模型采用的就是 Proactor 模式。**


异步 I/O 模型是理想的 I/O 模型，在异步 I/O 模型中，当用户线程发起读取数据的操作之后，立刻就可以开始去做其它的事。因此不会对用户线程产生任何阻塞。

内核收到 read 请求之后，会生成一个内核线程，该线程会等待数据准备完成，然后将数据拷贝到用户区，当这一切都完成之后，内核会给用户线程发送一个信号，告诉它 read 操作完成了。

用户线程甚至不用阻塞去到内核区拷贝数据，因为内核线程已经将数据拷贝至用户区了。因此异步 I/O 才是真正的异步实现，在两个阶段都做到了异步。

# 参考

[Linux 网络编程的5种IO模型：异步IO模型](https://www.cnblogs.com/schips/p/12575933.html)

[IO多路复用API总结](https://cloud.tencent.com/developer/article/1922028)

[彻底理解 IO 多路复用实现机制](https://juejin.cn/post/6882984260672847879)

[Linux I/O 多路复用，select / poll / epoll 详解](https://blog.csdn.net/weixin_52622200/article/details/126369699)

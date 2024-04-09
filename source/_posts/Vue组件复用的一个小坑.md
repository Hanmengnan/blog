---
title:  Vue组件复用的一个小坑
tags:  [前端,Vue]
categories: [笔记]
date: 2021/1/21 20:46:25
---


![](https://b3logfile.com/bing/20190410.jpg?imageView2/1/w/960/h/540/interlace/1/q/100)

在推进项目的过程中，我构建了这样的一个组件：“一个列表存在两种不同的样式：自动滚动与手动滚动”。

实现思路其实很简单，预留一个`props`参数用于标识样式，然后使用样式绑定的方式进行设置就可以了。

#### 动效实现

> 说一下怎么实现“自动滚动”的动效吧，动效如下图所示。

![20210121215740.gif](https://b3logfile.com/file/2021/01/20210121215740-a31d9411.gif)

1. 设定计时器：设定一个计时器，按照一定时间间隔将向上移动的行数加一。
   
   ```
   timer = setInterval(() => {
       if (this.activeIndex < this.body.length) {
           this.activeIndex += 1;
       } else {
       this.activeIndex = 0;
       }
     }, 1000); // 移动时间间隔
   }
   ```
2. 计算偏移量：`this.$refs.list`是将要移动的列表，通过`scrollHeight`选取内容的全部高度（包括被隐藏的部分），除以行数得出每行的高度，只要每次向上移动一行的高度，就相当于在向上移动了。当然也可以将这个滚动值设为常数。将滚动值乘以当前已经移动的行数，再取负值，在样式中设为离顶部的距离，向上移动的部分就因为溢出而被隐藏了。
   
   ```
   top: function() {
       let move = 0;
       if (this.activeIndex !== 0) {
           move = this.$refs.list.scrollHeight / this.body.length; // 动态计算移动步长
         }
       return -this.activeIndex * move + "px"; // 当前移动距离
   }
   ```
3. 销毁计时器：作为一个好习惯，在组件销毁时，计时器一定也要进行销毁。
   
   ```
   beforeDestroy() {
        clearInterval(timer);
     }
   ```

---

但问题就出在了这个计时器上。初期我是使用`let`变量对计时器进行保存的，这样在复用组件的过程中，实际上多个组件的计时器都在引用这个计时器`timer`，当一个组件销毁时他就要销毁这个计时器`timer`，其他组件中的`timer`因为引用的也是它，所以都同时被销毁了。

导致效果不尽如人意：
![20210121233746.gif](https://b3logfile.com/file/2021/01/20210121233746-cb270cb1.gif)



将`timer`改为组件的的成员变量，这样`timer`的生命周期就完全和拥有它的组件同步了，不会被其他组件所销毁了。

修改后效果达到预期：

![20210121233351.gif](https://b3logfile.com/file/2021/01/20210121233351-24685f5d.gif)
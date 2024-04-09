---
title: JavaScript 任务的执行顺序
author: siegelion
date: 2021/2/4 12:11
tags: [JavaScript,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---

## 前言

上一文，我们介绍了`Promise`对象，`Promise`结束时执行的是异步操作，但这里提到的异步操作他的执行顺序是怎样的？

让我看一个例子：

```javascript
var p = new Promise(function (resolve, reject) {
    console.log("start");
    resolve("ok");
});

p.then((msg) => console.log(msg));
console.log("end");
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210204094535649.png)

咦~这里也许会十分让人困惑，`Promise`对象定义后会立即执行，按照代码的顺序，应该是先执行`then`语句啊。

但是这样想就在是按照以往同步操作的思路去思考，但是`then`语句中执行的是异步操作，所以执行顺序与我们所想不一致。

## 同步任务与异步任务



![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210204094252915.png)

如图所示，程序首先将同步任务加入任务栈，任务栈中的同步任务将首先执行，同步任务执行后从**异步任务队列**中取出任务加入任务栈，接着执行任务栈中的任务。

对应上文的程序，执行顺序则为：

1. 执行`Promise`语句，打印出`start`。
2. 将`then`语句加入异步任务队列。
3. 打印出`end`。
4. 同步任务执行完毕，执行异步任务`then`语句，打印出`ok`。

## 宏任务与微任务

### 提出问题

我们再来看一段代码：

```javascript
console.log(1)

setTimeout(function(){
    console.log(2)
},0)

new Promise(function(resolve){
    resolve()
}).then(function(){
    console.log(3)
})

console.log(4)
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210204101617891.png)

如果先执行同步任务打印出的`1`和`4`，接下来应该执行异步任务，那么是不是应该先打印`2`，但为什么先打印的是`2`呢？

### 两种任务

这里需要引入宏任务和微任务的概念。所以宏任务与微任务并非为异步任务的子集，而是任务形式的另一种划分。

#### 宏任务

**宏任务**（macrotask），**可以理解是每次执行栈执行的代码就是一个宏任务**（包括每次从事件队列中获取一个事件回调并放到执行栈中执行）。

> 浏览器为了能够使得宏任务与DOM任务能够有序的执行，**会在一个宏任务执行结束后，在下一个宏任务执行开始前，对页面进行重新渲染**。

宏任务包括：

| #                       | 浏览器 | `Node.js` |
| ----------------------- | ------ | --------- |
| 主代码块                | √      | √         |
| `setTimeout`            | √      | √         |
| `setInterval`           | √      | √         |
| `setImmediate`          | x      | √         |
| `requestAnimationFrame` | √      | x         |

> 优先级：主代码块 > setImmediate > MessageChannel > setTimeout / setInterval

#### 微任务

**微任务**（microtask），**可以理解是在当前 task 执行结束后立即执行的任务**。也就是说，在当前task任务后，下一个task之前，在渲染之前。

微任务包括：

| #                  | 浏览器 | Node |
| ------------------ | ------ | ---- |
| `process.nextTick` | x      | √    |
| `MutationObserver` | √      | x    |
| `Promise`          | √      | √    |

> 优先级：process.nextTick > Promise > MutationObserver

---

在事件循环中，每进行一次循环操作称为 tick，每一次 tick 的任务[处理模型](https://www.w3.org/TR/html5/webappapis.html#event-loops-processing-model)是比较复杂的，但关键步骤如下：

- 执行一个宏任务（栈中没有就从事件队列中获取）
- 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中
- 宏任务执行完毕后，立即执行当前微任务队列中的所有微任务（依次执行）
- 当前宏任务执行完毕，开始检查渲染，然后GUI线程接管渲染
- 渲染完毕后，JS线程继续接管，开始下一个宏任务（从事件队列中获取）

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/1053223-20180831162350437-143973108.png)

## 复习

### 看一个例子

```javascript
//主线程直接执行
console.log('1');
//丢到宏事件队列中
setTimeout(function () {
    console.log('2');
    process.nextTick(function () {
        console.log('3');
    })
    new Promise(function (resolve) {
        console.log('4');
        resolve();
    }).then(function () {
        console.log('5')
    })
})
//微事件1
process.nextTick(function () {
    console.log('6');
})
//主线程直接执行
new Promise(function (resolve) {
    console.log('7');
    resolve();
}).then(function () {
    //微事件2
    console.log('8')
})
//丢到宏事件队列中
setTimeout(function () {
    console.log('9');
    process.nextTick(function () {
        console.log('10');
    })
    new Promise(function (resolve) {
        console.log('11');
        resolve();
    }).then(function () {
        console.log('12')
    })
})
```

1. 首先执行主代码中的同步任务，首先打印`1`，然后将`setTimeout`中的函数加入宏任务队列，`nextTick`加入微任务队列。
2. 执行主代码`Promise`中的代码，打印`7`，`then`方法加入微任务队列，第二个`setTimeout`加入宏任务队列，主代码执行结束。
3. 开始执行微任务队列中的任务，打印`6`与`8`。
4. 执行宏任务队列中的下一个任务，打印`2`，将`nextTick`加入微任务队列，执行`Promise`，打印`4`，并将该对象的`then`加入微任务队列。
5. 再次开始执行微任务队列中的任务，打印`3`和`5`。
6. 再开始执行第二个`setTimeout`中的代码，打印`9`，微任务队列加入`nextTick`，执行`Promise`打印`11`，`then`加入微任务队列。
7. 执行微任务队列，打印`10`与`12`。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210204112302045.png)

> 这里需要注意，`nextTick`的优先级高于`then`方法，因此如果将`nextTick`移至`then`之后，还是会先执行。



### 再看一个更复杂的例子

#### 须知：

由于因为`async await` 本身就是`promise generator`的语法糖。所以`await`后面的代码是微任务，所以

```javascript
async function async1() {
	console.log('async1 start');
	await async2();
	console.log('async1 end');
}
```

等价于

```javascript
async function async1() {
	console.log('async1 start');
	Promise.resolve(async2()).then(() => {
                console.log('async1 end');
        })
}
```



#### 问题：

```javascript
async function async1() {
    console.log('async1 start');
    
    await async2();
    setTimeout(function() {
        console.log('setTimeout1')
    },0)
    
   /* 等价于
    Promise.resolve(async2()).then(() => {
        setTimeout(function () {
            console.log('setTimeout1')
        }, 0)
    })
  */
}
async function async2() {

	setTimeout(function() {
		console.log('setTimeout2')
	},0)
}
console.log('script start');

setTimeout(function() {
    console.log('setTimeout3');
}, 0)
async1();

new Promise(function(resolve) {
    console.log('promise1');
    resolve();
}).then(function() {
    console.log('promise2');
});
console.log('script end');
```



1. 执行主代码中定义的`async1`与`async2`，打印`script start`，`setTimeout`加入宏任务队列。

2. 执行`async1`，打印`async1 start`，执行`await`，也就是先执行`async2`，再在微任务队列中加入`then`后的方法。

3. 执行`Promise`，打印`promise1`，并在微任务队列中加入`then`后的内容，打印`script end`，主代码执行结束。

4. 执行微任务队列中的内容，队列中的第一个任务是一个`setTimeout`，也就是在宏任务队列中加入内容，第二个微任务是打印`promise 2`，微任务执行结束。

5. 执行下一个宏任务，打印`setTimeout3`。

6. 没有微任务，执行下一个宏任务，打印`setTimeout2`。

7. 没有微任务，执行下一个宏任务，打印`setTimeout1`。

    

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210204115542615.png)

[这里有更多的异步笔试题](https://github.com/Advanced-Frontend/Daily-Interview-Question/issues/7)
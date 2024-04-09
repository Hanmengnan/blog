---
title: ES6 Promise
author: siegelion
date: 2021/2/3 12:57
tags: [JavaScript,ES6,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---



## `Promise`

### 基本用法

1. 对象的状态不受外界影响，只有内部执行才能决定`Promise`的状态。

2. 一旦状态改变，就不会再变，任何时候都可以得到这个结果，如果改变已经发生了，你再对`Promise`对象添加回调函数，也会立即得到这个结果。
3. 可以对`Promise`添加多个回调函数。

4. `Promise`一旦新建就会立即执行，无法中途取消。

5. `Promise`内部抛出的错误不会反应到外部。

`Promise`构造函数接受一个函数作为参数，该函数的两个参数分别是`resolve`和`reject`。它们是两个函数，由 JavaScript 引擎提供，不用自己部署。



###  `then`

`.then` 的第一个参数是一个函数，该函数将在 `promise resolved `后运行并接收结果。

`.then` 的第二个参数也是一个函数，该函数将在 `promise rejected` 后运行并接收 `error`。

```javascript
promise.then(
  function(result) { /* handle a successful result */ },
  function(error) { /* handle an error */ }
);
```



### `catch`

- `Promise.prototype.catch()`方法是`.then(null, rejection)`或`.then(undefined, rejection)`的别名，用于指定发生错误时的回调函数。

- 如果 `Promise` 状态已经变成`resolved`，再抛出错误是无效的。因为` Promise` 的状态一旦改变，就永久保持该状态，不会再变了。

- `Promise` 对象的错误具有“冒泡”性质，会一直向后传递，直到被捕获为止。

    ```javascript
     promise.then().then().catch();
    ```

- `Promise` 中发生的错误，不会影响外界。

    ```javascript
    const someAsyncThing = function() {
      return new Promise(function(resolve, reject) {
        // 下面一行会报错，因为x没有声明
        resolve(x + 2);
      });
    };
    
    someAsyncThing().then(function() {
      console.log('everything is great');
    });
    
    setTimeout(() => { console.log(123) }, 2000);
    // Uncaught (in promise) ReferenceError: x is not defined
    // 123
    ```

    

###  `finally`

`finally()`方法用于指定不管 `Promise` 对象最后状态如何，都会执行的操作。是`then()`的特例。



### `all`

`Promise.all()`方法用于将多个 `Promise` 实例，包装成一个新的 `Promise` 实例。

上面代码中，`Promise.all()`方法接受一个数组作为参数，另外，`Promise.all()`方法的参数可以不是数组，但必须具有 `Iterator` 接口，且返回的每个成员都是 `Promise` 实例。

如果成员不是`Promise`实例，就会先调用下面讲到的`Promise.resolve`方法，将参数转为 `Promise` 实例，再进一步处理。



> 例如：

```javascript
const p = Promise.all([p1, p2, p3]);
```

1. 只有`p1`、`p2`、`p3`的状态都变成`fulfilled`，`p`的状态才会变成`fulfilled`，此时`p1`、`p2`、`p3`的返回值组成一个数组，传递给`p`的回调函数。

2. 只要`p1`、`p2`、`p3`之中有一个被`rejected`，`p`的状态就变成`rejected`，此时第一个被`reject`的实例的返回值，会传递给`p`的回调函数。

> 可以理解为“与”操作。



### `any`

该方法接受一组 `Promise` 实例作为参数，包装成一个新的 `Promise` 实例返回。只要参数实例有一个变成`fulfilled`状态，包装实例就会变成`fulfilled`状态；如果所有参数实例都变成`rejected`状态，包装实例就会变成`rejected`状态。

> 可以理解为”或“操作。



### `race`

方法的具体功能如方法名一样。

只要`p1`、`p2`、`p3`之中有一个实例率先改变状态，`p`的状态就跟着改变。那个率先改变的 `Promise` 实例的返回值，就传递给`p`的回调函数。



### `allSettled`

`Promise.allSettled()`方法接受一组 Promise 实例作为参数，包装成一个新的 Promise 实例。只有等到所有这些参数实例都返回结果。

```javascript
const promises = [
  fetch('/api-1'),
  fetch('/api-2'),
  fetch('/api-3'),
]
Promise.allSettled(promises).then();
```



### `resolve`

`Promise.resolve()`方法将现有对象转为 `Promise `对象。

1. 参数是一个 `Promise` 实例：

    如果参数是 `Promise `实例，那么`Promise.resolve`将不做任何修改、原封不动地返回这个实例。

2. 参数是一个`thenable`对象:

    > `thenable`对象指的是具有`then`方法的对象。

    `Promise.resolve()`方法会将这个对象转为 Promise 对象，然后就立即执行`thenable`对象的`then()`方法。

3. 参数不是具有`then()`方法的对象，或根本就不是对象:

    如果参数是一个原始值，或者是一个不具有`then()`方法的对象，则`Promise.resolve()`方法返回一个新的 `Promise `对象，状态为`resolved`。

    ```javascript
    const p = Promise.resolve('Hello');
    
    p.then(function (s) {
      console.log(s)
    });
    // Hello
    ```

4. 不带有任何参数:

    `Promise.resolve()`方法允许调用时不带参数，直接返回一个`resolved`状态的 `Promise `对象。但是需要注意：这个对象的`then`方法在本轮事件循环结束时执行。



### `reject`

上面代码生成一个 `Promise` 对象的实例，状态为`rejected`，回调函数会立即执行。



### `try`

由于`catch`只能捕获`Promise`对象内部的异步错误，要想捕获同步错误，需要写成这样：

```javascript
try {
  database.users.get({id: userId})
  .then(...)
  .catch(...)
} catch (e) {
  // ...
}
```

但是可以用`Promise.try`改写上述：

```javascript
Promise.try(() => database.users.get({id: userId}))
  .then(...)
  .catch(...)
```


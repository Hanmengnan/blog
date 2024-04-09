---
title: JavaScript的迷惑原型链
author: siegelion
date: 2021/4/12 11:13
tags: [JavaScript,前端,ES6]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---

> 虽然面试可能凉了，但是还是收获了知识，我还觉得OK~

准备前端的面试绕不开的一个问题就是“对JavaScript的原型链的理解”，我自然也详细了解了一下这个问题，随着问题的深入，我发现虽然之前我了解过原型链的思想，但是很多细节扣的都不够深入，我只是了解了原型链的基本思想，最深处还有宝藏（坑）等待发掘（懵逼）。

## ES5

先看一个`ES5`中对象继承的基本实现：

```javascript
var a = function () {
    this.name = "i am a";
}
a.prototype.sayHello = function () {
    console.log(this.name);
}

var b = function () {
    a.call(this);
    this.age = "22";
}

b.prototype = Object.create(a.prototype);
b.prototype.constructor = b;
```

这样就相当于`b`继承于`a`对吧？我们验证一下：

```javascript
var obj_a = new a();
var obj_b = new b();

console.log(obj_b instanceof a);
// true
```

> 这里提一下`instanceof`的原理：
>
> ```javascript
> a instanceof B
> ```
>
> 就是在对象`a`的原型链上检查是否有`B.prototype`

OK! 我们再深入一下，看一下原型链上的关系：

```javascript
console.log(Object.getPrototypeOf(obj_b) instanceof a);
// true
console.log(Object.getPrototypeOf(Object.getPrototypeOf(obj_b)) instanceof a);
// false
console.log(Object.getPrototypeOf(Object.getPrototypeOf(obj_b)) === a.prototype);
// true
```

也就是下图所示：

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210412095957344.png)

但是仅仅这样就算完了吗？那和之前的理解没啥区别啊。我们再继续深入一下：

```javascript
console.log(Object.getPrototypeOf(a.prototype) == Object.prototype)
// true

console.log(Object.getPrototypeOf(Object.prototype) === null);
// true
```

我们知道类`a`不继承于其他类了，那么`a`的原型对象作为一个对象，它一定是由`Object`函数生成的，那么它的原型对象就是`Object.prototype`。

`Object.prototype`的原型对象的比较特殊是`null`。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210412101114374.png)

**下面开始逐渐令人迷惑了：**

我们知道`a()`、`b()`甚至是`Object()`他们都是一个函数，那么函数也是一个对象，既然这样他们也是有原型的，也会拥有一条原型链，那么它的原型对象是是谁呢？

也许直接思考原型对象不太好得出结论，我们换一个角度，它的构造函数是谁？我们知道有这样一个函数：`Fucntion()`。可以用它生成任意的函数对象：

```javascript
console.log(Object.getPrototypeOf(a).constructor===Function)
// true
console.log(Object.getPrototypeOf(a)===Function.prortotype)
// true
```

那既然`a（）`作为一个函数它可以使上文的式子成立，那么`Function`本身作为一个函数的构造函数，它也是一个函数啊，那么上面的式子按理说也可以成立啊。

```javascript
console.log(Object.getPrototypeOf(Function).constructor === Function)
// true
```

对于这个迷惑的操作，可以理解为`Function`可以构造任何函数，`Function`作为一个函数，当然可以自己构造自己。

既然上面的式子成立，自然下面的式子也可以成立：

```javascript
console.log(Object.getPrototypeOf(Function) === Function.prototype)
// true
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210412104224220.png)

那么`Object()`作为一个函数，自然也可以使上文中提到的式子成立：

```javascript
console.log(Object.getPrototypeOf(Object) === Function.prototype)
// true
console.log(Object.getPrototypeOf(Object).constructor === Function)
// true
```

同时`Function.prototype`作为一个对象，那么它的原型对象也就是`Object`的原型对象。

```javascript
console.log(Object.getPrototypeOf(Function.prototype) ===Object.prototype)
//true
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210412104903232.png)

> 大功告成！

## ES6

在`ES6`中引入了`class`这一用法，它其实是一个语法糖，与`ES5`中的对象用法大同小异。唯一有些不同的地方在于：

```javascript
class a {
    name = "a";
    sayHello() {
        console.log(this.name);
    }
}
class b extends a {
    age = "22";
}

var obj_a = new a();
var obj_b = new b();

console.log(Object.getPrototypeOf(b)===a);
// true
console.log(Object.getPrototypeOf(b.prototype)===a.prototype)
// true
```

也就是说这时，`b()`作为一个函数，它的原型对象是`a()`。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210412111233009.png)


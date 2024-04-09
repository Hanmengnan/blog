---
title: JavaScript 对象与原型链
author: siegelion
date: 2021/2/2 14:40
tags: [JavaScript,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---

> 再次吐槽：`JavaScript`中对象的概念真的是太让人困扰了，不同于我接触过的其他语言。

### 构造函数与`new`

#### 构造函数

`JavaScript`中对象的实现是基于构造函数和原型链的。

构造函数就是对象的模板，使用`new`执行构造函数，就可以返回对象的一个实例。

其实构造函数与普通的函数并没有太大的区别，如下：

```javascript
var obj = function () {
  this.name = "myObj";
};
```

目前看来他与普通函数的区别，可能在于函数内部使用了`this`关键字。当使用`new`执行这个构造函数时，`this`指向新生成的对象实例，自然也就将属性（`name`）关联在了实例上。

如果不使用new去执行构造函数，也即如普通函数一般去调用该函数，一样是可以执行的。如前文中我们提及的那样。

```javascript
var temp = obj();
```

这时`obj`不存在最左临近对象，那么这时函数中的`this`指向全局对象，并且函数并没有返回值，所以`temp`为`undefined`，严格模式下会禁止这种情况的发生。

#### `new`关键字

##### 原理

1. 创建一个空对象，作为将要返回的对象实例。

2. 将这个空对象的原型，指向构造函数的`prototype`属性。

    > 这句话引入了一个属性prototype，我们还没提到他，暂且不要在这里纠结吧。

3. 将这个空对象赋值给函数内部的`this`关键字。

4. 开始执行构造函数内部的代码。

##### 返回值

- 若函数（无论是构造函数还是普通函数）的尾部存在`return`语句，并且返回的还是一个对象，那么经过`new`命令执行后，都会返回该对象。

- 反之函数的尾部并未返回一个对象（返回空或者其他值）
    - 构造函数：返回`this`指向的对象实例。
    - 普通函数：返回一个空对象。

### 原型链

#### `proptype`

还记得上文中提到的属性`prototype`吗，这是原型链中绕不开的一个属性，这是构造函数拥有的一个属性，这个属性对应一个对象，让我们看一个简单的例子：

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210202113524418.png)

我们声明了一个简单的构造函数`Person`，`Person`的`prototype`属性指向一个对象——原型对象。

这个对象可以看作是所有实例都可以访问的公共容器，实例是以引用方式访问容器内的内容，所以如果一个方法定义在了`prototype`上，那么所有的实例都可以访问，并且每次访问时都是在访问同一个方法。

我们是否有什么办法访问这个容器呢？我们可以利用`__prop__`这个属性去从对象上访问他的原型对象。

> 凡是对象都具有`__prop__`属性。

> 但是这个方法仅限于浏览器的环境下，因为这个不是规范中的内容，而是由浏览器自己添加的方法，所以在`node.js`的环境下是无法使用这个属性的。
>
> 可以使用`Object.getPrototypeOf`代替。

我们可以实例化一个对象`someone`，访问他的`__prop__`属性，再访问`Person`的`proptype`属性，我们可以看到他们指向一个对象。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210202122745071.png)

我们可以再验证一下：

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210202122835529.png)

当然作为`someone`的原型对象`Person`也是一个对象，所以它同样也拥有一个原型对象，在上图中我们可以看到，他的原型对象就是`Object`。因此对象有原型，原型对象又有原型对象，这也就构成了一个链条，这也就是原型链。

`Object`比较特殊，他的原型对象是`null`，`null`是原型链的尽头。



#### `constructor`

我们还可以发现，原型对象中都存在一个属性`constructor`，该属性指明了该对象由哪个构造函数生成。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210202125522559.png)

点开这个属性，我们可以看到他指向一个函数，也就是该对象的构造函数。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210202125601396.png)

---



下面这章图片，很好的解释了原型链：

> 虚线`[p]`：`__prop__`
>
> 实线`p`：`proptype`
>
> 实现`[new]`：`constructor`
>
> 椭圆：对象
>
> 矩形：函数

![原型链](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/v2-2e8ec703287854d174483ba5f9f937cf_720w.jpg)

#### 属性/方法获取

当在一个实例上执行一个方法，会先在实例对象上寻找该方法。

```javascript
function Person(nick, age) {
  this.nick = nick;
  this.age = age;
  this.sayNick = function () {
    console.log("nick");
  }
}

var someone = new Person("bob", 23);
someone.sayNick(); //nick
```

若在实例对象上找不到，则在对象的原型链上寻找。

```javascript
function Person(nick, age) {
  this.nick = nick;
  this.age = age;
}

Person.prototype.sayNick = function () {
  console.log(this.nick);
}
var someone = new Person("bob", 23);
someone.sayNick(); //bob
```

实例对象上定义的方法，会覆盖原型上的方法。

```javascript
function Person(nick, age) {
  this.nick = nick;
  this.age = age;
  this.sayNick = function () {
    console.log( "nick");
  }
}

Person.prototype.sayNick = function () {
  console.log(this.nick);
}
var someone = new Person("bob", 23);
someone.sayNick();
```



### `Object`对象相关方法

- `Object.getPrototypeOf()`

    获取对象的原型

    ```javascript
    Object.getPrototypeOf(Object.prototype) === null // true
    ```

- `Object.setPrototypeOf`

    设置对象的原型

    ```javascript
    var a = {};
    var b = {x: 1};
    Object.setPrototypeOf(a, b);
    Object.getPrototypeOf(a) === b // true
    ```

- `Object.create()`

    以实例对象作为新实例的原型对象，并返回该新创建的实例对象。

    ```javascript
    var B = Object.create(A);
    Object.getPrototypeOf(B) === A // true
    ```

    `A`作为`B`的原型对象，因此若是在`B`上调用`A`的方法，是以引用方式进行调用，`A`的方法修改，`B`再次调用时，调用的方法即为改变后的。

    B的构造函数和A相同。

- `isPrototypeOf`

    用来判断该对象是否为参数对象的原型。

- `Object.is() `

    比较两个值是否相等，克服了`NaN`不等于自身，以及`-0`与`+0`相等的问题。

- `Object.assign()`

    方法用于对象的合并，将源对象（source）的所有可枚举属性，复制到目标对象（target）。

    1. 只拷贝可枚举属性。
    2. 遇到非对象，会转换为对象进行拷贝。
    3. 拷贝的值为浅拷贝。
    4. 会替换同名属性。
    5. 会执行取值函数（`get`），将执行后的值拷贝。

### 对象继承

第一步是在子类的构造函数中，调用父类的构造函数。

```javascript
function Sub(value) {
  Super.call(this);
  this.prop = value;
}
```

第二步，是让子类的原型指向父类的原型，这样子类就可以继承父类原型。

```javascript
Sub.prototype = Object.create(Super.prototype);
Sub.prototype.constructor = Sub;
```

还有一种方法：

```javascript
Sub.prototype = new Super();
```

> 但是这样会在`Sub`的`proptype`中引入`Super`实例的属性，因此不推荐。


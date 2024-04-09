---
title: JavaScript this 关键字
author: siegelion
date: 2021/1/25 18:30
tags: [前端,JavaScript]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---

> 我接触过Python这门语言，Python中也是用`this`关键字实现了面向对象的某些概念。JavaScript中也有异曲同工之妙，所以见到`this`时我并不是很陌生，但是没想到JavaScript中的`this`的注意事项却比Python中的多得多，而且很让人头大。
>
> JavaScript真是一门混乱的语言！ 🥱

## 属性

### 指向对象

简单来说：`this`用来指代属性或方法当前所在的对象。

```javascript
var A = {
  name: '张三',
  describe: function () {
    return '姓名：'+ this.name;
  }
};
```

有的语境下`this`可能不指代任何对象，如下函数中出现`this`关键字，但是在这个函数未被调用的情况下，`this`不指代任何对象。

```javascript
var f = function () {
  console.log(this.data);
}
```

这时候`JavaScript`中的`this`游离于对象之外，任意一个函数的定义中的都可能出现`this`关键字。`this`指向的对象取决于他调用的对象，例如：

```javascript
var A = {
    name: '张三',
    describe: function () {
      return '姓名：'+ this.name;
    }
  };
  
  var B = {
    name: '李四'
  };
  
  B.describe = A.describe;
  console.log(B.describe === A.describe);
  // true
  console.log(B.describe());
  // "姓名：李四"
```

`describe`本为在对象`A`中定义的方法，将`B`对象的`describe`方法也指向内存中的相同位置，但因为调用其的对象不同，导致结果的不同。换句话也可以理解为他所处的环境（上下文），而环境的判断方式为——寻找`this`关键字的左邻最近对象。例如：

```javascript
var f = function () {
  console.log(`this`.x);
}
var obj1 = { data:"1",f: f };
var obj2 = { data:"2",f: f };

obj1.f(); //1
obj2.f(); //2
```

再看一个稍复杂的例子：

```javascript
var a = {
  p: 'Hello',
  b: {
    m: function() {
      console.log(this.p);
    }
  }
};
a.b.m(); // undefinded
```

### 指向`window`

距离包含`this`的函数`m`的最左临近对象为`b`，所以`this`所指的对象为`b`,`b`中不存在属性`p`，所以输出`undefined`。

但是也会出现不存在最左临近对象的情况，这时候`this`所指的就是顶层对象`window`。

```javascript
var f = function () {
  console.log(this.x);
}
f(); //undefined
```

这时候可以采用“严格模式”，“严格模式”下函数中的`this`不允许被指为`window`，不然则会报错。

```javascript
var f = function () {
  'use strict'
  console.log(this.x);
}
f(); //undefined
```

## 使用场合

1. 全局环境

    ```javascript
    this === window // true
    
    function f() {
      console.log(this === window);
    }
    f() // true
    ```

2. 构造函数

    ```javascript
    var Obj = function (p) {
      this.p = p;
    };
    ```

3. 对象中方法

    ```javascript
    var obj ={
      foo: function () {
        console.log(this);
      }
    };
    obj.foo() // obj
    ```

## 注意

### 多层嵌套下的`this`

下面的例子有些复杂，我想了一会。

 ```javascript
var o = {
  f1: function () {
    console.log(this);
    var f2 = function () {
      console.log(this);
    }();
  }
}

o.f1()
 ```

可以对其进行如下的变体：

```javascript
var o = {
  f1: function () {
    console.log(this);
    var f2 = function () {
      console.log(this);
    };
    f2();
  }
}

o.f1()
```

可以发现`f2`被声明定义之后调用，调用时它不存在最左临近对象，因此调用它的是`window`。

换一种思路，`f2`被调用的情况不属于`构造函数`也不属于`对象的方法`，因此一定是全局环境。两种方法得出的结论是一致的。

### 数组中的`this`

与上面同理，调用其的是`window`，不展开说了。

```javascript
var o = {
  v: 'hello',
  p: [ 'a1', 'a2' ],
  f: function f() {
    this.p.forEach(function (item) {
      console.log(this.v + ' ' + item);
    });
  }
}

o.f()
// undefined a1
// undefined a2
```

### 回调函数中的`this`

```javascript
var o = new Object();
o.f = function () {
  console.log(this === o);
}

// jQuery 的写法
$('#button').on('click', o.f);
```

此时调用函数的对象为`DOM`元素，也即`this`的指向对象。

## 绑定this的方法

### `call()`

函数实例的`call`方法，可以指定函数内部`this`的指向（即函数执行时所在的作用域），然后在所指定的作用域中，调用该函数。

`call`方法的参数，应该是一个对象。如果参数为空、`null`和`undefined`，则默认传入全局对象。

```javascript
var n = 123;
var obj = { n: 456 };

function a() {
  console.log(this.n);
}

a.call() // 123
a.call(null) // 123
a.call(undefined) // 123
a.call(window) // 123
a.call(obj) // 456
```

如果`call`方法的参数是一个原始值，那么这个原始值会自动转成对应的包装对象，然后传入`call`方法。

```javascript
var f = function () {
  return this;
};

f.call(5)
// Number {[[PrimitiveValue]]: 5}
```

`call`方法还可以接受多个参数。`call`的第一个参数就是`this`所要指向的那个对象，后面的参数则是函数调用时所需的参数。

```javascript
function add(a, b) {
  return a + b;
}

add.call(this, 1, 2) // 3
```

### `apply()`

`apply`方法的作用与`call`方法类似。唯一的区别就是，它接收一个数组作为函数执行时的参数，使用格式如下:

```javascript
function f(x, y){
  console.log(x + y);
}

f.call(null, 1, 1) // 2
f.apply(null, [1, 1]) // 2
```

### `bind()`

`bind()`方法用于将函数体内的`this`绑定到某个对象，然后返回一个新函数。

```javascript
var counter = {
  count: 0,
  inc: function () {
    this.count++;
  }
};

var func = counter.inc.bind(counter);
func();
counter.count // 1
```

`bind()`也可以接收多个参数，这些参数将会绑定原函数的参数。

#### 注意&用法

1. `bind`每次运行就返回一个新函数，所以下面的用法是无效的。

    ```javascript
    element.addEventListener('click', o.m.bind(o));
    element.removeEventListener('click', o.m.bind(o));
    ```

2. 与回调函数结合，解决`this`指向的问题。

    ```javascript
    var counter = {
      count: 0,
      inc: function () {
        'use strict';
        this.count++;
      }
    };
    
    function callIt(callback) {
      callback();
    }
    
    callIt(counter.inc.bind(counter));
    counter.count // 1
    ```

    若不这样做，`callback`调用时的`this`指向即为`window`。


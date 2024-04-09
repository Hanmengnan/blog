---
title: ES6 class
author: siegelion
date: 2021/2/2 22:40
tags: [JavaScript,ES6,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---



### 前言

`class`是`ES6`中引入的语法糖，其中的绝大部分的功能都可以使用`ES5`的方法做到。`class`可以看作是构造函数的另一种写法。

```javascript
class myClass {
  constructor() {
    {
      //...
    }
  }
}
// 等同于
var myClass = function () {
  //...
}
```

## `class`中的方法

在`class`中定义方法，就相当于在原型对象上定义方法。

```javascript
class B {}
const b = new B();

b.constructor === B.prototype.constructor // true
```

### `constructor`方法

```javascript
myClass.prototype.constructor === myClass
```

构造函数本身起到构造作用，使用`class`方法后，需要在类中引入一个`constructor`方法，该方法必须在“类”中存在，如果没有显式定义，引擎会自动为其添加一个空的`constructor`方法。

`constructor`默认返回实例对象即`this`，当然也可以如构造函数一样返回另外一个对象。

![image-20210202200829671](https://gitee.com/hanmengnan/images-of-notes/raw/master/notes/image-20210202200829671.png)

如上图，当`constructor`方法返回一个对象，`new`命令执行后返回的就是该对象，当`constructor`有返回值，但是返回的不是一个对象时，默认返回的还是`this`实例对象。

`class`必须使用`new`命令调用，不能如构造函数般地直接调用。

`constructor`中定义的实例对象的属性，可以使用一种新的写法：

```javascript
class foo {
  bar = 'hello';
  baz = 'world';

  constructor() {
    // ...
  }
}
```

等同于：

```javascript
class foo {
  constructor() {
     this.bar = 'hello';
  	 this.baz = 'world';
  }
}
```

### 静态

#### 静态方法

在`class`中的方法前加上`static`关键字，就说明该方法是静态方法，静态方法不会被实例对象继承，需要通过类来直接调用，静态方法中的`this`关键字指向类，而不是实例对象。

父类的静态方法可以被子类继承。

#### 静态属性

定义在类上的属性。

```javascript
class Foo {
}

Foo.prop = 1;
Foo.prop // 1
```

### 其他

#### `new.target`

- 构造函数中使用：返回`new`命令调用的构造函数，如果构造函数不是使用`new`命令或者`Reflect.construct()`进行调用，则返回`undefinded`。
- 类内部调用：返回当前的类。

#### `class`注意事项

1.  `class`内部默认采用“严格模式”。
2. 不存在“类”提升。
3. 方法名前加`*`，表示这是一个`Generator`方法。
4. `this`指向类的实例。



## `class` 继承

`class`中的继承是通过`extend`关键字实现的。

但是`ES6`中的继承机制与`ES5`不同。

`ES5`中是先存在子类的`this`对象，然后将其绑定至父类的构造函数上，执行父类的构造函数，然后再执行子类的构造函数，之后再将子类的原型对象指向父类，将原型中的构造函数指向自己的构造函数。

`ES6`则是需要调用父类的构造函数，附带作用是生成一个`this`对象，之后才能在`this`上进行添加与修改。所以`class`中进行继承，在子类的构造函数中若是想使用`this`关键字，需要先调用`super`方法。

### `Super`关键字

`super`关键字有两种用法：

1. 作为函数：代表父类的构造函数，虽然代表父类的构造函数，但是调用父类的构造函数时，其中的this却指向子类实例，super作为方法使用时，只能出现在子类的构造函数中。

    ```javascript
    class A {
      constructor() {
        console.log(new.target.name);
      }
    }
    class B extends A {
      constructor() {
        super();
      }
    }
    new A() // A
    new B() // B
    ```

    

2. 作为对象：

    - 指向父类的原型对象：可以使用其调用父类原型对象上的方法，但是需要注意此时，方法中的this也指向子类实例。

    ```javascript
    class A {
      constructor() {
        this.x = 1;
      }
      print() {
        console.log(this.x);
      }
    }
    
    class B extends A {
      constructor() {
        super();
        this.x = 2;
      }
      m() {
        super.print();
      }
    }
    
    let b = new B();
    b.m() // 2
    ```

    - 指向父类：可以使用其调用父类上的静态方法，此时this指向子类而不是子类实例。

##  `prototype`属性和`__proto__`属性

class 同时拥有这两个属性：

```javascript
class A {
}

class B extends A {
}
```

- 从构造函数的角度出发：`B`作为构造函数，`B`对应的原型对象的原型肯定是构造函数`A`的原型对象，这是在ES5中就成立的。

    ```javascript
    B.prototype.__proto__ = A.prototype;
    ```

    

- 但是`ES6`中引入了静态方法，`B`继承了`A`的静态方法，`A`、`B`可以直接调用静态方法，此时可以将其看作一个特殊的对象，那么对象是存在`__prop__`方法的，这时`B`的原型即为`A`。

    ```javascript
    B.__proto__ = A;
    ```

    
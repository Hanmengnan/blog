---
title: JavaScript 闭包理解
author: siegelion
date: 2021/1/29 14:30
tags: [JavaScript,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---



> 第一次接触闭包时，是在使用`Python`时遇到了一个功能需求，这个功能需要有一列按钮，按钮i在点击时触发回调函数打印出各自的索引。

我首先想到的是这样的实现方式：

```python
def displayIndex(index):
    print(index)

def ButtonSet():
    .....
    for index in range(5):
        ButtonList.append(Button)
        ButtonList[index].clicked.connect(displayIndex(index))
    .....
```

但结果并不如人意，每次点击的时候打印出的都是“5”。我因为这个问题苦恼了很长时间，后来BBfat给我指出了这种情况应该使用闭包来做。将代码改成了下面的样子：

```python
def makeFunc(index):
    def displayIndex():
        print(index)
    return displayIndex

def ButtonSet():
    .....
    for index in range(5):
        ButtonList.append(Button)
        ButtonList[index].clicked.connect(makeFunc(index))
    .....
```

这下问题完美解决了，但当时他并没有告诉我这样做的原理，我也就稀里糊涂的过去了。

----

现在接触到了`JavaScript`，闭包是这个语言中的一个高级特性，所以我这次决定将他弄懂。

### 访问原本无法访问的作用域

闭包的作用和变量的作用域息息相关，`JavaScript`中存在着两种作用域，全局作用域和函数作用域。函数内部可以读取全局作用域的变量，但反之则不可以，也就是作用域链的概念，子作用域可以向上回溯寻找父作用域中的变量，但是父作用域无法读取子作用域中的值。

既然这样，那么是否可以借助孙作用域来读取子作用域中的值，无疑是可以的。

```javascript
function f1() {
  var n = 999;
  function f2() {
    console.log(n);
  }
  return f2;
}

var result = f1();
result(); // 999
```

这就是闭包，即能够读取其他函数中变量的函数，由于函数中的值只能由子函数读取，所以闭包也可以理解为定义在函数中的函数，闭包可以记住诞生的环境，通过他可以访问原本无法访问的作用域。

### 保存运行状态

闭包的另一个作用就是让这些变量始终保持在内存中，即闭包可以使得它诞生环境一直存在。

```javascript
function createIncrementor() {
  var count = 5;
  return function () {
    return count++;
  };
}

var inc = createIncrementor();

console.log(inc()) // 5
console.log(inc()) // 6
console.log(inc()) // 7
```

我们知道函数创建时会在内存中创建属于他的作用域链，关联父作用域，初始化属于自己作用域其中的变量等。

函数运行时会存在运行时上下文，上下文中的变量就是通过复制（而不是引用）函数函数作用域链（保存在内存中）而来的。

上下文所用的内存会由系统检测到不再使用时自动回收，下一次运行时再次复制，所以上下文每次运行时的变量值都会和函数创建（声明）时一样。

但上文中闭包被创建时，闭包同样是一个函数，系统需要为他创建作用域，此时父函数的运行时上下文即为他的父作用域，他引用的变量`count`是通过指针的方式，指向了父函数运行时上下文的某块内存。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210129122703695.png)

我们可以看到只有被闭包引用的变量被包含进来了，未被引用的`temp`变量并未被引用。

父函数运行结束，系统开始释放他的运行时上下文，但是发现有一块内存视乎还在被使用（引用），所以并没有释放`count`所在的内存。父函数变量的状态得以保存。

因为这个变量是通过指针引用的方式被指向的，所以下一次`inc`运行时，用到这个变量还回去内存中的相应位置寻找，但此时`count`已经被修改为6而不是5。

> 可能会有疑惑？子函数使用`count`的方式为什么一定是指针引用方式，而不是复制？

我们可以类比一下！

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210129130027465.png)

函数`func`属于函数作用域，使用了父作用域（全局）中的变量`a`，如果不是采用引用的方式，如何解释`a`的值在函数外一样增加了呢？

所以我们可以得出，子作用域引用父作用域的变量的方式是指针引用。

### 封装私有属性和方法

```javascript
function Person(name) {
  var _age;
  function setAge(n) {
    _age = n;
  }
  function getAge() {
    return _age;
  }

  return {
    name: name,
    getAge: getAge,
    setAge: setAge
  };
}

var p1 = Person('张三');
p1.setAge(25);
p1.getAge() // 25
```

这个闭包看起来有些不太一样，这是因为这次返回的变量不是函数了，而是一个对象，我觉得他更像是立即执行函数。

#### 立即执行函数

```javascript
var module1 = (function () {
　var _count = 0;
　var m1 = function () {
　  //...
　};
　var m2 = function () {
　　//...
　};
　return {
　　m1 : m1,
　　m2 : m2
　};
})();
```

立即执行函数也可以达到封装属性的作用，因为函数执行后返回的对象只有变量的存取方法，而无法直接获取到变量本身。

---

> OK! 让我们回到开头的这个问题，`JavaScript`也会遇到开头`Python`的那种问题。

比如：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>JS Bin</title>
    <style>
      ul {
        list-style: none;
        padding: 0;
      }
      li {
        border: 1px solid black;
      }
    </style>
  </head>
  <body>
    <ul>
      <li>1</li>
      <li>2</li>
      <li>3</li>
      <li>4</li>
      <li>5</li>
      <li>6</li>
    </ul>
    <script>
      var ul = document.getElementsByTagName("ul")[0];

      var liList = ul.getElementsByTagName("li");
      for (var i = 0; i < 6; i++) {
        liList[i].onclick = function (i) {
          alert(i); // 为什么 alert 出来的总是 6，而不是 1、2、3、4、5
        };
      }
    </script>
  </body>
</html>
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20210129_133442.gif)

可以看到和`Python`中出现的情况类似。

我们分析一下为什么？

![1.gif](https://i.loli.net/2021/01/29/FfiIOhPMGmaDTqJ.gif)

可以看到，回调函数`function`中的`i`是引用方式进行传递的，在执行之前，他会随着全局环境（父作用域）中的值改变，在循环结束时，父作用域中的`i`的值已经变成了6，因此所有引用的值也变成了6，这个时候点击回调函数调用，弹出的值则为6。

我们要做的就是，在绑定的同时就将以后执行时的参数确定。

利用立即执行函数则可解决这个问题：

```javascript
var ul = document.getElementsByTagName("ul")[0];

var liList = ul.getElementsByTagName("li");
for (var i = 0; i < 6; i++) {
    !(function (i) {
        liList[i].onclick = function (i) {
            alert(i); // 为什么 alert 出来的总是 6，而不是 1、2、3、4、5
        };
    })(i);
}
```

利用立即执行函数与闭包结合也可以解决，因为闭包可以保存运行时状态，立即执行函数执行则让他保存下执行时的状态，而执行“立即执行函数”时就是在进行绑定，所以可以解决这个问题：

```javascript
var ul = document.getElementsByTagName("ul")[0];

var liList = ul.getElementsByTagName("li");
for (var i = 0; i < 6; i++) {
    liList[i].onclick = (function (i) {
        return function () {
            alert(i); 
        };
    })(i);
}
```

---

>晚上看了看`ES6`的特性，发现使用新的特性一句话就可以解决这个问题！！！

```javascript
var ul = document.getElementsByTagName("ul")[0];

var liList = ul.getElementsByTagName("li");
for (let i = 0; i < 6; i++) {
    liList[i].onclick = function () {   
        alert(i); 
    };
}
```

![3](https://i.loli.net/2021/01/29/NAjSZBWUOxqPFbd.gif)
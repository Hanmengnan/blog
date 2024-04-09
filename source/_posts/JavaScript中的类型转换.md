---
title: JavaScript中的类型转换
author: siegelion
date: 2021/1/26 13:20
tags: [JavaScript,前端]
categories: [笔记]
index_img: https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/src=http___201903.oss-cn-hangzhou.aliyuncs.com_js_900301-ec7ecc156a655d4fcf94361d71057e87.jpg&refer=http___201903.oss-cn-hangzhou.aliyuncs.webp
---



### 前言

> 这两个方法一般是交由JS去隐式调用，以满足不同的运算情况。
>
> 在数值运算里，会优先调用`valueOf()`。
>
> 在字符串运算里，会优先调用`toString()`。

#### `valueOf`

| **对象** |                        **返回值**                        |
| :------: | :------------------------------------------------------: |
|  Array   |                    返回数组对象本身。                    |
| Boolean  |                         布尔值。                         |
|   Date   | 存储的时间是从 1970 年 1 月 1 日午夜开始计的毫秒数 UTC。 |
| Function |                        函数本身。                        |
|  Number  |                         数字值。                         |
|  Object  |                 对象本身。这是默认情况。                 |
|  String  |                        字符串值。                        |
|          |          Math 和 Error 对象没有 valueOf 方法。           |

#### `toString`

| **对象** |                           **示例**                           |
| :------: | :----------------------------------------------------------: |
|  Array   | ![image-20210125143938687](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210125143938687.png) |
| Boolean  |                           布尔值。                           |
|   Date   | ![image-20210125144225513](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210125144225513.png) |
| Function | ![image-20210125144053258](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210125144053258.png) |
|  Number  |                           数字值。                           |
|  Object  | ![image-20210125144124726](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210125144124726.png) |
|  String  |                          字符串值。                          |

### 对象方法

#### `Number()`

* 构造器：

    ```javascript
    var a = new Number('123');
    typeof(a); //object
    a instanceof Number; //true
    ```

* 类型转换：

    ````javascript
    var b = Number('123');
    typeof(b); //number
    b instanceof Number; //false
    ````

    - 原始类型

        ```javascript
        // 数值：转换后还是原来的值
        Number(324) // 324
        
        // 字符串：如果可以被解析为数值，则转换为相应的数值。转换前会移除字符串两端的空格、换行符等。
        Number('324') // 324
        
        // 字符串：如果不可以被解析为数值，返回 NaN
        Number('324abc') // NaN
        
        // 空字符串转为0
        Number('') // 0
        
        // 布尔值：true 转成 1，false 转成 0
        Number(true) // 1
        Number(false) // 0
        
        // undefined：转成 NaN
        Number(undefined) // NaN
        
        // null：转成0
        Number(null) // 0
        ```

        

    - 复合类型（对象）

        ```javascript
        Number({a: 1}) // NaN
        Number([1, 2, 3]) // NaN
        Number([5]) // 5
        ```

        之所以会这样，是因为`Number`背后的转换规则比较复杂。

        1. 调用对象自身的`valueOf`方法。如果返回原始类型的值，则直接对该值使用`Number`函数，不再进行后续步骤。

        2. 如果`valueOf`方法返回的还是对象，则改为调用对象自身的`toString`方法。如果`toString`方法返回原始类型的值，则对该值使用`Number`函数，不再进行后续步骤。

        3. 如果`toString`方法返回的是对象，就报错。

#### `String()`

- 构造器：（同上）

- 类型转换：（同上）

    - 原始类型

        ```javascript
        String(123) // "123"
        String('abc') // "abc"
        String(true) // "true"
        String(undefined) // "undefined"
        String(null) // "null"
        ```

        

    - 复合类型

        ```javascript
        String({a: 1}) // "[object Object]"
        String([1, 2, 3]) // "1,2,3"
        String(function(){})//"function(){}"
        ```

        `String`方法背后的转换规则，与`Number`方法基本相同，只是互换了`valueOf`方法和`toString`方法的执行顺序。

        1. 先调用对象自身的`toString`方法。如果返回原始类型的值，则对该值使用`String`函数，不再进行以下步骤。
        2. 如果`toString`方法返回的是对象，再调用原对象的`valueOf`方法。如果`valueOf`方法返回原始类型的值，则对该值使用`String`函数，不再进行以下步骤。
        3. 如果`valueOf`方法返回的是对象，就报错。

#### `Boolean()`

- 构造器：（同上）

- 类型转换：（同上）

    - 原始类型

        以下5个值转换为`false`，其余值全部为`true`。

        ```javascript
        Boolean(undefined) // false
        Boolean(null) // false
        Boolean(0) // false
        Boolean(NaN) // false
        Boolean('') // false
        ```

    - 复合类型

        注意，所有对象（包括空对象）的转换结果都是`true`，甚至连`false`对应的布尔对象`new Boolean(false)`也是`true`。

        ```javascript
        Boolean({}) // true
        Boolean([]) // true
        Boolean(new Boolean(false)) // true
        ```

### 全局方法

#### `parserInt()`

- `parseInt`方法用于将字符串从左至右转为整数。

- 如果字符串头部有空格，空格会被自动去除。

- 如果字符串第一个字符不能转换为数字，返回`NaN`。空字符串也会返回`NaN`。

- 如果字符串以`0x`或`0X`开头，会按十六进制转换。

- 对于那些会自动转为科学计数法的数字，`parseInt`会将科学计数法的表示方法视为字符串，因此导致一些奇怪的结果。

    ```javascript
    parseInt(1000000000000000000000.5) // 1
    parseInt('1e+21') // 1
    parseInt(0.0000008) // 8
    parseInt('8e-7') // 8
    ```

- 可以指定转换的进制。

    ```javascript
    parseInt('1000', 2) // 8
    parseInt('1000', 6) // 216
    parseInt('1000', 8) // 512
    ```

    - 如果第二个参数不是数值，会被自动转为一个整数，这个整数只有在2到36之间。
    - 如果第二个参数是`0`、`undefined`和`null`，则直接忽略，按照10进制进行转换。
    - 如果转换时，字符串中出现非法字符或者进制数非法，则返回`NaN`。

#### `parserFloat()`

- `parseFloat`方法用于将一个字符串转为浮点数。
- 如果字符串符合科学计数法，则会按照科学计数法的实际值进行转换。
- `parseFloat`方法会自动过滤字符串前导的空格。
- 字符串的第一个字符不能转化为浮点数，则返回`NaN`。空字符串也会返回`NaN`。

#### `isNaN()`

> 相当于 `isNaN( Number ( ) )`


---
title: 有关Cookie、Session、Token以及Storage
author: siegelion
date: 2021/3/2 12:40
tags: [前端]
categories: [笔记]
index_img: 
---



> 因为一开始做的项目大部分都是WEB相关的，所以`Cookie`、`Session`是经常可以接触到的，但是只是知其然而不知其所以然。昨天再一次遇到，正好有时间于是便一次彻底地将其弄清楚了。



## `Cookie`

### `Cookie`的工作方式

``Cookie``是浏览器提供的功能，是存储在浏览器中的纯文本文件，每个浏览器的安装目录下都应该有一个目录去存储这些文件。每次浏览器向一个域名发起`HTTP`请求时都会检查是否拥有该域名下的``Cookie``，如果拥有则在请求头中携带上。如下：

![image-20210116113555581](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210116113555581.png)

``Cookie``可以在客户端（浏览器）存储一些经常用到的信息，如用户身份信息，以便用户每次请求都不必验证身份只要在`Cookie`携带约定好的信息，服务器就可以识别用户的身份。但是只有每次请求都携带的信息才适合存储在`Cookie`中，滥用``Cookie``会增加网络开销，在`localStorage`出现前``Cookie``就曾经被当作存储工具“滥用”。

`Cookie`的大小被限制为`4KB`，每个域名下的`Cookie`数量被限制为`20`个，当然这也不是绝对的，有些浏览器的数量可能要大于`20`个。`Cookie`的属性之间用一个分号和空格隔开。



### `Cookie`的属性选项

`Cookie`的属性包括一些功能选项：`expires/max-age`、``domain``、``path``、``secure``、``HttpOnly``，设置一个`Cookie`时可以选择通过设置这写选项来起到相应的功能，当然也可以不设置，若不设置这些值将被自动设置为默认值。

![image-20210116120611983](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210116120611983.png)

#### `expires`

`expires`用来表示`Cookie`的失效时间，是一个GMT格式的数据对于失效的`Cookie`浏览器会自动清空，如果没有主动设置这一时间，那么`Cookie`会被自动设置为``Session``，即为“会话`Cookie`”这种`Cookie`在浏览器关闭后会被清除。

`expires`是`http/1.0`协议中的选项，在新的`http/1.1`中这一选项已经被`max-age`选项取代。`max-age`指的是生命周期的时间（`expires` = `Cookie`的创建时间 + `max-age `）。`max-age`有三种取值，正数：有效期；零：`Cookie`应该删除；负数：`Session`类型的`Cookie`。

#### `domain` 和 `path`

`domain`即为域名，`path`即为路径，二者加在一起便构成了url，url便决定了`Cookie`何时可以被访问，何时请求中会被自动添加`Cookie`。

`domain`的默认值为当前网页的域名，`path`的默认值为当前的目录。

需要注意：

1. 在发送XHR请求时，即使`domain`和`path`满足条件，但是浏览器也不会添加`Cookie`到请求头中。
2. `domain`不可以设置为域名的公共后缀，如：`.com`、`.cn`。

#### `secure`

`secure`保证了`Cookie`只有在安全的请求中才会被发送，如请求使用的`HTTPS`等安全协议。默认情况下`secure`选项为空，即为不设置，所以这时候`HTTP`和`HTTPS`请求都可以发送`Cookie`。

但是需要注意的是，如果在客户端中想要修改`Cookie`中的`secure`选项，需要当前的网页使用的是`HTTPS`协议。

#### `HttpOnly`

这个选项用来设置``Cookie``是否能通过 `js` 去访问。默认情况下，``Cookie``不会带``HttpOnly``选项(即为空)，所以默认情况下，客户端是可以通过`js`代码去访问（包括读取、修改、删除等）这个``Cookie``的。当``Cookie``带``HttpOnly``选项时，客户端则无法通过`js`代码去访问（包括读取、修改、删除等）这个``Cookie``。



### 设置`Cookie`

#### 服务端设置`Cookie`

服务端通过在响应头中加入set-`Cookie`字段对`Cookie`进行设置，一个set-`Cookie`字段只能设置一个`Cookie`。服务端可以对上文提到的全部`Cookie`选项进行设置。

![image-20210116130800157](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210116130800157.png)

#### 客户端设置`Cookie`

客户端可以通过以下语句对`Cookie`进行设置。

```javascript
document.Cookie = "name=Jonh; ";
```

但是需要注意的是：

1. HTTP网页无法对`secure`字段进行设置。
2. 客户端无法对设置了`HttpOnly`的`Cookie`进行更改。

#### 修改 / 删除`Cookie`

1. 修改`Cookie`只要保证`domain`和`path`不变的情况，对`Cookie`的其他情况进行修改就可以了。
2. 删除`Cookie`同理，只要对`expires`字段进行更改，将其改为一个过期的时间就可以使浏览器删除该`Cookie`了。

## `Session`

既然有了`Cookie`对用户的信息和状态进行标识为什么还要引入`Session`呢？因为`Cookie`是以明文传输的，他的作用是标识用户、记录状态等。那我们是不是可以用一个无实际意义的字串表示用户，防止敏感信息以明文暴露了。

因此，像签名这样敏感的信息是不应该在`Cookie`中以明文进行传输的，而是以`Session`存储在服务端，`Session`像是账本，这样的数据不会直接交给用户，而是只交给用户一个`Session`ID，也就是账户的户号。

`Session`一般存储在服务端的内存中，也可以存储在内存数据库中。一个`Session`会持续一段时间，通过`Cookie`中携带`SessionID`的方式可以做到持久访问的会话，这种会话便称为`Session`，每次用户访问都会更新`Session`的生命，长时间未访问的`Session`会被移出内存，保存至速度相对较慢的位置。`Session`也有一段时间的生命周期，过时会被清理。

> `Session` 并不是某一种具体的实现而是一种方案，只要可以实现持久访问的会话的方案都可以称`Session`。

## `Token`

``Token``的理念与`Session`不同，`Session`是需要存储在后端，占用一定的空间，但是`Token`本身不占用空间，他只是将用户的敏感信息以一种加密的方式进行变换，实际上`Token`中携带的就是用户信息的加密变体。`Token`的使用可以参考`JWT`。

- `Session`：在后端保存了一个身份证数据库，交给用户一个身份证号，每次用户只需要用身份证号查询信息就可以了。耗费存储空间。同时还会遇到的一个问题是：“如果采用负载均衡的方式，可能上一次请求服务器A的时候将`Session`存储在了服务器上，下一次请求时因为负载均衡将我的请求发送到服务器B上，此时`Session`就不存在了。”
- `Token`：将用户的身份证进行加密，加密后将身份证还给用户，每次校验用户信息与状态，只需要将其解密就可以了。耗费解密时间。

但`Session`ID和`Token`都是以`Cookie`的形式在客户端进行存储的。

## `LocalStorage` / `SessionStorage`

二者都属于浏览器端存储，是HTML5的新特性，都继承自`Storage`，因此语法和原理上有极高的相似性。

#### 相同点：

1. 可以通过下面的方式访问：

    ```javascript
    window.localStorage
    window.`Session`Storage
    ```

2. 两者的操作语法是一致的（这里以localStorage为例）：

    ```javascript
    localStorage.setItem("name", "carter");  // 设置name: carter
    
    localStorage.age = "24";   // 设置age: 24
    
    localStorage.getItem("name");  // 获取name的值carter
    
    localStorage.removeItem("name"); // 删除name
    
    localStorage.clear();  // 清空localStorage
    ```

3. 均遵守同源规则
4. 存储大小都为5MB



#### 不同点：

1. 失效时间不同
    - `localStorage`：不会失效，会永久保存，需要手动删除。
    - `SessionStorage`：跟会话绑定，会话结束即清除。但注意：刷新、回退不会清除。
2. 有效范围不同
    - `localStorage`：同源跨标签可以共用。
    - `SessionStorage`：不同标签不能共用。

## `IndexedDB`

尽管`Storage`的`5M`的空间已经相对较大了，但仍然无法满足所有的前端存储需求。为此，`HTML5`规范推出了前端的事务型数据库`indexedDB`。它可以存储大量的结构化数据，具有几乎可以媲美后端数据库的读写性能。



### 感谢：

[彻底理解Token、Cookie、Session](https://www.cnblogs.com/moyand/p/9047978.html)

[Session,Token相关区别](https://www.cnblogs.com/xiaozhang2014/p/7750200.html)

[Cookie/Session 的机制与安全](https://harttle.land/2015/08/10/`Cookie`-`Session`.html)

[聊一聊Cookie](https://segmentfault.com/a/1190000004556040#articleHeader6)

[五分钟带你了解啥是JWT](https://zhuanlan.zhihu.com/p/86937325)




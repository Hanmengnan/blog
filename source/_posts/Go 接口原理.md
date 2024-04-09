---
title: Go 中的“面向对象”
author: siegelion
tags:
  - Go
categories:
  - 笔记
date created: 2023-03-14 11:28
date updated: 2023-03-15 16:01
---

# Go 中的”面向对象“

`Golang` 不同于 `Java`，并不是完全面向对象的，他没有对象和类的概念。

而是如引入了 `struct` 和 `interface` 的概念，曾经看到过一个说法，我觉得说的还挺形象的，Go 没有面向对象只有字段集和方法集🐶。

## 鸭子类型

鸭子类型的设计哲学是：如果某个东西，它具备鸭子拥有的一切能力，那么这个东西他就是一只鸭子。也即是说一个 `struct` 如果他实现了一个 `interface` 对应方法集中的全部方法，那么就可以被认为实现了这个接口。

这种设计思路关注对象的行为，而不是类型。这是动态类型语言如 `Python` 崇尚的一种设计哲学。例如，在 Python 中当调用此函数的时候，可以传入任意类型，只要它实现了 `say_hello()` 函数就可以。如果没有实现，运行过程中会出现错误。

```python
def hello_world(coder):
    coder.say_hello()
```

`Golang` 虽然本身又是一门静态类型语言，但在其设计时借鉴了动态类型语言的设计方式，也实现了这种设计。

```go
type duck interface {
	eat()
	swim()
}

type maleDuck struct {
	name string
}
type femaleDuck struct {
	name string
}

func (s maleDuck) eat() {
}
func (s maleDuck) swim() {
}
func (s femaleDuck) eat() {
}
func (s femaleDuck) swim() {
}

func beDuck(aDuck duck) {
    aDuck.eat()
    aDuck.swim()
}

func main() {
    var m maleDuck{"A"}
    var f femaleDuck{"B"}
    beDuck(m)
    beDuck(f)
}

```

## 方法调用

### 值类型与指针类型

在调用方法的时候，值类型既可以调用 `值接收者` 的方法，也可以调用 `指针接收者` 的方法；指针类型既可以调用 `指针接收者` 的方法，也可以调用 `值接收者` 的方法。

```go
package main

import "fmt"

type Person struct {
	age int
}

func (p Person) howOld() int {
	return p.age
}

func (p *Person) growUp() {
	p.age += 1
}

func main() {
	// qcrao 是值类型
	qcrao := Person{age: 18}
	// 值类型 调用接收者也是值类型的方法
	fmt.Println(qcrao.howOld())
	// 值类型 调用接收者是指针类型的方法
	qcrao.growUp()
	fmt.Println(qcrao.howOld())
	// ----------------------
	// stefno 是指针类型
	stefno := &Person{age: 100}
	// 指针类型 调用接收者是值类型的方法
	fmt.Println(stefno.howOld())
	// 指针类型 调用接收者也是指针类型的方法
	stefno.growUp()
	fmt.Println(stefno.howOld())
}
```

这其实是语法糖在起作用。

### 值接收者与指针接收者

```go
package main

import "fmt"

type coder interface {
	code()
	debug()
}

type Gopher struct {
	language string
}

func (p Gopher) code() {
	fmt.Printf("I am coding %s language\n", p.language)
}

func (p *Gopher) debug() {
	fmt.Printf("I am debuging %s language\n", p.language)
}

func main() {
	var c1 coder = &Gopher{"Go"}
	c1.code()
	c1.debug()
	
    var c2 coder = Gopher{"Go"}
	c2.code()
	c2.debug() // fatal
	
```

当一个将一个结构体的 **值** 或者 **指针** 赋值给一个 **接口** 后，就会展示出值/指针类型的接收者是否真正实现了一个接口的方法，因为并没有直接调用方法，因此这个时候语法糖便无法起作用了，也就露出了小犄角：

- 实现了接收者是值类型的方法，页会隐含地也实现了接收者是指针类型的方法。
- 但实现了接收者是指针类型的方法，却不会实现了接收者是值类型的方法。

# 接口

`iface` 和 `eface` 都是 Go 中描述接口的底层结构体，区别在于 `iface` 描述的接口包含方法，而 `eface` 则是不包含任何方法的空接口：`interface{}`。

## 空接口的结构

```go
type eface struct { 
	_type *_type
	data  unsafe.Pointer
}
```

- `_type` 是 Go 语言类型的运行时表示，表示的是赋给空接口的对象的类型，其中包含了很多类型的元信息，例如：类型的大小、哈希、对齐以及种类等。
- `data` 则指向接口具体的值，一般而言是一个指向堆内存的指针。

```go
type _type struct {
	size       uintptr
	ptrdata    uintptr
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	equal      func(unsafe.Pointer, unsafe.Pointer) bool
	gcdata     *byte
	str        nameOff
	ptrToThis  typeOff
}
```

- `size` 字段存储了类型占用的内存空间，为内存空间的分配提供信息。
- `hash` 字段能够帮助我们快速确定类型是否相等。

## 非空接口的结构

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

`iface` 内部维护两个指针：

- `tab` 指向一个 `itab` 实体， 它表示接口的类型以及赋给这个接口的实体类型。
- `data` 则指向接口具体的值，一般而言是一个指向堆内存的指针。

```go
type itab struct {
	inter  *interfacetype
	_type  *_type
	link   *itab
	hash   uint32 // copy of _type.hash. Used for type switches.
	bad    bool   // type does not implement interface
	inhash bool   // has this itab been added to hash?
	unused [2]byte
	fun    [1]uintptr // variable sized
}
```

- `inter` 它描述的是接口的类型。
- `_type` 同上文空接口中的 `_type` 字段。
- `hash` 同上文中空接口中的 `hash` 字段。
- `fun` 是一个动态大小的数组，存储了一组函数指针。虽然该变量被声明成大小固定的数组，但是在使用时会通过原始指针获取其中的数据，所以 `fun` 数组中保存的元素数量是不确定的。

重点看一下 `interfacetype` 类型，它描述的是接口的类型，可以看到，它包装了 `_type` 类型，`_type` 实际上是描述 Go 语言中各种数据类型的结构体。

```go
type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}
```

![image.png](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/20230314115702.png)

## 接口的动态类型和动态值

从源码里可以看到：`iface` 包含两个字段：`tab` 是接口表指针，指向类型信息；`data` 是数据指针，则指向具体的数据。它们分别被称为 `动态类型` 和 `动态值`。

```go
package main

import "fmt"

type Coder interface {
	code()
}

type Gopher struct {
	name string
}

func (g Gopher) code() {
	fmt.Printf("%s is coding\n", g.name)
}

func main() {
	var c Coder
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)

	var g *Gopher
	fmt.Println(g == nil)

	c = g
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)
}
```

**输出**

```
true
c: <nil>, <nil>
true
false
c: *main.Gopher, <nil>
```

实际上对一个接口进行多次赋值的时候，实际进行修改的内容，也即动态类型与动态值这两个内容。

## 接口断言

### 非空接口

类型断言时会将目标类型的 `hash` 与接口变量中的 `itab.hash` 进行比较。

### 空接口

从 `eface._type` 中获取类型，仍然会使用 `_type.hash` 与变量的类型比较。

## 接口转换原理

从 `iface` 的源码可以看到，实际上它包含接口的类型 `interfacetype` 和 实体类型的类型 `_type`，也就是说生成一个 `itab` 同时需要接口的类型和实体的类型。

当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。例如某类型有 `m` 个方法，某接口有 `n` 个方法，则很容易知道这种判定的时间复杂度为 `O(mn)`，Go 会对方法集的函数按照函数名的字典序进行排序，所以实际的时间复杂度为 `O(m+n)`。

## 动态派发

动态派发（Dynamic dispatch）是在运行期间选择具体多态操作（方法或者函数）执行的过程，它是面向对象语言中的常见特性。Go 语言虽然不是严格意义上的面向对象语言，但是接口的引入为它带来了动态派发这一特性，调用接口类型的方法时，如果编译期间不能确认接口的类型，Go 语言会在运行期间决定具体调用该方法的哪个实现。

```go
func main() {
	var c Duck = &Cat{Name: "draven"}
	c.Quack()
	c.(*Cat).Quack()
}
```

1. 第一次以 `Duck` 接口类型的身份调用，调用时需要经过运行时的动态派发。
2. 第二次以 `*Cat` 具体类型的身份调用，编译期就会确定调用的函数。

# 参考

- [深度解密 Go 语言之关于 interface 的 10 个问题](https://qcrao.com/post/dive-into-go-interface/)
- [Go 语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-interface/#42-%E6%8E%A5%E5%8F%A3)

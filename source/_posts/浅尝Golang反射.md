---
title: 浅尝Golang反射
author: siegelion
date: 2021/8/23 10:44
tags: [Go,后端]
categories: [笔记]
---

# 0. 前言

对反射的尝试来源于项目中某个`CRDU`功能的开发，当时我使用`Go`做一个`WEB`项目的后端开发，数据库选用的是`MongoDB`，功能其实也非常的简单，查询数据库某个表中的一条数据，并将得到的数据返回。使用伪代码描述逻辑即为：

```go
type A struct{
    ...
}
func query_data_A(...) []A {
    var data_A A
    var list_A []A
    cursor, err := database.Collection("data_A").Find()
    for cursor.Next(context.TODO()) {
		err := cursor.Decode(data_A)
		list_A.Append(data_A)
	}
    
    return list_A
}
```

这个代码确实很简单，当时也没多考虑。当随着项目的推进，出现了大量逻辑相似的代码，他们的唯一不同之处在于：每个函数需要返回的**数据类型**不同。随着相似代码的多次出现，我实在忍无可忍，开始思考是否有一种办法可以将其中相似逻辑的代码进行复用，提高代码的利用率。

我第一个想到的办法就是**泛型**，但是`Go`并没有为开发者提供泛型的功能，所以这条路堵死了。于是我想到了**空接口**，任何类型都实现了空接口，虽然每个函相似数返回的都是不同数据的数组，但只要将返回值的类型改为空接口，就可以解决返回值的问题了。

但是这样并没有解决全部的问题，因为在所有代码中有一步是必须的：

```go
var data_A A // 声明A类型的变量
err := cursor.Decode(data_A) //将数据解析到该类型变量上
```

如何根据传参声明不同类型的数据（`data_A`）和不同类型的数组（`list_A`）,是我必须解决的。这个问题一时毫无头绪，不知该如何解决。所以我决定问一问`BBFat`同学的意见，他立刻想到：我的这个需求需要在运行时，动态的修改程序（修改程序中声明的变量类型），无疑这就是**反射**这个特性应该解决的事情，所以我的这个需求无疑是使用**反射**进行解决。最终经过`BBfat`的帮助和反复实验，给出了如下的代码：

```go
func find(infoListPtr interface{}, cursor *mongo.Cursor) {
	infoListValue := reflect.ValueOf(infoListPtr).Elem()                // 数组 Value
	infoListType := infoListValue.Type()                                // 数组 Type
	infoELemType := infoListType.Elem()                                 // 数组元素 Type
	infoSlice := reflect.MakeSlice(reflect.SliceOf(infoELemType), 0, 0) // 声明同上文数组 Type 的切片

	for cursor.Next(context.TODO()) {
		newInfo := reflect.New(infoELemType)      // 声明同上文数组元素 Type 的变量
		err := cursor.Decode(newInfo.Interface()) // 游标解析
		if err != nil {
			log.Fatalln(err)
			return
		}
		infoSlice = reflect.Append(infoSlice, newInfo.Elem()) // 将解析出的元素加入切片
	}

	infoListValue.Set(infoSlice) // 将切片设置到数组上
}
```

# 1. 反射 `reflect` 包

- 官方文档：[Package reflect](https://golang.google.cn/pkg/reflect/)
- 社区文档：[中文翻译](https://studygolang.com/pkgdoc)

## 1.1. `reflect.Type`

`Type`是一个接口类型，用来表示一个go类型，该接口拥有以下一系列方法：

```go
type Type interface {
    // Kind返回该接口的具体分类
    Kind() Kind
    // Name返回该类型在自身包内的类型名，如果是未命名类型会返回""
    Name() string
    // PkgPath返回类型的包路径，即明确指定包的import路径，如"encoding/base64"
    // 如果类型为内建类型(string, error)或未命名类型(*T, struct{}, []int)，会返回""
    PkgPath() string
    // 返回类型的字符串表示。该字符串可能会使用短包名（如用base64代替"encoding/base64"）
    // 也不保证每个类型的字符串表示不同。如果要比较两个类型是否相等，请直接用Type类型比较。
    String() string
    // 返回要保存一个该类型的值需要多少字节；类似unsafe.Sizeof
    Size() uintptr
    // 返回当从内存中申请一个该类型值时，会对齐的字节数
    Align() int
    // 返回当该类型作为结构体的字段时，会对齐的字节数
    FieldAlign() int
    // 如果该类型实现了u代表的接口，会返回真
    Implements(u Type) bool
    // 如果该类型的值可以直接赋值给u代表的类型，返回真
    AssignableTo(u Type) bool
    // 如该类型的值可以转换为u代表的类型，返回真
    ConvertibleTo(u Type) bool
    // 返回该类型的字位数。如果该类型的Kind不是Int、Uint、Float或Complex，会panic
    Bits() int
    // 返回array类型的长度，如非数组类型将panic
    Len() int
    // 返回该类型的元素类型，如果该类型的Kind不是Array、Chan、Map、Ptr或Slice，会panic
    Elem() Type
    // 返回map类型的键的类型。如非映射类型将panic
    Key() Type
    // 返回一个channel类型的方向，如非通道类型将会panic
    ChanDir() ChanDir
    // 返回struct类型的字段数（匿名字段算作一个字段），如非结构体类型将panic
    NumField() int
    // 返回struct类型的第i个字段的类型，如非结构体或者i不在[0, NumField())内将会panic
    Field(i int) StructField
    // 返回索引序列指定的嵌套字段的类型，
    // 等价于用索引中每个值链式调用本方法，如非结构体将会panic
    FieldByIndex(index []int) StructField
    // 返回该类型名为name的字段（会查找匿名字段及其子字段），
    // 布尔值说明是否找到，如非结构体将panic
    FieldByName(name string) (StructField, bool)
    // 返回该类型第一个字段名满足函数match的字段，布尔值说明是否找到，如非结构体将会panic
    FieldByNameFunc(match func(string) bool) (StructField, bool)
    // 如果函数类型的最后一个输入参数是"..."形式的参数，IsVariadic返回真
    // 如果这样，t.In(t.NumIn() - 1)返回参数的隐式的实际类型（声明类型的切片）
    // 如非函数类型将panic
    IsVariadic() bool
    // 返回func类型的参数个数，如果不是函数，将会panic
    NumIn() int
    // 返回func类型的第i个参数的类型，如非函数或者i不在[0, NumIn())内将会panic
    In(i int) Type
    // 返回func类型的返回值个数，如果不是函数，将会panic
    NumOut() int
    // 返回func类型的第i个返回值的类型，如非函数或者i不在[0, NumOut())内将会panic
    Out(i int) Type
    // 返回该类型的方法集中方法的数目
    // 匿名字段的方法会被计算；主体类型的方法会屏蔽匿名字段的同名方法；
    // 匿名字段导致的歧义方法会滤除
    NumMethod() int
    // 返回该类型方法集中的第i个方法，i不在[0, NumMethod())范围内时，将导致panic
    // 对非接口类型T或*T，返回值的Type字段和Func字段描述方法的未绑定函数状态
    // 对接口类型，返回值的Type字段描述方法的签名，Func字段为nil
    Method(int) Method
    // 根据方法名返回该类型方法集中的方法，使用一个布尔值说明是否发现该方法
    // 对非接口类型T或*T，返回值的Type字段和Func字段描述方法的未绑定函数状态
    // 对接口类型，返回值的Type字段描述方法的签名，Func字段为nil
    MethodByName(string) (Method, bool)
    // 内含隐藏或非导出方法
}
```

### 1.1.1. `kind`方法解析

其他方法都比较好理解，其中容易存在歧义的是`Kind`方法，该方法返回的是变量的基础类型，何为基础类型请参见下面的例子：

```go
package main

import (
	"fmt"
	"reflect"
)

type person struct {
	age  int
	name string
}

func main() {
	var someone person = person{age: 22, name: "han"}

	varType := reflect.TypeOf(someone)
	fmt.Println(varType.Name()) // person
	fmt.Println(varType.Kind()) // struct
}
```

`Go`中可以定义千万种数据类型，但是`Go`中定义了26中基础类型，无论是何种利用`struct`定义的类型，他们的基础类型都是`struct`，`Kind`方法的返回值也即是这26中基础类型之一：

```go
const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    Int16
    Int32
    Int64
    Uint
    Uint8
    Uint16
    Uint32
    Uint64
    Uintptr
    Float32
    Float64
    Complex64
    Complex128
    Array
    Chan
    Func
    Interface
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```

### 1.1.2. 常用方法

关于`reflect.Type`的方法，一般都是只读方法，所以很少使用（我认为的....）。

相反，另外一些方法则经常用到：

```go
func TypeOf(i interface{}) Type
// TypeOf返回接口中保存的值的类型，TypeOf(nil)会返回nil
func PtrTo(t Type) Type
// PtrTo返回类型t的指针的类型
func SliceOf(t Type) Type
// SliceOf返回类型t的切片的类型
func MapOf(key, elem Type) Type
// MapOf返回一个键类型为key，值类型为elem的映射类型。如果key代表的类型不是合法的映射键类型（即它未实现go的==操作符），本函数会panic
func ChanOf(dir ChanDir, t Type) Type
// ChanOf返回元素类型为t、方向为dir的通道类型。运行时GC强制将通道的元素类型的大小限定为64kb。如果t的尺寸大于或等于该限制，本函数将会panic
```

其中`reflect.TypeOf`接受一个空接口类型，此时根据传入的实参变量，`Typeof`方法的返回值可分为两种情况：

- 如果是一个绑定了实例的接口变量或者是一个实例变量，则返回实例的`Type`
- 如果是一个未绑定实例的空接口，则返回接口`interface`的`Type`

## 1.2. `reflect.Value`

`reflect.Value`是`Go`反射的核心，他保存着实例值的信息，因为相比于`reflect.Type`他提供了更多“写”方法，使用其才可以做到在运行时动态地修改程序。`reflect.Value`是一个`struct`并且向开发者提供了丰富的方法：

```go
func ValueOf(i interface{}) Value
// 返回接口绑定实例的Value值
func Zero(typ Type) Value
// Zero返回一个持有类型typ的零值的Value。注意持有零值的Value和Value零值是两回事。Value零值表示不持有任何值。例如Zero(TypeOf(42))返回一个Kind为Int、值为0的Value。Zero的返回值不能设置也不会寻址。
func New(typ Type) Value
// New返回一个Value类型值，该值持有一个指向类型为typ的新申请的零值的指针，返回值的Type为PtrTo(typ)。
func NewAt(typ Type, p unsafe.Pointer) Value
// NewAt返回一个Value类型值，该值持有一个指向类型为typ、地址为p的值的指针。
func Indirect(v Value) Value
// 返回持有v持有的指针指向的值的Value。如果v持有nil指针，会返回Value零值；如果v不持有指针，会返回v。
func MakeSlice(typ Type, len, cap int) Value
// MakeSlice创建一个新申请的元素类型为typ，长度len容量cap的切片类型的Value值。
func MakeMap(typ Type) Value
// MakeMap创建一个特定映射类型的Value值。
func MakeChan(typ Type, buffer int) Value
// MakeChan创建一个元素类型为typ、有buffer个缓存的通道类型的Value值。
func (v Value) Kind() Kind
// Kind返回v持有的值的分类，如果v是Value零值，返回值为Invalid
func (v Value) Type() Type
// 返回v持有的值的类型的Type表示。
func (v Value) Convert(t Type) Value
// Convert将v持有的值转换为类型为t的值，并返回该值的Value封装。如果go转换规则不支持这种转换，会panic。
func (v Value) Elem() Value
// Elem返回v持有的接口保管的值的Value封装，或者v持有的指针指向的值的Value封装。如果v的Kind不是Interface或Ptr会panic；如果v持有的值为nil，会返回Value零值。
func (v Value) NumMethod() int
// 返回v持有值的方法集的方法数目。
func (v Value) Method(i int) Value
// 返回v持有值类型的第i个方法的已绑定（到v的持有值的）状态的函数形式的Value封装。返回值调用Call方法时不应包含接收者；返回值持有的函数总是使用v的持有者作为接收者（即第一个参数）。如果i出界，或者v的持有值是接口类型的零值（nil），会panic。
func (v Value) MethodByName(name string) Value
// 返回v的名为name的方法的已绑定（到v的持有值的）状态的函数形式的Value封装。返回值调用Call方法时不应包含接收者；返回值持有的函数总是使用v的持有者作为接收者（即第一个参数）。如果未找到该方法，会返回一个Value零值。
func (v Value) CanAddr() bool
// 否可以获取v持有值的指针。可以获取指针的值被称为可寻址的。如果一个值是切片或可寻址数组的元素、可寻址结构体的字段、或从指针解引用得到的，该值即为可寻址的。
func (v Value) Addr() Value
// 函数返回一个持有指向v持有者的指针的Value封装。如果v.CanAddr()返回假，调用本方法会panic。Addr一般用于获取结构体字段的指针或者切片的元素（的Value封装）以便调用需要指针类型接收者的方法。
func (v Value) UnsafeAddr() uintptr
// 返回指向v持有数据的地址的指针（表示为uintptr）以用作高级用途，如果v不可寻址会panic。
func (v Value) CanInterface() bool
// 如果CanInterface返回真，v可以不导致panic的调用Interface方法。
func (v Value) Interface() (i interface{})
// 本方法返回v当前持有的值（表示为/保管在interface{}类型）。
func (v Value) CanSet() bool
// 如果v持有的值可以被修改，CanSet就会返回真。只有一个Value持有值可以被寻址同时又不是来自非导出字段时，它才可以被修改。如果CanSet返回假，调用Set或任何限定类型的设置函数（如SetBool、SetInt64）都会panic。
func (v Value) SetMapIndex(key, val Value)
// 用来给v的映射类型持有值添加/修改键值对，如果val是Value零值，则是删除键值对。如果v的Kind不是Map，或者v的持有值是nil，将会panic。key的持有值必须可以直接赋值给v持有值类型的键类型。val的持有值必须可以直接赋值给v持有值类型的值类型。
func (v Value) Set(x Value)
// 将v的持有值修改为x的持有值。如果v.CanSet()返回假，会panic。x的持有值必须能直接赋给v持有值的类型。
```

### 1.2.1. 注意事项

- `MakeSlice`与`SliceOf`搭配使用
- `MakeMap`与`MapOf`搭配使用
- `MakeChan`与`ChanOf`搭配使用
- `Elem`与`Indirect`都可以将指针类型的`Value`值转化为指针指向类型的`Value`值，但`Elem`会导致`panic`
- `Interface`方法可以将`Value`值对应的实例绑定到空接口。
- 使用`Set`方法，可以实例对应的`Value`值改变，进而改变实例值。

# 2. 反射规则

## 2.1. 转化规则

- 实例可以与`Value`相互转换

    - 实例到`Value`

    ```go
    func ValueOf(i interface{}) Value
    ```

    - `Value`到实例

    ```go
    func (v Value) Interface() (i interface{})
    ```

    或

    ```go
    func (v Value) CanSet() bool
    func (v Value) Set(x Value)
    ```

    参照下面的例子：

    ```go
    
    package main
    
    import (
    	"fmt"
    	"reflect"
    )
    
    type person struct {
    	age  int
    	Name string
    }
    
    func main() {
    	var someone person = person{age: 22, Name: "han"}
    
    	val := reflect.ValueOf(&someone) // 将指向实例的指针绑定到接口上传入ValueOf方法,因为Go中的函数传参都是值拷贝，因此这时接口绑定的是指针值的拷贝
        fmt.Println(val.CanSet()) // false 作为副本的指针值的拷贝，是不能进行Set的，因为是在副本上进行操作
        
        val = val.Elem() // 将指针的Value转换为指针指向的实例的Value
    	fmt.Println(val.CanSet()) // true 这时是在原实例上进行操作，因此可以进行Set
    
    	if val.CanSet() {
    
    		fmt.Println(val.FieldByName("age").CanSet()) // false struct中以小写名字开头的字段，外部是不可见的，因此无法Set
    		fmt.Println(val.FieldByName("Name").CanSet()) // true
    
    		newName := reflect.ValueOf("meng")
    		val.FieldByName("Name").Set(newName)
    	}
    
    	fmt.Println(someone) // {22 meng}
    
    }
    ```

- 实例可以与`Type`相互转换

    - 实例到`Type`

    ```go
    func TypeOf(i interface{}) Type
    ```

    - `Type`到实例

    ```go
    func New(typ Type) Value
    func (v Value) Interface() (i interface{})
    ```

- `Value`可以与`Type`相互转换

    - `Value`到`Type`

    ```go
    func (v Value) Type() Type
    ```

    - `Type`到`Value`

    ```go
    func New(typ Type) Value
    ```

- 指针类型的`Value`可以与值类型的`Value`相互转换

    - 指针类型的`Value`到值类型的`Value`

    ```go
    func (v Value) Elem() Value
    ```

    或

    ```go
    func Indirect(v Value) Value
    ```

    - 值类型的`Value`到指针类型的`Value`

    ```go
    func (v Value) Addr() Value
    ```

- 指针类型的`Type`可以与值类型的`Type`相互转换

    - 指针类型的`Type`到值类型的`Type`

    ```go
    func (t Type) Elem() Type
    ```

    - 值类型的`Type`到指针类型的`Type`

    ```go
    func (t Type) PtrTo() Type
    ```


![image-20210822230451707](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20210822230451707.png)

## 2.2. 反射三定律

1. 反射可以从接口值得到反射对象
2. 反射可以从反射对象得到接口值
3. 若要修改一个反射对象，则其值必须可以修改
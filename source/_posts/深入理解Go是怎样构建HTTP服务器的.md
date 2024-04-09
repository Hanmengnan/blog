---
title:  深入理解Go是怎样构建HTTP服务器的
author: siegelion
tags:  [Go]
categories: [笔记]
date: 2022/5/08 14:00:25
---



实际上我很早就看过《Go Web编程》这本书，其中的一章很详细地介绍了Go中构建一个最简单的服务器的方法， Go在标准库中为我们提供了一个`net/http`包，这个包中提供了完善的功能来帮助我们构建一个Web服务器，Go中很多Web框架的底层实际上也是借助了这个标准库来实现自己的功能。

当时看完这本书后，觉得自己已经掌握了相关的知识，但在前两天打算写一个最简单的服务器打包成`Docker`镜像来进行`Kubernetes`的学习的时候，却在这里犯了难，所以打算重新读一下相关知识，在这里也记录一下。

## 一个最简单的HTTP服务器

```go
import "net/http"

func RunServer() {
	http.ListenAndServe("0.0.0.0:8080", nil)
}

```

这个服务器调用`http`包中的`ListenAndServe`函数，监听8080端口，但不做任何处理。                                                                                                                                            

我们看一下这个函数的源码：

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

实际上是创建了一个`Server`类型的结构体，然后调用了其的`ListenAndServe`方法。那么我们可以将我们的服务器代码改为：

```go
import "net/http"

func RunServer() {
	server := &http.Server{
		Addr:    "0.0.0.0:8080",
		Handler: nil,
	}
	server.ListenAndServe()
}

```

实际上起到的效果是一样的。

## 处理器

接下来让我们关注一下`http.ListenAndServe`函数的第二个参数，这个参数是一个`Handler`类型，这个类型是一个接口类型，接口的定义如下：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

这个接口只有一个方法，也即为`ServeHTTP`。因此我们如果传入的这个参数，只要是一个 拥有该方法的处理器类型即可。我们可以将代码再稍作修改：

```go
import "net/http"

type MyHandler struct{}

func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

func RunServer() {
	handler := &MyHandler{}
	server := &http.Server{
		Addr:    "0.0.0.0:8080",
		Handler: handler,
	}
	server.ListenAndServe()
}

```

## 多路复用器

但这是有一个问题，我们现在访问8080端口的任何URL返回的结果都是一样的，显然我们不希望如此，那么是否可以像其他Web框架一样，根据不同的路由进行不同的处理呢？答案当然是可以的，这时就需要引入我们的多路复用器，我们来看一个最简单的多路复用器：

```go
import "net/http"

type HelloHandler struct{}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

type HiHandler struct{}

func (h *HiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

func RunServer() {
	helloHandler := &HelloHandler{}
	hiHandler := &HiHandler{}

	server := &http.Server{
		Addr: "0.0.0.0:8080",
	}
	http.Handle("/hello", helloHandler)
	http.Handle("/hi", hiHandler)
	
	server.ListenAndServe()
}
```

我们使用了`http`包中的`Handle`函数，将不同的路由绑定到不同的处理器上，实现了我们的需求。我们对这个函数也很好奇，因此我们可以来看一下这个函数的源码：

```go
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }
```

这个函数的源代码很简单，只有一行，也即调用了`DefaultServeMux`变量的`Handle`方法，这个变量实际上是一个`ServeMux`类型的变量，我们查看一下这个变量的初始化过程发现，这个变量仅仅是一个利用默认值初始化后的`ServeMux`变量，那么我们有理由认为，我们也初始化一个这样的变量，然后调用这个变量的Handle方法也能起到一样的效果：

```go
import "net/http"

type HelloHandler struct{}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

type HiHandler struct{}

func (h *HiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

func RunServer() {
	severMux := http.NewServeMux()
	helloHandler := &HelloHandler{}
	hiHandler := &HiHandler{}

	server := &http.Server{
		Addr: "0.0.0.0:8080",
	}
	severMux.Handle("/hello", helloHandler)
	severMux.Handle("/hi", hiHandler)

	server.ListenAndServe()
}
```

事实上确实如此。

此外我们还从源码中发现一件有趣的事，那就是`ServeMux`类型也有一个`ServeHTTP`方法，根据一开始的例子，我们知道`http.Server`类型的`Handler`字段是一个接口，要将变量赋给这个字段，必须拥有一个`ServeHTTP`方法，这样这个变量才能成为一个处理器，也就是说我们可以直接将一个多路复用器当作一个处理器来使用：

```go
import "net/http"

func RunServer() {
	severMux := http.NewServeMux()

	server := &http.Server{
		Addr: "0.0.0.0:8080",
		// Handler: severMux,
		Handler: http.DefaultServeMux,
	}

	server.ListenAndServe()
}

```

使用这样的方式启动服务器后，我们访问8080端口，会发现虽然不会报错，但是会返回404状态。这是为什么呢？

我们关注一下`ServeMux`类型的`ServeHTTP`方法，这个方法的调用链有点深，我们一点一点看：

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	...
    h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

这里又出现了一个`ServeHTTP`函数，我们先跳过其看上面的`Handler`函数。

```go
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	...
	return mux.handler(host, r.URL.Path)
}
```

这个函数在末尾调用了一下`handler`函数。

```go
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}
```

`handler`函数中多次调用了一个`match`函数，这个`match`函数的作用就是根据`host`和`path`两个变量找到对应的处理器然后一层层返回给上层，这个函数的最后有一个`NotFoundHandler`函数，生成一个找不到相应处理器时的兜底情况。

这也就解释了为什么我们访问8080端口时返回404，因为我们使用`DefaultServeMux`多路复用器作为处理器，实际上这个多路复用器并没有绑定任何处理器，处理请求时只能返回404了。

## 多路复用器的匹配原则

我们看一下match函数的源码：

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}

```

根据源码我们可以发现，要匹配多路复用器中的处理器，实际上是通过多路复用器的map一个字典去进行匹配，如果字典中找不到对应的处理器，那么再到一个以”/“结尾的路由切片中寻找，当前的路由是否是任何路由的前缀，如果是那么也可以认为匹配。

## 处理器函数

除了可以将处理器在多路复用器上绑定路由外，还可以使用处理器函数来绑定路由：

```go
import "net/http"

func sayHello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}
func RunServer() {
	severMux := http.NewServeMux()
	severMux.HandleFunc("/", sayHello)
    
	server := &http.Server{
		Addr:    "0.0.0.0:8080",
		Handler: severMux,
	}

	server.ListenAndServe()
}

```

我们也来看一下这个函数的源码：

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

这个函数将我们传入的处理器函数转为`HandlerFunc`类型，没错是一个类型转换，然后将这个类型作为第二个参数调用`mux.Handle`方法，这也就和处理器的调用一致了，但一个处理器是一个拥有`SeveHTTP`的变量，这里是怎么实现转换的呢？

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

```

这里的转换简直让人叹为观止！`HandlerFunc`这个类型拥有一个`ServeHTTP`方法，这个方法的实现就是调用`HandlerFunc`自身。这样就可以实现，处理器函数到一个处理器的转化。






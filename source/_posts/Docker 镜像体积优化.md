---
title: Docker 镜像体积优化
author: siegelion
date: 2022/03/15 16:15
tags: [Docker,后端]
categories: [笔记]
---

在我使用`docker`对我目前负责的一个项目进行部署时，遇到了这样的一个问题，使用`dockerfile`构建出的`docker`镜像的体积太大了。当时我只觉得这是不可避免的，毕竟容器的底层还是运行着一个操作系统，加之`Golang`的编译环境，再加上项目的代码以及静态资源，最终构建出的镜像体积为`1.1G`似乎是一件再正常不过的事情。

### Version 1

以下是我第一版用于构建`docker`镜像的`dockerfile`，可以看到为了保证最终容器的兼容性，我使用`1.16`版本的`Golang`镜像作为基础镜像，该镜像的下层为`Ubuntu`镜像。

```dockerfile
# Version: 0.1
FROM golang:1.16

COPY . /$GOPATH/src/Fouda/
WORKDIR /$GOPATH/src/Fouda/

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod tidy
RUN go build ./cmd/main.go

EXPOSE 8089:8089
CMD ["go","run","./cmd/main.go"]
```

![image-20220315155643212](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315155643212.png)

打包之后，查看一下生成的镜像，发现该镜像的大小为`1.1G`，显然达到了一个不能接受的大小，但令人难以置信的是一开始我就将这个镜像推送至`DockerHub`，然后又将这个镜像拉取到部署到服务器上。

### Version 2

由于考虑到，上一个版本中我们是以`Ubuntu`作为底层镜像来进行镜像构建的，但实际上我们并不需要`Ubuntu`中附加的诸多软件和功能，我们仅仅需要一个能够运行代码的环境，那么有没有这样的操作系统镜像呢？`Alpine` 操作系统是一个面向安全的轻型 `Linux` 发行版。它不同于通常 `Linux` 发行版，`Alpine` 采用了 `musl libc` 和 `busybox` 以减小系统的体积和运行时资源消耗。`Alpine Docker`镜像也继承了`Alpine Linux`发行版的这些优势。相比于其他` Docker`镜像，它的容量非常小，仅仅只有`5 MB`左右（对比`Ubuntu`系列镜像接近`200 MB`），且拥有非常友好的包管理机制。官方镜像来自 `docker-alpine` 项目。为了减小`docker`镜像的体积，这次使用了`golang:alpine`作为基础镜像进行构建，其中移除了非必要的工具，只提供最基础的功能，因此相比我们第一个版本使用的Ubuntu镜像可以节省很多的体积。

```dockerfile
# Version: 0.2
FROM golang:alpine

COPY . /$GOPATH/src/Fouda/
WORKDIR /$GOPATH/src/Fouda/

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod tidy
RUN go build ./cmd/main.go

EXPOSE 8089:8089
CMD ["go","run","./cmd/main.go"]
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315160140580.png)

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315162339116.png)

如我们所预料的那样，使用这个方法改进后的镜像大小，下降了`400M`左右，但是`700M`还是让人无法接受，我们使用命令查看一下镜像构建时每层的大小。可以看到在`LABEL`命令执行前，文件的体积已经很大了。这是因为我们还是依赖`Golang`的编译环境，并且在编译`Golang`时还要通过网络拉取很多的依赖库，导致最终的镜像体积爆炸。

### Version 3

我们着手想办法继续减小构建出的镜像体积。我了解到一种分阶段构建的方法，也即先使用一个拥有编译环境的镜像对代码进行编译，然后将编译后的代码拷贝至一个新的`alpine`镜像中运行我们编译后的二进制文件。

```dockerfile
# Version: 0.3
FROM golang:alpine as compiler

WORKDIR /Fouda/
COPY . /Fouda/

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod tidy
RUN go build -o fouda ./cmd/main.go

FROM alpine

WORKDIR /fouda
COPY  --from=compiler /Fouda/fouda /fouda/fouda

EXPOSE 8089:8089
CMD ["./fouda"]
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315163227069.png)

通过这个方法，我们发现镜像的大小大幅下降，下降至30M。

### Version 4

到这里镜像的大小已经大幅度压缩了，还有什么办法可以进一步减小镜像的大小吗？办法还真有！`Alpine`相比于`Ubuntu`是一个大小足够小的操作系统镜像，但是其中还是携带了一部分的包管理工具，这些包管理工具对我们来说还是多余的，能否将他们也去掉呢？

这时候就需要用到`scratch`镜像，官方说明：该镜像是一个空的镜像，可以用作构建`docker`镜像的最小基础镜像使用，并且在其上可以运行没有依赖的二进制文件。这个镜像无疑是最满足我们需求的。

```dockerfile
# Version: 0.4
FROM golang:alpine as compiler

WORKDIR /Fouda/
COPY . /Fouda/

ENV CGO_ENABLED=0

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod tidy
RUN go build -o fouda ./cmd/main.go

FROM scratch

WORKDIR /fouda
COPY  --from=compiler /Fouda/fouda /fouda/fouda

EXPOSE 8089:8089
CMD ["./fouda"]
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315163741050.png)

构建完成，镜像的体积又一次减少了`7M`。

### Version 5

我们再过分一点，在`go build`阶段添加参数，去掉了调试信息以减小镜像尺寸，并且禁用`cgo`。

```dockerfile
# Version: 0.5
FROM golang:alpine as compiler

WORKDIR /Fouda/
COPY . /Fouda/

ENV CGO_ENABLED=0

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod tidy
RUN go build -ldflags="-s -w" -o fouda ./cmd/main.go

FROM scratch

WORKDIR /fouda
COPY  --from=compiler /Fouda/fouda /fouda/fouda

EXPOSE 8089:8089
CMD ["./fouda"]
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220315163801953.png)

镜像的体积又稍微减少了一些，至此我停止继续折腾它了。


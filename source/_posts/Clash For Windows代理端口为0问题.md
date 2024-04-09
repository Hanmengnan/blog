---
title:问题
author: siegelion
date: 2022/01/31 10:15
tags: [瞎折腾,网络]
published: false
categories: [笔记]
---

## 问题

之前的正常使用中，我的`Clash`的代理端口一直都是被设置为`7890`，但是在一次系统更新后，我发现不光是以往需要代理才能访问的网站，甚至是平时不需要代理就可以正常访问的网站，全部无法访问了，并且原因是由于`Clash`造成的。

![clash1](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/clash1.png)

在排除了`Clash`代理的配置文件是否过期和机场提供商是否跑路的问题后，我发现了一个严重的问题，我的代理端口变为了`0`，`0`号端口是一个非法端口，使用这个端口进行转发，毫无疑问会导致流量都流进了一个黑洞，也就导致了设备无法正常访问网络。

![clash2](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/clash2.png)

通过查看`Clash`日志，可以从日志记录中发现这样一条日志：

```powershell
time="2022-01-31T09:40:20+08:00" level=error msg="Start Mixed(http and socks5) server error: listen tcp :7890: bind: An attempt was made to access a socket in a way forbidden by its access permissions."
```

大致是说在进行端口绑定是出现了问题，无法在7890端口上建立`socket`连接。

## 解决

网络上大家对于这个问题出现的原因分析，大多是说系统更新后，由于使用了`Hyper-V`的功能，虚拟机需要映射一部分端口，并且在系统更新后对动态映射的端口范围进行了更改，导致占用了本来的`Clash`使用的端口。所以为了解决这个问题，

- 要么直接关闭`Hyper-V`。（怎么可能，`WSL`不用了？）
- 要么修改系统动态映射的端口范围。（我觉得还是不要动系统层面的设置，免得又出问题。）
- 要么对`Clash`使用的代理端口进行修改，改为不被系统占用的端口即可。（Nice！）

使用命令查看被系统排除在可用范围外的端口：

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220131102147660.png)

可以发现我们原本的`7890`正好处于`7872-7971`这个被排除的范围中，所以我们只要将代理端口改为范围外的任意一个即可。我这里选用的是7980端口。

![](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220131102338599.png)

已经可以正常访问互联网了。

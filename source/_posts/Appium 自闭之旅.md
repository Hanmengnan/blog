title: Appium 自闭之旅
tags:
  - 软件
categories: [杂]
author: siegelion
date: 2019-09-28 11:52:00
---

---
该篇文章主要针对于AppiumDesktop，AppiumServer最后一版在2016年已经停更了，随后推出的Desktop配置起来更加方便，更加易用，故此推荐Desktop。
注意：Desktop和Server的安装步骤相差较大，而搜索Appium得到的结果大多数是Server的，不要被误导了。
## 安装准备:
1. JAVA JDK: 注意下载最新版，不然后续步骤会提示你JDK版本过低
[JAVA SDK](https://www.oracle.com/technetwork/java/javase/overview/index.html)
- 下载安装之后，需要添加环境变量：
- [![先增加一个环境变量：JAVA_HOME](https://i.loli.net/2019/05/09/5cd436cbc10c3.png "先增加一个环境变量：JAVA_HOME")](https://i.loli.net/2019/05/09/5cd436cbc10c3.png "先增加一个环境变量：JAVA_HOME")
- [![再在Path中加入变量](https://i.loli.net/2019/05/09/5cd4370c0e6f6.png "再在Path中加入变量")](https://i.loli.net/2019/05/09/5cd4370c0e6f6.png "再在Path中加入变量")
2. [Android SDK](http://tools.android-studio.org/index.php/sdk "Android SDK")
- 同样需要配置环境变量
- [![](https://i.loli.net/2019/05/09/5cd437aebcef1.png)](https://i.loli.net/2019/05/09/5cd437aebcef1.png)
- [![](https://i.loli.net/2019/05/09/5cd4379d6af5f.png)](https://i.loli.net/2019/05/09/5cd4379d6af5f.png)

3. 在SDK安装目录下找到adb.exe ，使用命令行运行adb.exe
`./adb.exe devices ` 连接设备
1. 打开要自动化测试的APP，使用命令获取，应用包名和活动名，
`.\adb.exe shell dumpsys window | findstr mCurrentFocus`
1. 依照
- [该篇博客的内容，对所需功能json内容进行配置](https://blog.csdn.net/chenmozhe22/article/details/80527209 "该篇博客的内容，对所需功能json内容进行配置")

- [Appium 官方配置文档](https://github.com/appium/appium/blob/master/docs/en/writing-running-appium/caps.md "Appium 官方配置文档")

[![](https://i.loli.net/2019/05/09/5cd439a473b60.png)](https://i.loli.net/2019/05/09/5cd439a473b60.png)

6. 点击“启动服务”开始运行

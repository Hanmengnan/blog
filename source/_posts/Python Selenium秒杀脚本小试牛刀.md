title: Python Selenium秒杀脚本小试牛刀
tags:
  - 爬虫
  - Python
categories:
  - 笔记
author: siegelion
date: 2019-09-19 14:52:00
---

---
初次尝试写一个秒杀脚本是在2018.11月即将迎来双十一的时候，但由于当时没有深入探讨，所以最后不了了之，现在正值假期，我闲来无事，所以打算将这个功能完善一下。


	最近传出消息，美团、淘宝加强对selemium自动化测试软件的封杀，加之个人经验，这个脚本平时使用还好，但如果真的在双十一这样的高峰购物时间点会无法使用，因为平台会出现对下单商品，加以二维码的检测，而在没有模型训练的前提下，很难突破二维码的限制<


1.首先最开始的部分，会看起来有些傻，因为是用selenium拉起浏览器，所以浏览器内部不会保存你的登录cookie，而淘宝登录的cookie是以近10几位字段进行加密的密文，导致如果我们自己想用POST的方式的登录也不可能。所以只能在秒杀开始之前用手机扫码登录，做好准备。😑

[![](https://i.loli.net/2019/02/07/5c5bb9ac89178.png)](https://i.loli.net/2019/02/07/5c5bb9ac89178.png)

2.登录成功后，get方式跳转 "https://cart.taobao.com/cart.htm"（即进入购物车进行准备）

3.外加while死循环，内部加以if判断，用datetime库的datetime.now()函数获取当前时间，返回的是一个字符串

4.对字符串进行比较，如果等于则规定时间则跳转并break，否则打印当前时间并continue。

5.结算按钮的XPATH为 ‘//*[@id="J_Go"]’ 
[![](https://i.loli.net/2019/02/07/5c5bba82072af.png)](https://i.loli.net/2019/02/07/5c5bba82072af.png)
6.跳转到确认订单界面后在用一个死循环，原理同以上类似。

7.提交订单的XPATH为 ‘//*[@class="go-btn"]’
[![](https://i.loli.net/2019/02/07/5c5bba90ba6c2.png)](https://i.loli.net/2019/02/07/5c5bba90ba6c2.png)

下面给出源码：
```python
#2019.2.5测试可用

from selenium import webdriver
import time
import datetime
#拉起浏览器
browser = webdriver.Chrome()
#跳转淘宝首页
browser.get('https://www.taobao.com/')
#手动登录时间
time.sleep(10)
#跳转购物车
browser.get("https://cart.taobao.com/cart.htm")
while True:
    """
    获取当前时间
    在到达秒杀时间之前要手动标记好要下单的商品
    """
    nowtime=datetime.datetime.now().strftime('%H:%M:%S')
    #设置的秒杀时间
    if nowtime=='20:18:00':
        try:
            #提交订单按钮
            browser.find_element_by_xpath('//*[@id="J_Go"]').click()
        finally:
            break
    else:
        print(nowtime)
        continue
while True:
    try:
        #结算按钮
        browser.find_element_by_link_text(u"提交订单").click()
        browser.find_element_by_xpath('//*[@class="go-btn"').click()
        break
    except:
        print("正在提交订单")
        continue
```
PS：

作者在亲自使用过程中，发现了另外一个好用的秒杀脚本，是一个未上架的Chrome浏览器插件使用Javascript编写，可以分别使用Xpath和Jqurey定位，


\[video width="1920" height="1080" mp4="http://39.107.91.78/wp-content/uploads/2019/02/ceb0023485315680b33236ea673416c1-1.mp4"\]\[/video\]
自己觉得比我的更好。

给出Github链接：https://github.com/gongjunhao/seckill

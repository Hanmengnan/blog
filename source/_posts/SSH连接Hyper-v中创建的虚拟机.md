---
title: SSH 连接 Hyper-v 中创建的虚拟机
author: siegelion
date: 2021/12/28 17:15
tags: [瞎折腾,网络]
categories: [笔记]
---

1. 首先新建一个新的**虚拟网络交换机**，交换机的类型选择内部，因为这个交换机我们只是用来进行SSH连接的。

![image-20211228164807403](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228164807403.png)

2. 新建后，可以在控制面板的网络连接面板上，看到你新建出的虚拟网络交换机（实际上是建立了一块虚拟网卡）。

![image-20211228165255558](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228165255558.png)

3. 对虚拟网卡的属性进行配置

    ![image-20211228165540740](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228165540740.png)

    >  **IP地址**和**子网掩码**这里可以参照图中进行配置，也可以自行修改（前提是你确认自己修改的没问题）。

    ![image-20211228165610597](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228165610597.png)

    4. 将虚拟机上新建的虚拟网卡进行配置

        - 进入相应目录

            ```shell
            cd  /etc/sysconfig/network-scripts/
            ```

        - 将原有网卡的配置复制一份命名为**ifcfg-eth1**

            ```shell
            cp ./ifcfg-eth0 ./ifcfg-eth1
            ```

        - 修改网卡1的配置文件

            ```shell
            vim ./ifcfg-eth1 
            ```

            注意几个要点：

            - `BOOTPROTO`改为**static**
            - `NAME`和`DEVICE`改为`eth1`
            - `UUID`要进行注释掉，目的是不让其和网卡0的`UUID`相同
            - `IPADDR`,`NETWMASK`,`GATEWAY`根据在控制面板中的设置进行配置

        ![image-20211228171305905](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228171305905.png)

        - 保存退出

5. 重启机器

6. 在windows上进行SSH连接测试

    ![image-20211228172041126](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20211228172041126.png)
---
title: windows和mac快捷互传文件
tags: FTP
categories: 杂项
cover: false
description: 在win10上搭建FTP服务器
typora-root-url: ..
abbrlink: 7ead17b2
date: 2022-08-03 17:25:59
---



**在windows和mac之间互相传文件一直都很蛋疼,网上大多数教程说的在windows上设置共享文件夹的方法时而好用时而不好用,而且速度感人😅,需要找一个更快捷的互传文件方法**

在windows10上搭建FTP服务器,然后mac直接通过FileZilla等FTP软件来上传和下载就好了(**当然两者要在同一个局域网里**)



# 1安装IIS工具,启动FTP服务

**控制面板-程序和功能-启动或关闭windows功能**

<img src="/images/winftp/image-20220803173147596.png" alt="image-20220803173147596" style="zoom:67%;" />

把FTP下面的选项都点上:

<img src="/images/winftp/image-20220803173214403.png" alt="image-20220803173214403" style="zoom:67%;" />

设置开机启动FTP服务:

`win+r` 运行 `services.msc`:

<img src="/images/winftp/image-20220803173258129.png" alt="image-20220803173258129" style="zoom:67%;" />

找到其中的FTP服务,修改启动类型为自动

<img src="/images/winftp/image-20220803173306856.png" alt="image-20220803173306856" style="zoom:67%;" />



<br>

***

<br>

# 2创建一个用于FTP的用户账户

直接添加一个名为share的账户就好

<img src="/images/winftp/image-20220803173503472.png" alt="image-20220803173503472" style="zoom: 67%;" />

<br>

***

<br>

# 3配置FTP服务器站点

**控制面板-管理工具-Internet Information Services (IIS)管理器**

右键添加FTP站点:

<img src="/images/winftp/image-20220803173725345.png" alt="image-20220803173725345" style="zoom:67%;" />

这里添加一个用于传输文件的目录,用户连接FTP后就可以向这个位置上传,或从这个位置下载:

<img src="/images/winftp/image-20220803173746835.png" alt="image-20220803173746835" style="zoom:67%;" />

下一步,配置本地(FTP服务器)的IP和端口(这里没有使用FTP的默认端口21):

<img src="/images/winftp/image-20220803173834660.png" alt="image-20220803173834660" style="zoom:50%;" />

<img src="/images/winftp/image-20220803173903922.png" alt="image-20220803173903922" style="zoom:50%;" />

<br>

***

<br>

# 4 检查windows防火墙

左下角通过搜索打开 `windows安全中心`

左侧`防火墙和网络保护`--下方`允许应用通过防火墙`,进去之后找到FTP并勾选即可

<img src="/images/winftp/image-20220803174058206.png" alt="image-20220803174058206" style="zoom:67%;" />

<br>

***

<br>

# 5 配置成功

现在可以在mac端使用FileZilla等工具连接并上传下载文件了

<img src="/images/winftp/image-20220803174211731.png" alt="image-20220803174211731" style="zoom:67%;" />

主机的局域网IP需要在windows中通过`ipconfig`命令来查看

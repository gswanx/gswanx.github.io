---
title: 购买VPS和一些初始设置
tags: VPS
categories: VPS
cover: false
description: '记录一下购买VPS和做一些初始设置的过程,补充些知识点'
typora-root-url: ..
password: daddy987
message: 输入密码，查看文章
abbrlink: '4e671577'
date: 2022-08-02 20:20:42
---

<span style='color:black;background:yellow;font-size:25px;font-family:hei'>本文信息密度极低😅仅用于记录过程</span>

<br>

***

# 1补充知识点

## 1.1托管主机和非托管主机

- 托管主机:主机商客服承担VPS服务器的管理服务,包括硬件维护,系统和软件的升级和故障处理等,有时候还包办环境配置,贵...
- 非托管主机:主机客服只提供最基础的VPS配置,环境等等等都需要自己配,主机商只负责维护硬件,相对便宜..(这个时候快照功能就显得特别重要了)

<br>

## 1.2关于VPS的线路

<span id="001">所谓的三网就是指中国大陆主流的 **电信、联通和移动** 三家运营商</span>

[https://blog.laoda.de/archives/vps-route#1.-%E5%89%8D%E8%A8%80](https://blog.laoda.de/archives/vps-route#1.-前言)

 不同的VPS表明其走的线路是其评估速度,稳定性的重要指标*(一般看的是流量回程时走的线路,因为通常下载量是远远大于上传量的)*

 <br>

国内使用不同运营商网络的用户在访问 某VPS时:

回程一般来说先走该VPS接入的线路回国,回国后再从某位置(例如省级)转到用户运行商的线路上

- 例如一个接入**CN2 GIA**线路的VPS,流量回程时先接入电信的**CN2 GIA**网络,回国后再从国级或者省级节点转到客户运营商的网络,例如移动的AS9808(CMI)网络

<br>

>目前国内的 电信,联通,移动 三网拥有的线路主要包括:
>
>电信:
>
>- ChinaNet:163骨干网(AS4134), 中转节点的IP以**202.97**开头,承载普通质量的互联网业务,便宜
>
>- CN2(AS4809):中国电信发展的下一代承载网:(中转节点的IP为**59.43**开头)
>
>  - CN2-GT:半程走CN2,本身接入的还是老的163骨干网(AS4134),也就是去程和回程都会出现202.97开头的节点,出口之后才走CN2(AS4809)
>
>    例如下图,出国前走的是202.97, 出国后走的是59.43
>
>    <img src="/images/vps/image-20220802205243309.png" alt="image-20220802205243309" style="zoom: 33%;" />
>
>  - CN2-GIA:全程CN2,本身接入的网络就是CN2,因此中转节点都是**59.43**开头(也不排除偏远地区先通过163网络202.97接入,再从接入城市并入CN2线路)
>
>    例如下图:全程59.43:
>
>    <img src="/images/vps/image-20220802205406351.png" alt="image-20220802205406351" style="zoom: 33%;" />
>
>联通:
>
>- 联通169网络(AS4837):民用骨干网,定位相当于电信的163网络
>- AS9929 (联通A网, CU PM):定位相当于CN2, 更像CN2 GT, 服务于政企大客户. AS9929的价格比AS4837贵5倍以上. 和电信的GIA差距还很大(炸起来的时候完全没法用)
>- AS10099 (联通国际出口, CUG). 联通出国 可能是AS4837直接出国, 或 内地AS4837/AS9929, 出境接入AS10099
>
>移动:
>
>- CMI (China Mobile International Limited，CMI移动自己的国际出口, AS58453-AS9808) 一般回国有**Qos限速**(可加钱可升级保证国内带宽) 晚高峰可能会炸
>
>- CMCB (移动商宽) 国际出口高Qos等级特权
>
>  <br>
>
>  >- Qos:高峰时段,在路由出海前的最后一跳,根据优先级策略性人为丢包以减轻线路负担(贵的网络会提高这个优先级)
>  >
>  >- AS号码是用来标识自治系统号码的,自治系统之间使用BGP协议互通(计算机网络学过)

<br>

目前来说流量回国三网CN2GIA还是最好用的,不管用户使用的是哪个运营商的网络

关于x网接入,一般**CN2 GIA**的产品分单网/双网/三网:

-  单网一般指的是仅中国电信用户访问去回走 CN2 GIA 网络，联通/移动/教育网等用户访问过去，均走各自运营商的国际出口
-  双网一般指的是中国电信/中国联通用户访问过去，走 CN2 GIA 网络（特别地，该产品允许中国联通线路在省级跨网并入到中国电信的 CN2 网络)
- 三网一般指的是**中国电信/中国联通/中国移动三网用户的去回访问，均会在省级并入到 CN2 网络**，适用性最强,最快（典型产品为搬瓦工的 **DC9、DC6** 机房）,当然也最贵.....

<br>

***

<br>

# 2选购

看了好多视频,还是贪了三网回程GIA的线路,而且因为要做实验,对存储空间有一定的需求,最后一下买了搬瓦工的89刀型号

<https://bwh81.net/cart.php?a=confproduct&i=0>

最大化利用,后面搞完实验再拿来搭搭梯子和建站之类的

LA机房,1核2G,40G硬盘,2000T月流量,折后季度84刀,太痛了😭

<img src="/images/vps/image-20220802211026478.png" alt="image-20220802211026478" style="zoom: 50%;" />

搬瓦工提供的VPS控制面板,还是挺好用的,还有快照功能,能保存主机状态,出了问题直接恢复就好了

<img src="/images/vps/image-20220803003126965.png" alt="image-20220803003126965" style="zoom: 67%;" />



<br>

***

<br>

# 3初始设置

## 3.1重装系统

拿到VPS后发现初始的系统为CentOS,直接通过控制面板左侧的 <span style="color:black;background-color:#FFFF00">**Install new OS**</span>重装为需要的系统版本,这里重装了debian10

在此过程中面板会提醒设置临时的 root用户密码和 ssh连接端口

 <br>

***

<br>

## 3.2基本功能安装和安全性设置



debian下的包管理工具是**apt(Advanced Package Tool)**,常用命令包括:

拿到VPS后先执行<span style="color:black;background-color:#FFFF00">**apt update和apt upgrade**</span>

```shell
apt update   #查询软件更新
apt upgrade  #执行软件更新
apt install '包名'  #安装软件
```

### <span style="color:black;background-color:#FFFF00;font-size:25px">**3.2.1修改root用户的密码**</span>

```shell
passwd root #根据指示输入新密码即可
```

 <br>

### <span style="color:black;background-color:#FFFF00;font-size:25px">**3.2.2修改SSH端口**</span>

(改为非22的更安全一些)

```shell
nano /etc/ssh/sshd_config 
```

修改最下方的Port(尽量大一点的)

<img src="/images/vps/image-20220803004610619.png" alt="image-20220803004610619" style="zoom:67%;" />

>- nano是debian自带的文本编辑器(就当是vim)
>- 常用快捷键:
>  - ctrl+x :退出
>  - ctrl+o---回车---ctrl+x  :保存并退出
>  - ctrl+w :搜索

 <br>

### <span style="color:black;background-color:#FFFF00;font-size:25px">**3.2.3禁用Root用户**</span>

首先添加一个普通用户:

```shell
adduser 用户名 #添加新用户,后续跟着指示设置密码即可
```

安装sudo功能(允许普通用户临时拥有root权限)

```shell
apt update
apt install sudo
```

编辑sudo的配置文件

```shell
visudo #此命令用于打开sudo的配置文件/etc/sudoers.tmp,此文件只能以root用户的身份并使用visudo命令打开
```

添加如红线中的这一行,意思是此用户使用sudo功能时不需要额外输入密码(图个方便)

<img src="/images/vps/image-20220803005243834.png" alt="image-20220803005243834" style="zoom: 67%;" />

接下来再去编辑ssh配置文件

```shell
nano/etc/ssh/sshd_config
```

 修改PermitRootLogin yes 为 no即可,表示不允许以root用户的身份进行ssh连接

 <br>

###  <span style="color:black;background-color:#FFFF00;font-size:25px">**3.2.4更安全一些的方式:密钥登录**</span>

#### 3.2.4.1Windows上的配置:

直接使用xshell的密钥生成器来生成一对密钥,这里如果想要进一步提高安全性,还可以再给此密钥设置密码(当使用密钥登录时会需要输入此密码)

<img src="/images/vps/image-20220803120403346.png" alt="image-20220803120403346" style="zoom:67%;" />

生成了一个公钥和一个私钥,其中公钥需要上传到VPS上,而本机保留私钥作为登录vps的凭证

<img src="/images/vps/image-20220803120527857.png" alt="image-20220803120527857" style="zoom: 67%;" />

将公钥文件通过ftp上传到vps上,并改名为authorized_keys

公钥文件在vps上的存储目录为: **~/.ssh/authorized_keys**

修改VPS的ssh配置,允许使用密钥登录

```
sudo nano /etc/ssh/sshd_config
# 在底部添加 PubkeyAuthentication yes
```

<img src="/images/vps/image-20220803120902313.png" alt="image-20220803120902313" style="zoom:67%;" />



最后在Xshell中设置使用私钥来登录vps

连接属性--用户身份验证--方法选上Public Key,去掉Password--在设置中指定好本地的私钥文件

Public Key设置中的密码填写生成密钥对时设置的那个密码

<img src="/images/vps/image-20220803121723904.png" alt="image-20220803121723904" style="zoom:67%;" />



设置完成后,在Xshell的站点管理器中直接打开会话就可以自动通过密钥对来连接了



#### 3.2.4.2MACOS上的配置

这里再偷个懒,还是用刚刚windows用的那一对密钥就行了,把私钥复制到mac中的自定路径

然后在~/.ssh目录下创建一个名为**config**的文件,在其中添加如下内容:

```
Host          自己起一个好记的别名
HostName      VPS的IP地址
Port          VPS开放的SSH端口
User          连接VPS时的用户名
IdentityFile  私钥文件的路径
```

然后chmod 600修改这个config文件的权限

**注意这里权限不能给得过大,例如不能无脑777,系统会判定不安全从而拒绝连接**

配置完成后,在mac的命令行界面直接通过

```
ssh 别名
```

就可以快速连接了

<img src="/images/vps/image-20220803130243341.png" alt="image-20220803130243341" style="zoom:67%;" />



最后在windows和mac都设置完之后,到VPS中修改ssh设置禁用密码登录,只允许使用密钥登录:

```shell
sudo nano /etc/ssh/sshd_config
# PasswordAuthentication no
```

<img src="/images/vps/image-20220803130358040.png" alt="image-20220803130358040" style="zoom:67%;" />

设置完毕之后重启ssh:

```shell
sudo systemctl restart ssh
```



### 3.2.5使用FTP来传输文件

Windows下,在通过Xshell建立连接之后,点击上方工具栏中的FTP按钮就可以通过现有的连接配置来使用Xftp工具(先安装好)建立FTP连接

<img src="/images/vps/image-20220803133009579.png" alt="image-20220803133009579" style="zoom:67%;" />

MACOS下使用的是FIleZilla

<img src="/images/vps/image-20220803133125636.png" alt="image-20220803133125636" style="zoom:67%;" />

在站点管理器中选择使用密钥登录,并指定好密钥的存储位置就好了

<br>

***

<br>

## 3.3常用工具的安装

### git

```shell
sudo apt install git-core
git --version #验证安装是否成功
```

解决git clone github的仓库报错**Permission denied(publickey)**的问题:

这是因为本地 ~/.ssh目录下缺少用于从github克隆源码的密钥

```shell
ssh-keygen -t rsa -C "自定义备注信息" # 生成密钥对,执行后一路回车就行,密钥默认存储在~/.ssh目录
```

生成密钥之后,进入~/.ssh目录,打开公钥文件(带pub的那个),复制其文本内容(**以ssh-rsa开头,以生成密钥时填写的备注信息结尾**)

然后进入github自己的账户页面,添加ssh keys,将公钥的文本内容粘贴进入就行了(随便起一个名字)

<img src="/images/vps/image-20220803134202558.png" alt="image-20220803134202558" style="zoom:67%;" />

<br>

### 抓包工具tcpdump

```shell
sudo apt install tcpdump
```

使用tcpdump抓包时前面要加上sudo,否则会显示找不到命令..

目前的用法为:

```shell
sudo tcpdump -w test.cap
```

意思是将捕获的流量存储为test.cap,这个文件可以传到本地后用wireshark打开

todo:再开一篇新的记录一下tcpdump的参数使用

<br>

### 包管理工具yum

```bash
sudo apt update
sudo apt upgrade
sudo apt install yum
```



<br>

### 文本编辑工具Vim

还是用不惯nano,果断安装一个vim:

```shell
sudo apt install vim
```

<br>

### 编译工具make

```shell
sudo apt install make
```

<br>

### C语言编译工具gcc

```shell
sudo apt install gcc
```

<br>

### python库管理器pip(pip3)

```shell
apt-get install python3-pip
```

<br>

### GO语言环境

这个按照官网给的步骤安装是最靠谱的:<https://go.dev/doc/install>

从官网下载最新的安装包并解压:(根据自己的CPU架构选择版本,查看CPU架构见下面的其他命令)

```shell
wget https://golang.org/dl/go1.18.4.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.18.4.linux-amd64.tar.gz
```

将**/usr/local/go/bin**添加到环境变量:

```shell
sudo vim ~/.profile
```

将下面这一行追加到底部:

```
export PATH=$PATH:/usr/local/go/bin
# 后面将其他的路径加入环境变量也是一样的操作,更改目录即可
```

最后使其生效:

```shell
source ~/.profile
```

<br>

### 其他用到的命令

```shell
dpkg --print-architecture #查看CPU架构
cat /etc/debian_version #查看系统版本
echo $PATH #查看当前的环境变量
```



<br>

***

<br>

# 4 路由测试

一个测试vps回程路由的小脚本:

```shell
sudo wget -O jcnf.sh https://raw.githubusercontent.com/Netflixxp/jcnfbesttrace/main/jcnf.sh
sudo bash jcnf.sh
```

可以测试该vps到国内几个固定节点(不同地理位置和运行商线路)的路由情况

<img src="/images/vps/image-20220803135227748.png" alt="image-20220803135227748" style="zoom:67%;" />

<img src="/images/vps/image-20220803135233337.png" alt="image-20220803135233337" style="zoom:67%;" />

<img src="/images/vps/image-20220803135243191.png" alt="image-20220803135243191" style="zoom:67%;" />

<img src="/images/vps/image-20220803135248919.png" alt="image-20220803135248919" style="zoom:67%;" />

对应[第二部分](#001)的线路介绍,可以看到三网回程GIA的VPS,无论是电信,联通还是移动,回国前走的都是**59.43**节点

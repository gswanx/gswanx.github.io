---
title: Web杂项3_linux相关
abbrlink: d5ca9872
date: 2022-10-27 15:32:46
cover: false
tags:
  - CTF-Web
  - Linux
categories:
  - CTF
  - Web安全
---



# 获得文件读取权限后需要关注的文件

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>关注内网其他主机的信息</span>

```
/etc/passwd 用户信息
/etc/shadow
/etc/hosts //IP地址和域名的快速解析, 可能有内网主机名到内网ip的映射
/root/.bash_history //root的bash历史记录
/root/.ssh/authorized_keys
/root/.mysql_history //mysql的bash历史记录
/root/.wget-hsts
/opt/nginx/conf/nginx.conf //nginx的配置文件
/var/www/html/index.html
/etc/my.cnf //mysql的配置文件
/etc/httpd/conf/httpd.conf //httpd的配置文件
/proc/self/fd/fd[0-9]*(文件标识符)
/proc/mounts
/porc/config.gz
/proc/sched_debug // 提供cpu上正在运行的进程信息，可以获得进程的pid号，可以配合后面需要pid的利用
/proc/mounts // 挂载的文件系统列表
/proc/net/arp //arp表，可以获得内网其他机器的地址--可能有内网主机信息
/proc/net/route //路由表信息--可能有内网主机信息
/proc/net/tcp and /proc/net/udp // 活动连接的信息
/proc/net/fib_trie // 路由缓存--可能有内网主机信息
/proc/version // 内核版本
/proc/[PID]/cmdline // 可能包含有用的路径信息
/proc/[PID]/environ // 程序运行的环境变量信息，可以用来包含getshell
/proc/[PID]/cwd // 当前进程的工作目录
/proc/[PID]/fd/[#] // 访问file descriptors，某写情况可以读取到进程正在使用的文件，比如access.log
ssh
/root/.ssh/id_rsa
/root/.ssh/id_rsa.pub
/root/.ssh/authorized_keys
/etc/ssh/sshd_config
/var/log/secure
/etc/sysconfig/network-scripts/ifcfg-eth0
/etc/syscomfig/network-scripts/ifcfg-eth1
```



# 关于proc目录

>Linux系统上的`/proc`目录是一种文件系统，即`proc`文件系统。与其它常见的文件系统不同的是，`/proc `是一种伪文件系统（也即虚拟文件系统），存储的是当前内核运行状态的一系列特殊文件，用户可以通过这些文件查看有关系统硬件及当前正在运行进程的信息，甚至可以通过更改其中某些文件来改变内核的运行状态。
>
>简**单来讲，`/proc` 目录即保存在系统内存中的信息，大多数虚拟文件可以使用文件查看命令如`cat`、`more`或者`less`进行查看**，有些文件信息表述的内容可以一目了然，但也有文件的信息却不怎么具有可读性。



`/proc`目录下有很多命名为`/proc/进程号` 的子目录, 这些子目录中包含很多存储了与该进程相关的信息的文件

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>另外, 使用`/proc/self`能够取到当前进程所属的子目录,等价于`/proc/当前进程的pid`</span>

`/proc/pid`下的常见子目录包括:

- `/proc/pid/cmdline`: 启动当前进程的完整命令

  >在真正做题的时候，我们是不能通过命令的方式执行通过`cat`命令读取`cmdline`的，因为如果是`cat`读取`/proc/self/cmdline`的话，得到的是`cat`进程的信息，所以我们要通过题目的当前进程使用读取文件（如文件包含漏洞，或者`SSTI`使用`file`模块读取文件）的方式读取`/proc/self/cmdline`

- `/proc/pid/cwd`:一个指向当前进程运行目录的符号链接。可以通过查看`cwd`文件获取目标**指定进程环境的运行目录**.

- `/proc/pid/exe`:一个指向启动当前进程的可执行文件（完整路径）的符号链接。通过`exe`文件我们可以**获得指定进程的可执行文件的完整路径**

- `/proc/pid/environ`:当前进程的环境变量列表，彼此间用空字符（NULL）隔开。变量用大写字母表示，其值用小写字母表示。

- `/proc/pid/fd` 这是一个目录,其中包含了当前进程打开的每个文件的文件描述符,这些文件描述符是指向实际文件的一个符号链接，即每个通过这个进程打开的文件都会显示在这里。所以我们**可以通过`fd`目录里的文件获得指定进程打开的每个文件的路径以及文件内容**。

  其中每个文件描述符的命名为`/proc/pid/fd/id`,例如`/proc/pid/fd/3`

  当一个新进程建立时，此进程将默认有 0，1，2 的文件描述符,分别代表标准输入,标准输出,标准错误输出

  那么当打开一个新文件时,文件描述符会从3开始

  即便从外部（例如如`os.remove(SECRET_FILE)`）删除这个文件之后，在 `/proc/pid` 目录下的 `fd` 文件描述符目录下还是会有这个文件的文件描述符，**通过这个文件描述符我们即可得到被删除文件的内容**

***

# perl GET的命令执行漏洞



**`GET`底层调用了`open`,而`open`支持`file协议`,可以通过`file`协议来执行命令,(尾部需要有管道符)但前提是同目录下要存在一个和要执行的命令同名的文件. 例如:**

```
➜  test GET 'file:id|'
➜  test touch 'id|'
➜  test GET 'file:id|'
uid=1000(moxiaoxi) gid=1000(moxiaoxi) groups=1000(moxiaoxi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
这里在目录下创建了一个名为"id|"的文件之后, 使用GET 'file:id|'就能够成功地执行命令了
```

`perl`的`GET`还能够直接读取目录, 这里不通过底层`open`调用`file`协议,所以对于同目录下有无同名文件没有要求

```
GET /  读取根目录
```



例题:[[perlGET命令执行]SSRFme](383bfb1c.html)














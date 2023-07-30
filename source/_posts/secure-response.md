---
title: Linux+Windows的安全事件响应清单
description: '发现,定位,处理安全事件的各种角度以及简单手段'
cover: false
tags:
  - 安全
categories:
  - 杂项
typora-root-url: ..
password: daddy987
message: 输入密码，查看文章
abbrlink: d0bdcf10
date: 2022-10-03 21:54:15
---



# 1_网络情况

- **网络拓扑结构**

- **负载均衡情况**

  是否部署了负载均衡服务器，如果存在，其他的相同业务类型的服务器情况

- **防火墙配置**,是否加入`DMZ`区

- **路由器配置**,如果是内网的话,还要检查一下端口映射的情况



<br>

***

<br>

# 2_被攻击目标

- 当前`Web`应用程序的情况

  `Web`应用程序架构包括：中间件、数据库、语言等；网站域名、标题、被入侵后的表现，使用 **微步在线**等查询此域名或IP的风险状况

- 域名及相关备案信息

- 安全扫描,可以使用`wvs(Web Vulnerability Scanner)`对目标进行漏洞扫描

  >`WVS(Web Vulnerability Scanner)`是一个自动化的Web应用程序安全测试工具，它可以扫描任何可通过Web浏览器访问的和遵循HTTP/HTTPS规则的Web站点和Web应用程序。适用于任何中小型和大型企业的内联网、外延网和面向客户、雇员、厂商和其它人员的Web网站。
  >
  >功能包括:
  >
  >- `WebScanner` ：核心功能，web安全漏洞扫描 (深度，宽度 ，限制20个)
  >- `Site Crawler`：站点爬行，遍历站点目录结构
  >- `Target Finder` ：主机发现，找出给定网段中开启了80和443端口的主机
  >- `Subdomian Scanner` ：子域名扫描器，利用DNS查询
  >- `Blind SQL Injector` ：盲注工具
  >- `Http Editor http`：协议数据包编辑器
  >- `HTTP Sniffer` ： HTTP协议嗅探器 （fiddler，wireshark,bp）
  >- `HTTP Fuzzer`： 模糊测试工具 (bp)
  >- `Authentication Tester` ：Web认证破解工具

- 使用**vulmap**、**JexBoss**等专用漏洞扫描工具对目标进行中间件漏洞扫描. 

- 端口开放情况,可以使用`nmap`来扫描



<br>

***

<br>

# 3_Windows服务器事件分析

## 3.1账号情况

- 查看当前已登录的账号

  `query user`命令 : 显示有关用户服务器上用户会话远程桌面会话主机的信息。

  还可以使用`mimikaz`工具来查看当前内存中的用户情况:https://github.com/ParrotSec/mimikatz

- 检查用户目录`C:\Users`中是否有可疑文件,主要看`Desktop`,`Download`.

- 检查可疑账户:

  `win+R`输入` lusrmgr.msc`打开**本地用户和组**管理窗口,查看是否存在可疑账号。

  `net user <username>`命令查看每个用户的详情

- 使用 PC Hunter 或 D盾 查看当前系统中的用户情况

<br>

## 3.2系统日志风险分析

`win+R`输入` eventvwr.msc`打开**事件查看器**

事件查看器切换到**安全**子节点，右击选择**将所有事件另存为**进行导出，导出文件名为 `Security.evtx`。同样的方式导出 **系统节点**的日志

<img src="/images/secure-response/image-20221003222105309.png" alt="image-20221003222105309" style="zoom:67%;" />



接下来可以使用`LogParser`工具来分析日志:(命令行执行)

```shell
# 查询登录成功的事件:
LogParser.exe -i:EVT –o:DATAGRID "SELECT * FROM C:\Security.evtx where EventID=4624"
# 定登录时间范围的事件:
LogParser.exe -i:EVT –o:DATAGRID "SELECT * FROM C:\Security.evtx where TimeGenerated>'2018-06-19 23:32:11' and TimeGenerated<'2018-06-20 23:34:00' and EventID=4624"
# 提取登录成功的用户名和IP:
LogParser.exe -i:EVT –o:DATAGRID "SELECT EXTRACT_TOKEN(Message,13,' ') as EventType,TimeGenerated as LoginTime,EXTRACT_TOKEN(Strings,5,'|') as Username, EXTRACT_TOKEN(Message,38,' ') as Loginip FROM C:\Security.evtx where EventID=4624"
# 登录失败的所有事件:
LogParser.exe -i:EVT –o:DATAGRID "SELECT * FROM c:\Security.evtx where EventID=4625"
# 取登录失败用户名进行聚合统计:
LogParser.exe -i:EVT "SELECT EXTRACT_TOKEN(Message,13,' ') as EventType, EXTRACT_TOKEN(Message,19,' ') as user, count(EXTRACT_TOKEN(Message,19,' ')) as Times, EXTRACT_TOKEN(Message,39,' ') as Loginip FROM c:\Security.evtx where EventID=4625 GROUP BY Message"
# 系统开关机记录:
LogParser.exe -i:EVT –o:DATAGRID "SELECT TimeGenerated, EventID, Message FROM c:\System.evtx where EventID=6005 or EventID=6006"
```

<br>

## 3.3病毒、木马、后门风险排查

- 查毒软件查杀
- 使用`Process Hacker`dump内存数据并分析,主要针对`powershell`型木马

<br>

## 3.4异常风险分析

- **查看端口情况:**

  ```shell
  netstat -ano #端口使用情况
  netstat -ano | findstr "端口号"  # 查看端口对应的活动连接
  tasklit | findstr "进程号"    # 查看相应PID的进程
  ```

- **进程分析:** 使用**D盾**, **火绒剑**等工具

- **网络分析**

  1. `Wireshark`抓包流量分析
  2. 火绒剑分析网络使用情况
  3. 查看系统 **hosts** 文件是否有可疑的域名解析。
  4. 对可疑域名从 **微步在线** 上进行查询分析，提供域名（IP）的风险标签

- **开机启动项分析:**

  1. 开始菜单中的启动菜单项

  2. 命令`wmic startup list full`

  3. 运行`msconfig`查看系统的启动配置。

  4. 运行`regedit`,打开注册表并查看一下三个表项是否有可疑项

     ```
     HKEY_CURRENT_USER\software\micorsoft\windows\currentversion\run HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Runonce
     ```

  5. 运行`gpedit.msc`，查看组策略中`计算机配置` – `Windows设置` –` 脚本（启动/关机）`,右侧的启动配置中是否有可疑的启动脚本。

- **计划任务分析:**

  1. `控制面板` – `管理工具` – `任务计划程序`，打开计划任务程序
  2. 运行` taskschd.msc`

- **系统服务分析:**

  运行`services.msc`查看系统服务

- **运行时分析:**

  使用**火绒剑**对系统进行开启监控，查看当前系统对注册表、文件等操作。

  

# 4_Linux服务器事件分析

## 4.1历史命令分析

**优先对历史命令进行备份、分析。**

分别针对普通用户和`root`的 **`.bash_history `**文件进行备份、分析。使用` cat ~/.bash_history > history.txt `进行备份。

主要关注的内容包括：

- 是否存在敏感操作。

- 是否下载过文件。

- 是否存在远程连接其他服务器。

- 执行过的 shell 脚本。

<br>

## 4.2账号安全风险分析

- 对比 `/etc/passwd` 用户信息与 `/etc/shadow` **影子文件**中**用户的对应关系**，是否存在多余的用户。

  >`/etc/passwd`文件，是系统用户配置文件，存储了系统中所有用户的基本信息，并且所有用户都可以对此文件执行读操作。
  >
  >`/etc/shadow`文件，用于加密存储 Linux 系统中用户的密码信息，又称为“影子文件”。

- 使用 `who`命令查看当前登录用户信息，`tty`表示本地登录，`pts`表示远程登录。

- 使用 `stat /etc/passwd` 查看密码文件上一次修改的时间，如果最近被修改过，就可能存在问题。

- 使用 `cat /etc/passwd | grep -v nologin` 查看除了不可登录以外的用户都有哪些。

- 使用` cat /etc/passwd | grep x:0` 查看哪些用户为 `root` 权限。

- 使用 `cat /etc/passwd | grep /bin/bash` 查看哪些用户使用 `shell`。

- 使用 `awk '/\$1|\$6/{print $1}' /etc/shadow` 查看可疑远程登录的账号。

- 使用 `more /etc/sudoers | grep -v "^#\|^$" grep "ALL=(ALL)"` 查询具有`sudo` 权限的账号。

- 防止有隐匿用户，需要使用` lsof` 、`ps` 命令来排查。

  排查命令：

  ```shell
  lsof -i:22 | grep EST
  ps -ef | grep ssh  # 隐匿用户的sshd进程包含 notty 字样
  ```

> **隐匿用户：**通过命令 `ssh -lroot 192.168.1.80 /usr/bin/bash` 或者 `ssh -T root@192.168.1.80 /usr/bin/bash  -i` 来连接时，通过 `w` 和` last` 命令是无法查到此连接用户。  

 

## 4.3系统日志风险分析

**日志默认位置为 `/var/log/`，**备份所有的日志文件。

### 4.3.1_/var/log/secure（或 /var/log/auth.log）

- 定位有多少IP在爆破主机的 **root** 账号。

```shell
grep "Failed password for root" /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -nr | more
```

- 定位哪些IP在爆破。

```shell
grep "Failed password" /var/log/secure|grep -E -o "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)"|uniq -c
```

- 爆破用户名字典。

```shell
grep "Failed password" /var/log/secure|perl -e &apos;while($_=<>){ /for(.*?) from/; print "$1\n";}&apos;|uniq -c|sort -nr
```

- 登陆成功的IP有哪些。

```shell
grep "Accepted " /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -nr | more
```

- 登陆成功的日期、用户名、IP

```shell
grep "Accepted " /var/log/secure | awk '{print $1,$2,$3,$9,$11}'
```

<br>

### 4.3.2_/var/log/wtmp

此文件中**记录登录系统成功的用户信息**，是一个二进制文件，需要**使用 `last` 命令来查看**。

>`Linux last` 命令用于显示用户最近登录信息。
>
>单独执行 **last** 指令，它会读取位于` /var/log/`目录下，名称为 `wtmp` 的文件，并把该文件记录登录的用户名，全部显示出来。

可以使用` last -f wtmp` 来指要加载的`wtmp`文件。

<br>

### 4.3.3_/var/log/btmp

此文件中记录了**登录系统失败的用户信息**，是一个二进制文件，需要使用` lastb `命令来查看此文件内容。

下面的命令用来查看按照登录失败次数进行倒序排列的所有用户信息。

```shell
lastb | awk '{print $3}' | sort | uniq -c | sort -nr
```

其中:

- `lastb` 读取 `btmp `文件内容

- `awk '{print $3}' `提取第三列数据，此列为尝试登录者的IP地址

- `sort` 排序

- `uniq -c` 删除重复的行，并且在当前列右侧显示重复次数

- `sort -n `按照数字大小进行排序，`-r`降序排列

<br>

## 4.4异常风险分析

### 4.4.1端口与进程分析

```shell
netstat -antp #查看端口开放情况
ls -l /proc/<pid>/exe #查看端口对应链接的文件路径
ls -l /proc/*/exe | grep xxx #如果知道恶意程序的启动文件大致位置,使用此命令来发现无文件的恶意进程
netstat -antlp | grep 4.2.2.1 | awk '{print $7}' | cut -fl -d"/" 
# 如果知道可疑的IP地址，可以以此来获取程序PID。

#查找是否存在隐藏进程：
ps -ef | awk'{print $2}'|sort -n| uniq > tasklist1.txt
ls /proc | sort -n | uniq > tasklist2.txt
diff tasklist1.txt tasklist2.txt
```

<br>

### 4.4.2运行时分析

使用 `top` 来分析**当前的进程占用资源情况**。

使用` top -p pid` 监控指定进程。

使用` ps aux --sort=pcpu | head -10` 查看CPU占用率前十的进程。

<br>

### 4.4.3系统命令风险分析

需要使用 `stat` 命令查看系统常用命令是否被修改过，防止被入侵者替换系统命令。

需要查看的命令包括：`netstat`、`rm`、`ps`、`top`、`kill`

 <br>                        

### 4.4.4网络分析

使用` tcpdump -w file.bcap` 来对网络进行抓包分析。`-i interface` 指定监听的网卡名称；`-D` 列出服务器所有网卡。

查看 `/etc/hosts` 文件，是否存在可疑的DNS解析记录。

对可疑域名从 **微步在线** 上进行查询分析，提供域名（IP）的标签。

 <br>

### 4.4.5开机启动项分析

使用下面的命令分析启动项文件，是否包含可疑文件。

```shell
more /etc/rc.local
/etc/rc.d/rc[0~6].d/
ls -l /etc/rc.d/rc3.d/  # 分别查看 rc0.d ~ rc6.d 7个文件
```

<br>

### 4.4.6定时（计划）任务分析

对定时任务进行分析，分析是否存在可疑任务。

**使用 `crontab -l` 查看定时任务。**

使用` crontab -e `编辑定时任务。

使用 `crontab -u root -l` 查看 `root `用户任务计划。

使用 `ls /var/spool/cron/`查看每个用户自己的执行计划。

定时任务文件位置：

```
/var/spool/cron/*  #centos
/var/spool/cron/root
/var/spool/cron/crontabs/* #ubuntu
/var/spool/anacron/*
/etc/crontab
/etc/anacrontab
/etc/cron.hourly/*
/etc/cron.daily/*
/etc/cron.weekly/
/etc/cron.monthly/*
```

<br>



### 4.4.7系统服务分析

对系统当前服务进行分析。

**使用 `chkconfig --list` 查看当前的服务。**

对于不支持 `chkconfig` 的使用`systemctl list-unit-files | grep enabled`查看启用的服务。



# 5_Web应用程序风险分析

## 5.1Web应用日志分析

提取Web应用日志，对日志进行分析，查找是否有异常访问。

借助于分析工具对日志进行分析，如：`web-log-parser`。

## 5.2Web应用木马查杀

如果服务器部署了Web应用程序，需要对Web应用源代码、目录进行扫描，使用 **D盾**等工具进行全盘扫描。


























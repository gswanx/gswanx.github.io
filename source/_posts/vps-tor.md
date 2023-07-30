---
title: 在VPS上配置使用snowflake网桥的Tor
abbrlink: 3b7c15b1
date: 2022-08-03 15:01:04
description: '在VPS上配置Tor环境并抓取流量!'
cover: false
tags:
  - VPS
  - Tor
categories: 
  - Tor
typora-root-url: ..
password: daddy987
message: 输入密码，查看文章
---





记录一下在Debian10 VPS上配置Tor和Snowflake环境的过程以及踩过的坑(‾◡◝)

# 1 安装Tor

一开始想当然地直接

```shell
sudo apt install tor
```

但安装完后发现安装的Tor非常老

查了一下,官网雀食也说了不要这样安装

 <https://community.torproject.org/onion-services/setup/install>

<img src="/images/vps-tor/image-20220803152948470.png" alt="image-20220803152948470" style="zoom:67%;" />

所以先把官方给安装的Tor卸载掉 😭

```
sudo apt remove tor
```

选择通过源码来安装最新版本的tor: <https://www.torproject.org/download/tor/>

这里直接本地下载完通过FTP把安装包传到远程

解压并进入目录:

```shell
tar xzf tor-0.4.7.8.tar.gz; cd tor-0.4.7.8
```

接下来:

```shell
sudo ./configure
```

这一步骤会提示缺少很多依赖项,缺啥就按照提示补充安装啥就好了

<img src="/images/vps-tor/image-20220803153414146.png" alt="image-20220803153414146" style="zoom:80%;" />

接下来执行:

```shell
sudo make # 编译代码
sudo make install # 安装
tor --verion # 检查是否安装成功
```

><span style="font-size:25px">**关于configure,make,make install的补充知识点**</span>:
>
>- configure: 为后面的`make`和`make install`检查和准备依赖项,准备好软件的构建环境
>- make: 当`configure`配置环境完毕后,通过`make`来执行构建. make会执行在`Makefile`文件中定义的一系列任务,从而将源码编译成可执行文件.   有的源码包中并没有Makefile文件,一般是一个模版文件 `Makefile.in` 文件，然后 `configure` 根据系统的参数生成一个定制化的 `Makefile` 文件。
>- make install: 当执行完`make`后,可执行文件就已经被生成好了,通过`make install`将可执行文件、第三方依赖包和文档复制到正确的路径。 这个路径一般都在环境变量`PATH`中



这样以来,Tor就安装好了

<img src="/images/vps-tor/image-20220803155108372.png" alt="image-20220803155108372" style="zoom:80%;" />



此时Tor可以使用默认的配置文件torrc来运行,这个torrc的存储位置为 `/usr/local/etc/tor/torrc`

<img src="/images/vps-tor/image-20220803155316110.png" alt="image-20220803155316110" style="zoom: 67%;" />

但是此时尽管接入了tor网络,其他应用仍然需要配置走tor的socks代理端口,才能通过tor来访问网络

<br>

***

<br>

# 2 监控工具nyx

nyx是一个基于python3的Tor运行状态监控工具,通过pip来安装:

```
pip3 install nyx
```

安装好后黄字显示nyx被安装在了一个不在环境变量里的路径中

<img src="/images/vps-tor/image-20220803155950861.png" alt="image-20220803155950861" style="zoom:67%;" />

那么将这个目录加入环境变量就行:

```
sudo vim ~/.profile
```

在最底部加上这个目录就好了

<img src="/images/vps-tor/image-20220803160055066.png" alt="image-20220803160055066" style="zoom:67%;" />

能够运行nyx: 

![image-20220803160129889](/images/vps-tor/image-20220803160129889.png)

nyx连接的是Tor的控制端口(默认是9051),但初始的torrc配置文件中并没有指定控制端口

所以需要去修改torrc配置文件: 在底部加上这两行:

```
ControlPort 9051 
CookieAuthentication 1
```

之后就可以通过nyx来查看tor的运行状态了

第一页是流量监控:(通过方向键来切换)

<img src="/images/vps-tor/image-20220803160415810.png" alt="image-20220803160415810" style="zoom:67%;" />

第二页是当Tor的链路情况,包括每条链路的各个节点的情况:

<img src="/images/vps-tor/image-20220803160507979.png" alt="image-20220803160507979" style="zoom:67%;" />

第三页是一些其他配置的情况:(包括使用了什么网桥等等)

<img src="/images/vps-tor/image-20220803160558986.png" alt="image-20220803160558986" style="zoom:67%;" />

第四页是当前运行tor所使用的torrc文件内容:

<img src="/images/vps-tor/image-20220803160802180.png" alt="image-20220803160802180" style="zoom:67%;" />

<br>

***

<br>

# 3 配置Snowflake网桥

<https://gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake>

先把这个仓库下载到本地,这里使用git clone ssh地址 仍然会报错 **Permission denied(publickey)**

[购买VPS和一些初始设置](./4e671577.html)中3.3的解决办法只能使从github克隆不报错,而这里是gitlab...注册还需要审核

<img src="/images/vps-tor/image-20220803162329498.png" alt="image-20220803162329498" style="zoom:67%;" />

git clone下面这个https不会报错

成功克隆snowflakes源码后,进入client目录开始构建snowflake客户端:

```shell
go get
go build
```

上面两个过程都没报错,之后会在client目录下生成一个client可执行文件和一个torrc示例

<img src="/images/vps-tor/image-20220803162755595.png" alt="image-20220803162755595" style="zoom:67%;" />

<span id="002">根据自己的需要修改一下torrc文件</span>

```
# 使用网桥
UseBridges 1
DataDirectory datadir
# 指定snowflake网桥客户端的可执行文件路径
# 这里的snowflake的可执行文件路径改为绝对路径
ClientTransportPlugin snowflake exec /home/lust1n/snowflake/snowflake/client/client -log snowflake.log

Bridge snowflake 192.0.2.3:1 2B280B23E1107BB62ABFC40DDCC8824814F80A72 fingerprint=2B280B23E1107BB62ABFC40DDCC8824814F80A72 url=https://snowflake-broker.torproject.net.global.prod.fastly.net/ front=cdn.sstatic.net ice=stun:stun.voip.blackberry.com:3478,stun:stun.altar.com.pl:3478,stun:stun.antisip.com:3478,stun:stun.bluesip.net:3478,stun:stun.dus.net:3478,stun:stun.epygi.com:3478,stun:stun.sonetel.com:3478,stun:stun.sonetel.net:3478,stun:stun.stunprotocol.org:3478,stun:stun.uls.co.za:3478,stun:stun.voipgate.com:3478,stun:stun.voys.nl:3478


# 固定Socks代理端口,其他应用使用tor代理需要指定使用此端口作为代理
SOCKSPort 34627
# 指定控制端口
ControlPort 9051
CookieAuthentication 1

```



之后想要通过这个torrc配置来运行tor,需要通过`-f `参数来指定运行tor时加载的torrc配置文件

```shell
tor -f torrc 
```

由于这个torrc中配置了使用snowflake作为网桥,并且指定了snowflake client的可执行文件路径

此时运行tor就可以看到连接了snowflake网桥

![image-20220803163916456](/images/vps-tor/image-20220803163916456.png)

<br>

***

<br>

# 4 proxychains工具

安装 proxychains方便其他软件使用tor作为其代理

```shell
sudo apt install proxychains # 安装
sudo vim /etc/proxychains.conf # 编辑proxychains配置
```

<img src="/images/vps-tor/image-20220803164206125.png" alt="image-20220803164206125" style="zoom:67%;" />

这里把最下面的socks5代理修改为` 127.0.0.1:tor的监听端口`, 这个端口在[torrc](#002)中指定

之后执行命令时,如需走tor代理,在其他命令前加上proxychains就可以了,例如:

```shell
proxychains curl https://github.com/
```

通过专门的网站检查当前自己的IP:可以看到返回的已经不是自己原来的IP了

```shell
proxychains wget -qO - https://api.ipify.org; echo
```

<img src="/images/vps-tor/image-20220803164802162.png" alt="image-20220803164802162" style="zoom:67%;" />

通过nyx可以看到,这个IP正是某一条链路的出口节点IP:

<img src="/images/vps-tor/image-20220803164905318.png" alt="image-20220803164905318" style="zoom:67%;" />

<br>

***

<br>

# 5 使用TCPdump捕获Tor-Snowflake的流量

- 首先启动tcpdump抓包:

  ```
  sudo tcpdump -w test.cap
  ```

  ![image-20220803171409946](/images/vps-tor/image-20220803171409946.png)

- 然后在torrc的同目录下运行tor

  ```
  tor -f torrc
  ```

- 最后通过proxychains来运行curl,wget等命令来产生流量:

  基于python的简单脚本:

  ```python
  import os
  def read_sites(path="./inwall.txt"): # 这个列表里包含若干个常用网站的地址,每个一行
      sitelist = []
      with open(path, "r",encoding='UTF-8') as f:
          while True:
              site = f.readline()
              site = site.strip('\n')
              sitelist.append(site)
              if not site:
                  break
      return sitelist
  
  sitelist = read_sites()
  state = []
  site_index = 0
  for site in sitelist:
      os.mkdir("./{}".format(str(site_index)))
      if len(site) < 7:
          continue
      command = "proxychains wget -l 5 -p -np -k -H -t 5 -P ./{}/ --user-agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36' {}".format(str(site_index),site)
      del_command = "rm -rf ./{}".format(str(site_index)) 
      # 每下载完一个网站的资源,就将那个目录删除掉,因为现在只需要捕获的流量数据
      site_index = site_index + 1
      print("processing:{}".format(site)) # 输出当前处理到哪个网页
      if not os.system(command):
          state.append("{}:1".format(site))
          os.system(del_command)
      else:
          state.append("{}:0".format(site))
          os.system(del_command)
          continue
  # 命令示例:
  """
  wget -l 5 -p -np -k -H -t 5 -P /home/gswanx/shadow_1111/collect_site/weibo/  --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36" https://www.weibo.com
  """
  print(state)
  ```

  上述代码使用wget命令一次递归地下载列表里地网站资源(不仅下载首页,还递归地访问网站上的子链接)

  




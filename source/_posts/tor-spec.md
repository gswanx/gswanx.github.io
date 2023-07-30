---
title: TOR原理的不完全总结
description: '整理一下阅读Tor官方文档和一些论文时乱七八糟的笔记'
cover: false
tags:
  - Tor
categories:
  - Tor
typora-root-url: ..
abbrlink: 75310f08
date: 2022-08-18 16:52:41
---



# 0_啥是Tor

>Tor(**The Onion Router,直译:洋葱路由器**) 是用于实现匿名通信的软件, 是目前应用范围最广的匿名通信系统, 是通常所讲的「暗网」所依赖的底层技术. Tor本质上是为了保护用户的身份和访问内容等隐私信息而诞生的技术, 不仅可以用来访问暗网网站(域名一般以`.onion`结尾), 也可以用来访问普通的网站.
>
><br>
>
>Tor网络是由分布在全球各地的由志愿者提供的中继节点组成的.  之所以叫做「洋葱」路由器, 是因为Tor对隐私的保护主要是通过「**多层加密**」和「**多跳代理**」来实现的, 当一个用户需要接入Tor网络时, 客户端会通过特殊算法选择若干组, 每一组共三个节点分别作为`入口节点(Guard Relay)`,  `中间节点(Middle Relay)`和`出口节点(Exit Relay)`来构成一条`Tor链路(circuit)`.  当用户使用浏览器访问了`www.google.com` 这样的普通网站时, 他的请求数据包会在本地进行多层加密后发送到一条`Tor链路(circuit)`上,   这个数据包接下来会依次被转发到`入口节点(Guard Relay)`,  `中间节点(Middle Relay)`和`出口节点(Exit Relay)`, 每到一个节点, 这个数据包就会被进行一层解密, 这个过程就像**剥洋葱**一样. 当数据在`出口节点(Exit Relay)`剥下最后一层洋葱皮后, `出口节点(Exit Relay)`就得到了明文的请求包, 然后它再去访问`www.google.com`的服务器. 服务器返回响应的过程也与此类似, 数据同样会在经过链路上的各个节点时经历一个「剥洋葱」的过程. 
>
><br>
>
>从上面的过程里就能够看出: 一个不怀好意的攻击者无论在链路上的任何一个位置截获数据包, 他都无法**同时知道通信双方的身份以及通信的内容**.  当然,前面也只是用户访问一个普通的互联网网站的过程. 如果这个用户在浏览器中输入的是URL以`.onion`结尾的暗网网站, 通信的过程还会更加复杂, 不仅仅是客户端, 另外一边的暗网服务器也会建立一个三条的链路, 然后服务器的链路和客户端的链路再通过一个称为`汇聚节点(RPO)`的路由器沟通. 这个多出来过程主要是为了保护暗网服务器的匿名性.

<br>

想要访问暗网, 最简单的办法就是谷歌搜索Tor官网并下载Tor浏览器, 现在安装好浏览器之后的配置基本上都是傻瓜式的了. 不过在国内由于墙的存在, 大部分能够接入Tor网络的节点都被封锁了, 即使偶尔运气好能够连接上也是速度感人.  想体验的话一般还要先挂梯子或者使用Tor官方提供的网桥.. 当然,没有研究技术原理之类的特殊需求的话还是不要去尝试了.

<img src="/images/tor-spec/image-20220819104351867.png" alt="image-20220819104351867" style="zoom:67%;" />

<br>

***

<br>

# 1_Tor的简要结构和工作流程

**(简单了解的话只看这一部分就行啦)**

<img src="/images/tor-spec/1.png" alt="Tor网络的简单结构(出自:「匿名通信与暗网研究综述」罗军舟等)" style="zoom:67%;" />

## <span style="font-size:25px;font-weight:bold">1.1组件介绍</span>

- `客户端Client`:  <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称Client) </span>用户电脑上安装的本地程序(**例如Tor浏览器**), 将用户上网的数据封装成`Tor cell`并层层加密后发送出去.
- `洋葱路由器Onion Router`: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称OR)</span> 由志愿者和官方提供的中继节点.  Tor的默认链路由三跳节点组成,分别为`入口节点Guard Relay`<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称Entry)</span>, `中间节点Middle Relay`<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称Middle)</span>和`出口节点Exit Relay`<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称Exit)</span>. 其中`Entry`一般是可信度较高的守护节点.
- `隐藏服务器Hidden Server`: <span style='color:black;background:yellow;font-family:hei'>**(后面简称HS)**</span> 提供隐藏服务的服务器(简单来说就是那些暗网网站的服务器),必须使用Tor客户端接入Tor网络之后才能够访问到.
- `目录服务器Directory Server`: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称DS)</span> 保存了所有`OR`的IP地址,带宽等信息. `Client`首次启动时要向其请求这些信息, 以便完成节点的选择和链路的建立.
- `隐藏服务目录服务器Hidden Service Directory`: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称HSD)</span> 存储并向`client`提供`HS`的`引入节点IPO`和公钥等信息.
- `引入节点Introduction Point`: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称IPO)</span> 由`HS`来选择的具有特殊功能的`OR`. 一个`HS`在开始接入Tor网络时, 就会选择三个`OR`来充当它的`IPO`, 并与这些`IPO`分别建立三跳的一般Tor链路. 当某个`Client`需要连接此`HS`时,`Client`会首先从`HSD`处取得同一个`HS`关联的`IPO`的信息, 并把这个`IPO`视为一个一般的服务器来与其建立三跳链路, 通过这条链路来发送关于**客户端选择的`RPO`的信息**以及同`HS`的`握手数据`. 这些数据会经由`IPO`中转给`HS`. 
- `汇聚节点Rendezvous Point`: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>(后面简称RPO)</span> 由`Client`来选择的具有特殊功能的`OR`, 当一个`Client`需要连接某个`HS`时, 它会选择一个`OR`作为自己同`HS`的通信链路的汇聚点并与其建立三跳链路, `Client`将这个节点, 也就是`RPO`的信息通过`HS`的`IPO`告知给`HS`, 然后`HS`也会向`RPO`建立三跳链路. 由此形成的六跳链路就是最终`Client`和`HS`通信的链路.

<br>

<br>

## <span style="font-size:25px;font-weight:bold">1.2三跳访问的过程(用于访问一般的互联网网站)</span>

<img src="/images/tor-spec/2.png" alt="" style="zoom: 67%;" />

<div align='center' style='font-family:hei;font-weight:bold'>
    (Tor Client访问"维基百科"的过程)  
</div>

1. 用户开启了`Client`, `Client` 向`DS`请求当前活跃的`OR`的IP, 带宽等信息.
2. `Client`通过 <span id="jiaquanluyou" style='color:black;background:yellow;font-family:hei;font-weight:bold'>基于加权随机的路由选择算法</span> 来选择三个`OR`并建立三跳链路 (建立三跳链路的详细过程见[后面的"建立连接"部分](#part_4))
3. 访问数据在`Client`处被加密3次并发送给`Entry`<span style='font-family:hei;font-weight:bold'>(这三次只是Tor对访问数据的加密,事实上Tor算是应用层的协议, 而这些数据按照计网的五层架构向下封装时, 还会被TLS等常见协议进行加密, 如果算上这个的话, 数据其实是被加密了4次的)</span>
4. 访问数据在三个节点处各进行一次解密, 最后`Exit`解密获得明文数据, 得知这些数据是用来访问 "维基百科" 的, 并获得这个服务器的IP和端口, `Exit`就会去同 "维基百科" 的服务器建立HTTP连接并发起访问.
5. 服务器返回的数据在依次经过`Exit`,`Middle`,`Entry`的过程中会被它们使用各自同`Client`协商好的会话密钥层层加密,这样到达`Client`的数据又是经过了三层加密的「完整洋葱」, `Client`再使用自己的多个密钥对其进行依次解密.

<br>

<br>

## <span id= "part_1.3" style="font-size:25px;font-weight:bold">1.3六跳访问的过程(用于访问暗网网站)</span>

1. `HS`在选择三个`OR`作为自己的`IPO`, 并分别同它们建立三跳链路.
2. `HS`将自己的**服务描述符**上传到`HSD`, **服务描述符**的内容包括自己`IPO`的信息和 `RSA公钥`.
3. `Client`通过`HS`的域名`###.onion`进行访问时, 首先通过一个三跳链路访问`HSD`, 请求这个`HS`的`IPO`的信息和 `RSA公钥`.
4. `Client`选择一个`OR`作为自己的`RPO`, 并与其建立三跳链路.
5. `Client`建立到`IPO`的三跳链路, 并通过它将自己的`RPO`的信息以及`握手数据`发送给`IPO`, `IPO`再将这些信息发送给`HS`.
6. `HS`建立到`RPO`的三跳链路, 并对此链路进行认证, 同时通过`RPO`向`Client`响应自己的`握手信息`.
7. 至此, `Client`和`HS`可以通过这条由`RPO`沟通的六跳链路进行通信





<br>

***

<br>

# 2_Tor网络组成结构的细节补充

## 2.1目录服务器DS

<img src="/images/tor-spec/3.png" alt="哪些服务器可以充当DS" style="zoom: 67%;" />

<div align='center' style='font-family:hei;font-weight:bold'>
    (哪些服务器可以充当DS,来自官方文档)  
</div>

<span style='font-family:hei;font-weight:bold'>DS可以包括:</span>

- 权威的目录服务器,共9个,公开.
- 网桥目录服务器,匿名.
- 目录服务器的镜像(由一些普通的`OR`来充当),功能和权威的目录服务器一样,公开.
- 洋葱服务的目录服务器,也就是`HSD`,匿名

<br>

<span style='font-family:hei;font-weight:bold'>DS保存的内容包括:</span>

- 共识文档(consensus document): 当前Tor网络中每个`OR`的简要信息.
- 服务描述符(server descriptor): 当前Tor网络中每个`OR`的详细信息,包括公钥等.
- 额外信息(extra-info): 包括每个OR的补充信息 (例如历史时间戳等) .

>前面提到[Client使用加权路由选择算法来选择OR](#jiaquanluyou)依据的文件就是`共识文档(consensus document)`和`服务描述符(server descriptor)`, Client依据这两个文件提供的带宽信息和放缩因子计算各节点的加权值并按照`exit`,`entry`,`middle`的顺序选择节点 链路上任意两个节点应来自不同的c类网段



每个`OR`都会签署和更新自己的`服务描述符(server descriptor)`,并每18小时将其上传到9个半信任的**权威目录服务器**中的每一个..

然后这些**权威的目录服务器**会整合自己已知的所有`OR`的信息并形成一个`共识文档(consensus document)`. 9个**权威目录服务器**之间会根据特定算法相互交流, 最后形成一致的`共识文档(consensus document)`, 并各自对其签名.

活跃的`OR`或者接入Tor网络的`Client`每个小时从**权威目录服务器或者其镜像处**下载一次`共识文档(consensus document)`(有时只下载发生变化的部分) 和过期的`服务描述符(server descriptor)`

`DS`是公开的, 允许`OR`或`Client`通过直连链路, 以**HTTP**的方式请求服务.



<br>

<br>

## 2.2隐藏服务目录服务器HSD

<span id="jiaquanluyou" style='color:black;background:yellow;font-family:hei;font-weight:bold'>`HSD`并不是传统的中央服务器结构,而是DHT(分布式哈希表)结构</span>

><img src="/images/tor-spec/4.png" alt="分布式哈希表"  />

<img src="/images/tor-spec/5.png" alt="分布式哈希表(出自:「匿名通信与暗网研究综述」罗军舟等)" style="zoom:67%;" />

Tor网络中的`HSD`是由一系列稳定快速且可信的`OR`来组成的DHT结构. 这些`OR`在`DS`保存的`共识文档(consensus document)`中拥有特殊的flag标记.

`Client`通过要访问的URL:`"###.onion"`可以提取计算出该服务器的相关数据在哈希环上的存储位置, 然后去请求该位置上的`HSD`节点





<br>

***

<br>

# 3_Tor cell

## 3.1结构

<span id="jiaquanluyou" style='color:black;background:yellow;font-family:hei;font-weight:bold;font-size:25px'>Tor cell是Tor协议下通信的基本数据单元</span>

<img src="/images/tor-spec/6.png" alt="Tor cell的基本结构" style="zoom:67%;" />

其中, `Command`字段指示这个Tor cell要执行的功能, 其长度固定为1字节.

`Circuit ID`字段用于确认此Tor cell和哪一条Tor链路相关联

在Tor version1-3中, `Circuit ID`字段的长度为2字节, 这样以来, 一个Tor cell的大小固定为**512字节**.

在Tor version4之后, `Circuit ID`字段的长度为4字节, 一个Tor cell的大小固定为**514字节**.



一般来说, 如果一个Tor cell的长度不足, 则在payload部分进行填充, 直到这个cell的大小达到固定的512或者514字节

也有一部分特殊的Tor cell (由几个特殊的Command字段值来标识, 该值为**7或者大于128**) 大小是可变的

<br>

## <span id="jiaquanluyou" style='font-family:hei;font-weight:bold;font-size:25px'>3.2Command字段的值和相关含义:</span>

```
长度固定的Tor cell可以使用的Command值:
    0 -- PADDING     (Padding)         
用于链路的keep alive, 如果当前没有其他流量,则Client和OR之间或者OR和OR之间每隔一段时间就要发送PADDING cells       	       1 -- CREATE      (Create a circuit)   用于Client创建链路    
    2 -- CREATED     (Acknowledge create)     用于响应确认CREATE cell
    3 -- RELAY       (End-to-end data)        最常见,用于传输数据
    4 -- DESTROY     (Stop using a circuit)   
    5 -- CREATE_FAST (Create a circuit, no PK) 
    6 -- CREATED_FAST (Circuit created, no PK) 
    8 -- NETINFO     (Time and address info)   
    9 -- RELAY_EARLY (End-to-end data; limited)
    10 -- CREATE2    (Extended CREATE cell)    
    11 -- CREATED2   (Extended CREATED cell)   
    12 -- PADDING_NEGOTIATE   (Padding negotiation)
    
长度可变的Tor cell可以使用的Command值:
    7 -- VERSIONS    (Negotiate proto version) 
    128 -- VPADDING  (Variable-length padding) 
    129 -- CERTS     (Certificates)           
    130 -- AUTH_CHALLENGE (Challenge value)    
    131 -- AUTHENTICATE (Client authentication)
    132 -- AUTHORIZE (Client authorization)    (Not yet used)
```



**根据`Command`字段的值不同, 字节填充的模式也不同, 例如:**

- 7(VERSION): payload中不能包含任何额外的填充字节
- 3/9(RELAY/RELAY_EARLY): 填充的字节为随机值
- 132(AUTHORIZE): 没有使用, 如何设置填充字节也没有指明
- 其他固定长度的Tor cell: 填充字节设为`NUL`
- 其他可变长度的Tor cell: 填充字节设为`NUL`

<br><br>

## <span id="3.3" style="font-size:25px;font-weight:bold">3.3CREATE(D)/CREATE(D)2 cell</span>

这部分开始是[第四部分建立链路过程的补充知识](#part_4)

`CREATE(D)/CREATE(D)2 cell`一般用于相邻的两个节点之间用于建立链路的通信, 例如`Client`和`Entry`, `Entry`和`Middle`等等

`CREATE(D)2`是`CREATE(D)`的更新版本, 其格式是可扩充的

`CREATE2 cell`是由**握手的发起方**发送的, 其**payload**包含以下内容: 

```
HTYPE     (Client Handshake Type)     [2 bytes] 握手类型
HLEN      (Client Handshake Data Len) [2 bytes] 握手数据长度
HDATA     (Client Handshake Data)     [HLEN bytes] 握手数据
```

`CREATED2 cell`是由**握手的响应方**发送的, 其**payload**包含以下内容:

```
HLEN      (Server Handshake Data Len) [2 bytes] 握手数据长度
HDATA     (Server Handshake Data)     [HLEN bytes] 握手数据
```

<br>

其中`HTYPE`指示的握手类型包括:

```
0x0000  TAP  -- the original Tor handshake;
0x0001  reserved
0x0002  ntor -- the ntor+curve25519+sha256 handshake;
```

<br><br>

## 3.4RELAY cell

`REALY cell`用于端到端 (end to end) 的通信

`Client`可以向链路上的任何`OR`或服务器发送`REALY cell`, 其payload会使用每个途中经过的`OR`的会话密钥进行层层加密, 只有此cell的接收者才能最终解密得到明文的payload. 

结构: (**其中灰色部分是该`RELAY cell`的接收者才能最终解密看见的内容**)

<img id= "RELAY_cell" src="/images/tor-spec/8.png" alt="relay cell的基本结构" style="zoom:80%;" />

上图中可以看见, 除了Tor cell本身的 `Command`命令字段之外, `RELAY cell`内部还有一个自己的命令字段

其中,`STREAM ID`字段值一般由`Client`来选择, 通常是非0值, 如果该值为0,那么该流被视为是"控制流",即能够对整条链路的数据传输产生影响



<span id= "relay_command">`Relay Command`字段值一览:</span>

```
1 -- RELAY_BEGIN     [forward] 用于告知服务器开始传输应用数据
2 -- RELAY_DATA      [forward or backward] 用于传输应用数据
3 -- RELAY_END       [forward or backward]
4 -- RELAY_CONNECTED [backward] 用于服务器对RELAY_BEGIN进行响应
5 -- RELAY_SENDME    [forward or backward] [sometimes control]
6 -- RELAY_EXTEND    [forward]             [control]  
7 -- RELAY_EXTENDED  [backward]            [control]
8 -- RELAY_TRUNCATE  [forward]             [control]
9 -- RELAY_TRUNCATED [backward]            [control]
10 -- RELAY_DROP      [forward or backward] [control]
11 -- RELAY_RESOLVE   [forward]
12 -- RELAY_RESOLVED  [backward]
13 -- RELAY_BEGIN_DIR [forward]
14 -- RELAY_EXTEND2   [forward]             [control]
15 -- RELAY_EXTENDED2 [backward]            [control]

16..18 -- Reserved for UDP; Not yet in use, see prop339.

32-40 -- 用于IPO和RPO链路:
	  32 -- RELAY_COMMAND_ESTABLISH_INTRO
Sent from hidden service host to introduction point; establishes introduction point. 
            
      33 -- RELAY_COMMAND_ESTABLISH_RENDEZVOUS
Sent from client to rendezvous point; creates rendezvous point. 
            
      34 -- RELAY_COMMAND_INTRODUCE1
Sent from client to introduction point; requests introduction.
            
      35 -- RELAY_COMMAND_INTRODUCE2
Sent from introduction point to hidden service host; requests introduction. Same format as INTRODUCE1. 
            
      36 -- RELAY_COMMAND_RENDEZVOUS1
Sent from hidden service host to rendezvous point; attempts to join host's circuit to client's circuit. 

      37 -- RELAY_COMMAND_RENDEZVOUS2
Sent from rendezvous point to client; reports join of host's circuit to client's circuit.

      38 -- RELAY_COMMAND_INTRO_ESTABLISHED
Sent from introduction point to hidden service host; reports status of attempt to establish introduction point. 
      39 -- RELAY_COMMAND_RENDEZVOUS_ESTABLISHED
Sent from rendezvous point to client; acknowledges receipt of ESTABLISH_RENDEZVOUS cell. 

      40 -- RELAY_COMMAND_INTRODUCE_ACK
Sent from introduction point to client; acknowledges receipt of INTRODUCE1 cell and reports success/failure.


41-42 -- 用于链路填充,具体见第5部分

Used for flow control; see Section 4 of prop324.
43 -- XON             [forward or backward]
44 -- XOFF            [forward or backward]

```

<br>

### <span id="part_3.4.1">3.4.1 Digest, Recognized字段以及加密机制</span>

> `Digest`长度为4字节, `Recognized`长度为2字节
>
> 为保证端到端通信的完整性,`Client`会截取自己**运行摘要(running digest)**的前4字节,写入`RELAY cell`的`digest`字段. `OR`收到之后,会将其取出并同自己计算出的运行摘要来比较.
>
> `recognized`字段用于指示该cell是否仍是被加密的.  一般来说,如果该cell的`recognized`字段是非0值,**说明payload仍是被加密的**, 此时当前`OR`应该继续将该cell发送给下一个`OR` (**注意这样的检查有一定的概率出现错误, 因为此时recognized字段有2^-16的概率恰好为0**)
>
> 因此, 如果`recognized`字段的值为0, 为了避免可能发生的错误 (尽管概率很小) `OR`会继续检查`digest`字段的值是否匹配
>
> 如果`recognized`值为0,而`digest`值不匹配, 说明当前的`OR`并不是这个`RELAY cell`的接收者, 那么`OR`应该继续向下传递该cell
>
> <br>
>
> 假设`Client`想要给第三个`OR`(也就是`Exit`)发送 `RELAY cell`,那么`Client`会依次使用这个`Exit`以及其前面的`OR`的**会话密钥**迭代加密payload
>
> 该`RELAY cell`被发送到第一个`OR`时,第一个`OR`会对其进行第一次解密,**检查解密后的`Recognized`字段的值是否是0以及`digest`字段是否能够正确匹配**
>
> 如果不合要求, 则该`OR`会将解密后的payload部分(也就是[前面图中的灰色部分](#RELAY_cell))封装到一个新的Tor cell中(**也就是添加Tor cell最前面的`circuitID`和`Command`字段**)并向下转发.
>
> 直到第三个`OR`接收到这个cell,解密后检查`recognized`字段的值为0 , 并且`digest`字段能够匹配
>
> 此时这个`OR`会根据`Relay Command`的值以及`data`部分的内容来执行相应的操作
>
> <br>
>
> 从`Client`处进行被层层加密的是其实只是长度为**509B**的payload部分, Tor链路上的每一个`OR`都能够识别前面的`Circuit ID`和`Command`字段, 并且要对这两个字段进行检查和重新赋值(也就是重新封装**509B**的payload部分)
>
> 在此之前`Client`已经为**498B**的实际data payload加好了`RELAY`头,其中的`recognized`和`digest`字段只有本次传输的接收方能够正确识别
>
> 这个Tor cell到达每一个`OR`,并被该`OR`进行一次解密之后: 如果该`OR`不是目标节点,那么**509B**的payload**实际上仍然处于密文状态**, 那么这个`OR`实际上检查的是`recognized`和`digest`**这个位置上的字节值**(严格来说密文状态下这些字段是不存在的,只是到**相应位置**去检测), 当检查无法识别后,`OR`再重新为其增加长度为3B的tor头部字段(也就是`Circuit ID`和`Command`字段)
>
> (注意: 每个`OR`和`OR`之间的链路的`circuitID`都不同, 因此`OR`会在重新添加头部字段的过程中将`circuitID`换成它同下一个`OR`之间链路的`circuitID`)

<br>

由此其实可以看出:

对于一个512B的cell,其前3B的tor头部分(`circuitID`和`command`)是明文的,后面509B的payload(其中还包括relay头)是在tor的逻辑下进行加密的

这样的cell在按照计算机网络的五层结构向下封装时,**还会被TLS层进行加密**, 因此如果在传输链路上截获流量是无法识别出流量类型的, 但是如果攻击者控制了`entry`这样的节点,便可以提取此处经TLS解密后的流量,从而观察到3字节的tor明文头

<br>

### 3.4.2RELAY_EXTEND(2)/RELAY_EXTENDED(2)

`RELAY_EXTEND(2) cell`由`Client`发出, 用于**在建立链路阶段告知当前链路末端的`OR`去扩充链路长度** (和下一个`OR`建立连接)

例如`Client`向`Entry`发送`RELAY_EXTEND(2) cell`, 告知其去连接某个`OR`, 这个`OR`也就是`Client`正试图建立的链路上的`Middle`节点(此节点的信息会包含在这个cell的payload中),  `Entry`在解密得到明文的payload后, 就知道了应该去找那个`OR`建立连接, 并向这个`OR`发送命令字段为`CREATE`的 Tor cell. 

`RELAY_EXTENDED(2) cell`是在`OR`同下一个节点建立连接后, 对`RELAY_EXTEND(2) cell`的发出者的响应, 表示告知发出者已经按照其指示同下一个节点建立好了连接. 

<br>

`RELAY_EXTEND(2) cell`的payload包括以下内容: 

```
NSPEC      (Number of link specifiers)     [1 byte]
	NSPEC times:
		LSTYPE (Link specifier type)           [1 byte]
		LSLEN  (Link specifier length)         [1 byte]
		LSPEC  (Link specifier)                [LSLEN bytes]
HTYPE      (Client Handshake Type)         [2 bytes]
HLEN       (Client Handshake Data Len)     [2 bytes]
HDATA      (Client Handshake Data)         [HLEN bytes]
```

其中`link specifiers`描述了要连接的下一个节点的信息(IP,端口等),以及使用什么方式去连接它

另外, 这一部分是和`CREATE2 cell`的[payload](#3.3)是一致的

```
HTYPE      (Client Handshake Type)         [2 bytes]
HLEN       (Client Handshake Data Len)     [2 bytes]
HDATA      (Client Handshake Data)         [HLEN bytes]
```

当前链路末端会直接将这一部分封装到即将发给下一个节点的 `CREATE cell`中,作为其payload.

待扩展的`OR`收到`CREATE cell`后, 就可以利用客户端发送的握手数据来生成会话密钥并完成握手

随后,它会向之前的`OR`回复一个`CREATED cell`

之前的`OR`收到后,取出`CREATED cell`的payload,将其封装成`EXTENDED cell`,并将其发回`Client`

至此,`Client`同待扩展`OR`的握手完成, 链路也就扩展到了下一个节点上

<br>

### 3.4.3RELAY_BEGIN/RELAY_CONNECTED

由`Client`发出, 链路末端的`OR`(也就是`Exit`)接受, `Client`指示其去与要访问的服务器建立连接.

`RELAY_BEIGIN cell`的payload结构:

```
ADDRPORT [nul-terminated string] 包含目标服务器的IP和端口
FLAGS    [4 bytes] 
# FLAG的值和含义:
      1 -- IPv6 okay.  We support learning about IPv6 addresses and
           connecting to IPv6 addresses.
      2 -- IPv4 not okay.  We don't want to learn about IPv4 addresses
           or connect to them.
      3 -- IPv6 preferred.  If there are both IPv4 and IPv6 addresses,
           we want to connect to the IPv6 one.  (By default, we connect
           to the IPv4 address.)
      4..32 -- Reserved. Current clients MUST NOT set these. Servers
           MUST ignore them.
```

如果`Exit`无法找到目标server或者无法同目标server建立TCP连接,它会返回`RELAY_END cell`

如果成功建立连接,则返回`RELAY_CONNECTED cell`,其payload结构为:

```
       The IPv4 address to which the connection was made [4 octets]
       A number of seconds (TTL) for which the address may be cached [4 octets]
   or
       Four zero-valued octets [4 octets]
       An address type (6)     [1 octet]
       The IPv6 address to which the connection was made [16 octets]
       A number of seconds (TTL) for which the address may be cached [4 octets]
```

当连接建立成功之后,`Client`和server就可以通过`RELAY_DATA cells`来发送应用数据了

>在0.2.3.1-alpha版本之前,如果`exit`不支持"optimistic data"
>
>那么`Client`应该等待至收到`RELAY_CONNECTED cell` 后才能够开始发送`RELAY_DATA cells`
>
>后面的版本中,不必再有这个等待过程, `Client`发送完`RELAY_BEGIN cell`后可以立即开始发送`RELAY_DATA cells`



<br>

<br>

## 3.5REALY_EARLY cell

`REALY_EARLY cell`是一种特殊的`REALY cell`

**目前新版本的Tor规定, 携带`EXTEND`命令的 `REALY cell` 必须改为 `REALY_EARLY cell`** , 并且规定: 如果一个节点收到了累计8个以上`outbound`方向 (自客户端到服务器) 的`REALY_EARLY cell`,  或者收到了任何来自`inbound`方向 (自服务器至客户端) 的`REALY_EARLY cell` ,  则此节点应该**立刻关闭当前的连接**

因此可以看出, `REALY_EARLY cell`的意义是**防止恶意攻击者建立过长的链路以进行DOS等形式的攻击**, 那么如果某个`OR`收到了大于8个 `REALY_EARLY cell` 信元,就说明`Client`或者某些`OR`在恶意地通过发送`RELAY_EXTENDED(2) cell`来创建过长的链路, 从而耗尽Tor 网络的资源, 达到DOS攻击的效果

尽管这样做并不能完全地阻止攻击者创建过长的链路, 但是会增加其成本,攻击者需要至少在每8跳处指定一个恶意中继.(每连接7跳后,由该恶意中继继续构造`RELAY_EXTENDED(2) cell`并向下发送,因此前面构建的链路中的那些正常的`OR`并不能收到第8个`REALY_EARLY cell`,也就不会断开连接)



<br>

***

<br>

# <span id="part_4">4_建立连接和访问</span>

这里直接上一个画得特明白的图:

<img id= "RELAY_cell" src="/images/tor-spec/9.png" alt="Tor建立链路以及访问某服务器的过程"  />

在建立链路时, `Client`和`Entry`通过`CREATE2 cell`和`CREATED2 cell`来进行握手, 在这个过程中双方会生成一对会话密钥,这里称为`k1`

之后, `Client`为了同`Middle`进行握手建立连接, 它会向`Entry`发送`RELAY_EXTEND2 cell` , 和`Middle`的握手数据包含在这个cell的relay_payload中. `Client`使用`k1`加密了这个cell的payload(注意relay_payload和payload说的不是同一个东西), 因此只有`Entry`能够解密并识别其中的`RELAY_EXTEND2`指令.

`Entry`能够解密并识别了`RELAY_EXTEND2`指令, 取出relay_payload(也就是`Client`要发送给`Middle`的握手数据), 将其封装到一个`CREATE2 cell`中并发送给`Middle`

`Middle`会将自己的握手响应数据封装在`CREATED2 cell`中发给`Entry`, `Entry`再将其取出并封装到`RELAY_EXTENDED2`中发回给`Client`. 在此过程中, `Client`和`Middle`就协商得到了会话密钥`k2`

`Client`同`Exit`握手的过程同理, 只不过从`Client`发出的`RELAY_EXTEND2 cell`的payload会先使用`k2`加密, 再使用`k1`加密, 这样`Entry`会先使用`k1`对其进行一次解密, 发现仍然无法识别后([参考3.4.1](#part_3.4.1)), 就会将其继续发送给`Middle`, `Middle`使用`k2`再进行一次解密后, 发现可以识别到`RELAY_EXTEND2 cell`, 就会向前面的过程一样去找到`Exit`并向其发送`CREATE2 cell`.

在链路建立完成时, `Client`拥有三个会话密钥, 分别对应`Entry`,`Middle`和`Exit`

那么,当`Client`准备通过建立好的链路请求某个服务是, 它发送的请求包就是这种形式:(只有`Exit`能够最后解密得到目标服务的IP和端口,并由此发起请求)

<img id= "RELAY_cell" src="/images/tor-spec/10.png" alt=""  />

<br>

<br>

访问`HS`建立连接的过程见[1.3](#part_1.3)

`HS`在创建`IPO`链路时, 先按照常规流程建立三跳链路, 然后向最后一个`OR`发送一个 命令字段为`RELAY`, relay的命令字段为`establish_intro` ((relay命令字段值为32) 的cell (参考[RELAY_command字段值列表](#relay_command))

然后该`OR`回复一个relay命令字段为`intro_established` (relay命令字段值为38) 的cell, 如此, 该`OR`就成为了这个`HS`的 `IPO`

同理, `Client`在建立`RPO`链路时也是一样的过程, 建立三跳链路后向最后一个节点发送relay的命令字段为`establish_rendezvous`((relay命令字段值为33)的cell. 

然后该`OR`回复一个relay命令字段为`redezvous_established` (relay命令字段值为39) 的cell, 如此, 该`OR`就成为了这个`Client`的 `RPO`

<br>

>当`Client`访问`HS`时,
>
>`Client`和`HS`共同持有一对会话密钥`k'`, 在协商这对密钥过程中, `Client`的握手信息是通过`IPO`链路发送给`HS`的, 而`HS`的握手信息是通过`RPO`链路发给`Client`的.(参考[1.3](#part_1.3)).  `Client`向`HS`发送信息时,首先使用此密钥进行加密(**也就是说,此为最内层的加密**)
>
>`Client`持有`k'`,`k1`,`k2`,`k3` HS持有`k'`,`k4`,`k5`,`k6`(**对应的是用于传输应用数据的`RPO`链路上的六个节点**)
>
>也就是说,Tor cell在`Client`处会被加密四层, 当Tor cell到达`RPO`并经过第三次解密后仍是被`Client`和`HS`的密钥加密的
>
>随后,`RPO`将Tor cell继续转发,`OR4`,`OR5`,`OR6`会使用各自的密钥对其**加密** (这个过程相当于`RPO`是三跳链路中的服务端, 而`HS`是三跳链路中的客户端), 所以Tor cell到达`HS`时,又是被4层加密的`,HS`会依次使用`k6`,`k5`,`k4`,`k'`对其逐层解密,获得明文.
>
> `HS`给`Client`发数据也是类似的过程


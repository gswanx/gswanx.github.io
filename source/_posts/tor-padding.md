---
title: TOR的防御机制PADDING
description: '通过协商发送特殊数据包来混淆流量特征'
cover: false
abbrlink: 212eacc4
date: 2022-08-22 11:42:17
tags:
  - Tor
categories:
  - Tor
typora-root-url: ..

---





<span id="jiaquanluyou" style='color:black;background:yellow;font-family:hei;font-weight:bold;font-size:25px'>前置的一些基础原理见: [TOR原理](75310f08.html)</span>

# 0_概述



[TOR原理](75310f08.html)那篇文章的第三部分已经说明过, TOR cell是TOR中的基本通信单元, 一个TOR cell的大小固定为**512**或**514**字节

尽管TOR的机制能够有效地保护通信双方的身份以及通信内容不被泄露, 但是其实很早就有研究提出可以通过指纹攻击的方式来从流量模式 (**例如数据包的序列特征**) 入手来检测出TOR流量了. 对于监管者 (例如运营商) 来说, 其实也并不用彻底掌握用户到底在暗网中看到了啥不该看的内容, 只需要简单地检测出有暗网流量产生, 然后再把它BAN掉就行了. 因此这类能够有效检测出TOR流量的指纹攻击手段确实对TOR的安全性产生了不小的威胁. TOR必须采取一定的应对手段来进一步提升自己的安全性. 

>•Kwon, Albert et al. “Circuit Fingerprinting Attacks: Passive Deanonymization of Tor Hidden Services.” *USENIX Security Symposium* (2015).
>
>这篇文章利用简单的决策树算法, 依赖的特征仅仅是`Client`或`HS`在构建`IPO`或`RPO`链路时收发携带不同`Command`的Tor cell的顺序关系, 就能够检测出当前链路的类型(**Client端IPO/ Client端RPO/ HS端IPO/ HS端RPO**)
>
>当时也是立马引起了Tor官方的反应:
>
><img src="/images/tor-padding/image-20220822120634208.png" alt="image-20220822120634208" style="zoom:67%;" />
>
>很快, Tor官方就提出了应对手段, 并在0.4.1.1-alpha版本里首次实现, 也就是本篇讨论的重点:  **`Padding`机制**.



`Padding`说白了就是在正常的Tor流量中混入一些特殊且无用的数据包(Tor cell),  这些数据包由`Client`和 某个Tor链路上的一个`OR`协商后互相发送和接收 , 以破坏在`Client`和这个`OR`之间截获流量的观察者所看到的「Tor流量在数据包分布等方面的特征」, 这些刻意发送的数据包被称为 `Padding cell`.  

目前官方提出了两种`Padding`方式: `connection-level padding`和`circuit-level padding`, 分别应对处在不同位置的攻击者:



<div align='center' style='font-family:hei;font-weight:bold'>
    (下图中上方的链路为IPO链路, 下方的链路为RPO链路)
</div>

<img src="/images/tor-padding/image-20220822153045204.png" alt="image-20220822153045204" style="zoom:67%;" />

对于`connection-level padding`来说,后者就是普通的数据流量, 为了有所区分,当`circuit-level padding`工作时,`connection-level padding`不工作. 

<br>

***

<br>



# 1_connection-level padding

此种`Padding cell`的 `Command`字段值为**0**(`PADDING`)  (见[TOR原理3.2](75310f08.html)) , 一般由`Client`和`Entry`之间协商后发送和接收.

`connection-level padding`较为简单,  应对在`Client`和`Entry`之间控制了任何普通的非Tor网络节点的攻击者, 这些截获流量的攻击者因为没有Tor链路上`OR`之间协商的密钥, 能看见的只是被下层的TLS等协议加密后的密文流量.  

而与此相对, 即使不是Tor cell的接收者, 一个Tor链路上的`OR`也能够解读出一个Tor cell的前3字节内容, 这部分内容包含2字节的`Circuit ID`和1字节的`Command`, 能够解读`Command`字段也就意味着`OR`都能够识别出此类`Padding cell`. 因此, 如果一个攻击者控制了Tor链路上`Entry`这样的节点, `connection-level padding`也就失效了. 





<img src="/images/tor-padding/image-20220822154318021.png" alt="image-20220822154318021" style="zoom:67%;" />

在`Client`和`Entry`之间应用`connection-level padding`时:

两个节点共同持有一个关于TLS连接应用数据的计时器. 每当两端收到一个正常的数据包,它们都会根据一个特定的超时分布在1.5s至9.5s之间取一个**超时值**, 如果在这个超时值之前收到了下一个携带真实数据的Tor cell,那么重新选择该值; 如果在此之前没有收到真实的Tor cell,那么将会发送`Padding cell`

由此以来, 在不超过10s的超时时间到达之前, 链路上总会有真实的Tor cell或者`Padding cell`被发送, 那么分不清真实Tor cell和`Padding cell`的链路上的攻击者就无法通过数据包间隔时间等特征来记录流量模式

<br>

***

<br>

# 2_circuit-level padding

假设攻击者控制了Tor 链路上的`Entry`节点. 那么他能够看见在此被TLS解密后的Tor cell, 并读取每个Tor cell的前3字节数据. 也就能识别并丢弃来自`connection-level padding`的`Padding cell`

通过记录流量特征, 攻击者还能够察觉当前链路的类型, 是访问互联网服务的三跳链路, 还是用于访问`HS`的`IPO`或`RPO`链路

`circuit-level padding`的目的就是避免控制了`Entry`的攻击者识别出链路的类型

<img src="/images/tor-padding/image-20220822161841331.png" alt="image-20220822161841331" style="zoom:67%;" />

`circuit-level padding`由`Client`和`Middle`互相发送和接收`Padding cell`, 这里它们用于发送`Padding cell`的程序视为一个状态机, 称为`Padding Machine`

此种`Padding cell`的`Command`字段值为**3**(`RELAY`), `Relay Command`的值为10(`Drop`) ,对于只能在在`Entry`处观察到Tor cell的前3字节的攻击者来说, 此种cell和一般的Tor cell无异.

<br><br>

## 2.1PADDING协商

执行`circuit-level padding`的是部署在`Client`和`middle`上的**padding状态机**

在进行padding之前, 需要通过发送携带特殊的`Relay Command`字段值(**41/42)**的`RELAY cell`来进行PADDING协商:

<img src="/images/tor-padding/1.png" alt="padding_relay_cell" style="zoom:67%;" />

分别是由padding的发起方发送的`PADDING_NEGOTIATE_cell`(41)和由响应方发送的`PADDING_NEGOTIATED_cell`(42)

其中,请求方发送的`PADDING_NEGOTIATE_cell`的payload中包含以下内容:(下面的结构体struct)

<img src="/images/tor-padding/2.png" alt="PADDING_NEGOTIATE_cell_payload" style="zoom:67%;" />

其中各个字段:

- command = `「CIRCPAD_COMMAND_START」(2)`

- machine_type = `「CIRCPAD_MACHINE_CIRC_SETUP」(1)` **此值用于标记发送它的是发起padding的一方**

- michine_ctr : 当前链路上`machine instance`的数量, 用于消除关闭请求的歧义,如果该字段不符合事实,那么请求将被忽略

  <br>

当接收方收到`PADDING_NEGOTIATE_cell`时, 首先检查请求方请求的`version`等自己是否支持, 然后发送`PADDING_NEGOTIATED_cell`,其payload包含以下内容:

<img src="/images/tor-padding/3.png" alt="PADDING_NEGOTIATED_cell_payload" style="zoom:67%;" />



在收到`PADDING_NEGOTIATE_cell`之前, `Client`就可以开始发送`Padding cell`了



<br><br>

## 2.2对IPO链路的混淆

>tor中,访问互联网网站(非`HS`)的通信,cell序列一般遵循以下模式: **创建三跳链路 - 请求服务 - 交互应用数据**
>
> ([DATA]表示由`Client`发送给`Server`的Cell, DATA相反方向的Cell), padding机制需要尽可能把`IPO`和`RPO`链路上的cell序列伪装成此种模式:
>
><img src="/images/tor-padding/4.png" alt="Normal Circuit" style="zoom:67%;" />

一般来说, `IPO`链路上的Cell序列表现为:

<img src="/images/tor-padding/5.png" alt="IPO circuit" style="zoom: 80%;" />

和一般链路对比可以看出, 前面通过`RELAY_EXTEND cell`建立链路的阶段是相似的, 主要需要将`[INTRO1]`和`[INTRODUCE_ACK]`这里混淆成这种形式: 

<img src="/images/tor-padding/6.png" alt="DATA" style="zoom: 80%;" />

在`[INTRO1]`和`[INTRODUCE_ACK]`之间发送`PADDING_NEGOTIATE_cell`和`PADDING_NEGOTIATED_cell`进行padding协商:

<img src="/images/tor-padding/7.png" alt="NEGOTIATE" style="zoom: 80%;" />

在此之后, `Middle`会给`Client`发送7-10个`Padding cell`(**RELAY_drop**), 以模拟正常通信中服务器发回数据的过程

<br><br>

## 2.3对RPO链路的混淆

`RPO`链路的混淆目标也是一般的Tor三跳链路:

<img src="/images/tor-padding/4.png" alt="Normal Circuit" style="zoom:67%;" />

一般来说, `RPO`链路上的Cell序列表现为:

<img src="/images/tor-padding/8.png" alt="RPO Circuit" style="zoom: 80%;" />

添加`Padding cell`(**DROP**)后:(前面提过,`client`在收到`PADDING_NEGOTIATED_cell`之前就能开始发送`Padding cell`)

<img src="/images/tor-padding/9.png" alt="RPO PADDING Circuit" style="zoom: 80%;" />

在此之后,由`RPO`方向正常传回来自于`HS`的应用数据即可(不再需要`Padding cell`)

因为当`RPO`链路完全建立完成时,链路上会有来自隐藏服务器端的应用数据,对于无法看到payload的攻击者来说,来自`HS`的正常cell和`Padding cell`没有区别,因此此时padding可以停止.

<br><br>

## 2.4Padding Machine的工作机制

`Padding Machine`是一种状态机, 工作时根据实际情况在不同状态之间进行转换

首先定义数据流有两种状态:`Burst`和`gap`

<img src="/images/tor-padding/10.png" alt="Burst and Gap" style="zoom: 80%;" />

- `Burst` : 一段在短时间内集中发送的数据包序列(数据包的密度较大),,此时`Padding Machine`处于`burst mode`, padding的需求较小, 在发送`Padding cell`之前需等待更长的时延.
- `Gap` :一段时间跨度较长的数据包序列(数据包的密度较小),此时链路上的流模式特征性更强, 很可能被攻击者记录并分析, 因此padding的需求较大,`Padding Machine`处于`gap mode`,仅等待较短的时延后就可以发送`Padding cell`.

<br>

每个`Padding Machine`都会保存两个直方图`HG`和`HB`分别用于在`gap mode`和`burst mode`下选择合适的发送`Padding cell`的等待时延:

<img src="/images/tor-padding/11.png" alt="Histogram" style="zoom: 80%;" />

其横轴为时延值, 纵轴为`Tokens`, 表示在某个时延范围(`Bin`)中选择一个合适的具体时延值的可能性

<br>

`Padding Machine`的状态转移示意:(其中的S,B,G分别表示其实状态,`burst mode` 和`gap mode`)

<img src="/images/tor-padding/12.png" alt="Histogram" style="zoom: 80%;" />

- **burst mode:**

在起始状态(S)下,`client`正不断接收从服务器发回的真实数据,此时进入`burst mode`, 系统根据`HB`(`burst mode`对应的直方图)所给出的概率分布来选择一个发送时延`t` (此直方图是根据分析大量网络流量而形成的)(来自上一个`burst`的结束和下一个`burst`的开始之间的时间差的取样)

当此节点不断地处理真实数据包,系统会不断地根据`HB`来选择新的发送时延,不断重复, 系统会停留在`burst mode`

如果在时延`t`的范围内没有处理到真实数据包,那么`Padding cell`将被发送,系统进入`gap mode`

综上,在`burst mode`期间,如果发现了一个大于所选延时`t`的`IAT(inter-arrival time)` ,那么当前选择的延时`t`将失效,并且触发`gap mode`

 <br>

- **gap mode:**

在,通过`HG`来选择一个时延`t'`,并且在时延结束时发送`Padding cell`, `HG`同样是通过对大量站点流量中`burst`内的`IAT(inter-arrival time)`取样来构建的(这是为了模拟正常`burst`中的`IAT`)

当接收到一个真实的数据包,或者在`HG`中选择到了一个来自`infinity bin`(上上图中最右侧的那个bin)范围中的时延值

此时系统会从`gap mode`转向`burst mode`

 <br>

当在`burst mode`中,当从`HB`中选择到了一个来自`infinity bin`范围之中的时延值时,转向起始状态(S) (这种情况说明根据真实流量取样时,选择到了一个较大的`IAT`值,说明真实的通信双方当前已经停止通信或者处于其他数据包极其稀疏的阶段)

 

综上可知, Padding机制不会影响对真实的数据包的处理或转发(不应用时延), 处理一个真实的数据包意味着当前选择的时延值失效,

此时还会进行的操作: 在`HB`或`HG`中将`Token`返还给当前失效的时延值对应的`Bin`,并且从真实时延值对应的`Bin`处移除一个`Token`, 以此来校正分布

个人理解:如果收到真实数据包导致时延值失效,那么当前失效的时延值肯定大于收到这个包和收到上个包之间的`IAT`(也就是当前真实的时延),这说明当前仍处在频繁收到真实包的阶段,**选择一个时延值时会"暂时拿走"一个其对应`Bin`的`Token`**,此时如果不校正的话,高时延`Bin`的`Token`减少了一个,那么下次选择时,选择到更低时延的`Bin`的可能性会增加,多次循环后,可能会导致系统倾向选择到极低的时延值,导致在`burst mode`还没来得及接收真实数据包,就进入了`gap mode`,最终导致真假数据包混合.  因此,校正操作是归还之前"暂时拿走"的`Token`,并移除一个更短的真实时延对应`Bin`的`Token`, 那么下次选择就不会趋向于选择更短时延的`Token`

 

如果一个`Bin`的`Token`耗尽,那么在移除时会移除与之相邻的更大`Bin`的`Token`

如果所有`Bin`的`Token`都耗尽,那么`HB`或者`HG`返回初始状态

<br><br>










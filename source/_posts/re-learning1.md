---
title: re_learning1
abbrlink: ca48639a
date: 2022-10-29 20:18:20
cover: false
tags:
  - 逆向
categories:
  - 逆向
typora-root-url: ..
---

# IDA





# 动态调试

汇编语句级别的跟踪

在程序内部设置端点,行行跟踪程序执行, 进入或者略过一个函数, 查看各个变量的值(寄存器,栈等等)

常用工具: `OllyDBG`: 仅支持32位



`x64DBG`:32/64位





# 简单脱壳

有些壳注重代码压缩: `UPX`,`ASPack`等

有的壳注重代码保护(加密壳):`VMP`,`ASProtect`等



## UPX壳:

### 静态脱壳: 

`UPX`本身提供脱壳器

<img src="/images/re-learning1/image-20221029215225406.png" alt="image-20221029215225406" style="zoom:67%;" />

但是UPX是基于加壳后的可执行文件内存储标识来查找并操作的, 这些标识可以任意更改, 这会导致官方加壳器加壳失败

### 动态脱壳

- `UPX`的**保护现场**: 在可执行文件被OS载入开始执行之前, 壳程序为了保护当前寄存器等位置以被设置好的数据(防止它们被壳运行的过程更改) 会将这些数据入栈处理.  **在壳将控制权转交给主程序之前, 再恢复这些数据**

  <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>x86的汇编指令`pushad`可以一次性地将所有寄存器入栈</span>

  #### 一次简单的UPX动态脱壳

  加了壳的程序乱糟糟一片,放到`ida`中也无法分析

  <img src="/images/re-learning1/image-20221029230634992.png" alt="image-20221029230634992" style="zoom:80%;" />

  

  将加壳后的程序拖进`OllyDBG`中运行

  <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>下面OD的页面中，左上为反汇编窗口，左下为浏览程序内存数据的区域，右上为寄存器，右下为栈</span>

![image-20221029223511208](/images/re-learning1/image-20221029223511208.png)

可以看到这里开头就是用于**保护现场**的`pushad`指令

这里先`F8`单步执行，然后在寄存器区域右键－`HW break[ESP]`设置**硬件读取断点**

然后`F9`运行程序直到断点处,这里停在了`lea eax,dword ptr ss:[esp-0x80]处`

![image-20221029223846726](/images/re-learning1/image-20221029223846726.png)

这里`lea`指令用于取地址, 这里是将 `ss:[esp-0x80]`这个地址(注意是地址值本身)赋给`eax`

这个地址是当前栈顶的位置(`ss:sp`指向栈顶)再向上`0x80`个长度

然后下面三句指令, 

````
push 0x0   将0x0入栈(栈顶上移,也就是esp的值-1)
cmp esp,eax   比较esp和eax的值,影响各个标志位的值,这里实际上是用esp的值减去eax的值,如果两者不相等,则结果不为0,那么ZF标志位的值也就为0(非0)
jnz short 3-UPX_pa.00D72083    jnz:ZF标志位为0则跳转(ZF标志位标识执行结果是否为0),这里跳转回push 0x0的位置


以上三句就形成了一个循环, 一直将0x0入栈并减小esp的值,直到esp=eax
因为之前已经让eax = esp-0x80(地址值)
所以上面三句完成的操作是将栈空间向上清零了0x80
````

接下来:

```
jmp 3-UPX_pa.00D44DDC
从图中可以看到,这句跳转到了一个离图中指令地址较远的位置
```

因为壳程序和原程序一般不在同一个区段,这里实际上就是跳转到原程序

现在取消断点:

![image-20221029230730548](/images/re-learning1/image-20221029230730548.png)

然后将光标移动到`jmp 3-UPX_pa.00D44DDC`处, 按`F4`让程序运行到光标处

然后再按F8单步执行,程序跳转,此时发现程序跳转到了正常程序的开头位置:

![image-20221029231126136](/images/re-learning1/image-20221029231126136.png)



这里能看到函数的开头和结尾(`call`和`retn`)

现在对程序进行`Dump`: 菜单-`Plugin`-`OllyDump`-脱壳当前调试进程

<img src="/images/re-learning1/image-20221029231458633.png" alt="image-20221029231458633" style="zoom:67%;" />



这里可以发现入口点地址这里和看到的也有点不一样,而且此时脱壳会报错:

<img src="/images/re-learning1/image-20221029233057352.png" alt="image-20221029233057352" style="zoom:67%;" />

内存为只读。权限不够。这里主要是因为插件比较老,win7/10等X64平台中基址不同于XP等X32中的.

这里搞个xp虚拟机来弄






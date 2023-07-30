---
title: "Typora写markdown的慢慢入门\U0001F625"
tags: hexo
categories: 搭建博客
cover: false
typora-root-url: ..
abbrlink: 39e7fa88
description: 主要是markdown的基本语法
date: 2022-07-22 09:30:28
---



<span id="jump">这是一个锚点,是后面页内链接要跳转到的地方,这里的id可以自定义,用来和后面的链接对应</span>



# 1这是代码

直接连敲三下`(在键盘的tab上方)就可以打开下面这样的代码环境

```go
package snowflake_proxy
import (
    "fmt"
    "io"
    "log"
    "net"
    "regexp"
    "sync"
    "time"
    "git.torproject.org/pluggable-transports/snowflake.git/v2/common/event"
    "github.com/pion/ice/v2"
    "github.com/pion/sdp/v3"
    "github.com/pion/webrtc/v3"
)
var remoteIPPatterns = []*regexp.Regexp{
    /* IPv4 */
    regexp.MustCompile(`(?m)^c=IN IP4 ([\d.]+)(?:(?:\/\d+)?\/\d+)?(:? |\r?\n)`),
    /* IPv6 */
    regexp.MustCompile(`(?m)^c=IN IP6 ([0-9A-Fa-f:.]+)(?:\/\d+)?(:? |\r?\n)`),
}

```

***

<br>



# 2 自定义文字的表现效果

## 2.1 通过HTML元素

```html
<span style='color:blue;background:yellow;font-size:18px;font-family:hei'>测试字体</span>
```

<span style='color:blue;background:yellow;font-size:18px;font-family:hei'>测试字体</span>



```html
<div align='center' style='color:blue;font-size:18px;font-family:hei'>
    我是一个独立的块,上面的align='center'可以设置这个块里的文字居中  
</div>
```

<div align='center' style='color:blue;font-size:18px;font-family:hei'>
    我是一个独立的块,上面的align='center'可以设置这个块里的文字居中  
</div>
<br>

## 2.2 文字加粗

**文字加粗效果(在两边各加两个星号)**

```HTML
**文字加粗效果**
```

或者:

在`span`内通过:(尽量多用这种,因为markdown语法在转换成html时可能会出问题)

```html
<span style="font-weight:bold">被加粗的文字</span>
```



<br>



## 2.3 分割线

(输入三个星号自动变成分割线):

***

<br>

## 2.4 删除文字

在文字两边各加两个~

```
~~我被删除了!~~
```

~~我被删除了!~~

<br>

## 2.5 下划线

通过u标签包裹实现

```html
<u>我带下划线</u>
```

<u>我带下划线</u>

<br>

## 2.6 脚注

对我的补充说明,鼠标悬停在后面就会显示这些补充信息 [^脚注显示的文字]。

[^脚注显示的文字]: 要显示的补充信息写在这里

<br>

***

<br>

# 3 列表

## 3.1 无序列表

- 在一行的开头按下 减号"-" 然后按空格就可以产生列表了
- 回车自动生成下一项
  - 回车后tab生成二级列表

<br>

## 3.2 有序列表

1. 输入"1."然后按空格,自动生成有序列表
2. 输入回车会自动往下排
   1. 回车后tab生成二级列表

<br>

## 3.3 任务列表

```
先输入后面的文字
在前面输入"-"和"[]"
在两者之间加上空格
然后在[]中输入x
最后在[]和文字之间加上空格

- [x] 任务1
```

- [x] 今天学会搭建博客

- [x] hhh

<br>

***

<br>

# 4 区块

> 输入">" 然后按空格,自动产生一个区块
>
> 区块是可以嵌套的:
>
> > 再次输入">" 然后按空格
> >
> > 我是第二层区块
> >
> > - 在区块里面同样可以使用列表,我是插入在区块里的列表
> >
> > - 同理,在列表里也可以使用区块:
> >
> > - > 我是列表里的区块
> >   >
> >   > 哈哈哈哈哈



<br>

***

<br>

# 5 链接

## 5.1 链接到网址

```markdown
[链接名称](链接地址 "可选项,悬停时显示的文字")
```

例如: 

在编辑器里ctrl点击链接以打开链接

[我可以链接到菜鸟教程的链接页面](https://www.runoob.com/markdown/md-link.html "我是悬停文字")



也可以直接显示链接地址:

```  html
<链接地址>
```

例如:<https://www.runoob.com>

<br>

## 5.2 链接到同目录文件

链接到同目录其他md文件:

```markdown
[我可以链接到同目录下的hello_world.md](hello-world.md)
```

[我可以链接到同目录下的hello_world.md](hello-world.md)

<br>

## 5.3 页面内跳转

<br>

### 5.3.1 跳转到锚点

[由此跳转到](#jump)

```markdown
在某处定义锚点: <span id="自定义">这是一个锚点,是后面页内链接要跳转到的地方,这里的id可以自定义,用来和后面的链接对应</span>

用来跳转到锚点的链接 [由此跳转到](#自定义)
```

<br>

### 5.3.2 跳转到标题

[由此跳转到](#1这是代码)

踩坑记录: 像这样使用链接时,**被跳转的标题中不能有空格!!!!!**

例如如果链接是这样:

```
[由此跳转到](#1 这是代码)
```

那么在生成HTML后无法将其转化为链接

<br>

正确的形式:

```markdown
[由此跳转到](#1这是代码)
```

不过这样也太容易出问题了,以后还是得采用多采用跳转到锚点的方式

***

<br><br>



# **6 图片**

直接插入图片:

(由于在typora中配置了插入图片时图片的存储规则,将图片复制进来时,图片会自动存储到source/images/与此md文件同名的文件夹)



![image-20220722114856557](/images/markdownlearn/image-20220722114856557.png)



```markdown
![image-20220722095454051](/images/markdownlearn/image-20220722095454051.png)
```

![image-20220722095454051](/images/markdownlearn/image-20220722095454051.png)



<span id="s1">使用html来加载图片</span>

好处是可以调整图片大小,加图片标题等

```html
<div align='center'>
  <img src='/images/markdownlearn/image-20220722095454051.png' width=600px>
  <p align='center' style='font-size:15px;font-family:kaiti;color:black'>图片标题 </p>
</div>
```

<div align='center'>
  <img src='/images/markdownlearn/image-20220722095454051.png' width=600px>
  <p align='center' style='font-size:15px;font-family:kaiti;color:black'>图片标题 </p>
</div>

<br>

**关于typora编辑器和部署后浏览器中图片无法同时正确显示的问题**:

部署后,默认的根目录是public目录

此时图片的存储路径规则是 /images/markdown文件名/图片名 , 所以如果在编辑器中以这种路径模式描述图片,图片就能够在浏览器中正确显示

而在生成public之前,一个md文件中的图片默认是存储在 `../../source/images/markdown文件名`这个目录下的

此时去下图位置将source设置成图片根目录,就能够保持编辑器和浏览器中图片路径一致,而且还能正确显示了

<img src="/images/markdownlearn/image-20220722163205824.png" alt="image-20220722163205824" style="zoom: 67%;" />

<br>

<br>

***

<br>

# **7 表格**

直接插入表格即可(竟然也不支持合并单元格)

有合并单元格的需求的话,还得使用html的表格:

例如:

表格用整体table包裹

每一行用**tr**包裹(row)

表头中的文字用**th**包裹

具体单元格中的文字用**td**包裹

合并列用 colspan="合并的列数"

合并行用 rowspan="合并的行数"

```html
<table>
    <tr> 
        <th>班级</th><th>课程</th><th>平均分</th>
    </tr>
    <tr>
        <td rowspan="3">1班</td><td>语文</td><td>95</td>
    </tr>
    <tr>
        <td>数学</td><td>96</td>
    </tr>
    <tr>
        <td>英语</td><td>92</td>
    </tr>
</table>
```

<table>     <tr>         <th>班级</th><th>课程</th><th>平均分</th>     </tr>     <tr>         <td rowspan="3">1班</td><td>语文</td><td>95</td>     </tr>     <tr>         <td>数学</td><td>96</td>     </tr>     <tr>         <td>英语</td><td>92</td>     </tr> </table>

<br>

实践:

**注意!!! 列与列之间不要有空行**

```html
<table>
    <tr> 
        <th>交互双方</th><th>交互方式</th><th>具体情况</th>
    </tr>
    <tr>
        <td rowspan="4">client向broker注册</td>
        <td rowspan="3">HTTP POST<br>(SDP被包含在request body中)<br>(向broker下的'/client发送请求')</td>
        <td>Client->Broker:<br>```<br>POST /client HTTP<br>[offer SDP]<br>```</td>
    </tr>
    <tr>
        <td>Broker->Client<br>若成功匹配到一个proxy,则返回proxy的SDP:<br>```<br>HTTP 200 OK[answer SDP]<br>```</td>
    </tr>
    <tr>
        <td>Broker->Client<br>若没有匹配到可用的proxy,则返回503</td>
    </tr>
    <tr>
        <td>AMP(Accelerated Mobile Pages)<br>The broker's /amp/client endpoint receives client poll messages encoded into the URL path, and sends client poll responses encoded as HTML that conforms to the requirements of AMP (Accelerated Mobile Pages). This endpoint is intended to be accessed through an AMP cache, using the ampcache option of snowflake-client.</td>
        <td></td>
     </tr>
</table>
```

<table>
    <tr> 
        <th>交互双方</th><th>交互方式</th><th>具体情况</th>
    </tr>
    <tr>
        <td rowspan="4">client向broker注册</td>
        <td rowspan="3">HTTP POST<br>(SDP被包含在request body中)<br>(向broker下的'/client发送请求')</td>
        <td>Client->Broker:<br>```<br>POST /client HTTP<br>[offer SDP]<br>```</td>
    </tr>
    <tr>
        <td>Broker->Client<br>若成功匹配到一个proxy,则返回proxy的SDP:<br>```<br>HTTP 200 OK[answer SDP]<br>```</td>
    </tr>
    <tr>
        <td>Broker->Client<br>若没有匹配到可用的proxy,则返回503</td>
    </tr>
    <tr>
        <td>AMP(Accelerated Mobile Pages)<br>The broker's /amp/client endpoint receives client poll messages encoded into the URL path, and sends client poll responses encoded as HTML that conforms to the requirements of AMP (Accelerated Mobile Pages). This endpoint is intended to be accessed through an AMP cache, using the ampcache option of snowflake-client.</td>
        <td></td>
     </tr>
</table>
<br>

***

<br>

# 8 使用HTML标签

对于 HTML 的块级元素 div、table、pre 和 p，请在其前后使用**空行**与其它内容进行分隔。尽量**不要使用制表符（tabs）或空格（spaces）对 HTML 标签做缩进**，否则将影响格式。

在 HTML 块级标签内不能使用 Markdown 语法。

例如下面一个p区块中的加粗语法将不起作用:

```
 <p>italic and **bold**</p> 
```

 <p>italic and **bold**</p> 

***

<br>

## 8.1 常用标签:

### div区块

定义一个区域部分,在其中使用p标签来区分段落

可选项: 

- align: 此区块的位置: left/right/center
- style:
  - font-size:15px 字体大小
  - font-family:kaiti 字体
  - color:black 颜色
  - background-color:#FFFF00  背景颜色,这里的代码是黄色

```html
<div style="color:blue">   
    <p>这是一个在 div 元素中的文本。</p> 
</div>
```

<div style="color:blue">   
    <p>这是一个在 div 元素中的文本。</p>
    <p>这是一个在 div 元素中的文本。</p> 
</div>
<br>

### span行内文本设置

在其他区块中的文本行内,对一部分文字进行修改

例如:

```html
<p>我的母亲有 <span style="color:blue">蓝色</span> 的眼睛。</p>
```

<p>我的母亲有 <span style="color:blue">蓝色</span> 的眼睛。</p>

还可以设置字体大小,是否加粗等. `style`内整体用引号包裹,属性之间使用分号做间隔

```html
<span style="font-size:25px;font-weight:bold;background:yellow">组件介绍:</span>
```

<br>

### img插入图片

[看前面](#s1)

<br>

***

<br>

# 9 LaTex语法

参考: <https://lanlan2017.github.io/blog/83c2e83a/>

例如公式:

行内公式:

```
其他内容$a^2+b^2=c^2$其他内容
```

$a^2+b^2=c^2$



<br>

***

<br>

# 10 markdown转html时容易出现的问题

- 正文里尽量不用到 html的<标签>,如果必须要用,把它们放到代码框里

- 尽量多用html标签,例如在typora中如果想添加空行,不要直接回车换行,而是使用

  ``` html
  <br>

- 后面遇到问题再从这里补充

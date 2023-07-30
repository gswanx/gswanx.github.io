---
title: Apache的文件解析漏洞(特性)
description: 多后缀/.htaccess特性
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
abbrlink: aeaad0f
date: 2022-09-23 21:40:43
---

https://blog.werner.wiki/file-resolution-vulnerability-apache/

# 1.多后缀名

Apache认为, 一个文件可以有多个后缀,如:`111.txt.png.mp3.abc`

服务器在读取时, 会从右向左读取. 如果认识这个后缀, 就会以此来识别文件的类型, 如果不认识, 就会继续向左读取下一个后缀

例如上面这个文件名, 读取第一个`abc`发现不认识, 继续向左, 读取到`mp3`, 认识了, 那么就把此文件当成一个`mp3`文件来处理

如果一直到最左边都不认识, 则会把该文件当做**默认类型**进行处理, 一般是`text/plain`.

<br>

Apache认识的文件后缀记录在 名为`mime.types`的文件, 其路径为  `apache安装目录/conf/mime.types`

这个文件很长:

<img src="/images/ctf-web01/image-20220923214848650.png" alt="image-20220923214848650" style="zoom: 67%;" />

由此带来的问题:

有的网站为了防止用户上传恶意文件, 会写脚本对文件后缀进行检测, 如果不了解apache的这个特性, 可能编写的检测代码只会检测文件名最后面的这个后缀.

这样,如果用户上传一个名为`xxx.php.abc`的文件, 就能够绕过检测. 同时,这个文件在上传后还能够被apache识别为`php`文件并执行.

> 然而在实际测试的时候, 发现类似`aaa.php.xxx`的文件并不会被作为`php`程序执行
>
> 猜测:
>
> Apache虽然能够识别这是一个`php`文件, 但是将它交给`php`解释器之后, `php`的文件后缀解析规则却不是这样, 因此`php`解释器并不认为这是一个`php`文件, 故而不会把它当作`php`文件执行, 而是直接输出了文件内容.

在Apache的模块的配置文件中找到了`php5.conf` :

其中规定: 被当做`php`程序执行的文件名要符合正则表达式：

```
.+\.ph(p[345]?|t|tml)$
```

在`mime.types`中, 也有这一段:

<img src="/images/ctf-web01/image-20220923215253544.png" alt="image-20220923215253544" style="zoom:67%;" />

由此可见, `php3,php4,php5,pht,phtml `这几个后缀都能被识别为`php`文件

因此, 在绕过检测时, 也可以将文件后缀改成这几个.

测试，先准备文件`text.php`，其内容是输出Hello World：

```
 <?php echo 'HELLO WORLD'; ?>
```

然后在浏览器中打开它，成功显示“HELLO WORLD”。再修改该文件后缀为各种后缀，进行测试。测试结果是，以`php、phtml、pht、php3、php4和php5`为后缀，能成功看到“HELLO WORLD”；以`phps`为后缀，会报403错误，Forbidden；以`php3p`为后缀，**会在浏览器中看到源码**。

<br>

***

<br>

# 2.htaccess特性

> `htaccess`文件是Apache服务器中的一个配置文件，**提供了针对目录改变配置的方法**,**负责相关目录下的网页配置**。通过`.htaccess`文件，可以帮我们实现：网页301重定向、自定义404错误页面、改变文件扩展名、允许/阻止特定的用户或者目录的访问、禁止目录列表、配置默认文档等功能。
>
> `.htaccess`文件可以配置很多事情，如**是否开启站点的图片缓存、自定义错误页面、自定义默认文档、设置WWW域名重定向、设置网页重定向、设置图片防盗链和访问权限控制**。

<br>

此处只关心`.htaccess`文件的一个作用——**MIME类型修改**。如在`.htaccess`文件中写入：

```
AddType application/x-httpd-php xxx 
```

就能成功地使该`.htaccess`文件所在目录及其子目录中的后缀为`.xxx`的文件被Apache当做`php`文件。(**相当于建立一个`xxx`后缀到`php`文件类型的映射**)

另一种写法:

```
<FilesMatch "shell.jpg">
	SetHandler application/x-httpd-php
</FilesMatch>
```

该语句会让Apache把`shell.jpg`文件解析为`php`文件。

例题参考: [[MRCTF2020]你传你🐎呢](2e610037.html)
















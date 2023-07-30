---
title: SSRF
abbrlink: 813434ca
date: 2022-10-30 11:34:20
cover: false
tags:
  - CTF-Web
  - SSTI
categories:
  - CTF
  - Web安全
typora-root-url: ..

---





# SSRF服务端请求伪造

一般出现在服务端**从外部调用资源时**,如果没有对协议,IP,端口等进行限制,就有可能通过一些特殊协议访问到内网中的资源



# 常见利用协议

## file

获取文件内容,例如:

```
file:///etc/passwd
```



## gopher






























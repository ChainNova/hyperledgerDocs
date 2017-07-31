---
layout: page
title: "使用MkDocs管理你的CMS系统"
description: ""
---
{% include JB/setup %}

## 0x00 关于Sphinx和MkDocs
托管文档除了[用Jekyll管理你的Github Pages](./deploy-jekyll-on-the-github)外，<https://readthedocs.org/>也是使用的非常多，上面托管的文档用[Sphinx](http://www.sphinx-doc.org/)构建的比较多，文档也是比较复杂的，这里我们介绍一个非常好用的工具：[MkDocs](http://www.mkdocs.org/)。

MkDocs是研究Hyperledger的时候发现的一个工具，使用方法和配置都非常的简单，能够不用太关心建站的事情，能够聚焦到文档本身，看这个文章的长短就知道了。

## 0x01 安装和使用MkDocs

以下的操作步骤都按最简单的写，详细一点的配置参考[官方文档](http://www.mkdocs.org/)。

* 安装MkDocs

```
pip install mkdocs
```

* 创建文档工程

可以直接用mkdocs的命令：

```
mkdocs new my-project
cd my-project
```

生成的目录结构如下：

```
xxx
├── docs
│   └── index.md
└── mkdocs.yml

1 directory, 2 files
```

mkdocs.yml是配置文件，默认的配置项只有一个site_name：

```
site_name: My Docs
```

index.md的内容是一个mkdocs的使用说明：

```
# Welcome to MkDocs

For full documentation visit [mkdocs.org](http://mkdocs.org).

## Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs help` - Print this help message.

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.
```   

* 配置MkDocs

直接修改mkdocs.yml就可以，比如：
  - site_name：站点的名称
    比如我们的是这么设置的：
  - pages: 导航页面
  - theme: 站点主题
  - docs_dir: Markdown文档目录
  - site_dir: 生成的静态网页目录
  - dev_addr: 本地调试的监听地址和端口
  - markdown_extensions: 一些扩展

更多的配置项参考[官方文档](http://www.mkdocs.org/user-guide/configuration/)。

我们的一个工程是这么设置的：

```
site_name: 基于Hyperledger的基金管理
site_url: https://wutongtree.github.io/funds/
theme: simplex
site_description: '基于Hyperledger的基金管理'

docs_dir: markdowns
site_dir: docs

pages:
- 首页: index.md
- 功能描述: requirements.md
- 设计实现: designs.md
- 安装部署: deploy.md

markdown_extensions:
  - extra
  - tables
  - toc
  - fenced_code
  - smarty
  - mdx_math:
      enable_dollar_delimiter: True
  - footnotes

copyright: Copyright &copy; 2014-2016 <a href="https://wutongtree.com" target="_blank">wutongtree.com</a>
```

## 0x03 写文章
直接在docs目录下写Markdown文档就可以，也可以创建子目录，方便文档管理。 MkDocs对Markdown文档没有侵入性，完全按照标准的格式写就可以了，不像[Jekyll](./deploy-jekyll-on-the-githu)一样需要加一些文件头什么的，文档的链接也可以直接链接原始的Markdown文档，最终生成静态网页的时候会替换成最终的html页面。

## 0x04 本地测试

运行命令：

```
mkdocs serve
```

然后打开链接<http://127.0.0.1:8000>就可以预览了，它会监控本地页面的修改，所以一边改一边就能看到最终的效果了。

## 0x05 生成静态网页

运行命令：

```
mkdocs build
```

就会在site目录下看到生成的静态网页了，可以托管到任何一个空间服务商那里了。这个目录也是可以修改的，比如Github Pages用的docs目录，就可以在MkDocs里面改下。

## 0x06 解析没有在pages列表中的页面

官方的版本中，没有在mkdocs.yml配置文件的pages选项中出现过的页面不会转换为静态页面，也不能被其他页面链接。类似这样的错误或者警告：

```
WARNING -  The page "hyperledger/index.md" contained a hyperlink to "hyperledger/chaincode.md" which is not listed in the "pages" configuration.
```

这其实是他们的设计，因为他们还专门有一个选项[strict](http://www.mkdocs.org/user-guide/configuration/#strict)来说这个事情，如果有这种链接错误的话，要不要终止构建啊 :(

可是有的链接我确实不需要出现在导航栏，直接出现在页面的链接中就好了，否则我增加一个页面就要增加一个导航页面，文章多了就比较麻烦了。

如果你也需要这个功能，可以用我们修改的一个版本：

```
git clone https://github.com/wutongtree/mkdocs.git
cd mkdocs
sudo python setup.py install
```

修改的版本已经给官方提交了pull request，估计是不会通过的 :)

## 0x07 错误

如果发现错误：

```
Traceback (most recent call last):
  File "/usr/local/bin/mkdocs", line 11, in <module>
    sys.exit(cli())
  File "/Library/Python/2.7/site-packages/click/core.py", line 716, in __call__
    return self.main(*args, **kwargs)
  File "/Library/Python/2.7/site-packages/click/core.py", line 696, in main
    rv = self.invoke(ctx)
  File "/Library/Python/2.7/site-packages/click/core.py", line 1060, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/Library/Python/2.7/site-packages/click/core.py", line 889, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/Library/Python/2.7/site-packages/click/core.py", line 534, in invoke
    return callback(*args, **kwargs)
  File "/Library/Python/2.7/site-packages/mkdocs/__main__.py", line 137, in build_command
    ), clean_site_dir=clean)
  File "/Library/Python/2.7/site-packages/mkdocs/commands/build.py", line 289, in build
    build_pages(config)
  File "/Library/Python/2.7/site-packages/mkdocs/commands/build.py", line 249, in build_pages
    dump_json)
  File "/Library/Python/2.7/site-packages/mkdocs/commands/build.py", line 169, in _build_page
    site_navigation=site_navigation
  File "/Library/Python/2.7/site-packages/mkdocs/commands/build.py", line 35, in convert_markdown
    extension_configs=config['mdx_configs']
  File "/Library/Python/2.7/site-packages/mkdocs/utils/__init__.py", line 337, in convert_markdown
    extension_configs=extension_configs or {}
  File "/Library/Python/2.7/site-packages/markdown/__init__.py", line 159, in __init__
    configs=kwargs.get('extension_configs', {}))
  File "/Library/Python/2.7/site-packages/markdown/__init__.py", line 185, in registerExtensions
    ext = self.build_extension(ext, configs.get(ext, {}))
  File "/Library/Python/2.7/site-packages/markdown/__init__.py", line 264, in build_extension
    module = importlib.import_module(module_name_old_style)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/importlib/__init__.py", line 37, in import_module
    __import__(name)
ImportError: Failed loading extension 'mdx_math' from 'mdx_math', 'markdown.extensions.mdx_math' or 'mdx_mdx_math'
```

安装一下缺的库就可以了：

```
sudo pip install python-markdown-math
```

## 0xFF 总结
MkDocs非常适合不关心页面的程序员写文档了，主题也比较多，展示效果还是不错的。

---
title: 使用Github Pages和Jekyll建站小记
date: 2017-08-01 15:29:50
tags:
---

## 使用Github Pages建立博客

每个Github账户都能使用Github Pages这样一个免费空间为自己搭建一个博客, 像很多Hexo之类的博客就很流行, 也很美观, 但是实际上写起博客来还是会很重, 而且定制化不强, 其实主要还是博客都是别人帮你直接弄好的, 你用模板就行了, 所以经常博客出现了什么问题, 也没有很好的办法解决, 自己去定制的话, 也会很麻烦. 
当然, 你去把博客的源码完整地读一遍也是可以的. 

Github Pages现在建立博客很简单, 只需要新建一个名为 username.github.io 的仓库, github会自动为你的仓库建立在github pages上(以前是需要自己进入仓库设置里手动点选github pages才行的)

## 为什么选择Jekyll

[Jekyll](http://jekyll.com.cn/)是一个静态博客生成器, github pages也支持jekyll, 也就是说我们只需要将jekyll生成好的文件上传到你的username.github.io仓库里, github pages就会自动帮你构建jekyll环境, 将你的静态博客运行起来. 

为什么我要从wordpress弃用换成用jekyll生成静态博客呢？其实主要有这么几个原因


* wordpress会比较笨重, 而且有大量的功能都是我不需要的, 我如果只是纯粹想写作的话, 用wordpress会很累赘
* wordpress的文档都保存在数据库里, 相比起来想要提取以前写过的文档, 会麻烦很多, 不是直接以某种文件形式保存起来
* 我比较喜欢一种极简风, 现在的静态博客排版十分清晰爽朗, 让我可以一眼看清楚我写了哪些文章. 
* 使用jekyll设置好之后, 我只需要本地写好markdown博文后push到我的github仓库上, 即可更新博文, 相比wordpress更加直接快速, 且易于保存
* 可定制性很强, 虽然排版会很花时间, 但是我可以自由地管理各个参数. 而托管在github pages也省去了我管理服务器的麻烦. 


## 来吧, 本地搭建jekyll环境

本地搭建jekyll环境的话, 可以说jekyll的官方文档很贴心详细, 讲的浅显易懂, 所以我大部分操作都是根据官方文档进行的, 当然也有借鉴别人的经验. 

在安装jekyll之前, 需要解决一些软件依赖, 所以你首先需要安装如下：


* [Ruby](http://www.ruby-lang.org/en/downloads/) Jekyll是基于ruby的, 因此我们需要本地安装ruby. 但你放心, 我们并不需要编写ruby代码
* [RubyGems](http://rubygems.org/pages/download) 可以说这是ruby的一个包管理器, 我们可以使用gem命令很方便地安装很多东西
* Linux, Unix, or Mac OS X 不建议在windows平台下安装jekyll, 但是当然windows也有专门的[jekyll安装程序](http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html)


随后我们就需要使用RubyGem来帮助我们安装jekyll了, 我们只需要在终端输入以下命令即可安装jekyll

``` bash 
$ gem install jekyll
```

那么实际上我们的jekyll已经安装完毕了, 但是这里我还需要安装一个软件, 它是Bundler

Bundler是负责管理RubyGems组件的版本, 我们装上它就能保证本地的jekyll环境和远程github pages环境一致

安装命令也很简单, 终端输入以下命令

```bash
gem install bundler
```


* 注意！当我们安装好bundler后, 在之后我们的网站文件目录下, 我们需要建立 **Gemfile** 文件, 并添加以下两行：


``` bash
source 'https://ruby.taobao.org'
gem 'github-pages'
#添加好后在终端执行bundle install
```

## jekyll和bundle的可用命令

bundle的命令比较简单, 但是我们之后都需要用bundle来运行jekyll

``` bash
#用来确保本地与github pages的jekyll运行环境相同, 保持gem版本一致
bundle update 
#用于运行jekyll生成静态博客文件
bundle exec jekyll build 
#用于启动我们的静态博客, 通过访问 http://localhost:4000 预览网站
bundle exec jekyll serve 
```

jekyll的命令, 其实在bundle里我们就有接触了, 这里我们列一下：

``` bash
$ jekyll build
# => The current folder will be generated into ./_site

$ jekyll build --destination <destination>

# => The current folder will be generated into <destination>

$ jekyll build --source <source> --destination <destination>

# => The <source> folder will be generated into <destination>

$ jekyll build --watch

# => The current folder will be generated into ./_site,
# watched for changes, and regenerated automatically.

$ jekyll serve

# => A development server will run at http://localhost:4000/
# Auto-regeneration: enabled. Use `--no-watch` to disable.

$ jekyll serve --detach

# => Same as `jekyll serve` but will detach from the current terminal.
# If you need to kill the server, you can `kill -9 1234` where "1234" is the PID.
# If you cannot find the PID, then do, `ps aux | grep jekyll` and kil

$ jekyll serve --no-watch

# => Same as `jekyll serve` but will not watch for changes.
# As of version 2.4, the "serve" command will watch for changes automatically.
```

正常来说, 我们只需要用 jekyll build 和jekyll serve就行了, 当然在前面还要加上bundle exec

## jekyll 基本结构

关于jekyll基本结构, 最好还是看[官方文档](http://jekyll.com.cn/docs/structure/), 文档里有很详细的说明, 一般的jekyll网站目录结构如下

``` bash
.
├── _config.yml #jekyll的配置文件, 很重要也很有用
├── _drafts #存放草稿, 这个随意看个人习惯
|   ├── begin-with-the-crazy-ideas.textile
|   └── on-simplicity-in-technology.markdown
├── _includes   #你可以加载这些包含部分到你的布局或者文章中以方便重用.
|   ├── footer.html
|   └── header.html
├── _layouts    #我们上传的博文会根据layout类型来进行相应的排版布局
|   ├── default.html
|   └── post.html
├── _posts  #这里放的就是你的文章了. 文件格式很重要, 必须要符合: YEAR-MONTH-DAY-title.MARKUP
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.textile
|   └── 2009-04-26-barcamp-boston-4-roundup.textile
├── _data   #随意
|   └── members.yml
├── _site   #我们用jekyll生成好的静态html文件都存放在这里
└── index.html#首页
```

这里有几点必须要注意

- _post里的文件以及网站根目录下的html文件, 都必须要有YAML头信息, 才会被jekyll转换
- 凡是有"_"开头的文件夹, 在经过jekyll转换后, 都不会被包含在_site文件夹里
- 其他一些未被提及的目录和文件如  css 还有 images 文件夹,  favicon.ico 等文件都将被完全拷贝到生成的 site 中. 
- 我们在关于jekyll自动生成html文件时, 需要使用到[liquid语法](http://alfred-sun.github.io/blog/2015/01/10/jekyll-liquid-syntax-documentation/)

ok,那么开始生成自己的网站目录吧

## 建立本地的project

我们将github上的username.github.io仓库clone到本地后, 按照上面的目录结构, 将各个文件夹都建立起来之后, 就可以使用jekyll运行了. 

``` bash
$ bundle exec jekyll build
$ bundle exec jekyll serve
```

关于配置方面, 我大部分都是参考别人的仓库代码, 你也可以参考这个：[programminghistorian/jekyll](https://github.com/programminghistorian/jekyll)

这里我就不细说啦, 因为太多了. 看代码就是了

## 代码高亮

搭建好我们的博客后, 我们要将代码高亮. 这里其实我被困扰很久, 因为我尝试的几个markdown解析器, 都无法正确解析markdown的代码块格式并正确着色, 因此我这里是使用的[highlight.js](https://highlightjs.org/), 之后在我们的header.html里添加如下代码调用即可

``` html
<link rel="stylesheet" href="/css/atom-one-light.css">
<script src="/js/highlight.pack.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
```

## 域名绑定

其实就是将www.vancir.com的dns解析到我的vancir.github.io, 方法很简单, 但是容易出问题

我们只需要打开vancir.github.io仓库的settings在最下面的github pages项的custom domain上填上自己的域名, 不要加上任何比如说http或www的东西. 比如我的就是vancir.com
然后在你的域名管理平台上设置域名解析, 一般都会有新手快速设置一类的东西, 只需要我们填写ip地址. 

关于我们的github pages的ip地址的获取, 我们只需要简单ping一下

``` bash
ping vancir.github.io
```

得到ip地址后在域名解析处填好就可以了. 

## 心得

整个差不多弄完了, 当然你也可以弄些比如rss, mathjax, 评论之类的功能. 其实整个站都是比较简单的, 只是在很多细节问题上, 常常会困扰很久. 
但最起码的, css的排版样式问题是最最花时间了！
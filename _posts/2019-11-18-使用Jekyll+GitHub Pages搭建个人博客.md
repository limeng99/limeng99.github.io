---
layout: post
title: "使用Jekyll+GitHub Pages搭建个人博客"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/jekyll.jpg"
thumbnail: "assets/img/article/jekyll.jpg"
---

### 前言

Jekyll + GitHub Pages可以让你更加专注于博客内容，而不是如何搭建一个博客平台。Jekyll + GitHub Pages帮助你搭建专属于自己的个性化博客。

### Jekyll

#### 一、Jekyll是什么？

> *引用自官网*：
>  *Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，通过一个转换器（如 [Markdown](https://link.jianshu.com?t=http%3A%2F%2Fdaringfireball.net%2Fprojects%2Fmarkdown%2F)）和我们的 [Liquid](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FShopify%2Fliquid%2Fwiki) 渲染器转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。Jekyll 也可以运行在 [GitHub Page](https://link.jianshu.com?t=http%3A%2F%2Fpages.github.com%2F) 上，也就是说，你可以使用 GitHub 的服务来搭建你的项目页面、博客或者网站，而且是完全免费的。*

Jekyll就是将纯文本转化为静态博客网站，不需要数据库支持，也没有评论功能，想要评论功能的话可以借助第三方的评论服务。

#### 二、搭建本地Jekyll环境

*注：安装jekyll会用到ruby，最好不要用系统自带的，使用系统提供的ruby会出现没有权限问题，建议使用rbenv新安装一个ruby使用。*

具体使用rbenv安装ruby可以参考：
[Mac环境配置](https://limemg99.club/learning/2019/02/25/Mac%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE.html)

- 设置全局ruby版本
```
$ rbenv global 2.6.0 #例如设置新安装的2.6.0版本为全局版本
$ gem env home #验证gem
```
- 使用gem安装Jekyll
```
$ gem install jekyll
```
- 安装bundler
```
$ gem install bundler
```
- 使用Jekyll创建博客仓库
```
$ jekyll new myblog
```
- 进入myblog目录 开启Jekyll服务
```
$ cd myblog
$ jekyll serve
```
Jekyll服务默认端口是4000，打开浏览器，输入：[http://localhost:4000](http://localhost:4000)就能看到一个简单的博客页面。

#### 三、Jekyll的一些常用命令
```
当前文件夹中的内容将会生成到 ./_site 文件夹中。
$ jekyll build

当前文件夹中的内容将会生成到目标文件夹<destination>中。
$ jekyll build --destination <destination>

指定源文件夹<source>中的内容将会生成到目标文件夹<destination>中。
$ jekyll build --source <source> --destination <destination>

当前文件夹中的内容将会生成到 ./_site 文件夹中，查看改变，并且自动再生成。
$ jekyll build --watch

一个开发服务器将会运行在 http://localhost:4000/
$ jekyll serve

功能和`jekyll serve`命令相同，但是会脱离终端在后台运行。
如果你想关闭服务器，可以使用`kill -9 1234`命令，"1234" 是进程号（PID）。
如果你找不到进程号，那么就用`ps aux | grep jekyll`命令来查看，然后关闭服务器。
$ jekyll serve --detach

```
#### 四、Jeykll的目录结构
```
├── _config.yml  			(配置文件)
├── _drafts  				(drafts（草稿）是未发布的文章)
|   ├── begin-with-the-crazy-ideas.textile
|   └── on-simplicity-in-technology.markdown
├── _includes 			(加载这些包含部分到你的布局)
|   ├── footer.html
|   └── header.html
├── _layouts 			    (包裹在文章外部的模板)
|   ├── default.html
|   └── post.html
├── _posts 				  (这里都是存放文章)
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.textile
|   └── 2009-04-26-barcamp-boston-4-roundup.textile
├── _site 				(生成的页面都会生成在这个目录下)
├── .jekyll-metadata	  (该文件帮助 Jekyll 跟踪哪些文件从上次建立站点开始到现在没有被修改，哪些文件需要在下一次站点建立时重新生成。该文件不会被包含在生成的站点中。)
└── index.html 		   (网站的index)
```

### GitHub Pages

#### 一、创建一个仓库

转到GitHub并创建一个名为*username.github.io*的新存储库，在Settings里面找到Github Pages，选择一个主题

#### 二、克隆仓库

```
$ git clone https://github.com/username/username.github.io
```

#### 三、部署Blog

- 开启Jekyll服务

```
$ cd username.github.com

打开http://localhost:4000,可看见我们在Github上创建的主页
$ jekyll serve
```

- 选择你想要的主题，进行更改，然后推送到仓库

```
$ git add --all
$ git commit -m "jekyll页面"
$ git push origin master
```

- _config.yml配置必须符合GitHub Pages的规定模式

```
highlighter: rouge
markdown: kramdown
```

#### 四、申请个人域名

域名申请国内一般使用[万网](https://wanwang.aliyun.com/)，国外使用[Go Daddy](https://sg.godaddy.com/)

- 创建`CNAME`，添加你的域名

 ```
  $ git add CNAME
  $ git push origin master
 ```

- DNS提供商，DNS解析创建一个`CNAME`记录

```
主机记录www，记录类型为CNAME类型，CNAME表示别名记录，该记录可以将多个名字映射到同一台计算机,
记录值请写username.github.io
```

- 要创建`A`记录，顶点域指向GitHub Pages的IP地址

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```
- 自定义域升级HTTPS

```
GitHub的username.github.io仓库中，进入Settings, GitHub pages选项中勾选Enforce HTTPS选择即可。如不可勾选，请核对DNS解析中A记录中记录值是否正确
```


### Jekyll+GitHub Pages相关链接

##### 1. Jekyll官方中文文档

[Jekyll官方中文文档]( http://jekyllcn.com/docs/home/)

##### 2. GitHub Pages网址

[GitHub Pages官方网址](https://pages.github.com/)  [使用GitHub Pages帮助](https://help.github.com/en/github/working-with-github-pages)

##### 3. Jekyll主题网站，多种个性化主题
[Jekyll主题网站](http://jekyllthemes.org/)

##### 4. 冰霜之地大神的博客文章
[如何快速给自己构建一个温馨的"家"——用 Jekyll 搭建静态博客](https://halfrost.com/jekyll/)

##### 5.疑难杂症解决链接
[You don't have write permissions for the /Library/Ruby/Gems/2.0.0 directory](https://github.com/rbenv/rbenv/issues/938#issuecomment-285342541)
---
title: Hexo+Github搭建免费技术博客
date: 2020-01-08 12:05:45
tags: Hexo
category: Linux
---

最近想搭建了一个自己的博客网站，经过调研，最终决定使用Hexo+Github的形式搭建，大概花了3个小时的时间，这里简单记录一下过程，备用。

## 综述
Github提供了一个免费的GIthub Pages功能，简单说就是可以让你存放静态的资源文件，比如css、js、html、image，同时会给你分配一个免费域名，格式：https://xxx.github.io

所以你只需要注册一个Github账号，然后新建一个repository就行了，一毛线都不用花。

由于Github Pages只能托管静态资源，所以像WordPress这类博客肯定是无法运行的，这时候你有2种选择：

1. 手写静态页面，如果是前端大牛可以尝试这样
2. 第三方博客工具生成，如Hexo、jekyll

<!---more--->

简单说，Hexo就是一个工具，它可以根据markdown文档自动生成博客的静态HTML页面，同时呢，你还可以一键换主题，网上有很多开源的主题。

Hexo 和 Github这2个完全可以单独使用，但是把2个结合起来就完美了，一个用来生成博客的静态文件，一个用户托管静态资源，服务器和域名都省了。

## 环境准备
1. 我不会告诉你如何注册Github账号、以及安装使用Git，作为一名编程开发人员应该都会

2. 我其实也不想告诉你如何安装npm和node，但是我还是放个下载地址：https://nodejs.org/zh-cn

## Hexo应用
### 安装运行环境
```
npm install -g hexo-cli
```

安装完成后，在命令行下执行hexo，应该可以看到以下输出：

```
jwang@jun:~$ hexo
Usage: hexo <command>

Commands:
  help     Get help on a command.
  init     Create a new Hexo folder.
  version  Display version information.

Global Options:
  --config  Specify config file instead of using _config.yml
  --cwd     Specify the CWD
  --debug   Display all verbose messages in the terminal
  --draft   Display draft posts
  --safe    Disable all plugins and scripts
  --silent  Hide output on console

For more help, you can use 'hexo help [command]' for the detailed information
or you can check the docs: https://hexo.io/docs/
```

这里只是列出一部分命令，比较重要的就是init，它是用来创建一个新项目

### 创建博客
```
hexo init blog
```

文件夹名字自己起一个，它自动生成一个目录，里面文件如下：

```
├── _config.yml
├── db.json
├── package.json
├── package-lock.json
├── scaffolds
├── source
└── themes
```
> 简单说明一下，比较重要是有_config.yml文件,这是博客的配置，另外themes下是存放主题的目录，还有source下面的 _posts 目录，是博客文章markdown的源文件。

然后我们进入该目录，生成静态文件并启动服务预览一下:

```
cd blog

hexo g

hexo s
```
默认启动在本地4000端口，可以通过 -p 指定端口

### 写博客

```
hexo new <article-name>
```

其实有2种方式写文章，一种是使用上述命令 new 一个，它会自动在source目录的_posts里面创建一个markdown文件。另一种就是你自己手动创建。

但是注意，文章头部会有一些注解：

```
title: Hexo+Github搭建免费技术博客
date: 2020-01-08 12:05:45
tags: Hexo
category: 其它
```
Hexo在生成静态页面的时候会解析这些注解，然后做一些处理，比如tags是标签、category是文章分类，都会用到。

不要忘记，每次更新文章之后，都需要执行```hexo g```重新生成静态页面。

## 部署到GitPages
上面介绍如何使用Hexo生成博客，但是这时候只能在本地玩，如果你有自己的服务器的话，也可以不用GitPages，你把生成的静态文件，也就是public目录下的文件部署到你自己的服务器就行了。

如果你想部署到GitPages，那么继续接着看

>有一点需要注意，在创建GitPage仓库的时候，仓库名字最好是: 你的用户名.github.io 这种格式，如果不这样其实也行，就是分配的域名有点丑，比如说你仓库名字叫blog，那么域名就会变成 xxx.github.io/blog

打开Hexo的配置文件_config.yml，修改repo为刚创建的仓库地址：

```
deploy:
  type: git
  repo: https://github.com/YourgithubName/YourgithubName.github.io.git
  branch: master
```
为了更好的提交代码，我们需要安装一个插件 ```npm install hexo-deployer-git --save```

然后，我们使用```hexo d```就可以把代码提交到Github仓库了。

>等等。。。有人说网上很多文章还说要配置什么ssh秘钥，其实这块我觉得不是必须的，只是为了更方便的提交代码而已，具体步骤这里不再赘述。

## 更换主题
Hexo更换主题特别简单，只需要把主题文件夹clone到themes文件里面，然后修改_config.yml里面的 theme 配置项。

详细的步骤我这里就不解释了，你可以在Github使用 “hexo themes”关键字搜索，然后按照其readme文档说明安装即可，非常简单。

## 注意事项

1. 如果你配置了ssh秘钥，则必须把deploy配置里的repo配置 https 地址改成 ssh 地址。
2. 很多主题都有一个自己的_config.yml配置文件，里面有一些详细配置，比如是Next这个主题默认没有打开分类和标签项，需自己配置。
3. 如果想实现“阅读全文”这种效果，有2种方式，第一种是需自己在markdown里合适的位置作注解，默认是 ```<!--more-->```，还有一种在主题的配置里面，可以自动根据字数折叠，但是默认不推荐这种方式。
4. 每个主题都有很多自定义的配置项，比如样式、字体、评论、浏览数，很多默认都没开启，可以好好看一下，都有注释。
5. 最重要的一点，所有的东西都是开源的，如果你觉得很多样式或者细节不合适完全可以打开模板修改定制。

最后，如果你闲麻烦，觉得我的博客还可以，想参考一下可以访问我的[Github](https://github.com/wangbjun/blog_hexo)，所有的文件和配置都在里面，欢迎采用。

---
title: Hexo使用教程
author: 汪博全
tags:
- Hexo
- 已完结
categories:
- 技术
date: 2019-07-08 16:33:50
---

> 已完结

<!-- more -->

换了新电脑需要把原来博客的源文件挪过来，趁此机会整理一下hexo安装过程和常用命令
由于之前的博客在图片上有些小问题，索性顺便解决了

我的博客是部署在coding上的master分支，源文件存在另外一个分支，先clone下来

## 1 安装

### 1.1 安装Hexo

官网上已经给出了安装教程：
```bash
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo server
```
由于我们已经有了一个blog所以就不用init了，执行hexo server后就可以看到我们的博客了

### 1.2 安装NexT

不知道为什么NexT主题没有备份到git上，所以要重新安装，访问NexT的仓库地址：[NexT](https://github.com/theme-next/hexo-theme-next)，上面有安装教程

注：有很多hexo的教程里的NexT仓库地址还是旧的，那个已经停止维护了

## 2 配置

有两个地方需要配置，一个是站点配置文件，在博客的根目录`_config.yml`，另一个是主题配置文件，由于我们使用的是NexT主题，所以配置文件在`themes/next/_config.yml`，其他主题同理。以下只列举常用的配置，更详细的配置参考官方文档。

### 2.1 站点配置文件

* Site

这部分配置博客的一些基本信息，例如标题作者，这里要注意的是语言的配置，hexo官方说中文要写`zh-HANS`，但是由于我使用的是NexT主题，而NexT主题中对应的语言文件叫`zh-CN.yml`，所以这里还是填`zh-CN`

* URL

配置博客的地址，主要填`url`和`root`，例如我的博客地址是`http://wangboquan.coding.me/blog/`，那么`url`就填这个，`root`填`/blog`

* Writing

这里要启用`post_asset_folder`，其他保持默认就行，用途下面会讲。

* theme

使用什么主题就填哪个主题的名字，这里我填next

* deploy

配置博客部署的位置，type一般都git，repo就填仓库地址，branch填master

### 2.2 主题配置文件

其实这个保持默认也没关系，我就调了两个部分，一个是`scheme`，一个是`menu`。`scheme`可以指定NexT主题的风格，我比较喜欢Pisces，可以在侧边常驻菜单栏。`menu`可以调整菜单栏的内容。

## 3 写文章

运行命令

``` bash
$ hexo new "My New Post"
```

然后就会在`source/_post`下生成对应title的**md文件**，由于启用了`post_asset_folder`，还会生成一个**文件夹**，文件夹是用来存放图片等静态文件的，在md里面写文章内容就行了。

这里要说明的是，如果想在文章里插入图片，一种方法是使用图床，即把图片上传到图床生成链接，再在文章里插入这个链接。但是这样非常麻烦，除了维护博客之外还要维护图床，而且图片一多很难管理，我们更希望让图片和文章保存在一起。另一种方法是使用hexo-asset-image插件，可以让博客访问到上述文件夹中的内容。但是这个插件有一个问题，当博客是放在子文件夹下时（就像我的博客），无法生成正确的图片地址，访问就会出问题。

但是在hexo3版本之后，官方实现了这个功能

> 通过常规的 markdown 语法和相对路径来引用图片和其它资源可能会导致它们在存档页或者主页上显示 不正确。在Hexo 2时代，社区创建了很多插件来解决这个问题。但是，随着Hexo 3  的发布，许多新的标签插件被加入到了核心代码中。这使得你可以更简单地在文章中引用你的资源。
> ```
{% asset_path slug %}
{% asset_img slug [title] %}
{% asset_link slug [title] %}
> ```
> 比如说：当你打开文章资源文件夹功能后，你把一个 `example.jpg` 图片放在了你的资源文件夹中，如果通过使用相对路径的常规 markdown 语法 `![](/example.jpg)` ，它将不会出现在首页上。（但是它会在文章中按你期待的方式工作）正确的引用图片方式是使用下列的标签插件而不是 markdown ：
>```
{% asset_img example.jpg This is an example image %}
>```
> 通过这种方式，图片将会同时出现在文章和主页以及归档页中。

## 4 测试

写好文章后，我们先生成静态文件

``` bash
$ hexo g
```

然后运行测试服务器，看网页是否显示正常
``` bash
$ hexo s
```

有的时候我们需要清空静态文件，就可以运行下面的命令

``` bash
$ hexo clean
```

## 5 部署

``` bash
$ hexo d
```

## 6 草稿功能

### 6.1 新建草稿

```bash
$ hexo new draft <title>
```

### 6.2 预览草稿

```bash
$ hexo S --draft
```

### 6.3 草稿转正

```bash
$ hexo P <filename>
```

## 7 其他

``` bash
hexo g = hexo generate
hexo s = hexo server
hexo d = hexo deploy
```

http://blog.csdn.net/lsshlsw/article/details/50322465
https://segmentfault.com/q/1010000002561642
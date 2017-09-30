---
title: 搭建dokuwiki
author: 汪博全
tags:
- dokuwiki
- 已完结
categories:
- 技术
- 软件配置
date: 2017-02-28 21:44:00
---

> 已完结

<!-- more -->

### 一、前言
DokuWiki是一个开源wiki引擎程序，运行于PHP环境下。DokuWiki程序小巧而功能强大、灵活，适合中小团队和个人网站知识库的管理。

### 二、环境
在centos6 下安装apache2，php

`yum install httpd`

`yum install php`

`/etc/init.d/httpd start`

`chkconfig --levels 235 httpd on`

--- 开机自启，建议打开
### 三、安装
1）在官方网站下载最新的稳定版：http://download.dokuwiki.org/，然后解压缩到你的网站目录下，比如/var/www/html/dokuwiki。
apache默认的目录`/var/www/html`，故需要把解压后的目录拷贝到这下面

2）设置dokuwiki的访问权限

`chown -R apache:root /var/www/html/dokuwiki`

`chmod -R 664 /var/www/html/dokuwiki/`

`find /var/www/html/dokuwiki/ -type d -exec chmod 775 {} \`

3）访问 http://域名/dokuwiki/install.php 右上角，选择 `zh`，填写表格

4）为安全起见，删除`/var/www/html/dokuwiki`目录下的install.php

`rm /var/www/html/dokuwiki/install.php`

### 四、安全
如果你能通过上面这个 http://域名/dokuwiki/data/pages/wiki/dokuwiki.txt 链接，访问到dokuwiki.txt文件，那么表明你的网站的数据是不安全，因为dokuwiki是文本数据库，也就是别人可以直接拖库了。
官方指定data，conf，bin，inc这四个目录不能通过web访问浏览的，所以，我们要设置这些目录的权限，保证网站的数据安全。
详情见：https://www.dokuwiki.org/start?id=zh:security

解决方法：

1）以nginx配置的
在网站的nginx.conf配置文件的server段加上下面的代码：
```
location ~ /(data|conf|bin|inc)/
{
	deny all;
}
```
2）以apache配置的

在`/etc/httpd/conf`目录下，编辑httpd.conf文件，

```
<Directory /var/www/html/dokuwiki>
order deny,allow
allow from all
</Directory>
<LocationMatch "/dokuwiki/(data|conf|bin|inc)/">
order allow,deny
deny from all
satisfy all
</LocationMatch>
```

到这里，自建的wiki就完成了，可以通过 http://域名/dokuwiki/ 访问了

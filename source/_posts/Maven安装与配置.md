---
title: Maven安装与配置
author: 汪博全
tags:
- Java
- Maven
- 已完结
categories:
- 技术
- 软件配置
date: 2017-02-28 21:33:00
---

> 已完结

### 前言
Maven项目对象模型(POM)，可以通过一小段描述信息来管理项目的构建，报告和文档的软件项目管理工具。本文介绍如何安装与配置maven。

<!-- more -->

### 配置环境变量

|操作系统|	输出|
|------|------|
|Windows	|使用系统属性设置环境变量。M2_HOME=C:\Program Files\Apache SoftwareFoundation\apache-maven-3.2.5<br>M2=%M2_HOME%\bin<br>MAVEN_OPTS=-Xms256m -Xmx512m
|Linux	|打开命令终端设置环境变量。<br>export M2_HOME=/usr/local/apache-maven/apache-maven-3.2.5<br>export M2=\$M2_HOME/bin<br>export MAVEN_OPTS="-Xms256m -Xmx512m"
|Mac	|打开命令终端设置环境变量。<br> export M2_HOME=/usr/local/apache-maven/apache-maven-3.2.5<br>export M2=$M2_HOME/bin <br> export MAVEN_OPTS="-Xms256m -Xmx512m"

**添加 Maven bin 目录到系统路径中**
现在添加 M2 变量到系统“Path”变量中

|操作系统	|输出|
|-----|-----|
|Windows	|添加字符串 “;%M2%” 到系统“Path”变量末尾|
|Linux	|export PATH=\$M2:\$PATH|
|Mac	|export PATH=\$M2:$PATH|

**验证 Maven 安装**
现在打开控制台，执行以下 mvn 命令。

|操作系统	|输出	|命令|
|------|------|-----|
|Windows	|打开命令控制台|	c:\> mvn --version|
|Linux	|打开命令终端	|\$ mvn --version|
|Mac	|打开终端	|machine:~ joseph$ mvn --version|



### 基础配置
修改 Maven 配置文件（setting.xml），可修改全局配置或用户配置：

全局配置：`%M2_HOME%\conf\settings.xml`
用户配置：`用户目录\.m2\settings.xml`

（不要修改全局配置）

* 配置 Maven 镜像（以aliyun为例）

```xml
<mirrors>
...
    <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>*</mirrorOf>
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
...
</mirrors>
```

* 配置 Maven 仓库

```xml
<profiles>
...
        <profile>
            <id>nexus-aliyun</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <repositories>
                <repository>
                    <id>nexus-aliyun</id>
                    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>nexus-aliyun</id>
                    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
                </pluginRepository>
            </pluginRepositories>
        </profile>
...
</profiles>
```

### 高级配置

注意：以下高级配置可根据实际情况有选择性地使用。

* **配置本地仓库路径**

若需要指定 Maven 本地仓库的路径时，可进行如下配置：

```xml
<localRepository>D:/Repository/Maven</localRepository>
```

需要根据实际情况进行配置。

* **配置 HTTP 代理**

对于有些公司而言，需要配置 HTTP 代理才能上外网，可进行如下配置：

```xml
<proxies>
...
    <proxy>
         <active>true</active>
         <protocol>http</protocol>
         <host>xxx.xxx.xxx.xxx</host>
         <port>xxxx</port>
    </proxy>
...
</proxies>
```


### 命令行创建项目
进入项目目录，输入`mvn archetype:generate -DgroupId=com.companyname.bank -DartifactId=consumerBanking -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
`

### 问题

* **下载速度慢**

在用户配置中配置国内镜像

* **Generating Project in Batch mode 卡住问题**

手动下载archetype-catalog.xml文件 [下载地址](http://repo.maven.apache.org/maven2/archetype-catalog.xml)
然后复制到`用户目录\.m2\repository\org\apache\maven\archetype\archetype-catalog\2.4`下面；
然后在执行的命令后面加上增加参数`-DarchetypeCatalog=local`，变成读取本地文件即可。

在IDEA中：
perference -> 搜索maven -> 在maven下的runner中的vm options中输入`-DarchetypeCatalog=internal`
或
创建maven项目的时候在下面界面添加一个属性`archetypeCatalog = internal`

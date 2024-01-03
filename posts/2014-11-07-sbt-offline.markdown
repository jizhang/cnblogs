---
layout: post
title: "离线环境下构建 sbt 项目"
date: 2014-11-07 15:02
comments: true
categories: [Programming]
tags: [scala]
published: true
---

在公司网络中使用[sbt](http://www.scala-sbt.org/)、[Maven](http://maven.apache.org/)等项目构建工具时，我们通常会搭建一个公用的[Nexus](http://www.sonatype.org/nexus/)镜像服务，原因有以下几个：

* 避免重复下载依赖，节省公司带宽；
* 国内网络环境不理想，下载速度慢；
* IDC服务器没有外网访问权限；
* 用于发布内部模块。

sbt的依赖管理基于[Ivy](http://ant.apache.org/ivy/)，虽然它能直接使用[Maven中央仓库](http://search.maven.org/)中的Jar包，在配置时还是有一些注意事项的。

<!-- more -->

## 配置Nexus镜像

根据这篇[官方文档](http://www.scala-sbt.org/0.13/docs/Proxy-Repositories.html)的描述，Ivy和Maven在依赖管理方面有些许差异，因此不能直接将两者的镜像仓库配置成一个，而需分别建立两个虚拟镜像组。

![](http://www.scala-sbt.org/0.13/docs/files/proxy-ivy-mvn-setup.png)

安装Nexus后默认会有一个Public Repositories组，可以将其作为Maven的镜像组，并添加一些常用的第三方镜像：

* cloudera: https://repository.cloudera.com/artifactory/cloudera-repos/
* spring: http://repo.springsource.org/libs-release-remote/
* scala-tools: https://oss.sonatype.org/content/groups/scala-tools/

对于Ivy镜像，我们创建一个新的虚拟组：ivy-releases，并添加以下两个镜像：

* type-safe: http://repo.typesafe.com/typesafe/ivy-releases/
* sbt-plugin: http://dl.bintray.com/sbt/sbt-plugin-releases/

对于sbt-plugin，由于一些原因，Nexus会将其置为Automatically Blocked状态，因此要在配置中将这个选项关闭，否则将无法下载远程的依赖包。

## 配置sbt

为了让sbt使用Nexus镜像，需要创建一个~/.sbt/repositories文件，内容为：

```
[repositories]
  local
  my-ivy-proxy-releases: http://10.x.x.x:8081/nexus/content/groups/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
  my-maven-proxy-releases: http://10.x.x.x:8081/nexus/content/groups/public/
```

这样配置对大部分项目来说是足够了。但是有些项目会在构建描述文件中添加其它仓库，我们需要覆盖这种行为，方法是：

```bash
$ sbt -Dsbt.override.build.repos=true
```

你也可以通过设置SBT_OPTS环境变量来进行全局配置。

经过以上步骤，sbt执行过程中就不需要访问外网了，因此速度会有很大提升。

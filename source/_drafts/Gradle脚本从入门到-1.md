---
title: Gradle脚本从入门到...(1)
date: 2018-01-19 14:27:21
tags: [Gradle,Android,构建]
---

## 前言

之前也写过一篇关于Gradle的文章，但是想要All In Once，最后写了一篇长文，但是结果很不理想，感觉什么都要讲，但是很多东西又都没有讲清楚。刚好最近突发奇想，想要搭个自己blog，以此为契机把之前的文章再整理一下。言归正传，正文开始。

## Gradle介绍

Gradle是一个项目自动构建工具，属于代码之外，和ant、maven这类老牌的构建工具相比，最大的不同就是不同就是使用groovy脚本语言替代了xml作为项目构建的配置方式。相比于繁琐的xml，groovy无疑在代码简洁和已于理解上有着很大的优势。除此之外Gradle与maven在依赖控制上有着极大的共同之处，甚至Gradle的依赖管理就是在maven的基础上发展而来。

虽然Gradle的使用者大部分都是从AS而来，但是想要了解Gradle最好还是先抛开IDE，拥抱命令行。因为大部分情况下IDE都会对原始的Gradle做一层封装，一来不方便深入学习，二来不利于暴露问题。

### Gradle的安装

Gradle除了最初的从无到有的情况外，基本上不需要手动安装，然而我们最开始还真是一无所有。有两种方式安装Gradle：

1. 直接到[官网](https://gradle.org/releases/) 下载需要的对应平台的版本。
2. 使用包管理器直接下载，目前支持5种包管理，后续可能会增加，[详情请见](https://gradle.org/install/#install) 

安装完成之后，就是配置好对应的环境变量，需要配置的目录为`../gradle../bin` 配置方法请按各自平台进行。配置完成后，使用如下命令

```
$ gradle -v 
```

正常输出Gradle版本，表示安装完成。

#### Gradle Wrapper

前面说Gradle基本不需要手动安装，就是由于Gradle Wrapper的存在。Wrapper是一个使用了特定gradle版本的脚本，能够直接通过此脚本运行Gradle命令，即使此前没有下载过gradle，当然如果没有下载过gradle，wrapper脚本会去下载对应版本的Gradle。

> 使用AS打开github上的新项目时极有可能就会开在Gradle版本下载上，根本上网络问题，可以取消去手动下载。

 生成Wrapper非常简单，直接运行对应命令行指令

```
gradle wrapper
:wrapper

BUILD SUCCESSFUL in 0s
1 actionable task: 1 executed
```

即可生成path内对应版本的wrapper，生成的wraper主要包括如下文件

```
drwxr-xr-x   4 houyi  staff       128  1 29 13:55 .gradle
drwxr-xr-x   3 houyi  staff        96  1 29 13:55 gradle
-rwxr-xr-x   1 houyi  staff      5296  1 29 13:55 gradlew
-rw-r--r--   1 houyi  staff      2260  1 29 13:55 gradlew.bat
```

在gradle目录下






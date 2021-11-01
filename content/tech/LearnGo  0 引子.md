---
title: "LearnGo | 0 引子"
slug: "learn-go-0"
subtitle: ""
defcription: ""
tags:
    - "golang"
    - "learn"
date: 2020-09-15T13:04:31+08:00
draft: true
author: EvernessW


toc: false
katex: false
mermaid: false
---

自进入大学以来，代码已经码了三年，期间接触过的语言种类繁多，C++、Java、Python、Matlab、C#，样样能写但样样不精，仔细想来，自己对于程序语言，始终局限于用而止步于懂。来到大四，恰好有了大量的空余时间，决定选择一门语言细细吃透。

## Why Go？

萌生学习计划之初，考虑过将泛用性更强的C++作为学习对象，但奈何C++内容繁多，担心自己会知难而退，于是退而求其次敲定了风格相近且更为熟悉的Golang取而代之。

最早接触Golang，是由于其是Google的亲儿子，当时我对互联网编程饶有兴趣，于是跟着[飞雪无情](https://www.flysnow.org/)的[Golang gin 实战](https://www.flysnow.org/2019/12/10/golang-gin-quick-start.html)写了一个简单的CURD[后端API](https://git.everness.me/Everness/trace_api)。在开发的过程中，我喜欢上了Golang的简洁，从此一发不可收拾，现在Go已经替代了Python成为了我的小工具首选语言。

相较C系，Go加入了垃圾回收，防止了内存泄漏，同时语法也更加简单，源码可读性高；不过，Go也有着不支持泛型的致命缺陷，使得数据结构的实现变得困难。

## How to Go？

按照[TalkGo](https://talkgo.org/)的建议，我将学习计划分为以下三个方向：

* 常用库使用
* 常用库实现
* 项目开发

在最初的一段时间，我将专注于标准库的[文档](https://golang.org/pkg/)，学习`fmt`、`string`、`test`等标准库的使用，在有一定基础后，我会选择`gin`框架进行源码阅读，了解其实现方式，同时穿插项目的仿写进行巩固提高。我希望这个系列可以保持2-3天的更新频率，并能持续三个月以上的时间。如果系列文章中出现错误，还望读者多多指正。


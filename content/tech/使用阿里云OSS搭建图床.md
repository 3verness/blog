---
title: "使用阿里云OSS搭建图床"
slug: "image-hosting-with-aliyun-oss"
subtitle: ""
defcription: ""
tags:
    - "aliyun"
    - "oss"
    - "PicGo"
    - "tutorial"
date: 2020-04-14T11:54:07+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

很长一段时间，我都是用github作为图床托管博客上的图片，但现在github的访问并不是很稳定，并且有些自己做笔记写日记的图片并不想让其他人直接从仓库看到，因此考虑使用对象存储服务作为图床使用，既能提升国内的访问速度，也能一定程度上防止他人恶意看图。

## 创建对象存储服务

在阿里云上创建Bucket，专门用于存储公共图片：

![](https://img.ioyoi.me/20200414124842.webp)

需要填写一个独一无二的名称以进行访问，并将读写权限更改为公共读。

为了防止token泄露影响其它阿里云服务，这里创建一子用户进行该Bucket的管理，在控制台-访问控制-用户中选择[新建用户](https://ram.console.aliyun.com/users/new)，创建一子用户以管理图床存储。

![image-20200414125646075](https://img.ioyoi.me/20200414125650.webp)

勾选编程访问以生成AccessKey，创建完成后请立即保存该用户的AccessKey，该创建窗口关闭后将无法再次查看AccessKey Secret。

随后返回Buckedt管理页面对该子用户进行授权

![image-20200414130102955](https://img.ioyoi.me/20200414130105.webp)

向其提供读写权限即可

![image-20200414130148785](https://img.ioyoi.me/20200414130149.webp)

## 配置PicGo

上传工具依然还是选用[PicGo](https://picgo.github.io/PicGo-Doc/)

在配置中前四项依次填写之前保存的子用户ID、Secret、Bucket名称和Bucket地址。

![image-20200414130518531](https://img.ioyoi.me/20200414130519.webp)

之后便可以很方便的在[VSCode](https://marketplace.visualstudio.com/items?itemName=Spades.vs-picgo)和[Typora](https://support.typora.io/Upload-Image/#picgoapp-chinese-language-only)中使用PicGo上传图片并替换链接了~
---
title: "谈谈我用过的VPS们"
slug: "talk-about-vps"
subtitle: ""
defcription: ""
tags:
    - "vps"
date: 2020-04-28T18:13:07+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## 前言

前些日子做梯子的VPS IP地址被谷歌学术禁用了，在Vultr刷了好久的IP，一直找不到一个墙内能用的，于是告别了用了快两年的服务器，开始寻找新的服务提供商，正好结合之前用的一些线路体验，整理成了这篇文章。

这也算是一篇评测文，但打分100%主观，推荐仅供参考，想要求真知，建议还是自己折腾测试。

本人网络环境：北京联通手机卡，联通(家)/移动/教育网(学校)宽带，无电信设备。

## [Vultr](https://www.vultr.com/)

这算是使用时间最长的VPS厂商，使用的主要是日本机房和新加坡机房，各自用了一年半左右。日本机房联通移动均为ntt线路，新加坡机房联通为ntt绕日本，移动为直连，速度相当可以。

Vultr最大的优点就是换IP十分方便，重开机器只需要恢复镜像就能继续使用，不需要重新部署。缺点在于用的人实在太多了，晚上联通走的ntt线路必炸，且由于大量梯子教程吸引了许多小白使用vultr，可用IP骤减，每次刷地址都是个痛苦的过程。

推荐指数：**4**，Vultr的性价比还算可以，并且按小时计费，机房众多，可以随时更换IP，对新手比较友好，如果你生活在北上广深且使用联通网络，日本和新加坡的机房还是很值得考虑的。

## [GCP](https://cloud.google.com/gcp/)

Google Cloud Platform是我最早接触的VPS服务，原因是只要你有双币信用卡就能白嫖一年的免费使用，GCP的香港和台湾机房网络线路可以用稳的一批来形容，即便是晚高峰也能做到Youtube 2k秒开，缺点是价格较高且按流量计费，因此用完一年的免费额度后就不再使用了。

推荐指数：**4.5**，如果你不缺钱且求稳的话，谷歌云是相当不错的选择，台湾香港机房相比其他海外机房还有默认中文的一大优势。

## [Banwagon](https://bandwagonhost.com/aff.php?aff=59755)

现在使用的是鼎鼎大名的搬瓦工CN2 GIA DC6机房的服务器，联通移动走的都是CN2 GIA的线路，晚高峰时期基本不受影响，一个星期用下来还是比较稳的，缺点是太贵，49刀3个月的价格，只能说是一分钱一分货。

推荐指数：**4.5**，和谷歌云一样都很稳，在每月10TB+流量的情况下搬瓦工的性价比高于谷歌云，如果能抢到49刀一年的限量版简直血赚。另外网传DC3机房的CN2 GT线路也还可以，价格比GIA线路低了不少，也可以尝试一下。

## [Hostwinds](https://www.hostwinds.com/hosting/)

用过一个月的hostwinds，价格和Vultr持平，均为5刀一月，但并不是按小时计费，这家的优点是外网带宽特别大，轻易能跑出1Gbps的成绩，下载数据库的时候动辄1-200M/s，不过国内使用体验极差，下行带宽只有1-2MBps的样子，基本不能当梯子用，延迟也在300ms以上。

推荐指数：**1**，完全无法满足做梯子的要求。

## [Hostdare](https://www.hostdare.com/)

只用了三天就退款了，由于测试的时候联通的海底光缆恰好断了，因此不能确定下行带宽异常是暂时的还是永久的。这家的特色是China Optimized KVM主机，但是只有电信是CN2优化线路，联通依然是直连，下行带宽感人，Youtube 720p视频依然需要缓冲。另外这家是专有带宽，49刀每年的套餐只有50M的带宽。

推荐指数：**2**，电信用户可以考虑，算比较便宜的CN2线路，联通用户请绕行。

## 小结

如果做一个总结的话，我会推荐手头宽裕的人使用Banwagon的[CN2 GIA主机](https://bwh88.net/aff.php?aff=59775&pid=87)，确实稳定省心，手头紧的同志可以选择白嫖谷歌云、Banwagon的[GT主机](https://bwh88.net/aff.php?aff=59775&pid=57)。

## 拓展阅读

如果想详细了解国内运营商的出口线路，推荐一篇知乎专栏中的[文章](https://zhuanlan.zhihu.com/p/64467370)。

## 题外话：用VPS都做了些什么？

目前这台服务器跑的服务如下：

* 博客 [Hugo](https://gohugo.io/)+[hugo-theme-meme](https://themes.gohugo.io/hugo-theme-meme/)
* 笔记 [Hugo](https://gohugo.io/)+[ace-documentation](https://themes.gohugo.io/ace-documentation/)
* 梯子 [V2Ray](https://www.v2ray.com/)
* 网盘 [Filebrowser](https://github.com/filebrowser/filebrowser)
* 下载器 [aria2](https://aria2.github.io/)+[ariaNG](https://github.com/mayswind/AriaNg)
* Git服务 [gitea](https://gitea.io/)
* 自建的一些后端服务

几乎全都是go语言实现的东西，我其实是gopher的狂热粉了~

另外还有一些曾经用过的服务：

* [为知笔记](https://www.wiz.cn/zh-cn/docker)

  相当于一个自建的印象笔记，支持markdown，不过这个要求至少2G内存，换过服务器后实在是没钱加内存了，于是选择了hugo+ace-docunmentation的替代方案，同样使用markdown写作，使用git-hooks同步更新，缺点是手机上记笔记变得很麻烦。

* Email

  这个自己建着实有点无聊，给谁发邮件都会被归类到垃圾邮件里，没有什么实用价值，因为没有go语言实现就不再使用了。

* 图床

  之前用过一段时间[lychee](https://lychee.electerious.com/)图床，但是由于服务器在国外访问速度不是很理想弃用了，现在使用[阿里云OSS](../使用阿里云oss搭建图床/)作为图床，速度爽到飞起。


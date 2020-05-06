---
title: "WSL, Nice Job!"
slug: "windows-subsystem-for-linux"
subtitle: ""
defcription: ""
tags:
    - "linux"
date: 2020-05-06T20:39:00+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

Windows下的Linux子系统推出已经三年多了，但直至上周我才正式开始使用它，很遗憾之前错过了这么棒的命令行工具。在过去一段时间中，我曾经装过双系统，虚拟机，并在Windows中使用了很长时间的[git-bash](https://gitforwindows.org/)。网上关于Windows Terminal+WSL的配置教程已经很多，这里只想谈谈我眼中WSL的优点。

* 完整的Linux命令行体验

  相较git-bash提供的有限Linux命令，WSL提供了一套完整的Linux操作体验，其基本上可以完成所有Linux环境下的命令行操作，达到无限近似于于双系统的体验。将其shell更换为我更习惯的zsh并用[AutoHotKey](https://www.autohotkey.com/)设置Ctrl+Alt+T的热键后，感觉完全是一个使用Windows桌面的Linux系统。

  ![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200506205404.png)

* 轻量

  除了占用一定的储存空间外，WSL几乎没有任何资源占用，命令行的启动速度与Powershell相当，操作流畅，不会出现虚拟机下输入的卡顿感。

* 资源共享

  文件自动挂载，端口无需映射，可以帮你省去虚拟机上的许多麻烦。此外，在WSL环境下，依然可以运行WIndows上的可执行文件。

  *听说新版本的WSL 2将返回虚拟机实现，共享特性将大打折扣，不是很能理解工程师的设计思路。*

* 优秀的包管理器

  一直认为Windows的一大缺陷是缺少一个类似于brew、pacman一样优秀的包管理器。[^1]而WSL很大程度上可以补足这一点，曾经一次次手动下载更新Hugo和Annie的我终于得到了解放，也希望不久后Microsoft Store能真正的站起来吧。

  ![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200506210724.png)

对于经常在Windows下运行命令行而又不熟悉cmd命令的人，WSL是一个近乎完美的工具。

因此，这一次，我要说，**Nice job, Microsoft!**

[^1]: 不要谈[Chocolatey](https://chocolatey.org/)，用过都知道国内的速度实在感人。
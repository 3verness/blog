---
title: "从Syncthing谈文件同步工具"
slug: "syncthing-and-sync-tools"
subtitle: ""
defcription: ""
tags:
    - "justwrite"
    - "syncthings"
    - "tools"
date: 2022-05-02T13:39:08+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

{{<justwrite 4>}}

重装系统安装[SyncToy](https://www.microsoft.com/en-us/download/search.aspx?q=synctoy)时，发现巨硬已经把它从Download Center中移除，为了同步工位台式机和笔记本上的资料，我开始寻找替代品，最终发现了[Syncthing](https://syncthing.net/)这一神器。

## 优点

Syncthing完美实现了SyncToy的同步功能，同时做出了以下改进：

* 全平台可用。这意味着我可以方便的把照片同步到Pixel白嫖Google相册；
* 使用Scan和Watch两种方式检测文件改动。相较SyncToy的定期扫描，能够在消耗较少资源的情况下保持足够高的同步频率；
* 自带中转功能。Syncthing可以通过搭建Relay服务器实现中转功能，与Owncloud的中转功能不同，它并不需要转存文件至中转服务器，中转服务器仅提供转发流量的功能，因此小容量VPS也能胜任中转任务；同时Syncthing拥有大量[用户贡献的中转服务器](https://relays.syncthing.net/)（即便在国内），传输小文件完全够用。
* 文件过滤功能。类似于`.gitignore`，syncthing提供`.stignore`跳过一些文件的同步，防止一些不必要或差异化的文件传播，例如`node_moudle`。
* 开源。Syncthing的源码和协议均公开，使用所需的发现服务器与中转服务器均可以自行部署，你不会遇到[Resilio Sync](https://www.resilio.com/) (BitTorrent Sync)上Trecker服务器被墙带来的麻烦。

## 不足

当然，在使用Syncthing时也遇到了一些问题，主要集中在文件冲突和文件删除上：

* Syncthing的版本控制功能弱，当其检测到文件冲突时，会将修改日期靠前的文件加入后缀重命名并传播到各个设备，而我更希望它能够像复制文件一样询问具体保留哪一项；尽管由于同步的高频率，这个问题很少遇到，但偶尔生成的长名称文件看着多少有点心烦。

  {{< note info >}} 在Windows下，使用[SyncTrayzor](https://github.com/canton7/SyncTrayzor)客户端时，遇到冲突时会弹窗提示选择保留文件，但目录下仍会留下`~syncthing~{file_name}.tmp`文件。 {{< /note >}}

* 当在A设备删除同步文件夹中的文件时，经常出现Scan后从设备B重新复制一份的情况，导致文件无法被删除。但Syncthing Watch功能使用的`fsnotify`包应当是可以监听到删除动作的，猜测这可能是一个Bug。

* 当使用IDEA等其他工具在同步目录下编辑时，由于频繁的保存会不断触发同步，因此代码工程的同步最好还是交给git完成。

## 检测机制

如上文所述，Syncthing提供Scan和Watch两种方式检测文件改动。

Scan意味着扫描同步文件夹中的全部内容，计算SHA256值与另一设备比较，当出现不同时触发同步传输。这一计算过程消耗大量资源，因此无法实时进行，往往在启动时运行或作为定时任务周期进行。

实时的差异检测通过Watch实现，这一功能在不同操作系统下有着[不同的实现形式](https://github.com/fsnotify/fsnotify)，在大部分文件系统下Watcher可以通过监听文件系统事件完成，例如Linux下的[inotify](https://man7.org/linux/man-pages/man7/inotify.7.html)和Windows下的[ReadDirectoryChangesW](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw)，Watcher的资源消耗远远小于扫描，可以做到数秒周期的运行，保障了同步的实时性。通过监听事件实现的Watcher仅当进程运行时有效，若文件在Watcher未运行时发生更改将无法被检测，因此在启动时需要Scan的配合。

## 传输机制

当尝试自行部署Syncthings服务时，会发现官方提供了Discovery Server与Relay Server两个服务，这与Syncthing传输的工作机理有关。

发现服务器的功能是注册和分发地址。当一台设备启动Syncthing时，会生成一个UID作为设备标识，当全局发现功能启用时，客户端会每隔30分钟向设置的全局发现服务器告知在线状态，同时，当添加远程设备时，客户端也会向发现服务器查询该设备的在线状态与网络地址。

Syncthing的发现服务器仅提供地址信息，既不转发数据也不提供中转，所有的数据传输均是端到端的。对于拥有公网地址或处于同一NAT下的设备，Syncthing可以通过ip直连，而对于没有公网地址的设备，则需要中转服务器进行流量转发。

当前，Syncthing的转发机制是首先尝试直连，直连失败时立刻回落到流量转发，这意味着仅当两台设备均处于同一NAT或公网时才能直连，否则所有流量将通过中转服务器转发，这就给中转服务器带宽带来了很大的压力。实际上，即便两台设备位于不同NAT下，大多数场景下仍然可以通过“[打洞](https://bford.info/pub/net/p2pnat/)”的方式创建点对点的连接。如果中转服务器可以集成这一功能，打洞失败再回落中转，数据传输将不再受中转服务器线路的影响，体验将会更上一层楼。


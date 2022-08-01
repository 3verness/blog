---
title: "走进Torrent"
slug: "what-is-torrent"
subtitle: ""
defcription: ""
tags:
    - "justwrite"
    - "torrent"
date: 2022-08-01T15:04:04+08:00
draft: false
author: EvernessW


toc: true
katex: false
mermaid: false
---

## Torrent是什么？

Torrent是BitTorrent协议的一个索引文件，其本意是“洪流”，我们常把它叫做“种子”，大概是播种一颗“种子”，就能收获一组文件的缘故吧。

若想理解Torrent，首先要了解BT协议是如何工作的。BT协议是最为经典的点对点（peer-to-peer，P2P）协议，其原理是将下载文件分为指定大小的区块，下载时，客户端携带索引文件中包含的特征码向Tracker服务器发出请求，Tracker服务器回复同样请求该文件信息的其他用户地址，随后客户端向其他用户发起连接，并相互交换下载完成的区块，并依照索引文件中的文件结构构造为原始文件，最终实现完整文件的快速下载。

## Torrent中有什么？

### Bencode

Torrent实际上是一个文本文件，当使用编辑器打开时，会发现诸如`d8:announce34:http://mgtracker.org:2710/announce`一类的元数据信息以及一长串的乱码。

让我们先从元数据讲起，Torrent使用一种名为[Bencode](https://zh.m.wikipedia.org/zh-hans/Bencode)的编码方式，其支持字符串、整型数、列表和字典四种数据类型，最大的特点是会使用一个标识类型的字符与`e`包裹所存数据，具体如下：[^1]

* 字符串：无需包裹，但需要在字符串前添加长度，组成`{内容长度}:{内容}`的形式，例如`7:bencode`；
* 整型数：使用`i`与`e`包裹，例如`i4e`、`i-16e`；
* 列表：使用`l`与`e`包裹，随后的每一条数据均为链表中的一个值，一个列表中可包含多种数据类型，例如`l1:ai4ee`对应`["a", 4]`；
* 字典：使用`d`与`e`包裹，随后数据以`键值键值键值...`的方式存放，键与值的数据类型同样不做限制，例如`d1:ai1ei2e1:be`对应`{"a": 1, 2: "b"}`。

使用解码工具解码后，Torrrent文件的内容就很直观了。

```json
{
   "announce": "udp://tracker.openbittorrent.com:80/announce",
   "announce-list": [
      [
         "udp://tracker.openbittorrent.com:80/announce"
      ],
      [
         "udp://tracker.publicbt.com:80/announce"
      ]
   ],
   "comment": "Big Buck Bunny, Sunflower version",
   "created by": "uTorrent/3320",
   "creation date": 1387308159,
   "info": {
      "length": 355856562,
      "name": "bbb_sunflower_1080p_60fps_normal.mp4",
      "piece length": 524288,
      "pieces": "<hex>99 71 9B 2C 2E AA ...</hex>"
   }
}
```

### Torrent内容

一个完整的Torrent文件需要包含以下内容：[^2]

* `announce`：Tracker服务器地址；
* `announce-list`：一系列的备用Tracker服务器地址，由于各个服务器之间并不会同步信息，客户端通常会向多个甚至全部Tracker服务器均发送请求；
* `comment`：用户添加的附加说明；
* `creation date`：发布日期的Unix时间戳；
* `created by`：生成种子的软件信息；
* `info`：种子包含的文件信息。

`info`是种子中最为关键的部分，客户端向Tracker服务器发出请求时，就使用SHA1哈希值作为种子的特征识别码，换言之，只要两个种子的`info`部分完全相同，其余条目的更改不会影响该种子原文件的下载。`info`中包含的内容分为两部分，一部分为区块信息，另一部分为文件结构。

区块信息包括：

* `piece length`：每个区块包含的字节数，即分割区块的大小；
* `pieces`：各区块的20字节SHA1哈希值，其长度对应区块总数。

单文件的种子与包含多个文件的种子的文件结构部分略有差异，仅有单文件的文件结构较为简单，包括：

* `name`：发布者建议的文件名称；
* `length`：文件的字节长度，即文件大小。

多文件的`info`部分如下所示：

```json
	"info": {
      "files": [
         {
            "length": 662807209,
            "path": [
               "[DMG&VCB-Studio] Aquatope of White Sand [01][1080p][x264_aac][tc].mp4"
            ]
         },
         {
            "length": 501972919,
            "path": [
               "[DMG&VCB-Studio] Aquatope of White Sand [02][1080p][x264_aac][tc].mp4"
            ]
         },...
      ],
      "name": "[DMG&VCB-Studio] Aquatope of White Sand [1080p][tc]",
      "piece length": 4194304,
      "pieces": "<hex>9B 46 57 ...</hex>"
   }
```

其文件结构部分包括：

* `name`：发布者建议的文件根目录名称；
* `files`：文件列表；
  * `length`：文件的字节长度，即文件大小；
  * `path`：发布者建议的文件路径。

值得注意的是，文件路径是通过列表储存的，各级目录名称作为`path`中的一条，例如`a/b/c`应储存为`"path": ["a", "b", "c"]`。

随着BT协议的不断发展，Torrent文件也开始包含更多信息，例如`nodes`字段对DHT表的支持和`url-list`对可用HTTP下载地址的支持[^3]，在此不再详述。

### Magnet链

Magnet是BitTorrent的另一种索引方式，相较Torrent文件，Magnet基于DHT网络实现，不依赖Tracker服务器，能够直接在DHT网络中寻址，连接用户开始文件下载。因此磁力链理论上只需要一个特征码即可开始文件下载，例如`magnet:?xt=urn:btih:16c461b2a2437e6b6537d790ca41a9413734e8ad`，`xt`为eXact Topic缩写，是磁力链的特征码，`urn`表示Universe Resource Name，`btih`则表示使用BitTorrent的散列函数，除`btih`外，Magnet链也可以使用`ed2k`、`sha1`、`md5`等多种散列方式。

Magnet可以扩展更多参数[^4]，常见的如：

* `dn`：显示名称；
* `xl`：文件大小；
* `kt`：关键字；
* `tr`：Tracker服务器。

与Torrent文件相比，Magnet不需要客户端访问Tracker服务器，理论上只要你可访问的网段中有正在下载的用户，你便可以开启下载，因此在下载墙外资源时十分好用。不过，无Tracker的Magnet链在开始下载时需要较长的时间从网络中的其他用户处获取元数据，因此大多数资源发布时都会提供带Tracker地址的磁力链，这种下载方式就和使用Torrent文件时无异了。

## Torrent安全么？

在BT下载协议中，每位用户既是下载者，也是上传者，那么，如何保证文件的完整性和一致性呢？可以留意到Torrent中`pieces`部分提供了每个区块的散列值，因此当客户端每完成一个区块的下载时，都会计算该区块的散列值并与记录值比对，若不同，则丢弃重新下载。

那么，如果构建一个与待下载区块同大小的恶意区块，并且两区块拥有同样的散列值，那么其他下载者在获得该恶意区块时可以成功通过校验污染目标机器。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/202208012033116.webp)

从理论上讲，针对现在使用的SHA1散列算法，该种攻击方式是可行的；不过幸运的是，现有破解算法的代价十分高昂，CWI与谷歌安全团队在2017年发布了[SHA1散列碰撞算法](https://shattered.io/)，并以此生成了两份大小相同、内容不同的同散列值文件，此次碰撞攻击进行了九百万亿次散列计算，需要6500年的单CPU计算时间与110年的单GPU计算时间。[^5]这一数字也许对个人计算机来说过于惊人，但对于控制大量肉鸡的黑客与国家而言仍然是可行的，也许Torrent是时候更换更为安全的校验方式了。

![image-20220801213236244](https://awesome-image.oss-cn-beijing.aliyuncs.com/202208012132954.webp)



[^1]:https://zh.m.wikipedia.org/zh-hans/Bencode
[^2]: https://fileformats.fandom.com/wiki/Torrent_file
[^3]: https://en.wikipedia.org/wiki/Torrent_file#Extensions
[^4]: https://zh.wikipedia.org/zh-cn/%E7%A3%81%E5%8A%9B%E9%93%BE%E6%8E%A5
[^5]: https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html

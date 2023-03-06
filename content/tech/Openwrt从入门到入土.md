---
title: "Openwrt从入门到入土"
slug: "openwrt-start-to-end"
subtitle: ""
defcription: ""
tags:
    - "justwrite"
    - "openwrt"
date: 2022-03-16T22:31:08+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

{{<justwrite 3>}}

曾经向往连上路由器就能科学上网的透明代理体验，正好前些日子为了同步笔记本与工位台式机的资料，从校园二手平台上花88入了台小米路由器4a千兆版，获得了一次折腾的机会，但由于种种原因最终拔草弃坑，谨以此文记录一下折腾的过程。

## 准备工作

检查路由器当前系统版本是否能够使用[OpenWRTInvasion](https://github.com/acecilia/OpenWRTInvasion)，若不在支持列表内，建议通过手动升级或[修复工具](https://www.miwifi.com/miwifi_download.html)将系统刷为`firmwares/stock`中的2.28.62版本。

## 解锁ssh

实质上小米路由器的固件就是openwrt魔改而来，所以这里就不用折腾breed等bootloader工具了，直接使用Openwrt中自带的`mtd`命令刷入openwrt。

首先借助[OpenWRTInvasion](https://github.com/acecilia/OpenWRTInvasion)获得root权限并进入路由器后台，运行如下命令：

```sh
git clone git@github.com:acecilia/OpenWRTInvasion.git
cd OpenWRTInvasion
pip install -r requirements.txt
python remote_command_execution_vulnerability.py
```

按照提示输入路由器后台密码即可获得ssh命令，密码默认为`root`。

![](https://img.ioyoi.me/202203021049333.webp)

## 刷入固件

[下载](https://firmware-selector.openwrt.org/?version=21.02.2&target=ramips%2Fmt7621&id=xiaomi_mi-router-4a-gigabit)并安装官方固件包，由于分区限制，只有`/tmp`目录可以容纳固件包，在其他目录下载会提示写入错误。注意需根据版本选择对应的KERNEL包链接，同时可以把SYSUPGRADE包一同下载备用。

```sh
cd /tmp
curl https://downloads.openwrt.org/releases/21.02.2/targets/ramips/mt7621/openwrt-21.02.2-ramips-mt7621-xiaomi_mi-router-4a-gigabit-initramfs-kernel.bin -o openwrt.bin --insecure
mtd -e OS1 -r write openwrt.bin OS1
```

命令行提示Rebooting，路由器电源指示灯恢复蓝色常亮后，刷机成功，可以通过ssh或直接用浏览器打开[luci界面](http://192.168.1.1/cgi-bin/luci/)配置openwrt，密码处输入任意内容，均可进入后台。

![](https://img.ioyoi.me/202203021224639.webp)

## 修复WiFi

首次刷入系统后，进入luci界面会发现没有Wireless选项，这是由于Kernel包缺少了无线驱动导致系统未识别到无线设备。Openwrt论坛中也有不少用户反映该问题，大概率是固件包的缺陷。

该问题可简单地通过重新刷入SYSUPGRADE包解决。luci界面下，在System-Backup/Flash Firmware-Flash new firmware image选项中刷入下载好的同版本SYSUPGRADE包，注意取消勾选保留设置选项，等待刷机结束后luci界面自动刷新。

![](https://img.ioyoi.me/202203021224135.webp)

重新登陆管理界面，Network-Wireless选项卡已恢复，可以点选Enable按钮打开无线功能。

![](https://img.ioyoi.me/202203021225121.webp)

## 更换软件源

未部署科学上网插件前，访问官方软件仓库的速度极慢，可以通过更换国内源解决。值得注意的是，Openwrt默认不支持https证书验证，因此使用https仓库需要额外配置，这里图省事就直接用http了。

```sh
sed -i 's_https://downloads.openwrt.org_http://mirrors.tuna.tsinghua.edu.cn/openwrt_' /etc/opkg/distfeeds.conf
```

## 科学上网 & 弃坑

路由器能正常上网后开始鼓捣科学上网了，这时发现科学上网的插件（如[passwall](https://github.com/xiaorouji/openwrt-passwall)、[ssr-plus+](https://github.com/maxlicheng/luci-app-ssr-plus)、[openclash](https://github.com/vernesong/OpenClash)等）大多不在软件源仓库中，并且路由器上的Openwrt并不能从源码自行编译插件。交叉编译无论是编译打包插件源码的固件还是跨平台编译插件包基本都要整个虚拟机环境。网上流传的插件包也大多是x86版本，且依赖安装极其繁琐。

同时和tg群里的小伙伴们交流了解到，对于这种路由器来说，运行Xray都是勉勉强强，Clash更是毫无可能，遂决定弃坑，也许之后有了树莓派做旁路由之后再来折腾。

## 后记

不得不承认自己现在已经过了喜欢折腾的年纪，不同于之前的刨根问底，现在的需求变成了方便就好、能用就行，曾经折腾的乐趣如今只剩下了麻烦，大概我已经不再向往Geek也不再热爱技术了。


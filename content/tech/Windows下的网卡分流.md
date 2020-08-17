---
title: "Windows下的网卡分流"
slug: "route-on-windows"
subtitle: ""
defcription: ""
tags:
    - "windows"
date: 2020-08-17T10:38:28+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## 引子

身在学校，经常需要在限流校园网和宿舍宽带间反复横跳，每次都要手动切换甚是麻烦，因此研究了下Windows下的路由表，希望能一劳永逸的解决网络分流问题。

我的网络环境及需求如下：

* 无线网卡连接校园网，有线网卡连接宿舍宽带。
* ipv4通过有线网访问，ipv6通过无线网访问。
* 校园网内部地址及部分外网地址通过无线网访问。

## 原理

Windows下的网络请求由路由表`route`管理，按照内置规则按ip地址对请求进行网卡分流。

使用`route PRINT`命令可以查看当前路由表规则。

| 网络目标 | 网络掩码 |         网关          |      接口      | 跃点数 |
| :------: | :------: | :-------------------: | :------------: | :----: |
| 0.0.0.0  | 0.0.0.0  |  192.168.1.1（有线）  | 192.168.1.107  |   25   |
| 0.0.0.0  | 0.0.0.0  | 183.172.240.1（无线） | 183.172.243.31 |   55   |

当同一网络目标对应多个网关时，系统会按照跃点数（Metric）由低到高的优先级进行网关分配。可以注意到，由于系统默认将`0.0.0.0`分配至所有网关，因此所有ipv4地址都将通过跃点数较低的网关进行访问。

`route`命令提供的修改接口如下：

* `ADD`添加规则

  ```powershell
  route ADD <网络目标> MASK <网络掩码> <网关> METRIC <跃点数>
  ```

* `DELETE`删除规则

  ```powershell
  route DELETE <网络目标>
  ```

* `CHANGE`修改规则

  ```powershell
  route CHANGE <网络目标> MASK <网络掩码> <网关> METRIC <跃点数>
  ```

{{< note info >}}

添加`-p`命令可以永久更改，否则为临时更改，重启后失效。

{{< /note >}}

## 实例

在上述的应用场景中，具体操作如下：

* 首先删除默认规则

  ```powershell
  route DELETE 0.0.0.0
  ```

* 添加ipv4缺省值

  ```powershell
  route ADD 0.0.0.0 mask 0.0.0.0 192.168.1.1 –p
  ```

* 针对单ip添加规则（以本站ip为例）

  ```powershell
  route ADD 104.224.179.29 mask 255.255.255.0 183.172.240.1 -p
  ```

* 针对一组ip添加规则（以`166.111.*.*`为例）

  ```powershell
  route ADD 166.111.0.0 mask 255.255.255.0 183.172.240.1 -p
  ```

{{< note warning >}}

在`PRINT`与`DEL`命令中可以使用`*`通配符，但在`ADD`与`CHANGE`中必须使用完整的目标地址。

{{< /note >}}
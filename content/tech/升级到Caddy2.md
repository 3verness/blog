---
title: "升级到Caddy2"
slug: "upgrade-to-caddy2"
subtitle: ""
defcription: ""
tags:
    - "caddy"
    - "tutorial"
date: 2020-07-10T09:18:31+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

两个月前，Caddy更新了v2版本，相比v1版本，其主要特点是：

* localhost也能自动获取HTTPS
* 支持HTTP/3（~~还没有一个浏览器的稳定版支持~~
* 支持Zstandard压缩算法（~~同样还没有一个浏览器的稳定版支持~~
* 更复杂的Caddyfile配置

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200710095257.png)

虽然早就想更新到新版本，但折腾了两次V2Ray都不work，遂一直没有升级。今天发现V2Ray的[新白话文指南](https://guide.v2fly.org/advanced/wss_and_web.html)中给出了Caddy v2的配置方法，故趁热打铁把服务升级至新版本。

我使用Caddy主要是两个功能，静态网站托管与端口反向代理，这里主要介绍两项功能在v1和v2中的区别。

* 静态网站托管

  借助[static file server](https://caddyserver.com/docs/caddyfile/patterns#static-file-server)实现，v1版本中只需要一行：

  ```
  root /path/to/root
  ```

  v2版本中需要更改为：

  ```
  root * /var/www/blog
  file_server
  ```

* 反向代理

  对与HTTP代理，v1版本中依然只需要一行：

  ```
  proxy /path 127.0.0.1:8000
  ```

  但使用WS代理时需要特别说明：

  ```
  proxy /path 127.0.0.1:8000 {
      websocket
  }
  ```

  而v2版本的[reverse_proxy](https://caddyserver.com/docs/caddyfile/patterns#reverse-proxy)默认支持WS代理，这里给出一个通用的反向代理模板：

  ```
  reverse_proxy localhost:8000 {
      header_up Host {http.reverse_proxy.upstream.hostport}
      header_up X-Real-IP {http.request.remote}
      header_up X-Forwarded-For {http.request.remote}
      header_up X-Forwarded-Port {http.request.port}
      header_up X-Forwarded-Proto {http.request.scheme}
  }
  ```

  此外，v2提供[Named matchers](https://caddyserver.com/docs/caddyfile/matchers#named-matchers)来对反向代理提供更加详细的设定，比如V2Ray白话文教程中提供给的样例：

  ```
  @v2ray_websocket {
      path /ray
      header Connection *Upgrade*
      header Upgrade websocket
  }
  reverse_proxy @v2ray_websocket localhost:10000
  ```

* 压缩编码

  v1中：

  ```
  gzip
  ```

  v2中需使用[encode](https://caddyserver.com/docs/caddyfile/directives/encode#encode)命令，顺带把Zstandard压缩开启，相比gzip，[zstd的压缩率和速度均更强](https://engineering.fb.com/core-data/smaller-and-faster-data-compression-with-zstandard/)，万一哪天浏览器支持了呢：

  ```
  encode gzip zstd
  ```

* HTTP/3

  需要在Caddyfile中写入：

  ```
  experimental_http3
  ```

* 自启动

  Caddy提供了官方的[service文件](https://github.com/caddyserver/dist/blob/master/init/caddy.service)，下载覆盖即可。

以上基本就是我从Caddy v1迁移至v2使用的全部了，升级过后并没有什么新特性可用，使用体验也没有任何改善，但至少，短时间内应该不用瞎折腾了吧。


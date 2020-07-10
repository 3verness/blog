---
title: "使用Golang生成Service Worker"
slug: "workbox-in-golang"
subtitle: ""
defcription: ""
tags:
    - "golang"
    - "blog"
date: 2020-07-10T13:17:20+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

当你打开我的博客时，你可能会发现，你可以将博客安装为PWA应用已提供离线访问。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200710132023.png)

这项功能是通过Service Worker中的Workbox实现的，其会在网站根目录下生成`sw.js`的文件，控制静态文件的缓存并依照缓存策略提供请求路由，当更新网站后，需要更新`sw.js`文件以告知浏览器重新缓存新版本。

我的博客及笔记站均是通过git hooks的形式在服务器端进行页面生成与部署的，由于至今没有搞懂npm包的路径应该如何配置，在每次生成时都需要进行一次`npm install`，严重拖慢了`post-receive`脚本的运行速度，影响使用体验，因此决定写一个golang脚本生成`sw.js`。

## Workbox原理

首先对生成的`sw.js`进行一波分析：

```js
importScripts(`https://storage.googleapis.com/workbox-cdn/releases/5.1.3/workbox-sw.js`);

workbox.core.setCacheNameDetails({
    prefix: "Evergarden"
});

workbox.core.skipWaiting();

workbox.core.clientsClaim();

workbox.precaching.precacheAndRoute({
    revision: "6ff07b67de8fe8607ee204ea1bd9f034",
    url: "./index.html"
},...
);

workbox.precaching.cleanupOutdatedCaches();

workbox.routing.registerRoute(/\.(?:png|jpg|jpeg|gif|bmp|webp|svg|ico)$/, new workbox.strategies.CacheFirst({
    cacheName: "images",
    plugins: [new workbox.expiration.ExpirationPlugin({
        maxEntries: 1e3,
        maxAgeSeconds: 2592e3
    }), new workbox.cacheableResponse.CacheableResponsePlugin({
        statuses: [0, 200]
    })]
}));

workbox.googleAnalytics.initialize();
```

可以看到其核心为`precacheAndRoute`与`registerRoute`两个接口，其中`precacheAndRoute`中内容为根目录下所有文件的路径及hash值列表，而`registerRoute`中内容为以正则表达式代表的静态文件及其缓存方案，其中在博客进行版本更新时，仅`precacheAndRoute`中内容会进行更新。浏览器访问相关路径时，会检查`revision`值是否相同，若不相同则进行更新。

## Template实现

明白了`sw.js`的结构，我们需要做的就是遍历public中的文件，生成List并写入`sw.js`中，在Golang中可以使用标准库中的text/template进行代码实现。

首先创建一个模板文件，这里仅关注模板部分，完整文件可参照[`sw-template.js`](https://git.everness.me/Everness/blog/src/branch/master/sw-template.js)：

```js
workbox.precaching.precacheAndRoute([{{- range $index, $element := . -}}
{{- if ne $index 0 -}},
{{- end -}}
{revision:"{{$element.Revision}}",url:"./{{$element.Url}}"}
{{- end -}}]);
```

随后编写Golang脚本，遍历根目录下的所有文件，并计算其MD5值，最后应用模板生成文件。完整代码见[Gitea](https://git.everness.me/Everness/workbox)。

## 效果

速度比`workbox-cli`要快得多，从提交到生成完毕只需要2秒，更主要的是不需要解决依赖问题，单可执行文件即可运行。
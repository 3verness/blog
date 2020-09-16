---
title: "Hugo Shortcodes"
slug: "hugo-shortcodes"
subtitle: ""
defcription: ""
tags:
    - "hugo"
date: 2020-09-15T17:46:32+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

前些日子看上了hexo-butterfly主题的[Note标签]()，试了试Hugo上的[hugo-notice](https://github.com/martignoni/hugo-notice)觉得和主题不是很搭，于是决定用Hugo的Shortcodes自己实现一个。

## 内置Shortcodes

Hugo已经内置了部分[Shortcodes](https://gohugo.io/content-management/shortcodes/#use-hugos-built-in-shortcodes)功能，如gist显示：

```html
{{</* gist spf13 7896402 */>}}
```

{{< gist spf13 7896402 >}}

## 自定义Shortcodes

Hugo中的Shortcodes模板文件需要放入主题路径的`layout/shortcodes`文件夹中，并以`Shortcodes名.html`命名。

Hugo中的Shortcodes主要有两种形式，一种为One Tag式，如：

```html
{{</* figure src="/media/spf13.jpg" title="Steve Francia" */>}}
```

对于该种Shortcodes，访问其参数有两种方式，一种是通过`{{.Get "src"}}`按照属性名获取，另一种是使用`{{index .Param 0}}`按照参数位置进行访问。

另一种Shortcodes为带内容的Shortcodes，如：

```html
{{</* highlight go */>}} A bunch of code here {{</* /highlight */>}}
```

在这种情况下，tag中参数的访问与之前相同，其内容通过`.Inner`访问，如果需要对内容进行Markdown解析，可以使用`markdownify`函数。

## My Shortcodes

### 安装

{{<note warning>}}

该Shortcodes包仅支持MemE主题。

{{</note>}}

* 将短代码仓库添加至`theme`文件夹

  ```bash
  git clone https://git.everness.me/Everness/shortcodes.git themes/shortcodes
  ```

* 将`config.toml`中的主题一行改为

  ```toml
  theme = ["shortcodes", "meme"]
  ```

* 开始使用

### 功能

#### Note

```html
{{</* note primary */>}}
这是一条note
{{</* /note */>}}
```

目前支持的类型有：default, primary, success, info, warning, danger。

{{< note default >}}
这是一条note
{{< /note >}}

{{< note primary >}}
这是一条primary note
{{< /note >}}

{{< note success >}}
这是一条success note
{{< /note >}}

{{< note info >}}
这是一条info note
{{< /note >}}

{{< note warning >}}
这是一条warning note
{{< /note >}}

{{< note danger >}}
这是一条danger note
{{< /note >}}

#### 音乐播放

目前提供两种添加方式，一种通过MetingJS实现，可以实现网易云与QQ音乐的外链播放，如：

```html
{{</* meting "https://music.163.com/#/song?id=32046766" */>}}
```

{{< meting "https://music.163.com/#/song?id=32046766" >}}

另一种则可以手动播放本地歌曲，如：

```html
{{</* <music url="http://music.163.com/song/media/outer/url?id=536623501.mp3" name="Ref:rain" artist="Aimer" cover="http://music.163.com/song/media/outer/url?id=536623501.mp3" lrc="" */>}}
```

{{<note danger>}}

目前已知在相同页面添加两首歌曲会出现bug，等待修复。

{{</note>}}

#### 视频内嵌

优化了Youtube的播放并增加了Bilibili的外链播放器。

```html
{{</* ytb zCLOJ9j1k2Y */>}}
```

{{< ytb zCLOJ9j1k2Y >}}

```html
{{</* bili BV1KJ411C7qF */>}}
```

{{< bili BV1KJ411C7qF >}}

#### 友链

```html
{{</* friend name="Everness" url="https://everness.me" logo="https://everness.me/favicon.ico" motto="To discover. To change." */>}}
```



{{< friend name="Everness" url="https://everness.me" logo="https://everness.me/favicon.ico" motto="To discover. To change." >}}
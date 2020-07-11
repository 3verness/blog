---
title: "ME"
date: 2020-01-30T18:05:43+08:00
draft: false
---

欢迎来到Evergarden！这是一个佛系大学生的一亩三分地，在这里撒泼打滚，思考人生。

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/aplayer@1.10.1/dist/APlayer.min.css">

<div id="player">
    <pre class="aplayer-lrc-content">
        [00:19.220]On bended knee is no way to be free
        [00:23.940]Lifting up an empty cup, I ask silently
        [00:29.110]All my destinations will accept the one that's me
        [00:33.460]So I can breathe...
        [00:38.900]Circles they grow and they swallow people whole
        [00:43.820]Half their lives they say goodnight to wives they'll never know
        [00:48.460]A mind full of questions, and a teacher in my soul
        [00:52.960]And so it goes...
        [00:58.230]Don't come closer or I'll have to go
        [01:02.890]Holding me like gravity are places that pull
        [01:07.610]If ever there was someone to keep me at home
        [01:11.770]It would be you...
        [01:17.450]Everyone I come across, in cages they bought
        [01:22.200]They think of me and my wandering, but I'm never what they thought
        [01:27.160]I've got my indignation, but I'm pure in all my thoughts
        [01:31.300]I'm alive...
        [01:37.180]Wind in my hair, I feel part of everywhere
        [01:42.190]Underneath my being is a road that disappeared
        [01:46.860]Late at night I hear the trees, they're singing with the dead
        [01:51.290]Overhead...
        [01:56.840]Leave it to me as I find a way to be
        [02:01.390]Consider me a satellite, forever orbiting
        [02:06.070]I knew all the rules, but the rules did not know me
        [02:10.730]Guaranteed
    </pre>
</div>

<script src="https://cdn.jsdelivr.net/npm/aplayer@1.10.1/dist/APlayer.min.js"></script>

<script type="text/javascript">
const ap = new APlayer({
    container: document.getElementById('player'),
    lrcType: 2,
    audio: [{
        name: 'Guaranteed',
        artist: 'Eddie Vedder',
        url: 'http://music.163.com/song/media/outer/url?id=1304038.mp3',
        cover: 'http://p1.music.126.net/E5q7w2l7xiZijFAJmmiEKw==/18159534045169320.jpg?param=130y130',
    }],
});
</script>

## Why Blog？

**Recording** first, then **sharing**. 

写博客的一大目的是帮助自己养成记录的习惯，另外也希望锻炼一下自己的表达能力。现在回溯旧博客，往往会感到一些幼稚，因此旧站上的内容将不会迁移过来了，也许不久的将来，也会觉得现在的文字无趣而可笑，这大概就是成长吧。

如果您想要关注我的博客，可以使用[Atom](../atom.xml)或[RSS](../rss.xml)，也随时欢迎您的[交流指正](mailto:wangshaohang.0416.china@gmail.com)。



## Pieces

我将博客分为三个板块，分别是<font color="#f9c">日常</font>，<font color="#7eb">技术</font>和<font color="#9cf">白日梦</font>。

#### [<font color="#f9c">LIFE</font>](../life/)

这里记录的是生活中转瞬而逝的所思所想，也许过于悲观会引起您的不适，但那只是口中抱怨，我还在努力，挣扎着生活。

#### [<font color="#7eb">TECH</font>](../tech/)

这里存放了一些文档资料，内容杂而不精，一如学业与生活。

#### [<font color="#9cf">DREAM</font>](../dream/)

最后是一些书评、影评及游记，大概是惨淡人生中的偷闲即闲，知足便足吧。



## Thanks

博客自豪地由 [Hugo](https://gohugo.io/) 驱动，主题是 [MemE](https://github.com/reuixiy/hugo-theme-meme) ，使用 [Git Hook](https://git.everness.me/Everness/blog) 持续部署，由 [Service Worker](https://developers.google.com/web/fundamentals/primers/service-workers/) 提供PWA支持。
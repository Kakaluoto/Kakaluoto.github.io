---
title: 【多媒体】ffmpeg将webm格式转换成mp4
date: 2021-10-25
tags: [多媒体,ffmpeg]
cover: https://www.helloimg.com/images/2022/02/14/Gg1D2o.webp
mathjax: true
---

# 起因
手头有一部4K风景视频，辛辛苦苦从油管上下载下来，想要用wallpaper engine做成壁纸，却发现格式是webm,vp9编码，wallpaper engine并不支持，所以想要将格式进行转换。
怎么说呢，感觉还是ffmpeg更方便，就是参数很难调。所以在此记录一下。


# 转换成HEVC编码(H.265)
因为是使用的硬件编码，[硬件编码教程](https://www.jianshu.com/p/1d645b9d26d5)

```bash
ffmpeg -c:v vp9_cuvid -i video.webm -c:v hevc_nvenc -cq 0 video.mp4
```

参数解释：

+  -c:v ：指定编解码器
+ -i : 指定输入参数
+ vp9_cuvid : 使用nvdia硬件vp9解码器
+ hevc_enc:   使用nvdia硬件h.265编码器
+ -cq : 也就是软压的-crf参数很类似,控制码率的参数，0最低对应画质好，数字越大越差

转换之后发现wallpaper engine不支持hevc，麻了，而且码率输入进去的有20000K左右，出来只有2000K了，压得实在太狠了。

# 转换成AVC编码(h.264)
考虑到想要无损编码，至少码率不要损失得太严重，所以找到了新的解决办法

根据[这篇博客](https://blog.csdn.net/ETalien_/article/details/102931065)

这次采用的是硬解软编
输出直接用CPU编码了

```bash
ffmpeg -c:v vp9_cuvid -i video.webm -c:v libx264 -preset ultrafast -qp 0 video.mp4
```
参数解释：

+ libx264:   使用软件h.264编码器
+ -qp : constant quantizer恒定量化器模式,并且画质为无损的画质
+ -preset : 使用预设参数，preset由快到慢可以分为：ultrafast、superfast、veryfast、faster、fast、medium、slow、slower、veryslow、placebo这9个等级
+ ultralfast预设参数最快 

好家伙，最快的情况下CPU编码比GPU还快，但是输出码率超级无敌大，简直就是直接往硬盘里面写数据罢了，换成fast试过，马上就慢下来不敌GPU了。

最后委屈求全，手动指定码率，并且采用h.264硬件编码

```bash
ffmpeg -c:v vp9_cuvid -i video.webm -c:v h264_nvenc -b:v 23000k video.mp4
```
参数解释：

+ h264_nvenc:   使用硬件h.264编码器
+ -b:v : 控制平均码率

总算是解决了，即把视频转码了，也没损失太多码率保证了画质，然后可以上传wallpaper engine了！

# 参考文章：
[ffmpeg：码率控制模式、编码方式](https://blog.csdn.net/ETalien_/article/details/102931065) 
[ffmpeg实例，比特率码率（-b）、帧率（-r）和文件大小（-fs）相关操作](https://blog.csdn.net/yu540135101/article/details/84346146)
[ffmpeg实现硬件转码（使用FFmpeg调用NVIDIA GPU实现H265转码H264）](https://blog.csdn.net/qq_22633333/article/details/107701301)
[阅读3分 | ffmpeg无损转换mp4到webm可不可行？为你揭晓答案](https://cloud.tencent.com/developer/article/1641894)
[如何用FFmpeg将影像转成H.265/HEVC格式？](https://magiclen.org/ffmpeg-h265/)

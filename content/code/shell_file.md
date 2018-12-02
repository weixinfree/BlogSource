---
title: "终端生产力-file 命令"
date: 2018-11-26T20:49:00+08:00
draft: true
---

# A knife and your job


**bug 现象：**

> 线上反馈，[映客App](http://inke.cn/)大厅的一个Banner图在Android上不显示，iOS正常显示

**附加信息：**

> 有小伙伴拿到图片地址：http://img.ikstatic.cn/MTU0MjU0Mzc0NDg0MCM0I2pwZw==.jpg

**定位工具：**
```bash
~>> http http://img.ikstatic.cn/MTU0MjU0Mzc0NDg0MCM0I2pwZw==.jpg > demo.jpg  
~>> file demo.jgp
demo.jpg: TIFF image data, big-endian, direntries=15, height=171, bps=4, compression=none, PhotometricIntepretation=RGB, orientation=upper-left, width=620
```

> TIFF格式并不是常见的图片格式, 依稀记得Android平台原生支持的图片格式不包括TIFF。快速查了一下[Android支持的图片格式](https://yq.aliyun.com/articles/633621)

主要内容引用如下：

> Android 的图片编码解码是由 Skia 图形库负责的，Skia 通过挂接第三方开源库实现了常见的图片格式的编解码支持。目前来说，Android 原生支持的格式只有 JPEG、PNG、GIF、BMP 和 WebP (Android 4.0 加入)，在上层能直接调用的编码方式也只有 JPEG、PNG、WebP 这三种。目前来说 Android 还不支持直接的动图编解码。

> iOS 底层是用 ImageIO.framework 实现的图片编解码。目前 iOS 原生支持的格式有：JPEG、JPEG2000、PNG、GIF、BMP、ICO、TIFF、PICT，自 iOS 8.0 起，ImageIO 又加入了 APNG、SVG、RAW 格式的支持。在上层，开发者可以直接调用 ImageIO 对上面这些图片格式进行编码和解码。对于动图来说，开发者可以解码动画 GIF 和 APNG、可以编码动画 GIF。

[TIFF格式-wiki](https://zh.wikipedia.org/wiki/TIFF)

**BUG 解!**


> 记录有趣的，奇葩的bug
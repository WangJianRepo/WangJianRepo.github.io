---
layout: post
title: 基于压缩域的视频关键帧提取
excerpt: "在压缩域中提取视频I帧作为关键帧, 可用于视频关键帧提取、视频拷贝检测、视频检索等应用中."
categories: [video compression, key frame extraction]
comments: true
---

![I_P_and_B_frames](https://upload.wikimedia.org/wikipedia/commons/6/64/I_P_and_B_frames.svg "A sequence of video frames, consisting of two keyframes (I), one forward-predicted frame (P) and one bi-directionally predicted frame (B).
")

## 参考引用

* [Video compression picture types](https://en.wikipedia.org/wiki/Video_compression_picture_types)  
* [H.264码流以及H.264编解码的基本概念](https://maxwellqi.github.io/ios-h264-summ/)  
* [视频格式那么多，MP4/RMVB/MKV/AVI 等，这些视频格式与编码压缩标准 mpeg4，H.264.H.265 等有什么关系？](https://www.zhihu.com/question/20997688)  


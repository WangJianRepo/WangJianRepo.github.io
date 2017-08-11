---
layout: post
title: 基于压缩域的视频关键帧提取
excerpt: "在压缩域中提取视频I帧作为关键帧, 可用于视频关键帧提取、视频拷贝检测、视频检索等应用中."
categories: [video compression, key frame extraction]
comments: true
---

![I_P_and_B_frames](https://upload.wikimedia.org/wikipedia/commons/6/64/I_P_and_B_frames.svg "A sequence of video frames, consisting of two keyframes (I), one forward-predicted frame (P) and one bi-directionally predicted frame (B).
")

## FFMPEG读取视频相关

在解码之前的步骤int ret = av_read_frame(ic, &avpkt);中，通过判断包的flag就可以判断帧是否是关键帧了，此方法可以省去解码过程。

{% highlight cpp %}

AVPacket avpkt;
av_init_packet(&avpkt);

while (true) {
  //解封装
  int ret = av_read_frame(pFormatCtx, &avpkt);
  if (ret < 0) {
    break;
  }
  // 判断是音频帧还是视频帧
  int video_stream_idx = av_find_best_stream(pFormatCtx, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);
  if (avpkt.stream_index == video_stream_idx) {
    LOGD("read a video frame");
    // 判断是否为关键帧
    // 方法1：通过包标志位判断
    if (avpkt.flags & AV_PKT_FLAG_KEY) {
      LOGD("read a key frame");
    }
    // 方法2：通过解码后帧类型判断
    avcodec_decode_video(pCodecCtx, pFrame, &frameFinished, avpkt.data, avpkt.size);
    if(frameFinished)
    {
      if(pFrame->key_frame==1) // 这就是关键帧
      {}
    }
}

av_free_packet(&avpkt);

{% endhighlight %}

* [FFMPEG Tips (3) 如何读取每一帧的信息](http://ticktick.blog.51cto.com/823160/1872008)  
* [ffmpeg解码流程及解码跟踪和关键问题解析](http://blog.csdn.net/heng615975867/article/details/21602745)  
* [ffmpeg如何提取视频的关键帧？](http://www.dewen.net.cn/q/725/ffmpeg%E5%A6%82%E4%BD%95%E6%8F%90%E5%8F%96%E8%A7%86%E9%A2%91%E7%9A%84%E5%85%B3%E9%94%AE%E5%B8%A7%EF%BC%9F)  
* [FFmpeg 入门(1)：截取视频帧](http://www.samirchen.com/ffmpeg-tutorial-1/)  
* [FFmpeg Here is a list of all examples](https://ffmpeg.org/doxygen/trunk/examples.html) 
* [filtering_video.c](https://ffmpeg.org/doxygen/trunk/filtering_video_8c-example.html)  
* [FFmpeg源代码结构图 - 解码](http://blog.csdn.net/leixiaohua1020/article/details/44220151)  
* [ffmpeg 源代码简单分析 ： av_read_frame()](http://blog.csdn.net/leixiaohua1020/article/details/12678577)  
* [av_read_frame读出的视频流数据在AVPacket中的存储](http://blog.csdn.net/dancing_night/article/details/45742905)  




## 参考引用

* [Video compression picture types](https://en.wikipedia.org/wiki/Video_compression_picture_types)  
* [H.264码流以及H.264编解码的基本概念](https://maxwellqi.github.io/ios-h264-summ/)  
* [视频格式那么多，MP4/RMVB/MKV/AVI 等，这些视频格式与编码压缩标准 mpeg4，H.264.H.265 等有什么关系？](https://www.zhihu.com/question/20997688)  


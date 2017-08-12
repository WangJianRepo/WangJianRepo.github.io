---
layout: post
title: 视频压缩域中的帧类型与应用
excerpt: "本文介绍了帧类型，并提出基于I帧定位镜头关键帧，同时给出了基于FFMPEG的实现代码。"
categories: [video compression, key frame extraction, video sampling]
comments: true
---

## 引言
由于视频相关项目需要，最近需要研究不考虑音频的大规模视频的采样策略。因此，采样策略的有两个基本要求：  
* 代表性强：采样结果能够表征原始视频的主要内容，即采样结果是该视频的主要成分，尽可能去除无用信息
* 速度快：采样速度要尽可能的快，即消耗资源少，才能快速处理大规模视频

比较直接的方法就是先做基于帧间差的镜头分割，在每个镜头中取代表帧，从所有镜头的中取出的代表帧作为最终的采样结果。这个方法的优点是简单，容易实现；缺点也显而易见：  
* 阈值不好确定：分割镜头的帧间差阈值不好确定
* 大量重复帧：镜头中的代表帧会有大量重复
* 速度过慢：由于该方法需要解码大量的视频帧，并做帧间差，耗时耗资源，在实际工程应用中基本不可用。

需要大量解码，是导致速度慢的主要原因，现有大部分视频采样方法的痛点。既然视频要解码，那么为什么要对视频编码？编码即压缩，解码即解压缩，如果不压缩，视频占用空间太大。那视频是怎么压缩法？能否在压缩上找线索，寻找采样策略？能否不解码找到镜头分割点，进而找到关键帧？

## 视频编解码

压缩域中，帧通常分为I帧、P帧和B帧。其中，I帧作为关键帧, 可用于视频关键帧提取、视频拷贝检测、视频检索等应用中。

![I_P_and_B_frames](https://upload.wikimedia.org/wikipedia/commons/6/64/I_P_and_B_frames.svg "A sequence of video frames, consisting of two keyframes (I), one forward-predicted frame (P) and one bi-directionally predicted frame (B).
")

## FFMPEG读取视频相关

### FFMPEG命令找关键帧

FFMPEG命令过滤I帧命令：[Escaping characters](https://trac.ffmpeg.org/wiki/FilteringGuide#Escapingcharacters)

{% highlight shell %}
// FFMpeg 3.3.3 测试通过
ffmpeg -i input -vf select='eq(pict_type\,I)' -vsync vfr output_%04d.png        # to select only I frames
{% endhighlight %}

此命令确实能够找出I帧，与原视频比对后，基本没有问题，验证了找I帧的可行性。但这种方法找I帧速度较慢，CPU占用较高，i7-4770占用50%以上，24min的1080p，25fps的Mp4视频，耗时30s左右，故猜测这个命令实际上对每一帧都解码了，因此需要寻找一种方法：能否在不解码的条件下，判断帧类型。

猜想：在视频文件中，每一帧应该都有头信息，这些都信息不应该是解码后才得到的，而是实现写在了文件中，因此没准可以直接读取头信息，判断其帧类型。

看了[FFMPEG Tips (3) 如何读取每一帧的信息](http://ticktick.blog.51cto.com/823160/1872008) 和 [ffmpeg如何提取视频的关键帧？](http://www.dewen.net.cn/q/725/ffmpeg%E5%A6%82%E4%BD%95%E6%8F%90%E5%8F%96%E8%A7%86%E9%A2%91%E7%9A%84%E5%85%B3%E9%94%AE%E5%B8%A7%EF%BC%9F) 两篇文章后，基本验证可行性。

### 非解码方法找关键帧

在上面两篇博客中，可以发现，在解码之前的步骤int ret = av_read_frame(fmt_ctx, &packet)中，通过判断包的flag就可以判断帧是否是关键帧了([av_read_frame读出的视频流数据在AVPacket中的存储](http://blog.csdn.net/dancing_night/article/details/45742905))，此方法可以省去解码过程。参考[filtering_video.c](https://ffmpeg.org/doxygen/trunk/filtering_video_8c-example.html)写出如下代码：

{% highlight cpp %}
// FFMpeg 3.3.3 测试通过
// 此代码可以在不解码的情况下找出I帧位置，但无法解码，有待改进
#ifdef __cplusplus   
extern "C" {
#endif  

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/*Include ffmpeg header file*/

#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavfilter/buffersrc.h>
#include <libavutil/opt.h>


  static AVFormatContext *fmt_ctx;
  static AVCodecContext *dec_ctx;
  static int video_stream_index = -1;

  static int open_input_file(const char *filename)
  {
    int ret;
    AVCodec *dec;

    if ((ret = avformat_open_input(&fmt_ctx, filename, NULL, NULL)) < 0) {
      av_log(NULL, AV_LOG_ERROR, "Cannot open input file\n");
      return ret;
    }

    if ((ret = avformat_find_stream_info(fmt_ctx, NULL)) < 0) {
      av_log(NULL, AV_LOG_ERROR, "Cannot find stream information\n");
      return ret;
    }

    /* select the video stream */
    ret = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_VIDEO, -1, -1, &dec, 0);
    if (ret < 0) {
      av_log(NULL, AV_LOG_ERROR, "Cannot find a video stream in the input file\n");
      return ret;
    }
    video_stream_index = ret;

    /* create decoding context */
    dec_ctx = avcodec_alloc_context3(dec);
    if (!dec_ctx)
      return AVERROR(ENOMEM);
    avcodec_parameters_to_context(dec_ctx, fmt_ctx->streams[video_stream_index]->codecpar);
    av_opt_set_int(dec_ctx, "refcounted_frames", 1, 0);

    /* init the video decoder */
    if ((ret = avcodec_open2(dec_ctx, dec, NULL)) < 0) {
      av_log(NULL, AV_LOG_ERROR, "Cannot open video decoder\n");
      return ret;
    }

    return 0;
  }

  int main(int argc, char **argv)
  {
    int ret;
    AVPacket packet;
    AVFrame *frame = av_frame_alloc();

    if (!frame) {
      perror("Could not allocate frame");
      exit(1);
    }
    if (argc != 2) {
      fprintf(stderr, "Usage: %s file\n", argv[0]);
      exit(1);
    }

    av_register_all();
    avfilter_register_all();

    if ((ret = open_input_file(argv[1])) < 0)
      goto end;

    // 遍历所有packet
    int frame_num = 0;
    while (1) {
      // 解封装
      if ((ret = av_read_frame(fmt_ctx, &packet)) < 0)
        break;
      // 判断是音频帧还是视频帧
      if (packet.stream_index == video_stream_index) {
        // 判断是否为关键帧
        // 方法1：通过包标志位判断
        if (packet.flags == AV_PKT_FLAG_KEY) {	// Key Frame
          printf("I Frame: %d\n", frame_num);

          // 方法2：通过解码后帧类型判断
          while (ret >= 0) {
            ret = avcodec_receive_frame(dec_ctx, frame);
            if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
              printf("Fail to decode: %d\n", frame_num);
              break;
            }
            else if (ret < 0) {
              printf("Error while receiving a frame from the decoder: %d\n", frame_num);
              av_log(NULL, AV_LOG_ERROR, "Error while receiving a frame from the decoder\n");
              goto end;
            }
            // Processing
            if (frame->key_frame == 1) // Key Frame
            {
              printf("I Frame: %d\n", frame_num);
            }
            printf("Decode: %d\n", frame_num);
          }
        }
      }
      av_packet_unref(&packet);
      frame_num++;
    }
  end:
    avcodec_free_context(&dec_ctx);
    avformat_close_input(&fmt_ctx);
    av_frame_free(&frame);

    if (ret < 0 && ret != AVERROR_EOF) {
      fprintf(stderr, "Error occurred: %d\n", ret);
      exit(1);
    }

    exit(0);
  }

#ifdef __cplusplus   
}
#endif   

{% endhighlight %}

经粗略实验，24min的1080p，25fps的Mp4视频，此方法可以秒出结果。需要注意的是，packet还有一个属性data，存放着未解码的视频内容，因此，可以取消复制这些数据，减少IO，进一步加快速度，但对速度的提升效果如何，需要进一步尝试。粗略定了一下位置，基本可以从此入手优化该点：[memcpy(dst_data, src_sd->data, src_sd->size);](https://github.com/FFmpeg/FFmpeg/blob/949debd1d1df3a96315b3a3083831162845c1188/libavformat/utils.c#L1674)

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


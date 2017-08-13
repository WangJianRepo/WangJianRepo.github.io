---
layout: post
title: 视频压缩域中的帧类型与应用
excerpt: "本文介绍了帧类型，并提出基于I帧定位镜头关键帧，同时给出了基于FFMPEG的实现代码。"
categories: [video compression, key frame extraction, video sampling]
comments: true
---

## 1 引言
&emsp;&emsp;由于视频相关项目需要，最近需要研究不考虑音频的大规模视频的采样策略。因此，采样策略的有两个基本要求：  
* 代表性强：采样结果能够表征原始视频的主要内容，即采样结果是该视频的主要成分，尽可能去除无用信息
* 速度快：采样速度要尽可能的快，即消耗资源少，才能快速处理大规模视频

&emsp;&emsp;比较直接的方法就是先做基于帧间差的镜头分割，在每个镜头中取代表帧，从所有镜头的中取出的代表帧作为最终的采样结果。这个方法的优点是简单，容易实现；缺点也显而易见：  
* 阈值不好确定：分割镜头的帧间差阈值不好确定
* 大量重复帧：镜头中的代表帧会有大量重复
* 速度过慢：由于该方法需要解码大量的视频帧，并做帧间差，耗时耗资源，在实际工程应用中基本不可用。

&emsp;&emsp;需要大量解码，是导致速度慢的主要原因，现有大部分视频采样方法的痛点。既然视频要解码，那么为什么要对视频编码？编码即压缩，解码即解压缩，如果不压缩，视频占用空间太大。那视频是怎么压缩法？能否在压缩上找线索，寻找采样策略？能否不解码找到镜头分割点，进而找到关键帧？

## 2 视频编解码
### 2.1 基本概念
&emsp;&emsp;[视频编码](https://zh.wikipedia.org/wiki/%E8%A7%86%E9%A2%91%E7%BC%96%E7%A0%81)是指连续图像的编码，与静态图像编码着眼于消除图像内的冗余信息相对，视频编码主要通过消除连续图像之间的时域冗余信息来压缩视频。[视频压缩](https://zh.wikipedia.org/wiki/%E8%A6%96%E8%A8%8A%E5%A3%93%E7%B8%AE)（英语：Video compression）是指运用数据压缩技术将数字视频数据中的冗余信息去除，降低表示原始视频所需的数据量，以便视频数据的传输与存储。实际上，原始视频数据的数据量往往过大，例如未经压缩的电视质量视频数据的比特率高达216Mbps，绝大多数的应用无法处理如此庞大的数据量，因此视频压缩是必要的。目前最新的视频编码标准为ITU-T视频编码专家组（VCEG）和ISO／IEC动态图像专家组（MPEG）联合组成的联合视频组（JVT，Joint Video Team）所提出的[H.264/AVC](http://ip.hhi.de/imagecom_G1/assets/pdfs/JVT-G050.pdf)。本文以H.264编码为例，介绍编解码技术。H.264的介绍主要摘自[H.264码流以及H.264编解码的基本概念](https://maxwellqi.github.io/ios-h264-summ/)。

### 2.2 H.264编码原理
&emsp;&emsp;H.264是新一代的编码标准，以高压缩高质量和支持多种网络的流媒体传输著称，在编码方面，[我理解](https://maxwellqi.github.io/ios-h264-summ/)的他的理论依据是：参照一段时间内图像的统计结果表明，在相邻几幅图像画面中，一般有差别的像素只有10%以内的点,亮度差值变化不超过2%，而色度差值的变化只有1%以内。所以对于一段变化不大图像画面，我们可以先编码出一个完整的图像帧A，随后的B帧就不编码全部图像，只写入与A帧的差别，这样B帧的大小就只有完整帧的1/10或更小！B帧之后的C帧如果变化不大，我们可以继续以参考B的方式编码C帧，这样循环下去。这段图像我们称为一个序列（序列就是有相同特点的一段数据），当某个图像与之前的图像变化很大，无法参考前面的帧来生成，那我们就结束上一个序列，开始下一段序列，也就是对这个图像生成一个完整帧A1，随后的图像就参考A1生成，只写入与A1的差别内容。 

&emsp;&emsp;在H264协议里定义了三种帧，完整编码的帧叫I帧，参考之前的I帧生成的只包含差异部分编码的帧叫P帧，还有一种参考前后的帧编码的帧叫B帧。

&emsp;&emsp;H264采用的核心算法是帧内压缩和帧间压缩，帧内压缩是生成I帧的算法，帧间压缩是生成B帧和P帧的算法。

对**序列**的说明：

&emsp;&emsp;在H264中图像以序列为单位进行组织，一个序列是一段图像编码后的数据流，以I帧开始，到下一个I帧结束。

<center>![I_P_and_B_frames](https://upload.wikimedia.org/wikipedia/commons/6/64/I_P_and_B_frames.svg "A sequence of video frames, consisting of two keyframes (I), one forward-predicted frame (P) and one bi-directionally predicted frame (B).
")</center>

&emsp;&emsp;一个序列的第一个图像叫做 IDR 图像（立即刷新图像），IDR 图像都是 I 帧图像。H.264 引入 IDR 图像是为了解码的重同步，当解码器解码到 IDR 图像时，立即将参考帧队列清空，将已解码的数据全部输出或抛弃，重新查找参数集，开始一个新的序列。这样，如果前一个序列出现重大错误，在这里可以获得重新同步的机会。IDR图像之后的图像永远不会使用IDR之前的图像的数据来解码。

&emsp;&emsp;一个序列就是一段内容差异不太大的图像编码后生成的一串数据流。当运动变化比较少时，一个序列可以很长，因为运动变化少就代表图像画面的内容变动很小，所以就可以编一个I帧，然后一直P帧、B帧了。当运动变化多时，可能一个序列就比较短了，比如就包含一个I帧和3、4个P帧。

### 2.3 H.264的压缩方法

**分组:** 把几帧图像分为一组(GOP，也就是一个序列),为防止运动变化,帧数不宜取多。  
**定义帧:** 将每组内各帧图像定义为三种类型,即I帧、B帧和P帧;  
**预测帧:** 以I帧做为基础帧,以I帧预测P帧,再由I帧和P帧预测B帧;  
**数据传输:** 最后将I帧数据与预测的差值信息进行存储和传输。  

&emsp;&emsp;帧内（Intraframe）压缩也称为空间压缩（Spatial compression）。当压缩一帧图像时，仅考虑本帧的数据而不考虑相邻帧之间的冗余信息，这实际上与静态图像压缩类似。帧内一般采用有损压缩算法，由于帧内压缩是编码一个完整的图像，所以可以独立的解码、显示。帧内压缩一般达不到很高的压缩，跟编码jpeg差不多。  

&emsp;&emsp;帧间（Interframe）压缩的原理是：相邻几帧的数据有很大的相关性，或者说前后两帧信息变化很小的特点。也即连续的视频其相邻帧之间具有冗余信息,根据这一特性，压缩相邻帧之间的冗余量就可以进一步提高压缩量，减小压缩比。帧间压缩也称为时间压缩（Temporal compression），它通过比较时间轴上不同帧之间的数据进行压缩。帧间压缩一般是无损的。帧差值（Frame differencing）算法是一种典型的时间压缩法，它通过比较本帧与相邻帧之间的差异，仅记录本帧与其相邻帧的差值，这样可以大大减少数据量。

&emsp;&emsp;顺便说下有损（Lossy ）压缩和无损（Lossy less）压缩。无损压缩也即压缩前和解压缩后的数据完全一致。多数的无损压缩都采用RLE行程编码算法。有损压缩意味着解压缩后的数据与压缩前的数据不一致。在压缩的过程中要丢失一些人眼和人耳所不敏感的图像或音频信息,而且丢失的信息不可恢复。几乎所有高压缩的算法都采用有损压缩,这样才能达到低数据率的目标。丢失的数据率与压缩比有关,压缩比越小，丢失的数据越多,解压缩后的效果一般越差。此外,某些有损压缩算法采用多次重复压缩的方式,这样还会引起额外的数据丢失。

### 2.4 I帧、P帧和B帧

#### 2.4.1 I帧

&emsp;&emsp;为了更好地理解I帧的概念，我罗列了两种解释：

* 帧内编码帧 ，I帧表示关键帧，你可以理解为这一帧画面的完整保留，如同JPG和BMP图片；解码时只需要本帧数据就可以完成（因为包含完整画面）。

* 帧内编码帧 又称intra picture，I 帧通常是每个 GOP（MPEG 所使用的一种视频压缩技术）的第一个帧，经过适度地压缩，做为随机访问的参考点，可以当成图象。I帧可以看成是一个图像经过压缩后的产物。

I帧的特点：

* 它是一个全帧压缩编码帧。它将全帧图像信息进行JPEG压缩编码及传输  
* 解码时仅用I帧的数据就可重构完整图像  
* I帧描述了图像背景和运动主体的详情  
* I帧不需要参考其他画面而生成
* I帧是P帧和B帧的参考帧(其质量直接影响到同组中以后各帧的质量)
* I帧是帧组GOP的基础帧(第一帧),在一组中只有一个I帧
* I帧不需要考虑运动矢量
* I帧所占数据的信息量比较大

#### 2.4.2 P帧

为了更好地理解P帧的概念，我也罗列了两种解释：

* 前向预测编码帧。P帧表示的是这一帧跟之前的一个关键帧（或P帧）的差别，解码时需要用之前缓存的画面叠加上本帧定义的差别，生成最终画面。（也就是差别帧，P帧没有完整画面数据，只有与前一帧的画面差别的数据）
* 前向预测编码帧 又称predictive-frame，通过充分将低于图像序列中前面已编码帧的时间冗余信息来压缩传输数据量的编码图像，也叫预测帧

&emsp;&emsp;P帧的预测与重构：P帧是以I帧为参考帧,在I帧中找出P帧“某点”的预测值和运动矢量,取预测差值和运动矢量一起传送。在接收端根据运动矢量从I帧中找出P帧“某点”的预测值并与差值相加以得到P帧“某点”样值,从而可得到完整的P帧。

P帧特点:

* P帧是I帧后面相隔1~2帧的编码帧
* P帧采用运动补偿的方法传送它与前面的I或P帧的差值及运动矢量(预测误差)
* 解码时必须将I帧中的预测值与预测误差求和后才能重构完整的P帧图像
* P帧属于前向预测的帧间编码。它只参考前面最靠近它的I帧或P帧
* P帧可以是其后面P帧的参考帧,也可以是其前后的B帧的参考帧
* 由于P帧是参考帧,它可能造成解码错误的扩散
* 由于是差值传送,P帧的压缩比较高

#### 2.4.3 B帧

为了更好地理解P帧的概念，我依然罗列了两种解释：

* 双向预测内插编码帧。B帧是双向差别帧，也就是B帧记录的是本帧与前后帧的差别（具体比较复杂，有4种情况，但我这样说简单些），换言之，要解码B帧，不仅要取得之前的缓存画面，还要解码之后的画面，通过前后画面的与本帧数据的叠加取得最终的画面。B帧压缩率高，但是解码时CPU会比较累。

* 双向预测内插编码帧 又称bi-directional interpolated prediction frame，既考虑与源图像序列前面已编码帧，也顾及源图像序列后面已编码帧之间的时间冗余信息来压缩传输数据量的编码图像，也叫双向预测帧。

B帧的预测与重构：B帧以前面的I或P帧和后面的P帧为参考帧,“找出”B帧“某点”的预测值和两个运动矢量,并取预测差值和运动矢量传送。接收端根据运动矢量在两个参考帧中“找出(算出)”预测值并与差值求和,得到B帧“某点”样值,从而可得到完整的B帧。

B帧的特点：

* B帧是由前面的I或P帧和后面的P帧来进行预测的
* B帧传送的是它与前面的I或P帧和后面的P帧之间的预测误差及运动矢量
* B帧是双向预测编码帧
* B帧压缩比最高,因为它只反映丙参考帧间运动主体的变化情况,预测比较准确
* B帧不是参考帧,不会造成解码错误的扩散

&emsp;&emsp;I、B、P各帧是根据压缩算法的需要，是人为定义的,它们都是实实在在的物理帧。一般来说，I帧的压缩率是7（跟JPG差不多），P帧是20，B帧可以达到50。可见使用B帧能节省大量空间，节省出来的空间可以用来保存多一些I帧，这样在相同码率下，可以提供更好的画质。

#### 2.4.4 三种帧的不同

* I frame:自身可以通过视频解压算法解压成一张单独的完整的图片。
* P frame：需要参考其前面的一个I frame 或者B frame来生成一张完整的图片。
* B frame:则要参考其前一个I或者P帧及其后面的一个P帧来生成一张完整的图片。
* 两个I frame之间形成一个GOP，在x264中同时可以通过参数来设定bf的大小，即：I 和p或者两个P之间B的数量。
* 通过上述基本可以说明如果有B frame 存在的情况下一个GOP的最后一个frame一定是P。

## 3 实现
&emsp;&emsp;文章开头提到，视频采样策略的主要问题是解码，大量的解码直接导致了算法性能的下降，因此，能否在压缩上找线索，寻找采样策略？能否不解码找到镜头分割点，进而找到关键帧？有了上面的介绍，当某帧图像与之前的图像变化很大时，将会生成I帧，后面帧将参考I帧生成，是I帧的相似帧，I帧是一个序列中的关键帧。因此，初步的想法是将I帧所在位置作为镜头分割点，该I帧可以作为关键帧代表该镜头。该方法的专业说法叫做：基于压缩域提取关键帧。

&emsp;&emsp;那么，问题最终缩小为：寻找视频中的I帧。下面将介绍如何利用FFMPEG定位视频中的I帧，以及如何不对视频解码定位I帧。

#### 3.1 FFMPEG命令输出I帧

&emsp;&emsp;FFMPEG命令过滤I帧命令：参考[Escaping characters](https://trac.ffmpeg.org/wiki/FilteringGuide#Escapingcharacters)

{% highlight shell %}
// FFMpeg 3.3.3 测试通过
ffmpeg -i input -vf select='eq(pict_type\,I)' -vsync vfr output_%04d.png        # to select only I frames
{% endhighlight %}

&emsp;&emsp;此命令确实能够找出I帧，与原视频比对后，基本没有问题，验证了找I帧的可行性。但这种方法找I帧速度较慢，CPU占用较高，i7-4770占用50%以上，24min的1080p，25fps的Mp4视频，耗时30s左右，故猜测这个命令实际上对每一帧都解码了，因此需要寻找一种方法：能否在不解码的条件下，判断帧类型。

&emsp;&emsp;**猜想：**在视频文件中，每一帧应该都有头信息，这些都信息不应该是解码后才得到的，而是直接写在了文件中，因此没准可以直接读取头信息，判断其帧类型。

&emsp;&emsp;看了[FFMPEG Tips (3) 如何读取每一帧的信息](http://ticktick.blog.51cto.com/823160/1872008) 和 [ffmpeg如何提取视频的关键帧？](http://www.dewen.net.cn/q/725/ffmpeg%E5%A6%82%E4%BD%95%E6%8F%90%E5%8F%96%E8%A7%86%E9%A2%91%E7%9A%84%E5%85%B3%E9%94%AE%E5%B8%A7%EF%BC%9F) 两篇文章后，基本验证可行性。

#### 3.2 非解码方法定位I帧

&emsp;&emsp;在上面两篇博客中，可以发现，在解码之前的步骤int ret = av_read_frame(fmt_ctx, &packet)中，通过判断包的flag就可以判断帧是否是关键帧了([av_read_frame读出的视频流数据在AVPacket中的存储](http://blog.csdn.net/dancing_night/article/details/45742905))，此方法可以省去解码过程。参考[filtering_video.c](https://ffmpeg.org/doxygen/trunk/filtering_video_8c-example.html)写出如下代码：

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

&emsp;&emsp;经粗略实验，24min的1080p，25fps的Mp4视频，此方法可以秒出结果。

## 4 讨论
**进一步加速**  
&emsp;&emsp;上面在解封装后，直接根据packet的flag属性判断是否是关键帧，能够避免解码操作，速度大幅提升。需要注意的是，packet还有一个属性data，存放着未解码的视频内容，因此，可以取消复制这些数据，减少IO，进一步加快速度，但对速度的提升效果如何，需要进一步尝试。粗略定了一下位置，基本可以从此入手优化该点：[memcpy(dst_data, src_sd->data, src_sd->size);](https://github.com/FFmpeg/FFmpeg/blob/949debd1d1df3a96315b3a3083831162845c1188/libavformat/utils.c#L1674)

**关键帧的选取**  
&emsp;&emsp;前面说到，将I帧作为关键帧。但是由于I帧有是一个序列中的第一帧，且当该序列是渐变效果，如从物体轮廓图像慢慢出现清晰的物体，这时候I帧代表性不强，越往后，帧代表性越强。这种情况下，选取后面的P帧或者B帧效果看起来会更好。但是会带来两个问题：

* 选择第几个P帧或B帧好？
* 无论选择P帧还是B帧，都需要从最近的I帧解码到该P帧或B帧，带来耗时的解码操作。

**重复的I帧**  
&emsp;&emsp;几次实验后，发现.mp4视频I帧提取效果较好，连续的I帧内容相似的数量较少。但是.avi视频则效果较差，连续的I帧内容相似的数量较多。比如，实验中用了一个69000多帧.mp4格式的视频，I帧数量为660左右，将其转码为.avi后，I帧数量为6900左右。查看实际图像后发现，.avi中有大量的内容相似的I帧，当然，这些I帧时间接近。
&emsp;&emsp;在一些应用上，如视频拷贝检测或者视频检索，关键帧的数量直接决定了检测和检索速度。因此，在提取出I帧后，在所有I帧上进一步减少相似帧，能够进一步减少关键帧的数量。

## 5 结束语
&emsp;&emsp;视频采样可以作为视频拷贝检测、视频检索等应用中对视频降维的有效手段。本文从视频采样实际需求出发，对视频的编解码结束做了简要介绍，介绍了I帧、P帧和B帧，提出将I帧的位置作为镜头分割点，I帧作为镜头关键帧；然后，给出了利用FFMPEG在解码和非解码的情况下提取I帧的方法；最后对该方法做了进一步的讨论。欢迎大家在页面下方进行讨论。

## 参考引用
**视频压缩相关**  
* [Video compression picture types](https://en.wikipedia.org/wiki/Video_compression_picture_types)  #
* [H.264码流以及H.264编解码的基本概念](https://maxwellqi.github.io/ios-h264-summ/)  #
* [视频格式那么多，MP4/RMVB/MKV/AVI 等，这些视频格式与编码压缩标准 mpeg4，H.264.H.265 等有什么关系？](https://www.zhihu.com/question/20997688) 

**FFMPEG解码相关**  
* [FFMPEG Tips (3) 如何读取每一帧的信息](http://ticktick.blog.51cto.com/823160/1872008)  
* [ffmpeg解码流程及解码跟踪和关键问题解析](http://blog.csdn.net/heng615975867/article/details/21602745)  
* [ffmpeg如何提取视频的关键帧？](http://www.dewen.net.cn/q/725/ffmpeg%E5%A6%82%E4%BD%95%E6%8F%90%E5%8F%96%E8%A7%86%E9%A2%91%E7%9A%84%E5%85%B3%E9%94%AE%E5%B8%A7%EF%BC%9F)  
* [FFmpeg 入门(1)：截取视频帧](http://www.samirchen.com/ffmpeg-tutorial-1/)  
* [FFmpeg Here is a list of all examples](https://ffmpeg.org/doxygen/trunk/examples.html) 
* [filtering_video.c](https://ffmpeg.org/doxygen/trunk/filtering_video_8c-example.html)  
* [FFmpeg源代码结构图 - 解码](http://blog.csdn.net/leixiaohua1020/article/details/44220151)  
* [ffmpeg 源代码简单分析 ： av_read_frame()](http://blog.csdn.net/leixiaohua1020/article/details/12678577)  
* [av_read_frame读出的视频流数据在AVPacket中的存储](http://blog.csdn.net/dancing_night/article/details/45742905)  


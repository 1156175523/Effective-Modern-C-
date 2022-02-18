## **视频术语**

![image-20220218113846996](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20220218113846996.png)

**[视频]**

**网络抽象层面 NAL**: 格式化数据并提供头信息
	--平时的每一帧数据就是一个NAL单元(SPS与PPS除外)。在实际的H264数据帧中，往往帧前面带有 00 00 00 01 或 00 00 01 分隔符，一般来说编码器编出的首帧数据为 PPS 与 SPS，接着为 I 帧
**视频编码层面 VCL**: 视频数据的内容

**GOP**(一组数据多帧 I、B和P): 可以独立播放的一组帧数据 

**SPS**:序列参数
**PPS**:图像参数
**像素格式**: BGRA、RGBA、ARGB32、ARGB32、RGB32、YUV420

**RGB** 转换 YUV: 

​					R = Y + 1.4075 * (V - 128);
​					G = Y - 0.3455 * (U - 128) - 0.7169*(V - 128)
​					B = Y + 1.779 * (U - 128)

![image-20220218115322449](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20220218115322449.png)

**YUV**:YUV4:4:4、YUV4:2:2、YUV4:2:0
"Y":表示亮度，也就是灰度值
"U"和"V" 表示色度

![image-20220218115536658](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20220218115536658.png)

**[音频]**

**AAC**：有损压缩

**APE**：无损压缩格式类似 zip

**FLAC**：无损压缩格式类似 zip

**PCM**：原始音频

**采样率 sample_rate**:
**通道channels（左右声道）**:
**样本大小（格式）sample_size**: 
			-AV_SAMPLE_FMT_S16
			-AV_SAMPLE_FMT_FLTP
**样本类型 planar**:

**PTS**：显示时间    **DTS**：解码时间    ===》实现同步策略

## **ffmpeg SDK软硬编解码基础**

**解封装**：

* ~~av_register_all()~~		// 解封装、加封装 目前已经摒弃不用

  * avformat_network_init()		// rtsp流、http视频流
  * avformat_open_input(...)	   // 解析格式
      * 确保 av_register_all()、avformat_network_init() 已调用
      * *AVFormatContext** ps    //ps指向的对象值可以为nullptr 需要使用close关闭，否则内存需要自己维护，          然调用close接口会导致异常*
      * *const char \*url              //本地文件、http链接、rtsp链接*
      * *AVInputFormat \*fmt  //一般设置为nullptr-自动探测*
      * *AVDictionary **options  //字典数据一般设置为nullptr*
  * avformat_find_stream_info(...)   // 探索视频格式
  * av_find_best_stream(...)         // 判定音视频
  * AVFormatContext、AVStream、AVPacket
  * av_read_frame(...)    // 可判断关键帧

///目前新的接口调用流程 av_register_all 被摒弃：

![img](https://img-blog.csdnimg.cn/d21131331bc44720943df2d7be1e2b18.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5byl5b2m5bey5q27,size_20,color_FFFFFF,t_70,g_se,x_16)

**AVFormatContext**结构：

* AVIOContext *pb; char filename[1024];    // 自定义读写格式
* unsigned int nb_streams;    // streams的大小
* AVStream **streams;          // 视频数据
* int64_t duration;             // 以 AV_TIME_BASE 为单位表示一秒的流时长
* int64_t bit_rate;               // 比特率
* void avformat_close_input(AVFormatContext **s);    // 释放接口

**AVStream**结构：

* ~~AVCodecContext *codec~~;     // 过时了，新版本摒弃了
* AVRational time_base;          // 实际就是分数，AVStream数据的时间单元
* int64_t duration;                    // time_base 的倍数时长 （单位，秒）
* AVRational avg_frame_rate; // 帧率（分数表示）
* AVCodecParameters *codecpar;    // 音视频参数（用于替代 codec）
  * **AVCodecParameters**结构:
    * enum AVMediaType codec_type;    // 编码类型
    * enum AVCodecID codec_id;              // 编码格式
    * uint32_t codec_tag;    // 编码的额外信息
    * int format;    // 如果是视频-表示像素格式，如果是音频-表示采样格式
    * int widget; int height; //
    * uint64_t channel_layout; int channels; int sample_rate; int frame_size;

**av_read_frame()** 接口：

* int av_read_frame(AVFormatContext *s, AVPacket *pkt);

* AVFormatContext *s 
* AVPacket* pkt                 // 当使用同一个 pkt 存储数据时需要释放之前的内容
* return  0 - 成功，< 0 - 失败或者文件结束

**AVPacket**结构：

* AVBufferRef *buf;
* int64_t pts;     // 显示时间； 以时间计数（AVStream->time_base）为单位的时间
* int64_t dts;     // 解码时间； 以时间计数（AVStream->time_base）为单位的时间
* unit8_t *data; int size;    // 视频数据
* 使用到的相关接口：
  * AVPacket *av_packet_alloc(void);   // 创建并初始化默认值
  * AVPacket *av_packet_clone(const AVPacket *src);    // 创建并引用计数，等同 alloc + ref
  * int av_packet_ref(AVPacket *dst, const AVPacket *src);    // 增加引用
  * void av_packet_unref(AVPacket *pkt);    // 减少引用
  * void av_packet_free(AVPacket **pkt);    // 删除av_packet_alloc 分配的内存，并减少引用计数
  * void av_init_packet(AVPacket *pkt);  //初始化，已废弃
  * int av_packet_from_data(AVPacket *pkt, uint8_t *data, int size);    // 从 av_malloc 创建的数据，初始化pkt

**av_seek_frame**接口：

* int av_seek_frame(AVFormatContext *s, int stream_index, int64_t timestamp, int flags);
* int  stream_index；   // 流索引；-1 defaut，sdk自动选择流，并以AV_TIME_BASE指定为time_base
  * -- 通常使用视频流索引，因为视频存在关键帧而音频不存在

* int64_t timestamp;    // 以 AVStream->time_base 为单位的时间戳;如果没有指定流，则以 AV_TIME_BASE 为单位。
* int flags;    // 选择方向和搜索模式的标志
  * #define AVSEEK_FLAG_BACKWARD 1 ///< seek backward  向后
  * #define AVSEEK_FLAG_BYTE             2 ///< seeking based on position in bytes  任意数据
  * 
  * #define AVSEEK_FLAG_ANY              4 ///< seek to any frame, even non-keyframes  任意帧
  * #define AVSEEK_FLAG_FRAME         8 ///< seeking based on frame number  关键帧
* return >=0 表示成功

---------------------------------

**avcodec_find_decoder**接口：// **解码 **

* ~~avcodec_register_all()~~    // 注册解码器，目前最新版本已弃用
* const AVCodec *avcodec_find_decoder(enum AVCodecID id);                 // 根据ID获取注册的解码器
* const AVCodec *avcodec_find_decoder_by_name(const char *name);  // 根据名称获取注册的解码器
  * eg: avcodec_find_decoder_by_name("h264_mediacodec");

**AVCodecContext**结构： // 本次解码相关信息

* 使用到的相关接口：
  * AVCodecContext *avcodec_alloc_context3(const AVCodec *codec);     // 创建并初始化默认值
  * void avcodec_free_context(AVCodecContext **avctx);                             // 释放资源
  * int avcodec_open2(AVCodecContext *avctx, const AVCodec *codec, AVDictionary **options);  // 打开上下文
    * avctx 是经过 avcodec_alloc_context3 创建的上下文
    * codec 如果 avcodec_alloc_context3 已经指定则不设置，如果设置则必须保持跟之前一致
    * options 配置参数 ------ /libavcodec/options_table.h 
  * int avcodec_parameters_to_context(AVCodecContext *codec,
                                      const AVCodecParameters *par);    // 设置解码器的属性到指定解码器上下文
* 数据成员：
  * int thread_count;     // 线程数
  * AVRational time_base;    // 表示帧时间戳的基本时间单位（以秒为单位）
    * 编码：必须由用户设置
    * 解码：不推荐使用此字段进行解码，而是使用帧率

**AVFrame**结构：

* 使用到的相关接口：
  * AVFrame *av_frame_alloc(void);                // 申请 AVFrame 结构并初始化默认值，内部内存不申请
  * void av_frame_free(AVFrame **frame);   // 释放 AVFrame 以及内部资源
  * int av_frame_ref(AVFrame *dst, const AVFrame *src);  // 添加内存引用
  * void av_frame_unref(AVFrame *frame);                           // 解除内存引用
  * AVFrame *av_frame_clone(const AVFrame *src);           // av_frame_alloc()+av_frame_ref()
* 数据成员：
  * uint8_t *data[AV_NUM_DATA_POINTERS];
  * int linesize[AV_NUM_DATA_POINTERS];
    * 视频 - 一行数据的大小
    * 音频 - 一个通道数据的大小
    * ![image-20220218111247136](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20220218111247136.png)
  * int width, height;      // 视频
  * int nb_samples;        // 音频，单通道采样数据
  * int64_t pts;          // 帧的pts
  * int64_t pkt_dts;   // 包的dts
  * int sample_rate;  // 音频-采样率
  * uint64_t channel_layout;  // 音频-通道格式
  * int channels;  // 音频-通道数
  * int format;  // 格式
    * -1 if unknown or unset
    * 视频 - enum AVPixelFormat for video frames
    * 阴平 - enum AVSampleFormat for audio

**avcodec_send_packet()** 接口：

* int avcodec_send_packet(AVCodecContext *avctx, const AVPacket *avpkt);   // avpkt 会进行内存复制或者引用计数加一
  * the decoder will not write to the packet. The decoder may create a reference to the packet data (or copy it if the packet is not reference-counted).
  * 音频可能会存在多个帧数据

**avcodec_receive_frame()** 接口：

* int avcodec_receive_frame(AVCodecContext *avctx, AVFrame *frame);
  * set to a reference-counted video or audio frame (depending on the decoder type) allocated by the decoder. Note that the function will always call av_frame_unref(frame) before doing anything else
  * 音频可能会有多帧进行读取，所以需要循环处理

**FFmpeg 进行视频像素和尺寸的转换**：（可以使用硬件处理SDL替换，可选操作）

* 使用到的接口：
  * struct SwsContext *sws_getContext(int srcW, int srcH, enum AVPixelFormat srcFormat,
                                      int dstW, int dstH, enum AVPixelFormat dstFormat,
                                      int flags, SwsFilter *srcFilter,
                                      SwsFilter *dstFilter, const double *param);
    * 创建新的sws上下文
    * @param srcW 源图像的宽度
    * @param srcH 源图像的高度
    * @param srcFormat 源图片格式
    * @param dstW 目标图像的宽度
    * @param dstH 目标图像的高度
    * @param dstFormat 目标图像格式 
    * @param flags 指定用于重新缩放的算法和选项
      * #define SWS_FAST_BILINEAR     1           // 较常用
        #define SWS_BILINEAR                2
        #define SWS_BICUBIC                  4
        #define SWS_X                               8
        #define SWS_POINT                      0x10
        #define SWS_AREA                        0x20
        #define SWS_BICUBLIN                0x40
        #define SWS_GAUSS                     0x80
        #define SWS_SINC                         0x100
        #define SWS_LANCZOS                 0x200
        #define SWS_SPLINE                      0x400
  * truct SwsContext *sws_getCachedContext(struct SwsContext *context,
                                            int srcW, int srcH, enum AVPixelFormat srcFormat,
                                            int dstW, int dstH, enum AVPixelFormat dstFormat,
                                            int flags, SwsFilter *srcFilter,
                                            SwsFilter *dstFilter, const double *param);
    * 复用缓存中已存在的上下文，如果没有则新建
    * 处理单路视频时推荐使用，多线程使用时需要注意互斥，多线程建议使用sws_getContext
  * int sws_scale(struct SwsContext *c, const uint8_t *const srcSlice[],
                  const int srcStride[], int srcSliceY, int srcSliceH,
                  uint8_t *const dst[], const int dstStride[]);
    * @param c 先前使用 sws_getContext() 创建的缩放上下文
    * @param srcSlice 包含指向源切片平面的指针的数组，eg:frame->data
    * @param srcStride 包含源图像每个平面的步幅的数组，对应 frame->linesize（宽度）
    * @param srcSliceY 要处理的切片在源图像中的位置，即切片第一行的图像中的数字（从零开始计数） 
    * @param srcSliceH 源切片的高度，即切片的行数，
    * @param dst 包含指向目标图像平面的指针的数组
    * @param dstStride 包含目标图像每个平面的步幅的数组
    * @return 输出切片的高度
  * void sws_freeContext(struct SwsContext *swsContext);
    * 释放 swsContext

**FFmpeg实现音频重采样**：

* struct SwrContext *swr_alloc(void);

  * 分配音频重采样上下文

* struct SwrContext *swr_alloc_set_opts(struct SwrContext *s,
                                        int64_t out_ch_layout, enum AVSampleFormat out_sample_fmt,

  ​									  int out_sample_rate,
  ​                                      int64_t  in_ch_layout, enum AVSampleFormat  in_sample_fmt,

  ​									  int  in_sample_rate,
  ​                                      int log_offset, void *log_ctx);

  * 设置/重置公共参数（s 可以未alloc，这时会新分配）
  * @param s 现有的 Swr 上下文（如果可用），如果不可用，则为 NULL
  * @param out_ch_layout 输出通道布局 (AV_CH_LAYOUT_*) ，声道格式，立体声道
  * @param out_sample_fmt 输出样本格式 (AV_SAMPLE_FMT_*)，采样格式FLT，eg:格式
  * @param out_sample_rate 输出采样率（频率单位为Hz）
  * @param in_ch_layout 输入通道布局 (AV_CH_LAYOUT_*)，
  * @param in_sample_fmt 输入样本格式 (AV_SAMPLE_FMT_*)
  * @param in_sample_rate 输入采样率（频率单位为Hz）
  * @param log_offset 日志记录级别偏移
  * @param log_ctx 父日志上下文，可以为 NULL
  * @see swr_init(), swr_free()
  * @return 错误时返回 NULL，否则分配上下文

* int swr_convert(struct SwrContext *s, uint8_t \*\*out, int out_count,
                                  const uint8_t **in , int in_count);

  * 转换函数
  * @param s 分配 Swr 上下文，设置参数
  * @param out 输出缓冲区，在打包音频的情况下只需要设置第一个
  * @param out_count 每个通道的样本输出可用空间量
  * @param 在输入缓冲区中，只有第一个需要在打包音频的情况下设置 
  * @param in_count 一个通道中可用的输入样本数 

* int swr_init(struct SwrContext *s);

  * Initialize context after user parameters have been set. 使用 swr_alloc_set_opts 设置参数后初始化

* void swr_free(struct SwrContext **s);


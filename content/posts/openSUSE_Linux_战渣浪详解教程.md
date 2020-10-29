---
title: "openSUSE Linux 战渣浪详解教程"
date: 2013-02-24T00:00:00+08:00
draft: false
---
自从去年十一申请到哔哩哔哩账户，一直也没有加入阿婆主的行列中去，我心有愧…于是就做了一个授人以渔的从原理到实现的教程，讲述了一些阿婆主喜闻乐见的基础知识。

#### “战渣浪” 原理

* Av Fun 和哔哩哔哩的视频都是外链到渣浪、土豆、优酷这些视频站的。
* 平均码率大于 1047KB/s 的视频会被二次压缩（我没写错，我也不懂为什么会有这个奇葩的设定）
* 非 flv 封装格式的视频会被二次压缩
* 视频非 AVC 编码、音频非 AAC 编码会被二次压缩
* 二次压缩首先会掉帧，其次会过审。你投原创自然没问题也不需要战，我只看影视区的。

来战！

#### 查看平均码率

FFMpeg 查看码率的方法是 ffmpeg -i 视频文件

    Input #0, matroska,webm, from '谎言满屋.House.of.Lies.S01E01.Chi_Eng.AC3.1024X576.x264-五零字幕组.mkv':
    Metadata:
    creation_time   : 2012-06-06 12:52:18
    Duration: 00:33:28.38, start: 0.000000, bitrate: 1585 kb/s
    Stream #0:0: Video: h264 (Main), yuv420p, 1024x576 [SAR 1:1 DAR 16:9], 23.98 fps, 23.98 tbr, 1k tbn, 47.95 tbc (default)
    Stream #0:1: Audio: ac3, 48000 Hz, 5.1(side), s16, 384 kb/s (default)

MPlayer 查看码率的方法是 mplayer -vo null -ao null -identify -frames 0。其中最重要的是 -identify 选项，意思是去识别视频信息。-frames 0 表示不播放，-vo null -ao null 表示使用 null 驱动（也就是不使用视音频驱动，这样你就能通过 ssh 在服务器上识别视频信息了）。

    Clip info:
    creation_time: 2012-06-06 12:52:18
    ID_CLIP_INFO_NAME0=creation_time
    ID_CLIP_INFO_VALUE0=2012-06-06 12:52:18
    ID_CLIP_INFO_N=1
    Load subtitles in ./
    ID_FILENAME=谎言满屋.House.of.Lies.S01E01.Chi_Eng.AC3.1024X576.x264-五零字幕组.mkv
    ID_DEMUXER=lavfpref
    ID_VIDEO_FORMAT=H264
    ID_VIDEO_BITRATE=0
    ID_VIDEO_WIDTH=1024
    ID_VIDEO_HEIGHT=576
    ID_VIDEO_FPS=23.976
    ID_VIDEO_ASPECT=1.7778
    ID_AUDIO_FORMAT=8192
    ID_AUDIO_BITRATE=384000
    ID_AUDIO_RATE=48000
    ID_AUDIO_NCH=6
    ID_START_TIME=0.00
    ID_LENGTH=2008.38
    ID_SEEKABLE=1
    ID_CHAPTERS=0

这是最最基础的了。但是识别能力不强，比如上面的那个 H264 的 MKV 就没有识别出视频码率。

下面祭出更为强大的法宝：Mediainfo。听名字就知道它是专门识别多媒体信息的。它不但有图形界面，还整合到 KDE 的右键菜单中去了。

打开 Mediainfo，打开文件后，使用 HTML/Text 模式查看（Easy 模式不显示 overall bitrate 平均码率)。

#### 分离视频和音频

为什么要分离视频和音频？因为其实它们是分开编码的，比如 H264/MPEG-4 这些是视频编码，而 AAC/AC3 这些是音频编码，编码之后再用封装容器装起来，比如 MKV/MP4/FLV 都是封装格式。

而各个编码器内部还有不同的实现/标准，有些好一些有些差一些，比如 FFMpeg 中的 FAAC 音频编码器就非常基础了，不支持 HE-AAC 标准。当然还有更强的标准比如 Apple AAC 和 Fraunhofer AAC，那就需要商业软件比如 Apple Quicktime 来支持了。Linux 下能用上的最强 AAC 编码就是 Nero 公司的 Nero AAC 了。下载在[此](http://www.nero.com/enu/company/about-nero/nero-aac-codec.php)。解压就是三个可执行文件。当然爱用 RPM 的来[这里](https://www.box.com/s/5auhujfoecrb1u1mtjji)下，省得你注册了。关于各种 AAC 的比较见 Doom 9 论坛的[帖子](http://forum.doom9.org/archive/index.php/t-164813.html)。

由于我们要用 Nero AAC 来处理音频，而现实中我们下载的视频也经常是要么都不合格，要么视频不合格，要么音频不合格，总之总是要处理，我们就把这个分离作为标准步骤了。狗耳也可以使用 FAAC，那样不用分离要简单很多。

现在除了蓝光，1080p 和 720p 的视频基本都是用 MKV 封装的。解封装要用到 mkvtoolnix，重新封装为 MP4 要用到 gpac。当然你也可以使用 ffmpeg 单纯复制音频或单纯复制视频的方法做，那太低端太“俗手”了，所以我们只传授“正着”（都是围棋术语）。

先查看 MKV 封装的轨道：

    mkvmerge --identify 谎言满屋.House.of.Lies.S01E01.Chi_Eng.AC3.1024X576.x264-五零字幕组.mkv 
    轨道 ID 0: video (V_MPEG4/ISO/AVC)
    轨道 ID 1: audio (A_AC3)

有些可能有轨道 ID 2 也就是封装的字幕。

下面把两条轨道分别提取为中间文件：

    mkvextract tracks 谎言满屋.House.of.Lies.S01E01.Chi_Eng.AC3.1024X576.x264-五零字幕组.mkv 0:clip.h264 1:audio.ac3

clip.h264 和 audio.ac3 是中间文件名称，你也可以叫别的。另外这样的速度可比 FFMpeg 复制轨道要快多了。

    正在提取轨道 0 （CodecID 为 'V_MPEG4/ISO/AVC'）到文件 'clip.h264'。容器格式: AVC/h.264 elementary stream
    正在提取轨道 1 （CodecID 为 'A_AC3'）到文件 'audio.ac3'。容器格式: Dolby Digital (AC3)
    进度: 100%

#### 处理音频

先转换成 WAV （NeroAAC 只支持 WAV 输入 =。=）。

    ffmpeg -i audio.ac3 audio.wav

然后转换 WAV 为 HE-AAC

    neroAacEnc -br 64000 -2pass -if ./audio.wav -of ./audio.aac

-if 输入文件，-of 输出文件，-br 比特率，-2pass 跑两遍，第一遍大局观，第二遍压制。

码率小于 48 KBit/s 使用 HE-AAC v2; 介于 48～96 KBit/s 使用 HE-AAC，大于 96 使用 LC-AAC（会剔除高频，当然 96～256 都是 MP3 的优势范围，可惜渣浪不让用）。具体技术说明请见[维基百科](http://en.wikipedia.org/wiki/High-Efficiency_Advanced_Audio_Coding)和[这个帖子](http://www.hydrogenaudio.org/forums/index.php?showtopic=59284)。我们使用 NeroAAC 的目的就是为了在低码率环境下使用 HE-AAC 尽量保持好音质，而 Linux 的 FAAC 只支持 LC-AAC。

可以使用

    neroAacEnc -help

查看可用选项。注意，有些选项是冲突的。比如 –2pass 只能应用于动态码率 -br，再比如你用 128 KBit/s 的码率是肯定用不了 -he 的。

另外要保持和视频文件的和谐。如果你对一个 1080p 的画质使用 48 KBit/s 的渣音质，很容易造成音画不同步。

处理好用 SMPlayer 试听一下，发现音质没什么损失，但是文件大小降低了 6 倍。

#### 处理视频

根据 Doom9 的[这个帖子](http://forum.doom9.org/archive/index.php/t-108401.html)，目前 x264 开源项目处理的质量比商业软件还要高，所以没说的就它了。

***关于 x264 的版本***：

x264 有三个版本，x264 vanilla, x264 kMod, x264 tMod。具体的分辨见 Doom9 的[这个帖子](http://forum.doom9.org/showthread.php?t=164506)。其中用的比较多的是 x264_tMod（开发者似乎是个中国人？他的 wordpress 博客是中文名字）。一般发行版都没有带，openSUSE 比较强啦，RPM 在[此](https://www.box.com/s/oywdggnrbserz1epqchh)（现在 Packman 服务器怠机中，好了会推送过去）。编译也不是很难，用以下参数

    configure --prefix=/usr \
    --enable-pic \
    --enable-visualize \
    --enable-shared \
    --bit-depth=10 \
    --enable-nonfree \
    --enable-lto

就都编上了。也可以用我的 [spec](https://github.com/marguerite/openSUSE-SPEC-Non-Free/blob/master/x264_tMod.spec)，git clone 了源码后，做成我要求的压缩包，然后放到 /usr/src/packages/SOURCES 和 /usr/src/packages/SPECS 后运行：

    rpmbuild -ba x264_tMod.spec

然后到 /usr/src/packages/RPMS/x86_64 目录下去找 RPM 即可。

***关于 x264 的参数***

压片跟围棋一样有两种流派，技术流和天衍流（但围棋的两个流派虽然相互看不懂但是都尊重对方）。技术流就是了解每一个参数的意义，在自己给自己那无坚不摧的决定物质的意识施加的蛋疼限制下，试图寻找一个完美通吃的 setting。考虑到高压的时间，这种流派是在用生命在压片; 天衍流就是了解基本的方法和关键的几个参数，在特定外部限制下（比如渣浪的），凭第六感和大局观压片，用心灵压片，外界成不成那就要讲该片和你有没有缘了。

考虑到压片的黑魔法属性，一开始就投入技术参数的樊笼中只能作茧自缚。黑魔法属性是说，比如 x264 自己就有 100+ 个参数，每个参数又有 10 个档位，计算机可以算出排列组合但它没有视觉无法知道压出的片是不是可口; 而可悲的人类，哈哈，错一个参数都可能导致别人压出的是高清而你压出的是睾清，给你一个礼拜你都找不出来错哪儿了。另外看不懂宅们的思路，去 B 站投稿么，要的就是一个开心，找一个乐子（要不我煮碗面给你吃？），等你把所有参数都大彻大悟融汇贯通之后，人家早就发片过百部啦。

如果你依然毅然决然地决定踏上技术参数的不归路，有以下几篇堪称“人生哲学”的神文在此：

* [x264使用介绍](http://www.nmm-hd.org/doc/X264%E4%BD%BF%E7%94%A8%E4%BB%8B%E7%BB%8D)
* [x264设定](http://www.nmm-hd.org/doc/X264%E8%A8%AD%E5%AE%9A)
* [尽量不浪费压制时间的简单视频高压要点](http://blog.sina.com.cn/s/blog_3df9d2330100zcy4.html)

另外：

    x264_tMod --fullhelp

说得不比它们差，只是英文而已。就不再班门弄斧了。

直接上我的参数吧：

    x264_tMod --pass 1 --bitrate 950 --preset slow --tune film --vbv-bufsize auto_high --vf resize:512,288,,,,spline/pad:0,48,0,48 -o vieo.h264 clip.h264

之后把 –pass 1 改成 2 压第二遍。–bitrate 950 是比特率; –preset 和 –tune 是预设方案，这里我选的是慢速压电影; –vbv-bufsize 预缓冲秒数用的也是电影的预设 auto_high。

需要说明一下 video filter –vf。resize 是要把你的视频分辨率调整为多少，它和下面的 pad 相加才会得到最终的分辨率，所以不要在这里想当然地输入 512×384（哔哩哔哩的播放器尺寸），这样会生成一个 512x(384+48×2) 的分辨率。

***注意*** x264 vanilla 是没有这个 pad 和之后的 subtitle filter 的。这就是为什么我们要用 x264_tMod 的原因。还有一个原因是搬运工在 Windows 上常用的 MeGUI 的后端就是这个版本，转换比较容易一些，参数也好扒。不然的话也可以使用 FFMpeg 的 pad filter，但据 DOOM9 论坛[讲](http://forum.doom9.org/archive/index.php/t-143215.html)，不用 x264 加黑边会造成码率偏高。

另外提醒一下，去找别人分享的参数时一定要看清楚对方压的是什么片！现在 B 站贴吧里比较活跃的是新番，我曾套用新番的参数压出一个 56 MB 的电影，RMVB 睾画质…基本所有的教程里截图的参数都不能用来压电影。

***字幕问题***

（写教程时我压这片是渲染好字幕的，所以只能凭借去年 10 月研究的感觉来写）

因为比较重要所以要单独来讲。定理一，看我口型：

> 表把字幕留到视频变成 FLV 了才想着去加！

这是新手最容易犯的错误（因为觉得字幕不重要），也是尸山血海的错误。因为 Linux 下给 FLV 加字幕，在不把你之前的工作推倒把 FLV 转换掉的情况下，只有一种方法，使用 audemux 硬渲染 SSA 字幕（也很难在 Google 搜到）。由于 SSA 是一种非常罕见的字幕格式，基本宣告着您掉进参数/命令行的大坑里爬不出来了！

定理二:

> 表以为可以把字幕以轨道 2 的形式封装到 MKV/MP4！

如果你搜索 Linux 嵌入字幕，那你百分百会使用这种方法！这最后会让你整个人都囧掉，因为 MKV/MP4 再转 FLV 的时候自动把字幕轨道丢掉了。因为 FLV 这个封装格式就！没！有！字！幕！轨！道！

背景知识：嵌入是在封装格式支持字幕轨道的情况下，把 aas/srt 外挂到该轨道中，又叫软字幕，这种字幕可以轻松的提取出来; 渲染是把 aas/srt 转换为 vodsub 图片格式，然后使用过滤器写入到视频轨道的原视频中去，或外挂 vodsub（因为 vodsub 是图片不是文字嘛，所以就可以像娜姐那样加很多特效）。又叫硬字幕。根据不同的渲染方式，有时候根本提取不出来。

大体比较流行的字幕格式有两种，ass 和 srt。其中比较好的是 ass，转换方法也简单：

    ffmpeg -i *.srt *.ass

因为 FLV 就没有字幕轨道，所以我们不能使用嵌入的方法，而应该使用渲染的方法或者去 A/B 站使用外挂字幕。你搜到的 FFMpeg/MPlayer 方法基本都是嵌入，就是挂载为一条单独的轨道，这样子是不可以的。要用 x264 的 vfilter 来渲染。

    --vf subtitle: --sub "sub.ass"

和前面的写在一起就是：

    --vf resize:512,288,,,,spline/pad:0,48,0,48/subtitle: --sub "sub.ass"

PS: Windows 下一般都用 Avisynth 脚本来做这种字幕，Linux 下的移植 Avxsynth 我没研究，因为太烦了，FFMpeg 和 Mplayer 方法见下面：

* MPlayer 见[此文](http://ubuntuforums.org/showthread.php?t=1155877)，看到那些参数了吧，哈哈
* FFMpeg 见[此文](http://ffmpeg.org/trac/ffmpeg/wiki/How%20to%20burn%20subtitles%20into%20the%20video)，相比 MPlayer 简单一些，用滤镜的。

#### 合并视频和音频

现在我们已经分别处理好我们的音频和视频了。下面用 FFMpeg 将它们拼接为 FLV。

    ffmpeg -i video.h264 -i audio.aac -vcodec copy -acodec copy -map 0:v -map 1:a ./谎言满屋.House.of.Lies.S01E01.Chi_Eng.H264.HE-AC.marguerite.flv

-map 的意思是把 video 轨道定为 0, audio 轨道定为 1。可用 mediainfo 查看码率，1006 KBit/s！这是我第一次战渣浪呦！

下面用 MPlayer/VLC 播放器预览一下，如果没有出现音画不同步就可以拿去传渣浪了。如果出现了音画不同步，那分情况：

* 如果全局的音画不同步（多见于高画质配渣音质），可以使用 Audacity 来进行提前或退后。
* 如果是随机的音画不同步，多数是由于你压制音频的时候没有 -2pass 导致音频没有大局观。

#### 其它黑科技

* Linux 的 Avisynth 移植 [Avxsynth](https://github.com/avxsynth/avxsynth)。由于是官方移植，Windows 的脚本也通用的。等 Packman 恢复后提交过去。
* 前黑后黑。我的想法是使用 Kdenlive 做一段全黑屏视频，然后使用 MPlayer/FFMPEG/GPac 截取定长时间（算法见此)，然后使用 GPac 合并视频。
* 红色有角 N 倍速和 FLV Bug 我没找到技术规格。想来实现也不是很难。

#### 其它 Note

* 为什么不用强大的 HandBrake？因为它对 h264 支持的不好,wxWidget 也总卡死。
* 推荐一个图形界面？似乎可以使用 avidemux。


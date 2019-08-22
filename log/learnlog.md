# Windows 10 System Activation
1. 打开注册表编辑器：`win+R`，输入`Regedit`.
2. 修改`SkipRearm`的值为1：`SOFTWARE–>Microsoft–>Windows NT–>CurrentVersion–>SoftwareProtectionPlatform->SkipRearm`，重启电脑。
3. `win+X`选择管理员身份启动命令符，输入：`slmgr -rearm`,根据提示重启电脑。
4. `win+X`选择管理员身份启动命令符，输入：`slmgr /ipk DCPHK-NFMTC-H88MJ-PFHPY-QJ4BJ`,回车，弹出窗口提示：成功的安装了产品密钥。
5. 输入：`slmgr /skms xykz.f3322.org`，弹出窗口提示：密钥管理服务计算机名称成功设置为xykz.f3322.org。
6. 输入：`slmgr /ato`,按回车键后将弹出窗口提示：“成功的激活了产品”。
7. 至此，Win10正式企业版系统激活成功。
# 大端&小端数据模式
计算机的字节顺序模式分为大端数据模式和小端数据模式，它们是根据数据在内存中的存储方式来区分的。小端字节顺序的数据存储模式是按内存增大的方向存储的，即低位在前高位在后；大端字节顺序的数据存储方向恰恰是相反的，即高位在前，低位在后。
![大端&小端数据模式](image/20190817222404.png)
# PCM16LE双音道音频封装为WAVE格式音频
## 1. WAVE格式
WAVE文件是一种RIFF格式的文件。其基本块名称是“WAVE”，其中包含了两个子块“fmt”和“data”。从编程的角度简单说来就是由WAVE_HEADER、WAVE_FMT、WAVE_DATA、采样数据共4个部分组成。它的结构如下所示。
```c
 WAVE_HEADER
 WAVE_FMT
 WAVE_DATA
 PCM数据
```
其中前3部分的结构如下所示。在写入WAVE文件头的时候给其中的每个字段赋上合适的值就可以了。但是有一点需要注意：WAVE_HEADER和WAVE_DATA中包含了一个文件长度信息的dwSize字段，该字段的值必须在写入完音频采样数据之后才能获得。因此这两个结构体最后才写入WAVE文件中。
```c
typedef struct WAVE_HEADER{
		char fccID[4];
		unsigned long dwSize;
		char fccType[4];
	}WAVE_HEADER;
 
	typedef struct WAVE_FMT{
		char  fccID[4];
		unsigned long dwSize;
		unsigned short wFormatTag;
		unsigned short wChannels;
		unsigned long dwSamplesPerSec;
		unsigned long dwAvgBytesPerSec;
		unsigned short wBlockAlign;
		unsigned short uiBitsPerSample;
	}WAVE_FMT;
 
	typedef struct WAVE_DATA{
		char       fccID[4];
		unsigned long dwSize;
	}WAVE_DATA;
```
## 2. PCM16LE格式
PCM16LE双声道数据中左声道和右声道的采样值是间隔存储的。每个采样值占用2Byte空间。采样频率一律是44100Hz，采样格式一律为16LE。“16”代表采样位数是16bit。由于1Byte=8bit，所以一个声道的一个采样值占用2Byte。“LE”代表Little Endian，代表2 Byte采样值的存储方式为高位存在高地址中。
## 3. 视频播放器解码流程
视频播放器播放一个互联网上的视频文件，需要经过以下几个步骤：解协议，解封装，解码视音频，视音频同步。如果播放本地文件则不需要解协议，为以下几个步骤：解封装，解码视音频，视音频同步。他们的过程如图所示。
![视频播放器解码流程](image/20190818021421_1.png)
# H.264视频码流解析
## 1. 原理
H.264原始码流（裸流）是由一个接一个NALU组成的，它的功能分为两层，VCL（视频编码层）和NAL（网络提取层）：
`VCL(Video Coding Layer)+NAL(Network Abstraction Layer)`

- VCL：包括核心压缩引擎和块，宏块和片的语法级别定义，设计目标是尽可能地独立于网络进行高效的编码；
- NAL：负责将VCL产生的比特字符串适配到各种各样的网络和多元环境中，覆盖了所有片级以上的语法级别。

在VCL进行数据传输或存储之前，这些编码的VCL数据，被映射或封装进NAL单元（NALU）。

```一个NALU=一组对应于视频编码的NALU头部信息+一个原始字节序列负荷（RBSP,Raw Byte Sequence Payload）```

NALU头部`+`RBSP就相当于一个NALU（NAL Unit），每个单元都按独立的NALU传送。H.264的结构全部都是以NALU为主，理解了NALU，就理解了H.264的结构。

一个原始的NALU单元一般由`StartCode+NALU Header+NALU Payload`三部分组成，其中`StartCode`用于标识这是一个NALU单元的开始，必须是`00 00 00 01`(Slice&NALU的开始)或`00 00 01`。
![](image/20190818214218.png)
### (1) NAL Header
包括三部分，`forbidden_bit(1bit),nal_reference(2bits)（优先级）,nal_unit_type(5bits)(类型)`。
![nal_header](image/08.png)
### (2) RBSP
![rbsp序列举例](image/09.png)
![rbsp描述](image/10.png)
#### SODB与RBSP
SODB数据比特串->编码后的原始数据。

RBSP原始字节序列载荷->在原始编码数据的后面添加了结尾比特。一个bit“1”若干比特“0”，以便字节对齐。
![rbsp&sodb](image/12.png)
## 2. 从NALU出发了解H.264的专业术语
![H.264码流分层结构](image/06.png)
```c
1帧 = n个片
1片 = n个宏块
1宏块 = 16x16yuv数据
```
### （1）Slice（片）
NALU的主体中包含了Slice（片）。
```c
一个片 = slice header + slice data
```
片是H.264提出的新概念，一个图片有一个或者多个片，而片由nalu装载并进行网络传输。但是nalu不一定是切片，这是充分不必要条件，因为nalu还有可能装载着其他用作描述视频的信息。
#### 为什么要设置片？
设置片的目的是为了限制误码的扩散和传输，应使编码片相互间是独立的。某片的预测不能以其他片中的宏块为参考图像，这样某一片中的预测误差才不会传播到其他片中。

可以看到上图中，每个图像中，若干宏块(Macroblock)被排列成片。一个视频图像可编程一个或更多个片，每片包含整数个宏块 (MB),每片至少包含一个宏块。

#### 片有五种类型：
![片分类](image/20190818221012.png)
### (2) Macroblock(宏块)
宏块是视频信息的主要承载者，一个编码图像通常划分为多个宏块。视频解码最主要的工作室提供高效的方式从码流中获取宏块中的像素阵列。
![宏块](image/20190818221534.png)
H.264中，句法元素共被组织成序列、图像、片、宏块、子宏块五个层次。

句法元素的分层结构有助于更有效地节省码流。例如，再一个图像中，经常会在各个片之间有相同的数据，如果每个片都同时携带这些数据，势必会造成码流的浪费。更为有效的做法是将该图像的公共信息抽取出来，形成图像一级的句法元素，而在片级只携带该片自身独有的句法元素。
![句法元素的分层结构](image/20190818221921.png)
![宏块句法](image/11.png)
![宏块分类意义](image/20190818222215.png)
###（3）I、P、B帧与pts/dts
![](image/20190818222449.png)
### (4) GOP
GOP是画面组，一个GOP是一组连续的画面。GOP一般有两个数字，如M=3，N=12.M制定I帧与P帧之间的距离，N指定两个I帧之间的距离。由此可知GOP的结构为：
```
I BBP BBP BBP BB I
```
### (5) IDR
一个序列的第一个图像是IDR（立即刷新图像），IDR图像都是I帧图像。

I和IDR帧都使用帧内预测。I帧不用参考任何帧，但是之后的P帧和B帧是有可能参考这个I帧之前的帧的。
#### 作用
H.264 引入 IDR 图像是为了解码的重同步，当解码器解码到 IDR 图像时，立即将参考帧队列清空，将已解码的数据全部输出或抛弃，重新查找参数集，开始一个新的序列。这样，如果前一个序列出现重大错误，在这里可以获得重新同步的机会。IDR图像之后的图像永远不会使用IDR之前的图像的数据来解码。
# FFmpeg+SDL视频播放器的制作

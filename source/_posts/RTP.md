---
title: RTP
date: 2019-07-29 17:45:43
categories: 音视频
tags: 流媒体协议
---
## RTP 协议介绍

实时传输协议 RTP（Real-time Transport Protocol）是一个网络传输协议，它是由 IETF 的多媒体传输工作小组 1996 年在 RFC 1889 中公布的，后在 RFC3550 中进行更新。

它作为因特网标准在 [ RFC 3550 ](https://www.rfc-editor.org/rfc/rfc3550.txt) 有详细说明。
RTP 协议详细说明了在互联网上传递音频和视频的标准数据包格式。它一开始被设计为一个多播协议，但后来被用在很多单播应用中。RTP 协议常用于流媒体系统（配合 RTSP 协议），视频会议和一键通（Push toTalk）系统（配合 H.323 或 SIP），使它成为 IP 电话产业的技术基础。RTP 协议和 RTP 控制协议 RTCP 一起使用，而且它是建立在用户数据报协议上的（UDP）。

### RTP 和 RTCP

- 数据传输协议 RTP 用于实时传输数据。该协议提供的信息包括：时间戳（用于同步）、序列号（用于丢包和重排序检测）、以及负载格式（用于说明数据的编码格式）。

- 控制协议 RTCP，用于 QoS 反馈和同步媒体流。相对于 RTP 来说，RTCP 所占的带宽非常小，通常只有 5%。

### RTP 的优势

提到流媒体传输、视频监控、视频会议、语音电话（VOIP），都离不开 RTP 协议的应用，但是为什么要使用 RTP 来进行流媒体的传输呢？为什么一定要用 RTP？

像 TCP 这样的可靠传输协议，通过超时和重传机制来保证传输数据流中的每一个 bit 的正确性，但这样会使得无论从协议的实现还是传输的过程都变得非常的复杂。而且，当传输过程中有数据丢失的时候，由于对数据丢失的检测（超时检测）和重传，会数据流的传输被迫暂停和延时。

RTP 协议是一种基于 UDP 的传输协议，RTP 本身并不能为按顺序传送数据包提供可靠的传送机制，也不提供流量控制或拥塞控制，它依靠 RTCP 提供这些服务。这样，对于那些丢失的数据包，不存在由于超时检测而带来的延时，同时，对于那些丢弃的包，也可以由上层根据其重要性来选择性的重传。

### RTP 的协议层次

**流媒体体系结构**

流媒体应用中典型的协议体系结构。

{% asset_img transport.png transport %}

从图中可以看出，RTP 被划分在传输层，它建立在 UDP 上。同 UDP 协议一样，为了实现其实时传输功能，RTP 也有固定的封装形式。RTP 用来为端到端的实时传输提供时间信息和流同步，但并不保证服务质量。服务质量由 RTCP 来提供。

## RTP 工作机制

当应用程序建立一个 RTP 会话时，应用程序将确定一对目的传输地址。目的传输地址由一个网络地址和一对端口组成，有两个端口：一个给 RTP 包，一个给 RTC P 包，使得 RTP/RTCP 数据能够正确发送。RTP 数据发向偶数的 UDP 端口，而对应的控制信号 RTCP 数据发向相邻的奇数 UDP 端口（偶数的 UDP 端口 + 1），这样就构成一个 UDP 端口对。 RTP 的发送过程如下，接收过程则相反。
1. RTP 协议从上层接收流媒体信息码流（如 H.263），封装成 RTP 数据包；RTCP 从上层接收控制信息，封装成 RTCP 控制包。
2. RTP 将 RTP 数据包发往 UDP 端口对中偶数端口；RTCP 将 RTCP 控制包发往 UDP 端口对中的奇数端口。

RTP 分组只包含 RTP 数据，而控制是由 RTCP 协议提供。RTP 在 1025 到 65535 之间选择一个未使用的偶数 UDP 端口号，而在同一次会话中的 RTCP 则使用下一个奇数 UDP 端口号。端口号 5004 和 5005 分别用作 RTP 和 RTCP 的默认端口号。RTP 分组的首部格式如图 2 所示，其中前 12 个字节是必须的。

{% asset_img 3.png transport %}

### 应用层

RTP 应当是应用层的一部分。在应用的发送端，开发者必须编写用 RTP 封装分组的程序代码，然后把 RTP 分组交给 UDP。在接收端，RTP 分组通过 UDP 接口进入应用层后，还要利用开发者编写的程序代码从 RTP 分组中把应用数据块提取出来。

### RTP 报文

首先，我们看看 RTP 的包头。RTP 报文头格式（见 RFC3550 Page12）：

{% asset_img 2.png transport %}

- 版本号（V）：2 比特，用来标志使用的 RTP 版本。
- 填充位（P）：1 比特，如果该位置位，则该 RTP 包的尾部就包含附加的填充字节。
- 扩展位（X）：1 比特，如果该位置位的话，RTP 固定头部后面就跟有一个扩展头部。
- CSRC 计数器（CC）：4 比特，含有固定头部后面跟着的 CSRC 的数目。
- 标记位（M）：1 比特，该位的解释由配置文档（Profile）来承担。对于在 RTP 以最小的控制配置文件下在音频和视频会议下运行的音频流，标记位设置为 1，表示在一段静默期后发送的第一个数据包，否则设置为 0。
- 载荷类型（PayloadType）：7 比特，标识了 RTP 载荷的类型。
- 序列号（SN）： 16 比特，每发送一个 RTP 数据包，序列号增加  1。接收端可以据此检测丢包和重建包序列。
- 时间戳（Timestamp）：2 比特，记录了该包中数据的第一个字节的采样时刻。
- 同步源标识符（SSRC）：32 比特，同步源就是指 RTP 包流的来源。在同一个 RTP 会话中不能有两个相同的 SSRC 值。该标识符是随机选取的 RFC1889 推荐了 MD5 随机算法。
- 贡献源列表（CSRC List）：0~15 项，每项 32 比特，用来标志对一个 RTP 混合器产生的新包有贡献的所有 RTP 包的源。由混合器将这些有贡献的 SSRC 标识符插入表中。SSRC 标识符都被列出来，以便接收端能正确指出交谈双方的身份。

#### RTP 扩展头结构

{% asset_img 1.png transport %}

若 RTP 固定头中的扩展比特位置 1（注意：如果有 CSRC 列表，则在 CSRC 列表之后），则一个长度可变的头扩展部分被加到 RTP 固定头之后。头扩展包含 16 比特的长度域，指示扩展项中 32 比特字的个数，不包括 4 个字节扩展头（因此零是有效值）。

RTP 固定头之后只允许有一个头扩展。为允许多个互操作实现独立生成不同的头扩展，或某种特定实现有多种不同的头扩展，扩展项的前 16 比特用以识别标识符或参数。这 16 比特的格式由具体实现的上层协议定义。基本的 RTP 说明并不定义任何头扩展本身。

## RTP 会话

当应用程序建立一个 RTP 会话时，应用程序将确定一对目的传输地址。目的传输地址由一个网络地址和一对端口组成，有两个端口：一个给 RTP 包，一个给 RTCP 包，使得 RTP/RTCP 数据能够正确发送。RTP 数据发向偶数的 UDP 端口，而对应的控制信号 RTCP 数据发向相邻的奇数 UDP 端口（偶数的 UDP 端口 + 1），这样就构成一个 UDP 端口对。

### RTP 发送过程

1. RTP 协议从上层接收流媒体信息码流（如 H.263），封装成 RTP 数据包；RTCP 从上层接收控制信息，封装成 RTCP 控制包。
2. RTP 数据包发往 UDP 端口对中偶数端口；RTCP 将 RTCP 控制包发往 UDP 端口对中的接收端口。

## RTP profile 机制

RTP 为具体的应用提供了非常大的灵活性，它将传输协议与具体的应用环境、具体的控制策略分开，传输协议本身只提供完成实时传输的机制，开发者可以根据不同的应用环境，自主选择合适的配置环境、以及合适的控制策略。

这里所说的控制策略指的是你可以根据自己特定的应用需求，来实现特定的一些 RTCP 控制算法，比如前面提到的丢包的检测算法、丢包的重传策略、一些视频会议应用中的控制方案等等（这些策略我可能将在后续的文章中进行描述）。

对于上面说的合适的配置环境，主要是指 RTP 的相关配置和负载格式的定义。RTP 协议为了广泛地支持各种多媒体格式（如 H.264, MPEG-4, MJPEG, MPEG），没有在协议中体现出具体的应用配置，而是通过 profile 配置文件以及负载类型格式说明文件的形式来提供。对于任何一种特定的应用，RTP 定义了一个 profile 文件以及相关的负载格式说明。

## RTCP

服务质量的监视与反馈、媒体间的同步，以及多播组中成员的标识。在 RTP 会话期间，各参与者周期性地传送 RTCP 包。RTCP 包中含有已发送的数据包的数量、丢失的数据包的数量等统计资料，因此，各参与者可以利用这些信息动态地改变传输速率，甚至改变有效载荷类型。RTP 和 RTCP 配合使用，它们能以有效的反馈和最小的开销使传输效率最佳化，因而特别适合传送网上的实时数据。

```wiki
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+-------------------------------+
|V=2|P|   IC    |       PT      |             Length            |
+---------------+---------------+-------------------------------+
|                                                               |
|                  Format-specific information                  |
|                                                               |
|                                       +-----------------------+
|                                       |    Padding if P = 1   |
+---------------------------------------+-----------------------+
```

各个字段的含义为：

- 版本号，固定为 2；
- 填充（Padding）标志位，1 表示有填充；
- 条目数量（Item Count，简称 IC），在 RTCP 包的内容是条目列表时，用来表明条目数量；其他情况可作为别的含义；
- 包类型（Packet Type，也简称 PT），rfc3550 定义了五种标准包类型，分别是 sender report (SR), receiver report (RR), source description (SDES), goodbye (BYE), application-specific message (APP)；
- 长度，包头之后的内容总长度，单位是四字节，允许为 0；

RTCP 包不会单独的被传输，它需要打包在一起形成复合包（compound packets）进行传输。每一个复合包都会被一个底层的包封装（通常是 UDP/IP 包）用来传输。如果要对复合包进行加密，那么 RTCP 的包组的前缀通常是一个 32 位的随机数。复合包的结构如下图所示：

```wiki
+---------------------------------------------------------------+
|                                                               |
|                          IP header                            |
|                                                               |
+---------------------------------------------------------------+
|                                                               |
|                          UDP header                           |
|                                                               |
+---------------------------------------------------------------+
|                  Random prefix (if encrypted)                 |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|V=2|P|   IC    |       PT      |             Length            |
+---------------+---------------+-------------------------------+
|                                                               | first
|                                                               | RTCP
|                  Format-specific information                  | packet
|                                                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|V=2|P|   IC    |       PT      |             Length            |
+---------------+---------------+-------------------------------+
|                                                               | second
|                                                               | RTCP
|                  Format-specific information                  | packet
|                                                               |
+---------------------------------------------------------------+
```

RTCP 也是用 UDP 来传送的，但 RTCP 封装的仅仅是一些控制信息，因而分组很短，所以可以将多个 RTCP 分组封装在一个 UDP 包中。RTCP 有如下五种分组类型。

| 类型 |            缩写表示            |    用途    |
| :--: | :----------------------------: | :--------: |
| 200  |       SR(Sender  Report)       | 发送端报告 |
| 201  |      RR(Receiver Report)       | 接收端报告 |
| 202  | SDES(Source Description Items) |  源点描述  |
| 203  |              BYE               |  结束传输  |
| 204  |              .APP              |  特定应用  |

上述五种分组的封装大同小异，下面只讲述 SR 类型，而其它类型请参考 RFC3550。

发送端报告分组 SR（Sender Report） 用来使发送端以多播方式向所有接收端报告发送情况。SR 分组的主要内容有：相应的 RTP 流的 SSRC，RTP 流中最新产生的 RTP 分组的时间戳和 NTP，RTP 流包含的分组数，RTP 流包含的字节数。SR 包的封装如图所示：

```wiki
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|    RC   |   PT=SR=200   |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     SSRC of packet sender                     |
+---------------------------------------------------------------+
|                        NTP timestamp                          |
|                                                               |
+---------------------------------------------------------------+
|                        RTP timestamp                          |
+---------------------------------------------------------------+
|                    Sender's packet count                      |
+---------------------------------------------------------------+
|                    Sender's octet count                       |
+---------------------------------------------------------------+
|                   Receiver report block(s)                    |
|                                                               |
```

- 版本（V）：同 RTP 包头域。
- 填充（P）：同 RTP 包头域。
- 接收报告计数器（RC）：5 比特，该 SR 包中的接收报告块的数目，可以为零。 
- 包类型（PT）：8 比特，SR 包是 200。
- 长度域（Length）：16 比特，其中存放的是该 SR 包以 32 比特为单位的总长度减一。
- 同步源（SSRC）：SR 包发送者的同步源标识符。与对应 RTP 包中的 SSRC 一样。 
- NTP Timestamp（Network time protocol）：SR 包发送时的绝对时间值。NTP 的作用是同步不同的 RTP 媒体流。
- RTP Timestamp：与 NTP 时间戳对应，与 RTP 数据包中的 RTP 时间戳具有相同的单位和随机初始值。
- Sender's packet count： 从开始发送包到产生这个 SR 包这段时间里，发送者发送的 RTP 数据包的总数。SSRC 改变时，这个域清零。
- Sender's octet count： 从开始发送包到产生这个 SR 包这段时间里，发送者发送的净荷数据的总字节数（不包括头部和填充）。发送者改变其 SSRC 时，这个域要清零。
- 同步源 n 的 SSRC 标识符：该报告块中包含的是从该源接收到的包的统计信息。
- 丢失率 (Fraction Lost)： 表明从上一个 SR 或 RR 包发出以来从同步源 n(SSRC_n) 来的 RTP 数据包的丢失率。
- 累计的包丢失数目： 从开始接收到 SSRC_n 的包到发送 SR, 从 SSRC_n 传过来的 RTP 数据包的丢失总数。
- 收到的扩展最大序列号： 从 SSRC_n 收到的 RTP 数据包中最大的序列号，
- 接收抖动 (Interarrival jitter)： RTP 数据包接受时间的统计方差估计
- 上次 SR 时间戳 (Last SR,LSR)：取最近从 SSRC_n 收到的 SR 包中的 NTP 时间戳的中间 32 比特。如果目前还没收到 SR 包，则该域清零。
- 上次 SR 以来的延时 (Delay since last SR,DLSR)：上次从 SSRC_n 收到 SR 包到发送本报告的延时。

## RTP 时间戳

时间戳反映了 RTP 分组中的数据的第一个字节的采样时刻，在一次会话开始时的时间戳初值也是随机选择的。即使是没有信号发送时，时间戳的数值也要随时间不 断的增加。接收端使用时间戳可准确知道应当在什么时间还原哪一个数据块，从而消除传输中的抖动。时间戳还可用来使视频应用中声音和图像同步。

在 RTP 协议中并没有规定时间戳的粒度，这取决于有效载荷的类型，如采样频率为 90000 HZ，那么时间戳单位为 1/90000, 如果每秒发送 30 帧，那么时间戳增量就是 90000/30 = 3000。

时间戳增量是发送第二个 RTP 包相距发送第一个 RTP 包时的时间间隔，如果是视频应该是发送每帧的间隔时间。

## 代码

> RTP header

```c
/*
 * RTP header
 */
typedef struct 
{
#if 0   //BIG_ENDIA
    unsigned int version:2;   /* protocol version */
    unsigned int p:1;         /* padding flag */
    unsigned int x:1;         /* header extension flag */
    unsigned int cc:4;        /* CSRC count */
    unsigned int m:1;         /* marker bit */
    unsigned int pt:7;        /* payload type */
    unsigned int seq:16;      /* sequence number */

#else
    unsigned int cc:4;        /* CSRC count */
    unsigned int x:1;         /* header extension flag */
    unsigned int p:1;         /* padding flag */
    unsigned int version:2;   /* protocol version */
    unsigned int pt:7;        /* payload type */
    unsigned int m:1;         /* marker bit */
    unsigned int seq:16;      /* sequence number */
#endif
    u_int32 ts;               /* timestamp */
    u_int32 ssrc;             /* synchronization source */
    u_int32 csrc[1];          /* optional CSRC list */
} rtp_hdr_t;
```

> RTCP Common header

```c
/*
 * RTCP common header word
 */
typedef struct {
#if 0   //BIG_ENDIA
    unsigned int version:2;   /* protocol version */
    unsigned int p:1;         /* padding flag */
    unsigned int count:5;     /* varies by packet type */
#else
    unsigned int count:5;     /* varies by packet type */
    unsigned int p:1;         /* padding flag */
    unsigned int version:2;   /* protocol version */
#endif
    unsigned int pt:8;        /* RTCP packet type */
    unsigned short length;           /* pkt len in words, w/o this word */

} rtcp_common_t;
```

---
title: RTSP
date: 2019-08-10 15:37:46
categories: 音视频
tags: 流媒体协议
---
## RTSP 协议
> 实时流协议（Real Time Streaming Protocol，RTSP）是一种网络应用协议，专为娱乐和通信系统的使用，以控制流媒体服务器。该协议用于创建和控制终端之间的媒体会话。媒体服务器的客户端发布 VCR 命令，例如播放，录制和暂停，以便于实时控制从服务器到客户端（视频点播）或从客户端到服务器（语音录音）的媒体流。是 TCP/IP 协议体系中的一个应用层协议, 由哥伦比亚大学, 网景和 RealNetworks 公司提交的 IETF RFC 标准. 对应的 RFC 编号是 2326，可以在这里搜索 [RFC Editor](http://www.rfc-editor.org/search/rfc_search_detail.php)

该协议定义了一对多应用程序如何有效地通过 IP 网络传送多媒体数据. RTSP 在体系结构上位于 RTP 和 RTCP 之上, 它使用 TCP 或 RTP 完成数据传输. RTSP 被用于建立的控制媒体流的传输，它为多媒体服务扮演“网络远程控制”的角色。尽管有时**可以把 RTSP 控制信息和媒体数据流交织在一起传送**，但一般情况 RTSP 本身并不用于转送媒体流数据。媒体数据的传送可通过 RTP/RTCP 等协议来完成。

该协议用于 C/S 模型, 是一个基于文本的协议, 用于在客户端和服务器端建立和协商实时流会话.

### 协议体系

**一次基本的 RTSP 操作过程:**

- 首先，客户端连接到流服务器并发送一个 RTSP 描述命令（DESCRIBE）。
- 流服务器通过一个 SDP 描述来进行反馈，反馈信息包括流数量、媒体类型等信息。
- 客户端再分析该 SDP 描述，并为会话中的每一个流发送一个 RTSP 建立命令（SETUP），RTSP 建立命令告诉服务器客户端用于接收媒体数据的端口。流媒体连接建立完成后，
- 客户端发送一个播放命令（PLAY），服务器就开始在 UDP 上传送媒体流（RTP 包）到客户端。 在播放过程中客户端还可以向服务器发送命令来控制快进、快退和暂停等。
- 最后，客户端可发送一个终止命令（TERADOWN）来结束流媒体会话。

```wiki
    客户端->服务器：DESCRIBE
    服务器->客户端：200 OK (SDP)
    客户端->服务器：SETUP
    服务器->客户端：200 OK
    客户端->服务器：PLAY
    服务器->客户端：(RTP 包)
```

### 协议特点

- 可扩展性：新方法和参数很容易加入 RTSP。
- 易解析：RTSP 可由标准 HTTP 或 MIME 解析器解析。
- 安全：RTSP 使用网页安全机制。
- 独立于传输：RTSP 可使用不可靠数据报协议（EDP），可靠数据报协议（RDP）；如要实现应用级可靠，可使用可靠流协议。
- 多服务器支持：每个流可放在不同服务器上，用户端自动与不同服务器建立几个并发控制连接，媒体同步在传输层执行。
- 记录设备控制：协议可控制记录和回放设备。
- 流控与会议开始分离：仅要求会议初始化协议提供，或可用来创建惟一会议标识号。 特殊情况下，可用 SI 或 H.323 来邀请服务器入会。
- 适合专业应用：通过 SMPTE 时标，RTSP 支持帧级精度，允许远程数字编辑。
- 演示描述中立：协议没强加特殊演示或元文件，可传送所用格式类型；然而，演示描述至少必须包括一个 RTSP URL。
- 代理友好：此处，RTSP 明智地采用 HTTP 观念，使现在结构都可重用。结构包括 Internet 内容选择平台（PICS）。由于在大多数情况下控制连续媒体需要服务器状态，RTSP 不仅仅向 HTFP 添加方法。
- 适当的服务器控制：如用户启动一个流，必须也可以停止一个流。
- 传输协调：实际处理连续媒体流前，用户可协调传输方法。
- 性能协调：如基本特征无效，必须有一些清理机制让用户决定哪种方法没生效。

## 报文结构

### RTSP URL

一个终端用户是通过在播放器中输入 URL 地址开始进行观看流媒体业务的第一步，而对于使用RTSP协议的移动流媒体点播而言，URL 的一般写法如下：

一个以 “rtsp” 或是 “rtspu” 开始的 URL 链接用于指定当前使用的是 RTSP 协议。RTSP URL 的语法结构如下： `rtsp_url = (“rtsp:”| “rtspu:”) “//” host [“:”port”] /[abs_path]/content_name`

- host：可以是一个有效的域名或是 IP 地址。
- port：端口号，对于 RTSP 协议来说，缺省的端口号为 554。当我们在确认流媒体服务器提供的端口号为 554 时，此项可以省略。
- abs_path： 为 RTSP Server 中的媒体流资源标识，见下章节的录像资源命名。

RTSP URL 用来标识 RTSP Server 的媒体流资源，可以标识单一的媒体流资源，也可以标识多个媒体流资源的集合。

### RTSP 报文

RTSP 是一种基于文本的协议，用 CRLF 作为一行的结束符。

RTSP 有两类报文：**请求报文和响应报文**。请求报文是指从客户向服务器发送请求报文，响应报文是指从服务器到客户的回答。 由于 RTSP 是面向正文的（text-oriented），因此在报文中的每一个字段都是一些 ASCII 码串，因而每个字段的长度都是不确定的。 RTSP 报文由三部分组成，即**开始行、首部行和实体主体**。在请求报文中，开始行就是请求行。

RTSP 请求报文的方法包括：OPTIONS、DESCRIBE、SETUP、TEARDOWN、PLAY、PAUSE、GET_PARAMETER 和 SET_PARAMETER。

一个请求消息（a request message）即可以由客户端向服务端发起也可以由服务端向客户端发起。请求消息的语法结构如下： 

```wiki
Request = Request-Line
(general-header|request-header|entity-header)
CRLF
[message-body]
```

> Request Line

请求消息的第一行的语法结构如下：

```wiki
Request-Line = Method 空格 Request-URI 空格 RTSP-Version CRLF
```

其中在消息行中出现的第一个单词即是所使用的信令标志。目前已经有的信息标志如下：

```wiki
Method = “DESCRIBE” 
        | “ANNOUNCE”
        | “GET_PARAMETER”
        | “OPTIONS”
        | “PAUSE”
        | “PLAY”
        | “RECORD”
        | “REDIRECT”
        | “SETUP”
        | “SET_PARAMETER”
        | “TEARDOWN”
```

例子： `DESCRIBE rtsp://example.com/media.mp4 RTSP/1.0`

> Request Header Fields

在消息头中除了第一行的内容外，还有一些需求提供附加信息。其中有些是一定要的，后续我们会详细介绍经常用到的几个域的含义。 

```wiki
Request-header = Accept
    |	Accept-Encoding
    |	Accept-Language
    |	Authorization
    |	From
    |	If-Modified-Since
    |	Range
    |	Referer
    |	User-Agent
```

> 响应消息

响应消息的语法结构如下：

```wiki
Response = Status-Line 
(general-header |response-header|entity-header) 
CRLF 
[message-body]
```

> Status-Line

响应消息的第一行是状态行（status-line），每个元素之间用空格分开。除了最后的 CRLF 之外，在此行的中间不得有 CR 或是 LF 的出现。它的语法格式如下，

```wiki
Status-Line = RTSP-Version 空格 Status-Code 空格 Reason-Phrase CRLF
```

状态码（Status-Code）是一个三位数的整数，用于描述接收方对所收到请求消息的执行结果

Status-Code 的第一位数字指定了这个回复消息的种类，一共有 5 类：

- 1XX：Informational – 请求被接收到，继续处理。
- 2XX：Success – 请求被成功的接收，解析并接受。
- 3XX：Redirection – 为完成请求需要更多的操作。
- 4XX：Client Error – 请求消息中包含语法错误或是不能够被有效执行。
- 5XX：Server Error – 服务器响应失败，无法处理正确的有效的请求消息。

我们在处理问题时经常会遇到的状态码有如下：

```wiki
 Status-Code= “200": OK ,
              “400”: Bad Request ,
              “404”: Not Found ,
              “500”: Internal Server Error
```

> Response Header Fields

在响应消息的域中存放的是无法放在 Status-Line 中,而又需要传送给请求者的一些附加信息。

```wiki
Response-header = Location
Proxy-Authenticate
Public
Retry-After
Server
Vary
WWW-Authenticate
```

## 主要方法

| 方法          | 方向        | 对象  | 要求 |                             含义                             |
| :------------ | :---------- | :---- | ---- | :----------------------------------------------------------: |
| DESCRIBE      | C->S        | P   S | 推荐 | 检查演示或媒体对象的描述，DESCRIBE 的答复-响应组成媒体 RTSP 初始阶段 |
| ANNOUNCE      | C->S   S->C | P  S  | 可选 |  URL 识别的对象发送给服务 ，反之，ANNOUNCE 实时更新连接描述。  |
| GET_PARAMETER | C->S   S->C | P  S  | 可选 |             请求检查RUL指定的演示与媒体的参数值              |
| OPTIONS       | C->S   S->C | P  S  | 要求 |                 可在任意时刻发出 OPTIONS 请求                  |
| PAUSE         | C->S        | P  S  | 推荐 |                 PAUSE 请求引起流发送临时中断                  |
| PLAY          | C->S        | P  S  | 要求 |         PLAY 告诉服务器以 SETUP 指定的机制开始发送数据          |
| RECORD        | C->S        | P  S  | 可选 |           该方法根据演示描述初始化媒体数据记录范围           |
| REDIRECT      | S->C        | P  S  | 可选 |           重定向请求通知客户端连接到另一服务器地址           |
| SETUP         | C->S        | S     | 要求 |           对 URL 的 SETUP 请求指定用于流媒体的传输机制           |
| SET_PARAMETER | C->S   S->C | P  S  | 可选 |               请求设置演示或 URL 指定流的参数值                |
| TEARDOWN      | C->S        | P  S  | 要求 |         TEARDOWN 请求停止给定 URL 流发送，释放相关资源          |

注：P---演示，C---客户端,S---服务器，S（对象栏）---流

信令指的是在 Request-URI 中指定的需要被接收者完成的操作。信令（The method）大小写敏感，不能以字符 ”$” 开始，并且一定要是一个标记符。

## RTSP Header 参数

1. Accept：用于指定客户端可以接受的媒体描述信息类型。比如: `Accept: application/rtsl, application/sdp;level=2`。
2. Bandwidth：用于描述客户端可用的带宽值。
3. CSeq：指定了 RTSP 请求回应对的序列号，在每个请求或回应中都必须包括这个头字段。对每个包含一个给定序列号的请求消息，都会有一个相同序列号的回应消息。
4. Range：用于指定一个时间范围，可以使用 SMPTE、NTP 或 clock 时间单元。
5. Session：Session 头字段标识了一个 RTSP 会话。Session ID 是由服务器在 SETUP 的回应中选择的，客户端一当得到 Session ID 后，在以后的对 Session 的操作请求消息中都要包含 Session ID。
6. Transport: Transport 头字段包含客户端可以接受的转输选项列表，包括传输协议，地址端口，TTL 等。服务器端也通过这个头字段返回实际选择的具体选项。如: `Transport:RTP/AVP;multicast;ttl=127;mode="PLAY"`, `RTP/AVP;unicast;client_port=1289-1290;mode="PLAY"`

## 简单的 RTSP 消息交互过程

```wiki
C 表示 RTSP 客户端,S 表示 RTSP 服务端
第一步：查询服务器端可用方法 
C->S OPTION request  	询问 S 有哪些方法可用
S->C OPTION response 	S 回应信息的 public 头字段中包括提供的所有可用方法
第二步：得到媒体描述信息 
C->S DESCRIBE request  	要求得到 S 提供的媒体描述信息
S->C DESCRIBE response  S 回应媒体描述信息一般是sdp信息
第三步：建立 RTSP 会话 
C->S SETUP request 	Transport 头字段列出可接受的传输选项，请求S建立会话
S->C SETUP response S 建立会话通过 Transport 头字段返回选择的具体转输选项，并返回建立的 Session ID
第四步：请求开始传送数据
C->S PLAY request 	C 请求 S 开始发送数据
S->C PLAY response	S 回应该请求的信息
第五步： 数据传送播放中 
S->C 发送流媒体数据 	通过 RTP 协议传送数据
第六步：关闭会话，退出
C->S EARDOWN request 	C 请求关闭会话
S->C TEARDOWN response 	S 回应该请求
```
上述的过程只是标准的、友好的 RTSP 流程，但实际的需求中并不一定按此过程。 其中第三和第四步是必需的！第一步，只要服务器和客户端约定好有哪些方法可用，则 option 请求可以不要。第二步，如果我们有其他途径得到媒体初始化描述信息（比如 http 请求等等），则我们也不需要通过 RTSP 中的 describe 请求来完成。
---
title: RTSP
date: 2021-04-29 15:37:46
categories: 音视频
tags: 流媒体协议
---
## RTSP协议介绍
> 实时流协议（Real Time Streaming Protocol，RTSP）是一种网络应用协议，专为娱乐和通信系统的使用，以控制流媒体服务器。该协议用于创建和控制终端之间的媒体会话。媒体服务器的客户端发布VCR命令，例如播放，录制和暂停，以便于实时控制从服务器到客户端（视频点播）或从客户端到服务器（语音录音）的媒体流。是TCP/IP协议体系中的一个应用层协议, 由哥伦比亚大学, 网景和RealNetworks公司提交的IETF RFC标准. 对应的RFC编号是2326，可以在这里搜索 [RFC Editor](http://www.rfc-editor.org/search/rfc_search_detail.php)

该协议定义了一对多应用程序如何有效地通过IP网络传送多媒体数据. RTSP在体系结构上位于RTP和RTCP之上, 它使用TCP或RTP完成数据传输. RTSP被用于建立的控制媒体流的传输，它为多媒体服务扮演“网络远程控制”的角色。尽管有时**可以把RTSP控制信息和媒体数据流交织在一起传送**，但一般情况RTSP本身并不用于转送媒体流数据。媒体数据的传送可通过RTP/RTCP等协议来完成。

该协议用于C/S模型, 是一个基于文本的协议, 用于在客户端和服务器端建立和协商实时流会话.

### 协议体系

**一次基本的RTSP操作过程:**

- 首先，客户端连接到流服务器并发送一个RTSP描述命令（DESCRIBE）。
- 流服务器通过一个SDP描述来进行反馈，反馈信息包括流数量、媒体类型等信息。
- 客户端再分析该SDP描述，并为会话中的每一个流发送一个RTSP建立命令(SETUP)，RTSP建立命令告诉服务器客户端用于接收媒体数据的端口。流媒体连接建立完成后，
- 客户端发送一个播放命令(PLAY)，服务器就开始在UDP上传送媒体流（RTP包）到客户端。 在播放过程中客户端还可以向服务器发送命令来控制快进、快退和暂停等。
- 最后，客户端可发送一个终止命令(TERADOWN)来结束流媒体会话
```wiki
    客户端->服务器:DESCRIBE
    服务器->客户端:200 OK (SDP)
    客户端->服务器:SETUP
    服务器->客户端:200 OK
    客户端->服务器:PLAY
    服务器->客户端:(RTP包)
```

### 协议特点

- 可扩展性: 新方法和参数很容易加入RTSP.
- 易解析: RTSP可由标准HTTP或MIME解析器解析.
- 安全: RTSP使用网页安全机制.
- 独立于传输: RTSP可使用不可靠数据报协议(EDP), 可靠数据报协议(RDP); 如要实现应用级可靠, 可使用可靠流协议.
- 多服务器支持: 每个流可放在不同服务器上, 用户端自动与不同服务器建立几个并发控制连接, 媒体同步在传输层执行.
- 记录设备控制: 协议可控制记录和回放设备.
- 流控与会议开始分离: 仅要求会议初始化协议提供, 或可用来创建惟一会议标识号. 特殊情况下, 可用SIP或H.323来邀请服务器入会.
- 适合专业应用: 通过SMPTE时标, RTSP支持帧级精度, 允许远程数字编辑.
- 演示描述中立: 协议没强加特殊演示或元文件, 可传送所用格式类型; 然而, 演示描述至少必须包括一个RTSP URL.
- 代理与防火墙友好: 协议可由应用和传输层防火墙处理. 防火墙需要理解SETUP方法, 为UDP媒体流打开一个“缺口”.
- HTTP友好: 此处, RTSP明智地采用HTTP观念, 使现在结构都可重用. 结构包括Internet内容选择平台(PICS). 由于在大多数情况下控制连续媒体需要服务器状态, RTSP不仅仅向HTFP添加方法.
- 适当的服务器控制: 如用户启动一个流, 必须也可以停止一个流.
- 传输协调: 实际处理连续媒体流前, 用户可协调传输方法.
- 性能协调: 如基本特征无效, 必须有一些清理机制让用户决定哪种方法没生效. 这允许用户提出适合的用户界面.

## RTSP报文

### RTSP URL语法结构

一个终端用户是通过在播放器中输入URL地址开始进行观看流媒体业务的第一步，而对于使用RTSP协议的移动流媒体点播而言，URL的一般写法如下：

一个以“rtsp”或是“rtspu”开始的URL链接用于指定当前使用的是RTSP 协议。RTSP URL的语法结构如下： `rtsp_url = (“rtsp:”| “rtspu:”) “//” host [“:”port”] /[abs_path]/content_name`

- host: 可以是一个有效的域名或是IP地址。
- port: 端口号，对于RTSP协议来说，缺省的端口号为554。当我们在确认流媒体服务器提供的端口号为554时，此项可以省略 说明：当HMS服务器使用的端口号为554时，我们在写点播链接时，可以不用写明端口号，但当使用非554端口时，在RTSP URL中一定要指定相应的端口。
- abs_path: 为RTSPServer中的媒体流资源标识，见下章节的录像资源命名。

RTSP URL用来标识RTSPServer的媒体流资源，可以标识单一的媒体流资源，也可以标 识多个媒体流资源的集合。

### RTSP报文结构

RTSP是一种基于文本的协议，用CRLF作为一行的结束符。

RTSP有两类报文：**请求报文和响应报文**。请求报文是指从客户向服务器发送请求报文，响应报文是指从服务器到客户的回答。 由于 RTSP 是面向正文的(text-oriented)，因此在报文中的每一个字段都是一些 ASCII 码串，因而每个字段的长度都是不确定的。 RTSP报文由三部分组成，即**开始行、首部行和实体主体**。在请求报文中，开始行就是请求行.

RTSP请求报文的方法包括：OPTIONS、DESCRIBE、SETUP、TEARDOWN、PLAY、PAUSE、GET_PARAMETER和SET_PARAMETER。

一个请求消息（a request message）即可以由客户端向服务端发起也可以由服务端向客户端发起。请求消息的语法结构如下： 

```wiki
Request = Request-Line
(general-header|request-header|entity-header)
CRLF
[message-body]
```

#### Request Line

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

#### Request Header Fields

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

### 响应消息

响应消息的语法结构如下：

```wiki
Response = Status-Line 
(general-header |response-header|entity-header) 
CRLF 
[message-body]
```

#### Status-Line

响应消息的第一行是状态行（status-line），每个元素之间用空格分开。除了最后的CRLF之外，在此行的中间不得有CR或是LF的出现。它的语法格式如下，

```wiki
Status-Line = RTSP-Version 空格 Status-Code 空格 Reason-Phrase CRLF
```

状态码（Status-Code）是一个三位数的整数，用于描述接收方对所收到请求消息的执行结果

Status-Code的第一位数字指定了这个回复消息的种类，一共有5类：

- 1XX: Informational – 请求被接收到，继续处理
- 2XX: Success – 请求被成功的接收，解析并接受
- 3XX: Redirection – 为完成请求需要更多的操作
- 4XX: Client Error – 请求消息中包含语法错误或是不能够被有效执行
- 5XX: Server Error – 服务器响应失败，无法处理正确的有效的请求消息

我们在处理问题时经常会遇到的状态码有如下：

```wiki
 Status-Code= “200": OK ,
              “400”: Bad Request ,
              “404”: Not Found ,
              “500”: Internal Server Error
```

#### Response Header Fields**

在响应消息的域中存放的是无法放在Status-Line中,而又需要传送给请求者的一些附加信息。

```wiki
Response-header = Location
Proxy-Authenticate
Public
Retry-After
Server
Vary
WWW-Authenticate
```

### RTSP主要方法

| 方法          | 方向        | 对象  | 要求 |                             含义                             |
| :------------ | :---------- | :---- | ---- | :----------------------------------------------------------: |
| DESCRIBE      | C->S        | P   S | 推荐 | 检查演示或媒体对象的描述，DESCRIBE的答复-响应组成媒体RTSP初始阶段 |
| ANNOUNCE      | C->S   S->C | P  S  | 可选 |  URL识别的对象发送给服务 ，反之，ANNOUNCE实时更新连接描述。  |
| GET_PARAMETER | C->S   S->C | P  S  | 可选 |             请求检查RUL指定的演示与媒体的参数值              |
| OPTIONS       | C->S   S->C | P  S  | 要求 |                 可在任意时刻发出OPTIONS请求                  |
| PAUSE         | C->S        | P  S  | 推荐 |                 PAUSE请求引起流发送临时中断                  |
| PLAY          | C->S        | P  S  | 要求 |         PLAY告诉服务器以SETUP指定的机制开始发送数据          |
| RECORD        | C->S        | P  S  | 可选 |           该方法根据演示描述初始化媒体数据记录范围           |
| REDIRECT      | S->C        | P  S  | 可选 |           重定向请求通知客户端连接到另一服务器地址           |
| SETUP         | C->S        | S     | 要求 |           对URL的SETUP请求指定用于流媒体的传输机制           |
| SET_PARAMETER | C->S   S->C | P  S  | 可选 |               请求设置演示或URL指定流的参数值                |
| TEARDOWN      | C->S        | P  S  | 要求 |         TEARDOWN请求停止给定URL流发送，释放相关资源          |

注：P---演示，C---客户端,S---服务器，S（对象栏）---流

信令指的是在Request-URI中指定的需要被接收者完成的操作。信令（The method）大小写敏感，不能以字符”$”开始，并且一定要是一个标记符。

### RTSP头字段参数

1. Accept: 用于指定客户端可以接受的媒体描述信息类型。比如: `Accept: application/rtsl, application/sdp;level=2`
2. Bandwidth: 用于描述客户端可用的带宽值。
3. CSeq: 指定了RTSP请求回应对的序列号，在每个请求或回应中都必须包括这个头字段。对每个包含一个给定序列号的请求消息，都会有一个相同序列号的回应消息。
4. Range: 用于指定一个时间范围，可以使用SMPTE、NTP或clock时间单元。
5. Session: Session头字段标识了一个RTSP会话。Session ID 是由服务器在SETUP的回应中选择的，客户端一当得到Session ID后，在以后的对Session 的操作请求消息中都要包含Session ID.
6. Transport: Transport头字段包含客户端可以接受的转输选项列表，包括传输协议，地址端口，TTL等。服务器端也通过这个头字段返回实际选择的具体选项。如: `Transport:RTP/AVP;multicast;ttl=127;mode="PLAY"`, `RTP/AVP;unicast;client_port=1289-1290;mode="PLAY"`

### 简单的RTSP消息交互过程

```wiki
C表示RTSP客户端,S表示RTSP服务端
第一步：查询服务器端可用方法 
C->S OPTION request  	询问S有哪些方法可用
S->C OPTION response 	S回应信息的public头字段中包括提供的所有可用方法
第二步：得到媒体描述信息 
C->S DESCRIBE request  	要求得到S提供的媒体描述信息
S->C DESCRIBE response  S回应媒体描述信息一般是sdp信息
第三步：建立RTSP会话 
C->S SETUP request 	Transport头字段列出可接受的传输选项，请求S建立会话
S->C SETUP response S建立会话通过Transport头字段返回选择的具体转输选项，并返回建立的Session ID
第四步：请求开始传送数据
C->S PLAY request 	C请求S开始发送数据
S->C PLAY response	S回应该请求的信息
第五步： 数据传送播放中 
S->C 发送流媒体数据 	通过RTP协议传送数据
第六步：关闭会话，退出
C->S EARDOWN request 	C请求关闭会话
S->C TEARDOWN response 	S回应该请求
```
上述的过程只是标准的、友好的rtsp流程，但实际的需求中并不一定按此过程。 其中第三和第四步是必需的！第一步，只要服务器和客户端约定好有哪些方法可用，则option请求可以不要。第二步，如果我们有其他途径得到媒体初始化描述信息（比如http请求等等），则我们也不需要通过rtsp中的describe请求来完成。
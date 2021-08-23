---
title: rtsp 协议
top: false
cover: false
toc: true
mathjax: false
date: 2021-08-20 09:28:54
password:
summary: RFC2326 RTSP 文档翻译
categories: 流媒体
tags:
- rtsp
- translate
---

实时流协议 (RTSP) 是一种应用程序级协议，用于控制具有实时属性的数据的传送。 RTSP 提供了一个可扩展的框架，以实现实时数据（如音频和视频）的受控、按需交付。 数据源可以包括实时数据馈送和存储的剪辑。 该协议旨在控制多个数据传输会话，提供一种选择传输通道（例如 UDP、多播 UDP 和 TCP）的方法，并提供一种选择基于 RTP ([RFC 1889](https://datatracker.ietf.org/doc/html/rfc1889)) 的传输机制的方法。

## 1 介绍

### 1.1 目的

实时流协议 (RTSP) 建立和控制 单个或多个时间同步的连续流 音频和视频等媒体。它通常不提供 连续流本身，虽然连续的交错 带有控制流的媒体流是可能的（参见[第 10.12 节](#1012-嵌入交错二进制数据)）。 换句话说，RTSP 充当“网络远程控制” 多媒体服务器。

要控制的流集由演示描述(presentation description)定义。 本备忘录未定义演示描述的格式。

没有 RTSP 连接的概念； 相反，服务器维护一个由标识符标记的会话。 RTSP 会话与传输级连接（如 TCP 连接）无关。 在 RTSP 会话期间，RTSP 客户端可能会打开和关闭许多与服务器的可靠传输连接以发出 RTSP 请求。或者，它可以使用无连接传输协议，例如 UDP。

RTSP 控制的流可以使用 RTP [1]，但是 RTSP 不依赖于用于承载的传输机制 连续媒体。该协议在语法上有意相似 并运行到 HTTP/1.1 [2] 以便扩展机制到 HTTP 在大多数情况下也可以添加到 RTSP。但是，RTSP 的不同之处在于 HTTP 的几个重要方面：

* RTSP 引入了许多新方法并具有不同的协议标识符。
* 一个 RTSP 服务器需要在几乎所有情况下默认维护状态 与 HTTP 的无状态性质相反。
* RTSP 服务器和客户端都可以发出请求。
* 数据通过不同的协议在带外(out-of-band)传输（有一特例除外）。
* RTSP 被定义为使用 ISO 10646 (UTF-8) 而不是 ISO 8859-1， 与当前的 HTML 国际化努力一致 [3]。
* Request-URI 始终包含绝对 URI。因为 向后兼容历史错误，HTTP/1.1 [2] 请求中只携带绝对路径，并放置主机 名称在单独的头部字段中。

这使得“虚拟主机”的实现更容易，其中一台主机与一台主机 IP 地址托管多个文档树。

该协议支持以下操作：

* 从媒体服务器检索媒体：  
  客户端可以通过 HTTP 或其他一些方法请求演示描述。 如果演示正在被多播，则演示描述包含要用于连续媒体的多播地址和端口。 如果仅通过单播将演示发送给客户端，则出于安全原因，客户端会提供目的地。
* 邀请媒体服务器参加会议：  
  可以“邀请”媒体服务器加入现有会议，或者在演示中播放媒体，或者在演示中记录媒体的全部或子集。 这种模式对于分布式教学应用很有用。 会议中的几方可能会轮流“按下遥控器按钮”。
* 向现有演示添加媒体：
  特别是对于现场演示，如果服务器可以告诉客户端额外的媒体正在成为可用的。

RTSP 请求可以由代理、隧道和缓存处理，如 HTTP/1.1 [2]。

### 1.2 要求

本文档中的关键词“MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“MAY”和“OPTIONAL” 将按照 [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) [4] 中的描述进行解释。

### 1.3 术语

* 聚合控制（Aggregate control）：  
  服务器使用单个时间线控制多个流。对于音频/视频源，这意味着客户端可以发出单个播放或暂停消息来控制音频和视频源。
* 会议（Conference）：  
  多方多媒体演示，其中“多”意味着大于或等于一。
* 客户端（Client）：  
  客户端从媒体服务器请求连续的媒体数据。
* 连接（Connection）：  
  在两个以通讯为目的的程序之间建立传输层虚拟电路。
* 容器文件（Container file）：  
  可能包含多个媒体流的文件，这些媒体流在一起播放时通常包含一个演示。 RTSP 服务器可以提供对这些文件的聚合控制，尽管容器文件的概念没有嵌入到协议中。
* 连续媒体（Continuous media）：  
  source和sink之间存在时序关系的数据； 也就是说，接收端必须重现源端存在的时序关系。 连续媒体最常见的例子是音频和动态视频。 连续媒体可以是实时的（交互式），其中源和接收器之间存在“紧密”的时间关系，也可以是流媒体（回放），这种关系不太严格。
* 实体（Entity）：  
  作为请求或响应的有效负载传输的信息。一个实体由实体头字段形式的元信息和实体主体形式的内容组成，如第 8 节所述。
* 媒体初始化（Media initialization）：  
  数据类型/编解码器特定的初始化。 这包括诸如时钟速率、颜色表等。客户端播放媒体流所需的任何传输无关信息发生在流设置的媒体初始化阶段。
* 媒体参数（Media parameter）：  
  特定于媒体类型的参数，可以在流播放之前或期间更改。
* 媒体服务器（Media server）：
  为一个或多个媒体流提供播放或录制服务的服务器。 演示中的不同媒体流可能源自不同的媒体服务器。 媒体服务器可以与调用演示的 Web 服务器驻留在相同或不同的主机上。
* 媒体服务器间接（Media server indirection）：  
  将媒体客户端重定向到不同的媒体服务器。
* 媒体流（Media stream）：  
  单个媒体实例，例如音频流或视频流以及单个白板或共享应用程序组。 使用 RTP 时，流由 RTP 会话中的源创建的所有 RTP 和 RTCP 数据包组成。 这等效于 DSM-CC 流（[5]）的定义。
* 消息（Message）：  
  RTSP 通信的基本单元，由符合第 15 节中定义的语法并通过连接或无连接协议传输的八位字节的结构化序列组成。
* 参与者（Participant）：  
  会议成员。 参与者可以是机器，例如媒体记录或回放服务器。
* 演示（Presentation）：  
  一组一个或多个流作为完整的媒体馈送呈现给客户端，使用如下定义的演示描述。 在 RTSP 上下文中的大多数情况下，这意味着对这些流的聚合控制，但并非必须如此。
* 演示描述（Presentation description）：  
  演示描述包含有关演示中一个或多个媒体流的信息，例如编码集、网络地址和有关内容的信息。 其他 IETF 协议如 SDP ([RFC 2327](https://datatracker.ietf.org/doc/html/rfc2327) [6]) 使用术语“会话”来表示现场演示。 呈现描述可以采用多种不同的格式，包括但不限于会话描述格式SDP。
* 响应（Response）：  
  RTSP 响应。 如果是 HTTP 响应，则明确指出。
* 请求（Request）：  
  RTSP 请求。 如果是 HTTP 请求，则明确指出。
* RTSP 会话(RTSP session)：  
  完整的 RTSP“事务”，例如观看电影。 会话通常包括客户端为连续媒体流设置传输机制 (`SETUP`)，使用 `PLAY` 或 `RECORD` 启动流，并使用 `TEARDOWN` 关闭流。
* 传输初始化（Transport initialization）：  
  客户端和服务器之间传输信息（例如，端口号、传输协议）的协商。

### 1.4 协议属性

RTSP 具有以下属性：

* **可扩展**：  
  可以轻松地将新方法和参数添加到 RTSP。
* **易于解析**：  
  RTSP 可以被标准 HTTP 或 MIME 解析器解析。
* **安全**：  
  RTSP 重复使用网络安全机制。 所有 HTTP 身份验证机制，例如基本（[RFC 2068](https://datatracker.ietf.org/doc/html/rfc2068) [2，第 11.1 节]）和摘要身份验证（[RFC 2069](https://datatracker.ietf.org/doc/html/rfc2069) [8]）都可以直接应用。 还可以重用传输或网络层安全机制。
* **传输无关**：  
  RTSP 可以使用不可靠的数据报协议 (UDP) ([RFC 768](https://datatracker.ietf.org/doc/html/rfc768) [9])、可靠的数据报协议 (RDP（[RFC 1151](https://datatracker.ietf.org/doc/html/rfc1151)）未广泛使用的 [10]) 或可靠的流协议，例如 TCP ([RFC 793](https://datatracker.ietf.org/doc/html/rfc793) [11]），因为它实现了应用程序级的可靠性。
* **支持多服务器**：  
  演示中的每个媒体流都可以驻留在不同的服务器上。 客户端自动与不同的媒体服务器建立多个并发控制会话。 媒体同步在传输层执行。
* **记录设备的控制**：  
  该协议可以控制记录和播放设备，以及可以在两种模式之间交替的设备（“VCR”）。
* **流控与会议发起分离**：  
  流控与邀请媒体服务器加入会议分离。 唯一的要求是会议发起协议提供或可用于创建唯一的会议标识符。 特别地，SIP[12]或H.323[13]可用于邀请服务器参加会议。
* **适用于专业应用**：  
  RTSP 通过 SMPTE 时间戳支持帧级精度，允许远程数字编辑。
* **演示描述中立**：  
  协议不强加特定的演示描述或元文件格式，并且可以传达要使用的格式类型。 但是，演示描述必须至少包含一个 RTSP URI。
* **代理和防火墙友好**：  
  应用程序和传输层 (SOCKS [14]) 防火墙应该可以轻松处理该协议。 防火墙可能需要了解 `SETUP` 方法才能为 UDP 媒体流打开一个“洞”。
* **HTTP 友好**：  
  在合理的情况下，RTSP 重用 HTTP 概念，以便可以重用现有的基础设施。 该基础设施包括用于将标签与内容相关联的 PICS（互联网内容选择平台 [15,16]）。 然而，RTSP 不仅仅向 HTTP 添加方法，因为在大多数情况下控制连续媒体需要服务器状态。
* **适当的服务器控制**：  
  如果客户端可以启动流，则它必须能够停止流。 服务器不应以客户端无法停止流的方式开始向客户端流式传输。
* **传输协商**：  
  客户端可以在实际需要处理连续媒体流之前协商传输方法。
* **能力协商**：  
  如果基本功能被禁用，客户端必须有一些干净的机制来确定哪些方法不会被实现。 这允许客户端呈现适当的用户界面。 例如，如果不允许搜索，则用户界面必须能够禁止移动滑动位置指示器。

RTSP 的早期要求是多客户端功能。 但是，确定更好的方法是确保协议可以轻松扩展到多客户端场景。 流标识符可以被多个控制流使用，因此“传递远程”是可能的。 该协议不会解决几个客户端如何协商访问的问题； 这留给“社交协议”或其他一些楼层控制机制。

### 1.5 扩展 RTSP

由于并非所有媒体服务器都具有相同的功能，因此媒体服务器必然会支持不同的请求集。 例如：

* 服务器可能只能播放，因此不需要支持 `RECORD` 请求。
* 如果服务器仅支持实时事件，则它可能无法搜索（绝对定位）。
* 某些服务器可能不支持设置流参数，因此不支持 `GET_PARAMETER` 和 `SET_PARAMETER`。

服务器应该实现[第 12 节](#12-头部字段定义)中描述的所有头域。

由演示描述的创建者决定不要询问服务器的不可能。 这种情况与 HTTP/1.1 [2] 中的情况类似，其中 [H19.6] 中描述的方法不太可能在所有服务器上都得到支持。

RTSP 可以通过三种方式进行扩展，此处按支持的更改量级列出：

* 现有方法可以用新参数扩展，只要这些参数可以被接收者安全地忽略。（这相当于向 HTML 标签添加新参数。）如果客户端在不支持方法扩展时需要否定确认，则可以在 `Require`: 字段中添加与扩展对应的标签（参见第 12.32 节）。
* 可以添加新方法。如果消息的接收者不理解该请求，它会以错误代码 `501`（未实现）进行响应，并且发送者不应再次尝试使用此方法。客户端也可以使用 `OPTIONS` 方法来查询服务器支持的方法。服务器应该使用公共响应头列出它支持的方法。
* 可以定义新的协议版本，允许几乎所有方面（除了协议版本号的位置）改变。

### 1.6 操作模式

每个演示和媒体流都可以由一个 RTSP URL 标识。 整个演示和演示组成的媒体的属性由演示描述文件定义，其格式超出了本规范的范围。 呈现描述文件可以由客户端通过HTTP或电子邮件等其他方式获取，不一定存储在媒体服务器上。

出于本规范的目的，假定一个演示描述描述一个或多个演示，每个演示维护一个共同的时间轴。为了说明的简单并且不失一般性，假设演示描述恰好包含一个这样的演示。一个演示可能包含多个媒体流。

演示描述文件包含对构成演示的媒体流的描述，包括它们的编码、语言和其他参数，这些参数使客户端能够选择最合适的媒体组合。在此演示描述中，每个可由 RTSP 单独控制的媒体流由一个 RTSP URL 标识，该 URL 指向处理该特定媒体流的媒体服务器并命名存储在该服务器上的流。多个媒体流可以位于不同的服务器上；例如，音频和视频流可以跨服务器拆分以进行负载共享。该描述还列举了服务器能够使用的传输方法。

除了媒体参数之外，还需要确定网络目标地址和端口。 可以区分几种操作模式：

* 单播：  
  媒体通过客户端选择的端口号传输到 RTSP 请求的源。 或者，媒体在与 RTSP 相同的可靠流上传输。
* 多播，服务器选择地址：  
  媒体服务器选择多播地址和端口。 这是实况或近媒体点播传输的典型情况。
* 多播，客户端选择地址：  
  如果服务器要参加现有的多播会议，则多播地址、端口和加密密钥由会议描述给出，通过本规范范围之外的方式建立。

### 1.7 RTSP 状态

RTSP 控制可以通过独立协议发送的流，与控制通道无关。 例如，当数据通过 UDP 流动时，RTSP 控制可能发生在 TCP 连接上。 因此，即使媒体服务器没有收到 RTSP 请求，数据传递也会继续。 此外，在其生命周期内，单个媒体流可能由在不同 TCP 连接上顺序发出的 RTSP 请求控制。 因此，服务器需要维护“会话状态”才能将 RTSP 请求与流相关联。 状态转换在 A 节中描述。

RTSP 中的许多方法对状态没有贡献。 但是，以下在定义服务器上流资源的分配和使用方面起着核心作用：`SETUP`, `PLAY`, `RECORD`, `PAUSE`, 和 `TEARDOWN`。

* `SETUP`：  
  使服务器为流分配资源并启动 RTSP 会话。
* `PLAY` 和 `RECORD`：  
  在通过 SETUP 分配的流上开始数据传输。
* `PAUSE`：  
  在不释放服务器资源的情况下暂时停止流。
* `TEARDOWN`：  
  释放与流相关的资源。 RTSP 会话不再存在于服务器上。
  
  有助于状态的 RTSP 方法使用会话头字段（第 12.37 节）来标识其状态正在被操纵的 RTSP 会话。 服务器生成会话标识符以响应 SETUP 请求（[第 10.4 节](#104-setup)）。

### 1.8 与其他协议的关系

RTSP 在功能上与 HTTP 有一些重叠。 它还可能与 HTTP 交互，因为与流媒体内容的初始联系通常是通过网页进行的。 当前协议规范旨在允许 Web 服务器和实现 RTSP 的媒体服务器之间的不同切换点。 例如，可以使用 HTTP 或 RTSP 检索表示描述，这减少了基于 Web 浏览器的场景中的往返次数，同时还允许完全不依赖 HTTP 的独立 RTSP 服务器和客户端。

但是，RTSP 与 HTTP 的根本不同之处在于，数据传递是在不同的协议中带外进行的。 HTTP 是一种非对称协议，客户端发出请求，服务器响应。 **在 RTSP 中，媒体客户端和媒体服务器都可以发出请求。 RTSP 请求也不是无状态的； 他们可以设置参数并在请求被确认后很长时间继续控制媒体流。**

> 重用 HTTP 功能至少在两个方面具有优势，即安全性和代理。 要求非常相似，因此能够在缓存、代理和身份验证上采用 HTTP 工作是很有价值的。

虽然大多数实时媒体将使用 RTP 作为传输协议，但 RTSP 并不依赖于 RTP。

RTSP 假定存在可以表达包含多个媒体流的演示的静态和时间属性的演示描述格式。

## 2 符号约定

由于许多定义和语法与 HTTP/1.1 相同，因此本规范仅指向定义它们的部分，而不是复制它。 为简洁起见，[HX.Y] 指的是当前 HTTP/1.1 规范（[RFC 2068](https://datatracker.ietf.org/doc/html/rfc2068) [2]）的 X.Y 部分。

本文档中指定的所有机制均以散文和类似于 [H2.1] 中使用的增强型巴科斯-诺尔形式 (BNF) 进行描述。 它在 [RFC 2234](https://datatracker.ietf.org/doc/html/rfc2234) [17] 中有详细描述，不同之处在于该 RTSP 规范维护逗号分隔列表的“1#”符号。

在本备忘录中，我们使用缩进和较小类型的段落来提供背景和动机。 这是为了让没有参与规范制定的读者理解为什么事情是他们在 RTSP 中的方式。

## 3 协议参数

### 3.1 RTSP 版本

[[H3.1]](https://datatracker.ietf.org/doc/html/rfc2068#section-3.1) 适用，HTTP 被 RTSP 取代。

### 3.2 RTSP RUL

“rtsp”和“rtspu”方案用于通过RTSP协议引用网络资源。 本节定义了 RTSP URL 的特定于方案的语法和语义。

```text
rtsp_URL  =   ( "rtsp:" | "rtspu:" )
              "//" host [ ":" port ] [ abs_path ]
host      =   <IP 地址的合法 Internet 主机域名（点分十进制形式），如 RFC 1123 \cite{rfc1123} 的第 2.1 节所定义 >
port      =   *DIGIT
```

abs_path 在[H3.2.1] 中定义。

> 请注意，此时片段和查询标识符没有明确定义的含义，解释权留给 RTSP 服务器。

rtsp 方案要求使用可靠的协议（Internet 的 TCP 协议）发出命令，而 rtspu 方案使用不可靠的协议（Internet 的 UDP 协议）。

如果端口为空或未给出，则默认端口为 554。 语义是识别的资源可以由服务器上的 RTSP 控制，监听 TCP（方案“rtsp”）连接或主机端口上的 UDP（方案“rtspu”）数据包，资源的 Request-URI 是 rtsp_URL。

应尽可能避免在 URL 中使用 IP 地址（参见 [RFC 1924](https://datatracker.ietf.org/doc/html/rfc1924) [19]）。

演示或流由文本媒体标识符标识，使用 URL 的字符集和转义约定 [H3.2]（[RFC 1738](https://datatracker.ietf.org/doc/html/rfc1738) [20]）。 URL 可以指一个流或一个流的集合，即一个演示。 因此，第 10 节中描述的请求可以应用于整个演示或演示中的单个流。 请注意，某些请求方法只能应用于流，不能应用于演示，反之亦然。
  
例如，RTSP URL：`rtsp://media.example.com:554/twister/audiotrack`，标识了演示“twister”中的音频流，可以通过 TCP 连接向主机 `media.example.com` 的端口 `554` 发出的 RTSP 请求进行控制。

此外，RTSP URL：`rtsp://media.example.com:554/twister` 标识了可能由音频和视频流组成的演示“twister”。

这并不意味着在 URL 中引用流的标准方法。 演示描述定义了演示中的层次关系以及各个流的 URL。 演示描述可以将流命名为“a.mov”，并将整个演示命名为“b.mov”。

RTSP URL 的路径组件对客户端是不透明的，并且不暗示服务器的任何特定文件系统结构。

> 这种解耦还允许通过简单地替换 URL 中的方案来将表示描述与非 RTSP 媒体控制协议一起使用。

### 3.3 会议标识符

会议标识符对 RTSP 是不透明的，并使用标准 URI 编码方法进行编码（即，LWS 使用 % 进行转义）。 它们可以包含任何八位字节值。 会议标识符必须是全局唯一的。 对于 H.323，将使用会议 ID 值。

```text
conference-id =   1*xchar
```

会议标识符用于允许 RTSP 会话从媒体服务器正在参与的多媒体会议中获取参数。这些会议是由本规范范围之外的协议创建的，例如 H.323 [13] 或 SIP [12]。 例如，RTSP 客户端不是明确提供传输信息，而是要求媒体服务器使用会议描述中的值。

### 3.4 会话标识符

会话标识符是任意长度的不透明字符串。 线性空白必须是 URL 转义的。 会话标识符必须随机选择，并且必须至少有 8 个八位字节长，以便更难猜测。 （见第 16 节。）

```text
session-id   =   1*( ALPHA | DIGIT | safe )
```

### 3.5 SMPTE 相对时间戳
  
SMPTE 相对时间戳表示相对于剪辑开始的时间。 相对时间戳表示为 SMPTE 时间码，以确保帧级访问精度。 时间码的格式为 `hours:minutes:seconds:frames.subframes`，原点位于剪辑的开头。 默认的 smpte 格式为“SMPTE 30 drop”格式，帧率为 29.97 帧/秒。 通过使用“smpte time”的替代使用，可以支持其他 SMPTE 代码（例如“SMPTE 25”）。 对于时间值中的“帧”字段，可以假定值为 0 到 29。每秒 30 和 29.97 帧之间的差异是通过丢弃每分钟的前两个帧索引（值 00 和 01）来处理的，每十分钟除外 . 如果帧值为零，则可以省略。 子帧以帧的百分之一进行测量。

```text
smpte-range  =   smpte-type "=" smpte-time "-" [ smpte-time ]
smpte-type   =   "smpte" | "smpte-30-drop" | "smpte-25"
                                ; other timecodes may be added
smpte-time   =   1*2DIGIT ":" 1*2DIGIT ":" 1*2DIGIT [ ":" 1*2DIGIT ]
                    [ "." 1*2DIGIT ]

Examples:
    smpte=10:12:33:20-
    smpte=10:07:33-
    smpte=10:07:00-10:07:33:05.01
    smpte-25=10:07:00-10:07:33:05.01
```

### 3.6 正常播放时间(NPT)

正常播放时间 (NPT) 表示流相对于演示开始的绝对位置。时间戳由小数组成。小数点左边的部分可以用秒或小时、分钟和秒来表示。小数点右边的部分测量秒的分数。

演示的开始对应于 0.0 秒。负值没有定义。特殊常量 now 被定义为实时事件的当前时刻。它只能用于现场活动。

NPT 在 DSM-CC 中被定义为：“直观地说，NPT 是观众与节目相关联的时钟。它通常以数字方式显示在 VCR 上。NPT 在正常播放模式（比例 = 1）下正常推进，以更快的速度推进 “在快速向前扫描（高正比例）时的速率，在反向扫描（高负比例）时递减，并在暂停模式下固定。NPT（逻辑上）等效于 SMPTE 时间码。” [5]

```text
npt-range    =   ( npt-time "-" [ npt-time ] ) | ( "-" npt-time )
npt-time     =   "now" | npt-sec | npt-hhmmss
npt-sec      =   1*DIGIT [ "." *DIGIT ]
npt-hhmmss   =   npt-hh ":" npt-mm ":" npt-ss [ "." *DIGIT ]
npt-hh       =   1*DIGIT     ; any positive number
npt-mm       =   1*2DIGIT    ; 0-59
npt-ss       =   1*2DIGIT    ; 0-59

Examples:
    npt=123.45-125
    npt=12:05:35.3-
    npt=now-
```

> 语法符合 ISO 8601。 npt-sec 表示法针对自动生成进行了优化，ntp-hhmmss 表示法针对人类读者的使用进行了优化。 “now”常量允许客户端请求接收实时提要，而不是存储或延迟的版本。 这是必需的，因为绝对时间和零时间都不适合这种情况。

### 3.7 绝对时间

绝对时间表示为 ISO 8601 时间戳，使用 UTC (GMT)。 可以指示几分之一秒。

```text
utc-range    =   "clock" "=" utc-time "-" [ utc-time ]
utc-time     =   utc-date "T" utc-time "Z"
utc-date     =   8DIGIT                    ; < YYYYMMDD >
utc-time     =   6DIGIT [ "." fraction ]   ; < HHMMSS.fraction >

Example for November 8, 1996 at 14h37 and 20 and a quarter seconds UTC:

19961108T143720.25Z
```

### 3.8 选项标签

选项标签是用于在 RTSP 中指定新选项的唯一标识符。 这些标签用于 Require（第 12.32 节）和 Proxy-Require（第 12.27 节）标头字段。

语法:

```text
    option-tag   =   1*xchar
```

新 RTSP 选项的创建者应该使用反向域名作为该选项的前缀（例如，“com.foo.mynewfeature”是可以在“foo.com”上找到其发明者的特征的恰当名称），或者注册 互联网号码分配机构 (IANA) 的新选项。

#### 3.8.1 向 IANA 注册新的选项标签

注册新的 RTSP 选项时，应提供以下信息：

* 选项的名称和描述。 名称可以是任意长度，但长度不应超过 20 个字符。 名称不得包含任何空格、控制字符或句点。
* 表明谁对选项具有变更控制权（例如，IETF、ISO、ITU-T、其他国际标准化机构、财团或特定公司或公司集团）；
* 对进一步描述的引用（如果有），例如（按优先顺序）RFC、发表的论文、专利申请、技术报告、记录的源代码或计算机手册；
* 对于专有选项，联系信息（邮政和电子邮件地址）；

## 4 RTSP 消息

RTSP 是一种基于文本的协议，使用 UTF-8 编码（[RFC 2279](https://datatracker.ietf.org/doc/html/rfc2279) [21]）中的 ISO 10646 字符集。线路由 CRLF 终止，但接收方也应准备好将 CR 和 LF 自己解释为线路终止符。

> 基于文本的协议使以自描述方式添加可选参数变得更加容易。由于参数数量和命令频率低，处理效率不是问题。如果仔细完成基于文本的协议，还可以轻松地以 Tcl、Visual Basic 和 Perl 等脚本语言实现研究原型。
>
> 10646 字符集避免了棘手的字符集切换，但只要使用 US-ASCII，应用程序就看不到它。这也是用于 RTCP 的编码。 ISO 8859-1 直接转换为 Unicode，高位八位字节为零。设置了最高有效位的 ISO 8859-1 字符表示为 1100001x 10xxxxxx。 （参见 [RFC 2279](https://datatracker.ietf.org/doc/html/rfc2279) [21]）

RTSP 消息可以通过任何 8 位简洁的低层传输协议承载。

请求包含方法、方法所操作的对象以及进一步描述方法的参数。除非另有说明，否则方法是幂等的。方法还被设计为在媒体服务器上需要很少或不需要状态维护。

### 4.1 消息类型

详见[[H4.1]](https://datatracker.ietf.org/doc/html/rfc2068#section-4.1)

RTSP 消息包括从客户端到服务器的请求和响应 从服务器到客户端。

```rtsp
RTSP-message   = Request | Response     ; RTSP/1.0 messages
```

请求（第 5 节）和响应（第 6 节）消息使用 [RFC 822](https://datatracker.ietf.org/doc/html/rfc822) [9] 的通用消息格式来传输实体（消息的有效负载）。 两种类型的消息都包含一个起始行、一个或多个头域（也称为“头”）、一个空行（即在 CRLF 之前没有任何内容的行）指示头域的结尾，以及一个可选的 message-body。

```rtsp
generic-message = start-line
                  *message-header
                  CRLF
                  [ message-body ]

start-line      = Request-Line | Status-Line
```

为了健壮性，服务器应该忽略收到请求行的任何空行。换句话说，如果服务器正在读取消息开头的协议流并首先接收到 CRLF，则它应该忽略 CRLF。

### 4.2 消息头

详见[[H4.2]](https://datatracker.ietf.org/doc/html/rfc2068#section-4.2)

RTSP 标头字段，包括通用标头（第 4.5 节）、请求标头（第 5.3 节）、响应标头（第 6.2 节）和实体标头（第 7.1 节）字段，遵循与节中给出的相同的通用格式 [RFC 822 [9] 的 3.1 节](https://datatracker.ietf.org/doc/html/rfc822#section-3.1)。 每个头字段由一个名称后跟一个冒号（“:”）和字段值组成。 字段名称不区分大小写。 字段值前面可以有任意数量的 LWS，但首选单个 SP。 通过在每个额外的行之前至少添加一个 SP 或 HT，可以将头部字段扩展到多行。 应用程序在生成 RTSP 构造时应该遵循“通用形式”，因为可能存在一些无法接受通用形式之外的任何内容的实现。

```rtsp
message-header = field-name ":" [ field-value ] CRLF

field-name     = token
field-value    = *( field-content | LWS )

field-content  = <the OCTETs making up the field-value
                 and consisting of either *TEXT or combinations
                 of token, tspecials, and quoted-string >
```

接收具有不同字段名称的头字段的顺序并不重要。 然而，“好的做法”是先发送通用头域，然后是请求头域或响应头域，最后是实体头域。

当且仅当该头字段的整个字段值定义为逗号分隔列表 [即#(values)] 时，消息中可能会出现多个具有相同字段名称的消息头字段。 通过将每个后续字段值附加到第一个字段值，每个字段值用逗号分隔，必须可以将多个头部字段组合成一个“字段名称：字段值”对，而不改变消息的语义。 因此，接收具有相同字段名的头字段的顺序对组合字段值的解释很重要，因此当转发消息时，代理不得更改这些字段值的顺序。

### 4.3 消息体

详见[[H4.3]](https://datatracker.ietf.org/doc/html/rfc2068#section-4.3)

RTSP 消息的消息正文（如果有）用于携带与请求或响应相关联的实体正文。仅当应用了传输编码时，消息正文才与实体正文不同，如传输编码头字段所示（第 14.40 节）。

```rtsp
message-body = entity-body
             | <entity-body encoded as per Transfer-Encoding>
```

Transfer-Encoding 必须用于指示应用程序应用的任何传输编码，以确保消息的安全和正确传输。 Transfer-Encoding 是消息的属性，而不是实体的属性，因此可以由任何应用程序沿请求/响应链添加或删除。

消息中何时允许消息正文的规则因请求和响应而异。

通过在请求的消息头中包含 Content-Length 或 Transfer-Encoding 头字段来表示请求中消息正文的存在。只有当请求方法（第 5.1.1 节）允许实体主体时，消息主体才可以包含在请求中。

对于响应消息，消息中是否包含消息体取决于请求方法和响应状态代码（第 6.1.1 节）。对 HEAD 请求方法的所有响应不得包含消息正文，即使实体头字段的存在可能使人们相信它们确实如此。所有 1xx（信息性）、204（无内容）和 304（未修改）响应不得包含消息正文。所有其他响应都包含消息正文，尽管它的长度可能为零。

### 4.4 消息长度

当消息包含消息正文时，该正文的长度由以下之一确定（按优先顺序）：

1. 任何不包括消息体的响应消息（例如 1xx、204 和 304 响应）总是以头字段后的第一个空行结束，而不管消息中存在的实体头字段。 （注意：空行仅包含 CRLF。）
2. 如果存在 Content-Length 头字段（第 12.14 节），则其值（以字节为单位）表示消息体的长度。 如果该头域不存在，则假定值为零。
3. 由服务器关闭连接。（关闭连接不能用于指示请求正文的结束，因为这将使服务器无法发回响应。）

> 请注意，RTSP（目前）不支持 HTTP/1.1 “分块”传输编码（参见 [H3.6]）并且需要存在 Content-Length 头字段。
>> 鉴于返回的表示描述的长度适中，服务器应该始终能够确定它的长度，即使它是动态生成的，这使得分块传输编码变得不必要。 即使 Content-Length 必须存在，如果有任何实体主体，即使没有明确给出长度，规则也能确保合理的行为。

## 5 通用头部字段

参见 [[H4.5]](https://datatracker.ietf.org/doc/html/rfc2068#section-4.5)，除了 Pragma、Transfer-Encoding 和 Upgrade 标头没有定义：

```rtsp
general-header     =     Cache-Control     ; Section 12.8  
                   |     Connection        ; Section 12.10  
                   |     Date              ; Section 12.18  
                   |     Via               ; Section 12.43  
```

## 6 请求

从客户端到服务器（反之亦然）的请求消息在该消息的第一行中包括要应用于资源的方法、资源的标识符和使用的协议版本。

```rtsp
Request     =       Request-Line          ; Section 6.1
            *(      general-header        ; Section 5
            |       request-header        ; Section 6.2
            |       entity-header )       ; Section 8.1
                    CRLF
                    [ message-body ]      ; Section 4.3
```

### 6.1 请求行

```rtsp
Request-Line = Method SP Request-URI SP RTSP-Version CRLF

 Method         =         "DESCRIBE"              ; Section 10.2
                |         "ANNOUNCE"              ; Section 10.3
                |         "GET_PARAMETER"         ; Section 10.8
                |         "OPTIONS"               ; Section 10.1
                |         "PAUSE"                 ; Section 10.6
                |         "PLAY"                  ; Section 10.5
                |         "RECORD"                ; Section 10.11
                |         "REDIRECT"              ; Section 10.10
                |         "SETUP"                 ; Section 10.4
                |         "SET_PARAMETER"         ; Section 10.9
                |         "TEARDOWN"              ; Section 10.7
                |         extension-method

extension-method = token

Request-URI = "*" | absolute_URI

RTSP-Version = "RTSP" "/" 1*DIGIT "." 1*DIGIT
```

### 6.2 请求头域

```rtsp
request-header  =          Accept                   ; Section 12.1
                |          Accept-Encoding          ; Section 12.2
                |          Accept-Language          ; Section 12.3
                |          Authorization            ; Section 12.5
                |          From                     ; Section 12.20
                |          If-Modified-Since        ; Section 12.23
                |          Range                    ; Section 12.29
                |          Referer                  ; Section 12.30
                |          User-Agent               ; Section 12.41
```

> 请注意，与 HTTP/1.1 [2] 相比，RTSP 请求始终包含绝对 URL（即，包括方案、主机和端口）而不仅仅是绝对路径。
>> HTTP/1.1 要求服务器理解绝对 URL，但客户端应该使用 Host 请求标头。 这纯粹是为了向后兼容 HTTP/1.0 服务器，这一考虑不适用于 RTSP。

Request-URI 中的星号“*”表示该请求并不适用于特定资源，而是适用于服务器本身，并且仅在使用的方法不一定适用于资源时才允许。 一个例子是：

```rtsp
OPTIONS * RTSP/1.0
```

## 7 响应

[H6] 适用，但 HTTP-Version 被 RTSP-Version 取代。 此外，RTSP 定义了额外的状态代码，并没有定义一些 HTTP 代码。 表 1 中定义了有效的响应代码和它们可以使用的方法。

接收并解释请求消息后，接收方以 RTSP 响应消息进行响应。

```rtsp
Response    =     Status-Line         ; Section 7.1
            *(    general-header      ; Section 5
            |     response-header     ; Section 7.1.2
            |     entity-header )     ; Section 8.1
                  CRLF
                  [ message-body ]    ; Section 4.3
```

### 7.1 状态行

响应消息的第一行是状态行，由协议版本后跟数字状态代码和与状态代码关联的文本短语组成，每个元素由 SP 字符分隔。 除了最后的 CRLF 序列外，不允许使用 CR 或 LF。

```rtsp
Status-Line =   RTSP-Version SP Status-Code SP Reason-Phrase CRLF
```

#### 7.1.1 状态码和原因短语

Status-Code 元素是尝试理解和满足请求的 3 位整数结果代码。 这些代码在第 11 节中有完整定义。原因短语旨在提供状态代码的简短文本描述。 状态代码供自动机使用，原因短语供人类用户使用。 客户端不需要检查或显示原因短语。

状态代码的第一位数字定义了响应的类别。最后两位数字没有任何分类作用。第一个数字有 5 个值：

* 1xx：信息性 - 收到请求，继续处理
* 2xx：成功 - 成功接收、理解和接受动作
* 3xx：重定向 - 必须采取进一步行动才能完成请求
* 4xx：客户端错误 - 请求包含错误的语法或无法完成
* 5xx：服务器错误 - 服务器未能满足明显有效的请求

为 RTSP/1.0 定义的数字状态代码的各个值以及一组相应的原因短语示例如下所示。此处列出的原因短语仅供推荐 - 它们可能会被本地等效项替换而不影响协议。请注意，RTSP 采用了大多数 HTTP/1.1 [2] 状态代码，并添加了从 x50 开始的 RTSP 特定状态代码，以避免与新定义的 HTTP 状态代码发生冲突。

```rtsp
Status-Code =     "100"      ; Continue
            |     "200"      ; OK
            |     "201"      ; Created
            |     "250"      ; Low on Storage Space
            |     "300"      ; Multiple Choices
            |     "301"      ; Moved Permanently
            |     "302"      ; Moved Temporarily
            |     "303"      ; See Other
            |     "304"      ; Not Modified
            |     "305"      ; Use Proxy
            |     "400"      ; Bad Request
            |     "401"      ; Unauthorized
            |     "402"      ; Payment Required
            |     "403"      ; Forbidden
            |     "404"      ; Not Found
            |     "405"      ; Method Not Allowed
            |     "406"      ; Not Acceptable
            |     "407"      ; Proxy Authentication Required
            |     "408"      ; Request Time-out
            |     "410"      ; Gone
            |     "411"      ; Length Required
            |     "412"      ; Precondition Failed
            |     "413"      ; Request Entity Too Large
            |     "414"      ; Request-URI Too Large
            |     "415"      ; Unsupported Media Type
            |     "451"      ; Parameter Not Understood
            |     "452"      ; Conference Not Found
            |     "453"      ; Not Enough Bandwidth
            |     "454"      ; Session Not Found
            |     "455"      ; Method Not Valid in This State
            |     "456"      ; Header Field Not Valid for Resource
            |     "457"      ; Invalid Range
            |     "458"      ; Parameter Is Read-Only
            |     "459"      ; Aggregate operation not allowed
            |     "460"      ; Only aggregate operation allowed
            |     "461"      ; Unsupported transport
            |     "462"      ; Destination unreachable
            |     "500"      ; Internal Server Error
            |     "501"      ; Not Implemented
            |     "502"      ; Bad Gateway
            |     "503"      ; Service Unavailable
            |     "504"      ; Gateway Time-out
            |     "505"      ; RTSP Version not supported
            |     "551"      ; Option not supported
            |     extension-code

extension-code  =     3DIGIT
Reason-Phrase   =     *<TEXT, excluding CR, LF>
```

RTSP 状态代码是可扩展的。RTSP 应用程序不需要理解所有注册状态代码的含义，尽管这种理解显然是可取的。 但是，应用程序必须理解任何状态代码的类别，如第一个数字所示，并将任何无法识别的响应视为等同于该类的 x00 状态代码，除非不能缓存无法识别的响应。例如，如果客户端收到无法识别的 431 状态代码，它可以安全地假设其请求有问题，并将响应视为收到 400 状态代码。 在这种情况下，用户代理应该将响应返回的实体呈现给用户，因为该实体可能包含解释异常状态的人类可读信息。

| Code | reason                           | RTSP method     |
| ---- | -------------------------------- | --------------- |
| 100  | Continue                         | all             |
|      |                                  |
| 200  | OK                               | all             |
| 201  | Created                          | RECORD          |
| 250  | Low on Storage Space             | RECORD          |
|      |                                  |
| 300  | Multiple Choices                 | all             |
| 301  | Moved Permanently                | all             |
| 302  | Moved Temporarily                | all             |
| 303  | See Other                        | all             |
| 305  | Use Proxy                        | all             |
|      |                                  |
| 400  | Bad Request                      | all             |
| 401  | Unauthorized                     | all             |
| 402  | Payment Required                 | all             |
| 403  | Forbidden                        | all             |
| 404  | Not Found                        | all             |
| 405  | Method Not Allowed               | all             |
| 406  | Not Acceptable                   | all             |
| 407  | Proxy Authentication Required    | all             |
| 408  | Request Timeout                  | all             |
| 410  | Gone                             | all             |
| 411  | Length Required                  | all             |
| 412  | Precondition Failed              | DESCRIBE, SETUP |
| 413  | Request Entity Too Large         | all             |
| 414  | Request-URI Too Long             | all             |
| 415  | Unsupported Media Type           | all             |
| 451  | Invalid parameter                | SETUP           |
| 452  | Illegal Conference Identifier    | SETUP           |
| 453  | Not Enough Bandwidth             | SETUP           |
| 454  | Session Not Found                | all             |
| 455  | Method Not Valid In This State   | all             |
| 456  | Header Field Not Valid           | all             |
| 457  | Invalid Range                    | PLAY            |
| 458  | Parameter Is Read-Only           | SET_PARAMETER   |
| 459  | Aggregate Operation Not Allowed  | all             |
| 460  | Only Aggregate Operation Allowed | all             |
| 461  | Unsupported Transport            | all             |
| 462  | Destination Unreachable          | all             |
|      |                                  |
| 500  | Internal Server Error            | all             |
| 501  | Not Implemented                  | all             |
| 502  | Bad Gateway                      | all             |
| 503  | Service Unavailable              | all             |
| 504  | Gateway Timeout                  | all             |
| 505  | RTSP Version Not Supported       | all             |
| 551  | Option not support               | all             |

表1：状态代码及其在 RTSP 方法中的使用

#### 7.1.2 响应头域

响应头字段允许请求接收者传递关于响应的附加信息，这些信息不能放在状态行中。 这些头域提供了关于服务器的信息以及关于对由 Request-URI 标识的资源的进一步访问的信息。

```rtsp
response-header =     Location             ; Section 12.25
                |     Proxy-Authenticate   ; Section 12.26
                |     Public               ; Section 12.28
                |     Retry-After          ; Section 12.31
                |     Server               ; Section 12.36
                |     Vary                 ; Section 12.42
                |     WWW-Authenticate     ; Section 12.44
```

只有结合协议版本的更改才能可靠地扩展响应头字段名称。 然而，新的或实验性的头域可以被赋予响应头域的语义，如果通信中的所有各方都将它们识别为响应头域。 无法识别的头字段被视为实体头字段。

## 8 实体

如果请求方法或响应状态代码不受其他限制，则请求和响应消息可以传输实体。 实体由实体头字段和实体主体组成，尽管某些响应将仅包含实体头。

在本节中，发送者和接收者都指的是客户端或服务器，具体取决于谁发送和谁接收实体。

### 8.1 实体头域

实体头字段定义了关于实体主体的可选元信息，或者，如果不存在主体，则定义有关请求标识的资源的元信息。

```rtsp
entity-header       =    Allow               ; Section 12.4
                    |    Content-Base        ; Section 12.11
                    |    Content-Encoding    ; Section 12.12
                    |    Content-Language    ; Section 12.13
                    |    Content-Length      ; Section 12.14
                    |    Content-Location    ; Section 12.15
                    |    Content-Type        ; Section 12.16
                    |    Expires             ; Section 12.19
                    |    Last-Modified       ; Section 12.24
                    |    extension-header
extension-header    =    message-header
```

扩展头机制允许在不改变协议的情况下定义额外的实体头字段，但不能假设这些字段是接收者可识别的。 无法识别的头域应该被接收者忽略并由代理转发。

### 8.2 实体主体

详见[[H7.2]](https://datatracker.ietf.org/doc/html/rfc2068#section-7.2)

与 RTSP 请求或响应一起发送的实体主体（如果有）采用实体头字段定义的格式和编码。

```rtsp
entity-body    = *OCTET 
```

如第 4.3 节所述，实体正文仅在消息正文存在时才出现在消息中。 实体主体是通过对可能已被应用以确保消息的安全和正确传输的任何传输编码进行解码而从消息主体中获得的。

## 9 连接

RTSP 请求可以通过几种不同的方式传输：

* 用于多个请求-响应事务的持久传输连接；
* 每个请求/响应事务一个连接；
* 无连接模式。

传输连接的类型由 RTSP URI（[第 3.2 节](#32-rtsp-rul)）定义。 对于“rtsp”方案，假设一个持久连接，而“rtspu”方案要求在不建立连接的情况下发送RTSP请求。

与 HTTP 不同，RTSP 允许媒体服务器向媒体客户端发送请求。但是，这仅支持持久连接，因为媒体服务器没有可靠的方式到达客户端。此外，这是从媒体服务器到客户端的请求可能穿过防火墙的唯一方式。

### 9.1 流水线

支持持久连接或无连接模式的客户端可以“管道”其请求（即发送多个请求而不等待每个响应）。 服务器必须按照接收请求的相同顺序发送对这些请求的响应。

### 9.2 可靠性和确认

除非请求被发送到多播组，否则接收者会确认请求。 如果没有确认，发送方可能会在一个往返时间 (RTT) 超时后重新发送相同的消息。 往返时间的估计与 TCP ([RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123)) [18] 中一样，初始往返值为 500 毫秒。 一个实现可以缓存最后一个 RTT 测量作为未来连接的初始值。

如果使用可靠的传输协议来承载 RTSP，则不得重传请求； RTSP 应用程序必须依赖底层传输来提供可靠性。

> 如果底层可靠传输（例如 TCP）和 RTSP 应用程序都重传请求，则每个数据包丢失可能导致两次重传。 接收器通常不能利用应用层重传，因为传输栈不会在第一次尝试到达接收器之前传递应用层重传。 如果丢包是由于拥塞造成的，不同层的多次重传会加剧拥塞。
>
> 如果在小型 RTT LAN 上使用 RTSP，则用于优化初始 TCP 往返估计的标准程序，例如 T/TCP ([RFC 1644](https://datatracker.ietf.org/doc/html/rfc1644)) [22] 中使用的程序，可能会有所帮助。

时间戳标头（第 12.38 节）用于避免重传歧义问题 [23, p. 301] 并消除了对卡恩算法的需要。

每个请求在 CSeq 标头中携带一个序列号（第 12.17 节），对于每个传输的不同请求，该序列号递增 1。如果由于缺乏确认而重复请求，则该请求必须携带原始序列号（即，序列号不递增）。

实现 RTSP 的系统必须支持通过 TCP 承载 RTSP，并且可以支持 UDP。 RTSP 服务器的默认端口对于 UDP 和 TCP 都是 554。

发往同一个控制端点的多个 RTSP 数据包可能会被打包到单个低层 PDU 中或被封装到 TCP 流中。 RTSP 数据可以与 RTP 和 RTCP 数据包交错。

与 HTTP 不同，RTSP 消息必须包含一个 Content-Length 标头，只要该消息包含有效负载。否则，RTSP 包以紧跟在最后一个消息头之后的空行终止。

## 10 方法定义

方法标记指示要在请求 URI 标识的资源上执行的方法。 该方法区分大小写。 未来可能会定义新的方法。 方法名称不能以 $ 字符（十进制 24）开头，并且必须是标记。 方法总结在表 2 中。

| method        | direction  | object | requirement               |
| ------------- | ---------- | ------ | ------------------------- |
| DESCRIBE      | C->S       | P,S    | recommended               |
| ANNOUNCE      | C->S, S->C | P,S    | optional                  |
| GET_PARAMETER | C->S, S->C | P,S    | optional                  |
| OPTIONS       | C->S, S->C | P,S    | required (S->C: optional) |
| PAUSE         | C->S       | P,S    | recommended               |
| PLAY          | C->S       | P,S    | required                  |
| RECORD        | C->S       | P,S    | optional                  |
| REDIRECT      | S->C       | P,S    | optional                  |
| SETUP         | C->S       | S      | required                  |
| SET_PARAMETER | C->S, S->C | P,S    | optional                  |
| TEARDOWN      | C->S       | P,S    | required                  |

表 2：RTSP 方法概述、它们的方向以及它们操作的对象（P: presentation, S: stream）

> 注意：建议使用 PAUSE，但不是必需的，因为可以构建不支持此方法的全功能服务器，例如，用于实时提要。 如果一个服务器不支持一个特定的方法，它必须返回“501 Not Implemented”并且客户端不应该为这个服务器再次尝试这个方法。

### 10.1 OPTIONS

该行为等同于 [H9.2] 中描述的行为。一个 `OPTIONS` 请求可以在任何时候发出，例如，如果客户端将要尝试一个非标准请求。 它不会影响服务器状态。例如：

```rtsp
C->S:   OPTIONS * RTSP/1.0
        CSeq: 1
        Require: implicit-play
        Proxy-Require: gzipped-messages

S->C:   RTSP/1.0 200 OK
        CSeq: 1
        Public: DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE
```

> 请注意，这些必然是虚构的功能（人们希望我们不会故意忽略一个真正有用的功能，以便我们在本节中有一个强有力的例子）。

### 10.2 DESCRIBE

`DESCRIBE` 方法从服务器检索由请求 URL 标识的演示或媒体对象的描述。 它可以使用 Accept 头来指定客户端理解的描述格式。服务器以所请求资源的描述作为响应。`DESCRIBE` 回复-响应对构成了 RTSP 的媒体初始化阶段。例如：

```rtsp
C->S:   DESCRIBE rtsp://server.example.com/fizzle/foo RTSP/1.0
        CSeq: 312
        Accept: application/sdp, application/rtsl, application/mheg

S->C:   RTSP/1.0 200 OK
        CSeq: 312
        Date: 23 Jan 1997 15:35:06 GMT
        Content-Type: application/sdp
        Content-Length: 376

        v=0
        o=mhandley 2890844526 2890842807 IN IP4 126.16.64.4
        s=SDP Seminar
        i=A Seminar on the session description protocol
        u=http://www.cs.ucl.ac.uk/staff/M.Handley/sdp.03.ps
        e=mjh@isi.edu (Mark Handley)
        c=IN IP4 224.2.17.12/127
        t=2873397496 2873404696
        a=recvonly
        m=audio 3456 RTP/AVP 0
        m=video 2232 RTP/AVP 31
        m=whiteboard 32416 UDP WB
        a=orient:portrait
```

`DESCRIBE` 响应必须包含它描述的资源的所有媒体初始化信息。如果媒体客户端从 `DESCRIBE` 以外的来源获得演示描述，并且该描述包含一组完整的媒体初始化参数，则客户端应该使用这些参数，而不是通过 RTSP 请求相同媒体的描述。此外，服务器不应该使用 `DESCRIBE` 响应作为媒体间接的手段。

> 需要建立明确的基本规则，以便客户端能够清楚地知道何时通过 `DESCRIBE` 请求媒体初始化信息，何时不请求。通过强制 `DESCRIBE` 响应包含它描述的一组流的所有媒体初始化，并阻止将 `DESCRIBE` 用于媒体间接，我们避免了可能由其他方法导致的循环问题。
> 媒体初始化是任何基于 RTSP 的系统的要求，但 RTSP 规范并未规定必须通过 `DESCRIBE` 方法完成此操作。 RTSP 客户端可以通过三种方式接收初始化信息：
>
> * 通过 RTSP 的 `DESCRIBE` 方法；
> * 通过一些其他协议（HTTP、电子邮件附件等）；
> * 通过命令行或标准输入（因此作为使用 SDP 文件或其他媒体初始化格式启动的浏览器帮助应用程序工作）。
>
> 为了实际互操作性，强烈建议最小服务器支持 `DESCRIBE` 方法，并强烈建议最小客户端支持充当“帮助应用程序”的能力，从标准输入、命令行和/或其他适合客户端操作环境的方式。

### 10.3 ANNOUNCE

`ANNOUNCE` 方法有两个目的：

* 当从客户端发送到服务器时，`ANNOUNCE` 将请求 URL 标识的演示或媒体对象的描述发布到服务器。
* 当从服务器发送到客户端时，`ANNOUNCE` 实时更新会话描述。

如果将新媒体流添加到演示中（例如，在现场演示期间），则应再次发送整个演示描述，而不仅仅是附加组件，以便可以删除组件。例如：

```rtsp
C->S:   ANNOUNCE rtsp://server.example.com/fizzle/foo RTSP/1.0
        CSeq: 312
        Date: 23 Jan 1997 15:35:06 GMT
        Session: 47112344
        Content-Type: application/sdp
        Content-Length: 332

        v=0
        o=mhandley 2890844526 2890845468 IN IP4 126.16.64.4
        s=SDP Seminar
        i=A Seminar on the session description protocol
        u=http://www.cs.ucl.ac.uk/staff/M.Handley/sdp.03.ps
        e=mjh@isi.edu (Mark Handley)
        c=IN IP4 224.2.17.12/127
        t=2873397496 2873404696
        a=recvonly
        m=audio 3456 RTP/AVP 0
        m=video 2232 RTP/AVP 31

S->C:   RTSP/1.0 200 OK
        CSeq: 312
```

### 10.4 SETUP

对 URI 的 `SETUP` 请求指定用于流媒体的传输机制。客户端可以对已经播放的流发出 `SETUP` 请求以更改传输参数，服务器可以允许。如果它不允许这样做，它必须以错误“455 Method Not Valid In This State”响应。为了任何中间防火墙的利益，客户端必须指明传输参数，即使它对这些参数没有影响，例如，服务器在哪里通告固定的多播地址。

> 由于 `SETUP` 包括所有传输初始化信息，防火墙和其他中间网络设备（需要此信息）可以免于解析 `DESCRIBE` 响应的更艰巨的任务，该响应已保留用于媒体初始化。

`Transport` 头指定客户端可接受的传输参数，用于数据传输；响应将包含服务器选择的传输参数。

```rtsp
C->S:   SETUP rtsp://example.com/foo/bar/baz.rm RTSP/1.0
        CSeq: 302
        Transport: RTP/AVP;unicast;client_port=4588-4589

S->C:   RTSP/1.0 200 OK
        CSeq: 302
        Date: 23 Jan 1997 15:35:06 GMT
        Session: 47112344
        Transport: RTP/AVP;unicast;
          client_port=4588-4589;server_port=6256-6257
```

服务器生成会话标识符以响应 `SETUP` 请求。 如果对服务器的 `SETUP` 请求包含会话标识符，则服务器必须将此设置请求捆绑到现有会话中或返回错误“459 Aggregate Operation Not Allowed”（参见第 11.3.10 节）。

### 10.5 PLAY

`PLAY` 方法告诉服务器通过 `SETUP` 中指定的机制开始发送数据。**在任何未完成的 `SETUP` 请求被确认为成功之前，客户端不得发出 `PLAY` 请求。**

`PLAY` 请求将正常播放时间定位到指定范围的开始，并传送流数据直到到达范围结束。`PLAY` 请求可能会被流水线化（排队）；服务器必须将 `PLAY` 请求按顺序执行。也就是说，**在前一个 `PLAY` 请求仍处于活动状态时到达的 `PLAY` 请求会被延迟，直到第一个请求完成。**

> 这允许精确编辑。

例如，无论下面示例中的两个 `PLAY` 请求到达的间隔有多近，服务器将首先播放 10 到 15 秒，然后紧随其后的是 20 到 25 秒，最后是 30 秒到结束。

```rtsp
C->S:   PLAY rtsp://audio.example.com/audio RTSP/1.0
        CSeq: 835
        Session: 12345678
        Range: npt=10-15

C->S:   PLAY rtsp://audio.example.com/audio RTSP/1.0
        CSeq: 836
        Session: 12345678
        Range: npt=20-25

C->S:   PLAY rtsp://audio.example.com/audio RTSP/1.0
        CSeq: 837
        Session: 12345678
        Range: npt=30-
```

有关更多示例，请参阅 `PAUSE` 请求的说明。

没有 `Range` 标头的 `PLAY` 请求是合法的。 除非流已暂停，否则它会从头开始播放流。 如果流已通过 `PAUSE` 暂停，则流传输将在暂停点恢复。 如果一个流正在播放，这样的 `PLAY` 请求不会引起进一步的动作，客户端可以使用它来测试服务器的活跃度。

`Range` 头也可能包含一个时间参数。此参数指定应开始播放的 UTC 时间。如果在指定时间后收到消息，则立即开始播放。时间参数可用于辅助从不同源获得的流的同步。

对于点播流，服务器回复将播放的实际范围。如果媒体源需要将请求的范围与有效的帧边界对齐，则这可能与请求的范围不同。如果请求中未指定范围，则在回复中返回当前位置。回复中的范围单位与请求中的范围单位相同。

播放完所需的范围后，演示会自动暂停，就像发出了暂停请求一样。

以下示例从 SMPTE 时间码 0:10:20 开始播放整个演示，直到剪辑结束。播放时间为1997年1月23日15时36分。

```rtsp
     C->S: PLAY rtsp://audio.example.com/twister.en RTSP/1.0
           CSeq: 833
           Session: 12345678
           Range: smpte=0:10:20-;time=19970123T153600Z

     S->C: RTSP/1.0 200 OK
           CSeq: 833
           Date: 23 Jan 1997 15:35:06 GMT
           Range: smpte=0:10:22-;time=19970123T153600Z
```

为了回放现场演示的录音，可能需要使用时钟单位：

```rtsp
     C->S: PLAY rtsp://audio.example.com/meeting.en RTSP/1.0
           CSeq: 835
           Session: 12345678
           Range: clock=19961108T142300Z-19961108T143520Z

     S->C: RTSP/1.0 200 OK
           CSeq: 835
           Date: 23 Jan 1997 15:35:06 GMT
```

仅支持播放的媒体服务器必须支持 npt 格式，并且可以支持时钟和 smpte 格式。

### 10.6 PAUSE

`PAUSE` 请求会导致流传输暂时中断（停止）。如果请求 URL 命名了一个流，则只会暂停该流的播放和记录。例如，对于音频，这相当于静音。如果请求 URL 为一个演示或一组流命名，则该演示或组中所有当前活动流的传递将停止。恢复播放或录制后，必须保持曲目同步。任何服务器资源都被保留，尽管服务器可以在暂停了 `SETUP` 消息中会话头的超时参数指定的持续时间后关闭会话并释放资源。例如：

```rtsp
     C->S: PAUSE rtsp://example.com/fizzle/foo RTSP/1.0
           CSeq: 834
           Session: 12345678

     S->C: RTSP/1.0 200 OK
           CSeq: 834
           Date: 23 Jan 1997 15:35:06 GMT
```

`PAUSE` 请求可能包含一个 Range 标头，指定何时停止流或演示。我们将这一点称为“暂停点”。标头必须只包含一个值而不是时间范围。流的正常播放时间设置为暂停点。暂停请求在服务器第一次遇到任何当前挂起的 `PLAY` 请求中指定的时间点时生效。如果 Range 标头指定了任何当前挂起的 `PLAY` 请求之外的时间，则返回错误“457 Invalid Range”。如果媒体单元（例如音频或视频帧）恰好在暂停点开始呈现，则不会播放或录制。如果缺少 Range 标头，则在收到消息时立即中断流传递，并将暂停点设置为当前正常播放时间。

`PAUSE` 请求会丢弃所有排队的 `PLAY` 请求。然而，必须保持媒体流中的暂停点。没有 Range 标头的后续 `PLAY` 请求从暂停点恢复。

例如，如果服务器有 10 到 15 和 20 到 29 范围的播放请求待处理，然后收到 NPT 21 的暂停请求，它将开始播放第二个范围并在 NPT 21 处停止。如果暂停请求是针对 NPT 12 的并且服务器正在 NPT 13 处播放第一个播放请求，服务器立即停止。如果暂停请求是针对 NPT 16，则服务器在完成第一个播放请求后停止并丢弃第二个播放请求。

再举一个例子，如果服务器收到了播放范围 10 到 15 然后是 13 到 20（即重叠范围）的请求，则 NPT=14 的 `PAUSE` 请求将在服务器播放第一个范围时生效，第二个 `PLAY` 请求实际上被忽略，假设 `PAUSE` 请求在服务器开始播放第二个重叠范围之前到达。 无论 `PAUSE` 请求何时到达，它都会将 NPT 设置为 14。

如果服务器已经在 Range 标头中指定的时间之外发送了数据，则 `PLAY` 仍会在该时间点恢复，因为假定客户端在该时间点之后丢弃了数据。 这确保了无间隙的连续暂停/播放循环。

### 10.7 TEARDOWN

`TEARDOWN` 请求停止给定 URI 的流传递，释放与之关联的资源。 如果 URI 是此演示的演示 URI，则与该会话关联的任何 RTSP 会话标识符都不再有效。 除非所有传输参数都由会话描述定义，否则必须在会话再次播放之前发出 `SETUP` 请求。例如：

```rtsp
     C->S: TEARDOWN rtsp://example.com/fizzle/foo RTSP/1.0
           CSeq: 892
           Session: 12345678
     S->C: RTSP/1.0 200 OK
           CSeq: 892
```

### 10.8 GET_PARAMETER

`GET_PARAMETER` 请求检索 URI 中指定的演示或流的参数值。 回复和响应的内容留给实现。 没有实体主体的 `GET_PARAMETER` 可用于测试客户端或服务器的活跃度（“ping”）。例如：

```rtsp
     S->C: GET_PARAMETER rtsp://example.com/fizzle/foo RTSP/1.0
           CSeq: 431
           Content-Type: text/parameters
           Session: 12345678
           Content-Length: 15

           packets_received
           jitter

     C->S: RTSP/1.0 200 OK
           CSeq: 431
           Content-Length: 46
           Content-Type: text/parameters

           packets_received: 10
           jitter: 0.3838
```

“text/parameters”部分只是参数的示例类型。 这种方法是故意松散定义的，目的是在进一步实验后定义回复内容和响应内容。

### 10.9 SET_PARAMETER

此方法请求为由 URI 指定的演示或流设置参数值。

一个请求应该只包含一个参数，以允许客户端确定特定请求失败的原因。如果请求包含多个参数，则服务器必须仅在所有参数都可以成功设置的情况下对请求进行操作。服务器必须允许将参数重复设置为相同的值，但它可以禁止更改参数值。

> 注意：媒体流的传输参数必须只能用 SETUP 命令设置。限制将传输参数设置为 SETUP 是为了防火墙的利益。

参数以细粒度的方式拆分，以便可以有更有意义的错误指示。但是，如果需要原子设置，则允许设置多个参数可能是有意义的。想象一下设备控制，客户端不希望摄像机平移，除非它也可以同时倾斜到正确的角度。例如：

```rtsp
     C->S: SET_PARAMETER rtsp://example.com/fizzle/foo RTSP/1.0
           CSeq: 421
           Content-length: 20
           Content-type: text/parameters

           barparam: barstuff

     S->C: RTSP/1.0 451 Invalid Parameter
           CSeq: 421
           Content-length: 10
           Content-type: text/parameters

           barparam
```

“text/parameters”部分只是参数的示例类型。 这种方法是故意松散定义的，目的是在进一步实验后定义回复内容和响应内容。

### 10.10 REDIRECT

重定向请求通知客户端它必须连接到另一个服务器位置。 **它包含必需的标头 `Location`**，它指示客户端应该发出对该 URL 的请求。 它可能包含参数 Range，该参数指示重定向何时生效。 如果客户端想继续为这个 URI 发送或接收媒体，客户端必须在指定的主机上为当前会话发出一个 `TEARDOWN` 请求和一个新会话的 `SETUP` 请求。

此示例请求在给定的播放时间将此 URI 的流量重定向到新服务器：

```rtsp
     S->C: REDIRECT rtsp://example.com/fizzle/foo RTSP/1.0
           CSeq: 732
           Location: rtsp://bigserver.com:8001
           Range: clock=19960213T143205Z-
```

### 10.11 RECORD

该方法根据呈现描述开始记录一系列媒体数据。 时间戳反映开始和结束时间 (UTC)。 如果未指定时间范围，请使用演示说明中提供的开始或结束时间。 如果会话已经开始，请立即开始录制。

服务器决定是否将记录的数据存储在 request-URI 或另一个 URI 下。 如果服务器不使用请求 URI，则响应应该是 `201`（已创建）并包含一个描述请求状态和引用新资源的实体，以及一个位置头。

支持录制现场演示的媒体服务器必须支持时钟范围格式； smpte 格式没有意义。

在该示例中，媒体服务器先前被邀请参加所指示的会议。

```rtsp
     C->S: RECORD rtsp://example.com/meeting/audio.en RTSP/1.0
           CSeq: 954
           Session: 12345678
           Conference: 128.16.64.19/32492374
```

### 10.12 嵌入（交错）二进制数据

某些防火墙设计和其他情况可能会强制服务器交错 RTSP 方法和流数据。除非必要，否则通常应避免这种交织，因为它使客户端和服务器操作复杂化并增加了额外的开销。仅当 RTSP 通过 TCP 传输时，才应使用交错二进制数据。

RTP 数据包等流数据由 ASCII 美元符号（24 位十六进制）封装，后跟一字节通道标识符，后跟封装的二进制数据的长度，以网络字节顺序为二进制、两字节整数。流数据紧随其后，没有 CRLF，但包括上层协议头。每个 $ 块正好包含一个上层协议数据单元，例如一个 RTP 数据包。

信道标识符在传输报头中用交错参数定义（第 12.39 节）。

当传输选择是 RTP 时，RTCP 消息也由服务器通过 TCP 连接交错。默认情况下，RTCP 数据包在高于 RTP 通道的第一个可用通道上发送。客户端可以在另一个通道上显式地请求 RTCP 数据包。这是通过在传输头的交错参数中指定两个通道来完成的（第 12.39 节）。

> 当两个或多个流以这种方式交错时，需要 RTCP 进行同步。 此外，这提供了一种方便的方法，可以在网络配置需要时通过 TCP 控制连接隧道传输 RTP/RTCP 数据包，并在可能的情况下将它们传输到 UDP。

```rtsp
     C->S: SETUP rtsp://foo.com/bar.file RTSP/1.0
           CSeq: 2
           Transport: RTP/AVP/TCP;interleaved=0-1

     S->C: RTSP/1.0 200 OK
           CSeq: 2
           Date: 05 Jun 1997 18:57:18 GMT
           Transport: RTP/AVP/TCP;interleaved=0-1
           Session: 12345678

     C->S: PLAY rtsp://foo.com/bar.file RTSP/1.0
           CSeq: 3
           Session: 12345678

     S->C: RTSP/1.0 200 OK
           CSeq: 3
           Session: 12345678
           Date: 05 Jun 1997 18:59:15 GMT
           RTP-Info: url=rtsp://foo.com/bar.file;
             seq=232433;rtptime=972948234

     S->C: $\000{2 byte length}{"length" bytes data, w/RTP header}
     S->C: $\000{2 byte length}{"length" bytes data, w/RTP header}
     S->C: $\001{2 byte length}{"length" bytes  RTCP packet}
```

## 11 状态代码定义

在适用的情况下，会重复使用 HTTP 状态 [[H10]](https://datatracker.ietf.org/doc/html/rfc2068#section-10) 代码。 具有相同含义的状态码在此不再赘述。 有关哪些请求可能返回哪些状态代码的列表，请参见表 1。

### 11.1 Success 2xx

#### 11.1.1 250 Low on Storage Space

服务器在收到由于存储空间不足而可能无法完全完成的 RECORD 请求后返回此警告。 如果可能，服务器应该使用 Range 标头来指示它仍然可以记录的时间段。 由于服务器上的其他进程可能同时消耗存储空间，因此客户端应仅将其作为估计值。

### 11.2 Redirection 3xx

详见[[H10.3]](https://datatracker.ietf.org/doc/html/rfc2068#section-10.3)。

在 RTSP 中，重定向可用于负载平衡或将流请求重定向到拓扑更靠近客户端的服务器。 确定拓扑接近度的机制超出了本规范的范围。

### 11.3 Client Error 4xx

* `405 Method Not Allowed`
  
  请求 URI 所标识的资源不允许使用请求中指定的方法。 响应必须包含一个 Allow 头，其中包含所请求资源的有效方法列表。 如果请求尝试使用在 `SETUP` 期间未指明的方法，也将使用此状态代码，例如，如果即使传输头中的模式参数仅指定了 `PLAY`，也会发出 `RECORD` 请求。

* `451 Parameter Not Understood`

  请求的接收者不支持请求中包含的一个或多个参数。

* `452 Conference Not Found`

  媒体服务器不知道会议头域指示的会议。

* `453 Not Enough Bandwidth`

  由于带宽不足，请求被拒绝。 例如，这可能是资源预留失败的结果。

* `454 Session Not Found`

  会话标头中的 RTSP 会话标识符丢失、无效或已超时。

* `455 Method Not Valid in This State`

  客户端或服务器无法在当前状态下处理此请求。 响应应该包含一个 Allow 标头以使错误恢复更容易。

* `456 Header Field Not Valid for Resource`

  服务器无法处理所需的请求标头。 例如，如果 `PLAY` 包含 Range 头字段，但流不允许查找。

* `457 Invalid Range`

  给定的范围值超出范围，例如，超出了演示的结尾。
  
* 458 Parameter Is Read-Only

  `SET_PARAMETER` 设置的参数可以读取但不能修改。

* 459 Aggregate Operation Not Allowed
  
  所请求的方法可能不适用于相关 URL，因为它是一个聚合（表示）URL。该方法可以应用于流 URL。

* 460 Only Aggregate Operation Allowed

  所请求的方法可能不适用于相关 URL，因为它不是聚合（表示）URL。该方法可以应用于演示 URL。

* 461 Unsupported Transport

  传输字段不包含受支持的传输规范。

* 462 Destination Unreachable

  由于无法到达客户端地址，无法建立数据传输通道。此错误很可能是客户端尝试在传输字段中放置无效目标参数的结果。

* 551 Option not supported

  不支持在 Require 或 Proxy-Require 字段中给出的选项。应返回 Unsupported 标头，说明不支持的选项。

## 12 头部字段定义

HTTP/1.1 [2] 或其他未在此处列出的非标准标头字段目前没有明确定义的含义，接收方应忽略。表 3 总结了 RTSP 使用的报头字段。

在 “type” 列中：

* “g” 指定在请求和响应中都能找到的通用请求头，
* “R” 指定请求头，
* “r” 指定响应头，
* “e” 指定实体头字段。

在 “support” 列中标记为 “req.” 的字段必须由接收者为特定方法实现，而标记为 “opt.” 的字段是可选的。

> 请注意，并非所有标记为 “req.” 的字段都会在这种类型的每个请求中发送。 “req.”意味着只有客户端（对于响应头）和服务器（对于请求头）必须实现这些字段。

最后一列列出了此头部字段有意义的方法；名称“entity”是指返回消息正文的所有方法。在本规范中，`DESCRIBE` 和 `GET_PARAMETER` 属于此类。

   | Header             | type | support | methods                   |
   | ------------------ | ---- | ------- | ------------------------- |
   | Accept             | R    | opt.    | entity                    |
   | Accept-Encoding    | R    | opt.    | entity                    |
   | Accept-Language    | R    | opt.    | all                       |
   | Allow              | r    | opt.    | all                       |
   | Authorization      | R    | opt.    | all                       |
   | Bandwidth          | R    | opt.    | all                       |
   | Blocksize          | R    | opt.    | all but OPTIONS, TEARDOWN |
   | Cache-Control      | g    | opt.    | SETUP                     |
   | Conference         | R    | opt.    | SETUP                     |
   | Connection         | g    | req.    | all                       |
   | Content-Base       | e    | opt.    | entity                    |
   | Content-Encoding   | e    | req.    | SET_PARAMETER             |
   | Content-Encoding   | e    | req.    | DESCRIBE, ANNOUNCE        |
   | Content-Language   | e    | req.    | DESCRIBE, ANNOUNCE        |
   | Content-Length     | e    | req.    | SET_PARAMETER, ANNOUNCE   |
   | Content-Length     | e    | req.    | entity                    |
   | Content-Location   | e    | opt.    | entity                    |
   | Content-Type       | e    | req.    | SET_PARAMETER, ANNOUNCE   |
   | Content-Type       | r    | req.    | entity                    |
   | CSeq               | g    | req.    | all                       |
   | Date               | g    | opt.    | all                       |
   | Expires            | e    | opt.    | DESCRIBE, ANNOUNCE        |
   | From               | R    | opt.    | all                       |
   | If-Modified-Since  | R    | opt.    | DESCRIBE, SETUP           |
   | Last-Modified      | e    | opt.    | entity                    |
   | Proxy-Authenticate |      |         |
   | Proxy-Require      | R    | req.    | all                       |
   | Public             | r    | opt.    | all                       |
   | Range              | R    | opt.    | PLAY, PAUSE, RECORD       |
   | Range              | r    | opt.    | PLAY, PAUSE, RECORD       |
   | Referer            | R    | opt.    | all                       |
   | Require            | R    | req.    | all                       |
   | Retry-After        | r    | opt.    | all                       |
   | RTP-Info           | r    | req.    | PLAY                      |
   | Scale              | Rr   | opt.    | PLAY, RECORD              |
   | Session            | Rr   | req.    | all but SETUP, OPTIONS    |
   | Server             | r    | opt.    | all                       |
   | Speed              | Rr   | opt.    | PLAY                      |
   | Transport          | Rr   | req.    | SETUP                     |
   | Unsupported        | r    | req.    | all                       |
   | User-Agent         | R    | opt.    | all                       |
   | Via                | g    | opt.    | all                       |
   | WWW-Authenticate   | r    | opt.    | all                       |

   表3： RTSP 头域概述

请求头详见[RFC 2326](https://datatracker.ietf.org/doc/html/rfc2326#section-12)

## 13 缓存

在 HTTP 中，响应-请求对被缓存。 RTSP 在这方面有很大的不同。响应不可缓存，但 `DESCRIBE` 返回的或包含在 `ANNOUNCE` 中的演示描述除外。 （因为除了 `DESCRIBE` 和 `GET_PARAMETER` 之外的任何响应都不会返回任何数据，缓存对于这些请求来说并不是真正的问题。）但是，对于连续媒体数据来说是可取的，通常相对于 RTSP 在带外传送，要缓存，以及会话描述。

在接收到 `SETUP` 或 `PLAY` 请求时，代理确定它是否具有连续媒体内容及其描述的最新副本。它可以通过分别发出 `SETUP` 或 `DESCRIBE` 请求并将 Last-Modified 标头与缓存副本的标头进行比较来确定副本是否是最新的。如果副本不是最新的，它会根据需要修改 SETUP 传输参数并将请求转发到源服务器。随后的控制命令如 `PLAY` 或 `PAUSE` 然后未经修改地传递代理。代理将连续媒体数据传送给客户端，同时可能制作本地副本供以后重用。缓存允许的确切行为由第 12.8 节中描述的缓存响应指令给出。如果缓存当前正在向请求者提供流，则它必须回答任何 DESCRIBE 请求，因为流描述的低级细节可能在源服务器上已更改。

> 请注意，RTSP 缓存与 HTTP 缓存不同，属于“直通”类型。缓存不是从源服务器检索整个资源，而是简单地复制流数据，因为它在流向客户端的途中经过。因此，它不会引入额外的延迟。

对于客户端来说，RTSP 代理缓存就像一个普通的媒体服务器，对于媒体源服务器来说就像一个客户端。正如 HTTP 缓存必须为它缓存的对象存储内容类型、内容语言等，媒体缓存必须存储表示描述。通常，缓存会从表示描述中消除所有传输引用（即多播信息），因为这些与从缓存到客户端的数据传递无关。有关编码的信息保持不变。如果缓存能够翻译缓存的媒体数据，它将创建一个新的演示描述，其中包含它可以提供的所有编码可能性。

## 14 示例

以下示例涉及非标准的流描述格式，例如 RTSL。 以下示例不能用作这些格式的参考。

### 14.1 媒体点播（单播）

客户端 C 从媒体服务器 A (audio.example.com) 和 V (video.example.com) 请求电影。 媒体描述存储在网络服务器 W 上。 媒体描述包含对演示及其所有流的描述，包括可用的编解码器、动态 RTP 有效负载类型、协议栈以及语言或版权限制等内容信息。 它还可以给出关于电影时间线的指示。

在这个例子中，客户端只对电影的最后部分感兴趣。

```rtsp
     C->W: GET /twister.sdp HTTP/1.1
           Host: www.example.com
           Accept: application/sdp

     W->C: HTTP/1.0 200 OK
           Content-Type: application/sdp
           v=0
           o=- 2890844526 2890842807 IN IP4 192.16.24.202
           s=RTSP Session
           m=audio 0 RTP/AVP 0
           a=control:rtsp://audio.example.com/twister/audio.en
           m=video 0 RTP/AVP 31
           a=control:rtsp://video.example.com/twister/video

     C->A: SETUP rtsp://audio.example.com/twister/audio.en RTSP/1.0
           CSeq: 1
           Transport: RTP/AVP/UDP;unicast;client_port=3056-3057

     A->C: RTSP/1.0 200 OK
           CSeq: 1
           Session: 12345678
           Transport: RTP/AVP/UDP;unicast;client_port=3056-3057;
                      server_port=5000-5001

     C->V: SETUP rtsp://video.example.com/twister/video RTSP/1.0
           CSeq: 1
           Transport: RTP/AVP/UDP;unicast;client_port=3058-3059

     V->C: RTSP/1.0 200 OK
           CSeq: 1
           Session: 23456789
           Transport: RTP/AVP/UDP;unicast;client_port=3058-3059;
                      server_port=5002-5003

     C->V: PLAY rtsp://video.example.com/twister/video RTSP/1.0
           CSeq: 2
           Session: 23456789
           Range: smpte=0:10:00-

     V->C: RTSP/1.0 200 OK
           CSeq: 2
           Session: 23456789
           Range: smpte=0:10:00-0:20:00
           RTP-Info: url=rtsp://video.example.com/twister/video;
             seq=12312232;rtptime=78712811

     C->A: PLAY rtsp://audio.example.com/twister/audio.en RTSP/1.0
           CSeq: 2
           Session: 12345678
           Range: smpte=0:10:00-

     A->C: RTSP/1.0 200 OK
           CSeq: 2
           Session: 12345678
           Range: smpte=0:10:00-0:20:00
           RTP-Info: url=rtsp://audio.example.com/twister/audio.en;
             seq=876655;rtptime=1032181

     C->A: TEARDOWN rtsp://audio.example.com/twister/audio.en RTSP/1.0
           CSeq: 3
           Session: 12345678

     A->C: RTSP/1.0 200 OK
           CSeq: 3

     C->V: TEARDOWN rtsp://video.example.com/twister/video RTSP/1.0
           CSeq: 3
           Session: 23456789

     V->C: RTSP/1.0 200 OK
           CSeq: 3
```

即使音频和视频轨道在两个不同的服务器上，并且可能在稍微不同的时间开始并且可能相互漂移，客户端也可以使用标准 RTP 方法同步两者，特别是 RTCP 发送方中包含的时间尺度 报告。

### 14.2 容器文件的流式传输

出于本示例的目的，容器文件是一个存储实体，其中存在属于同一最终用户呈现的多个连续媒体类型。实际上，容器文件表示一个 RTSP 演示，其每个组件都是 RTSP 流。容器文件是一种广泛使用的存储此类演示的方式。当组件作为独立的流传输时，希望在服务器端为这些流维护一个公共上下文。

> 这使服务器能够轻松保持单个存储手柄打开。它还允许在服务器对流进行任何优先级排序的情况下平等地处理所有流。

演示作者也可能希望防止客户端对流进行选择性检索，以保留组合媒体演示的艺术效果。类似地，在这种紧密绑定的表示中，希望能够通过使用聚合 URL 的单个控制消息来控制所有流。

以下是使用单个 RTSP 会话控制多个流的示例。它还说明了聚合 URL 的使用。客户端 C 向媒体服务器 M 请求演示。 影片存储在容器文件中。 客户端已获得容器文件的 RTSP URL。

```rtsp
     C->M: DESCRIBE rtsp://foo/twister RTSP/1.0
           CSeq: 1

     M->C: RTSP/1.0 200 OK
           CSeq: 1
           Content-Type: application/sdp
           Content-Length: 164

           v=0
           o=- 2890844256 2890842807 IN IP4 172.16.2.93
           s=RTSP Session
           i=An Example of RTSP Session Usage
           a=control:rtsp://foo/twister
           t=0 0
           m=audio 0 RTP/AVP 0
           a=control:rtsp://foo/twister/audio
           m=video 0 RTP/AVP 26
           a=control:rtsp://foo/twister/video

     C->M: SETUP rtsp://foo/twister/audio RTSP/1.0
           CSeq: 2
           Transport: RTP/AVP;unicast;client_port=8000-8001

     M->C: RTSP/1.0 200 OK
           CSeq: 2
           Transport: RTP/AVP;unicast;client_port=8000-8001;
                      server_port=9000-9001
           Session: 12345678

     C->M: SETUP rtsp://foo/twister/video RTSP/1.0
           CSeq: 3
           Transport: RTP/AVP;unicast;client_port=8002-8003
           Session: 12345678

     M->C: RTSP/1.0 200 OK
           CSeq: 3
           Transport: RTP/AVP;unicast;client_port=8002-8003;
                      server_port=9004-9005
           Session: 12345678

     C->M: PLAY rtsp://foo/twister RTSP/1.0
           CSeq: 4
           Range: npt=0-
           Session: 12345678
     M->C: RTSP/1.0 200 OK
           CSeq: 4
           Session: 12345678
           RTP-Info: url=rtsp://foo/twister/video;
             seq=9810092;rtptime=3450012

     C->M: PAUSE rtsp://foo/twister/video RTSP/1.0
           CSeq: 5
           Session: 12345678

     M->C: RTSP/1.0 460 Only aggregate operation allowed
           CSeq: 5

     C->M: PAUSE rtsp://foo/twister RTSP/1.0
           CSeq: 6
           Session: 12345678

     M->C: RTSP/1.0 200 OK
           CSeq: 6
           Session: 12345678

     C->M: SETUP rtsp://foo/twister RTSP/1.0
           CSeq: 7
           Transport: RTP/AVP;unicast;client_port=10000

     M->C: RTSP/1.0 459 Aggregate operation not allowed
           CSeq: 7
```

在第一个失败实例中，客户端尝试暂停演示的一个流（在本例中为视频）。 对于该演示，服务器不允许这样做。 在第二种情况下，聚合 URL 可能不用于 `SETUP`，并且每个流需要一个控制消息来设置传输参数。

> 这使传输标头的语法保持简单，并允许防火墙轻松解析传输信息。

### 14.3 单流容器文件

一些 RTSP 服务器可能会将所有文件视为“容器文件”，而其他服务器可能不支持这样的概念。 因此，客户端应该使用会话描述中为请求 URL 设置的规则，而不是假设始终使用一致的 URL。 下面是一个多流服务器如何期望提供单流文件的示例：

```rtsp
          Accept: application/x-rtsp-mh, application/sdp
          CSeq: 1

    S->C  RTSP/1.0 200 OK
          CSeq: 1
          Content-base: rtsp://foo.com/test.wav/
          Content-type: application/sdp
          Content-length: 48

          v=0
          o=- 872653257 872653257 IN IP4 172.16.2.187
          s=mu-law wave file
          i=audio test
          t=0 0
          m=audio 0 RTP/AVP 0
          a=control:streamid=0

    C->S  SETUP rtsp://foo.com/test.wav/streamid=0 RTSP/1.0
          Transport: RTP/AVP/UDP;unicast;
                     client_port=6970-6971;mode=play
          CSeq: 2

    S->C  RTSP/1.0 200 OK
          Transport: RTP/AVP/UDP;unicast;client_port=6970-6971;
                     server_port=6970-6971;mode=play
          CSeq: 2
          Session: 2034820394

    C->S  PLAY rtsp://foo.com/test.wav RTSP/1.0
          CSeq: 3
          Session: 2034820394

    S->C  RTSP/1.0 200 OK
          CSeq: 3
          Session: 2034820394
          RTP-Info: url=rtsp://foo.com/test.wav/streamid=0;
            seq=981888;rtptime=3781123
```

注意 `SETUP` 命令中的不同 URL，然后切换回 `PLAY` 命令中的聚合 URL。 当有多个流具有聚合控制时，这完全有意义，但在流数为 1 的特殊情况下不太直观。

在这种特殊情况下，建议服务器原谅发送以下内容的实现：

```rtsp
    C->S  PLAY rtsp://foo.com/test.wav/streamid=0 RTSP/1.0
          CSeq: 3
```

在最坏的情况下，服务器应该发回：

```rtsp
    S->C  RTSP/1.0 460 Only aggregate operation allowed
          CSeq: 3
```

人们还希望服务器实现也能接受以下内容：

```rtsp
    C->S  SETUP rtsp://foo.com/test.wav RTSP/1.0
          Transport: rtp/avp/udp;client_port=6970-6971;mode=play
          CSeq: 2
```

由于此文件中只有一个流，因此这意味着什么并不含糊。

### 14.4 使用多播的现场媒体演示

媒体服务器 M 选择多播地址和端口。 这里，我们假设网络服务器只包含一个指向完整描述的指针，而媒体服务器 M 维护完整描述。

```rtsp
     C->W: GET /concert.sdp HTTP/1.1
           Host: www.example.com

     W->C: HTTP/1.1 200 OK
           Content-Type: application/x-rtsl

           <session>
             <track src="rtsp://live.example.com/concert/audio">
           </session>

     C->M: DESCRIBE rtsp://live.example.com/concert/audio RTSP/1.0
           CSeq: 1

     M->C: RTSP/1.0 200 OK
           CSeq: 1
           Content-Type: application/sdp
           Content-Length: 44

           v=0
           o=- 2890844526 2890842807 IN IP4 192.16.24.202
           s=RTSP Session
           m=audio 3456 RTP/AVP 0
           a=control:rtsp://live.example.com/concert/audio
           c=IN IP4 224.2.0.1/16

     C->M: SETUP rtsp://live.example.com/concert/audio RTSP/1.0
           CSeq: 2
           Transport: RTP/AVP;multicast

     M->C: RTSP/1.0 200 OK
           CSeq: 2
           Transport: RTP/AVP;multicast;destination=224.2.0.1;
                      port=3456-3457;ttl=16
           Session: 0456804596

     C->M: PLAY rtsp://live.example.com/concert/audio RTSP/1.0
           CSeq: 3
           Session: 0456804596

     M->C: RTSP/1.0 200 OK
           CSeq: 3
           Session: 0456804596
```

### 14.5 在现有会话中播放媒体

会议参与者 C 希望媒体服务器 M 将演示磁带回放到现有会议中。 C 向媒体服务器表明网络地址和加密密钥已由会议提供，因此不应由服务器选择。 该示例省略了简单的 ACK 响应。

```rtsp
     C->M: DESCRIBE rtsp://server.example.com/demo/548/sound RTSP/1.0
           CSeq: 1
           Accept: application/sdp

     M->C: RTSP/1.0 200 1 OK
           Content-type: application/sdp
           Content-Length: 44

           v=0
           o=- 2890844526 2890842807 IN IP4 192.16.24.202
           s=RTSP Session
           i=See above
           t=0 0
           m=audio 0 RTP/AVP 0

     C->M: SETUP rtsp://server.example.com/demo/548/sound RTSP/1.0
           CSeq: 2
           Transport: RTP/AVP;multicast;destination=225.219.201.15;
                      port=7000-7001;ttl=127
           Conference: 199702170042.SAA08642@obiwan.arl.wustl.edu%20Starr

     M->C: RTSP/1.0 200 OK
           CSeq: 2
           Transport: RTP/AVP;multicast;destination=225.219.201.15;
                      port=7000-7001;ttl=127
           Session: 91389234234
           Conference: 199702170042.SAA08642@obiwan.arl.wustl.edu%20Starr

     C->M: PLAY rtsp://server.example.com/demo/548/sound RTSP/1.0
           CSeq: 3
           Session: 91389234234

     M->C: RTSP/1.0 200 OK
           CSeq: 3
```

### 14.6 录像

会议参与者客户端C要求媒体服务器M记录会议的音频和视频部分。 客户端使用 `ANNOUNCE` 方法向服务器提供有关记录会话的元信息。

```rtsp
     C->M: ANNOUNCE rtsp://server.example.com/meeting RTSP/1.0
           CSeq: 90
           Content-Type: application/sdp
           Content-Length: 121

           v=0
           o=camera1 3080117314 3080118787 IN IP4 195.27.192.36
           s=IETF Meeting, Munich - 1
           i=The thirty-ninth IETF meeting will be held in Munich, Germany
           u=http://www.ietf.org/meetings/Munich.html
           e=IETF Channel 1 <ietf39-mbone@uni-koeln.de>
           p=IETF Channel 1 +49-172-2312 451
           c=IN IP4 224.0.1.11/127
           t=3080271600 3080703600
           a=tool:sdr v2.4a6
           a=type:test
           m=audio 21010 RTP/AVP 5
           c=IN IP4 224.0.1.11/127
           a=ptime:40
           m=video 61010 RTP/AVP 31
           c=IN IP4 224.0.1.12/127

     M->C: RTSP/1.0 200 OK
           CSeq: 90

     C->M: SETUP rtsp://server.example.com/meeting/audiotrack RTSP/1.0
           CSeq: 91
           Transport: RTP/AVP;multicast;destination=224.0.1.11;
                      port=21010-21011;mode=record;ttl=127
     M->C: RTSP/1.0 200 OK
           CSeq: 91
           Session: 50887676
           Transport: RTP/AVP;multicast;destination=224.0.1.11;
                      port=21010-21011;mode=record;ttl=127

     C->M: SETUP rtsp://server.example.com/meeting/videotrack RTSP/1.0
           CSeq: 92
           Session: 50887676
           Transport: RTP/AVP;multicast;destination=224.0.1.12;
                      port=61010-61011;mode=record;ttl=127

     M->C: RTSP/1.0 200 OK
           CSeq: 92
           Transport: RTP/AVP;multicast;destination=224.0.1.12;
                      port=61010-61011;mode=record;ttl=127

     C->M: RECORD rtsp://server.example.com/meeting RTSP/1.0
           CSeq: 93
           Session: 50887676
           Range: clock=19961110T1925-19961110T2015

     M->C: RTSP/1.0 200 OK
           CSeq: 93
```

## 15 语法

详见[RFC 2326](https://datatracker.ietf.org/doc/html/rfc2326#section-15)

## 16 安全注意事项

详见[RFC 2326](https://datatracker.ietf.org/doc/html/rfc2326#section-16)

## 附录 A：RTSP 协议状态机

RTSP 客户端和服务器状态机描述了从 RTSP 会话初始化到 RTSP 会话终止的协议行为。

状态是基于每个对象定义的。 对象由流 URL 和 RTSP 会话标识符唯一标识。 任何使用聚合 URL 表示由多个流组成的 RTSP 演示的请求/回复都将对所有流的各个状态产生影响。 例如，如果演示 /movie 包含两个流，/movie/audio 和 /movie/video，则以下命令：

```rtsp
     PLAY rtsp://foo.com/movie RTSP/1.0
     CSeq: 559
     Session: 12345678
```

将对 /movie/audio 和 /movie/video 的状态产生影响。

> 这个例子并不意味着用标准的方式来表示 URL 中的流或与文件系统的关系。 见第 3.2 节。

请求 OPTIONS、ANNOUNCE、DESCRIBE、GET_PARAMETER、SET_PARAMETER 对客户端或服务器状态没有任何影响，因此未在状态表中列出。

### A.1 客户端状态机

客户端可以假设以下状态：

* `Init`:  
  `SETUP` 已发送，等待回复。
* `Ready`:  
  在播放状态下收到 `SETUP` 回复或 `PAUSE` 回复。
* `Playing`:  
  收到 `PLAY` 回复
* `Recording`:  
  收到 `RECORD` 回复

通常，客户端在收到对请求的回复时更改状态。 请注意，某些请求在未来的时间或位置（例如 `PAUSE`）有效，并且状态也会相应更改。 如果对象不需要明确的 `SETUP`（例如，它可以通过多播组获得），则状态从 `Ready` 开始。 在这种情况下，只有两种状态，`Ready` 和 `Playing`。 当达到请求范围的末尾时，客户端还会将状态从“`Playing`/`Recording`”更改为“`Ready`”。

“下一个状态”列指示在收到成功响应 (2xx) 后假定的状态。 如果请求产生 3xx 的状态码，则状态变为 `Init`，而 4xx 的状态码不会产生状态变化。 未为每个状态列出的消息不得由处于该状态的客户端发出，但不影响状态的消息除外，如上所列。 从服务器接收 `REDIRECT` 相当于从服务器接收 3xx 重定向状态。

   | state     | message sent | next state after response     |
   | --------- | ------------ | ----------------------------- |
   | Init      | SETUP        | Ready                         |
   |           | TEARDOWN     | Init                          |
   | Ready     | PLAY         | Playing                       |
   |           | RECORD       | Recording                     |
   |           | TEARDOWN     | Init                          |
   |           | SETUP        | Ready                         |
   | Playing   | PAUSE        | Ready                         |
   |           | TEARDOWN     | Init                          |
   |           | PLAY         | Playing                       |
   |           | SETUP        | Playing (changed transport)   |
   | Recording | PAUSE        | Ready                         |
   |           | TEARDOWN     | Init                          |
   |           | RECORD       | Recording                     |
   |           | SETUP        | Recording (changed transport) |

### A.2 服务器状态机

服务器可以呈现以下状态：

* `Init`:  
  初始状态，尚未收到有效的 `SETUP`。
* `Ready`:  
  上次 `SETUP` 接收成功，发送回复或播放后，上次接收 `PAUSE` 成功，回复发送。
* `Playing`:  
  最后 `PLAY` 接收成功，回复已发送。 正在发送数据。
* `Recording`:  
  服务器正在记录媒体数据。

通常，服务器会在收到请求时更改状态。 如果服务器处于播放或录制状态并且处于单播模式，如果它在定义的时间间隔内没有从客户端收到“健康”信息，例如 RTCP 报告或 RTSP 命令，它可以恢复到 `Init` 并拆除 RTSP 会话 ，默认为一分钟。 服务器可以在 Session 响应头中声明另一个超时值（第 12.37 节）。 如果服务器处于就绪状态，并且在超过一分钟的时间间隔内没有收到 RTSP 请求，则它可以恢复到 `Init`。 请注意，某些请求（例如 `PAUSE`）可能会在未来的时间或位置生效，并且服务器状态会在适当的时间发生变化。 在客户端请求的范围结束时，服务器从播放或录制状态恢复到就绪状态。

`REDIRECT` 消息在发送时立即生效，除非它具有指定重定向何时生效的 Range 标头。 在这种情况下，服务器状态也会在适当的时候发生变化。

如果对象不需要明确的 `SETUP`，则状态从 `Ready` 开始，只有两种状态，`Ready` 和 `Playing`。

“下一个状态”列指示发送成功响应 (2xx) 后假定的状态。 如果请求导致状态代码为 3xx，则状态变为 Init。 状态代码 4xx 不会导致任何更改。

| state     | message received | next state |
| --------- | ---------------- | ---------- |
| Init      | SETUP            | Ready      |
|           | TEARDOWN         | Init       |
| Ready     | PLAY             | Playing    |
|           | SETUP            | Ready      |
|           | TEARDOWN         | Init       |
|           | RECORD           | Recording  |
| Playing   | PLAY             | Playing    |
|           | PAUSE            | Ready      |
|           | TEARDOWN         | Init       |
|           | SETUP            | Playing    |
| Recording | RECORD           | Recording  |
|           | PAUSE            | Ready      |
|           | TEARDOWN         | Init       |
|           | SETUP            | Recording  |

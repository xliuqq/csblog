# HTTP

HTTP：**H**yper **T**ext **T**ransfer **P**rotocol 

Header

- 空格编码是`%20`，但是application/x-www-form-urlencoded媒体类型会将空格编码为`+`号；



## HTTPS

HTTPS：**HTTP** over **S**ecure Socket Layer

- HTTP下加入SSL层，一个加密/身份验证层（在HTTP与TCP之间），端口443；
- SSL(Secure Sockets Layer 安全套接层)及其继任者传输层安全(Transport Layer Security，TLS)是为网络通信提供安全及数据完整性的一种安全协议；
- 采用https的服务器必须从CA（Certificate Authority）申请一个用于证明服务器用途类型的证书。该证书只有用于对应的服务器的时候，客户端才信任此主机。
- HTTPS的性能优化：http://mp.weixin.qq.com/s/V62VYS8KFNKxJxfzMYefrw



## HTTP/1.x

文本协议。

**解析**：HTTP header 各个 fields 使用 `\r\n` 分隔，然后跟 body 之间使用 `\r\n\r\n` 分隔。解析完 header 之后，我们才能从 header 里面的 `content-length` 拿到 body 的 size，从而读取 body。整个流程并不高效。

**交互**：一个连接每次只能一问一答。client需要跟server建立多条连接。

**推送**：HTTP/1.x没有推送机制，一般采用 轮询 或者 Web-socket方式。

## HTTP/2

二进制协议。

- Stream : 一个双向流，一条连接可以有多个streams；
- Message : 等价于request和response；
- Frame : 数据传输的最小单位。每个Frame都属于一个特定的stream或者整个连接。一个message可能由多个frame组成。

### Frame格式

格式如下：

```txt
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

Length : Frame的长度，默认最大长度是16KB，可以显示设置max frame size；

Type : Frame类型，如DATA, HEADERS, PRIORITY等；

Flag, R : 保留位；

Stream Identifier : 标志所属的stream，如果为0，则表示这个frame属于整条连接； 

Frame Payload : 不同Type有不同格式。

### 多路复用

HTTP/2的通过**stream支持连接的多路复用**，提高连接利用率。Stream的重要特性：

- 一条连接可以包含多个streams，多个streams发送的数据互相不影响；
- Stream可以被client和server单方面或者共享使用；
- Stream可以被任意一端关闭；
- Stream会确定好发送frame的顺序，另一端会按照接受到的顺序来处理；
- **Stream用一个唯一ID来标识**，client创建的是奇数，server创建的是偶数。

提高一个stream的并发，可以调大MAX_CONCURRENT_STREAMS。

一条连接通常只有一个线程来处理，所以并不能充分利用服务器多核的优势。同时，**每个请求编解码还是有开销的，所以用一条连接还是会出现瓶颈**。

### 流控

- Flow control 是单向的。Receiver 可以选择给 stream 或者整个连接设置 window size。
- Flow control 是基于信任的。Receiver 只是会给 sender 建议它的初始连接和 stream 的 flow control window size。
- Flow control 不可能被禁止掉。当 HTTP/2 连接建立起来之后，client 和 server 会交换 SETTINGS frames，用来设置 flow control window size。
- Flow control 是 hop-by-hop，并不是 end-to-end 的，也就是我们可以用一个中间人来进行 flow control。

HTTP/2 默认的 window size 是 64 KB。

### HPACK

无需发送重复的Header，提供静态和动态的table。



## Content-Type

> content-type 是  Http Header 中的一个字段，在此单独列出。



### application/octet-stream

> 二进制文件没有特定或已知的 subtype，即使用 `application/octet-stream`
>
> - 浏览器针对此类型的响应，会将其保存下载下来（配合 Content-Disposition）；
> - 如果是其它的客户端（比如 curl，Java），可以自行处理；



### text/event-stream

> 不应该设置 WriteTimeOut

webpack热更新需要向浏览器推送信息，一般都会想到websocket，但是还有一种方式，叫做**Server-Sent Events**（简称SSE）。

- SSE是 websocket 的一种轻型替代方案，其响应 Type 类型为 `text/event-stream`。

| sse                                     | websocket             |
| --------------------------------------- | --------------------- |
| http 协议                               | 独立的 websocket 协议 |
| 轻量，使用简单                          | 相对复杂              |
| 默认支持断线重连（JS 自带 EventSource） | 需要自己实现断线重连  |
| 文本传输                                | 二进制传输            |
| 支持自定义发送的消息类型                | -                     |



## Other Headers

### Transfer-Encoding

> 当包体长度未知或者我们想把包体拆成几个小块传输时（比如推送场景），即 Content-Length 长度未知时。

当服务器返回 `Transfer-Encoding: chunked` 时，表明此时服务器会对返回的包体进行 `chunked` 编码，每个 chunk 的格式如下所示：

```shell
${chunk-length}\r\n${chunk-data}\r\n
```

其中，`${chunk-length}` 表示 chunk 的字节长度，使用 16 进制表示，`${chunk-data}` 为 chunk 的内容。

当 chunk 都传输完，需要额外传输 `0\r\n\r\n` 表示结束。

- HTTP 的包已经对该格式进行解析和处理，应用端直接读取 Body，如下（假设每个chunk-data按 `\n`划分）

  ```go
  rep, _ := client.Get(url)
  reader := bufio.NewReader(rep.Body)
  defer rep.Body.Close()
  for {
      // handle chunked data
      data, _ := reader.ReadString('\n')
      log.Printf("receive: [%s]", data)
  }
  ```

  

### Range

如果请求一个资源时， HTTP响应中出现如下所示的 'Accept-Ranges'， 且其值不是none， 那么服务器支持范围请求。

```bash
# Response 的响应
curl -I http://i.imgur.com/z4d4kWk.jpg

HTTP/1.1 200 OK
...
Accept-Ranges: bytes
Content-Length: 577454964
Content-Range: bytes 0-577454963/577454964
```

在如上响应中，Accept-Ranges: bytes 代表可以使用字节作为单位来定义请求范围。这里的 Response Headers中的 Content-Length: 577454964则代表该资源的大小。

在处理HTTP Range 请求时，有三个相关的状态：

- 206 Partial Content——> HTTP Range 请求成功
- 416 Requested Range Not Satisfiable status.——> HTTP Range 请求超出界限
- 200 OK——> 不支持范围请求

#### 单个范围请求

```bash
curl http://i.imgur.com/z4d4kWk.jpg -i -H "Range: bytes=0-1023"

# HTTP/1.1 206 Partial Content Content-Range: bytes 0-1023/577454964 Content-Length: 1024 ... (binary content)
```

正常情况下 server 返回 206 部分内容响应，Content-Length表示的是现在请求的范围大小，而Content-Range则表示的是这部分消息在完整资源中的位置。

#### 多范围请求

```bash
curl http://www.example.com -i -H "Range: bytes=0-50, 100-150"
# HTTP/1.1 206 Partial Content 
# Content-Type: multipart/byteranges; boundary=3d6b6a416f9b5 
# Content-Length: 282 
# --3d6b6a416f9b5 
# Content-Type: video/mp4
# Content-Range: bytes 0-50/577454964
# ... binary data...
# --3d6b6a416f9b5 
# Content-Type: video/mp4
# Content-Range: bytes 100-150/577454964

# 多部分 byterange，每个部分包含自己的Content-Type 和 Content-Range
```

#### 条件范围请求

当继续请求更多资源时，你需要确保被存储的资源在上一帧收到后没有被改变。

- 如果条件得到满足，range请求将会被发出，server 发回带有适当正文的206 partial content 应答；
- 如果条件不满足则返回完整资源，并显示200 OK状态。

If-Range 头可以与Last-Modified 验证程序，或者与 ETag 一起使用。

> If-Range: Wed, 21 Oct 2015 07:28:00 GMT





## 错误码

| 错误码               | 信息                                             |
| -------------------- | ------------------------------------------------ |
| 206(Partial Content) | 部分资源                                         |
| 400(Bad  Request)    | 语法错误无法解读请求                             |
| 401(Unauthorized)    | 无权限访问资源，但身份认证后可以访问权限时       |
| 403(Forbidden)       | 不允许客户端获得资源的访问权限，即使通过身份认证 |
| 404(Not  found)      | 未找到资源                                       |
| 405(Not  Allowed)    | 资源不允许使用某个HTTP方法时                     |
| 406(Not  Acceptable) | 请求不可接受                                     |
| 409(Conflict)        | 请求与资源的当前状态有冲突时                     |
| 410(Gone)            | 资源以前存在，但今后不会存在                     |



## WebSocket

> 服务端可以通过检查请求的协议来区分是HTTP还是WebSocket的请求
>
> - Upgrade头部字段值为websocket，则表示客户端希望升级到WebSocket协议；

**单个TCP连接上进行全双工通信**的协议。

Websocket 通过HTTP 1.1 协议的101状态码进行握手。

在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。




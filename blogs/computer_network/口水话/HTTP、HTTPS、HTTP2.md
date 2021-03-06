---
HTTP、HTTPS、HTTP2 口水话
---

#### 目录

1. HTTP
2. HTTPS
3. HTTP2

#### HTTP

HTTP/1.0 在 1996 年发布，它增加了 HEAD、POST 等方法；增加了相应状态码，标记可能的错误原因；引入了协议版本号的概念；引入了 HTTP Header 的概念，让 HTTP 处理请求和响应更加灵活；传输的数据不再仅限于文本；

HTTP/1.0 已经有现代 HTTP 的雏形了，但是随着互联网的发展，各种图片、音视频格式的出现，HTTP/1.0 显然已不能满足需求，同时还有短连接造成的巨大性能损耗，没多久，HTTP/1.1 发布，它增加了 PUT、DELETE 等新方法；增加了缓存管理和控制；支持长连接；允许响应数据分块；强制要求 Host 头部，让互联网主机托管成为可能；

现在广泛使用的版本就是 HTTP/1.1，很难想象，这竟然是一个二十多年前的版本。HTTP 基于请求响应模型，这就导致应用层面的队头阻塞问题，同时并未对 Header 头部进行压缩，当携带大量头部数据，比如 Cookie 时，传输效率自然不堪。Google 率先发布了 SPDY 协议，后续 IETF 组织在 SPDY 协议的基础上，在 2015 年正式发布 HTTP/2，此时 HTTP 已经没有小版本号的概念了。HTTP/2 增加了多路复用；二进制分帧层，不再是纯文本；头部压缩 HPACK 算法；请求优先级；服务器推送；增强了安全性，“事实上”的加密通信；

但是 HTTP/2 只是在应用层解决了由请求响应模型造成的队头阻塞问题，并未解决在传输层由于 TCP 必须保证资源的有序到达所带来的 TCP 队头阻塞问题，那该怎么办呢？那就把 TCP 换掉咯？HTTP/3 正是这样做的，HTTP/3 底层不在依赖于 TCP，而是 UDP。目前 HTTP/3 正处于草案阶段，它减少了 TCP 三次握手以及 TLS 握手时间；改进了拥塞控制；避免队头阻塞的多路复用；连接迁移；前向冗余纠错。

#### HTTPS

HTTPS 的产生是为了解决 HTTP 的不安全性，它包括三个方面，机密性、完整性和身份验证。它在 HTTP 之下 TCP 之上加了一个 TLS 层。TLS 层最核心的就是加密套件：

![](https://i.loli.net/2019/12/25/lXIRgqPwK8AYWa7.png)

其中，ECDHE 密钥交换协议用于协商出对称加密算法所需的密钥，也就是 AES 算法，它的密钥长度为 128 位，工作模式为 GCM。非对称加密算法 RSA 用作身份验证，SHA256 是一种 MAC 算法，用于解决完整性。以上已经解决了机密性和完整性，但是身份验证还需要 PKI 的参与。

PKI 即公钥基础设施，服务器会把它的公钥（也就是 RSA 的公钥）给 CA 机构，CA 机构会给服务器签发一个数字证书。在客户端建立连接的时候，服务器会通过 Certificate 子消息将证书链传给客户端，告诉客户端我就是你要通信的那个人。

RSA 也能作为密钥交换算法，但是因为其性能低下，需要大量指数运算消耗 CPU，同时不具备前向安全性，所以现在基本上看不到采用 RSA 来实现密钥交换。为了解决运算效率以及保障前向安全性，目前都会使用 ECDHE 作为密钥交换。ECDHE = ECC 椭圆曲线 + DHE 密钥交换。动态的 DH 每次协商都是生成不同的 DH 参数保证了前向安全性，而且 ECC 密钥长度短，运算快。224 位的 ECC 密钥就能够达到 2048 位的 RSA 的安全强度。

对称加密，加密解密使用同一个密钥，对称加密的工作原理可以简单的看成是异或运算。但是实际情况下，明文长度肯定远大于密钥长度，这就需要对明文进行拆分，每一个分组是 16 字节，可以采用的分组模式有 ECB、CBC、CTR。ECB 不能隐藏数据特征，CBC 不能并行运行，所以通常采用 CTR 工作模式。当然，如果最后一个明文块不足 16 字节，就需要对其进行填充，填充方式一般采用 PKCS#7。

了解了密码学之后，TLS 就很简单了，用 WireShark 抓一下 TLS 包即可。

![](https://i.loli.net/2019/12/25/zIEy8hkwqCPWAJj.png)

但是不能每次进行 HTTPS 连接都需要 TLS 握手吧？

肯定不是，所以有了 TLS 会话机制，可以基于 SessionID 和  Session Ticket。

TLS1.3 在 2018 年正式发布，它移除了很多已证明不安全的套件，在第一次握手时客户端把所有的密码套件及其公钥都发给了服务端，也就不再需要 Server Key Exchange 和 Client Key Exchange，从 2 RTT 到 1 RTT。同时，TLS1.3 支持基于会话回复的 0RTT 握手。

#### HTTP2

HTTP2 有三个核心概念，Strem 流、Message 消息、Frame 帧。在一个 TCP 连接上可以有多个流，也就是并发多请求，实现多路复用；以前浏览器是只能对一个域名开启 6 个并发链接；同时流还可以设置优先级，让服务器优先处理；一条流上可以有多个 Message 消息，每个 Message 都可以携带多个 Frame 帧，帧类型包括 HEADER 帧和 DATA 帧，也就对应于 HTTP/1.1 中的 header 和 data。在 HTTP/1.1 中，服务端是没法主动向客户端推送信息的，客户端只能以轮询的方式去请求，在 HTTP2 中有了服务器推送，也就是服务端可以提前将资源推送至浏览器缓存。

HTTP2 只在应用层解决了队头阻塞问题，并未在 TCP 层解决队头阻塞问题。TCP 的报文传输时无序，接收时组装。如果队头包没有到达，即使后序数据包已经接收到了也是没办法交给上层应用程序处理的，只能等待重发。而在 HTTP3 采用了 UDP，UDP 先天没有队列的概念，自然就解决了队头阻塞的问题，但是它仍然保留了 TCP 的可靠性，这也需要 QUIC 协议自己去实现重传机制、拥塞控制等机制。


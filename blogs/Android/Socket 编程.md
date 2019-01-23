---
Socket 编程基础
---

#### 目录

1. 思维导图
2. 概述
3. UDP Socket
4. TCP Socket
5. 参考

#### 概述

Sccket 中文意思是插座的意思，专业术语称之为套接字，它把 TCP/IP 封装成了调用接口供开发者调用，也就是说通过调用 Socket 相关 API 来实现网络通讯。在 Java 中也存在 Socket 相关 API，主要分为两个，分别是基于 UDP 传输协议的 Socket 和基于 TCP 传输协议的 Socket。

#### UDP Socket

UDP 是无连接的，只要提供对方的 IP 地址和端口号就能进行数据的传输，其中 IP 负责定位主机端口负责定位应用。知道了目标 IP 和目标端口号通过 Java 中的 UDP Socket 就能进行 I/O 传输。


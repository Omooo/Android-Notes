---
Http 与 Https
---

#### 目录

1. Http
   - 报文格式
   - 缓存机制
2. Https
   - 加密
   - 身份验证
   - 消息完整性验证
3. HttpURLConnection
4. 常见问题汇总
5. 参考

#### Http

Http 即超文本传输协议，是一种建立在 TCP 上的无状态连接，属于应用层协议，其传输的内容都是明文的，不安全。Http 是互联网的基础协议，用于客户端与服务器端之间的通信，它规定了客户端与服务端之间的通信格式，包括请求和响应的格式。

基本的工作流程是客户端发送一个 Http 请求，服务端收到请求之后开始处理，处理结果返回给客户端，客户端对结果进行处理并展示。现在最流行的 Http 版本是 HTTP/1.1。

##### 报文格式

报文是在 Http 应用程序之间发送的数据块，它可以分为请求报文和响应报文，不过它们都是由三部分组成：起始行、首部以及主体。

请求报文：

```xml
<method><path><version>
<headers>

<entity-body>  
  
//起始行  
GET /tools/mockapi/448/weBankstep HTTP/1.1	
//首部  
Host: www.wanandroid.com
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Referer: https://www.wanandroid.com/tools/mockapi
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: xxx 
//主体
Get 请求通常不会发送主体，所以为空  
```

响应报文：

```xml
<version><status-code><reason-phrase>
<headers>

<entity-body>
  
//起始行
HTTP/1.1 200 OK
//首部  
Server: Apache-Coyote/1.1
Cache-Control: private
Expires: Thu, 01 Jan 1970 08:00:00 CST
Content-Type: application/json;charset=utf-8
Transfer-Encoding: chunked
Date: Fri, 15 Mar 2019 13:30:00 GMT
//主体
Response: {
    "idInfoExisted": true,
    "openAccountStep": "NONE"
}
```

method：请求方法，常见的有 GET、POST、DELETE 等等。

path：请求路径

version：HTTP 版本，常见的是 HTTP/1.1

status-code：状态码，不同的状态码对应不同的响应状态

```
状态码主要分为五类：
1xx : 临时性消息
	100：继续发送
	101：正在切换协议
2xx : 响应成功
	200：响应成功
3xx : 重定向
	301：域名永久移动
	302：暂时移动
	304：内容未改变，请求被重定向到客户端本地缓存
4xx : 客户端错误
	400：客户端请求错误，服务器不理解请求的语法
	401：未授权，要求进行身份验证
	403：被禁止，服务器拒绝请求
	404：找不到内容，服务器找不到请求的网页
5xx : 服务器错误
	500：服务器不可用
	503：服务不可用
```

reason-phrase：原因短句，对状态码进行简单的描述

Host：接受请求的服务器的主机名和端口

Server：包含了处理请求的源服务器所用到的软件相关信息

......

##### 缓存机制

Web 缓存是一种保存 Web 资源副本并在下次请求时直接使用该副本的技术。

Web 缓存可以分为这几种：浏览器缓存、CDN 缓存、服务器缓存、数据库缓存。因为可能会直接使用副本免于重新发送请求或者仅仅确认资源没变无需重新传输资源实体，Web 缓存可以减少延迟加快网页打开速度、重复利用资源减少网络带宽消耗、降低请求次数或者减少传输内容从而减轻服务器压力。

这里主要阐述浏览器 http 缓存，浏览器 http 缓存可以分为**强制缓存和协商缓存**。强制缓存和协商缓存最大也是最根本的区别在于：强制缓存命中的话不会发请求到服务器，协商缓存一定会发请求到服务器。通过资源的请求首部字段验证资源是否命中协商缓存，如果协商缓存命中，服务器会将这个请求返还，但是不会返回这个资源的实体，而是通知客户端可以从缓存中加载这个资源。

#### 加密方式

加密方式分为对称加密和非对称加密。

##### 对称加密

通信双方采用相同密钥进行加解密，即：

```java
encrpty(明文，密钥) = 密文
decrpty(密文，密钥) = 明文
```

##### 非对称加密

传输数据采用公钥加密，必须采用公钥对应的私钥才能解密，即：

```java
encrpty(明文，公钥) = 密文
decrpty(密文，私钥) = 明文
```

对称加密速度快，加密时 CPU 资源消耗少；非对称加密对待加密的数据的长度有比较严格的要求，不能太长，但是实际上消息可能会很长，所以用对称加密来加密数据。

#### Https

Https 就是在 http 上新增了一个 SSL/TLS，它提供了三个基本服务：

1. 加密
2. 身份验证（数字证书）
3. 消息完整性校验

##### 加密

加密的过程可以分为三步：

1. 客户端发起 https 请求，服务端生成公钥和私钥，并把公钥返回给客户端
2. 客户端验证公钥的合法性，验证通过之后用公钥加密密钥，并把经过公钥加密后的密钥传输给服务端
3. 服务端通过私钥解密得到密钥，这个密钥是用于客户端和服务端进行加密通信的对称加密的密钥了

但是服务端的公钥如何被坏人替换了不就全完了？为了防止中间人攻击，所以就有了身份校验。

##### 身份校验

为了保证服务端给客户端下发的公钥是真正的公钥，而不是中间人伪造的公钥，就需要引入数字证书。数字证书是服务端主动去权威机构申请的，证书中包含了由 CA 加密过的服务端的公钥，所以服务端现在只需要向客户端下发数字证书即可。

数字证书中的服务端的公钥是如何加密的呢？是通过非对称加密，只不过这里采用的是权威机构自己的私钥加密的。既然服务端的公钥被权威机构的私钥加密了，那客户端只能通过拿到权威机构的公钥才能解密呀，那这个权威机构的公钥如何安全的传输给客户端呢？这不就陷入了死循环了嘛？

其实权威机构的公钥不需要传输，因为权威机构会和主流的浏览器或操作系统合作，将它们的公钥内置在浏览器或操作系统中。客户端收到证书之后，只需要从证书中找到权威机构的信息，并从本地环境中找到权威机构的公钥，就能正确解密拿到服务端返回的公钥了。

但是如何整个证书都被中间人替换了，或者证书内容被替换了呢？

这些数字证书本身就已经提供了方案，数字证书中除了包含加密之后的服务端公钥，权威机构的信息之外，还包含了证书内容的数字签名（先通过 hash 函数计算得到证书数字摘要，然后用权威机构的私钥加密数字摘要得到数字签名），签名计算方法以及证书对应的域名。这样一来，客户端收到证书之后：

- 使用权威机构的公钥解密数字证书，得到服务端的公钥以及证书的数字签名，然后根据证书上的描述的计算证书签名的方法计算一下当前证书的签名，与收到的签名做对比，如果一样，表示证书一定是服务器下发的，没有被中间人篡改过。中间人虽然有权威机构的公钥，能够解析证书并篡改，但是篡改完成之后中间人需要将证书重新加密，但是中间人没有权威机构的私钥，无法加密。强行加密只会导致客户端无法解密。

##### Https 握手过程

1. 客户端发送 random1 + 支持的加密算法 + SSL 版本信息
2. 服务端发送 random2 + 选择的加密算法 + 证书
3. 客户端验证证书，并用公钥加密的 random3
4. 服务端解密 random3，此时两端共有 random1、random2、random3，使用这 3 个随机数通过加密算法计算对称加密秘钥即可

以上只有 random3 是加密的，所以用 random1+2+3 这三个随机数加密生成密钥。

#### HttpURLConnection

生在一个好年代呀，从一开始使用网络请求就是用的 OkHttp，对于 HttpURLConnection 浑然不知，现在基本上也不会用到 HttpURLConnection 了，所以就简单回顾一下就行了：

```java
    public String get(String path) throws IOException {
        URL url=new URL(path);
        HttpsURLConnection urlConnection = (HttpsURLConnection) url.openConnection();
        urlConnection.setConnectTimeout(5000);
        urlConnection.setRequestMethod("GET");
        if (urlConnection.getResponseCode() == HttpURLConnection.HTTP_OK){
            InputStream stream=urlConnection.getInputStream();
            return convertStreamToString(stream);
        }
        return null;
    }

    public String convertStreamToString(InputStream is) {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[1024];
            int len = 0;
            while ((len = is.read(buffer)) != -1) {
                baos.write(buffer, 0, len);
            }
            baos.close();
            is.close();
            byte[] byteArray = baos.toByteArray();
            return new String(byteArray);
        } catch (IOException e) {
            return null;
        }
    }
```

#### 常见问题汇总

1. 数字证书以及数字签名的作用？

   数字证书能避免公钥被替换，而数字签名是为了校验公钥的合法性。

2. 握手过程为什么要用到 3 个随机数？

   Https 握手过程中出现的 3 个随机数，只有第三个是加密的随机数。由于很多主机可能产生弱随机数，因此三个随机数叠加更接近随机，比较安全。

3. GET 和 POST 的区别？

   通常对于这个问题，我们都能回答出：GET 请求的参数会拼接在 URL 后面，可见不安全；而 POST 请求的参数是放在主体中的，不可见安全。GET 请求可被缓存，而 POST 不行，等等。

   以上都是从表现上看的，下面会从语义上来阐述两者的区别。

   GET 的语义是获取资源，而 POST 的语义是处理资源。

   当然用 GET 修改信息，POST 获取资源也是可以的，但是并不符合其语义。这里就提到了 RFC 定义了几个性质：

   安全性：如果一个方法的语义在本质上是只读的，那么这个方法就是安全的。客户端向服务端的资源发起的请求如果是使用了安全的方法，就不应该引起服务端任何状态变化，因此也是无害的。

   幂等性：幂等的概念是指同一个请求方法执行多次和仅执行一次的效果完全相同。引入幂等性主要是为了处理同一个请求重复发送的情况，比如在请求响应前失去连接，如果方法是幂等的，也就可以放心的重发一次请求。这也是浏览器在后退/刷新遇到 POST 请求会提示用户的原因：因为 POST 语义不是幂等的，重复请求可能会带来意想不到的后果。

   可缓存性：顾名思义就是一个方法是否可以缓存，绝大多数浏览器仅仅支持 GET、HEAD 缓存。

   但是**协议不等于实现**，所以我上面说用 GET 修改信息也是可以的。

   所以，GET 和 POST 请求仅仅是在语义上的不同：GET 是安全、幂等、可缓存的；而 POST 是非安全、非幂等、（大部分实现）不可缓存的。

   参考：[https://www.zhihu.com/question/28586791](https://www.zhihu.com/question/28586791)

   这里纠正一个错误：GET 请求发一个包，POST 请求发两个包。

#### 参考

[海绵宝宝也懂的HTTPS](https://juejin.im/post/5c341549e51d45524860cf99)

[https://www.jianshu.com/p/ca7df01a9041](https://www.jianshu.com/p/ca7df01a9041)

[浏览器HTTP缓存机制](https://juejin.im/post/5a673af06fb9a01c927ed880)

[HTTP 协议完全解析](https://juejin.im/post/5c629c7c518825622f12da14)

[HTTP 必知必会的那些](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492918&idx=1&sn=40030cd46cbba2bc639602aa341f3739&chksm=8eec8649b99b0f5f09aedc590901f61e1729ff03e6381a3127890381841dfcdd14af282a13fc&scene=21#wechat_redirect)


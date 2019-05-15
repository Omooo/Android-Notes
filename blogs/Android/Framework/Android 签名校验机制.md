---
Android 签名校验机制 v1、v2、v3
---

#### 目录

1. 概述
2. v1
3. v2
4. v3
5. 参考

#### 概述

Android 支持以下三种应用签名方案：

- v1：基于 JAR 签名
- v2：APK 签名方案 v2（在 Android 7.0 引入）
- v3：APK 签名方案 v3（在 Android 9 引入）

为了最大限度地提高兼容性，请按照 v1、v2、v3 的先后顺序采用**所有**方案对应用进行签名。与只通过 v1 方案签名的应用相比，还通过 v2+ 方案签名的应用能够更快的安装到 Android 7.0 及更高版本的设备上。更低版本的 Android 平台会忽略 v2+ 签名，这就需要应用包含 v1 签名。

#### v1 方案：JAR 签名

v1 利用的是 jarsigner 工具，它是 JDK 自带的。它会对所有的文件（包括 META-INF 中与签名不相关的文件）都会被签名。

在 APK 的 META-INF 目录下一般有三个文件：.MF、.SF 和 .RSA 三个文件。

**（1）.MF 文件：**

APK 当中的原始文件信息用摘要算法如 SHA1 计算得到的摘要信息并用 base64 编码保存，以及对应采用的摘要算法如 SHA1。

**（2）.SF 文件：**

.MF 文件的摘要信息以及 .MF 文件当中每个条目在用摘要算法计算得到的摘要信息并用 base64 编码保存。

**（3）.RSA 文件：**

存放证书信息，公钥信息，以及用私钥对 .SF 文件的加密数据即签名信息，这段数据是无法伪造的，除非有私钥，另外，.RSA 文件还记录了所用的签名算法等信息。

APK 包在安装的时候，是按照从 （3）到（1）到顺序依次校验的。先用公钥还原签名信息，然后和 .SF 文件中的信息对比，然后用同样的摘要算法对 .MF 文件里面的每一个条目计算对应的摘要信息，然后对比 .MF 文件是否一致。

在这个过程中，我没发现有两点：

1. 在校验的过程中需要解压，因为 .MF 文件的摘要信息是基于原始未压缩文件内存，因此在校验的时候就需要解压出原始数据，而这个解压操作无疑是耗时操作。
2. APK 包的完整性校验不够强。这里我们可以看到，如果我们在 APK 签名后，对 APK 包中没有涉及到原始文件的数据块做改变，那么这层校验机制就会失效。所以，在美团的打包工具 Walle 的做法就是在 META-INF 中添加空文件来实现多渠道打包，因为在 META-INF 中添加新文件是不需要重新签名的。这也同时说明了，其实存在过度签名的问题。

为了解决这些问题，Android 7.0 中引入了 APK 签名方案 v2。

#### v2 方案

APK 签名方案 v2 是一种全文件签名方案，该方案能够发现对 APK 受保护部分进行的所有更改，从而有助于加快验证速度并**增强完整性保证**。

使用 APK 签名方案 v2 进行签名时，会在 APK 文件中插入一个 APK 签名分块，该分块位于 " ZIP 中央目录" 部分之前并紧邻该部分。在 " APK 签名分块" 内，v2 签名和签名者身份信息会存储在其中。

![](https://i.loli.net/2019/05/13/5cd8e2f3387d818425.png)

v2 签名机制不存在解压原始数据，签名校验时间显著减少，因此安装时间也相应减少。同时，它是基于 APK 的二进制内存做的签名信息（APK Signing Block 签名块本身不参与加密校验），因此打包后改变 APK 的其他三部分的任何字节都会导致签名校验不通过。这也说明了，zipalign 是要在 apksigner 之前的。

APK Signing Bolck 由这几个部分组成：两个用来标示这个区块长度的八字节 + 这个区块的魔数（APK Sig Bolck 42）+ 这个区块所承载数据（ID-value）。

| 偏移 | 字节数 | 描述                                        |
| ---- | ------ | ------------------------------------------- |
| @0   | 8      | 这个 Block 的长度（本字段的长度不计算在内） |
| @8   | n      | 一组 ID-value                               |
| @-24 | 8      | 这个 Bolck 的长度（和第一个字段一样值）     |
| @-16 | 16     | 魔数 "APK Sig Block 42"                     |

 v2 的签名信息是以 ID（0x7109871a）的 ID-value 来保存在这个区块中，这是一组 ID-value，但是在验证的时候，只是通过 ID 为 0x7109871a 的来获取 APK Signature Scheme v2 Block，对这个区块中其他的 ID-value 选择了忽略。所以 Walle 提供一个自定义的 ID-value 并写入该区域，从而为快速生成渠道包服务。

想要了解是如何做的，请参考：[新一代开源Android渠道包生成工具Walle](https://tech.meituan.com/2017/01/13/android-apk-v2-signature-scheme.html)

在 APK 签名分块内，也就是 ID 为 0x7109871a，它里面有一个叫做 **signed data** 的数据结构。APK 签名方案 v2 负责保护第1、3、4 部分的完整性，以及第二部分包含的 APK 签名方案 v2 分块中的 signed data 分块的完整性。

第1、3、4 部分的完整性通过其内容的一个或多个摘要来保护，这些摘要存储在 signed data 分块中，而这些分块则通过一个或多个签名保护。第1、3、4 部分都会被拆分为多个大小为 1MB 的连续块分段计算摘要，摘要以分段方式计算，以便通过并行处理来加快计算速度。

APK 签名方案 v2 是在 Android 7.0 中引入的，为了使 APK 可在 Android 6.0 及更低版本的设备上安装，应先使用 JAR 签名功能对 APK 进行签名，然后再使用 v2 方案对其进行签名。

#### v3 方案

Android 9 支持 APK 密钥轮转，这使得应用能够在 APK 更新过程中更改签名密钥。为了实现轮转，APK 必须指示新旧签名密钥之间的信任级别。为了支持密钥轮转，我们将 APK 签名方案从 v2 更新为 v3，以允许使用新旧密钥。v3 在 APK 签名分块中添加了有关受支持的 SDK 版本和 proof-of-rotation 结构的信息。

v3 APK 签名分块的格式与 v2 相同，APK 的 v3 签名会存储为一个 ID 为 0xf05368c0 的 ID-value。看来 Google 给自己留了后路，以后再增加版本，肯定又是增加新的 ID-value，在高版本的 SDK 设备上解析即可。

更多请参考：[Android 文档 APK 签名方案 v3](https://source.android.com/security/apksigning/v3)

#### 参考

[Oracle Signed_JAR_File](https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#Signed_JAR_File)

[https://source.android.com/security/apksigning](https://source.android.com/security/apksigning)

[分析Android V2新签名打包机制](https://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651232457&idx=1&sn=90b16c3868a341272b8f1aa26d6c0122&chksm=f1d9e5aac6ae6cbcfaecb07bdd280abf81a46f1937c43f61e69d7f78d64350943356f5443d58&scene=27#wechat_redirect)

[新一代开源Android渠道包生成工具Walle](https://tech.meituan.com/2017/01/13/android-apk-v2-signature-scheme.html)


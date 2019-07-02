---
AOP 之 AspectJ 与 ASM
---

#### 目录

1. 概述

2. AspectJ

3. ASM

   - 前言

   - 自定义 Gradle Plugin
   - Transform API
   - ASM

#### 概述

AOP 即面向切面编程，它是一种编程思想，能对一些事物进行统一处理。在 Android 中，ActivityLifecycleCallbacks 就可以看作是一种 AOP 的实现，它能对 Activity 的生命周期进行统一管理。

AOP 只是一种方法论，它的实现有很多种，动态代理、AspectJ、ASM、Javassist 等等，每个实现方式都要能动态生成字节码，这也是 AOP 的核心。

这里推荐一篇网易的 [AOP技术在客户端的应用与实践](https://mp.weixin.qq.com/s?__biz=MzUxODg0MzU2OQ==&mid=2247483887&idx=1&sn=d54e3f210a4f31f477dba06c3dcd352e&scene=21#wechat_redirect)。

这里只介绍 AspectJ 和 ASM，Aspect 上手简单，与 Java 完全互操作，ASM 更加灵活，短小精悍，性能没得说，毫不夸张的说，ASM 无所不能，可以看作是 AOP 的终极方案，但是上手还是需要一点操作的。

#### AspectJ

AspectJ 使用简单，熟悉了切点表达式就能直接上手了。

入门看一下两篇就够了：

[AOP AspectJ 字节码 语法 MD](https://baiqiantao.github.io/Java/aop/qQnamy/)

[关于AspectJ，你需要知道的一切](http://linbinghe.com/2017/65db25bc.html)

这时候就能简单的写一些 Demo 了，但是显然还不够，你还需要写一些实际的工程，才能更加深刻的理解 AspectJ 到底能做什么有意思的事。

首推 JakeWharton 的 [hugo](https://github.com/JakeWharton/hugo)，拥有近 7K star，这个库是用来打印方法耗时、方法参数等的，说出来你可能不信，整个 Project 只有五个类，你完全可以对照自己写一个。

然后是一个 [QuickPermissions](https://github.com/QuickPermissions/QuickPermissions) 的库，是用来解决动态权限的，最初是在 Medium 上看的一篇文章，原文地址： [the-easiest-way-to-integrate-run-time-permissions-in-android](https://medium.com/reversebits/the-easiest-way-to-integrate-run-time-permissions-in-android-828f60710b22)，虽然是 Kotlin 写的，你可以试试用 Java 写，核心思想就那些。

除此之外，AspectJ 还能解决一些系统 bug，比如 Toast 不弹出等等，相见：[Toast与Snackbar的那点事](https://tech.meituan.com/2018/03/29/toast-snackbar-replace.html)

虽然 AspectJ 实现原理是字节码插桩，但是实际上你会发现，你完全不需要任何字节码基础就能上手了。

#### ASM

ASM 是一个广泛使用的字节码操作库，前面之所以 ASM 需要一点操作，是因为你需要熟悉字节码，还有一些 ASM 中各种 API 之间的关系。其次，既然是操作字节码的，那我们肯定需要先拿到字节码，这就涉及到了 Gradle Transform API 了。但是使用 Gradle Transform API 是依赖于 Gradle Plugin 的，所以可以看出想学 ASM，还是需要很多前置知识的。

##### 字节码基础

《深入理解 Java 虚拟机》第六章。

##### 自定义 Gradle Plugin

这里就只能委屈你看我写的 [Gralde Plugin 入门指南](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/Gradle/Gradle_Plugin_Guide.md) 和 [Gralde Plugin 实践之 TinyPng Plugin](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/Gradle/TinyPngPlugin.md)

##### Transform API

应用在构建的时候，首先是通过 javac 将源码编译成 class 文件，然后 class 文件再经过 dx 工具生成 dex 文件。而 Transform 就可以干预打包流程，在 class 转 dex 的时候对 class 进行再处理，修改字节码或插入字节码。

这里不得不放出旧版本的 Gradle 构建流程图：

![](https://i.loli.net/2019/07/02/5d1b06fa4830776940.png)

Gradle 内置的 Transform（Task）有 mergeManifest、Proguard 等等，和我们自定义的 Transform 形成了一个 Trasnform 链，我们定义的 Transform 会首先执行：

![](https://i.loli.net/2019/07/02/5d1b08351411e92955.png)

图片来源：[https://juejin.im/entry/577b03438ac2470061afb130](https://juejin.im/entry/577b03438ac2470061afb130)

##### ASM

刚开始学 ASM 的时候，我还是建议先在 IDEA 里面先写一些 Demo，比如给类添加一个方法、给已有方法添加代码、删除已有方法等；

官方中文文档：[ASM4 使用指南](https://github.com/Omooo/Android-Notes/blob/master/books/Android/ASM4%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.pdf)

这里推荐：[手摸手增加字节码往方法体内插代码](http://www.wangyuwei.me/2017/01/22/%E6%89%8B%E6%91%B8%E6%89%8B%E5%A2%9E%E5%8A%A0%E5%AD%97%E8%8A%82%E7%A0%81%E5%BE%80%E6%96%B9%E6%B3%95%E4%BD%93%E5%86%85%E6%8F%92%E4%BB%A3%E7%A0%81/)

然后就可以试着用 ASM 实现 hugo 了，推荐：[ASM实战统计方法耗时](http://www.wangyuwei.me/2017/03/05/ASM%E5%AE%9E%E6%88%98%E7%BB%9F%E8%AE%A1%E6%96%B9%E6%B3%95%E8%80%97%E6%97%B6/)

到这里就没了，Emmmm，网上大多 ASM 教程都是统计方法耗时，很尴尬。
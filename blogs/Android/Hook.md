---
Hook
---

#### 目录

1. 思维导图
2. 概述
3. 实现方式
4. 参考

#### 思维导图

#### 概述

Hook 就是勾子的意思，之前我以为 Hook 技术是很难理解的呢，没想到真正去了解的时候是那么简单。

一般 Hook 的实现方式是通过反射和代理实现，只能 Hook 当前的应用程序进程。通过 Hook 框架来实现，比如 Xposed，可以实现全局 Hook，但是需要 root。

#### 实现方式

##### 代理实现

代理分为静态代理和动态代理，这里就不过多阐述了，详见：

[工厂模式](https://github.com/Omooo/Android-Notes/blob/master/blogs/DesignMode/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F.md)

Hook View 的 onClick 事件我记得在写埋点的时候写过，这里就不再多说了，现在来 Hook startActivity 的实现：



#### 参考
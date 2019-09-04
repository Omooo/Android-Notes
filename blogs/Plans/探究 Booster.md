---
Booster 探究之路
---

> Booster 是一款专门为移动应用设计的易用、轻量级且可扩展的质量优化框架，通过动态发现和加载机制提供可扩展的能力。它不仅仅是一个框架，而且还内置了很多质量优化工具。
>
> Booster 主要由 Transform 和 Task 组成，Transform 主要用于处理字节码，Task 主要用于处理资源，为了满足特异的优化需求，Booster 提供了 Transformer SPI and VariantProcessor SPI 允许开发者进行定制。

### 构想

Booster 是滴滴开源的移动 APP 质量优化框架，开源地址：[Booster](https://github.com/didi/booster)。

对比 Matrix，相比之下学习 Booster 的学习曲线没那么陡峭。只需要简单的了解 Gradle Plugin、Transfrom、ASM 就可以上手了，如果你没这些基础也没有关系，只要愿意去学，我们完全可以花费前两周时间好好去补这一部分基础知识，大家也可以互相把好的资料分享出来，让你避免走弯路。

### 时期安排

暂定为两个月，如果你基础很好或者你很熟悉 Booster，欢迎加入，请带带我 :)

![](https://i.loli.net/2019/09/04/tykZqwsj6dLPWUM.jpg)

### 最后期望

1. 熟悉 Kotlin 函数式编程
2. 掌握 Gradle Plugin 的编写、Transfrom / Task 运用、以及字节码操作库 ASM 的使用
3. 对 APP 质量监控 / 性能优化有进一步理解

最后希望每一个小伙伴不仅可以说出 Booster 的实现原理，还能都可以**从零搭建**自己的 APM 框架，然后实现 Booster 里面的几个小模块。


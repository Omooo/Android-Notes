---
Dalvik JIT ART
---

#### 目录

1. 思维导图
2. Android 虚拟机发展史
3. Dalvik 和 ART 的区别
4. JIT 和 AOT 的区别
5. 参考

#### 思维导图

![](https://i.loli.net/2018/12/30/5c2847c900abb.png)

#### Android 虚拟机发展史

##### Android 诞生之初

Dalvik 担任虚拟机的角色，每次运行程序的时候，Dalvik 负责加载 dex/odex 文件并解析成机器码交由系统调用。

##### Android 2.2 JIT 登场

和其他大多数 JVM 一样，Dalvik 使用 JIT 进行即时编译，借助 Java HotSpot VM，JIT 编译器可以对热点代码进行编译优化，将 dex/odex 中的 Dalvik Code ( Smali 指令集 ) 翻译成相当精简的 Native Code 去执行，JIT 的引入使得 Dalvik 的性能提升了 3~6 倍。

##### Android 4.4 ART 和 AOT 登场

在 Android 4.4 带来了全新的虚拟机运行环境 ART 和全新的编译策略 AOT，此时 ART 和 Dalvik 是共存的。

##### Android 5.0 ART 全面取代 Dalvik

至此，Dalvik 退出历史舞台，AOT 也成为了唯一的编译方式。

##### Android 7.0 JIT 回归

形成了 AOT / JIT 混合编译模式。该混合模式综合了 AOT 和 JIT 的各种优点，使得应用在安全速度加快的同时，运行速度、存储空间和耗电量等指标都得到了优化。

#### Dalvik 和 ART 的区别

ART 相比于 Dalvik 的优点：

1. 预编译 AOT
2. 垃圾回收方面的优化
   - 只有一次（ 而非两次 ）GC 暂停
   - 在 GC 保持暂停状态期间并行处理
   - 压缩 GC 以减少后台内存使用和碎片

3. 开发和调试方面的优化

#### AOT 和 JIT 的区别

##### JIT：

![](https://i.loli.net/2018/12/30/5c28227e9df60.png)

JIT 的优点：

1. 对热点代码进行编译优化

JIT 的缺点：

1. 每次启动应用都需要重新编译
2. 运行时比较耗电

##### AOT

![](https://i.loli.net/2018/12/30/5c283927b3b96.png)

优点：

1. 应用启动更快、运行速度更加流畅
2. 耗电问题得以优化

缺点：

1. 应用安装和系统升级之后的应用优化比较耗时
2. 优化后的文件会占用额外的存储空间

#### 参考

[https://juejin.im/post/5c232907f265da61662482b4](https://juejin.im/post/5c232907f265da61662482b4)

[https://source.android.com/devices/tech/dalvik?hl=zh-cn](https://source.android.com/devices/tech/dalvik?hl=zh-cn)

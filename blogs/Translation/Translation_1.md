---
两年开发经验给我带来哪些教训？
---

#### 前言

本文摘自：[What 2 Years of Android Development Have Taught Me the Hard Way](https://blog.aritraroy.in/what-my-2-years-of-android-development-have-taught-me-the-hard-way-52b495ba5c51)

#### 正文

作者给出了他的十八条建议：

1. 不要造轮子

   对于一些网络库、图片加载库等都已经有了很好的实现，完全没必要造轮子，重复造轮子很浪费时间。但是如果你觉得只是用别人的东西，自己学不到东西，别着急，慢慢往下看。

2. 选择合适的库

   虽然现在第三方库很多，但是在做选择时候，还是要考虑清楚。比如这个库的 star 数、文档是否全、是否经常维护等等，都是需要考虑到的。

3. 读代码

   通常，我们看代码的时间比写代码多。你现在写的代码，都是你在某些时候某些地方读到的。好在，Android 是开源的，也有很多优秀的库可以去试着读，你会收获很多。[这里](https://github.com/pcqpcq/open-source-android-apps)有很多开源的项目你可以参考。

4. 加强你的代码风格。

   可以参考 [AOSP 代码样式指南](https://source.android.com/source/code-style.html) 和 [这个编码规范](https://github.com/ribot/android-guidelines/blob/master/project_and_code_guidelines.md)。

5. 你需要使用 ProGuard

   可以参考 [ProGuard](https://stuff.mit.edu/afs/sipb/project/android/sdk/android-sdk-linux/tools/proguard/docs/index.html#/afs/sipb/project/android/sdk/android-sdk-linux/tools/proguard/docs/manual/usage.html) 和 [DexGuard](https://www.guardsquare.com/en/products/dexguard)。

6. 使用正确的架构

   请参考 [GoogleSimple](https://github.com/googlesamples/android-architecture) 和 [androidmvp](https://github.com/antoniolg/androidmvp)。

7. UI 就像一个笑话，如果你必须去解释它，那是很糟糕的

   通常，不必去关心 UI，那是设计师应该关心的问题。

8. 分析是你最好的朋友

   比如

9. 是

10. 

    

     
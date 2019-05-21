---
深入理解 JNI
---

#### 目录

1. 前言
2. 实例分析之 MediaScanner
3. Java 层的 MediaScanner

#### 前言

JNI 是 Java Native Interface 的缩写，中文译为 " Java 本地调用 "。通俗的说，JNI 是一种技术，通过这种技术可以做到 Java 中的函数和 C/C++ 编写的 Native 函数相互调用。

Android 中大量使用了 JNI 技术，本节就以 MediaScanner 来讲解 JNI。

#### 实例分析之 MediaScanner

MediaScanner 是 Android 平台中多媒体系统的重要组成部分，它的功能是扫描媒体文件，得到诸如歌曲时长，歌曲信息等媒体信息，并将它们存入到媒体数据库中，以供其他应用程序使用。

![](https://i.loli.net/2019/05/15/5cdbab5edbb0391147.png)

Java 世界对应的是 MediaScanner，而这个 MediaScanner 类有一些函数需要由 Native 层来实现。JNI 层对应的是 libmedia_jni.so。media_jni 是 JNI 库的名字，其中下划线前的 media 是 Native 库层的名字，这里就是 libmedia 库。下划线后的 ini 表示它是一个 JNI 库。JNI 库的名字可以随便取，但是 Android 平台基本上都是采用 “ lib模块名_jni.so “ 的命名方式。

从上面的分析可知：JNI 层必须实现为动态库的形式，这样 Java 虚拟机才能加载它并调用它的函数。

#### Java 层的 MediaScanner

```java
public class MediaScanner implements AutoCloseable {
  	//1. 加载对应的 JNI 库
    static {
        System.loadLibrary("media_jni");
        native_init();
    }
  	//...
  	//2. 声明 Native 函数，用 native 关键字标示，表示它将由 JNI 层完成
    private native boolean processFile(String path, String mimeType, MediaScannerClient client);
    private static native final void native_init();
  
}    
```

如果 Java 要调用 native 函数，就必须通过一个位于 JNI 层的动态库来实现。动态库就是运行时加载的库，那么在什么时候以及什么地方加载这个库呢？

原则是是，在调用 native 函数前，任何时候、任何地方加载都可以。通常的做法是在类的 static 语句中加载，调用 System.loadLibrary 方法就可以了。其函数的参数是动态库的名字，即 media_jni。系统会自动根据不同的平台扩展成真实的动态库文件名，例如在 Linux 系统上会扩展成 libmedia_jni.so，而在 Windows 平台上则会扩展成 media_jni.dll。

在 Java 层使用 JNI 技术真是太容易了，只需要完成两项工作即可：

1. 加载对应的 JNI 库
2. 声明由关键字 native 修饰的函数

但是在 JNI 层，要完成的任务可没那么轻松了。

#### JNI 层 MeidaScanner

看下 android.media.MediaScanner.cpp 源码：

```c++
//这个函数是 native_init 的 JNI 层实现
static void
android_media_MediaScanner_native_init(JNIEnv *env)
{
    //...
}

//这个函数是 processFile 的 JNI 层实现
static jboolean
android_media_MediaScanner_processFile(
        JNIEnv *env, jobject thiz, jstring path,
        jstring mimeType, jobject client)
{
    //...
}
```

Java 层，native_init 函数位于 android.media 包中，它的全路径名应该是 android.media.MediaScanner.native_init，而 JNI 层函数的名字就是把 "." 替换成了 "_"，通过这样，就可以把 Java 中的 Native 函数和 JNI 层的函数关联起来了。

其实上面说的就是 JNI 函数的注册问题。注册，即将 Java 层的 native 函数和 JNI 层对应的实现函数关联起来。JNI 函数的注册方法实际上有以下两种：

1. 静态注册

   以这种注册方式就是根据函数名来找对应的 JNI 函数，它需要 Java 的工具程序 javah 参与。javah 会根据 class 文件生成一个叫 output.h 的 JNI 层头文件，在生成的 output.h 文件里，声明了对应的 JNI 层函数，只要实现里面的函数即可。

   这个头文件的名字一般都会使用 packagename_class.h 的样式，例如 MediaScanner 对应的 JNI 层头文件就是 android_media_MediaScanner.h。

   当 Java 层调用 native_init 函数时，它会从对应的 JNI 库中寻找 Java_android_media_MediaScanner_native_init 函数，如果没有，就会报错。如果找到，则会把这个 native_init 和 Java_android_media_MediaScanner_native_init 建立一个关联关系，其实就是保存 JNI 层函数的函数指针。以后在调用 native_init 函数时，直接使用这个函数指针就可以了，当然这项工作是由虚拟机完成的。

   这样做会很麻烦，初次调用 native 函数时还要根据函数名搜索对应的 JNI 层函数来建立关联关系，这样会影响运行效率。根据上面介绍可知，Java native 函数是通过函数指针来和 JNI 层函数建立关联关系的，如果直接让 native 函数知道 JNI 层对应函数的函数指针，不就万事大吉啦吗？这就要介绍动态注册了。

2. 动态注册

    Java native 函数和 JNI 函数是一一对应的，在 JNI 中，用一个叫 JNINativeMethod 的结构来保存这种关联关系。

   
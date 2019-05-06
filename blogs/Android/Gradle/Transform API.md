---
Gradle Plugin 之 Transform API
---

#### 前言

在前面两篇文章中，我们熟悉了 [Gradle Plugin](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/Gradle/Gradle_Plugin.md) 的编写，同时也实践了一下，写了一个 [TinyPngPlugin](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/Gradle/TinyPngPlugin.md) ，利用 TinyPng 在打包时压缩 res 下的所有的 png 图片，这是一个非常好的实践，希望你也能掌握。

本篇文章接着来讲解 Transfrom API，Transform 是用来对 class 转 dex 文件之前的 class 文件进行操作。

![](https://i.loli.net/2019/04/22/5cbd314be60b9.png)

这里的 Input 和 Output 都是 class 文件或者 jar 包，因此，我们就可以实现字节码插桩。

#### Transform

使用 Transform 依赖需要首先添加依赖：

```java
implementation 'com.android.tools.build:gradle:3.3.2'
```

Transfrom 使用还是相对简单的：

```java
public class DemoTransform extends Transform {

    /**
     * Transform 标识名
     * 比如我在 app module 下依赖了这个 Plugin
     * 那么在 app/build/intermediates/transforms/
     * 下，就能看到我们的自定义 DemoTransform
     */
    @Override
    public String getName() {
        return "DemoTransform";
    }

    /**
     * 设置文件输入类型
     * 类型在 TransformManager 下有定义
     * 这里我们获取 class 文件类型
     */
    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    /**
     * 设置文件所属域
     * 同样在 TransformManager 下有定义
     * 这里指定为当前工程
     */
    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    /**
     * 是否支持增量编译
     */
    @Override
    public boolean isIncremental() {
        return false;
    }

    @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);
    }
}
```

这里最重要的方法就是 transform 方法。

#### ASM

ASM 是一个字节码操作库。
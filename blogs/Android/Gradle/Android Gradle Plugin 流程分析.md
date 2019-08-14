---
Android Gradle Plugin 主要流程分析
---

#### 前言

> 本文摘自：[Android Gradle Plugin 插件主要流程](https://github.com/5A59/android-training/blob/master/gradle/android_gradle_plugin-%E4%B8%BB%E8%A6%81%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90.md)
>
> 感谢美团大佬 [@5A59](https://github.com/5A59) 的分享！

#### 概述

![Android Plugin 流程分析概述.png](https://i.loli.net/2019/08/14/Ubpnexm3cK5yOPW.png)

#### 准备工作

这个阶段主要有过程：

1. 检查插件版本
2. 检查 module 是否重名
3. 初始化插件信息

我们知道，在自定义 Plugin 的时候，需要定义一个 xxx.properties 文件，文件里面就是该 Plugin 的实现类，而文件名 xxx 就是 apply plugin 的插件 ID，因此，我们只需要查看 com.android.application.properties 文件就可以了，内容如下：

```
implementation-class=com.android.build.gradle.AppPlugin
```

这里定义的入口类就是 AppPlugin，而 AppPlugin 是继承至 BasePlugin。

AppPlugin 里面没有做过多的操作，主要是重写了 createTaskManager 和 createExtension，剩下的大部分工作还是在 BasePlugin 里做的。

```java
    protected void apply(@NonNull Project project) {
		// 检查插件版本
        checkPluginVersion();
        // 检查 module 是否重名
        checkModulesForErrors();

		// 插件初始化信息
        PluginInitializer.initialize(project, projectOptions);
        ProfilerInitializer.init(project, projectOptions);

        ProcessProfileWriter.getProject(project.getPath())
                .setAndroidPluginVersion(Version.ANDROID_GRADLE_PLUGIN_VERSION)
                .setAndroidPlugin(getAnalyticsPluginType())
                .setPluginGeneration(GradleBuildProject.PluginGeneration.FIRST);
```

#### 配置项目

这个阶段有四个过程：

1. 检查 gradle 版本是否匹配
2. 创建 AndroidBuilder 和 DataBindingBuilder
3. 引入 java plugin 和 jacoco plugin
4. 设置构建完成以后的缓存清理工作，添加了 BuildListener，在 buildFinished 回调里做清理工作

```java
// BasePlugin
private void configureProject(){
	// 1.
    checkGradleVersion();
    // 2.
    androidBuilder = new AndroidBuilder();
    dataBindingBuilder = new DataBindingBuilder();
    // 3.
    project.getPlugins().apply(JavaBasePlugin.class);
    project.getPlugins().apply(JacocoPlugin.class);
    // 4.
     PreDexCache.getCache().clear();
}
```

#### 配置 Extension

这一阶段主要做了以下几件事情：

1. 创建 AppExtension，也就是 build.gradle 里用到的 android{} dsl

   ```
   // AppPlugin
   protected BaseExtension createExtension(){
       return project.getExtensions().create("android", AppExtension.class);
   }
   ```

2. 创建依赖管理、ndk 管理、任务管理、variant 管理

   ```
   
   ```

   

3. 
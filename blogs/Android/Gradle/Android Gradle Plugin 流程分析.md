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

   ```java
   private void configureExtension() {
   		ndkHandler = new NdkHandler();
       variantFactory = createVariantFactory();
     	taskManager = createTaskManager();
     	variantManager = new VariantManager();
   }
   ```

3. 注册新增配置的回调函数，包括 signingConfig、buildType、productFlavor

   ```java
   				// BasePlugin#configureExtension()
           signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);
   
           buildTypeContainer.whenObjectAdded(
                   buildType -> {
                       SigningConfig signingConfig =
                               signingConfigContainer.findByName(BuilderConstants.DEBUG);
                       buildType.init(signingConfig);
                     	// addBuildType，会检查命名是否合法，然后创建 BuildTypeData
                       variantManager.addBuildType(buildType);
                   });
   				// addProductFlavor 会检查命名是否合法，然后创建 ProductFlavor
           productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);
   
           // map whenObjectRemoved on the containers to throw an exception.
           signingConfigContainer.whenObjectRemoved(
                   new UnsupportedAction("Removing signingConfigs is not supported."));
           buildTypeContainer.whenObjectRemoved(
                   new UnsupportedAction("Removing build types is not supported."));
           productFlavorContainer.whenObjectRemoved(
                   new UnsupportedAction("Removing product flavors is not supported."));
   ```

4. 创建默认的 debug 签名，创建 debug 和 release 两个 buildType

   ```java
       public void createDefaultComponents(
               @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
               @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
               @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs) {
           // must create signing config first so that build type 'debug' can be initialized
           // with the debug signing config.
           signingConfigs.create(DEBUG);
           buildTypes.create(DEBUG);
           buildTypes.create(RELEASE);
       }
   ```

#### 创建不依赖 flavor 的 Task

```java
    private void createTasks() {
      	// createTasksBeforeEvaluate
      	// 创建不依赖 flavor 的 Task
        threadRecorder.record(
                ExecutionType.TASK_MANAGER_CREATE_TASKS,
                project.getPath(),
                null,
                () ->
                        taskManager.createTasksBeforeEvaluate(
                                new TaskContainerAdaptor(project.getTasks())));

      	// createAndroidTasks
      	// 创建构建的 Task
        project.afterEvaluate(
                project ->
                        threadRecorder.record(
                                ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                project.getPath(),
                                null,
                                () -> createAndroidTasks(false)));
    }
```

上述准备、配置阶段完成以后，就开始创建构建需要的 Task 了，是在 BasePlugin.createTasks() 里实现的，主要有两步，**创建不依赖 flavor 的 task 和创建构建 task。**

先看不依赖 flavor 的 task，其实先在 TaskManager.createTaskBeforeEvaluate()。这里主要创建了几个 task，包括 uninstallAll、deviceCheck、connectedCheck、preBuild、extractProguardFiles、sourcesSets、assemebleAndroidTest、compileLint、lint、lintChecks、cleanBuildCache、resolveConfigAttr、consumeConfigAttr。

这些 task 都是不需要依赖 flavor 数据的公共 task。

#### 创建构建 Task

在介绍下面的流程之前，先明确几个概念，flavor、dimension、variant。

在 Android Gradle Plugin 3.x 之后，每个 Flavor 必须对应一个 Dimension，可以理解为 Flavor 分组，然后不同 Dimension 里的 Flavor 组合成一个 Variant。

举个例子：

```java
    flavorDimensions("size", "color")
    productFlavors {
        big {
            dimension "size"
        }
        small {
            dimension "size"
        }
        blue {
            dimension "color"
        }
        red {
            dimension "color"
        }
    }
```

上面配置对应生成的 variant 就是 bigBlue、bigRed、smallBlue、smallRead，在这个基础上，再加上 buildType，就是 bigBlueDebug、bigRedDebug、smallBlueDebug、smallRedDebug、bigBlueRelease、bigRedRelease、smallBlueRelease、smallRedRelease。

createAndroidTasks 的调用时机和上面不一样，是在 project.afterEvaluate 里调用的，还记得之前文章里说道的 afterEvaluate 回调嘛？这个时候所有模块配置已经完成了。所以在这个阶段可以获取对应的 flavor 以及其他配置了。

在 BasePlugin.createAndroidTasks 里，是调用 VariantManager.createAndroidTasks 完成工作的。


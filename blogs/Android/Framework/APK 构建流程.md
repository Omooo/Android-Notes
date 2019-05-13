---
Android APK 构建流程
---

#### 目录

1. 前言
2. 构建流程

2. Android Gradle Plugin 源码分析

#### 前言

本篇文章大量摘自 [Apk 打包流程梳理](https://juejin.im/entry/58b78d1b61ff4b006cd47e5b) 以及其文章中的参考文章，我觉得你只要把那篇文章及其参考文看完，熟悉 APK 构建流程和安装流程完全不在话下。

当然，看完文章之后，你可以试试自己画一下构建流程图。

本篇主要用来记录学习过程，再次感谢大佬们的分享精神～

#### 构建流程

先来一张 [Android 配置构建](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process) 官网上的一张图：

![](https://i.loli.net/2019/05/07/5cd1888bcfafa.png)

这个新版的图，比较好理解，就不过阐述了，我还是喜欢老版的经典构建图：

![](https://i.loli.net/2019/05/07/5cd14ad86262c.png)

从上面的流程图，我可以看出 APK 构建流程可以分为以下七步：

1. 通过 aapt 打包 res 资源文件，生成 R.java、resources.arsc 和 res 文件（二进制和非二进制如 res/raw 和 pic 保持原样）
2. 处理 .aidl 文件，生成对应的 Java 接口文件
3. 通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，生成 .class 文件
4. 通过 dx/d8 工具，将 .class 文件和第三方库中的 .class 文件处理生成 classes.dex
5. 通过 apkbuilder 工具，将 aapt 生成的 resources.arsc 和 res 文件、assets 文件和 classes.dex 一起打包生成 apk
6. 通过 Jarsigner 工具，对上面的 apk 进行 debug 或 release 签名
7. 通过 zipalign 工具，对签名后的 apk 进行对齐处理

##### 第一步：aapt

aapt（Android Asset Packaging Tool），用于打包资源文件，生成 R.java 和编译后的资源。aapt 的源码位于 frameworks/base/tools/aapt2 目录下。

##### 第二步：aidl

用于处理 aidl 文件，源码位于 frameworks/base/tools/aidl。输入 aidl 后缀的文件，输出可用于进程通信的 C/S 端 Java 代码，位于 build/generated/source/aidl。

##### 第三步：Java 源码编译

有了 R.java 和 aidl 生成的 Java 文件，再加上工程的源代码，现在就可以使用 javac 进行正常的 java 编译生成 class 文件了。

输入：java source 的文件夹（另外还包括了 build/generated 下的：R.java，aidl 生成的 Java 文件以及 BuildConfig.java）

输出：对于 gradle 编译，可以在 build/intermediates/classes 里，看到输出的 class 文件。

##### 第四步：dex

使用 dx/d8 工具将 class 文件转化为 dex 文件，生成常量池，消除冗余数据等。

##### 第五步：apkbuilder

打包生成 APK 文件，现在都已经通过 sdklib.jar 的 ApkBuilder 类进行打包了，输入为我们之前生成的包含 resources.arsc 的 .ap_ 文件，上一步生成的 dex 文件，以及其他资源如 jni、jar 包内的资源。

##### 第六步：Jarsigner

对 apk 文件进行签名，APK 需要签名才能在设备上进行安装。现在不仅可以使用 jarsigner，还可以使用 apksigner。

##### 第七步：zipalign

对签名后的 apk 文件进行对齐处理，使得 apk 中所有资源文件距离文件起始偏移为四个字节的整数倍，从而在通过内存映射访问 apk 文件时会更快。

需要注意的，使用不同的签名工具对对齐是有影响的。如果使用 jarsigner，则只能在 APK 文件签名后执行 zipalign；如果使用 apksigner，则只能在 APK 文件签名之前执行 zipalign。

详见官网描述：[zipalign](https://developer.android.com/studio/command-line/zipalign)

详细构建流程图：

![](https://i.loli.net/2019/05/08/5cd2a35578f65.png)

#### Android Gradle Plugin 

有了以上的初步了解之后，再来看看 Android Gradle Plugin 是如何实现构建流程的。

我们使用的 apply plugin: 'com.android.application' 和 apply plugin:'com.android.library' 插件对应的类分别是 AppPlugin 和 LibraryPlugin。它们都是继承 BasePlugin：

```java
public abstract class BasePlugin<E extends BaseExtension2> implements Plugin<Project>{
    @Override
    public final void apply(@NonNull Project project) {
      	//核心方法
        basePluginApply(project);
      	//因为 apply 是 final
      	//所以提供一个 pluginSpecificApply 用于 AppPlugin 和 LibraryPlugin 加入特殊逻辑
        pluginSpecificApply(project);
    }
}
```

看到重写的 apply 方法是 final，就很开心了，也就是说我们只需要分析 BasePlugin 里面的逻辑就好了。核心方法是 basePluginApply，源码如下：

```java
    private void basePluginApply(@NonNull Project project) {
      	//核心三个方法
				configureProject();
      	configureExtension();
				createTasks();
    }
```

##### configureProject()

```
    private void configureProject() {
   			//主构建器类，它提供了处理构建的所有数据，比如 DefaultProductFlavor、DefaultBuildType 以及一些依赖项，在执行的时候使用它们特定的构建步骤
        AndroidBuilder androidBuilder = new AndroidBuilder();
				//...
        // 依赖 Java 插件
        project.getPlugins().apply(JavaBasePlugin.class);
				//Plugin 的全局变量
        new GlobalScope（）
    }
```

##### configureExtension()

```java
    private void configureExtension() {
        ObjectFactory objectFactory = project.getObjects();
      	//创建 BuildType、ProductFlavor、SigningConfig、BaseVariantOutput 的容器
        final NamedDomainObjectContainer<BuildType> buildTypeContainer =
                project.container());
        final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer =
                project.container());
        final NamedDomainObjectContainer<SigningConfig> signingConfigContainer =
                project.container());

        final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
                project.container(BaseVariantOutput.class);

        project.getExtensions().add("buildOutputs", buildOutputs);

        sourceSetManager = new SourceSetManager();
				//创建 BaseExtension
        extension = createExtension();

        globalScope.setExtension(extension);

        variantFactory = createVariantFactory(globalScope, extension);
				//创建 TaskManager
        taskManager = createTaskManager();
				//创建 VariantManager
        variantManager = new VariantManager();

        registerModels(registry, globalScope, variantManager, extension, extraModelInfo);
        variantFactory.createDefaultComponents(
                buildTypeContainer, productFlavorContainer, signingConfigContainer);
    }
```

##### createTask()

```java
    private void createTasks() {
      	//注册一些默认 Task，比如 uninstallAll、deviceCheck、connectedCheck 等
        taskManager.createTasksBeforeEvaluate()

        createAndroidTasks();
    }

    final void createAndroidTasks() {
      	//...
        List<VariantScope> variantScopes = variantManager.createAndroidTasks();
    }
```

然后就一直到了 TaskManager#createTasksForVariantScope 方法，它是一个抽象方法，我们看它在 ApplicationTaskManager 中的实现：

```java
    public void createTasksForVariantScope(@NonNull final VariantScope variantScope) {
        createAnchorTasks(variantScope);
      	//checkXxxManifest 检查 Manifest 文件存在、路径
        createCheckManifestTask(variantScope);

        handleMicroApp(variantScope);

        //把依赖放到 TransformManager 中
        createDependencyStreams(variantScope);

        // Add a task to publish the applicationId.
        createApplicationIdWriterTask(variantScope);

        // 合并 manifest
        createMergeApkManifestsTask(variantScope);

        // Add a task to create the res values
        createGenerateResValuesTask(variantScope);

        // Add a task to compile renderscript files.
        createRenderscriptTask(variantScope);

        // 合并 resource
        createMergeResourcesTask(
                variantScope,
                true,
                Sets.immutableEnumSet(MergeResources.Flag.PROCESS_VECTOR_DRAWABLES));

        // 合并 assets 文件夹
        createMergeAssetsTask(variantScope);

        // 生成 BuildConfig.java 文件
        createBuildConfigTask(variantScope);

        // Add a task to process the Android Resources and generate source files
        createApkProcessResTask(variantScope);

        // Add a task to process the java resources
        createProcessJavaResTask(variantScope);

        createAidlTask(variantScope);

        // Add external native build tasks
        createExternalNativeBuildJsonGenerators(variantScope);
        createExternalNativeBuildTasks(variantScope);

        // Add a task to merge the jni libs folders
        createMergeJniLibFoldersTasks(variantScope);

        // Add feature related tasks if necessary
        if (variantScope.getType().isBaseModule()) {
            // Base feature specific tasks.
            taskFactory.register(new FeatureSetMetadataWriterTask.CreationAction(variantScope));

            createValidateSigningTask(variantScope);
            // Add a task to produce the signing config file.
            taskFactory.register(new SigningConfigWriterTask.CreationAction(variantScope));

            if (extension.getDataBinding().isEnabled()) {
                // Create a task that will package the manifest ids(the R file packages) of all
                // features into a file. This file's path is passed into the Data Binding annotation
                // processor which uses it to known about all available features.
                //
                // <p>see: {@link TaskManager#setDataBindingAnnotationProcessorParams(VariantScope)}
                taskFactory.register(
                        new DataBindingExportFeatureApplicationIdsTask.CreationAction(
                                variantScope));

            }
        } else {
            // Non-base feature specific task.
            // Task will produce artifacts consumed by the base feature
            taskFactory.register(
                    new FeatureSplitDeclarationWriterTask.CreationAction(variantScope));
            if (extension.getDataBinding().isEnabled()) {
                // Create a task that will package necessary information about the feature into a
                // file which is passed into the Data Binding annotation processor.
                taskFactory.register(
                        new DataBindingExportFeatureInfoTask.CreationAction(variantScope));
            }
            taskFactory.register(new MergeConsumerProguardFilesTask.CreationAction(variantScope));
        }

        // Add data binding tasks if enabled
        createDataBindingTasksIfNecessary(variantScope, MergeType.MERGE);

        // Add a compile task
        createCompileTask(variantScope);

        createStripNativeLibraryTask(taskFactory, variantScope);


        if (variantScope.getVariantData().getMultiOutputPolicy().equals(MultiOutputPolicy.SPLITS)) {
            if (extension.getBuildToolsRevision().getMajor() < 21) {
                throw new RuntimeException(
                        "Pure splits can only be used with buildtools 21 and later");
            }

            createSplitTasks(variantScope);
        }


        TaskProvider<BuildInfoWriterTask> buildInfoWriterTask =
                createInstantRunPackagingTasks(variantScope);
        createPackagingTask(variantScope, buildInfoWriterTask);

        // Create the lint tasks, if enabled
        createLintTasks(variantScope);

        taskFactory.register(new FeatureSplitTransitiveDepsWriterTask.CreationAction(variantScope));

        createDynamicBundleTask(variantScope);
    }
```

#### 参考

[Apk 打包流程梳理](https://juejin.im/entry/58b78d1b61ff4b006cd47e5b)

[Android App 编译流程详解](https://gitbook.cn/gitchat/activity/5cb3f38f6f093c03b1bfe0f5)
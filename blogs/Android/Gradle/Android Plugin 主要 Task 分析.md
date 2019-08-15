---
Android Plugin 主要 Task 分析
---

> 本文摘自：[Android Gradle Plugin 主要 Task 分析](https://github.com/5A59/android-training/blob/master/gradle/android_gradle_plugin-%E4%B8%BB%E8%A6%81task%E5%88%86%E6%9E%90.md)
>
> 感谢美团大佬 [@5A59](https://github.com/5A59) 的分享！

### 目录

1. 打包流程
2. Task 对应实现类
3. 如何去读 Task 的代码

### 打包流程

这次，我们以 Task 的维度来看 APK 的打包流程。在项目的根目录执行以下命令：

```
./gradlew :app:assembleDebug --console=plain
```

输出结果为：

```
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:compileDebugAidl NO-SOURCE
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:checkDebugManifest UP-TO-DATE
> Task :app:generateDebugBuildConfig UP-TO-DATE
> Task :app:prepareLintJar UP-TO-DATE
> Task :app:generateDebugSources UP-TO-DATE
> Task :app:javaPreCompileDebug UP-TO-DATE
> Task :app:mainApkListPersistenceDebug UP-TO-DATE
> Task :app:generateDebugResValues UP-TO-DATE
> Task :app:generateDebugResources UP-TO-DATE
> Task :app:mergeDebugResources UP-TO-DATE
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
> Task :app:processDebugManifest UP-TO-DATE
> Task :app:processDebugResources UP-TO-DATE
> Task :app:compileDebugJavaWithJavac UP-TO-DATE
> Task :app:compileDebugSources UP-TO-DATE
> Task :app:mergeDebugShaders UP-TO-DATE
> Task :app:compileDebugShaders UP-TO-DATE
> Task :app:generateDebugAssets UP-TO-DATE
> Task :app:mergeDebugAssets UP-TO-DATE
> Task :app:checkDebugDuplicateClasses UP-TO-DATE
> Task :app:mergeExtDexDebug UP-TO-DATE
> Task :app:mergeLibDexDebug UP-TO-DATE
> Task :app:transformClassesWithDexBuilderForDebug UP-TO-DATE
> Task :app:mergeProjectDexDebug UP-TO-DATE
> Task :app:validateSigningDebug UP-TO-DATE
> Task :app:signingConfigWriterDebug UP-TO-DATE
> Task :app:mergeDebugJniLibFolders UP-TO-DATE
> Task :app:transformNativeLibsWithMergeJniLibsForDebug UP-TO-DATE
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:transformResourcesWithMergeJavaResForDebug UP-TO-DATE
> Task :app:packageDebug UP-TO-DATE
> Task :app:assembleDebug UP-TO-DATE
```

上面就是打包一个 APK 需要的 Task。

### Task 对应实现类

我们先看看每个 task 都是做什么的，以及其对应的实现类。

先回忆一下，我们在前面分析 Android Plugin 主要流程时说过，task 的实现可以在 TaskManager 里找到，创建 Task 的方法主要是两个，TaskManager.createTasksBeforeEvaluate() 和 ApplicationTaskManager.createTasksForVariantScope()，所有这些 Task 的实现，也是在这两个类就可以找到，下面列出了各个 Task 的作用及实现类。

| Task                                                 | 对应实现类                  | 作用                                                         |
| ---------------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| preBuild                                             |                             | 空 Task，只做锚点使用                                        |
| preDebugBuild                                        |                             | 空 Task，只做锚点使用，与 preBuild 区别是这个 task 是 variant 的锚点 |
| compileDebugAidl                                     | AidlCompile                 | 处理 aidl                                                    |
| compileDebugRenderscript                             | RendersciptCompiler         | 处理 renderscript                                            |
| checkDebugManifest                                   | CheckManifest               | 检查 manifest 是否存在                                       |
| generateDebugBuildConfig                             | GenerateBuildConfig         | 生成 BuildConfig.java                                        |
| prepareLintJar                                       | PrepareLintJar              | 拷贝 lint jar 包到指定位置                                   |
| generateDebugResValues                               | GenerateResValues           | 生成 res values generated.xml                                |
| generateDebugResources                               |                             | 空 Task，锚点                                                |
| mergeDebugResources                                  | MergeResources              | 合并资源文件                                                 |
| createDebugCompatibleScreenManifest                  | CompatibleScreensManifest   | manifest 文件中生成 compatible-screens，指定屏幕适配         |
| processDebugManifest                                 | MergeManifests              | 合并 manifest 文件                                           |
| splitsDiscoveryTaskDebug                             | SplitsDiscovery             | 生成 split-list.json，用于 apk 分包                          |
| processDebugResources                                | ProcessAndroidResources     | aapt 打包资源                                                |
| generateDebugSources                                 |                             | 空 Task，锚点                                                |
| javaPreCompileDebug                                  | JavaPreCompileTask          | 生成 annotationProcessors.json 文件                          |
| compileDebugJavaWithJavac                            | AndroidJavaCompile          | 编译 java 文件                                               |
| compileDebugNdk                                      | NdkCompile                  | 编译 ndk                                                     |
| compileDebugSources                                  |                             | 空 Task，锚点                                                |
| mergeDebugShaders                                    | MergeSourceSetFolders       | 合并 shader文件                                              |
| compileDebugShaders                                  | ShaderCompile               | 编译 shaders                                                 |
| generateDebugAssets                                  |                             | 空 Task，锚点                                                |
| mergeDebugAssets                                     | MergeSourceSetFolders       | 合并 assets 文件                                             |
| transformClassesWithDexBuilderForDebug               | DexArchiveBuilderTransform  | class 打包 dex                                               |
| transformDexArchiveWithExternalLibsDexMergerForDebug | ExternalLibsMergerTransform | 打包第三方库的 dex，在 dex 增量的时候就不需要再 merge 了，节省时间 |
| transformDexArchiveWithDexMergerForDebug             | DexMergerTransform          | 打包最终的 dex                                               |
| mergeDebugJniLibFolders                              | MergeSouceSetFolders        | 合并 jni lib 文件                                            |
| transformNativeLibsWithMergeJniLibsForDebug          | MergeJavaResourcesTransform | 合并 jnilibs                                                 |
| transformNativeLibsWithStripDebugSymbolForDebug      | StripDebugSymbolTransform   | 去掉 native lib 里的 debug 符号                              |
| processDebugJavaRes                                  | ProcessJavaResConfigAction  | 处理 java res                                                |
| transformResourcesWithMergeJavaResForDebug           | MergeJavaResourcesTransform | 合并 java res                                                |
| validateSigningDebug                                 | ValidateSigningTask         | 验证签名                                                     |
| packageDebug                                         | PackageApplication          | 打包 apk                                                     |
| assembleDebug                                        |                             | 空 Task，锚点                                                |

### 如何去读 Task 的代码

在 Gradle Plugin 中的 Task 主要有三种，一种是普通的 task，一种是增量的 task，一种是 transform，下面分别看下这三种 task 怎么去读。

如何去读？

1. 看 Task 继承的父类，一般来说，会继承 DefaultTask、IncrementalTask
2. 看 @TaskAction 注解的方法，此方法就是这个 Task 做的事情

如何读 IncrementalTask？

我们先看看这个类，这个类表示的是增量 Task，什么是增量呢？是相对于全量来说的，全量我们可以理解为调用 clean 以后第一次编译的过程，这个就是全量编译，之后修改了代码或者资源文件，再次编译，就是增量编译。

其中比较重要的有以下几个方法：

```java
public abstract class IncrementalTask extends BaseTask {
    // ...
    @Internal
    protected boolean isIncremental() { 
        // 是否需要增量，默认是 false
        return false;
    }

    // 需要子类实现，全量的时候执行的任务
    protected abstract void doFullTaskAction() throws Exception;

    // 增量的时候执行的任务，默认是什么都不执行，参数是增量的时候修改过的文件
    protected void doIncrementalTaskAction(Map<File, FileStatus> changedInputs) throws Exception {
    }

    @TaskAction
    void taskAction(IncrementalTaskInputs inputs) throws Exception {
        // 判断是否是增量
        if(this.isIncremental() && inputs.isIncremental()) { 
            this.doIncrementalTaskAction(this.getChangedInputs(inputs));
        } else {
            this.getProject().getLogger().info("Unable do incremental execution: full task run");
            this.doFullTaskAction();
        }
    }

    // 获取修改的文件
    private Map<File, FileStatus> getChangedInputs(IncrementalTaskInputs inputs) {
        Map<File, FileStatus> changedInputs = Maps.newHashMap();
        inputs.outOfDate((change) -> {
            FileStatus status = change.isAdded()?FileStatus.NEW:FileStatus.CHANGED;
            changedInputs.put(change.getFile(), status);
        });
        inputs.removed((change) -> {
            FileStatus var10000 = (FileStatus)changedInputs.put(change.getFile(), FileStatus.REMOVED);
        });
        return changedInputs;
    }
}
```

简单介绍了 IncrementalTask 之后，我们这里强调一下，如何去读一个增量 Task 的代码，主要有四步：

1. 首先这个 Task 要继承 IncrementalTask
2. 其次看 isIncremental 方法，如果返回 true，说明支持增量，返回 false 则不支持
3. 然后看 doFullTaskAction 方法，是全量的时候执行的操作
4. 最后看 doIncrementalTaskAction 方法，这里是增量的时候执行的操作

如何读 Transform？

1. 继承自 Transfrom
2. 看其 transform 方法的实现

### 重点 Task 实现分析

上面每个 task 已经简单说明了具体做什么以及对应的实现类，下面选了几个比较重要的来分析一下其实现。为什么分析这几个呢？这几个代表了 gradle 自动生成代码、资源的处理，以及 dex 的处理，算是 apk 打包过程中比较重要的几环。

generateDebugBuildConfig

processDebugManifest

mergeDebugResources

processDebugResources

transformClassesWithDexBuilderForDebug

transformDexArchiveWithExternalLibsDexMergerForDebug

transformDexArchivesWithDexMergerForDebug

分析过程主要下面几个步骤，整体实现图、调用链路以及重要代码分析。

#### generateDebugBuildConfig

实现类：

GenerateBuildConfig。

整体实现图：

![generateDebugBuildConfig.png](https://i.loli.net/2019/08/15/lC5I3NmgsZR7oKf.png)

代码调用链路：

```
GenerateBuildConfig.generate -> BuildConfigGenerator.generate -> JavaWriter
```

主要代码分析：

在 GenerateBuildConfig 中，主要生成代码的步骤如下：

1. 生成 BuildConfigGenerator
2. 添加默认的属性，包括 DEBUG、APPLICATION_ID、FLAVOR、VERSION_CODE、VERSION_NAME
3. 添加自定义属性
4. 调用 JavaWrite 生成 BuildConfig.java 文件

```java
// GenerateBuildConfig.generate()  
@TaskAction
void generate() throws IOException {
    // ...
    BuildConfigGenerator generator = new BuildConfigGenerator(
            getSourceOutputDir(),
            getBuildConfigPackageName());
    // 添加默认的属性，包括 DEBUG，APPLICATION_ID，FLAVOR，VERSION_CODE，VERSION_NAME
    generator
            .addField(
                    "boolean",
                    "DEBUG",
                    isDebuggable() ? "Boolean.parseBoolean(\"true\")" : "false")
            .addField("String", "APPLICATION_ID", '"' + appPackageName.get() + '"')
            .addField("String", "BUILD_TYPE", '"' + getBuildTypeName() + '"')
            .addField("String", "FLAVOR", '"' + getFlavorName() + '"')
            .addField("int", "VERSION_CODE", Integer.toString(getVersionCode()))
            .addField(
                    "String", "VERSION_NAME", '"' + Strings.nullToEmpty(getVersionName()) + '"')
            .addItems(getItems()); // 添加自定义属性

    List<String> flavors = getFlavorNamesWithDimensionNames();
    int count = flavors.size();
    if (count > 1) {
        for (int i = 0; i < count; i += 2) {
            generator.addField(
                    "String", "FLAVOR_" + flavors.get(i + 1), '"' + flavors.get(i) + '"');
        }
    }

    // 内部调用 JavaWriter 生成 java 文件
    generator.generate();
}
```

#### mergeDebugResources

实现类：

MergeResources

整体实现图：

![mergeDebugResources.png](https://i.loli.net/2019/08/15/NBpbjuHMoAyY2EX.png)

调用链路：

```
MergeResources.doFullTaskAction -> ResourceMerger.mergeData -> MergedResourceWriter.end -> QueueableAapt2.compile -> 
```


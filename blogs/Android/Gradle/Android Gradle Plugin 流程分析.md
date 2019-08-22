---
Android Gradle Plugin 主要流程分析
---

### 前言

> 本文摘自：[Android Gradle Plugin 插件主要流程](https://github.com/5A59/android-training/blob/master/gradle/android_gradle_plugin-%E4%B8%BB%E8%A6%81%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90.md)
>
> 感谢美团大佬 [@5A59](https://github.com/5A59) 的分享！

### 概述

![Android Plugin 流程分析概述.png](https://i.loli.net/2019/08/14/Ubpnexm3cK5yOPW.png)

### 准备工作

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

### 配置项目

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

### 配置 Extension

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

### 创建不依赖 flavor 的 Task

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

### 创建构建 Task

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

创建 task 的时候，会先通过 populateVariantDataList 生成 flavor 相关的数据结构，然后调用 createTasksForVariantData 创建 flavor 对应的 task。

分别看下这两个方法所做的事情：

1. populateVariantDataList

   在方法里，会先根据 flavor 和 dimension 创建对应的组合，存放在 flavorComboList 里，之后调用 createVariantDataForProductFlavors 创建对应的 VariantData。

   其中重要的几个方法：

   ```java
   // 创建 flavor 和 dimension 的组合
   List<ProductFlavorCombo<CoreProductFlavor>> flavorComboList =
                       ProductFlavorCombo.createCombinations(
                               flavorDimensionList,
                               flavorDsl);
   // 为每个组合创建 VariantData
   for (ProductFlavorCombo<CoreProductFlavor>  flavorCombo : flavorComboList) {
       //noinspection unchecked
       createVariantDataForProductFlavors(
               (List<ProductFlavor>) (List) flavorCombo.getFlavorList());
   }
   ```

   创建出来的 VariantData 都是 BaseVariantData 的子类，里面保存了一些 Task，可以看一下 BaseVariantData 里的一些重要的结构，对 BaseVariantData 有个大概的了解。

   ```java
   public abstract class BaseVariantData implements TaskContainer {
       private final GradleVariantConfiguration variantConfiguration;
       private VariantDependencies variantDependency;
       private final VariantScope scope;
       public Task preBuildTask;
       public Task sourceGenTask;
       public Task resourceGenTask; // 资源处理
       public Task assetGenTask;
       public CheckManifest checkManifestTask; // 检测manifest
       public AndroidTask<PackageSplitRes> packageSplitResourcesTask; // 打包资源
       public AndroidTask<PackageSplitAbi> packageSplitAbiTask;
       public RenderscriptCompile renderscriptCompileTask; 
       public MergeResources mergeResourcesTask; // 合并资源
       public ManifestProcessorTask processManifest; // 处理 manifest
       public MergeSourceSetFolders mergeAssetsTask; // 合并 assets
       public GenerateBuildConfig generateBuildConfigTask; // 生成 BuildConfig
       public GenerateResValues generateResValuesTask;
       public Sync processJavaResourcesTask;
       public NdkCompile ndkCompileTask; // ndk 编译
       public JavaCompile javacTask; 
       public Task compileTask;
       public Task javaCompilerTask; // java 文件编译
       // ...
   }
   ```

   VariantData 里保存了很多 Task，下一步就是要创建这些 task。

2. createTasksForVariantData

   创建完 variant 数据，就要给每个 variantData 创建对应的 task，对应的 task 有 assembleXxxTask、prebuildXxx、generateXxxSource、generateXxxResources、generateXxxAssets、processXxxManifest 等等，重点关注几个方法：

   ```java
   VariantManager.createAssembleTaskForVariantData() // 创建 assembleXXXTask
   TaskManager.createTasksForVariantScope() // 是一个抽象类，具体实现在 ApplicationTaskManager.createTasksForVariantScope()
   TaskManager.createPostCompilationTasks() // 创建 .class to dex 的 task， 创建 transformTask，我们创建的 transform 就是这个阶段添加进来的，是在 addCompileTask 里调用的
   
   // createTasksForVariantScope 是一个抽象方法，具体实现在子类中，可以看一下 ApplicationTaskManager.createTasksForVariantScope()
   // createTasksForVariantScope 里的实现，如果在业务中有需要查看相关 task 源码时，可以来这里找
   void createTasksForVariantScope() {
       this.createCheckManifestTask(tasks, variantScope); // 检测 manifest
       this.handleMicroApp(tasks, variantScope);
       this.createDependencyStreams(tasks, variantScope);
       this.createApplicationIdWriterTask(tasks, variantScope); // application id 
       this.createMergeApkManifestsTask(tasks, variantScope); // 合并 manifest
       this.createGenerateResValuesTask(tasks, variantScope);
       this.createRenderscriptTask(tasks, variantScope);
       this.createMergeResourcesTask(tasks, variantScope, true); // 合并资源文件
       this.createMergeAssetsTask(tasks, variantScope, (BiConsumer)null); // 合并 assets
       this.createBuildConfigTask(tasks, variantScope); // 生成 BuildConfig
       this.createApkProcessResTask(tasks, variantScope); // 处理资源
       this.createProcessJavaResTask(tasks, variantScope);
       this.createAidlTask(tasks, variantScope); // 处理 aidl
       this.createShaderTask(tasks, variantScope);
       this.createNdkTasks(tasks, variantScope); // 处理 ndk
       this.createExternalNativeBuildJsonGenerators(variantScope);
       this.createExternalNativeBuildTasks(tasks, variantScope);
       this.createMergeJniLibFoldersTasks(tasks, variantScope); // 合并 jni
       this.createDataBindingTasksIfNecessary(tasks, variantScope); // 处理 databinding
       this.addCompileTask(tasks, variantScope); 
       createStripNativeLibraryTask(tasks, variantScope);
       this.createSplitTasks(tasks, variantScope);
       this.createPackagingTask(tasks, variantScope, buildInfoWriterTask); // 打包 apk
       this.createLintTasks(tasks, variantScope); // lint 
   }
   
   // createPostCompilationTasks 实现:
   // 处理 Android Transform
   void createPostCompilationTasks() {
       for (int i = 0, count = customTransforms.size(); i < count; i++) {
           Transform transform = customTransforms.get(i);
           // TransformManager.addTransform 实际上是为 transform 创建了一个 Task
           transformManager
                   .addTransform(tasks, variantScope, transform)
                   .ifPresent(t -> {
                       if (!deps.isEmpty()) {
                           t.dependsOn(tasks, deps);
                       }
                       // if the task is a no-op then we make assemble task depend on it.
                       if (transform.getScopes().isEmpty()) {
                           variantScope.getAssembleTask().dependsOn(tasks, t);
                       }
                   });
       }
   }
   ```

### 总结

![plugin-summary.png](https://i.loli.net/2019/08/22/1zcRLBIFxNrQqDg.png)

这里总结几个要点：

1. com.android.application 入口类是 AppPlugin，但大部分工作都是在 BasePlugin 里完成的。
2. build.gradle 里见到的 android {} dsl 是在 BasePlugin.configureExtension() 里声明的。
3. 主要的 task 是在 BasePlugin.createAndroidTask 里生成的。
4. 主要 Task 的实现可以在 TaskManager 中找到。
5. Transform 会转化为 TransformTask。


---
Gradle 练习之一 --- 输出项目第三方库以及本地依赖库的权限信息
---

#### 前言

标题看起来可能有点恍惚，简单来说就是对于 app module 下面的依赖，即：

```groovy
dependencies {
		// 第三方库
    implementation 'com.android.support:appcompat-v7:28.0.0'
    // 本地依赖库
    implementation project(":mylibrary")
}
```

获取它们的 Manifest 文件里面的权限信息。

输出结果示例：

```groovy
/**
 * Path: /Users/omooo/AndroidStudioProjects/Projects/MyApplication/app/src/main/AndroidManifest.xml
 * ModuleName: app
 * 
 * <uses-permission android:name="android.permission.INTERNET" />
 * <uses-permission android:name="android.permission.CALL_PHONE" />
 * <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
 * <uses-permission android:name="android.permission.RECEIVE_SMS" />
 */


/**
 * Path: /Users/omooo/AndroidStudioProjects/Projects/MyApplication/mylibrary/src/main/AndroidManifest.xml
 * ModuleName: mylibrary
 * 
 * <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
 * <uses-permission android:name="android.permission.INTERNET" />
 */

```

#### 入门

较早之前，我们 App 因为私自获取用户敏感权限被警告了。组长就让我看看能不能获取每个 module 的权限信息，然后格式化输出。当时心想还不是简单的一批，终于到了秀操作的时候了。

写一个 writePermissionToFile.gradle 脚本，放在根目录，如下：

```groovy
import java.util.regex.Matcher
import java.util.regex.Pattern

boolean firstWrite = true

project.afterEvaluate {

    if (!firstWrite || !project.hasProperty('android')) return
    def extension = project.android
    if (extension.class.name.contains('BaseAppModuleExtension')) {
        project.android.applicationVariants.all { variant ->
            writeFile(variant.getCheckManifest().manifest, project.name)
        }
    } else if (extension.class.name.contains('LibraryExtension')) {
        project.android.libraryVariants.all { variant ->
            writeFile(variant.getCheckManifest().manifest, project.name)
        }
    }
    firstWrite = false
}

static def writeFile(File manifestFile, String moduleName) {
    def fileT = new File("allPermissionOfModuleFile.txt")
    if (!fileT.exists()) {
        fileT.createNewFile()
    }
    if (fileT.getText().contains(manifestFile.path)) return
    Pattern pattern = Pattern.compile("<uses-permission.+./>")
    Matcher matcher = pattern.matcher(manifestFile.getText())
    String firstMatch = ""
    if (!matcher.find()) {
        return
    } else {
        firstMatch = matcher.group()
    }
    fileT.append("/**" + "\n")
    fileT.append(" * Path: " + manifestFile.path + "\n")
    fileT.append(" * ModuleName: " + moduleName + "\n")
    fileT.append(" * " + "\n")
    fileT.append(" * " + firstMatch + "\n")
    while (matcher.find()) {
        fileT.append(" * " + matcher.group() + "\n")
    }
    fileT.append(" */" + "\n")
    fileT.append("\n")
    fileT.append("\n")
}
```

然后在项目根目录的 build.gradle 中添加：

```groovy
boolean enablePrintPers = true
subprojects {
    if (enablePrintPers) {
        apply from: '../writePermissionToFile.gradle'
    }
}
```

同步一下，在项目根目录就会生成一个 allPermissionOfModuleFile，内容如前言所示。

总体来说还是很简单的，甚至你还可以拿到 merge 之后的 Manifest 文件，然后剔除指定权限，代码如下：

```groovy
/**
 * 以下注释掉的代码是个骚操作
 * 它可以在打包的时候移除掉指定的权限
 * 注意：这个时候的 manifestFile 已经是 merge 之后的了
 * 其实就是字符串替换
 */
variant.outputs.each { output ->
    output.processResources.doFirst { pm ->
        String manifestPath = output.processResources.manifestFile
        def file = new File(manifestPath)
        def manifestContent = file.getText()
        String permission = '<uses-permission android:name="android.permission.RECEIVE_SMS" />'
        manifestContent = manifestContent.replace(permission, '')
        file.delete()
        new File(manifestPath).write(manifestContent)
    }
}
```

开开心心的给我们组长汇报，组长说要是能把第三方库里面的权限也一起拿了就好了。

我...

#### 进阶

那怎么才能在 Gradle 构建的时候，拿到第三方库呢？其实拿到第三方库的下载的绝对路径就好了。

其实也很简单，直接在 app module 的 build.gradle 添加：

```groovy
configurations.implementation.setCanBeResolved(true)
configurations.implementation.resolve().each {
    if (it.name.endsWith(".aar")) {
        println(it.name)
        println(it.path)

        File aarFile = it
        copy {
            from zipTree(aarFile)
            into getBuildDir().path + "/destAAR/" + aarFile.name
        }
        String manifestFilePath = getBuildDir().path + "/destAAR/" + aarFile.name + "/AndroidManifest.xml"
        writeFile(new File(manifestFilePath), it.name)
    }
}
```

这里的拿到的路径如下：

```
appcompat-v7-28.0.0.aar
/Users/omooo/.gradle/caches/modules-2/files-2.1/com.android.support/appcompat-v7/28.0.0/132586ec59604a86703796851a063a0ac61f697b/appcompat-v7-28.0.0.aar
constraint-layout-1.1.3.aar
/Users/omooo/.gradle/caches/modules-2/files-2.1/com.android.support.constraint/constraint-layout/1.1.3/2f88a748b5e299029c65a126bd718b2c2ac1714/constraint-layout-1.1.3.aar
support-fragment-28.0.0.aar
/Users/omooo/.gradle/caches/modules-2/files-2.1/com.android.support/support-fragment/28.0.0/f21c8a8700b30dc57cb6277ae3c4c168a94a4e81/support-fragment-28.0.0.aar
animated-vector-drawable-28.0.0.aar
```

可以看到，是 gradle caches 里面下载好的缓存的第三方库的绝对路径，所以也就有一个致命的问题，那就是 app module 不能有 implementation project(":mylibrary") 这种本地源码依赖的方式，不然以上代码执行到这一步就会直接挂掉。

```groovy
configurations.implementation.resolve().each {}
```

那能不能在执行这个把 implementation project(":mylibrary") 这种方式依赖的给过滤掉呢？毕竟这种方式依赖的我们在初级阶段就做好了。

但是很可惜，我并没找到办法。

#### 再进阶

这里就要祭出杀手锏了，来自 [@海海](https://github.com/HiWong) 大佬的思路，直接抄 Gradle 源码。

既然是抄源码，那就要首先在 app module 依赖一下 Gralde 源码了：

```java
implementation 'com.android.tools.build:gradle:3.4.2'
```

然后在 app module 的 build.gralde 添加：

```groovy
import com.android.build.gradle.internal.dependency.ArtifactCollectionWithExtraArtifact
import com.android.build.gradle.internal.publishing.AndroidArtifacts
import org.gradle.internal.component.local.model.OpaqueComponentArtifactIdentifier

project.afterEvaluate {
    boolean isFirst = true
    project.android.applicationVariants.all { variant ->
        if (!isFirst) return
        ArtifactCollection manifests = variant.getVariantData().getScope().getArtifactCollection(
                AndroidArtifacts.ConsumedConfigType.RUNTIME_CLASSPATH,
                AndroidArtifacts.ArtifactScope.ALL,
                AndroidArtifacts.ArtifactType.MANIFEST)
        final Set<ResolvedArtifactResult> artifacts = manifests.getArtifacts()

        for (ResolvedArtifactResult artifact : artifacts) {
            println(artifact.getFile().absolutePath)
            println(getArtifactName(artifact))
            writeFile(artifact.getFile(), getArtifactName(artifact))
        }
        isFirst = false
    }
}

static String getArtifactName(ResolvedArtifactResult artifact) {
    ComponentIdentifier id = artifact.getId().getComponentIdentifier()
    if (id instanceof ProjectComponentIdentifier) {
        return ((ProjectComponentIdentifier) id).getProjectPath()

    } else if (id instanceof ModuleComponentIdentifier) {
        ModuleComponentIdentifier mID = (ModuleComponentIdentifier) id
        return mID.getGroup() + ":" + mID.getModule() + ":" + mID.getVersion()

    } else if (id instanceof OpaqueComponentArtifactIdentifier) {
        return id.getDisplayName()
    } else if (id instanceof ArtifactCollectionWithExtraArtifact.ExtraComponentIdentifier) {
        return id.getDisplayName()
    } else {
        throw new RuntimeException("Unsupported type of ComponentIdentifier")
    }
}
```

代码是参考 ProcessApplicationManifest 这个 Task 来写的，如果在较低版本的 Gradle 则名为 MergeTasks，看名字就知道这个 Task 是干什么的了。

剩下还有一个需要完善的，那就是把整个流程封装成一个 Task 就好了，这就需要涉及到动态添加依赖了，毕竟这种方法必须要依赖 Gradle 源码了。

（逃～
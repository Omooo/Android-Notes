---
Gralde Plugin 实践之 TinyPng Plugin
---

#### 前言

在上一篇文章中，我们熟悉了如何去实现一个[自定义的 Gradle Plugin](https://github.com/Omooo/Android-Notes/blob/master/blogs/Android/Gradle/Gradle%20Plugin.md)，本来按照计划这篇文章是讲 Transform API，但是考虑到学完新知识最好能实践一下，之前也讲到可以利用 TinyPng 在构建项目的时候批量压缩 res 下的所有 png 图片，今天我们就来实践一下，这个并不涉及到 Transform API 的使用，但是需要熟悉 Groovy 一些常见的操作，比如 Extensions、 Json 的解析和生成等，整个项目很简单，代码并不多，大胆 fork [TinyPngPlugin](https://github.com/surpriseprojects/TinyPngPlugin) 吧。

#### 实现方式

我们可以直接利用 TinyPng 给的 API 即可，文档地址为：

[https://tinypng.com/developers/reference/java](https://tinypng.com/developers/reference/java)

核心源码如下：

```java
Tinify.setKey("YOUR_API_KEY");
Tinify.fromFile("unoptimized.png").toFile("optimized.png");
```

也就是说，我们只需要配置 API_KEY 就好了，文件的输入路径和输出路径肯定就是 res 文件夹下的 drawable 或 mipmap 了。不过我们肯定想知道，压缩前后的图片的大小差距，这里可以通过生成 Json 文件来查看。

需要注意的是，如果你想把这个 Plugin 上传到仓库给别人使用，那肯定不能在工程里面写死 API_KEY，这个很好理解，也就是说我们需要用户可以动态配置 API_KEY。这就要牵扯到 Extensions 了。

#### Extensions 

Extensions 即自定义配置项，这是什么呢？我们可以参考 app 模块的 build.gradle 文件：

```java
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "top.omooo.pluginproject"
        minSdkVersion 23
        targetSdkVersion 28
        //...
    }
   	//...
}
```

如何把 compileSdkVersion 和 applicationId 的值解析出来呢？很简单，我们可以直接在 app 模块的 build.gradle 文件添加以下代码（即第一篇文章的第一种方式）：

```groovy
class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project target) {
        target.task("testExtensions") << {
            def android = target['android']
            println(android.compileSdkVersion)
            def defaultConfig = android['defaultConfig']
            println(defaultConfig.applicationId)
        }
    }
}

apply plugin: MyPlugin
```

然后执行 Task 就可以看到输出了。

熟悉了以上操作，这时候我们就可以确定我们的自定义配置项为以下结构：

```groovy
tinyInfo {
    //资源目录
    resourceDir = [
            "app/src/main/res",
            "other_module/src/main/res"
    ]
    resourcePattern = [
            "drawable[a-z-]*",
            "mipmap[a-z-]*"
    ]
    whiteList = [

    ]
    apiKey = "******"
}
```

我们可以为以上的结构写一个 Bean 类，命名为 TinyPngExtension.groovy：

```java
class TinyPngExtension {

    ArrayList<String> resourceDir
    ArrayList<String> resourcePattern
    ArrayList<String> whiteList
    String apiKey

    TinyPngExtension() {
        resourceDir = []
        resourcePattern = []
        whiteList = []
        apiKey = null
    }


    @Override
    public String toString() {
        return "TinyPngExtension.groovy{" +
                "resourceDir=" + resourceDir +
                ", resourcePattern=" + resourcePattern +
                ", whiteList=" + whiteList +
                ", apiKey='" + apiKey + '\'' +
                '}'
    }
}
```

需要注意的是，这里的**变量名一定要和配置项的字段名一致**。

这个 Plugin 的大部分代码都在 Task，所以我们把这个 Task ( TinyPngTask.groovy ) 抽出来：

```java
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction

class TinyPngTask extends DefaultTask {
    TinyPngExtension mTinyPngExtension

    TinyPngTask() {
        mTinyPngExtension = project.tinyInfo
    }

    @TaskAction
    void run() {
        println("Task run~")
        println(mTinyPngExtension.toString())
    }
}
```

注意，这里我们用了一个 @TaskAction 的注解，这个注解表示该方法表示为 Task 的执行实体，方法名随意写都行。

然后我们在 Plugin 里去执行这个 Task ( CrazyPlugin.groovy )：

```groovy
class CrazyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {

        project.extensions.create("tinyInfo", TinyPngExtension)

        project.afterEvaluate {
            project.task("tinyTask", type: TinyPngTask)
        }
    }
}
```

回顾上一篇文章，我们说到，既然我们修改了 Plugin 的代码，就要重新生成 jar 包上传依赖，即执行：

```
./gradlew task uploadArchives
```

然后在 app 模块的 build.gradle 文件里，我们就可以 apply Plugin 并且添加 Extensions 了：

```groovy
apply plugin: 'top.omooo.crazy_plugin'

tinyInfo {
    //资源目录
    resourceDir = [
            "app/src/main/res",
            "other_module/src/main/res"
    ]
    resourcePattern = [
            "drawable[a-z-]*",
            "mipmap[a-z-]*"
    ]
    whiteList = [

    ]
    apiKey = "******"
}
```

然后我们在执行：

```
./gradlew task tinyTask
```

就可以输出我们的自定义配置项了。也就是说，我们写的自定义配置项可以在 Task 中获取的。

#### Json 解析与生成

其实本来我是不想讲这个的，主要是因为太简单了，但是不讲这个好像没什么可写的了 :) 

```java
class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project target) {
        target.task("testJson") << {

            def jsonFile = new File("${project.projectDir}/test.json")
            if (!jsonFile.exists()) {
                jsonFile.createNewFile()
            }
            //把 List 写入 json 文件中
            ArrayList<String> list = ["Omooo", "Tom", "Test"]
            def jsonOutput = new JsonOutput()
            def json = jsonOutput.toJson(list)
            jsonFile.write(jsonOutput.prettyPrint(json), "utf-8")

            //把 json 文件读成 List
            def readList = new JsonSlurper().parse(jsonFile, "utf-8")
            println("${readList.toString()}")
        }
    }
}

apply plugin: MyPlugin
```

完全不用我过多解释了，大家都能看懂。但是这个有什么用嘛？

我们会通过配置项中拿到资源目录，然后遍历资源目录，批量压缩目录下的所有 png 图片，这时候我想把压缩前后的图片大小记录下来，就要用到写 Json 文件；下次压缩的时候，之前压缩过的图片，肯定不希望再次压缩了，所以就需要用到读 Json 文件转成 List，在这 List 的里面的图片就不用压缩了。

#### 结果

这是压缩后的结果 Json 文件：

```xml
[
    {
        "preSize": "6.73KB",
        "md5": "cebfdfd96b51a2bbde7cfa40b7ea2997",
        "postSize": "3.48KB",
        "path": "app/src/main/res/mipmap-xhdpi/ic_launcher_round.png"
    },
		//...
    {
        "preSize": "74.46KB",
        "md5": "d506a51ab02f8e527b2d5b13b24ba9f7",
        "postSize": "16.30KB",
        "path": "app/src/main/res/drawable/thread_lifecycle.png"
    }
]
```

```
//*************//
Task finish，compress 11 files
Before total size: 141400
After total size: 49327
//*************//
```

效果还是很明显的～

#### 代码

[TinyPngPlugin](https://github.com/surpriseprojects/TinyPngPlugin)

本文无限感谢 [https://github.com/waynell/TinyPngPlugin](https://github.com/waynell/TinyPngPlugin) 

其实代码并没有太大差别，毕竟代码就那些。

还记得自己当初 fork 这个库的时候，对于自定义 Plugin 流程不熟悉，还有就是不熟悉 Groovy 语法等一系列原因，一直没有去自己写一遍。当你熟悉了自定义 Plugin 流程后，Groovy 语法并不多，主要的我都讲到了，其实整个流程还是很简单的，还是开头那句话：

放心大胆的去 Fork 吧～
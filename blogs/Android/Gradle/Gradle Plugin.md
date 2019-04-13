---
Gradle Plugin 入门指南
---

#### 前言

首先，先要知道为什么需要自定义 Gradle Plugin 呢？有哪些应用场景呢？

还不是因为可以结合 Transform API 做一些很神奇的事？我们可以在自定义 Plugin 中去注册我们的 Transform，以此来干预打包流程。

Transform 在 Android 中都会被封装成 Task，Task 就是 Gradle 的灵魂。Android 中常见的 Task 有 ProGuard 混淆、MultiDex 重分包等等，我们写的 Transform 是会首先执行的，执行完之后再执行 Android 自带的 Transfrom。因此我们可以利用 TinyPng 在打包的时候批量压缩 res 下的所有 png 图片，甚至可以修改字节码，把项目中的 new Thread 全部替换成 CustomThread 等，Transform API 还是很强大的，本节先讲如何去自定义一个 Gradle Plugin，下一篇会仔细讲 Transform API 的玩法。

笔者刚开始写 Gradle Plugin 的时候，估计和大家一样，网上搜一些文章然后去实践，然鹅各种奇怪问题，自定义 Plugin 本身代码并不多，容易出错的就是配置上的问题，这种问题很难发现，所以本文就是来为大家填坑。

#### 热身

自定义 Gradle Plugin 有三种办法：

1. 直接在 build.gradle 文件中写
2. 新建 buildSrc 工程
3. 新建 Java Library 工程

首先，我们尝试第一种方式去写，每个 Gradle 工程都会有一个 build.gradle 文件，这个文件配置了工程所需的各种配置信息，它是一个 Groovy 文件，所以我们可以直接在文件里面写：

```java
class MyPlugin implements Plugin<Project>{
    @Override
    void apply(Project target) {
        target.task("hello"){
            println('Hello Plguin~')
        }
    }
}

apply plugin: MyPlugin
```

在 Plugin 中我们定义了一个 hello 的 Task，然后 apply 这个 Plugin，现在我们可以打开 Terminal 执行这个 Task：

```
./gradlew task -q hello
```

就输出 Hello Plugin~ 啦。这里 -q 参数表示静默执行，后面就是我们定义的 Task name，还是很简单的。

如果这个你还有问题，就不用往下看了 :)

第二种 buildSrc 方式，也是官方文档中的方式，但是这种方式定义的 Plugin 只能在当前工程使用，如果我想把这个 Plugin 发布出去就不行了，所以我们直接鄙弃这种方式。

第三种扩展性就很强了，所以本文就使用新建 Java Library 方式去自定义 Plugin。

#### 正文

首先新建一个 Java Library 的工程，名叫 "my_plugin"，然后在 my_plugin 工程的 build.gradle 添加一些依赖：

```java
apply plugin: 'java-library'
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    //Gradle Plugin 依赖
    implementation gradleApi()
    //本地发布 Plugin
    implementation localGroovy()
}

sourceCompatibility = "7"
targetCompatibility = "7"
```

这时候其实我们只有一个 Java 包，所以需要新建一个对应的 groovy 包，如下：

![](https://i.loli.net/2019/04/13/5cb12ec91412a.jpg)

MyPlugin 中即是我们的自定义的 Plugin，新建 MyPlugin 文件的时候一定是 Groovy 文件，**注意要有 .groovy 后缀**，代码如下：

```java
package top.omooo.my_plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.task('myPlugin'){
            println('MyPlugin Start ~')
        }
    }
}
```

代码还是一如既往的简单～

图中有几个点需要解释一下：

1. 我新建了一个 resources 文件夹

   这里是用来注册我们自定义的 Plugin，需要注意是，该目录下的**文件夹命名**一定不能错，笔者最开始就是因为把 META-INF 错写成了 META_INF，折腾了好久才发现。

2. top.omooo.my_plugin.properties

   这个文件的内容就是我们的自定义 Plugin，按图上写就行了。不过需要注意的是，文件名为 top.omooo.my_plugin，文件名是可以随意定义的，但是最好能是包名，这就是我们的 Plugin 的 Id，回顾上面，我们依赖了 apply plugin:  'groovy'，这里的 groovy 就是 Plugin Id，所以当我们在外部模块依赖我们的自定义 Plugin 的时候就是：

   ```java
   apply plugin: 'top.omooo.my_plugin'
   ```

但是这时候我们的 Plugin 还不能被 app 模块依赖，因为还没发布呢，那该如何发布呢？

我们可以选择发布到 maven 仓库，也可以在本地发布，不过在发布之前，都要做一件事，那就是指定 Plugin 的标示信息：

```java
apply plugin: 'java-library'
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    //Gradle Plugin 依赖
    implementation gradleApi()
    //本地发布 Plugin
    implementation localGroovy()
}

//Plugin 标示信息
group 'top.omooo.my_plugin'
version '1.0'

//本地发布，发布到根目录的 /repo 文件夹下
uploadArchives{
    repositories {
        mavenDeployer{
            repository(url :uri('../repo'))
        }
    }
}

sourceCompatibility = "7"
targetCompatibility = "7"

```

我们引入一个依赖，其实是有三部分组成的：

```java
compile 'com.android.tools.build:gradle:3.3.2'
//groupId: com.android.tools.build
//artifactId: gradle
//version: 3.3.2
```

但是你可能会问，这里我们为什么没有写 artifactId，因为 artifactId 默认为工程名字，所以我们可以不写，所以如果我们在引入我们的 Plugin 的时候，就需要这样写了：

```java
top.omooo.my_plugin:my_plugin:1.0
```

这时候我们我们还没有真正的发布，需要执行：

```java
./gradlew task uploadArchives
```

执行完后，就可以看到我们根目录生成了一个 repo 文件夹，里面就是我们的插件 jar 包了：

![](https://i.loli.net/2019/04/13/5cb13493e7a44.jpg)

本地有了 Plugin，我们就可以在主工程即 Project 的 build.gradle 引入了：

```java
buildscript {
    repositories {
        google()
        jcenter()
        maven{
            //本地 Plugin 地址，注意只有一个 .
            url uri('./repo')
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.3.2'
        //自定义的 Plugin
        classpath 'top.omooo.my_plugin:my_plugin:1.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        maven{
            url uri('./repo')
        }
        google()
        jcenter()
        
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

```

这是我们就可以在我们依赖的包中看到那个熟悉的身影了，这时候一定要**检查看看我们写的类是不是打入了 jar 包**：

![](https://i.loli.net/2019/04/13/5cb13d3c93d33.jpg)

完成之后，我们就可以在工程下的所有模块依赖这个 Plugin，比如我们可以在 app module 下依赖这个模块：

```java
apply plugin: 'top.omooo.my_plugin'
```

之后，我们肯定会改 MyPlugin 里面的代码的，这时候就要重新上传依赖，用的还是那个命令：

```java
./gradlew task uploadArchives
```

OK，大功告成～
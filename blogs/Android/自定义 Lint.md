---
自定义 Lint
---

![](https://i.loli.net/2019/07/09/5d243bc10f1aa59752.png)

#### 前言

首先，先要仔细想想，什么时候需要用到自定义 Lint 呢？

这就要发挥你的想象力了，我最初是想推 webp，用以替换掉 png，这样就需要开发者每次自觉的将 png 转化为 webp，但是又不可能结对编程，监督他进行转化，所以自定义 Lint 就派上用场了。

以下是项目中的自定义 Lint 实现效果：

![](https://i.loli.net/2019/07/09/5d243d756b7ea70892.png)

以上 Detector 都有什么用呢？

ToastDetector：Android 自带的 Lint 规则，用于提示 Toast 忘记调用 show 方法，在 BuiltinIssueRegistry 类可以查看内置的所有的 Detector，这也是我实现自定义 Lint 的主要参考依据。

SampleCodeDetector：Google Sample 中的 Demo，用于检测文本表达式中是否包含特定的字符串。

PngDetector：用于检测所有 layout 或 java 文件中引用的 png 资源，提示使用 webp。

LogDetector：用于检测使用 Log 类的 i、d、e 等日志输出的方法，提示使用统一的日志工具类。严重程度为 Error，所以默认情况下，会中断编译流程。如果严重程度为 Warning，仅仅是报黄警告。

ThreadDetector：用于检测直接通过 new Thread 创建线程，提示应该使用统一线程池。

**源码地址：**[https://github.com/Omooo/CustomLint](https://github.com/Omooo/CustomLint)

#### 正文

先来看一个简单的自定义 Lint 是怎么样一步一步写出来的，该例子实际上来自 [https://github.com/googlesamples/android-custom-lint-rules](https://github.com/googlesamples/android-custom-lint-rules) ，该 Rep 就一个 SampleCodeDetector，也就是上文中所说的。

自定义 Lint 一共可以分为四步：

##### 第一步：创建 java library 工程

在 build.gradle 文件里添加依赖：

```groovy
compileOnly "com.android.tools.lint:lint-api:26.4.1"
compileOnly "com.android.tools.lint:lint-checks:26.4.1"
```

##### 第二步：创建 Detector

```java
public class SampleDetector extends Detector implements Detector.UastScanner {

    //第一步：定义 ISSUE
    public static final Issue ISSUE = Issue.create(
            "ShortUniqueId",    //唯一 ID
            "Lint Mentions",    //简单描述
            "Blah blah blah.",  //详细描述
            Category.CORRECTNESS,   //问题种类（正确性、安全性等）
            6,  //权重
            Severity.WARNING,   //问题严重程度（忽略、警告、错误）
            new Implementation(     //实现，包括处理实例和作用域
                    SampleCodeDetector.class,
                    Scope.JAVA_FILE_SCOPE));

    //第二步：定义检测类型以及处理逻辑
    //检测类型包括文本表达式、调用相关表达式等
    @Override
    public List<Class<? extends UElement>> getApplicableUastTypes() {
        return Collections.singletonList(ULiteralExpression.class);
    }

    @Override
    public UElementHandler createUastHandler(@NotNull JavaContext context) {
        return new UElementHandler() {
            @Override
            public void visitLiteralExpression(@NotNull ULiteralExpression expression) {
                String string = UastLiteralUtils.getValueIfStringLiteral(expression);
                if (string == null) {
                    return;
                }
                if (string.contains("Omooo") && string.matches(".*\\bOmooo\\b.*")) {
                    //第三步：符合条件，上报 ISSUE
                    context.report(ISSUE, expression, context.getLocation(expression),
                            "This code mentions `Omooo`");
                }
            }
        };
    }
}
```

Lint API 中内置了很多 Scanner：

| Scanner 类型          | Desc                     |
| --------------------- | ------------------------ |
| UastScanner           | 扫描 Java、Kotlin 源文件 |
| XmlScanner            | 扫描 XML 文件            |
| ResourceFolderScanner | 扫描资源文件夹           |
| ClassScanner          | 扫描 Class 文件          |
| BinaryResourceScanner | 扫描二进制资源文件       |

更多请参考：[https://static.javadoc.io/com.android.tools.lint/lint-api/25.3.0/com/android/tools/lint/detector/api/package-summary.html](https://static.javadoc.io/com.android.tools.lint/lint-api/25.3.0/com/android/tools/lint/detector/api/package-summary.html)

注意：

这里需要注意的一点是，如果对应的 ISSUE 严重程度为错误（Severity.ERROR），那么在默认情况下，会中断编译流程，当然，你也可以配置 LintOptions 来抑制 Lint 错误。

##### 第三步：注册 Detector

```java
public class CustomIssueRegistry extends IssueRegistry {

    @NotNull
    @Override
    public List<Issue> getIssues() {
        return Arrays.asList(
                SampleCodeDetector.ISSUE);
    }

    @Override
    public int getApi() {
        return ApiKt.CURRENT_API;
    }
}
```

这里可以注册多个 Detector，目前最新版本的 Lint 内置了 360 种 Detector，都在 BuiltinIssueRegistry 类中，可以作为我们编写自定义 Lint 的最佳参考案例。

##### 第四步：引入自定义 Lint

首先需要在 lint_library 中的 build.gradle 文件中添加，完整代码为：

```groovy
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    compileOnly "com.android.tools.lint:lint-api:26.4.1"
    compileOnly "com.android.tools.lint:lint-checks:26.4.1"
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"

jar {
    manifest {
        attributes("Lint-Registry-v2": "top.omooo.lint_library.CustomIssueRegistry")
    }
}
```

然后就可以在 app module 中的 build.gradle 中引入并使用了：

```groovy
dependencies {
    //...
    lintChecks project(":lint_library")
}
```

#### Lint 进阶

以上，一个简单的自定义 Lint 就写完了，但是有点意犹未尽的感觉。这时候就需要你发挥想象力，想想自己需要什么。你可以参考我给的源码，或者参考 Android 内置的 Lint 源码，看看它们能做什么。

这一小节很重要，但是我并不会给你讲如何去实现某某功能，自己看源码学习，因为真的不难哇。

#### 最后

如果你很懒，很烦每次都敲一遍 ./gradlew lint 去查看 Lint 输出，那么可以把执行 Lint 任务挂载在每次安装 Debug 包之前，即：

```groovy
/**
 * 在执行 assembleDebug Task 之前挂载 lintDebug
 */
project.afterEvaluate {
    def assembleDebugTask = project.tasks.find { it.name == 'assembleDebug' }
    def lintTask = project.tasks.find { it.name == 'lintDebug' }
    assembleDebugTask.dependsOn(lintTask)
}
```


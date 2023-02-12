---
ASM 应用与实践
---

#### 目录

1. 引言
   1. ASM 是什么
   2. ASM 有什么用

2. ASM 的两类 API
3. ASM 示例
   1. 读取 ArrayList 类
   2. 输出特定方法耗时
   3. 删除方法里面的日志语句
   4. 线程重命名
   5. 序列化检查

4. ASM 不能做什么
5. 总结
6. 更多

#### 引言

当我们接触一个新知识点时，一般会有两个疑问，它是什么？它有什么用？

所以在一开始呢，就需要先回答这两个问题。

##### ASM 是什么

简单一句话总结：ASM 是一个 **字节码操作库**。

这句话解读下来有三点：

1. 字节码：ASM 操作的目标是字节码，字节码即 JVM 执行的一种指令格式，它既可以来自 Java 代码编译，也可以来自于 Kotlin、Grovvy 代码编译，只要是符合 JVM 规范的字节码即可。
2. 操作：即增删改查。ASM 可以修改已有类的字节码，或者直接生成二进制格式的类。
3. 库：ASM 是一个工具类库，开箱即用。

ASM 依赖库体积小、字节码操作速度快，这也是它有别于其他字节码操作库的原因。

##### ASM 有什么用

ASM 的应用极其广泛，Java 中 [Lambda 表达式调用点](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/lang/invoke/InnerClassLambdaMetafactory.java) 的生成、[反射的动态实现](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/e5b1823a897e/src/share/classes/sun/reflect/MethodAccessorGenerator.java)，Android 中 [BuildConfig](https://github.com/jrodbx/agp-sources/blob/d33681de938188ab30a173b626e6dca6949b13c1/7.2.2/com.android.tools.build/gradle/com/android/build/gradle/tasks/GenerateBuildConfig.kt) 类的生成等，都是通过 ASM 来实现了。

除此之外，一些 Android 的质量优化框架 [Booster](https://github.com/didi/booster)、[ByteX](https://github.com/bytedance/ByteX)、[Matrix](https://github.com/Tencent/matrix) 等，也都是使用 ASM 来操作字节码。了解这些 APM 库的实现原理，也是需要我们首先熟悉 ASM 的使用。

这其实也我分享 ASM 的原因之一，还有一个原因是网上关于 ASM 的资料较少，很少能拿来即跑的。

这次分享的重点 **不在于 ASM 怎样使用而在于 ASM 有什么用。**

#### ASM 的两类 API

ASM 提供了两类 Api，一类是 Core Api，一类是 Tree Api。

```JSON
// Core API
implementation "org.ow2.asm:asm:9.4"
// Tree API
implementation "org.ow2.asm:asm-tree:9.4"
```

Core Api 是以事件回调的形式访问字节码，这种方式占用内存小、访问速度快；而 Tree Api 是把整个字节码全部读到内存，占用内存大，但 Api 使用简单、能够很好的契合函数式编程。

#### ASM 示例

##### 读取 ArrayList 类

先来一个开胃菜，使用 Tree Api 读取 ArrayList 类，输出其前两个属性名和方法名：

```Kotlin
private fun readArrayListByTreeApi() {
    // 1. 从类的全限定名、或字节数组、或二进制字节流中读取字节码
    val classReader = ClassReader(ArrayList::class.java.canonicalName)
    // 2. 以 ClassNode 形式表示字节码
    val classNode = ClassNode(Opcodes.ASM9)
    classReader.accept(classNode, ClassReader.SKIP_CODE)
    classNode.apply {
        println("name: $name\n")
        // 3. 读取属性
        fields.take(2).forEach {
            println("field: ${it.name} ${Modifier.toString(it.access)} ${it.desc} ${it.value}")
        }
        println()
        // 4. 读取方法
        methods.take(2).forEach {
            println("method: ${it.name} ${Modifier.toString(it.access)} ${it.desc}")
        }
    }
}

// 输出
name: java/util/ArrayList

field: serialVersionUID private static final J 8683452581122892189
field: DEFAULT_CAPACITY private static final I 10

method: <init> public (I)V
method: <init> public ()V
```

##### 输出特定方法耗时

接下来就是网上举的最多的一个例子，输出方法的耗时。我们以 ASM 来实现类似 [hugo](https://github.com/JakeWharton/hugo) 的功能，输出以注解 @MeasureTime 标记的方法的耗时，修改前的 Java 文件如下：

```Java
public class MeasureMethodTime {

    @MeasureTime
    public void measure() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

期望修改后的 class 文件如下：

```Kotlin
public class MeasureMethodTimeTreeClass {
    public MeasureMethodTimeTreeClass() {
    }

    @MeasureTime
    public void measure() {
        long var3 = System.currentTimeMillis();

        long var5;
        try {
            Thread.sleep(2000L);
        } catch (InterruptedException var7) {
            RuntimeException var10000 = new RuntimeException(var7);
            var5 = System.currentTimeMillis();
            System.out.println(var5 - var3);
            throw var10000;
        }

        var5 = System.currentTimeMillis();
        System.out.println(var5 - var3);
    }
}
```

使用 Tree Api 操作核心代码如下：

```Kotlin
classNode.methods.forEach { methodNode ->
    // 该方法的注解列表中包含 @MeasureTime
    if (methodNode.invisibleAnnotations?.map { it.desc }
            ?.contains(Type.getDescriptor(MeasureTime::class.java)) == true) {
        val localVariablesSize = methodNode.localVariables.size
        // 在方法的第一个指令之前插入 System.currentTimeMillis()
        val firstInsnNode = methodNode.instructions.first
        methodNode.instructions.insertBefore(firstInsnNode, InsnList().apply {
            add(MethodInsnNode(Opcodes.INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J"))
            add(VarInsnNode(Opcodes.LSTORE, localVariablesSize + 1))
        })

        // 在方法 return 指令之前插入
        methodNode.instructions.filter {
            it.opcode.isMethodReturn()
        }.forEach {
            methodNode.instructions.insertBefore(it, InsnList().apply {
                add(MethodInsnNode(Opcodes.INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J"))
                // 注意，Long 是占两个局部变量槽位的，所以这里要较之前 +3，而不是 +2
                add(VarInsnNode(Opcodes.LSTORE, localVariablesSize + 3))
                add(FieldInsnNode(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"))
                add(VarInsnNode(Opcodes.LLOAD, localVariablesSize + 3))
                add(VarInsnNode(Opcodes.LLOAD, localVariablesSize + 1))
                add(InsnNode(Opcodes.LSUB))
                add(MethodInsnNode(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(J)V"))
            })
        }
    }
}
```

使用 Tree Api 非常容易实现，因为整个类都已经读取完毕，每个方法的局部变量表的大小已经确定了，新增局部变量只需要在后面追加即可。使用 Core Api 稍许麻烦一些，不过 ASM 也提供了 *LocalVariablesSorter* 类，为了方便生成局部变量获取其索引。

可能有的同学会问，Kotlin 已经提供了 measureTimeMillis 这种顶层函数来避免手动计算耗时，那这个示例的实践意义在哪呢？

在做启动优化时，一个关键的步骤是获取启动阶段的 Trace 文件，但 Systrace 默认的监控点大多都是系统调用，这就需要我们在每个方法入口和出口处，自动加上 Trace#beginSection 和 Trace#endSection，以生成以下 Trace 文件：

![image-20230212130513763](https://i.328888.xyz/2023/02/12/cD4iC.png)

##### 删除方法里面的日志语句

修改前的 Java 源文件：

```Kotlin
public class DeleteLogInvoke {

    public String print(String name, int age) {
        System.out.println(name);
        String result = name + ": " + age;
        System.out.println(result);
        System.out.println("Delete current line.");
        System.out.println("name = " + name + ", age = " + age);
        System.out.printf("name: %s%n", name);
        System.out.println(String.format("age: %d", age));
        return result;
    }
}
```

期望修改后的 class 文件：

```Kotlin
public class DeleteLogInvokeCoreClass {
    public DeleteLogInvokeCoreClass() {
    }

    public String print(String var1, int var2) {
        String var3 = var1 + ": " + var2;
        return var3;
    }
}
```

解决思路是：删除 GETSTATIC out 和 INVOKEVIRTUAL println 之间的所有指令。

可能又有同学会问为啥不用 Proguard 呢？Proguard 提供了 assumenosideeffects 来 [移除日志代码](https://www.guardsquare.com/manual/configuration/examples#logging)：

```Kotlin
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int i(...);
    public static int w(...);
    public static int d(...);
    public static int e(...);
}
```

原因是，**Proguard 无法删除一些隐式的中间调用。**

对于以下这段 Kotlin 代码：

```Kotlin
Log.i("MainActivity", "onCreate: $packageName")
```

生成的 Dex 字节码如下：

```kotlin
    invoke-virtual {p0}, Landroid/content/Context;->getPackageName()Ljava/lang/String;

    move-result-object p1

    const-string v0, "onCreate: "

    invoke-static {v0, p1}, kotlin.jvm.internal.Intrinsics.stringPlus(Ljava/lang/String;Ljava/lang/Object;)Ljava/lang/String;

    move-result-object p1

    const-string v0, "MainActivity"

    invoke-static {v0, p1}, Landroid/util/Log;->i(Ljava/lang/String;Ljava/lang/String;)I
```

删之后，发现中间的 getPackageName 调用和字符串拼接调用仍然存在：

```kotlin
    invoke-virtual {p0}, Landroid/content/Context;->getPackageName()Ljava/lang/String;

    move-result-object p1

    const-string v0, "onCreate: "

    invoke-static {v0, p1}, Landroidx/constraintlayout/widget/R$id;->kotlin.jvm.internal.Intrinsics.stringPlus(Ljava/lang/String;Ljava/lang/Object;)Ljava/lang/String;
```

其实 Proguard 提供了 assumenoexternalsideeffects、assumenoexternalreturnvalues 来删除这些中间调用，但在 R8 （在 AGP 3.4及其以上，R8 是 [默认编译器](https://developer.android.com/studio/build/shrink-code#enable) 了）上已经不支持该配置了：

```ceylon
> Task :app:minifyReleaseWithR8
WARNING: R8: Ignoring option: -assumenoexternalsideeffects
WARNING: R8: Ignoring option: -assumenoexternalreturnvalues
```

而使用 ASM 则可以彻底删除干净。

##### 线程重命名

该示例是来源于 [Booster - 线程重命名](https://booster.johnsonlee.io/zh/guide/performance/multithreading-optimization.html#线程重命名)。

线程管理一直是非常头痛的事情，应用在启动时就可能初始化了几十上百个线程在跑。开发者面临的问题有：

1. 可能存在低优先级的子线程抢占 CPU，导致主线程 UI 响应能力降低、主线程空等子线程的锁等，过多的资源竞争意味着过多的资源都浪费在了线程调度上
2. 不可控的线程创建可能导致 OOM
3. 线程命名默认是以 Thread-{N} 形式命名，不知道线程是在哪个模块哪个类创建的，不利于问题排查

通过对线程重命名 --- 为线程名增加当前类名的前缀，当 APM 工具上报异常信息或对线程进行采样时，采集到的线程信息对于排查问题十分有帮助。

修改前的 Java 源代码：

```Kotlin
public class ThreadReName {

    public static void main(String[] args) {
        // 不带线程名称
        new Thread(new InternalRunnable()).start();

        // 带线程名称
        Thread thread0 = new Thread(new InternalRunnable(), "thread0");
        System.out.println("thread0: " + thread0.getName());
        thread0.start();

        Thread thread1 = new Thread(new InternalRunnable());
        // 设置线程名字
        thread1.setName("thread1");
        System.out.println("thread1: " + thread1.getName());
        thread1.start();
    }
}
```

修改后的 class 文件如下：

```Kotlin
public class ThreadReNameTreeClass {
    public ThreadReNameTreeClass() {
    }

    public static void main(String[] var0) {
        (new ShadowThread(new InternalRunnable(), "sample/ThreadReNameTreeClass#main-Thread-0")).start();
        ShadowThread var1 = new ShadowThread(new InternalRunnable(), "thread0", "sample/ThreadReNameTreeClass#main-Thread-1");
        System.out.println("thread0: " + var1.getName());
        var1.start();
        ShadowThread var2 = new ShadowThread(new InternalRunnable(), "sample/ThreadReNameTreeClass#main-Thread-2");
        var2.setName(ShadowThread.makeThreadName("thread1", "sample/ThreadReNameTreeClass#main-Thread-3"));
        System.out.println("thread1: " + var2.getName());
        var2.start();
    }
}
```

输出：

```Kotlin
thread0: sample/ThreadReNameTreeClass#main-Thread-1#thread0

thread1: sample/ThreadReNameTreeClass#main-Thread-3#thread1
```

这种替换系统调用的做法，应用也比较多。比如替换系统默认的 SharedPreferences 实现以避免可能的卡顿、ANR；替换方法调用为系统类或第三库的代码兜底等。

留一个思考题，Toast 在 Android 7.1 上存在抛 BadTokenException 的崩溃情况，我们简化一下这个系统 bug，如果有以下代码：

```Kotlin
public class ReplaceMethodInvoke {
    public static void main(String[] args) {
        // throw NPE
        new Toast().show();
    }
}

public class Toast {
    private String msg = null;

    public void show() {
        System.out.println("Toast: " + msg + ", msg.length: " + msg.length());
    }
}
```

该如何避免其在运行时崩溃呢？

##### 序列化检查

该示例是来源于[ ByteX - 序列化检查](https://github.com/bytedance/ByteX/blob/master/serialization-check-plugin/README.md)。

对于 Java 的序列化，有以下几条规则需要遵守（也就是 IDEA Inspections 里的几条规则）：

1. 实现了 Serializable 的类未提供 serialVersionUID 字段
2. 实现了 Serializable 的类包含非 transient、static 的字段，这些字段并未实现 Serializable 接口
3. 未实现 Serializable 接口的类，包含 transient、serialVersionUID 字段
4. 实现了 Serializable 的非静态内部类，它的外层类并未实现 Serializable 接口

对于以下 Java 代码：

```Kotlin
public class SerializationCheck implements Serializable {

    private ItemBean1 itemBean1;
    private ItemBean2 itemBean2;
    private transient ItemBean3 itemBean3;
    private String name;
    private int age;

    static class ItemBean1 {
    }

    static class ItemBean2 implements Serializable {
    }

    static class ItemBean3 {
    }
}
```

需要检查出来并输出：

```Kotlin
Attention: Non-serializable field 'itemBean1' in a Serializable class [sample/SerializationCheckCoreClass]
Attention: This [sample/SerializationCheckCoreClass] class is serializable, but does not define a 'serialVersionUID' field.
```

这种类静态分析的能力，应用场景也比较多。比如检查是否存在调用不存在的字段或方法、隐私合规 Api 调用监测等等。

#### ASM 不能做什么

通过上面的讲述，相信你已经了解了 ASM 的强大之处了，似乎 ASM 无所不能，但相比于 ASM 能做什么，能够认识到它不能做什么，有时候会显得更加重要。所以，这一小节用来讨论下 ASM 不能做什么。

如果仍然用一句话总结，那就是 不支持动态分析或者说是运行时分析。

我们来看一个例子，比如我们想检测 Android 项目中有哪些 assets 资源未被使用。

一个显而易见的思路是：

1. 收集项目中所有的 AAR 包含的 assets 资源名
2. 在 Transform 阶段，check 以下调用，收集所有已使用的 assets 资源名

```Kotlin
context.getAssets().open("fileName.json")
```

两个结果集一对比，就可以知道哪些 assets 资源未被使用了。

如果真是这样的调用，其实还可以通过静态分析拿到，因为 ldc 指令可以拿到对应的 value 即 "fileName.json"，但是如果这个文件名是 方法的入参或出参，那就没有办法了，因为只有在运行时才能知道具体的值是什么。

[Matrix#UnusedAssetsTask](https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-apk-canary/src/main/java/com/tencent/matrix/apk/model/task/UnusedAssetsTask.java) 给出的方案是：搜索 smali 文件中引用字符串常量的指令，判断引用的字符串常量是否某个 assets 文件的名称。

#### 总结

最后总结一下，ASM 的应用场景有：

1. Static Analysis

   静态分析；序列化检查、检查是否存在调用不存在的字段或方法、隐私合规 Api 调用监测等等，都属于该应用范畴。

2. AOP

   面向切面编程；输出特定方法耗时、线上代码覆盖率监测等都属于该应用范畴。

3. Hook

   对原有代码逻辑做增强，一般实现手段有反射和静态/动态代理；线程重命名、点击防手抖、修复第三方库或系统 bug 等都属于该应用范畴。

不在 ASM 的应用范畴里：动态分析。

#### 更多

1. [https://github.com/Omooo/ASM-Task](https://github.com/Omooo/ASM-Task)
2. [Chapter 4. The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4.1)
2. [内部分享的 PPT](https://github.com/Omooo/Android-Notes/tree/master/PPT/ASM%20%E5%BA%94%E7%94%A8%E4%B8%8E%E5%AE%9E%E8%B7%B5)

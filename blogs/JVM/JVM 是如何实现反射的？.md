---
JVM 是如何实现反射的？
---

#### 目录

1. 前言
2. 反射调用的实现
3. 反射调用的开销
4. 总结

#### 前言

反射是 Java 语言中一个相当重要的特性，它允许正在运行的 Java 程序观测，甚至是修改程序的动态行为。

举例来说，我们可以通过 Class 对象枚举该类中的所有方法，我们还可以通过 Method.setAccessible（位于 java.lang.reflect 包，该方法继承自 AccessibleObject）绕过 Java 语言的访问权限，在私有方法所在类之外的地方调用该方法。

今天我们便来了解一下反射的实现机制，以及它性能糟糕的原因。

#### 反射调用的实现

首先，我们来看看方法的反射调用，也就是 Method.invoke，是怎么实现的。

```java
public final class Method extends Executable {
  ...
  public Object invoke(Object obj, Object... args) throws ... {
    ... // 权限检查
    MethodAccessor ma = methodAccessor;
    if (ma == null) {
      ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
  }
}
```

如果你查阅 Method.invoke 的源代码，那么你会发现，它实际上委派给 MethodAccessor 来处理。MethodAccessor 是一个接口，它有两个已有的具体实现：一个通过本地方法来实现反射调用（NativeMethodAccessorImpl），另一个则使用了委派模式（DelegatingMethodAccessorImpl）。为了方便记忆，我便用 “本地实现” 和 “委派实现” 来指代这两者。

每个 Method 实例的第一次反射调用都会生成一个委派实现，它所委派的具体实现便是一个本地实现。本地实现非常容易理解，当进入了 Java 虚拟机内部之后，我们便拥有了 Method 实例所指向方法的具体地址。这时候，反射调用无非就是将传入的参数准备好，然后调用进入目标方法。

```java
public class Main {

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("Main");
        Method method = clazz.getMethod("target", int.class);
        method.invoke(null, 233);
    }

    public static void target(int i) {
        new Exception("#" + i).printStackTrace();
    }
}

java.lang.Exception: #233
	at Main.target(Main.java:16)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at Main.main(Main.java:12)
```

为了方便理解，我们可以打印一下反射调用到目标方法时的栈轨迹。

可以看到，反射调用先是调用了 Method.invoke，然后进去委派实现（DelegatingMethodAccessorImpl），再然后进入本地实现（NativeMethodAccessorImpl），最后到达目标方法。

这里你可能会疑问，为什么反射调用还要采取委派实现作为中间层？直接交给本地实现不可以么？

其实，Java 的反射调用机制还设立了另外一种动态生成字节码的实现（下称动态实现），直接使用 invoke 指令来调用目标方法。之所以采取委派实现，便是为了能够在本地实现以及动态实现中切换。

```java
// 动态实现的伪代码，这里只列举了关键的调用逻辑，其实它还包括调用者检测、参数检测的字节码。
package jdk.internal.reflect;

public class GeneratedMethodAccessor1 extends ... {
  @Overrides    
  public Object invoke(Object obj, Object[] args) throws ... {
    Test.target((int) args[0]);
    return null;
  }
}
```

动态实现和本地实现相比，其运行效率要快上 20 倍。这是因为动态实现无需经过 Java 到 C++ 再到 Java 的切换，但由于生成字节码十分耗时，仅调用一次的话，反而是本地实现要快上 3 到 4 倍。

考虑到很多反射调用仅会执行一次，Java 虚拟机设置了一个阈值 15，当某个反射调用的调用次数在 15 之下时，采用本地实现；当达到 15 时，便开始动态生成字节码，并将委派实现的委派对象切换至动态实现，这个过程我们称之为 Inflation。

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("Main");
        Method method = clazz.getMethod("target", int.class);
        for (int i = 0; i < 20; i++) {
            method.invoke(null, i);
        }
    }

    public static void target(int i) {
        new Exception("#" + i).printStackTrace();
    }
}

# 使用 -verbose:class 打印加载的类
$ java -verbose:class Main    


[0.216s][info][class,load] java.util.IdentityHashMap$KeySet source: jrt:/java.base
java.lang.Exception: #0
[0.216s][info][class,load] java.lang.StackTraceElement$HashedModules source: jrt:/java.base
        at Main.target(Main.java:17)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        at Main.main(Main.java:12)

	
...	//省略

[0.234s][info][class,load] sun.reflect.misc.ReflectUtil source: jrt:/java.base
[0.234s][info][class,load] jdk.internal.reflect.ClassFileConstants source: jrt:/java.base
[0.234s][info][class,load] jdk.internal.reflect.AccessorGenerator source: jrt:/java.base
[0.235s][info][class,load] jdk.internal.reflect.MethodAccessorGenerator source: jrt:/java.base
[0.235s][info][class,load] jdk.internal.reflect.ByteVectorFactory source: jrt:/java.base
[0.235s][info][class,load] jdk.internal.reflect.ByteVector source: jrt:/java.base
[0.236s][info][class,load] jdk.internal.reflect.ByteVectorImpl source: jrt:/java.base
[0.236s][info][class,load] jdk.internal.reflect.ClassFileAssembler source: jrt:/java.base
[0.236s][info][class,load] jdk.internal.reflect.UTF8 source: jrt:/java.base
[0.236s][info][class,load] jdk.internal.reflect.Label source: jrt:/java.base
[0.237s][info][class,load] jdk.internal.reflect.Label$PatchInfo source: jrt:/java.base
[0.238s][info][class,load] jdk.internal.reflect.MethodAccessorGenerator$1 source: jrt:/java.base
[0.238s][info][class,load] jdk.internal.reflect.ClassDefiner source: jrt:/java.base
[0.238s][info][class,load] jdk.internal.reflect.ClassDefiner$1 source: jrt:/java.base
[0.239s][info][class,load] jdk.internal.reflect.GeneratedMethodAccessor1 source: __JVM_DefineClass__
java.lang.Exception: #15
        at Main.target(Main.java:17)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        at Main.main(Main.java:12)
java.lang.Exception: #16
        at Main.target(Main.java:17)
        at jdk.internal.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        at Main.main(Main.java:12)
```

可以看到，在第 15 次（从 0 开始数）反射调用时，我们便触发了动态实现的生成。这时候，Java 虚拟机额外加载了不少类。其中，最重要的当属 GeneratedMethodAccessor1。并且，从第 16 次反射调用开始，我们便切换至这个刚刚生成的动态实现。

反射调用的 Inflation 机制是可以通过参数来关闭的。这样一来，在反射调用一开始便会直接生成动态实现，而不会使用委派实现或者本地实现。

#### 反射调用的开销

下面，我们便来拆解反射调用的性能开销。

在刚才的例子中，我们先后进行了 Class.forName，Class.getMethod 以及 Method.invoke 三个操作。其中，Class.forName 会调用本地方法，Class.getMethod 则会遍历该类的公有方法。如果没有匹配到，它还将遍历父类的公有方法。可想而知，这两个操作都非常耗时。

值得注意的是，以 getMethod 为代表的查找方法操作，会返回查找得到结果的一份拷贝。因此，我们应当避免在热点代码中使用返回 Method 数组的 getMethods 或者 getDeclaredMethods 方法，以减少不必要的堆空间消耗。

在实践中，我们往往会在应用程序中缓存 Class.forName 和 Class.getMethod 的结果。因此，下面我们就只关注反射调用本身的性能开销。

#### 总结

在默认情况下，方法的反射调用为委派实现，委派给本地实现来进行方法调用。在调用超过 15 次之后，委派实现便会将委派对象切换至动态实现。这个动态实现的字节码是自动生成的，它将直接使用 invoke 指令来调用目标方法。

方法的反射调用会带来不少性能开销，原因主要有三个：变成参数方法导致的 Object 数组，基于类型的自动装箱、拆箱，还有最重要的方法内联。


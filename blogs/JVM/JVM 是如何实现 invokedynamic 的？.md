---
JVM 是如何实现 invokedynamic 的？
---

#### 目录

1. 前言
2. 方法句柄的概念
3. 方法句柄的操作
4. 方法句柄的实现
5. 简单小结

#### 前言

在 Java 中，方法调用会被编译为 invokestatic、invokespecial、invokevirtual 以及 invokeinterface 四种指令。这些指令与包含目标方法类名、方法名以及方法描述符的符号引用捆绑。在实际运行之前，Java 虚拟机将根据这个符号引用链接到具体的目标方法。

可以看到，这四种指令中，Java 虚拟机明确要求方法调用需要提供目标方法的类名。在这种体系下，我们有两种解决方案，一是提供其类似代理的办法，另外一种则是使用反射。

显然，比起直接调用，这两种方法都相当复杂，执行效率也可想而知。为了解决这个问题，Java 7 引入了一条新的指令 invokedynamic。该指令的调用机制抽象出调用点这一概念，并允许应用程序将调用点链接至任意符合条件的方法上。

作为 invokedynamic 的准备工作，Java 7 引入了更加底层、更加灵活的方法抽象：方法句柄（MethodHandle）。

#### 方法句柄的概念

方法句柄是一个强类型的，能够被直接执行的引用。该引用可以指向常规的静态方法或者实例方法，也可以指向构造器或者字段。当指向字段时，方法句柄实则指向包含字段访问字节码的虚构方法，语义上等价于目标字段的 getter 或者 setter 方法。

这里需要注意的是，它并不会直接指向目标字段所在类的 getter/setter，毕竟你无法保证已有的 getter/setter 方法就是在访问目标字段。

方法句柄的类型（MethodType）是由所指向方法的参数类型以及返回类型组成的。它是用来确认方法句柄是否适配的唯一关键。当使用方法句柄时，我们其实并不关心方法句柄所指向方法的类名或者方法名。

方法句柄的创建是通过 MethodHandles.Lookup 类来完成的。它提供了多个 API，既可以使用反射 API 中的 Method 来查找，也可以根据类、方法名以及方法句柄类型来查找。

当使用后者这种查找方法时，用户需要区分具体的调用类型，比如说对于 invokestatic 调用的静态方法，我们需要使用 Lookup.findStatic 方法；对于用 invokevirtual 调用的实例方法以及用 invokeinterface 调用的接口方法，我们需要使用 findVirtual 方法；对于用 invokespecial 调用的实例方法，我们则需要使用 findSpecial 方法。

调用方法句柄，和原本对应的调用指令是一致的。也就是说，对于原本用 invokevirtual 调用的方法句柄，它也会采用动态绑定；而对于原本用 invokespecial 调用的方法句柄，它会采用静态绑定。

```java
public class Main {

    private void show(String s) {
        System.out.println(s);
    }

    public static void main(String[] args) throws Throwable {
        // 第一种方式
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        Method method = Main.class.getDeclaredMethod("show", String.class);
        MethodHandle mH1 = lookup.unreflect(method);
        mH1.invoke(new Main(), "23333");

        // 第二种方式
        MethodType methodType = MethodType.methodType(void.class, String.class);
        MethodHandle mH2 = lookup.findSpecial(Main.class, "show", methodType, Main.class);
        mH2.invoke(new Main(), "23333");
    }
}
```

方法句柄同样也有权限问题，但它与反射 API 不同，其权限检查是在句柄的创建阶段完成的。在实际调用过程中，Java 虚拟机并不会检查方法句柄的权限。如果该句柄被多次调用的话，那么与反射调用相比，它将省下重复权限检查的开销。

需要注意的是，方法句柄的访问权限不取决于方法句柄的创建位置，而是取决于 Lookup 对象的创建位置。

举个例子，对于一个私有字段，如果 Lookup 对象是在私有字段所在类中获取的，那么这个 Lookup 对象便拥有对该私有字段的访问权限，即时是在所在类的外边，也能够通过该 Lookup 对象创建该私有字段的 getter 或者 setter。

由于方法句柄没有运行时权限检查，因此，应用程序需要负责方法句柄的管理。一旦它发布了某些指向私有方法的方法句柄，那么这些私有方法便被暴露出去了。

#### 方法句柄的操作

方法句柄的调用可分为两种，一种需要严格匹配参数类型的 invokeExact。它有多严格呢？假设一个方法句柄将接收一个 Object 类型的参数，如果你直接传入 String 作为实际参数，那么方法句柄的调用会在运行时抛出方法类型不匹配的异常。正确的调用方式是将该 String 显式转化为 Object 类型。

在普通 Java 方法调用中，我们只有在选择重载方法时，才会用到这种显式转化。这是因为经过显式转化后，参数的声明类型发生了改变，因此有可能匹配到不同的方法描述符，从而选取不同的目标方法。调用方法句柄也是利用同样的原理，并且涉及了一个签名多态性的概念、

```java
  public final native @PolymorphicSignature Object invokeExact(Object... args) throws Throwable;
```

方法句柄 API 有一个特殊的注解类 @PolymorphicSignature。在碰到被它注解的方法调用时，Java 编译器会根据所传入参数的声明类型来生成方法描述符，而不是采用目标方法所声明的描述符。

在刚才的例子中，当传入的参数是 String 时，对应的方法描述符包含 String 类；而当我们转化为 Object 时，对应的方法描述符则包含 Object 类。

invokeExact 会确认该 invokevirtual 指令对应的方法描述符，和该方法句柄的类型是否严格匹配。在不匹配的情况下，便会在运行时抛出异常。

如果你需要自动适配参数类型，那么你可以选取方法句柄的第二种调用方式 invoke。它同样是一个签名多态性的方法。invoke 会调用 MethodHandle.asType 方法，生成一个适配器方法句柄，对传入的参数进行适配，在调用原方法句柄。调用原方法句柄的返回值同样也会先进行适配，然后再返回给调用者。

方法句柄还支持增删改参数的操作，这些操作都是通过生成另外一个方法句柄来实现的。这其中，改操作就是刚刚介绍的 MethodHandle.asType 方法。删操作指的是将传入的部分参数就地抛弃，再调用另一个方法句柄，它对应的 API 是 MethodHandles.dropArguments 方法。

增操作则非常有意思，它会往传入的参数中插入额外的参数，再调用另一个方法句柄，它对应的 API 是 MethodHandle.bindTo 方法。Java 8 中捕获类型的 Lambda 表达式便是用这种操作来实现的。

#### 方法句柄的实现

下面我们来看看 HotSpot 虚拟机中方法句柄调用的具体实现。（由于篇幅原因，这里只讨论 DirectMethodHandle）。

前面提到，调用方法句柄所使用的 invokeExact 或者 invoke 方法具备签名多态性的特性。它们会根据具体的传入参数来生成方法描述符。那么，拥有这个描述符的方法实际存在吗？对 invokeExact 或者 invoke 的调用具体会进入哪个方法呢？

```java
public class Main {

    private void show(String s) {
        new Exception().printStackTrace();
    }

    public static void main(String[] args) throws Throwable {
        MethodType methodType = MethodType.methodType(void.class, String.class);
        MethodHandle mH2 = MethodHandles.lookup().findSpecial(Main.class, "show", methodType, Main.class);
        mH2.invokeExact(new Main(), "23333");
    }
}
```

和查阅反射调用的方式一样，我们可以通过新建异常实例来查看栈轨迹。打印出来的栈轨迹如下：

```
java.lang.Exception
	at Main.show(Main.java:9)
	at Main.main(Main.java:15)
```

也就是说，invokeExact 的目标方法就是方法句柄指向的方法。

先别高兴的太早。我们刚刚提过，invokeExact 会对参数的类型进行效验，并在不匹配的情况下抛出异常。如果它直接调用了方法句柄所指向的方法，那么这部分参数类型效验的逻辑将无处安放。因此，唯一的可能就是 Java 虚拟机隐藏了部分栈信息。

当我们启用了 -XX:+showHiddenFrames 这个参数来打印被 Java 虚拟机隐藏了的栈信息时，你会发现 main 方法和目标方法中间隔着两个貌似是生成的方法。

```
java -XX:+UnlockDiagnosticVMOptions -XX:+ShowHiddenFrames Main

java.lang.Exception
        at Main.show(Main.java:9)
        at java.base/java.lang.invoke.DirectMethodHandle$Holder.invokeSpecial(DirectMethodHandle$Holder:1000011)
        at java.base/java.lang.invoke.LambdaForm$MH/0x0000000800060c40.invokeExact_MT(LambdaForm$MH:1000020)
        at Main.main(Main.java:15)

```

实际上，Java 虚拟机会对 invokeExact 调用做特殊处理，调用至一个共享的、与方法句柄类型相关的特殊适配器中。这个适配器是一个 LambdaForm，在这个适配器中，它会调用 Invokes.checkExactType 方法来检查参数类型，然后调用 Invokes.checkCustomized 方法。后者会在方法句柄的执行次数超过一个阈值时进行优化。最后，它会调用方法句柄的 invokeBasic 方法。

//...

方法句柄的调用和反射调用一样，都是间接调用。因此，它也会面临无法内联的问题。不过，与反射调用不同的是，方法句柄的内联瓶颈在于即时编译器能否将该方法句柄识别为常量。

#### 简单小结

invokedynamic 底层机制的基石：方法句柄。

方法句柄是一个强类型、能够被直接执行的引用。它仅关心所指向方法的参数类型以及返回类型，而不关心方法所在类以及方法名。方法句柄的权限检查发生在创建过程中，相较于反射调用节省了调用时反复权限检查的开销。

方法句柄可以通过 invokeExact 以及 invoke 来调用。其中，invokeExact 要求传入的参数和所指向方法的描述符严格匹配。方法句柄还支持增删改参数的操作，这些操作是通过生成另一个充当适配器的方法句柄来实现的。

方法句柄的调用和反射调用一样，都是间接调用，同样会面临无法内联的问题。


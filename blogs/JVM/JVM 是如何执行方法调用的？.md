---
JVM 是如何执行方法调用的？
---

#### 目录

1. 思维导图

2. 问题描述

3. Java 编译器识别方法

   - 三个阶段

4. JVM 识别方法

   - 五个指令
     - invokestatic
     - invokespecial
     - invokevirtual
     - invokeinterface
     - invokedynamic

5. 虚方法调用

   - 涉及指令
     - invokevirtual
     - invokeinterface

   - 方法表

6. 内联缓存

7. 总结

#### 思维导图

![](https://i.loli.net/2019/02/11/5c618baac073b.png)

#### 问题描述

问一下代码输出？

```java
public class MethodDemo {
    public static void main(String[] args) {
        invoke("s");
        invoke(2);
        invoke(null, 1);
        invoke(null, 1, 1);
        invoke(null, new Object[1]);
    }

    private static void invoke(Object object) {
        System.out.println("执行方法一");
    }

    private static void invoke(String s) {
        System.out.println("执行方法二");
    }

    private static void invoke(Integer integer) {
        System.out.println("执行方法三");
    }

    private static void invoke(int i) {
        System.out.println("执行方法四");
    }

    private static void invoke(Object o1, Object... o2) {
        System.out.println("执行方法五");
    }

    private static void invoke(String s, Object o1, Object... o2) {
        System.out.println("执行方法六");
    }
}
```

其实就需要了解 Java 编译器和 JVM 是如何区分方法的。区分方法可以分为区分重载方法和区分重写方法，方法重载在编译阶段就能确定下来，而方法重写则需要运行时才能确定。这也就对应着以下的 Java 编译器识别方法和 JVM 识别方法。

#### Java 编译器识别方法

重载方法在编译过程中即可完成识别，具体到每一个方法调用，Java 编译器会根据所传入的参数的声明类型来选取重载方法，选取的过程分为三个阶段：

1. 在不考虑对基本类型自动装拆箱、以及可变长参数的情况下选取重载方法
2. 如果在第一个阶段没有找到适配的方法，那么在允许自动装拆箱，但不允许可变长参数的情况下选择重载方法
3. 如果在上两个阶段都没有找到适配的方法，那么在允许自动装拆箱以及可变长参数的情况下选取重载方法

如果 Java 编译器在同一个阶段找到了多个适配的方法，那么它会在其中选择一个最为贴切的，而决定贴切程度的一个关键就是形式参数类型的继承关系。

在问题描述中，invoke("s") 会执行方法二，而不会执行方法一。由于 String 是 Object 的子类，因此 Java 编译器会认为方法二更加贴切。

#### JVM 识别方法

JVM 识别方法的关键在于类名、方法名以及方法描述符。前面两个就不做过多解释了，至于方法描述符，它是由方法的参数类型**以及返回类型**所构成。这也是方法重写所要求的。

由于对重载方法的区分在编译阶段已经完成，所以可以认为在 JVM 不存在重载这一概念，因此，在某些文章中，重载也被称为静态绑定，或者编译时多态，而重写则被称为动态绑定。

这个说法在 JVM 语境下并非完全正确，这是因为某个类中的重载方法可能被它的子类所重写，因此 Java 编译器会将所有对非私有实例方法的调用编译为需要动态绑定的类型。

确切的说，Java 虚拟机中的静态绑定指的是在解析时便能够直接识别目标方法的情况，而动态绑定则指的是需要在运行过程中根据调用者的动态类型来识别目标方法的情况。

具体来说，Java 字节码中与调用相关的指令共有五种：

1. invokestatic : 用于调用静态方法
2. invokespecial : 用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所有实现接口的默认方法
3. invokevirtual : 用于调用非私有实例方法
4. invokeinterface : 用于调用接口方法
5. invokedynamic : 用于调用动态方法

对于 invokestatic 以及 invokespecial 而言，Java 虚拟机能够直接识别具体的目标方法。而对于 invokevirtual 和 invokeinterface 而言，在绝大部分情况下，虚拟机需要在执行过程中，根据调用者的动态类型，来确定具体的目标方法。

唯一例外在于，如果虚拟机能够确定目标方法有且只有一个，比如方法被 final 修饰，那么它就可以不通过动态类型，直接确定目标方法。

#### 虚方法调用

在上面讲到，Java 里所有非私有实例方法调用都被编译成 invokevirtual 指令，而接口方法调用都会被编译成 invokeinterface 指令，这两种指令，均属于 Java 虚拟机中的虚方法调用。

在大多数情况下，Java 虚拟机需要根据调用者的动态类型，来确定虚方法调用的目标方法，这个过程我们称为动态绑定。相对于静态绑定的非虚方法调用来说，虚方法调用更加耗时。

Java 虚拟机采用了一种用空间换时间的策略来实现动态绑定，它为每个类生成一张方法表，用以快速定位目标方法。

##### 方法表

在类加载的准备阶段，除了为静态字段分配内存之外，还会构造与该类相关联的方法表。方法表是 JVM 实现动态绑定的关键所在。

方法表本质上是一个数组，每个数组元素指向一个当前类及其祖先类中非私有的实例方法。

方法表满足两个特性：

1. 子类方法表中包含父类方法表中的所有方法
2. 子类方法在方法表中的索引值，与它所重写的父类方法的索引值相同

我们知道，方法调用指令中的符号引用会在执行之前解析成实际引用。对于静态绑定的方法调用而言，实际引用将指向具体的目标方法。对于动态绑定的方法调用而言，实际引用则是方法表的索引值。

```java
public abstract class SuperClass {
    abstract void doSomething();

    @Override
    public String toString() {
        return "SuperClass";
    }

    class ChildClass extends SuperClass{
        @Override
        void doSomething() {

        }
        
        public void play(){
            
        }

        @Override
        public String toString() {
            return "ChildClass";
        }
    }
}
```

方法表：

|      | SuperClass 方法表                 | ChildClass 方法表 |
| ---- | --------------------------------- | ----------------- |
| 0    | toString                          | toString          |
| 1    | doSomething（抽象方法，不可执行） | doSomething       |
| 2    |                                   | play              |

实际上，使用了方法表的动态绑定与静态绑定相比，仅仅多出几个内存解析引用操作：访问栈上的调用者，读取调用者的动态类型，读取该类型的方法表，读取方法表中某个索引值所对应的目标方法。相对于创建并初始化 Java 栈帧来说，这几个内存解引用操作的开销简直可以忽略不计。

那么是不是就可以认为虚方法调用对性能没有太大影响呢？

其实是不能的，上诉优化的效果看起来好不错，但实际上仅存于解释执行中，或者即时编译代码的最坏情况中。这是因为即时编译还拥有另外两种性能更好的优化手段：内联缓存和方法内联。

#### 内联缓存

内联缓存是一种加快动态绑定的优化技术。它能够缓存虚方法调用中调用者的动态类型，以及该类型所对应的目标方法。在之后的执行过程中，如果碰到已缓存的类型，内联缓存便会直接调用该类型所对应的目标方法。如果没有碰到已缓存的类型，内联缓存则会退化至使用基于方法表的动态绑定。

虽然内联缓存附带内联两字，但是它并没有内联目标方法。这里需要明确的是，任何方法调用除非被内联，否则都会有固定开销。这些开销来源于保存程序在该方法中的执行位置，以及新建、压入和弹出新方法所使用的栈帧。

#### 总结

1. 重载的方法识别在编译阶段就能确定，而重写的方法则需要在运行时确定
2. 虚方法调用包括非私有方法和接口方法调用，它们的动态绑定是通过方法表来实现的
3. 即时编译器会使用内联缓存来加速动态绑定

#### 参考

[JVM是如何执行方法调用的？（上）](https://time.geekbang.org/column/article/11539)

[JVM是如何执行方法调用的？（下）](https://time.geekbang.org/column/article/12098)
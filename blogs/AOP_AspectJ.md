---
AOP 之 AspectJ
---

#### 目录

1. 思维导图
2. AOP
   - 概念
   - 术语
   - 实现方式
3. AspectJ
   - 引入
   - 注解定义
   - 切点表达式
   - ProceedingJoinPoint、JoinPoint
   - 示例
     - 全局点击事件拦截
     - 方法统计耗时
     - 权限申请
4. 参考

#### 思维导图

![](https://i.loli.net/2018/12/17/5c17b43b9662f.png)

#### AOP

##### 概念

AOP 即面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。利用 AOP 可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高开发效率。

在手动埋点的时候，需要在每个需要埋点的地方都添加代码，导致页面臃肿。还有就是在需要统计方法执行的时间的时候，不可能在每个方法执行前后都添加一段代码，所以就可以用到 AOP 的思想，它可以对一组相同的事物做统一处理。

##### 术语

1. Advice 通知、增强处理

   即想要添加的功能，比如自动化埋点、统计方法耗时等等。

2. JoinPoint 连接点

   允许你通知（ Advice ）的地方，方法的前前后后都是连接点。

3. PointCut 切入点

   从连接点上，选择切点。

4. Aspect 切面

   切面是通知和切入点的结合。现在发现了吧，没连接点什么事，连接点只是为了让你更好的理解切点从哪来。

5. Weaving 织入

   把切面应用到目标对象来创建新的代理对象的过程。

##### 实现方式

AOP 和 OOP 都只是一种编程思想，实现 AOP 的方式有以下几种：

1. AspectJ

   一个 Java 语言的面向切面编程的无缝扩展。

2. Javassist for Android

   用于字节码操作的知名 Java 类库 Javassist 的 Android 平台移植版。

3. DexMaker

   Dalvik 虚拟机上，在编译器或者运行时生成代码的 Java API。

4. ASMDEX

   一个类似 ASM 的字节码操作库，运行在 Android 平台，操作 Dex 字节码。

#### AspectJ

##### 引入

引入直接看文末参考文章。

##### 注解定义

- @Aspect

  声明切面，标记类。

- @Pointcut(切点表达式)

  定义切点，标记方法。

- @Before(切点表达式)

  前置通知，切点之前执行。

- @Around(切点表达式)

  环绕通知，切点前后执行。

- @After(切点表达式)

  后置通知，切点之后执行。

- @AfterReturning(切点表达式)

  返回通知，切点方法返回结果之后执行。

- @AfterThrowing(切点表达式)

  异常通知，切点抛出异常时执行。

以上定义的切点都必须在标记了 @Aspect 的类中使用。

##### 切点表达式

切点表达式的结构如下：

```java
execution(<@注解类型模式>? <修饰符模式>? <返回类型模式> <方法名模式>(<参数模式>) <异常模式>?)
within(<@注解类型模式>? <修饰符模式>? <返回类型模式> <方法名模式>(<参数模式>) <异常模式>?)
```

其中注解类型模式、修饰符模式、异常模式都是可选的。

 其中 execution() 是方法切点函数，表示目标类中满足某个匹配模式的方法连接点。而 winth() 是目标类切点函数，表示满足某个匹配模式的特定域中的类的所有连接点，比如某个包下所有的类。

##### ProceedingJoinPoint、JoinPoint

ProceedingJoinPoint 用于环绕通知，它是 JointPoint 的子类，与其不同的是 ProceedingJoinPoint 提供了 proceed() 方法用于执行切点方法。

```java
    @Around("execution( * cn.* (..))")
    public void test(ProceedingJoinPoint joinPoint) throws Throwable {
        //在方法执行之前执行
        int i = 0;
        joinPoint.proceed();
        //在方法执行之后执行
        i++;
        
        //获取签名信息
        MethodSignature signature= (MethodSignature) joinPoint.getSignature();
        signature.getName();    //方法名
        signature.getDeclaringTypeName();    //类名
        signature.getReturnType();  //返回类型
        signature.getParameterNames();      //方法参数名
        signature.getParameterTypes();      //方法参数类型
    }
```

##### 示例

全局事件拦截：

```java
    @Before("execution(* android.view.View.OnClickListener.onClick(android.view.View))")
    public void onViewClickBefore(JoinPoint joinPoint) {
        Log.i(TAG, "onViewClickBefore: 点击事件之前执行");
    }

    @After("execution(* android.view.View.OnClickListener.onClick(android.view.View))")
    public void onViewClickAfter(JoinPoint joinPoint) {
        Log.i(TAG, "onViewClickAfter: 点击事件之后执行");
    }
```

但是我们发现其实切点表达式是一样的，那就就可以抽成一个 Pointcut 来复用，即：

```java
    @Before(value = "onViewClick()")
    public void onViewClickBefore(JoinPoint joinPoint) {
        Log.i(TAG, "onViewClickBefore: 点击事件之前执行");
    }

    @After(value = "onViewClick()")
    public void onViewClickAfter(JoinPoint joinPoint) {
        Log.i(TAG, "onViewClickAfter: 点击事件之后执行");
    }

    @Pointcut("execution(* android.view.View.OnClickListener.onClick(android.view.View))")
    public void onViewClick(){

    }
```

方法统计耗时：

```java
    @Around("execution( * com.example.omooo.demoproject.MainActivity.* (..))")
    public void bootTracker(ProceedingJoinPoint joinPoint) throws Throwable {
        long beginTime = SystemClock.currentThreadTimeMillis();
        joinPoint.proceed();
        long endTime = SystemClock.currentThreadTimeMillis();
        long dx = endTime - beginTime;
        Log.i(TAG, joinPoint.getSignature().getDeclaringType().getName() + "#" + joinPoint.getSignature().getName()
                + " " + dx + " ms");
    }
// com.example.omooo.demoproject.MainActivity#playAnimation 2 ms
```

权限申请：

以上两个示例都是按照相似性统一切，而通过自定义注解修饰切人点，能够达到精确切。



#### 参考

[Android 面向切面编程（AOP）](https://www.jianshu.com/p/aa1112dbebc7)
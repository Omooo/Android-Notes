---
UncaughtExceptionHandler
---

#### 目录

1. 思维导图
2. 概述
3. 源码分析
   - Thread#UncaughtExceptionHandler
   - ThreadGroup
   - 总结
4. 参考

#### 思维导图

#### 概述

在虚拟机中，当一个线程如果没有显式处理（即 try catch）异常而抛出时会将该异常事件报告给该线程对象的 UncaughtExceptionHandler 进行处理，如果线程没有设置 UncaughtExceptionHandler，则默认会把异常栈信息输出到终端而使程序直接崩溃。所以如果我们想在线程意外崩溃时做一些处理就可以通过实现 UncaughtExceptionHandler 来满足需求。

#### 源码分析

##### Thread#UncaughtExceptionHandler

```java
public class Thread{
        public interface UncaughtExceptionHandler {
        void uncaughtException(Thread t, Throwable e);
    }

    private volatile UncaughtExceptionHandler uncaughtExceptionHandler;

    private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;

	/**
	 * 静态方法，设置一个全局的异常处理器
	 */
    public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
         defaultUncaughtExceptionHandler = eh;
     }

    public static UncaughtExceptionHandler getDefaultUncaughtExceptionHandler(){
        return defaultUncaughtExceptionHandler;
    }

    /**
     * 注意，如果 uncaughtExceptionHandler 为空就返回 group
     * 这里的 group 其实是 ThreadGroup
     */
    public UncaughtExceptionHandler getUncaughtExceptionHandler() {
        return uncaughtExceptionHandler != null ?
            uncaughtExceptionHandler : group;
    }

    /**
     * 对指定的线程设置异常处理器
     */
    public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
        checkAccess();
        uncaughtExceptionHandler = eh;
    }
}
```

看到这就能明白了 setDefaultUncaughtExceptionHandler 和 setUncaughtExceptionHandler 的区别的。线程崩溃时异常抛出的顺序是先调用 Thread 的 getUncaughtExceptionHandler 查看自己是否有自己对象特有的 handler，如果有就直接处理，没有就调用 group（即 ThreadGroup）的逻辑处理。

##### ThreadGroup

```java
public class ThreadGroup implements Thread.UncaughtExceptionHandler {
    //...
    public void uncaughtException(Thread t, Throwable e) {
        if (parent != null) {
            //通常情况下，parent 为 null
            parent.uncaughtException(t, e);
        } else {
            Thread.UncaughtExceptionHandler ueh =
                Thread.getDefaultUncaughtExceptionHandler();
            if (ueh != null) {
                ueh.uncaughtException(t, e);
            } else if (!(e instanceof ThreadDeath)) {
                System.err.print("Exception in thread \""
                                 + t.getName() + "\" ");
                e.printStackTrace(System.err);
            }
        }
    }
}
```

这里就清楚了，上面返回的 group 其实就是 ThreadGroup，所以它肯定也是实现 UncaughtExceptionHandler 接口的，它 uncaughtException 的处理逻辑就是先拿全局的 UncaughtExceptionHandler，如果为空就按正常流程抛出异常堆栈信息。

##### 总结

1. UncaughtExceptionHandler 是一个接口，我们可以通过实现它来处理程序中未被 try-cache 的异常，它有全局异常处理和针对某个线程的异常处理之分
2. 异常在抛出时，会先检查当前线程是否设置了 UncaughtExceptionHandler，如果有就直接处理，如果没有就跳到 ThreadGroup 中处理，ThreadGroup 也实现了 UncaughtExceptionHandler 接口，它的 uncaughtException 的实现逻辑是判断是否有全局异常处理器，如果有就处理，没有就抛出异常堆栈信息，我们平时的异常信息就是从这里抛出的。

#### 参考

[UncaughtExceptionHandler 相关问题解析](https://mp.weixin.qq.com/s/ghnNQnpou6-NemhFjpl4Jg)


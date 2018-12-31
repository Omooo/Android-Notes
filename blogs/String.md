---
String
---

#### 目录

1. 思维导图
2. 源码解析
   - 类继承关系
   - 类成员变量
   - 类成员方法
   - 相关静态方法
3. 对象内存分配
4. 参考

#### 思维导图

![](https://i.loli.net/2018/12/31/5c29ad8944245.png)

#### 源码解析

##### 类继承关系

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    }
```

String 类由 final 修饰，是一个不可变类。

##### 类成员变量

```java
// JDK 10
private final byte[] value;
private final byte coder;	//采取的编码方式
static final boolean COMPACT_STRINGS;

static {
    COMPACT_STRINGS = true;		//压缩 String 存储空间
}
@Native static final byte LATIN1 = 0;
@Native static final byte UTF16  = 1;
```

在 JDK 9 之前，用的是 char 数组来存储 String 的值，之后就用 byte 数组来存储，char 是两个 byte，比如在存储 ‘A’ 这个字符串时只需一个 byte，就会造成空间浪费。

String 支持多种编码，但是如果不指定编码的话，它可能使用两种编码方式，分别是 LATIN1 和 UTF16，LATIN1 其实就是 ISO 编码，属于单字节编码，而 UTF16 为双字节编码。

String 在表示因为字符或者数字时，会可能存在浪费空间的情况，比如在存储 what 字符串时：

![](https://i.loli.net/2018/12/31/5c29a0e29ddef.png)

在 java 9 之后就变成了：

![](https://i.loli.net/2018/12/31/5c29a1094834b.png)

可以看到，压缩之后存储更加紧凑了。默认是开启压缩的，即 COMPACT_STRINGS 默认为 true。

##### 类成员方法

```java
   //计算长度
	public int length() {
        return value.length >> coder();
    }
    byte coder() {
        return COMPACT_STRINGS ? coder : UTF16;
    }
	//获取指定位置的字符
    public char charAt(int index) {
        if (isLatin1()) {
            return StringLatin1.charAt(value, index);
        } else {
            return StringUTF16.charAt(value, index);
        }
    }
	//...
```

既然改变了编码方式，计算长度就需要考虑编码方式了，如果是 UTF16，双字节编码，那就是右移一位即长度为之前的 1/2。同时，也能看出来，默认采用的是单字节编码即 ISO 编码。

剩下就是 String#intern() 方法：

```java
public native String intern();
```

过于重要，下面解释。

#### 对象内存分配

String 对象创建有两种方式：

1. 字面量赋值

   ```java
   String str = "Omooo"
   ```

   这样创建字符串对象，首先会去常量池中找有没有这个字符串，如果有就直接指向，没有就先往常量池中添加再指向。

2. new 创建

   ```java
   String str = new String("Omooo");
   ```

   当然，我们肯定不会这样写。如果这样写了，它会做两件事。

   首先在堆上创建该字符串对象，然后去看常量池中是否有该字符串，如果有就算了，没有就往常量池中添加一个。

String 对象的内存分配讲完了，那就看这一道题：

```java
String str1 = new String("str")+new String("01");
str1.intern();
String str2 = "str01";
System.out.println(str2==str1);
```

输出 true。

在 JDK 1.7 之后，intern 方法做了些改变，进行拷贝的时候不是拷贝对象，而是拷贝地址值。

那么在想想一下两个呢？

```java
String str1 = new String("str")+new String("01");
String str2 = "str01";
str1.intern();
System.out.println(str2==str1);

String str1 = new String("str")+new String("01");
String str2 = "str01";
str1 = str1.intern();
System.out.println(str2==str1);
```

#### 参考

[String 源码浅析(一)](https://juejin.im/post/5c2588d8f265da6110371d2b)

[Java9后String的空间优化](https://blog.csdn.net/wangyangzhizhou/article/details/80371653)

[String类相关面试题很难？不要方，问题不大](https://www.jianshu.com/p/d416a074409d)
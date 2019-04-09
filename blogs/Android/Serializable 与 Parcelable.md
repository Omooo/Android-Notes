---
Serializable 与 Parcelable
---

#### 目录

1. 前言
2. 概述
3. 具体使用
4. 区别对比
5. Serializable 常见问题
   - serialVersionUID
   - static 与 transient 关键字
   - 父类的序列化
   - 序列化存储规则
   - 自定义序列化和反序列化规则
6. Externalizable
7. 参考

#### 前言

Serializable 和 Parcelable 都是用来对象序列化的，那为什么需要序列化呢？我觉得有三种场景：

1. 持久化存储
2. 通过 Socket 进行网络传输
3. 深拷贝

#### 概述

Serializable 和 Parcelable 是用来序列化和反序列化的，其中 Serializable 是 Java 提供的一个序列化接口，它是一个空接口，专门为对象提供标准的序列化和反序列化操作，使用起来比较简单。而 Parcelable 则稍显复杂，实现该接口重写两个模版方法，并且需要提供一个 Creator。

Serializable 的原理是通过 ObjectInputStream 和 ObjectOutputStream 来实现的，整个序列化过程使用了大量的反射和临时变量，而且在序列化对象的时候，不仅会序列化对象本身，还需要递归序列化对象引用的其他对象。整个过程计算非常复杂，而且因为存在大量反射和 GC 的影响，序列化的性能会比较差。另外一方面，因为序列化文件包含的信息非常多，导致它的大小比 Class 文件本身还要大很多，这样又会导致 I/O 读写上的性能问题。

#### 具体使用

##### Serializable

```java
public class User implements Serializable {

    private static final long serialVersionUID = -541329592684050557L;

    public String name;
    public int age;
    
}
```

##### Parcalable

```java
public class User implements Parcelable {

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
    public String name;
    public int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    protected User(Parcel in) {
        this.name = in.readString();
        this.age = in.readInt();
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
        dest.writeInt(this.age);
    }
}
```

#### 区别对比

##### 实现差异

Serializable 的实现，只需要实现一个 Serializable 接口即可，这只是给对象打了一个标记，系统会自动将其序列化；而 Parcelable 的实现，不仅需要实现 Parcelable 接口并重写模版方法，还要提供一个 CREATOR，并实现 Parcelable.Creator 接口，并实现读写的抽象方法。

##### 效率对比

Serializable 使用 I/O 读写存储在硬盘上，序列化过程中产生大量的临时变量，会引起频繁 GC，效率低下。而 Parcelable 是直接在内存中读写，更加高效。

#### Serializable 常见问题

##### serialVersionUID

它是用来辅助序列化和反序列化的，虽然在序列化的时候系统会自动生成一个 UID，但是还是推荐在手动提供一个 serialVersionUID。序列化操作的时候系统会把当前类的 serialVersionUID 写入序列化文件中，当反序列化的时候会去检测文件中的 serialVersionUID，判断它是否与当前类的 serialVersionUID 一致，如果一致就说明序列化类与当前类版本一致，可以反序列化成功，否则就可能抛异常。手动添加 serialVersionUID 的话，即使当类结构发生变化时，系统也会尽可能的恢复原有类结构，也不至于抛 InvalidClassException 异常。

##### static 与 transient 关键字

静态变量是属于类的，而序列化是保存对象的状态，因此序列化并不保存静态变量；transient 关键字的作用是阻止变量序列化，在反序列化后变量值都被设置为初始值。

##### 父类的序列化

一个子类实现了 Serializable 接口，而父类没有实现 Serializable 接口，序列化该子类对象，然后反序列化后输出父类定义的成员变量的值，该数值与序列化时的数值不同。

要想将父类对象也序列化，就需要让父类对象也实现 Serializable 接口。如果父类对象不实现的话，就需要有默认的无参的构造方法。在父类没有实现 Serializable 接口时，虚拟机是不会序列化父对象的，而一个 Java 对象的构造必须先有父对象，才有子对象，反序列化也不例外。所以反序列化时，为了构造父对象，只能调用父类的无参构造方法作为默认的父对象。因此当我们取父对象的变量值时，它的值是调用父类无参构造函数后的值，即初始值。

因此，我们也可以通过将不需要序列化的成员变量放到未 Serializable 的父类当中，达到和 transient 关键字一样的效果。

##### 序列化存储规则

当对一个对象进行序列化两次写入文件，那么文件大小是不是比序列化一次文件大小的两倍呢？然后再反序列化两次后得到两个对象，这两个对象相等嘛？

事实上，第二次写入文件只增加了 5 个字节，并且两个对象是相等的。

Java 序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件是同一个对象时，并不会再将对象的内容进行存储，而只是再次存储一份引用，上面增加的 5 字节的存储空间就是新增引用和一些控制信息的空间。反序列化时，恢复引用关系，所以两者引用是相等的。

##### 自定义序列化和反序列化规则

在序列化过程中，如果被序列化的类中定义了 writeObject 和 readObject 方法，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户自定义的序列化和反序列化。通过这两个方法，我们可以对某些字段做一些特殊修改，也可以实现序列化的加密功能。

如果没有这样的方法，则默认调用的是 ObjectOutputStream 的 defaultWriteObject 方法和 ObjectInuputStream 的 defaultReadObject 方法。

writeReplace 和 readResovle 方法。这两个方法代理序列化的对象，可以实现自定义返回的序列化实例。那它有什么用呢？通过它们我们可以实现对象序列化的版本兼容。在反序列化生成新的对象的时候会破坏单例，我们可以添加一个 readResolver 方法直接返回单例对象即可。

调用流程：

```java
//序列化
writeReplace、writeObject
//反序列化
readObject、readResolve
```

##### 实例

下面给出示例代码，以说明上文大部分问题。可以看出，一个 Serializable 的确可以挖掘很多知识点呀。

```java
public class ChildBean extends SuperBean implements Serializable {

    private String sex;
    private transient String age;

    public ChildBean(String sex, String age) {
        this.sex = sex;
        this.age = age;
        System.out.println("执行 ChildBean 有参构造方法");
    }

    public ChildBean(){
        System.out.println("执行 ChildBean 无参构造方法");
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "ChildBean{" +
                "sex='" + sex + '\'' +
                ", age='" + age + '\'' +
                '}';
    }
}
```

```java
public class SuperBean{

    protected String name;

    @Override
    public String toString() {
        return "SuperBean{" +
                "name='" + name + '\'' +
                '}';
    }
}
```

```java
public class SerialTest {
    public static void main(String[] args) throws Exception {
        ChildBean childBean = new ChildBean("Sex", "Age");
        childBean.name = "Omooo";
        System.out.println("反序列化前的对象：" + childBean.toString());
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFile"));
        oos.writeObject(childBean);
        oos.flush();
        System.out.println("第一次写入文件的大小：" + new File("tempFile").length());
        oos.writeObject(childBean);
        oos.close();
        System.out.println("第二次写入文件的大小：" + new File("tempFile").length());

        File file = new File("tempFile");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        ChildBean childBean1 = (ChildBean) ois.readObject();
        System.out.println("反序列化后的对象：" + childBean1.toString());
        System.out.println("反序列化后父对象中的成员变量值：" + childBean1.name);
        ChildBean childBean2 = (ChildBean) ois.readObject();
        ois.close();
        System.out.println("反序列化两次对象是否相等：" + (childBean1 == childBean2));
    }
}
```

```
执行 ChildBean 有参构造方法
反序列化前的对象：ChildBean{sex='Sex', age='Age'}
第一次写入文件的大小：80
第二次写入文件的大小：85
反序列化后的对象：ChildBean{sex='Sex', age='null'}
反序列化后父对象中的成员变量值：null
反序列化两次对象是否相等：true
```

#### Externalizable

Externalizable 继承了 Serializable 接口，并且需要实现 writeExternal 和 readExternal 两个方法，用来自定义序列化和反序列化逻辑。这是很有必要的，毕竟如果我们在 Serializable 自定义序列化和反序列化逻辑，需要我们自己提供 writeObject、readObject 方法，方法的参数已经抛出的异常都的我们自己写，每次我都是去看 ArrayList 的实现，有了 Externalizable，我们就可以只关注逻辑本身了。

同时，前面我们说过，Serializable 在反序列化的时候是默认不走构造方法的，所以如果构造方法里面存在某些逻辑就需要注意了，而 Externalizable 是会走无参构造方法的。

示例：

```java
public class ExternalizableTest {

    public static void main(String[] args) throws Exception {
        Bean bean = new Bean();
        bean.setName("Omooo");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("tempFileV2"));
        oos.writeObject(bean);
        oos.flush();
        oos.close();
        System.out.println("静态变量反序列前: " + Bean.age);

        File file = new File("tempFileV2");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        Bean readObject = (Bean) ois.readObject();
        System.out.println(readObject.getName());
    }

    static class Bean implements Externalizable {


        private String name;
        public static int age;

        public Bean() {
            System.out.println("执行无参构造方法");
            age++;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(name);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            this.name = in.readObject().toString();
            System.out.println("静态变量反序列化后: " + Bean.age);
        }
    }
}
```

#### 参考

[Java 对象序列化](https://www.ibm.com/developerworks/cn/java/j-5things1/index.html)

[Java 序列化的高级认识](<https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html>)
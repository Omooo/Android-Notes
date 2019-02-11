---
Serializable 与 Parcelable
---

#### 目录

1. 概述
2. 具体使用
3. 区别对比
4. 参考

#### 概述

Serializable 和 Parcelable 是用来序列化和反序列化的，其中 Serializable 是 Java 提供的一个序列化接口，它是一个空接口，专门为对象提供标准的序列化和反序列化操作，使用起来比较简单。而 Parcelable 则稍显复杂，实现该接口重写两个模版方法，并且需要提供一个 Creator。

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

#### 参考

[序列化与反序列化之 Parcelable 和 Serializable 浅析](https://juejin.im/entry/57e8d42e816dfa005ef310be)
---
说说 ThreadLocal 的原理
---

1. ThreadLocal 的适用于什么场景？
2. ThreadLocal 的使用方式是怎样的？
3. ThreadLocal 的实现原理怎样的？

```java
private static void prepare(boolean quitAllowed){
    sThreadLocal.set(new Looper(quitAllowed));
}
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
public static Looper myLooper(){
    return sThreadLocal.get();
}
```

```java
private static final ThreadLocal<Choreographer> sThreadInstance = new ThreadLocal<Choreographer>(){
    @Override
    protected Choreographer initialValue(){
        Looper looper = Looper.myLooper();
        return new Choreographer(looper);
    }
}
public static Choreographer myLooper(){
    return sThreadLocal.get();
}
```

```java
public T get(){
    Thread currentThread = Thread.currentThread();
    Values values = values(currentThread);
    if(values!=null){
        Object[] table = values.table;
        int index = hash & values.mask;
        if(this.reference == table[index]){
            return (T)table[index + 1];
        }else{
            values = initializeValues(currentThread);
        }
    }
    return (T)values.getAfterMiss(this);
}
Object getAfterMiss(ThreadLocal<?> key){
    Object[] table = this.table;
    int index = key.hash & mask;
    Object value = key.initialVaule();
    table[index] = key.reference;
    table[index + 1] = value;
    size++;
    cleanUp();
    return value;
}

public void set(T value){
    Thread currentThread = Thread.currentThread();
    Values values = values(currentThread);
    if(values == null){
        values = initalizeValues(currentThread);
    }
    values.put(this, value);
}
```

#### 总结

1. 同一个 ThreadLocal 对象，在不同线程 get 返回不同的 value
2. Thread 对象里有张表，保存 ThreadLocal 到 value 的映射关系
3. 这张表是怎么实现的？
4. 是如何解决 hash 冲突的？
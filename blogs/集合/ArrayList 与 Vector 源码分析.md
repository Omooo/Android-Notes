---
ArrayList 和 Vector 源码分析
---

#### 前言

基于JDK1.10。

#### ArrayList

ArrayList实现了List接口、RandomAccess接口，可以插入空数据以及支持随机访问。

ArrayList相当于动态数组，里面有两个重要属性，elementData以及size。

```java
transient Object[] elementData; //数据
private int size;	//数组大小
```

首先看一下**构造方法**（只罗列其中一种）：

```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

也就是说我们可以指定容量大小，这样可以避免扩容导致的性能损耗。

**第一种方式添加数据时：**

```java
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)	//第一步：扩容效验
            elementData = grow();	
        elementData[s] = e;	//第二步：在尾部添加数据并且size+1
        size = s + 1;
    }
```

- 首先是扩容效验
- 把数据插入到最后，并且 size + 1

**第二种方式添加数据时：**

```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);	//第一步：判断插入的index是否合理
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)	//第二步：扩容效验
            elementData = grow();
        System.arraycopy(elementData, index, elementData, index + 1,s - index);	//第三步：拷贝数组，并把index后的数据往后移一位，把数据插入到index位置上，size + 1
        elementData[index] = element;
        size = s + 1;
    }
```

- 首先判断插入的index是否合理，以及依旧进行扩容效验
- 拷贝数组插入数据，size + 1

**扩容效验：**

```java
    private Object[] grow() {
        return grow(size + 1);
    }
    private Object[] grow(int minCapacity) {
        return elementData = Arrays.copyOf(elementData, newCapacity(minCapacity));	//拷贝旧数组，容量大小为以前的1.5倍
    }
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);	//容量扩大为之前的1.5倍
        if (newCapacity - minCapacity <= 0) {
            if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);	//private static final int DEFAULT_CAPACITY = 10;
            if (minCapacity < 0) // overflow
                throw new OutOfMemoryError();
            return minCapacity;
        }
        return (newCapacity - MAX_ARRAY_SIZE <= 0)	//private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
            ? newCapacity
            : hugeCapacity(minCapacity);
    }
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE)
            ? Integer.MAX_VALUE
            : MAX_ARRAY_SIZE;
    }
```

可以看出，如果是非极端的情况下（比如初始容量很小和很大），都会将容量扩大至以前的1.5倍。

所以，ArrayList的性能损耗主要在于数据的拷贝，所以可以适当情况下指定容量大小，应该尽量减少扩容操作，更要避免在指定位置插入数据的操作。

**序列化：**

仔细看的话肯定能看到前面的elementData是用transient修饰的，也就是拒绝数据被自动序列化。因为ArrayList并不是所有位置都有数据，所以没必要全部序列话，应该只序列化有数据的部分。

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioral compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {	//只序列化有数据的部分
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```

#### Vector

和ArrayList很相似，同样实现了List、RandomAccess接口，可以插入空数据以及支持随机访问。内部通过动态数组来实现，不过add、get方法都加了synchronized同步锁。

```java
    public synchronized boolean add(E e) {
        modCount++;
        add(e, elementData, elementCount);
        return true;
    }
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        elementCount = s + 1;
    }
    public void add(int index, E element) {
        insertElementAt(element, index);
    }
```

所以，可以把Vector理解为一个加锁的ArrayList。

#### 更新

1. ArrayList 无参构造器初始化时，默认大小是空数组，并不是大家常说的 10，10 是在第一次 add 的时候扩容的数组值。
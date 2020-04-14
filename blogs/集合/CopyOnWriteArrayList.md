---
CopyOnWriteArrayList
---

#### 引导语

在 ArrayList 的类注释上，JDK 就提醒了我们，如果要把 ArrayList 作为共享变量的话，是线程不安全的，推荐我们自己加锁或者使用 Collections.synchronizedList 方法，其实 JDK 还提供了另外一种线程安全的 List，叫做 CopyOnWriteArrayList，这个 List 具有以下特征：

1. 线程安全的，多线程环境下可以直接使用，无需加锁
2. 通过锁 + 数组拷贝 + volatile 关键字保证了线程安全
3. 每次数组操作，都会把数组拷贝一份出来，在新数组上进行操作，操作成功之后再赋值回去

#### 新增

```java
    public boolean add(E e) {
        synchronized (lock) {
            Object[] es = getArray();
            int len = es.length;
            es = Arrays.copyOf(es, len + 1);
            es[len] = e;
            setArray(es);
            return true;
        }
    }

    public void add(int index, E element) {
        synchronized (lock) {
            Object[] es = getArray();
            int len = es.length;
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException(outOfBounds(index, len));
            Object[] newElements;
            int numMoved = len - index;
            if (numMoved == 0)
                newElements = Arrays.copyOf(es, len + 1);
            else {
                newElements = new Object[len + 1];
                System.arraycopy(es, 0, newElements, 0, index);
                System.arraycopy(es, index, newElements, index + 1,
                                 numMoved);
            }
            newElements[index] = element;
            setArray(newElements);
        }
    }
```

简单的 add 方法，是拷贝原有数组然后插入到数组尾部。而 add 到指定位置时，是需要拷贝两次的。

这里有一个问题，都已经加锁了，为什么还需要拷贝数组呢，而不是在原来数组上面进行操作呢，原因主要为：

1. volatile关键字修饰的是数组，如果我们简单的在原来数组上修改其中某几个元素的值，是无法触发可见性的，我们必须通过修改数组的内存地址才行，也就说要对数组进行重新赋值才行
2. 在新的数组上进行拷贝，对老数组没有任何影响，只有新数组完全拷贝完成之后，外部才能访问到，降低了在赋值过程中，老数组数据变动的影响

#### 移除

移除指定位置的元素：

```java
    public E remove(int index) {
        synchronized (lock) {
            Object[] es = getArray();
            int len = es.length;
            E oldValue = elementAt(es, index);
            int numMoved = len - index - 1;
            Object[] newElements;
            if (numMoved == 0)
                newElements = Arrays.copyOf(es, len - 1);
            else {
                newElements = new Object[len - 1];
                System.arraycopy(es, 0, newElements, 0, index);
                System.arraycopy(es, index + 1, newElements, index,
                                 numMoved);
            }
            setArray(newElements);
            return oldValue;
        }
    }
```

批量移除：

```java
    boolean bulkRemove(Predicate<? super E> filter, int i, int end) {
        // assert Thread.holdsLock(lock);
        final Object[] es = getArray();
        // Optimize for initial run of survivors
        for (; i < end && !filter.test(elementAt(es, i)); i++)
            ;
        if (i < end) {
            final int beg = i;
            final long[] deathRow = nBits(end - beg);
            int deleted = 1;
            deathRow[0] = 1L;   // set bit 0
            for (i = beg + 1; i < end; i++)
                if (filter.test(elementAt(es, i))) {
                    setBit(deathRow, i - beg);
                    deleted++;
                }
            // Did filter reentrantly modify the list?
            if (es != getArray())
                throw new ConcurrentModificationException();
            final Object[] newElts = Arrays.copyOf(es, es.length - deleted);
            int w = beg;
            for (i = beg; i < end; i++)
                if (isClear(deathRow, i - beg))
                    // 赋值到临时数组里面
                    newElts[w++] = es[i];
            System.arraycopy(es, i, newElts, w, es.length - i);
            setArray(newElts);
            return true;
        } else {
            if (es != getArray())
                throw new ConcurrentModificationException();
            return false;
        }
    }
```

在进行批量移除的时候，并不会直接对数组中的元素进行挨个删除，而是先对数组中的值进行循环判断，把我们不需要删除的数据放到临时数组中，最后临时数组中的数据就是我们不需要删除的数据。

这个和 ArrayList 的批量删除的思想类似，所以我们在需要删除多个元素的时候，最好都使用这种批量删除的思想，而不是采用在 for 循环中使用单个删除的方法，单个删除的话，在每次删除的时候都会进行一次数组拷贝，很消耗性能。

#### 迭代

在 CopyOnWriteArrayList 类注释中，明确说明了，在其迭代的过程中，即使数组的原值被改变了，也不会抛出 ConcurrentModificationException 异常，其根源在于数组的每次变动，都会生成新的数组，不会影响老数组，迭代过程中，根本就不会发生迭代数组的变动。
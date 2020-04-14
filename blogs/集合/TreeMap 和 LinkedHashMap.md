---
TreeMap 和 LinkedHashMap
---

#### TreeMap

TreeMap 底层的数据结构就是红黑树，和 HashMap 的红黑树结构一样。

不同的是，TreeMap 利用了红黑树左节点小，右节点大的性质，根据 key 进行排序，使每个元素能够插入到红黑树大小适当的位置，维护了 key 的大小关系，适用于 key 需要排序的场景。

因为底层使用的是平衡红黑树的结构，所以 containsKey、get、put、remove 等方法的时间复杂度都是 log(n)。

TreeMap#put 方法源码：

```java
    public V put(K key, V value) {
        Entry<K,V> t = root;
        // 根节点为空，直接新建
        if (t == null) {
            // compare 方法限制了 key 不能为 null
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            // 自旋找到 key 应该新增的位置
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
		//...
        return null;
    }
```

从源码中，可以得到：

1. 新增节点时，利用了红黑树左小右大的特性，从根节点不断往下查找，直到找到节点为 null 为止，节点为 null 说明到达了叶子节点
2. 查找过程中，发现 key 值已经存在，直接覆盖
3. TreeMap 是禁止 key 为 null 的

#### LinkedHashMap

HashMap 是无序的，TreeMap 可以按照 key 进行排序，那有木有 Map 是可以维护插入顺序的呢？那就是 LinkedHashMap。

LinkedHashMap 本身是继承 HashMap 的，所以它拥有 HashMap 的所有特性，在此基础上还提供了两大特性：

1. 按照插入顺序进行访问
2. 实现了最近最少使用策略

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
    // 链表头
    transient LinkedHashMap.Entry<K,V> head;
    // 链表尾
    transient LinkedHashMap.Entry<K,V> tail;
    static class Entry<K,V> extends HashMap.Node<K,V>{
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next){
            super(hash,ket,value,next);
        }
    }
    // 控制两种访问模式的字段，默认是 false
    // true 按照访问顺序，会把经常访问的 key 放到队尾
    // false 按照插入顺序提供访问
    final boolean accessOrder;
}
```

LinkedHashMap 只提供了单向访问，即按照插入的顺序从头到尾进行访问，不能像 LinkedList 那样可以双向访问。

看源码会发现，我们用 LinkedHashMap 的 get/put 方法，其实都是调用到 HashMap 的 get/put 方法，那么 LinkedHashMap 是如何实现自己独特的特性呢？

其实答案在 HashMap 的三个空方法：

```java
    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }
```

这三个方法是在 HashMap get/put/remove 时会调用的方法，LinkedHashMap 重写了这些方法，然后会在这些方法里面进行排序。

所以，LinkedHashMap 实现的重点有两个：

1. 扩展 HashMap.Entry 使其拥有链表结构
2. 重写 HashMap 里面三个方法
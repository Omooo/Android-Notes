---
HashSet 和 TreeSet 源码解析
---

#### HashSet

HashSet 就是维护一个不包含重复元素的集合。

它的实现很简单，就是通过 HashMap 来做的。

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();
    
	public HashSet() {
        map = new HashMap<>();
    }
        
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
}
```

#### TreeSet

TreeSet 大致的结构和 HashSet 相似，底层组合的是 TreeMap，所以继承了 TreeMap key 能够排序的功能，迭代的时候，也可以按照 key 的排序顺序进行迭代。

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
	private transient NavigableMap<E,Object> m;
	private static final Object PRESENT = new Object();
    
    public TreeSet() {
        this(new TreeMap<>());
    }
    
    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }
}
```

但是和 HashSet 不同的是，TreeSet 可以实现从头或从尾进行遍历，这时是 TreeSet 定义接口，让 TreeMap 去实现的。
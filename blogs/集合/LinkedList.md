---
LinkedList
---

#### 整体结构

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{}
```

既然实现了 Deque 接口，就说明 LinkedList 其实是一个双向链表。

#### 方法对比

| 方法含义 | 返回异常  | 返回特殊值 | 底层实现                                      |
| -------- | --------- | ---------- | --------------------------------------------- |
| 新增     | add(e)    | offer(e)   | 底层实现相同                                  |
| 删除     | remove(e) | poll(e)    | 链表为空时，remove 会抛出异常，poll 返回 null |
| 查找     | element() | peek()     | 链表为空，element 会抛出异常，peek 返回 null  |

Queue 接口注释建议 add 方法操作失败时抛出异常，但 LinkedList 实现的 add 方法一直返回 true。

#### 迭代器

因为 LinkedList 要实现双向的迭代访问，所以我们使用 Iterator 接口肯定不行了，因为 Iterator 只支持从头到尾的访问。Java 新增了一个迭代接口，叫做 ListIterator，这个接口提供了向前和向后的迭代方法，如下所示：

| 迭代顺序         | 方法                                 |
| ---------------- | ------------------------------------ |
| 从尾到头迭代方法 | hasPrevious、previous、previousIndex |
| 从头到尾迭代方法 | hasNext、next、nextIndex             |


---
Bundle、ArrayMap、SparseArray
---

#### 目录

1. 思维导图
2. Bundle
3. ArrayMap
4. SparseArray
5. 参考

#### 思维导图

#### Bundle

在了解 Bundle 之前，先思考一个问题：

Android 为什么要设计出 Bundle 而不是直接使用 HashMap 来进行数据传递？

原因有两个：

1. Bundle 内部是由 ArrayMap 实现的，ArrayMap 在设计上比传统的 HashMap 更多考虑的是内存优化。
2. Bundle 使用的是 Parcelable 序列化，而 HashMap 使用 Serializable 序列化。

#### ArrayMap

ArrayMap 的内部实现是两个数组，一个 int 数组存储 key 的哈希值，一个对象数组保存 key 和 value，内部使用二分法对 key 进行排序，所以在添加、删除、查找数据的时候，都会使用二分法查找，适合小数据量操作，如果在数据量比较大的情况下，那么它的性能将退化。而 HashMap 内部则是数组+链表结构，所以在数据量较小的时候，HashMap 的 Entry Array 比 ArrayMap 占用更多的内存。同时，ArrayMap 避免了自动装箱。

在 Android 中，建议用 ArrayMap 来替换 HashMap。

#### SparseArray

SparseArray 和 ArrayMap 类似，但是 SparseArray 只能存储 key 为 int 型的，它比 ArrayMap 少了计算 key 的哈希值，建议用 SparseArray\<V> 替换 HashMap\<Integer,V>。

类似的还有 SparseIntArray 代替 HashMap\<Integer,Integer> 等。

#### 参考

[Android为什么要设计出Bundle而不是直接使用HashMap来进行数据传递](https://github.com/ZhaoKaiQiang/AndroidDifficultAnalysis/blob/master/02.Android%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E8%AE%BE%E8%AE%A1%E5%87%BABundle%E8%80%8C%E4%B8%8D%E6%98%AF%E7%9B%B4%E6%8E%A5%E4%BD%BF%E7%94%A8HashMap%E6%9D%A5%E8%BF%9B%E8%A1%8C%E6%95%B0%E6%8D%AE%E4%BC%A0%E9%80%92.md)

[如何通过 ArrayMap 和 SparseArray 优化 Android App](https://github.com/xitu/gold-miner/blob/master/TODO/android-app-optimization-using-arraymap-and-sparsearray.md)
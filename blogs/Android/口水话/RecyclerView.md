---
RecyclerView
---

#### 目录

1. 缓存机制
2. 优化措施

#### 缓存机制

#### 优化措施

1.首先考虑数据缓存和分页

如果数据量比较大，可以选择分页加载，常见的比如订单列表。在做积分商城时，有多个 Tab，每个 Tab 根据 typeId 来请求数据，为了避免来回切换请求网络数据，我会把数据用 SparseArray 缓存下来，再次获取数据就不需要网络请求了，这种数据缓存的策略一般用在数据来回切换以及一次就能获取完数据的场景。

2.一些通用的优化手段

比如说避免 ItemView 的过渡绘制、减少布局嵌等，或者在适当情况下不通过 xml 来创建 View，而是直接 new View 来创建。

3.升级 RecyclerView 版本

一般在版本升级中都会静默享受一些优化措施，比如在 25 版本以上，默认添加了 Prefetch 功能，即数据预取功能。

4.如果 Item 高度固定，可以 setHasFixedSize(true) 来避免 requestLayout。如果不需要动画，也可以关闭默认动画。

5.通过 setItemViewCacheSize 来加大 RecyclerView 缓存，如果多个 RecyclerView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以设置 setRecycledViewPool 来共用一个 RecyclerViewPool。

6.对 ItemView 设置监听器，不要对每个 Item 都调用 addXxListener，应该共用一个 Listsner，然后根据 id 来进行不同的操作，避免创过多的内部类。

7.如果一个 ItemView 就占一屏，可以通过重写 LayoutManager 的 getExtraLayoutSpace 来增加 RecyclerView 预留的额外空间，也就是现实范围之外，应该额外缓存的空间。
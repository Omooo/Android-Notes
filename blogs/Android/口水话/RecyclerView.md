---
RecyclerView
---

#### 目录

1. 缓存机制
2. 优化措施

#### 缓存机制

RecyclerView 有四级缓存，这套复用机制的核心在 RecyclerView 的内部类 Recycler 中。

一级缓存是指 mAttachedScrap 和 mChangedScrap，用来缓存屏幕内的 ViewHolder；mAttachedScrap 存储的是当前还在屏幕中的 ViewHolder，按照 id 和 position 来查找 ViewHolder，mChangedScrap 表示数据已经改变的 ViewHolder 列表，存储 notifyXxx 方法时需要改变的 ViewHolder。

二级缓存是指 mCachedViews，用来缓存移除屏幕之外的 ViewHolder，默认情况下缓存容量为 2，可以通过 setViewCacheSize 方法来改变缓存的容量大小。

三级缓存是指 ViewCacheExtension，一个抽象类，需要用户自定义实现，基本上不考虑。

四级缓存是指多个 RecyclerView 共享的 RecyclerViewPool，ViewHolder 缓存池，在有限的 mCachedViews 中如果存不下新的 ViewHolder 时，就会把 ViewHolder 存入 RecyclerViewPool 中，它是按照 Type 来查找 ViewHolder，每个 Type 默认最多缓存 5 个。

取缓存的逻辑就在 Recycler 的 tryGetViewHolderForPositionByDeadline 方法里，逻辑非常清晰，从上到下依次取，取不到就调用 Adapter 的 createViewHolder 创建一个。

存缓存是调用了 Adapter 的 notifyXxx 方法，这时候就会调用到 Recycler 的 scrapView 方法把屏幕上所有的 ViewHolder 回收到 mAttachedScrap 和 mChangedScrap，区别这两种是看 ViewHolder 是否发生了变化。二级缓存和四级缓存是再重新布局时会发生，或者是在复用时，一级缓存的 ViewHolder 失效了就会移至二级缓存，二级缓存满了就移至 RecyclerViewPool。

#### 优化措施

1.首先考虑数据缓存和分页

如果数据量比较大，可以选择分页加载，常见的比如订单列表。在做积分商城时，有多个 Tab，每个 Tab 根据 typeId 来请求数据，为了避免来回切换请求网络数据，我会把数据用 SparseArray 缓存下来，再次获取数据就不需要网络请求了，这种数据缓存的策略一般用在数据来回切换以及一次就能获取完数据的场景。

2.一些通用的优化手段

比如说避免 ItemView 的过渡绘制、减少布局嵌套等，或者在适当情况下不通过 xml 来创建 View，而是直接 new View 来创建。

3.升级 RecyclerView 版本

一般在版本升级中都会静默享受一些优化措施，比如在 25 版本以上，默认添加了 Prefetch 功能，即数据预取功能。

4.如果 Item 高度固定，可以 setHasFixedSize(true) 来避免 requestLayout。如果不需要动画，也可以关闭默认动画。

5.通过 setItemViewCacheSize 来加大 RecyclerView 缓存，如果多个 RecyclerView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以设置 setRecycledViewPool 来共用一个 RecyclerViewPool。

6.对 ItemView 设置监听器，不要对每个 Item 都调用 addXxxListener，应该共用一个 Listener，然后根据 id 来进行不同的操作，避免创过多的内部类。

7.如果一个 ItemView 就占一屏，可以通过重写 LayoutManager 的 getExtraLayoutSpace 来增加 RecyclerView 预留的额外空间，也就是显示范围之外，应该额外缓存的空间。

8.局部刷新
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

1.优先考虑缓存

这里说的缓存是指缓存网络数据，通常在多 Tab + RecyclerView 时，可以考虑将数据缓存下拉
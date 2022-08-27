---
ViewBinding
---

#### 目录

1. 概述
2. 基本使用
3. 进阶使用
4. 与 findViewById 的区别
5. 参考

#### 概述

ViewBinding 本身还是很好使用的，其作用是为了替代 findViewById （或其延伸库如 ButterKnife、ktx 部分能力）操作。启动视图绑定后，系统会为模块内的每个 xml 布局都生成一个绑定类，这个绑定类包含了所有 View Id 的引用。

#### 基本使用

见官方文档：[视图绑定](https://developer.android.com/topic/libraries/view-binding)

#### 进阶使用

使用 Kotlin 委托可以简化 ViewBinding 的使用，见：

[https://github.com/yogacp/android-viewbinding](https://github.com/yogacp/android-viewbinding)

[https://github.com/androidbroadcast/ViewBindingPropertyDelegate](https://github.com/androidbroadcast/ViewBindingPropertyDelegate)

#### 与 findViewById 的区别

与使用 findViewById 相比，视图绑定有一些显著的优点：

1. Null 安全。

   由于视图绑定会创建对视图的直接引用，因此不存在因视图 ID 无效而引发的空指针异常。此外，如果视图仅出现在布局的某些配置中，则绑定类中包含其引用的字段会使用 @Nullable 标记。比如对于同名的日夜间模式两个 xml layout，如果一个 View 仅出现其中一个 xml 中，那么该引用会被声明为可空。

2. 类型安全

   每个绑定类中的字段均具有与它们在 xml 文件中引用的视图相匹配的类型，这意味着不存在类型转换异常。如果在两个 xml 中，同一个 View id 对应于不同类型的 View ，那么该引用的类型会被声明成 View，比如在两个 xml 里面 id 一致，但是类型分别是 TextView 和 Button，绑定类中该 id 的 类型就是 View 而非 TextView（即使 TextView 是 Button 的父类）。

#### 参考

[https://developer.android.com/topic/libraries/view-binding](https://developer.android.com/topic/libraries/view-binding)

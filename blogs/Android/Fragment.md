---
Fragment
---

#### 目录

1. 思维导图
2. 概述
3. 设计原因
4. 基本使用
   - xml 声明
   - 代码设置
   - 添加没有 UI 的 fragment
5. 生命周期
6. 管理 Fragment 和执行事务
7. 与 Activity 通信
8. 常见问题汇总
   1. 创建 Fragment 实例传递数据
   2. getActivity() 引用问题
   3. FragmentTransaction 的 add 和 replace 区别
   4. getChildFragmentManager()
   5. 回退栈的理解
   6. Fragment 重叠
   7. onActivityResult()
9. 参考

#### 思维导图

![](https://i.loli.net/2019/02/16/5c67a742033b0.png)

#### 概述

Fragment 译为 “片段”，必须始终嵌入在 Activity 中，其生命周期直接受宿主 Activity 生命周期的影响。当你以片段作为 Activity 布局的一部分添加时，它存在于 Activity 视图层次结构的某个 ViewGroup 内部，并且片段会定义其自己的视图布局，可以通过在 Activity 的布局文件中以 \<fragment> 形式插入，或者通过代码进行插入。

**不过，Fragment 并非必须成为 Activity 布局的一部分，可以根据需要将没有 UI 的 Fragment 作为 Activity 的不可见工作线程。**下文中，我们会根据 Google 提供的一个实例来分析其作用。

#### 设计原因

既然已经有了 Activity 来展示 UI，为什么还需要 Fragment 呢？

根据官方文档的说法，Android 在 Android 3.0 引入的 Fragment，主要是为了给大屏幕（如平板电脑）更加动态和灵活的 UI 设计提供支持。

当然，我觉得在我们的实际开发中，Fragment 可以把 Activity 分离出多个可重用的组件，它们都有自己的生命周期，同时Fragment 之间的切换更加流畅，比 Activity 更加轻量。

#### 基本使用

基本使用分为在 xml 里面声明和代码设置。

##### xml 声明

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment
        android:id="@+id/fragment"
        android:name="com.example.omooo.fragment.FirstFragment"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
</LinearLayout>
```

当系统创建此 Activity 布局时，会实例化布局中指定的每个片段，并为每个片段调用 onCreateView 方法，系统会直接插入片段返回的 View 来替代 \<fragment> 元素。

**注意：**

每个片段都需要一个唯一的标识符，重启 Activity 时，系统可以使用该标识符来恢复片段（也可以使用该标识符来获取 fragment 以执行某些事务，比如将其移出），可以通过三种方式为 fragment 提供 ID：

1. 通过 android:id 属性提供唯一 ID
2. 通过 android:tag 属性提供唯一字符串
3. 如果都没设置，系统会使用容器视图的 ID

##### 代码设置

```java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction transaction = fragmentManager.beginTransaction();
transaction.add(R.id.fl_content, new FirstFragment());
transaction.commit();
```

##### 添加没有 UI 的 fragment

上面的两种使用都是向 Activity 添加 UI，不过，我们还可以使用 Fragment 为 Activity 提供后台行为，而不需要显示 UI。

要想添加没有 UI 的 fragment，请使用 FragmentTransaction#add(Fragment,String)，第二个参数即 tag，这是标识它的唯一方式，之后，我们可以通过 findFragmentByTag 获取该 Fragment。

以没有 UI 的 Fragment 为 Activity 提供后台行为，这里直接看 Google 给的 APIDemos 的 [FragmentRetainInstance](https://android.googlesource.com/platform/development/+/master/samples/ApiDemos/src/com/example/android/apis/app/FragmentRetainInstance.java) 实例就行了，里面有一个有意思的点：

1. setRetainInstance(true)

   设置为 true，当 Activity 异常销毁时，Fragment 不会被重新创建，它的实例会保留，当重建 Activity 时，会自动拿来用。

不过，平心而论，我并不觉得这个例子有什么指导意义。。

#### 生命周期

![](https://i.loli.net/2019/02/15/5c6656e256348.png)

#### 管理 Fragment 和执行事务

##### 管理 Fragment

管理 Fragment，需要用到 FragmentManager，想用获取它，可以通过在 Activity 中调用 getSupportFragmentManager()。

通过 FragmentManager 执行的操作包括：

1. 通过 findFragmentById 或 findFragmentByTag 获取 Fragment
2. 通过 popBackStack 将 Fragment 从返回栈中弹出
3. 通过 addOnBackStackChangedListener 注册一个侦听返回栈变化的侦听器
4. 通过 beginTransaction 获取 FragmentTransaction

##### 执行事务

在 Activity 中使用 Fragment 一大优点是，可以根据用户行为通过它们执行添加、移出、替换以及其他操作。

常用的 API 有 add()、remove()、replace() 方法，如果想把 Fragment 添加到返回栈，则可以使用 addToBackStack()，不过最后所有的操作都需要 commit。

对于每个 Fragment 事务，都可以在提交前调用 setTransition() 来设置过渡动画。

调用 commit() 不会立即执行事务，而是在 UI 线程可以执行该操作时再安排其他线程上运行。不过，如果有必要，可以在 UI 线程调用 commitNow() 以立即执行 commit 提交的事务。通常不必这样做，除非其他线程中的作业依赖该事务。

注意：只能在 Activity 保存其状态之前使用 commit 提交事务，如果试图在该时间之后提交，则会引发异常。这是因为如需恢复 Activity，则提交后的状态可能会丢失。对于丢失提交无关紧要的情况，可以使用 commitAllowingStateLoss()。

#### 与 Activity 通信

Fragment 可以通过 getActivity() 获取 Activity 实例，以此来调用具体实例 Activity 中的公有方法。

Activity 也可以使用 findFragmentById 或 findFragmentByTag 来获取 Fragment 实例，以此来调用具体 Fragment 内的公有方法。

当然，最推荐的做法就是接口回调。

#### 常见问题汇总

1. 创建 Fragment 实例传递数据

   ```java
   public static OneFragment newInstance(int args){
       OneFragment oneFragment = new OneFragment();
       Bundle bundle = new Bundle();
       bundle.putInt("someArgs", args);
       oneFragment.setArguments(bundle);
       return oneFragment;
   }
   @Override
   public void onCreate(@Nullable Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       Bundle bundle = getArguments();
       int args = bundle.getInt("someArgs");
   }
   ```

   之所以不用带参的构造方法，原因在于 Activity 在一些特殊情况下会发生销毁并重建的情形，比如屏幕旋转、内存吃紧等；对应的，依附于 Activity 存在的 Fragment 也会发生类似的状况。而一旦重建，Fragment 便会调用默认的无参构造函数，导致无法执行有参构造函数进行初始化工作。

2. getActivity() 引用问题

   当 Fragment 中存在类似网络请求之类的异步耗时任务时，该任务执行完毕回调 Fragment 的方法用到 Activity 对象时，可能宿主 Activity 已经销毁，从而引发空指针异常，所以最好都判空。

   一般情况下，获取 Context，可以通过 getContext() 获取。

3. FragmentTransaction add 和 replace 区别

   最开始我觉得使用 add 的话在 addBackStack，在按返回键时应该会退到上一个 Fragment，而使用 replace 的在 addBackStack，replace 会把之前的 fragment 清除掉，所以在按返回键时只是把当前 fragment 清除掉，并不会回退到上一个 fragment。

   这完全是错误的理解！

   ```java
       public void showFirstFragment(View view) {
           mFragmentManager = getSupportFragmentManager();
           mFragmentTransaction = mFragmentManager.beginTransaction();
           mFragmentTransaction.add(R.id.fl_content, new FirstFragment());
           mFragmentTransaction.addToBackStack("fragment1");
           mFragmentTransaction.commit();
       }
   
       public void showSecondFragment(View view) {
           //注意：这里重新实例化了一次 FragmentTransaction
           //不然多次调用同一个 FragmentTransaction 的 commit 会崩溃
           mFragmentTransaction = mFragmentManager.beginTransaction();
           mFragmentTransaction.replace(R.id.fl_content, new SecondFragment());
           mFragmentTransaction.addToBackStack("fragment2");
           mFragmentTransaction.commit();
       }
   
       public void pop(View view) {
           mFragmentManager.popBackStack();
       }
   ```

   这里要分开理解，add 和 replace 影响的只是界面，而控制回退的，是事务。add  的时候是把一个 Fragment 添加到容器里，多次 add 时，视图是添加多层；而使用 replace 是先 remove 掉相同 id 的所有 Fragment，然后在 add。

   而至于会退栈，和事务有关，跟使用 add 还是 replace 没有任何关系。

4. getChildFragmentManager() 

   在 Activity 前乳 Fragment 时，需要使用 FragmentManager，通过 Activity 提供的 getSupportFragmentManager() 方法即可获取，用于管理 Activity 里面嵌入的所有一级 Fragment。

   然后有时候，我们会在 Fragment 里面继续嵌入多级 Fragment，这时候就需要通过 Fragment 来获取 FragmentManager 对象。

   ```java
   FirstFragment firstFragment = new FirstFragment();
   FragmentManager fragmentManager = firstFragment.getChildFragmentManager();
   ```

5. 回退栈的理解

   通过 addToBackStack 保存当前事务，当用户按下返回键时，如果回退栈中保存有之前的事务，便会执行事务回退，而不是 finsh 掉当前 Activity。

6. Fragment 重叠问题

   前面说过，当 Activity 销毁并重建的时候，Activity 重新执行 onCreate 方法，那么不就是创建两次 Fragment 而导致 UI 重叠嘛？

   解决方法也很简单：

   ```java
       protected void onCreate(@Nullable Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_fragment);
           mFrameLayout = findViewById(R.id.fl_content);
   
           if (savedInstanceState != null) {
               mFirstFragment = (FirstFragment) mFragmentManager.findFragmentByTag("fragment1");
           } else {
               mFirstFragment = FirstFragment.newInstance();
               mFragmentTransaction.add(mFirstFragment, "fragment1");
           }
       }
   ```

7. onActivityResult()

   Fragment 类提供了 startActivityForResult 方法用于 Activity 间的页面跳转和数据回传，其实内部也是调用了 Activity 的对应方法，但是在页面返回时 Fragment 没有提供 setResult 方法，但是可以通过拿宿主 Activity 实现。

   但是仍要注意的一点是，嵌套的 Fragment 需要一级一级的分发。

#### 参考

[Fragment 官方文档](https://developer.android.com/guide/components/fragments?hl=zh-cn)

[Android Fragment 的使用，一些你不可不知的注意事项](http://yifeng.studio/2016/12/15/android-fragment-attentions/)
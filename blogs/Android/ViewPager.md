---
ViewPager
---

#### 目录

1. 思维导图
2. 前言
3. 自定义 PageTransform
4. 常见问题
5. 参考

#### 思维导图

#### 前言

关于 ViewPager 的使用就不多说了，相信大家都会玩。写这篇文章的主要目的在于总结一下 PageTransform 该怎么玩？它是用来控制切换动画。

![](https://i.loli.net/2019/01/12/5c394dcc6c369.gif)

这种效果现在已经很普遍了，重点就在自定义 PageTransform。

首先左右边距的显示该怎么做呢？很简单，只需两步：

1. clipChildren 属性设置父布局不裁剪子 View

   ```java
   <LinearLayout
       xmlns:android="http://schemas.android.com/apk/res/android"
       android:orientation="vertical"
       android:clipChildren="false"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <android.support.v4.view.ViewPager
           android:id="@+id/vp"
           android:clipChildren="false"
           android:layout_marginLeft="30dp"
           android:layout_marginRight="30dp"
           android:layout_width="match_parent"
           android:layout_height="200dp"/>
   </LinearLayout>
   ```

2. 设置页面边距

   ```
   mViewPager.setPageMargin(30);
   ```

完成之后页面就是以下效果了：

![](https://i.loli.net/2019/01/12/5c3951ae75faf.png)

切换的动画效果就得依赖于 PageTransform 了。

#### 自定义 PageTransform

```java
public class MyPageTransform implements ViewPager.PageTransformer {
    @Override
    public void transformPage(@NonNull View view, float v) {
        int width = view.getWidth();
        //给不同状态的页面设置不停的效果
        //通过 v 的值来判断页面所处的状态
        if (v < -1) {   //滑出得页面
            view.setScrollX((int) (width * 0.75 * -1));
        } else if (v <= 1) {
            //虽然代码一样，但是值不一样
            if (v < 0) {   //正在消失的页面
                view.setScrollX((int) (width * 0.75 * v));
            } else { //正在进入的页面
                view.setScrollX((int) (width * 0.75 * v));
            }
        } else { //划入的页面
            view.setScrollX((int) (width * 0.75));
        }

    }
}
```

参数 View 是指当前的页面以及前后的页面，v 可以判断是哪个页面。v 的取值是[-1,1]。

当前页面 v 等于 0，前一个页面的值是 -1，后一个页面的值是 1。在滑动的时候，其实就是 v 值的变化。

#### 常见问题

##### FragmentPagerAdapter 和 FragmentStatePagerAdapter 的区别

FragmentPagerAdapter 对于不再需要的 fragment，选择调用 onDetach() 方法，仅销毁视图，不会销毁 fragment 实例。

FragmentStatePagerAdapter 会销毁不再需要的 fragment，当当前事务提交以后，会彻底的将 fragment 从当前 FragmentManager 中移除，state 表明，销毁时，会将其 onSaveInstanceState(Bundle outState) 中的 bundle 信息保存下来，当用户切换回来，可以通过该 bundle 恢复生成新的 fragment，也就是说，你可以在 onSaveInstanceState(Bundle outState) 方法中保存一些数据，在 onCreate 中进行恢复创建。

由此可见，使用 FragmentStatePagerAdapter 更省内存，但是销毁后新建也是需要时间的。一般情况下，如果是结合 TabLayout 展示四五个 Tab 的话，可以直接使用 FragmentPagerAdapter，如果 ViewPager 展示特别多的条目时，建议使用 FragmentStatePagerAdapter。
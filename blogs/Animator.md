---
Animator
---

#### 目录

1. 思维导图
2. 帧动画
   - 使用方式
   - 优缺点
   - 应用场景
3. 补间动画
   - 位移动画
   - 缩放动画
   - 旋转动画
   - 透明度动画
   - 组合动画
   - 优缺点
   - 应用场景
4. 属性动画
5. 参考

#### 思维导图

#### 帧动画

##### 使用方式

1. xml 定义方式

   在 drawable 目录下定义一个 drawable-list，然后给图片设置 res 就好了。

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <animation-list xmlns:android="http://schemas.android.com/apk/res/android">
       <item android:drawable="@drawable/ic_vip" android:duration="100"/>
       <item android:drawable="@drawable/ic_account" android:duration="100"/>
       <item android:drawable="@drawable/ic_clouse" android:duration="100"/>
   	//...
   </animation-list>
   ```

   ```java
   mIvFrame.setImageResource(R.drawable.anim_frame);
   AnimationDrawable drawable = (AnimationDrawable) mIvFrame.getDrawable();
   drawable.start();
   drawable.stop();
   ```

2. Java 代码实现

   其实就是用代码构建 AnimationDrawable。

   ```java
   mAnimationDrawable=new AnimationDrawable();
   for (int i = 0; i < 10; i++) {
       if (i<5){
       mAnimationDrawable.addFrame(getDrawable(R.drawable.ic_vip),100);
       }else {
          mAnimationDrawable.addFrame(getDrawable(R.drawable.ic_clouse),100);
       }
   }
   mIvFrame.setImageDrawable(mAnimationDrawable);
   mAnimationDrawable.start();
   mAnimationDrawable.stop();
   ```

##### 优缺点

优点：使用简单

缺点：图片是全部加载到内存中，可能导致 OOM。

##### 应用场景

基本上很少使用，可以看成 GIF 图。

#### 补间动画

##### 位移动画

1. xml 定义方式

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <set xmlns:android="http://schemas.android.com/apk/res/android">
       <translate
           android:duration="1000"
           android:toXDelta="100"
           android:toYDelta="100"
           android:fromYDelta="0"
           android:fromXDelta="0"/>
   </set>
   ```

   ```java
   mAnimation = AnimationUtils.loadAnimation(mContext, R.anim.anim_trans);
   mIvFrame.startAnimation(mAnimation);
   ```

2. Java 代码实现方式

   ```java
   TranslateAnimation animation = new TranslateAnimation(0, 100, 0, 100);
   animation.setDuration(1000);
   animation.setFillAfter(true);
   mIvFrame.startAnimation(animation);
   ```

##### 其他动画

类似，不举例了。

##### 优缺点

优点：使用简单

缺点：只改变显示，不改变实际属性

##### 应用场景

基本上大部分动画都能实现，常见于 Activity 转场动画等等

#### 属性动画

![](https://i.loli.net/2018/12/29/5c271c4643fc8.png)



#### 参考

[Android 动画 Animator 家族使用指南](https://juejin.im/post/5c245f086fb9a049c043146a)
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
   - 位移、旋转、缩放、透明度动画
   - 优缺点
   - 应用场景
4. 属性动画
   - 层次关系
   - ValueAnimator
     - ObjectAnimator
     - TimeAnimator
   - AnimatorSet
5. 插值器、估值器
   - TypeEvaluator
     - IntEvaluator
     - FloutEvaluator
     - ArgbEvaluator
   - TimeInterpolator / Interpolator / BaseInterpolator
     - LinearInterpolator
     - AccelerateDecelerateInterpolator
6. 参考

#### 思维导图

![](https://i.loli.net/2018/12/30/5c281a641cf56.png)

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

##### 层次关系

Animator

- AnimatorSet
- ValueAnimator
  - ObjectAnimator
  - TimeAnimator

##### ValueAnimator

常用使用方式：

```java
    private void setAnimator() {
        mValueAnimator = new ValueAnimator();
        mValueAnimator.setIntValues(0, 500);
        mValueAnimator.setDuration(2000);
        mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                Log.i(TAG, "onAnimationUpdate: " + animation.getAnimatedValue());
                float y = ((int) animation.getAnimatedValue()) * 1.0f;
                mIvFrame.setTranslationY(y);
            }
        });
        mValueAnimator.start();
    }
```

##### ObjectAnimator

```java
        ObjectAnimator animator = ObjectAnimator
                .ofFloat(mBtnStart, "translationY", 0, 200)
                .setDuration(2000);
        animator.start();
```

ObjectAnimator 在每次更新的时候会自动走 setXxx 方法，所以就不需要像 ValueAnimator 一样手动添加监听器，但是 ValueAnimator 的灵活性更好。

##### TimeAnimator

同样是继承至 ValueAnimator，但是它只做一件事：提供一个时间流，每 18ms 回调一次。

```java
        mTimeAnimator = new TimeAnimator();
        mTimeAnimator.setTimeListener(new TimeAnimator.TimeListener() {
            @Override
            public void onTimeUpdate(TimeAnimator animation, long totalTime, long deltaTime) {
                //动画执行的总时间_动画从上一桢到当前桢的间隔时间，单位都是毫秒
                Log.i(TAG, "onTimeUpdate: " + totalTime + "   " + deltaTime);
            }
        });
```

运行结果：

```java
2018-12-30 09:01:54.344 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 0   0
2018-12-30 09:01:54.346 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 0   0
2018-12-30 09:01:54.356 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 7   7
2018-12-30 09:01:54.375 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 24   17
2018-12-30 09:01:54.388 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 42   18
2018-12-30 09:01:54.407 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 60   18
2018-12-30 09:01:54.424 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 78   18
2018-12-30 09:01:54.444 6656-6656/com.example.omooo.demoproject I/AnimatorActivity: onTimeUpdate: 96   18
```



#### 估值器、插值器

估值器表示属性的从初始值过渡到结束值变化的具体数值，而插值器则表示变化率，比如先加速后减速（默认）、匀速等等。

##### 估值器

估值器都需要实现 TypeEvaluator 接口：

```java
public interface TypeEvaluator<T> {
    public T evaluate(float fraction, T startValue, T endValue);
}
```

其实就是根据初始值和结束值算出一个值。

系统已经有几个默认实现，比如 ArgbEvaluator、IntEvaluator、FloatEvaluator 等等。其实我们在用 ValueAnimator.ofArgb() 的时候，内部就是用 ArgbEvaluator 实现的，那对于 ValueAnimator.ofInt()、ValueAnimator.ofFloat() 方法是不是也是一样的道理呢？

下面将介绍如何自定义估值器，所实现的功能：

![](https://i.loli.net/2018/12/29/5c274685d3572.gif)

首先定义按钮所需要的属性：

```java
public class ButtonInfo {
    public int color;
    public int x;
    public int y;

    public ButtonInfo(int color, int x, int y) {
        this.color = color;
        this.x = x;
        this.y = y;
    }

    public ButtonInfo() {
    }
}
```

自定义估值器：

```java
public class ButtonEvaluator implements TypeEvaluator {

    @Override
    public Object evaluate(float fraction, Object startValue, Object endValue) {
        ButtonInfo start = (ButtonInfo) startValue;
        ButtonInfo end = (ButtonInfo) endValue;
        ButtonInfo buttonInfo = new ButtonInfo();
        ArgbEvaluator argbEvaluator = new ArgbEvaluator();
        buttonInfo.color = (int) argbEvaluator.evaluate(fraction, ((ButtonInfo) startValue).color, ((ButtonInfo) endValue).color);
        buttonInfo.x = (int) (fraction * (end.x - start.x) + start.x);
        buttonInfo.y = (int) (fraction * (end.y - start.y) + start.y);
        return buttonInfo;
    }
}
```

运用：

```java
    private void setTransAnimator() {
        mValueAnimator = ValueAnimator.ofObject(new ButtonEvaluator(), new ButtonInfo(0xff94E1F7, 0, 0)
                , new ButtonInfo(0xffF35519, 500, 500));
        mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                ButtonInfo buttonInfo = (ButtonInfo) animation.getAnimatedValue();
                mBtnStart.setBackgroundColor(buttonInfo.color);
                mBtnStart.setTranslationX(buttonInfo.x);
                mBtnStart.setTranslationY(buttonInfo.y);
            }
        });
        mValueAnimator.setDuration(5000);
        mValueAnimator.start();
    }
```

可以，看出，实际上自定义估值器还是很简单。

##### 插值器

插值器需要实现 TimeInterpolator 或 Interpolator：

```java
public interface TimeInterpolator {
    float getInterpolation(float input);
}

public interface Interpolator extends TimeInterpolator {
	
}
```

Android 系统内置了几种实现，默认是先加速后减速，即 AccelerateDecelerateInterpolator ，看一下源码实现：

```java
public class AccelerateDecelerateInterpolator extends BaseInterpolator
        implements NativeInterpolatorFactory {
    public float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
	//...
}
```

四五十行代码，首先明确的就是 value 值是 0 ~ 1，所以以上就是表示余旋函数在 pi/2 和 pi 之间，就是先加速后减速。

那我们在看一下 LinearInterpolator 的源码：

```java
/**
 * An interpolator where the rate of change is constant
 */
@HasNativeInterpolator
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
    public float getInterpolation(float input) {
        return input;
    }
    //...
}
```

那在自定义插值器就易如反掌了呀，定义一个先减速后加速的插值器，如下：

```java
public class DeceAcceInterpolator implements TimeInterpolator {
    @Override
    public float getInterpolation(float input) {
        //余旋函数 0 ~ PI/2
        return (float) Math.cos(input * Math.PI / 2);
    }
}
```

然后给 ValueAnimator 设置自定义的插值器：

```java
mValueAnimator.setInterpolator(new DeceAcceInterpolator());
```

![](https://i.loli.net/2018/12/29/5c274e1d38083.gif)

啊？怎么反向了？看来设置变化率需要在 x 轴的下方呀。至于为什么？留个坑。

#### 参考

[Android 动画 Animator 家族使用指南](https://juejin.im/post/5c245f086fb9a049c043146a)
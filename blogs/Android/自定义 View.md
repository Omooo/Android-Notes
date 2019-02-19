---
自定义 View 
---

#### 目录

1. 前言
2. 基础知识储备
   - 坐标系
   - 颜色
3. 自定义 View 
   - 分类和流程
   - Canvas 之绘制图形
   - Canavs 之画布操作
4. 实战
5. 参考

#### 前言

自定义 View 系列直接看 [GcsSloop 自定义 View 系列](http://www.gcssloop.com/customview/CustomViewIndex/) 就可以了，熟悉了 API 能自定义简单的 View，该系列最后都一个例子来练习，可以参考 [https://github.com/Omooo/ChartsDemo](https://github.com/Omooo/ChartsDemo) 中的代码，没错，也是我的～

这些知识长时间不实践就忘的差不多了，于是再来一遍。

#### 基础知识储备

##### 坐标系

Android 中的屏幕坐标系是以屏幕的左上角为坐标原点的，向右为 x 正轴，向下是 y 正轴。

这里就要提一下 View 的坐标系了，View 的坐标系统是相对于父控件而言的：

```java
getTop()	//获取子 View 左上角到父 View 顶部的距离
getLeft()	//获取子 View 左上角到父 View 左边的距离
getBottom() //获取子 View 右下角到父 View 顶部的距离
getRight()	//获取子 View 右上角到父 View 左边的距离
    
getBottom() - getTop() = View 的高
getRight() - getLeft() = View 的宽
```

MotionEvent 中的 getXxx 和 getRawXxx 的区别：

```java
event.getX()	//触摸点相对于其所在 View 坐标系的坐标
event.getY()
    
event.getRawX()	//触摸点相对于屏幕坐标系的坐标
event.getRawY()	    
```

##### 颜色

Android 支持的颜色模式有：

| 颜色模式 | 备注                 |
| -------- | -------------------- |
| ARGB8888 | 四通道高精度（32位） |
| ARGB4444 | 四通道低精度（16位） |
| RGB565   | 屏幕默认模式（16位） |

RGB 代表红绿蓝三原色，A 代表透明度，后面的数值表示该类型用多少位二进制来描述。

```java
#f00	//低精度 - 不带透明通道红色
#af00	//低精度 - 带透明通道红色
#ff0000		//高精度 - 不带透明通道红色
#aaff0000	//高精度 - 带透明通道红色	
```

有了基础知识储备，接下来就开始进入自定义 View 了～～～

#### 自定义 View 分类和流程

自定义 View 可以分为两类：一类是自定义 ViewGroup，另一种是自定义 View。自定义 ViewGroup 一般是利用已有的 View 按照特定的布局方式来实现新的组件，比如带自动换行的水平的线性布局等。自定义 View 一般是由于没有现成的 View 可以使用，需要自己实现 onDraw 来绘制。

自定义 View 的流程也是一个通用的套路：

![](https://i.loli.net/2019/02/18/5c6a2e4942328.jpg)

##### 构造函数

```java
public class MyCustomView extends View {

    //在 Activity 中以 new MyCustomView(this) 创建 View
    public MyCustomView(Context context) {}

    //在 xml 中创建 View
    public MyCustomView(Context context, @Nullable AttributeSet attrs) {}

    //为 View 指定样式
    public MyCustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {}

    //API > 21
    public MyCustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {}
```

我们只需要实现前两个构造函数即可，AttributeSet 用于获取自定义属性等。

##### onMeasure()

用于测量 View 的大小。

你可能会问，既然我们在 xml 里面可以指定 View 的宽高尺寸，为什么还需要自己测量呢？

这是因为，View 的大小不仅由自身所决定，同时也会受父控件的影响，比如我们设置 warp_content 或 match_parent。

```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //获取宽度尺寸和宽度测量模式
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        
        setMeasuredDimension(widthSize, heightSize);
    }
```

onMeasure 中的参数可以翻译成测量规格，它有两部分组成：宽高实际尺寸和宽高测量模式。

测量模式有三种：

| 模式        | 二进制值 | 描述                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| UNSPECIFIED | 00       | 默认值，父控件没有给子 View 任何限制，子 View 可以设置为任意大小，一般用在系统中，我们可以不管 |
| EXACTLY     | 01       | 表示父控件已经确切指定了子 View 的大小，对应于 match_parent 和 确切数值 100dp |
| AT_MOST     | 10       | 表示子 View 的大小存在上限，一般是父 View 大小，对应于 warp_content |

所以在测量规格中，只需要两个 bit 就能表示完测量模式，而事实上正是这样做的，测量规格是一个 int 数值，32位，前两位表示测量模式，后三十位表示测量数值。

##### onSizeChanged()

在视图大小发生改变时调用。

既然在测量完 View 并使用 setMeasuredDimension 函数之后 View 的大小基本上已经确定了，那为什么还要再次确认 View 的大小呢？

这是因为 View 的大小不仅由 View 本身控制，而且受父控件的影响，所以我们在确定 View 大小的时候最好使用系统提供的 onSizedChanged 回调函数。

```java
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
    }
```

w、h 即是 View 最终的大小。

##### onLayout()

确定布局的函数是 onLayout，它用于确定子 View 的位置，在定义 ViewGroup 中会用到，它调用的是子 View 的 layout 函数。

在自定义 ViewGroup 中，onLayout 一般是循环取出子 View，然后经过计算得出各个子 View 位置的坐标值，然后用以下函数设置子 View 位置。

```java
child.layout(l,t,r,b)
```

##### onDraw()

onDraw 是实际绘制的部分，使用 Canvas 绘制。

```java
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
    }
```

#### 自定义 View 之 Canvas 绘制基本图形


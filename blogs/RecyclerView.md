---
RecyclerView
---

#### 目录

1. 思维导图

2. 基本使用

   - 复杂布局的实现、添加头布局、尾布局
   - 上拉刷新、下拉加载

3. 高级玩法

   - LayoutManager

   - ItemDecoration
   - ItemAnimator
   - ItemTouchHelper
   - 结合 SnapHelper
   - 万能 Adapter

4. 源码分析系列

   - DefaultItemAnimator
   - 缓存机制
     - ListView 的 RecycleBin
     - RecyclerView 的 Recycler
     - 局部刷新
     - 两者区别

5. 其他

   - 扩展 RecyclerView

   - 嵌套滑动
   - 与 ListView 对比
     - RecyclerView 优缺点
     - ListView 优缺点

6. 参考

#### 思维导图

#### 基本使用

##### 复杂布局的实现

其实就是多个 ItemType 的场景，实现起来也很简单。定义多个 ItemTpye 和 ViewHolder，在 onCreateViewHolder 中通过 itemType 返回不同的 ViewHolder，onBindViewHolder 时根据 ViewHolder 的不同在设置不同的数据，完事。

这里想说一下头布局和尾布局的实现方式，其实也可以用上面的方式解决，但是我们可以用一种更加优雅的方式解决，那就是使用装饰者模式来实现扩展。

```java
public class BaseRvAdapterWrapper extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final int TYPE_HEADER = 0;
    private static final int TYPE_NORMAL = 1;
    private static final int TYPE_FOOTER = 2;
    private BaseRvAdapter mBaseRvAdapter;
    private View mHeaderView;
    private View mFooterView;

    public BaseRvAdapterWrapper(BaseRvAdapter baseRvAdapter) {
        mBaseRvAdapter = baseRvAdapter;
    }

    public void setHeaderView(View headerView) {
        mHeaderView = headerView;
    }

    public void setFooterView(View footerView) {
        mFooterView = footerView;
    }

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
        if (i == TYPE_HEADER) {
            return new RecyclerView.ViewHolder(mHeaderView) {
            };
        } else if (i == TYPE_NORMAL) {
            return mBaseRvAdapter.onCreateViewHolder(viewGroup, i);
        } else {
            return new RecyclerView.ViewHolder(mFooterView) {
            };
        }
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder viewHolder, int i) {
        if (i == 0 || i == mBaseRvAdapter.getItemCount() + 1) {
            return;
        } else {
            mBaseRvAdapter.onBindViewHolder((BaseRvAdapter.MyViewHolder) viewHolder, i - 1);
        }
    }

    @Override
    public int getItemCount() {
        return mBaseRvAdapter.getItemCount() + 2;
    }

    @Override
    public int getItemViewType(int position) {
        if (position == 0) {
            return TYPE_HEADER;
        } else if (position == mBaseRvAdapter.getItemCount() + 1) {
            return TYPE_FOOTER;
        } else {
            return TYPE_NORMAL;
        }
    }
}
```

BaseRvAdapter 就是我们平常写的最基本的 Adapter，ItemView 都一样的时候。

关于复杂布局，其实也可以参考阿里开源的 V-Layout，不过它是内置了很多自定义 LayoutManager。

##### 上拉刷新、下拉加载

#### 高级玩法

##### LayoutManager

常见实现类（LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager），下面列出 LayoutManager 常用的 API：

```java
canScrollHorizontally();//能否横向滚动
canScrollVertically();//能否纵向滚动
scrollToPosition(int position);//滚动到指定位置

setOrientation(int orientation);//设置滚动的方向
getOrientation();//获取滚动方向

findViewByPosition(int position);//获取指定位置的Item View
findFirstCompletelyVisibleItemPosition();//获取第一个完全可见的Item位置
findFirstVisibleItemPosition();//获取第一个可见Item的位置
findLastCompletelyVisibleItemPosition();//获取最后一个完全可见的Item位置
findLastVisibleItemPosition();//获取最后一个可见Item的位置
```

自定义 LayoutManager：

##### ItemDecoration

RecyclerView 通过 addItemDecoration 方法添加 Item 之间的分割线，高版本 Android 已经提供默认实现，想改的话只需要 copy 一下这个类改一下 Drawable 即可。

```java
public class DividerItemDecoration extends ItemDecoration {
    public static final int HORIZONTAL = 0;
    public static final int VERTICAL = 1;
    private static final String TAG = "DividerItem";
    private static final int[] ATTRS = new int[]{16843284};
    private Drawable mDivider;
    private int mOrientation;
    private final Rect mBounds = new Rect();

    public DividerItemDecoration(Context context, int orientation) {
        TypedArray a = context.obtainStyledAttributes(ATTRS);
        this.mDivider = a.getDrawable(0);
        if (this.mDivider == null) {
            Log.w("DividerItem", "@android:attr/listDivider was not set in the theme used for this DividerItemDecoration. Please set that attribute all call setDrawable()");
        }

        a.recycle();
        this.setOrientation(orientation);
    }

    public void setOrientation(int orientation) {
        if (orientation != 0 && orientation != 1) {
            throw new IllegalArgumentException("Invalid orientation. It should be either HORIZONTAL or VERTICAL");
        } else {
            this.mOrientation = orientation;
        }
    }

    public void setDrawable(@NonNull Drawable drawable) {
        if (drawable == null) {
            throw new IllegalArgumentException("Drawable cannot be null.");
        } else {
            this.mDivider = drawable;
        }
    }

    public void onDraw(Canvas c, RecyclerView parent, State state) {
        if (parent.getLayoutManager() != null && this.mDivider != null) {
            if (this.mOrientation == 1) {
                this.drawVertical(c, parent);
            } else {
                this.drawHorizontal(c, parent);
            }

        }
    }

    private void drawVertical(Canvas canvas, RecyclerView parent) {
        canvas.save();
        int left;
        int right;
        if (parent.getClipToPadding()) {
            left = parent.getPaddingLeft();
            right = parent.getWidth() - parent.getPaddingRight();
            canvas.clipRect(left, parent.getPaddingTop(), right, parent.getHeight() - parent.getPaddingBottom());
        } else {
            left = 0;
            right = parent.getWidth();
        }

        int childCount = parent.getChildCount();

        for(int i = 0; i < childCount; ++i) {
            View child = parent.getChildAt(i);
            parent.getDecoratedBoundsWithMargins(child, this.mBounds);
            int bottom = this.mBounds.bottom + Math.round(child.getTranslationY());
            int top = bottom - this.mDivider.getIntrinsicHeight();
            this.mDivider.setBounds(left, top, right, bottom);
            this.mDivider.draw(canvas);
        }

        canvas.restore();
    }

    private void drawHorizontal(Canvas canvas, RecyclerView parent) {
        canvas.save();
        int top;
        int bottom;
        if (parent.getClipToPadding()) {
            top = parent.getPaddingTop();
            bottom = parent.getHeight() - parent.getPaddingBottom();
            canvas.clipRect(parent.getPaddingLeft(), top, parent.getWidth() - parent.getPaddingRight(), bottom);
        } else {
            top = 0;
            bottom = parent.getHeight();
        }

        int childCount = parent.getChildCount();

        for(int i = 0; i < childCount; ++i) {
            View child = parent.getChildAt(i);
            parent.getLayoutManager().getDecoratedBoundsWithMargins(child, this.mBounds);
            int right = this.mBounds.right + Math.round(child.getTranslationX());
            int left = right - this.mDivider.getIntrinsicWidth();
            this.mDivider.setBounds(left, top, right, bottom);
            this.mDivider.draw(canvas);
        }

        canvas.restore();
    }

    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
        if (this.mDivider == null) {
            outRect.set(0, 0, 0, 0);
        } else {
            if (this.mOrientation == 1) {
                outRect.set(0, 0, 0, this.mDivider.getIntrinsicHeight());
            } else {
                outRect.set(0, 0, this.mDivider.getIntrinsicWidth(), 0);
            }

        }
    }
}
```

[DividerItemDecoration](https://android.googlesource.com/platform/development/+/cc33d7e/samples/Support7Demos/src/com/example/android/supportv7/widget/decorator/DividerItemDecoration.java)

##### ItemAnimator

RecyclerView 通过 setItemAnimator 方法设置添加、删除、移动、改变的动画效果。

RecyclerView 提供了默认的 ItemAnimator 实现类：DefaultItemAnimator。

DefaultItemAnimator 是继承至 SimpleItemAnimator 类，而 SimpleItemAnimator 继承至 ItemAnimator。在自定义 ItemAnimator 时只需要继承 SimpleItemAnimator 即可，因为该类提供的 API 更加清晰易懂。

继承 SimpleItemAnimator 需要实现的方法有：

```java
    	//在 SimpleItemAnimator 中定义的抽象方法 
    	
    	//当 Item 移除时调用
    	public abstract boolean animateRemove(ViewHolder var1);
		//当 Item 添加时调用
    	public abstract boolean animateAdd(ViewHolder var1);
		//当 Item 移动时调用
    	public abstract boolean animateMove(ViewHolder var1, int var2, int var3, int var4, int var5);
		//当显式调用 notifyItemChanged() 或 notifyDataSetChanged() 时被调用
    	public abstract boolean animateChange(ViewHolder var1, ViewHolder var2, int var3, int var4, int var5, int var6);
    
    	//在 ItemAnimator 中定义的抽象方法
    	
    	//执行动画
    	//RecyclerView 动画的执行方式并不是立即执行，而是每帧执行一次，
    	//比如两帧之间添加了多个 Item，则会将这些要执行的动画 Pending 住，保存在成员变量中
    	//等到下一帧一起执行，该方法的执行前提是前面 animatorXxx 返回 true
        public abstract void runPendingAnimations();
		//当 Item 动画执行完毕时调用
        public abstract void endAnimation(@NonNull RecyclerView.ViewHolder var1);
		//当所有动画执行完毕时调用
        public abstract void endAnimations();
		//是否有动画在执行或者将要执行
        public abstract boolean isRunning();
```

##### ItemTouchHelper

##### 结合 SnapHelper

##### 万能 Adapter

#### 源码分析系列

##### DefaultItemAnimator

DefaultItemAnimator 类是 Android 提供的默认的动画类。首先看一下其成员变量：

```java
//...
private ArrayList<ViewHolder> mPendingRemovals = new ArrayList();
private ArrayList<ViewHolder> mPendingAdditions = new ArrayList();
ArrayList<ViewHolder> mAddAnimations = new ArrayList();
ArrayList<ViewHolder> mRemoveAnimations = new ArrayList();
```

这里只列举了移除和添加 Item 时所需的动画，可以看出动画的执行单位是 ViewHolder，并且可以分为两类，一类是保存将要执行的动画，一类是保存当前正在执行的动画。

然后在看 animateRemove 和 animateAdd 方法：

```java
    public boolean animateRemove(ViewHolder holder) {
        this.resetAnimation(holder);
        this.mPendingRemovals.add(holder);
        return true;
    }
    public boolean animateAdd(ViewHolder holder) {
        this.resetAnimation(holder);
        holder.itemView.setAlpha(0.0F);
        this.mPendingAdditions.add(holder);
        return true;
    }
```

这两个方法是在移除和添加 Item 的时候会调用的方法，就是用 mPengdingXxx 来保存即将要执行的动画，值得注意的是，添加 Add 动画的时候会先把该 Item 置为全透明，这和我们设置 DefaultItemAnimator 后所看的是一致的。

接着就是 runPendingAnimations 方法，这个方法前面说过，就是将要执行的动画集合。源码如下：

```java
    public void runPendingAnimations() {
        boolean removalsPending = !this.mPendingRemovals.isEmpty();
        boolean movesPending = !this.mPendingMoves.isEmpty();
        boolean changesPending = !this.mPendingChanges.isEmpty();
        boolean additionsPending = !this.mPendingAdditions.isEmpty();
        if (removalsPending || movesPending || additionsPending || changesPending) {
            //移除 Item 动画
            Iterator var5 = this.mPendingRemovals.iterator();
            while(var5.hasNext()) {
                ViewHolder holder = (ViewHolder)var5.next();
                this.animateRemoveImpl(holder);
            }

            this.mPendingRemovals.clear();
            final ArrayList additions;
            Runnable adder;
            //移动 Item 动画
            if (movesPending) {
                additions = new ArrayList();
                additions.addAll(this.mPendingMoves);
                this.mMovesList.add(additions);
                this.mPendingMoves.clear();
                adder = new Runnable() {
                    public void run() {
                        Iterator var1 = additions.iterator();

                        while(var1.hasNext()) {
                            DefaultItemAnimator.MoveInfo moveInfo = (DefaultItemAnimator.MoveInfo)var1.next();
                            DefaultItemAnimator.this.animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY, moveInfo.toX, moveInfo.toY);
                        }

                        additions.clear();
                        DefaultItemAnimator.this.mMovesList.remove(additions);
                    }
                };
                if (removalsPending) {
                    View view = ((DefaultItemAnimator.MoveInfo)additions.get(0)).holder.itemView;
                    ViewCompat.postOnAnimationDelayed(view, adder, this.getRemoveDuration());
                } else {
                    adder.run();
                }
            }

            if (changesPending) {
                additions = new ArrayList();
                additions.addAll(this.mPendingChanges);
                this.mChangesList.add(additions);
                this.mPendingChanges.clear();
                adder = new Runnable() {
                    public void run() {
                        Iterator var1 = additions.iterator();

                        while(var1.hasNext()) {
                            DefaultItemAnimator.ChangeInfo change = (DefaultItemAnimator.ChangeInfo)var1.next();
                            DefaultItemAnimator.this.animateChangeImpl(change);
                        }

                        additions.clear();
                        DefaultItemAnimator.this.mChangesList.remove(additions);
                    }
                };
                if (removalsPending) {
                    ViewHolder holder = ((DefaultItemAnimator.ChangeInfo)additions.get(0)).oldHolder;
                    ViewCompat.postOnAnimationDelayed(holder.itemView, adder, this.getRemoveDuration());
                } else {
                    adder.run();
                }
            }

            if (additionsPending) {
                additions = new ArrayList();
                additions.addAll(this.mPendingAdditions);
                this.mAdditionsList.add(additions);
                this.mPendingAdditions.clear();
                adder = new Runnable() {
                    public void run() {
                        Iterator var1 = additions.iterator();

                        while(var1.hasNext()) {
                            ViewHolder holder = (ViewHolder)var1.next();
                            DefaultItemAnimator.this.animateAddImpl(holder);
                        }

                        additions.clear();
                        DefaultItemAnimator.this.mAdditionsList.remove(additions);
                    }
                };
                if (!removalsPending && !movesPending && !changesPending) {
                    adder.run();
                } else {
                    long removeDuration = removalsPending ? this.getRemoveDuration() : 0L;
                    long moveDuration = movesPending ? this.getMoveDuration() : 0L;
                    long changeDuration = changesPending ? this.getChangeDuration() : 0L;
                    long totalDelay = removeDuration + Math.max(moveDuration, changeDuration);
                    View view = ((ViewHolder)additions.get(0)).itemView;
                    ViewCompat.postOnAnimationDelayed(view, adder, totalDelay);
                }
            }

        }
    }
```

可以看出 Remove 动画的是最先执行的，然后依次是 Move 动画、Change 动画、Add 动画。

然后通过 Iterator 迭代器遍历 List 执行动画，显然真正执行动画的逻辑肯定就在 animateXxxImpl 方法里面了。

但是细心的你肯定会发现，为什么除了 Remove 动画，其他都需要在 Runnable 里构造执行体呢？

从 Runnable 的 run 方法的执行条件可以看出，如果有 Remove 动画，那么其他动画必须延时执行，如果没有，那就立即执行。

然后就可以看看动画真正的实现了，这里就拿 Add 动画为例，即 animateAddImpl：

```java
    void animateAddImpl(final ViewHolder holder) {
        final View view = holder.itemView;
        final ViewPropertyAnimator animation = view.animate();
        this.mAddAnimations.add(holder);
        animation.alpha(1.0F).setDuration(this.getAddDuration()).setListener(new AnimatorListenerAdapter() {
            public void onAnimationStart(Animator animator) {
                DefaultItemAnimator.this.dispatchAddStarting(holder);
            }

            public void onAnimationCancel(Animator animator) {
                view.setAlpha(1.0F);
            }

            public void onAnimationEnd(Animator animator) {
                animation.setListener((AnimatorListener)null);
                DefaultItemAnimator.this.dispatchAddFinished(holder);
                DefaultItemAnimator.this.mAddAnimations.remove(holder);
                DefaultItemAnimator.this.dispatchFinishedWhenDone();
            }
        }).start();
    }
```

可以看到，动画真正的实现是通过 [ViewPropertyAnimator](https://developer.android.com/reference/android/view/ViewPropertyAnimator?hl=zh-cn) 来实现的，前面说到，在 Add 动画的时候会把该 ViewHolder 设置为全透明，所以这是一个透明度渐变的过程，getAddDuration 是写死的 120 毫秒，而且动画是不可逆的。

那我们再想想为什么不用 ObjectAnimator 来实现呢？

按照官方的说法，此类可以为多个同时动画提供更好的性能，而且在多个动画同时执行，并不会导致某个动画属性执行失效，并且使用更加简单。

此类不是由调用者构造的，而是通过 View.animate() 返回的对应 View 的 ViewPropertyAnimator 对象的引用。这给我们使用 AnimatorSet 又带来了一种新选择，get 。

上面都是使用 mPengdingXxx，那 mAxxAnimations 有什么

##### 总结：

1. 
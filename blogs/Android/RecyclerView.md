---
RecyclerView
---

#### 目录

1. 思维导图

2. 基本使用

   - 复杂布局的实现、添加头布局、尾布局
   - 下拉刷新、上拉加载

3. 高级玩法

   - LayoutManager

   - ItemDecoration
   - ItemAnimator
   - ItemTouchHelper
   - 结合 SnapHelper
   - 万能 Adapter

4. 源码分析系列

   - DefaultItemAnimator
   - LinearSnapHelper
   - 缓存机制
     - ListView 的 RecycleBin
     - RecyclerView 的 Recycler
     - 两者区别
   - 局部刷新

5. 其他

   - 扩展 RecyclerView

   - 嵌套滑动
   - 与 ListView 对比
     - RecyclerView 优缺点
     - ListView 优缺点

6. 参考

#### 思维导图

![](https://github.com/Omooo/Android-Notes/blob/master/images/Android/RecyclerView.png?raw=true)

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

##### 下拉刷新、上拉加载

上拉刷新直接用 SwipeRefreshLayout 就行了。

下拉加载即：

```java
        mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(@NonNull RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                LinearLayoutManager manager = (LinearLayoutManager) recyclerView.getLayoutManager();
                // 当不滑动时
                if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                    //获取最后一个完全显示的itemPosition
                    int lastItemPosition = manager.findLastCompletelyVisibleItemPosition();
                    int itemCount = manager.getItemCount();

                    // 判断是否滑动到了最后一个item，并且是向上滑动
                    if (lastItemPosition == (itemCount - 1) && isSlidingUpward) {
                        //加载更多
                        onLoadMore();
                    }
                }
            }

            @Override
            public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                // 大于0表示正在向上滑动，小于等于0表示停止或向下滑动
                isSlidingUpward = dy > 0;
            }
        });
```



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

不过，实际上，最简单的做法就是在每个 ItemView 的底部加一个 Divider，然后在 Adapter 里判断是最后一个 ItemView 的时候隐藏该 Divider 即可。

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

但是即使这样，需要实现的方法还是一大堆，并且还的自己去写动画的添加、回收等等。所以我建议直接看 [**recyclerview-animators**](https://github.com/wasabeef/recyclerview-animators) 中的 BaseItemAnimator，然后继承它只需要重写动画逻辑就好了，不必在考虑繁琐的动画添加、回收等工作。

##### ItemTouchHelper

ItemTouchHelper 能用来实现 RecyclerView Item 的上下拖拽以及滑动删除功能，实现起来可以说是究极简单。

分为两步：

1. 实现 ItemTouchHelper.Callback 回调
2. 把 ItemTouchHelper 绑定到 RecyclerView 上

第一步，多说无益：

```java
public class DefaultItemTouchHelper<T> extends ItemTouchHelper.Callback {

    private BaseRvAdapterWrapper mBaseRvAdapterWrapper;
    private List<T> mList;

    public DefaultItemTouchHelper(BaseRvAdapterWrapper baseRvAdapterWrapper, List<T> list) {
        mBaseRvAdapterWrapper = baseRvAdapterWrapper;
        mList = list;
    }

    /**
     * 设置支持滑动、拖拽的方向
     */
    @Override
    public int getMovementFlags(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder) {
        int dragFlag = ItemTouchHelper.UP | ItemTouchHelper.DOWN;   //上下拖拽
        int swipeFlag = ItemTouchHelper.START | ItemTouchHelper.END;    //左右滑动
        return makeMovementFlags(dragFlag, swipeFlag);
    }

    /**
     * 拖拽时回调
     */
    @Override
    public boolean onMove(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, @NonNull RecyclerView.ViewHolder viewHolder1) {
        int form = viewHolder.getAdapterPosition();
        int to = viewHolder1.getAdapterPosition();
        Collections.swap(mList, form, to);
        mBaseRvAdapterWrapper.notifyItemMoved(form, to);
        return true;
    }

    /**
     * 滑动时回调
     */
    @Override
    public void onSwiped(@NonNull RecyclerView.ViewHolder viewHolder, int i) {
        int position = viewHolder.getAdapterPosition();
        mList.remove(position);
        mBaseRvAdapterWrapper.notifyItemRemoved(position);
    }

    /**
     * 状态改变时回调
     */
    @Override
    public void onSelectedChanged(@Nullable RecyclerView.ViewHolder viewHolder, int actionState) {
        super.onSelectedChanged(viewHolder, actionState);
        if (actionState != ItemTouchHelper.ACTION_STATE_IDLE) {
            viewHolder.itemView.setBackgroundColor(Color.parseColor("#ffffff")); //设置拖拽和侧滑时的背景色
        }
    }

    /**
     * 拖拽或滑动完成之后回调
     */
    @Override
    public void clearView(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder) {
        super.clearView(recyclerView, viewHolder);
        viewHolder.itemView.setBackgroundColor(Color.parseColor("#FFFFFF"));
    }
    
    /**
     * 如果想自定义动画，可以重写这个方法
     * 根据偏移量来设置
     */
    @Override
    public void onChildDraw(@NonNull Canvas c, @NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive) {
        super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive);
    }
}
```

第二步：

```java
ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new DefaultItemTouchHelper<>(mBaseRvAdapterWrapper, mData));
itemTouchHelper.attachToRecyclerView(mRecyclerView);
```

OK，完事。

##### 结合 SnapHelper

SnapHelp 能够辅助 RecyclerView 在滚动结束时将 Item 对齐到某个位置。

SnapHelp 是一个抽象类，Android 提供了 LinearSnapHelper，可以让 RecyclerView 滚动停止时 Item 停留在中间位置，又提供了 PagerSnapHelper，可以让 RecyclerView 像 ViewPager 一样的效果，一次只能滑动一个，并且 Item 居中显示，和 LinearSnapHelper 的区别在于 LinearSnapHelper 支持惯性滑动，所以一次能滑动多个。

使用起来极其简单：

```java
LinearSnapHelper linearSnapHelper = new LinearSnapHelper();
linearSnapHelper.attachToRecyclerView(mRecyclerView);

PagerSnapHelper pagerSnapHelper = new PagerSnapHelper();
pagerSnapHelper.attachToRecyclerView(mRecyclerView);
```

在分析 SnapHelper 原理之前，先来了解一下 Fling 概念。Fling 操作从手指离开屏幕瞬间被触发，在停止滚动时结束，即惯性滑动。

SnapHelpr 是一个抽象类，它有三个抽象方法：

```java
    @Override
    public int findTargetSnapPosition(RecyclerView.LayoutManager layoutManager, int i, int i1) {
        return 0;
    }

    @Nullable
    @Override
    public View findSnapView(RecyclerView.LayoutManager layoutManager) {
        return null;
    }

    @Nullable
    @Override
    public int[] calculateDistanceToFinalSnap(@NonNull RecyclerView.LayoutManager layoutManager, @NonNull View view) {
        return new int[0];
    }
```

findTargetSnapPosition 方法会根据触发 Fling 操作的速率来找到 RecyclerView 需要滚动到哪个位置，该位置对应的 ItemView 就是那个需要对齐的列表项。我们把这个位置称为 targetSnapPosition，对应的 View 称为 targetSnapView，如果找不到 targetSnapPosition，就返回 RecyclerView.NO_POSITION。

findSnapView 方法会找到当前 LayoutManager 最接近对齐位置的那个 ItemView，该 View 称为 SnapView，对应的 position 称为 SnapPosition。如果返回 null，则表示没有要对齐的 View，也就不会做对齐调整。

calculateDistanceToFinalSnap 方法会计算需要对齐的 ItemView 与目标 View 之间的距离，返回一个大小为二的 int 数组，分别对应 x 轴和 y 轴方向上的距离。

![](https://github.com/Omooo/Android-Notes/blob/master/images/SnapView.png?raw=true)



##### 万能 Adapter

```java
public abstract class QuickAdapter<T> extends RecyclerView.Adapter<QuickAdapter.VH> {

    private List<T> mDatas;

    public QuickAdapter(List<T> datas) {
        mDatas = datas;
    }

    public abstract int getLayoutId(int viewType);

    public abstract void convert(VH viewHolder, T data, int position);

    @NonNull
    @Override
    public VH onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
        return VH.get(viewGroup, getLayoutId(i));
    }

    @Override
    public void onBindViewHolder(@NonNull VH vh, int i) {
        convert(vh, mDatas.get(i), i);
    }

    @Override
    public int getItemCount() {
        return mDatas.size();
    }

    static class VH extends RecyclerView.ViewHolder {

        private SparseArray<View> mViews;
        private View mConvertView;

        public VH(@NonNull View itemView) {
            super(itemView);
            mConvertView = itemView;
            mViews = new SparseArray<>();
        }

        public static VH get(ViewGroup parent, int layoutId) {
            View convertView = LayoutInflater.from(parent.getContext()).inflate(layoutId, parent, false);
            return new VH(convertView);
        }

        public <T extends View> T getView(int id) {
            View view = mViews.get(id);
            if (view == null) {
                view = mConvertView.findViewById(id);
                mViews.put(id, view);
            }
            return (T) view;
        }

        public void setText(int id, String value) {
            TextView view = getView(id);
            view.setText(value);
        }
    }
}
```



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

上面都是使用 mPengdingXxx，那 mXxxAnimations 有什么用呢？我们只看到它在动画执行前添加一个 ViewHolder，动画执行完毕之后又移除这个 ViewHoler。

其实它的作用很简单，就是判断动画是否正在在执行：

```java
    public boolean isRunning() {
        return !this.mPendingAdditions.isEmpty() || !this.mPendingChanges.isEmpty() || !this.mPendingMoves.isEmpty() || !this.mPendingRemovals.isEmpty() || !this.mMoveAnimations.isEmpty() || !this.mRemoveAnimations.isEmpty() || !this.mAddAnimations.isEmpty() || !this.mChangeAnimations.isEmpty() || !this.mMovesList.isEmpty() || !this.mAdditionsList.isEmpty() || !this.mChangesList.isEmpty();
    }
```

总结：

1. 动画的执行是有优先级之分的，Remove > Move > Change > Add
2. 我们重写的 animateXxx 等方法只是把将要执行的动画添加到执行队列（ List ）中，然后在 runPendingAnimations 一并执行
3. 真正的动画效果是通过 ViewPropertyAnimator 来实现的，它在多个动画同时执行时表现出更好的性能

##### LinearSnapHelper

首先看 LinearSnapHelper#attachToRecyclerView 方法：

```java
    public void attachToRecyclerView(@Nullable RecyclerView recyclerView) throws IllegalStateException {
        //...
        this.setupCallbacks();
        this.mGravityScroller = new Scroller(this.mRecyclerView.getContext(), new DecelerateInterpolator());
        this.snapToTargetExistingView();
    }
```

创建一个 Scroller 对象用于辅助计算 Fling 的总距离，然后调用 snapToTargetExistingView 实现对 SnapView 的滚动对齐。

LinearSnapHelper#snapToTargetExistingView 方法：

```java
    void snapToTargetExistingView() {
        if (this.mRecyclerView != null) {
            LayoutManager layoutManager = this.mRecyclerView.getLayoutManager();
            if (layoutManager != null) {
                View snapView = this.findSnapView(layoutManager);
                if (snapView != null) {
                    int[] snapDistance = this.calculateDistanceToFinalSnap(layoutManager, snapView);
                    if (snapDistance[0] != 0 || snapDistance[1] != 0) {
                        this.mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
                    }

                }
            }
        }
    }
```

看到这大致流程就清楚了，首先是先找出 SnapView，然后计算出 SnapView 需要滚动的距离，最后通过 RecyclerView#smoothScrollBy 平滑滚动到指定位置就完事了。

那它内部是如何计算距离的呢？

```java
    public int[] calculateDistanceToFinalSnap(@NonNull LayoutManager layoutManager, @NonNull View targetView) {
        int[] out = new int[2];
        if (layoutManager.canScrollHorizontally()) {
            out[0] = this.distanceToCenter(layoutManager, targetView, this.getHorizontalHelper(layoutManager));
        } else {
            out[0] = 0;
        }

        if (layoutManager.canScrollVertically()) {
            out[1] = this.distanceToCenter(layoutManager, targetView, this.getVerticalHelper(layoutManager));
        } else {
            out[1] = 0;
        }

        return out;
    }
```

和我们前面说的一致，该方法会返回一个大小为二的整形数组，分别用来表示需要滚动的 X 距离和 Y 距离。而且对于 LinearSnapHelper 来说，它最终是以 RecyclerView 的中心对齐的，无疑，LinearSnapHelper#distanceToCenter 方法就是返回中心对齐的距离：

```java
    private int distanceToCenter(@NonNull LayoutManager layoutManager, @NonNull View targetView, OrientationHelper helper) {
        int childCenter = helper.getDecoratedStart(targetView) + helper.getDecoratedMeasurement(targetView) / 2;
        int containerCenter;
        if (layoutManager.getClipToPadding()) {
            containerCenter = helper.getStartAfterPadding() + helper.getTotalSpace() / 2;
        } else {
            containerCenter = helper.getEnd() / 2;
        }

        return childCenter - containerCenter;
    }
```

看到这才发现距离都是通过 OrinetationHelper 来计算的，那就看一下方法的实参，即 this.getHorizontalHelper(layoutManager) 方法：

```java
    @NonNull
    private OrientationHelper getVerticalHelper(@NonNull LayoutManager layoutManager) {
        if (this.mVerticalHelper == null || this.mVerticalHelper.mLayoutManager != layoutManager) {
            this.mVerticalHelper = OrientationHelper.createVerticalHelper(layoutManager);
        }

        return this.mVerticalHelper;
    }
```

再往下看 OrientationHelper.createVerticalHelper(layoutManager) 方法：

```java
    public static OrientationHelper createHorizontalHelper(LayoutManager layoutManager) {
        return new OrientationHelper(layoutManager) {
    
            //...
            public int getEnd() {
                return this.mLayoutManager.getWidth();
            }

            public int getStartAfterPadding() {
                return this.mLayoutManager.getPaddingLeft();
            }

            public int getDecoratedMeasurement(View view) {
                LayoutParams params = (LayoutParams)view.getLayoutParams();
                return this.mLayoutManager.getDecoratedMeasuredWidth(view) + params.leftMargin + params.rightMargin;
            }

            public int getDecoratedStart(View view) {
                LayoutParams params = (LayoutParams)view.getLayoutParams();
                return this.mLayoutManager.getDecoratedLeft(view) - params.leftMargin;
            }

            public int getTotalSpace() {
                return this.mLayoutManager.getWidth() - this.mLayoutManager.getPaddingLeft() - this.mLayoutManager.getPaddingRight();
            }
        };
    }
```

看到这再回到 LinearSnapHelper#distanceToCenter 方法中去，它所做的事就是通过 LayoutManager 分别获取 SnapView 的中心坐标以及 RecyclerView 的中心坐标，两者之差就是要滑动的距离。

总结：

1. 首先通过 findTargetSnapPosition 方法获取到需要对齐的 ItemView
2. 然后通过 distanceToCenter 方法来计算 ItemView 距离中心的距离，计算距离是通过 LayoutManager 来计算该 ItemView 中心点距离 RecyclerView 中心点的距离
3. 通过 RecyclerView#smoothScrollBy 滚动该距离

##### 缓存机制

RecyclerView 比 ListView 多两级缓存，支持多个离 ItemView 缓存，支持开发者自定义缓存处理逻辑。支持所有的 RecyclerView 共用同一个 RecyclerViewPool。

ListView 为了保证 ItemView 的复用，实现了一套回收机制，该回收机制的实现类是 RecyclerBin，它实现了两级缓存：

1. View[] mActiveViews 

   缓存屏幕上的 View，在该缓存里的 View 不需要调用 getView()。

2. ArrayList\<View>[] mScrapViews

   每个 ItemType 对应一个列表作为回收站，缓存由于滚动而消失的 View，此处的 View 如果被复用，会以参数的形式传给 getView()。

RecyclerView 是以 ViewHolder 单位来进行回收，Recycler 是 RecyclerView 回收机制的实现类，它实现了四级缓存：

1. mAttachedScrap

   缓存在屏幕上的 ViewHolder。

2. mCachedViews

   缓存屏幕外的 ViewHolder，默认为两个。ListView 对于屏幕外的缓存都会调用 getView。

3. mViewCacheExtensions

   需要用户定制，默认不实现。

4. mRecyclerPool

   缓存池，多个 RecyclerView 共用。

RecyclerView#getViewForPosition 方法：

```java
        View getViewForPosition(int position, boolean dryRun) {
            return this.tryGetViewHolderForPositionByDeadline(position, dryRun, 9223372036854775807L).itemView;
        }

        @Nullable
        RecyclerView.ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
            if (position >= 0 && position < RecyclerView.this.mState.getItemCount()) {
                //...
                if (holder == null) {
                    //...
                    type = RecyclerView.this.mAdapter.getItemViewType(offsetPosition);
                    if (RecyclerView.this.mAdapter.hasStableIds()) {
                        holder = this.getScrapOrCachedViewForId(RecyclerView.this.mAdapter.getItemId(offsetPosition), type, dryRun);
                        //...
                    }

                    if (holder == null && this.mViewCacheExtension != null) {
                        View view = this.mViewCacheExtension.getViewForPositionAndType(this, position, type);
                        //...
                    }

                    if (holder == null) {
                        holder = this.getRecycledViewPool().getRecycledView(type);
                        //...
                    }

                    if (holder == null) {
						//...
                        holder = RecyclerView.this.mAdapter.createViewHolder(RecyclerView.this, type);
                       //...
                    }
                }
                }
                return holder;
            } else {
                throw new IndexOutOfBoundsException("Invalid item position " + position + "(" + position + "). Item count:" + RecyclerView.this.mState.getItemCount() + RecyclerView.this.exceptionLabel());
            }
        }
```

从上诉实现可以看出，依次从 mAttachedScrap、mCachedView、mViewCacheExtension、mRecyclerPool 寻找可复用的 ViewHolder，如果是从 mAttachedScrap 或 mCachedViews 中获取的 ViewHolder，则不会调用 onBindViewHolder，而如果从 mViewCacheExtension 或 mRecyclePool 中获取的 ViewHolder，则会调用 onBindViewHolder。如果上述没拿到缓存的 ViewHolder，则会通过 createViewHolder 来创建。

具体来说：

1.层级不同

ListView 两级缓存：

|              | 是否需要回调 createView | 是否需要回调 bindView | 生命周期                                                   | 备注                           |
| ------------ | ----------------------- | --------------------- | ---------------------------------------------------------- | ------------------------------ |
| mActiveViews | 否                      | 否                    | onLayout 函数周期内                                        | 用于屏幕内 ItemView 的快速重用 |
| mScrapViews  | 否                      | 是                    | 与 mAdapter 一致，当 mAdapter 更改时，mScrapViews 即被清空 |                                |

RecyclerView 四级缓存：

|                     | 是否需要 createView | 是否需要 bindView | 生命周期                                                     | 备注                                                         |
| ------------------- | ------------------- | ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| mAttachedScrap      | 否                  | 否                | onLayout 函数周期内                                          | 用于屏幕内 ItemView 的快速重用                               |
| mCacheViews         | 否                  | 否                | 与 mAdapter 一致，当 mAdapter 被更改时，mCacheViews 即被缓存至 mRecyclerPool | 默认上限为二，即缓存屏幕外两个 ItemView                      |
| mViewCacheExtension |                     |                   |                                                              | 不直接使用，需要用户定制，默认不实现                         |
| mRecyclerPool       | 否                  | 是                | 与自身生命周期一致，不再被引用时即被释放                     | 默认上限为五，但是技术上可以实现所有 RecyclerViewPool共用同一个 |

2.缓存对象不同

RecyclerView 缓存的对象为 ViewHolder，ListView 缓存 View。

##### 局部刷新

RecyclerView 已经提供了类似 notifyItemRemoved 等等局部刷新的 API，那么 ListView 如何实现局部刷新呢？

```java
    private void updateItemView(ListView listView, int position, List<String> data) {
        int firstPos = listView.getFirstVisiblePosition();
        int lastPos = listView.getLastVisiblePosition();
        if (position >= firstPos && position <= lastPos) {
            //可见时才更新，不可见时则在 getView 时更新
            View view = listView.getChildAt(position);
            VH vh = view.getTag();
            vh.textView.setText(data.get(position));
        }
    }
```



#### 其他

##### 扩展 RecyclerView

setEmptyView()

##### 嵌套滑动

为了支持嵌套滑动，子 View 必须实现 NestedScrollingChild 接口，父 View 必须实现 NestedScrollingParent 接口，而 RecyclerView 实现了 NestedScrollingChild 接口。

##### 与 ListView 对比

RecyclerView 相比 ListView，有一些明显的优点：

1. 默认已经实现了 View 的复用，而且复用机制更加完善
2. 支持局部刷新
3. Item 添加、删除动画，Item 实现拖拽、侧滑删除等
4. 更灵活的 LayoutManager

当然，ListView 相比 RecyclerView 也有优点：

1. addHeaderView、addFooterView 添加头视图、尾视图
2. 通过 android:divider 设置自定义分割线
3. setOnItemClickListener、setOnItemLongClickListener 设置点击事件以及长按事件



#### 参考

[让你明明白白的使用RecyclerView——SnapHelper详解](https://www.jianshu.com/p/e54db232df62)

[Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)

[RecyclerView 必知必会](https://mp.weixin.qq.com/s/CzrKotyupXbYY6EY2HP_dA?)

[RecyclerView 文章集](https://github.com/CymChad/CymChad.github.io)
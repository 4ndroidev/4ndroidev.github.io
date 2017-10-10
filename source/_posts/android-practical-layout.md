title: Android 实用布局
date: 2017-10-09 20:30:54
tags:
  - Android
  - Layout
---

## 目录

|英文名|中文名|
|---|---|
|FlowLayout|流式布局|
|NineGridLayout|九宫格布局|
|BoundLayout|回弹布局|
|RefreshLayout|下拉刷新布局|

## 贩剑

Q: Github 一堆相应的UI组件库，为什么要重复造轮子?

A: 别人的劳斯莱斯轮子未必适合我这破单车。另外，做自己力所能及的事情，多练习也有好处。


## 基础

`ViewGroup`作为容器类，基本上布局都继承该类或其子类，重写`onMeasure`和`onLayout`方法进行自定义布局。其中官方早已实现五大布局：`FrameLayout`，`LinearLayout`，`RelativeLayout`，`TableLayout`和`AbsoluteLayout`。随后，`support`库也出了不少优秀布局，如`ConstraintLayout`，`CoordinatorLayout`等。以上都是官方叼炸天的布局，下面说说作为一个平民，我能做到的布局，由易到难。

<!-- more -->

## FlowLayout

描述：相对简单，只需要关注满行后换行操作。`onMeasure`和`onLayout`都是一个`for`循环的操作。另外使用`ListAdapter`实现子视图适配相对好，不局限于仅适合文本视图，其次在`RecyclerView`中使用时，`ViewHolder`的重用也可以达到子视图重用。由于比较简单，直接贴代码。

```java
public class FlowLayout extends ViewGroup {

    private final static int DEFAULT_VERTICAL_SPACE = 10;
    private final static int DEFAULT_HORIZONTAL_SPACE = 6;

    private ListAdapter mAdapter;
    private DataSetObserver mObserver = new DataSetObserver() {
        @Override
        public void onChanged() {
            startUpdate();
        }
    };
    private int mVerticalSpace;
    private int mHorizontalSpace;
    private int mRowHeight;
    private OnItemClickListener mOnItemClickListener;

    public FlowLayout(Context context) {
        this(context, null);
    }

    public FlowLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public FlowLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        if (attrs != null) {
            TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.FlowLayout);
            mVerticalSpace = array.getDimensionPixelOffset(R.styleable.FlowLayout_verticalSpace, DEFAULT_VERTICAL_SPACE);
            mHorizontalSpace = array.getDimensionPixelOffset(R.styleable.FlowLayout_horizontalSpace, DEFAULT_HORIZONTAL_SPACE);
            array.recycle();
        }
    }

    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null) {
            mAdapter.unregisterDataSetObserver(mObserver);
        }
        mAdapter = adapter;
        if (mAdapter != null) {
            mAdapter.registerDataSetObserver(mObserver);
        }
        startUpdate();
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int measureWidth = getDefaultSize(0, widthMeasureSpec);
        int paddingHorizontal = getPaddingLeft() + getPaddingRight();
        int paddingVertical = getPaddingTop() + getPaddingBottom();
        int actualWidth = measureWidth - paddingHorizontal;
        int count = getChildCount();
        int freeWidth = actualWidth;
        int row = 0;
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
            mRowHeight = Math.max(mRowHeight, child.getMeasuredHeight());
            int childWidth = child.getMeasuredWidth();
            int space = freeWidth == actualWidth ? 0 : mHorizontalSpace;
            if (childWidth + space > freeWidth) {
                freeWidth = actualWidth - childWidth;
                row++;
            } else if (childWidth + space == freeWidth) {
                freeWidth = actualWidth;
                row++;
            } else {
                freeWidth -= childWidth + space;
            }
        }
        if (freeWidth < actualWidth) row++;
        setMeasuredDimension(measureWidth, paddingVertical + row * mRowHeight + Math.max(0, row - 1) * mVerticalSpace);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int paddingLeft = getPaddingLeft();
        int paddingRight = getPaddingRight();
        int paddingTop = getPaddingTop();
        int row = 0;
        int actualWidth = getMeasuredWidth() - paddingLeft - paddingRight;
        int freeWidth = actualWidth;
        for (int i = 0, count = getChildCount(); i < count; i++) {
            View child = getChildAt(i);
            int childWidth = child.getMeasuredWidth();
            int childHeight = child.getMeasuredHeight();
            int space = freeWidth == actualWidth ? 0 : mHorizontalSpace;
            int left, top;
            if (childWidth + space > freeWidth) {
                freeWidth = actualWidth - childWidth;
                row++;
                left = paddingLeft;
                top = paddingTop + row * mRowHeight + row * mVerticalSpace;
            } else if (childWidth + space == freeWidth) {
                left = actualWidth - freeWidth + space;
                top = paddingTop + row * mRowHeight + row * mVerticalSpace;
                freeWidth = actualWidth;
                row++;
            } else {
                left = actualWidth - freeWidth + space;
                top = paddingTop + row * mRowHeight + row * mVerticalSpace;
                freeWidth -= childWidth + space;
            }
            child.layout(left, top, left + childWidth, top + childHeight);
        }
    }

    @Override
    public void onViewRemoved(View child) {
        super.onViewRemoved(child);
        child.setOnClickListener(null);
    }

    @Override
    public void onViewAdded(View child) {
        super.onViewAdded(child);
        child.setOnClickListener(v -> {
            if (mOnItemClickListener != null) {
                int position = indexOfChild(child);
                mOnItemClickListener.onItemClick(this, child, position, mAdapter.getItemId(position));
            }
        });
    }

    public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
        mOnItemClickListener = onItemClickListener;
    }

    private void startUpdate() {
        if (mAdapter == null) {
            removeAllViews();
            return;
        }
        int expectCount = mAdapter.getCount();
        int childCount = getChildCount();
        if (childCount > expectCount) {
            for (int i = childCount - 1; i >= expectCount; i--) {
                removeViewAt(i);
            }
            for (int i = 0; i < expectCount; i++) {
                mAdapter.getView(i, getChildAt(i), this);
            }
        } else {
            for (int i = 0; i < childCount; i++) {
                mAdapter.getView(i, getChildAt(i), this);
            }
            for (int i = childCount; i < expectCount; i++) {
                addView(mAdapter.getView(i, null, this));
            }
        }
        requestLayout();
    }

    public interface OnItemClickListener {
        void onItemClick(FlowLayout parent, View view, int position, long id);
    }

}
```

## NineGridLayout

描述：相对简单，根据容器大小，分成三栏，额外根据子视图数目，特殊地，当子视图数目为一时，视图占两行两列；当子视图数目为四时，分布为两行两列；其余按照三个一排即可。同样适合使用`ListAdapter`作适配，上代码。

```java
public class NineGridLayout extends ViewGroup {

    private final static int DEFAULT_DIVIDER_WIDTH = 6;

    private int mDividerWidth;
    private ListAdapter mAdapter;
    private DataSetObserver mObserver = new DataSetObserver() {
        @Override
        public void onChanged() {
            startUpdate();
        }
    };
    private OnItemClickListener mOnItemClickListener;

    public NineGridLayout(Context context) {
        this(context, null);
    }

    public NineGridLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public NineGridLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        if (attrs != null) {
            TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.NineGridLayout);
            mDividerWidth = array.getDimensionPixelOffset(R.styleable.NineGridLayout_dividerWidth, DEFAULT_DIVIDER_WIDTH);
            array.recycle();
        }
    }

    /**
     * 根据父组件大小，决定子组件大小
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int width = getDefaultSize(0, widthMeasureSpec);
        int itemSize = (int) ((width - getPaddingLeft() - getPaddingRight() - 2 * mDividerWidth) / 3.0f);
        if (itemSize < 0)
            throw new IllegalStateException("measureWith must more than the sum of padding and dividerWidth!");
        int height = getPaddingTop() + getPaddingBottom();
        int count = getChildCount();
        if (count == 1) {
            height += 2 * itemSize + mDividerWidth;
        } else if (count <= 3) {
            height += itemSize;
        } else if (count <= 6) {
            height += 2 * itemSize + mDividerWidth;
        } else {
            height += 3 * itemSize + 2 * mDividerWidth;
        }
        setMeasuredDimension(width, height);
        if (count == 1) {
            int singleSize = 2 * itemSize + mDividerWidth;
            int childWithMeasureSpec = MeasureSpec.makeMeasureSpec(singleSize, MeasureSpec.EXACTLY);
            int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(singleSize, MeasureSpec.EXACTLY);
            getChildAt(0).measure(childWithMeasureSpec, childHeightMeasureSpec);
        } else {
            int childWithMeasureSpec = MeasureSpec.makeMeasureSpec(itemSize, MeasureSpec.EXACTLY);
            int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(itemSize, MeasureSpec.EXACTLY);
            for (int i = 0; i < count; i++) {
                getChildAt(i).measure(childWithMeasureSpec, childHeightMeasureSpec);
            }
        }
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int count = getChildCount();
        int columns;
        if (count <= 3) {
            columns = count;
        } else if (count == 4) {
            columns = 2;
        } else {
            columns = 3;
        }
        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingRight = getPaddingRight();
        int itemSize = (int) ((getMeasuredWidth() - paddingLeft - paddingRight - 2 * mDividerWidth) / 3.0f);
        if (count == 1) {
            int singleSize = 2 * itemSize + mDividerWidth;
            getChildAt(0).layout(paddingLeft, paddingTop, paddingLeft + singleSize, paddingTop + singleSize);
        } else {
            for (int i = 0; i < count; i++) {
                int row = i / columns;
                int column = i % columns;
                int left = paddingLeft + column * (itemSize + mDividerWidth);
                int top = paddingTop + row * (itemSize + mDividerWidth);
                getChildAt(i).layout(left, top, left + itemSize, top + itemSize);
            }
        }
    }

    @Override
    public void onViewRemoved(View child) {
        super.onViewRemoved(child);
        child.setOnClickListener(null);
    }

    @Override
    public void onViewAdded(View child) {
        super.onViewAdded(child);
        child.setOnClickListener(v -> {
            if (mOnItemClickListener != null) {
                int position = indexOfChild(child);
                mOnItemClickListener.onItemClick(this, child, position, mAdapter.getItemId(position));
            }
        });
    }

    public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
        mOnItemClickListener = onItemClickListener;
    }

    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null) {
            mAdapter.unregisterDataSetObserver(mObserver);
        }
        mAdapter = adapter;
        if (mAdapter != null) {
            mAdapter.registerDataSetObserver(mObserver);
        }
        startUpdate();
    }

    private void startUpdate() {
        if (mAdapter == null) {
            removeAllViews();
            return;
        }
        int expectCount = mAdapter.getCount();
        if (expectCount > 9) {
            throw new IllegalStateException("NineGridView can host at most 9 children!");
        }
        int childCount = getChildCount();
        if (childCount > expectCount) {
            for (int i = childCount - 1; i >= expectCount; i--) {
                removeViewAt(i);
            }
            for (int i = 0; i < expectCount; i++) {
                mAdapter.getView(i, getChildAt(i), this);
            }
        } else {
            for (int i = 0; i < childCount; i++) {
                mAdapter.getView(i, getChildAt(i), this);
            }
            for (int i = childCount; i < expectCount; i++) {
                addView(mAdapter.getView(i, null, this));
            }
        }
        requestLayout();
    }

    public interface OnItemClickListener {
        void onItemClick(NineGridLayout parent, View view, int position, long id);
    }

}
```

## BoundLayout

描述：实现目前`Bilibili`番剧时间表的回弹布局，这种交互用作提示，感觉很友好。并可耻盗用了`Bilibili`两张图做`Demo`。相对复杂，上述两简单组件都没涉及手势，该组件涉及到手势处理。大致实现，就是当子视图不能横向或纵向滚动时，拦截手势，使用`ViewCompat.offsetLeftAndRight`或`ViewCompat.offsetTopAndBottom`进行横向或纵向偏移视图。代码有500来行，不适合贴全代码，仅上个较通用的手势代码，如果想看全代码，请到`github`

github: [BoundLayout](https://github.com/4ndroidev/BoundLayout)

![boundlayout.gif](/images/android-practical-layout/boundlayout.gif)

手势代码：

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (!isEnabled()) return false;
    int action = ev.getActionMasked();
    int pointerIndex;
    switch (action) {

        case MotionEvent.ACTION_DOWN:
            mActivePointerId = ev.getPointerId(0);
            isBeingDragged = false;
            pointerIndex = ev.findPointerIndex(mActivePointerId);
            if (pointerIndex < 0) {
                return false;
            }
            mLastMotionX = ev.getX(pointerIndex);
            mLastMotionY = ev.getY(pointerIndex);
            break;

        case MotionEvent.ACTION_MOVE:
            if (mActivePointerId == MotionEvent.INVALID_POINTER_ID) {
                return false;
            }
            pointerIndex = ev.findPointerIndex(mActivePointerId);
            if (pointerIndex < 0) {
                return false;
            }
            startDragging(ev.getX(pointerIndex), ev.getY(pointerIndex));
            break;

        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            isBeingDragged = false;
            mActivePointerId = MotionEvent.INVALID_POINTER_ID;
            break;
    }
    return isBeingDragged;
}

public boolean onTouchEvent(MotionEvent ev) {
    if (!isEnabled()) return false;
    int action = ev.getActionMasked();
    int pointerIndex;
    switch (action) {

        case MotionEvent.ACTION_DOWN:
            mActivePointerId = ev.getPointerId(0);
            isBeingDragged = false;
            break;

        case MotionEvent.ACTION_MOVE: {
            pointerIndex = ev.findPointerIndex(mActivePointerId);
            if (pointerIndex < 0) {
                return false;
            }
            startDragging(ev.getX(pointerIndex), ev.getY(pointerIndex));
            float x = ev.getX(pointerIndex);
            float y = ev.getY(pointerIndex);
            float value = mOrientation == HORIZONTAL ? x : y;
            float lastValue = mOrientation == HORIZONTAL ? mLastMotionX : mLastMotionY;
            if (isBeingDragged) {
                int offset = (int) ((value - lastValue) / DRAGGING_RESISTANCE);
                if (mDirection == DIRECTION_POSITIVE && mContentOffset + offset < 0 ||
                        mDirection == DIRECTION_NEGATIVE && mContentOffset + offset > 0) {
                    offset = -mContentOffset;
                }
                offsetChildren(offset);
                mLastMotionX = x;
                mLastMotionY = y;
            }
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            if (isBeingDragged) {
                isBeingDragged = false;
                animateOffsetToZero();
            }
            mActivePointerId = MotionEvent.INVALID_POINTER_ID;
            return false;

        case MotionEvent.ACTION_POINTER_DOWN: {
            pointerIndex = ev.getActionIndex();
            if (pointerIndex < 0) {
                return false;
            }
            mActivePointerId = ev.getPointerId(pointerIndex);
            mLastMotionX = ev.getX(pointerIndex);
            mLastMotionY = ev.getY(pointerIndex);
            break;
        }

        case MotionEvent.ACTION_POINTER_UP:
            pointerIndex = ev.getActionIndex();
            int pointerId = ev.getPointerId(pointerIndex);
            if (pointerId == mActivePointerId) {
                final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                mActivePointerId = ev.getPointerId(newPointerIndex);
                mLastMotionX = ev.getX(newPointerIndex);
                mLastMotionY = ev.getY(newPointerIndex);
            }
            break;
    }

    return true;
}

private boolean canScroll(View view, float x, float y, int direction) {
    if (view instanceof ViewGroup) {
        ViewGroup viewGroup = (ViewGroup) view;
        int scrollX = viewGroup.getScrollX();
        int scrollY = viewGroup.getScrollY();
        int count = viewGroup.getChildCount();
        for (int i = count - 1; i >= 0; --i) {
            View child = viewGroup.getChildAt(i);
            if (x + scrollX >= child.getLeft() &&
                    x + scrollX < child.getRight() &&
                    y + scrollY >= child.getTop() &&
                    y + scrollY < child.getBottom() &&
                    canScroll(child, x + scrollX - child.getLeft(), y + scrollY - child.getTop(), direction)) {
                return true;
            }
        }
    }
    return mOrientation == HORIZONTAL ? view.canScrollHorizontally(direction) : view.canScrollVertically(direction);
}

private void startDragging(float x, float y) {
    if (isBeingDragged) return;
    if (mOrientation == HORIZONTAL) {
        startDraggingHorizontal(x, y);
    } else {
        startDraggingVertical(x, y);
    }
}

private void startDraggingHorizontal(float x, float y) {
    float diffX = x - mLastMotionX;
    float diffY = y - mLastMotionY;
    if (Math.abs(diffX) < Math.abs(diffY)) return;
    if (diffX > mTouchSlop && !canScroll(mContent, x, y, DIRECTION_NEGATIVE) ||
            diffX < -mTouchSlop && !canScroll(mContent, x, y, DIRECTION_POSITIVE)) {
        mLastMotionX = mLastMotionX + (diffX > 0 ? mTouchSlop : -mTouchSlop);
        mDirection = diffX > 0 ? DIRECTION_POSITIVE : DIRECTION_NEGATIVE;
        isBeingDragged = true;
        requestDisallowInterceptTouchEvent(true);
    }
}

private void startDraggingVertical(float x, float y) {
    float diffX = x - mLastMotionX;
    float diffY = y - mLastMotionY;
    if (Math.abs(diffX) > Math.abs(diffY)) return;
    if (diffY > mTouchSlop && !canScroll(mContent, x, y, DIRECTION_NEGATIVE) ||
            diffY < -mTouchSlop && !canScroll(mContent, x, y, DIRECTION_POSITIVE)) {
        mLastMotionY = mLastMotionY + (diffY > 0 ? mTouchSlop : -mTouchSlop);
        mDirection = diffY > 0 ? DIRECTION_POSITIVE : DIRECTION_NEGATIVE;
        isBeingDragged = true;
        requestDisallowInterceptTouchEvent(true);
    }
}

private void offsetChildren(int offset) {
    if (offset == 0) return;
    if (mOrientation == HORIZONTAL) {
        offsetHorizontal(offset);
    } else {
        offsetVertical(offset);
    }
}

private void offsetHorizontal(int offset) {
    if (mHeader != null) {
        int displayMode = ((LayoutParams) mHeader.getLayoutParams()).getDisplayMode();
        if (displayMode == LayoutParams.DISPLAY_MODE_EDGE && mHeader.getLeft() <= 0) {
            if (mHeader.getLeft() + offset <= 0) {
                mHeaderOffset += offset;
                ViewCompat.offsetLeftAndRight(mHeader, offset);
            } else {
                mHeaderOffset = 0;
                ViewCompat.offsetLeftAndRight(mHeader, 0 - mHeader.getLeft());
            }
        } else if (displayMode == LayoutParams.DISPLAY_MODE_SCROLL) {
            mHeaderOffset += offset;
            ViewCompat.offsetLeftAndRight(mHeader, offset);
        }
    }
    if (mContent != null) {
        mContentOffset += offset;
        ViewCompat.offsetLeftAndRight(mContent, offset);
    }
    if (mFooter != null) {
        int displayMode = ((LayoutParams) mFooter.getLayoutParams()).getDisplayMode();
        if (displayMode == LayoutParams.DISPLAY_MODE_EDGE && mFooter.getRight() >= getMeasuredWidth()) {
            if (mFooter.getRight() + offset >= getMeasuredWidth()) {
                mFooterOffset += offset;
                ViewCompat.offsetLeftAndRight(mFooter, offset);
            } else {
                mFooterOffset = 0;
                ViewCompat.offsetLeftAndRight(mFooter, getMeasuredWidth() - mFooter.getRight());
            }
        } else if (displayMode == LayoutParams.DISPLAY_MODE_SCROLL) {
            mFooterOffset += offset;
            ViewCompat.offsetLeftAndRight(mFooter, offset);
        }
    }
}

private void offsetVertical(int offset) {
    if (mHeader != null) {
        int displayMode = ((LayoutParams) mHeader.getLayoutParams()).getDisplayMode();
        if (displayMode == LayoutParams.DISPLAY_MODE_EDGE && mHeader.getTop() <= 0) {
            if (mHeader.getTop() + offset <= 0) {
                mHeaderOffset += offset;
                ViewCompat.offsetTopAndBottom(mHeader, offset);
            } else {
                mHeaderOffset = 0;
                ViewCompat.offsetTopAndBottom(mHeader, 0 - mHeader.getTop());
            }
        } else if (displayMode == LayoutParams.DISPLAY_MODE_SCROLL) {
            mHeaderOffset += offset;
            ViewCompat.offsetTopAndBottom(mHeader, offset);
        }
    }
    if (mContent != null) {
        mContentOffset += offset;
        ViewCompat.offsetTopAndBottom(mContent, offset);
    }
    if (mFooter != null) {
        int displayMode = ((LayoutParams) mFooter.getLayoutParams()).getDisplayMode();
        if (displayMode == LayoutParams.DISPLAY_MODE_EDGE && mFooter.getBottom() >= getMeasuredHeight()) {
            if (mFooter.getBottom() + offset >= getMeasuredHeight()) {
                mFooterOffset += offset;
                ViewCompat.offsetTopAndBottom(mFooter, offset);
            } else {
                mFooterOffset = 0;
                ViewCompat.offsetTopAndBottom(mFooter, getMeasuredHeight() - mFooter.getBottom());
            }
        } else if (displayMode == LayoutParams.DISPLAY_MODE_SCROLL) {
            mFooterOffset += offset;
            ViewCompat.offsetTopAndBottom(mFooter, offset);
        }
    }
}
```

## RefreshLayout

描述：下拉刷新，挺多人为这个东西搞到头大，官方的`SwipeRefreshLaout`虽然看上去很好，但是定制能力比较弱，不符合大众口味。众口难调，便出现了一些优秀库，热门的有[Ultra-Pull-To-Refresh](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh)，[🔥SmartRefreshLayout](https://github.com/scwang90/SmartRefreshLayout)等。

然而我贩剑要自己实现，仅因为我想做一个`fling`效果自然的下拉刷新，另外嵌套滑动的出现令我意识到做下拉刷新相对容易了。

以下介绍的下拉刷新主要原理就是嵌套滑动，下拉刷新主要涉及的是头部的显示于隐藏，支持嵌套滑动的组件，在`onNestedScroll`和`onNestedPreScroll`方法中进行头部的显示和隐藏。而不支持嵌套滑动的组件，可以通过分发手势时，进行类似嵌套滑动处理。另外处理`fling`时，向下滚动时，先隐藏头部，再进行内容滚动；向上滚动时，先内容滚动，如果时刷新状态，则滚动显示头部。

github: [RefreshLayout](https://github.com/4ndroidev/RefreshLayout)

![recyclerview_sample.gif](/images/android-practical-layout/recyclerview_sample.gif)![nestedscrollview_sample.gif](/images/android-practical-layout/nestedscrollview_sample.gif)
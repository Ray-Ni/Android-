# RecyclerView浅析

## RecyclerView 和 ListView 异同点

作为ListView的升级版，RecyclerView和ListView的相同点包括：

* 都支持列表的展示，能够滚动（请原谅我的废话）

* 同样通过Adapter来实现数据与View的绑定

* 都支持通过ViewType来支持不同的显示样式

他们的区别点更多：

* ViewHolder对于ListView来说，是最佳实践，但并非强制要求，而对RecyclerView来说，则是强制要求

* ListView支持Item devider属性，而RecyclerView则需要通过设置ItemDecoration来实现

* ListView支持Header，而RecyclerView并不支持

* ListView支持ItemClickListener, 而RecyclerView则支持ItemTouchListener

* RecyclerView可以通过匹配LayoutParamter来支持Grid，StaggeredGrid效果，并且支持横向滚动

* AdsListView\(ListView的基类\).LayoutParameter不支持margin，而RecyclerView可以

## RecyclerView的实现

## 类结构

![](https://img.alicdn.com/tps/TB13_FsLpXXXXXUXFXXXXXXXXXX-1714-1004.png)

与ListView相比，RecyclerView胜在分工明确：

* Recycler负责实现ViewHolder的回收重用

* LayoutManager负责实现itemview的布局，并且处理滚动动画效果

* ItemDecoration负责在LayoutManager的基础上，微调itemview的布局

* Adapter负责创建ViewHolder，以及ViewHolder和data数据之间的绑定关系

这样的设计，解耦了RecyclerView的内部模块，每个内部模块可以专心的实现自己的功能，而不必担心影响其他的部分，另外，提供了更加丰富有效的定制手段来自定义部分效果。

## 核心代码分析

以下拉滚动为例分析RecyclerView的实现逻辑：

1.0 onTouchEvent

```js

public void onTouchEvent(MotionEvent e){

    ......

    final boolean canScrollHorizontally = mLayout.canScrollHorizontally();

    final boolean canScrollVertically = mLayout.canScrollVertically();

    ......

    final MotionEvent vtev = MotionEvent.obtain(e);

    ......

    switch (action) {

        ......

        case  MotionEvent.ACTION_MOVE:{

        ......

        final int x = (int) (MotionEventCompat.getX(e, index) + 0.5f);

        final int y = (int) (MotionEventCompat.getY(e, index) + 0.5f);

        int dx = mLastTouchX - x;

        int dy = mLastTouchY - y;

        ......

        mLastTouchX = x - mScrollOffset[0];

        mLastTouchY = y - mScrollOffset[1];

        if (scrollByInternal(canScrollHorizontally ? dx : 0,

             canScrollVertically ? dy : 0,

             vtev)) {    

                 getParent().requestDisallowInterceptTouchEvent(true);

             }

       }

    }

}

```

计算与上一个touch消息相比，Y轴的位移，并调用scrollByInternal函数

1.1 scrollByInternal

```js

boolean scrollByInternal(int x, int y, MotionEvent ev) {

  int unconsumedX = 0, unconsumedY = 0;  int consumedX = 0, consumedY = 0;

  ......

  if (mAdapter != null) {

    ......

    if (x != 0) {

      consumedX = mLayout.scrollHorizontallyBy(x, mRecycler, mState);        

      unconsumedX = x - consumedX;    

    }

    if (y != 0) {

      consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);       

      unconsumedY = y - consumedY; 

    } 

    ......

  }

  ......

}

```

调用LayoutManager.scrollVerticalBy函数

2.0 LinearLayoutManager.scrollVerticalBy

```js

@Overridepublic int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {

  if (mOrientation == HORIZONTAL) {

    return 0;

  } 

  return scrollBy(dy, recycler, state);

}

```

简单的调用scrollBy函数

2.1 LinearLayoutManager.scrollBy

```js

int scrollBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {

  if (getChildCount() == 0 || dy == 0) { 

    return 0; 

  } 

  ......

  final int layoutDirection = dy > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;   

  final int absDy = Math.abs(dy);

  updateLayoutState(layoutDirection, absDy, true, state);

  final int consumed = 

    mLayoutState.mScrollingOffset + fill(recycler, mLayoutState, state, false);

  if (consumed < 0) { 

    return 0;

  }

  final int scrolled = absDy > consumed ? layoutDirection * consumed : dy;     

  mOrientationHelper.offsetChildren(-scrolled);

  ......

  return scrolled;

}

```

主要调用了updateLayoutState， fill， mOrientationHelper.offsetChildren三个函数

2.2 LinearLayoutManager.updateLayoutState

```js

private void updateLayoutState(int layoutDirection, int requiredSpace, boolean canUseExistingSpace, RecyclerView.State state) {

  ......

  if (layoutDirection == LayoutState.LAYOUT_END) {

    mLayoutState.mExtra += mOrientationHelper.getEndPadding();    // get the first child in the direction we are going

    final View child = getChildClosestToEnd();    // the direction in which we are traversing children

    mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD : LayoutState.ITEM_DIRECTION_TAIL;

    mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;    mLayoutState.mOffset = mOrientationHelper.getDecoratedEnd(child);    // calculate how much we can scroll without adding new children (independent of layout)

    scrollingOffset = mOrientationHelper.getDecoratedEnd(child) - mOrientationHelper.getEndAfterPadding();

  } else {

    ......

  }

  mLayoutState.mAvailable = requiredSpace;

  if (canUseExistingSpace) {

     mLayoutState.mAvailable -= scrollingOffset;

  }

  mLayoutState.mScrollingOffset = scrollingOffset;

}

```

这个函数主要是计算了几个值，为后面的View重用和滚动做准备，有两个值需要注意一下：

mScrollingOffset是不添加新元素的情况下，能够滚动的具体

mAvailable是需要通过添加新元素来补充的滚动距离

2.3 LinearLayoutManager.fill

```js

int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {

  final int start = layoutState.mAvailable;

  ......

  int remainingSpace = layoutState.mAvailable + layoutState.mExtra;

  LayoutChunkResult layoutChunkResult = new LayoutChunkResult();

  while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {

    layoutChunkResult.resetInternal();

    layoutChunk(recycler, state, layoutState, layoutChunkResult);

  

    ......

    /**

     * Consume the available space if:

     * * layoutChunk did not request to be ignored

     * * OR we are laying out scrap children

     * * OR we are not doing pre-layout

     */

    if (!layoutChunkResult.mIgnoreConsumed || mLayoutState.mScrapList != null || !state.isPreLayout()) {

      layoutState.mAvailable -= layoutChunkResult.mConsumed;

      // we keep a separate remaining space because mAvailable is important for recycling

      remainingSpace -= layoutChunkResult.mConsumed;

    }

    ......

  }

  ......

  return start - layoutState.mAvailable;

}

```

通过循环调用layoutChunk函数来添加足够的View，以满足滚动的需求

2.4 LinearLayoutManager.layoutChunk
```js
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);
        if (view == null) {
            // if we are laying out views in scrap, this may return null which means there is
            // no more items to layout.
            result.mFinished = true;
            return;
        }
        LayoutParams params = (LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
                addView(view, 0);
            }
        } else {
            ......
        }
        measureChildWithMargins(view, 0, 0);
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
        int left, top, right, bottom;
        
        ......
        
        // We calculate everything with View's bounding box (which includes decor and margins)
        // To calculate correct layout position, we subtract margins.
        layoutDecorated(view, left + params.leftMargin, top + params.topMargin,
                right - params.rightMargin, bottom - params.bottomMargin);
        
        ......
    }
```
layoutChunk函数主要做了四件事：
- 调用layoutState.next()获得一个View
- 把获得的View添加到RecyclerView中
- 调用measureChildWithMargin函数，计算子元素的size
- 调用layoutDecorated函数，layout子元素

3.0 LayoutState.next
```js
       /**
         * Gets the view for the next element that we should layout.
         * Also updates current item index to the next item, based on {@link #mItemDirection}
         *
         * @return The next element that we should layout.
         */
        View next(RecyclerView.Recycler recycler) {
            if (mScrapList != null) {
                return nextViewFromScrapList();
            }
            final View view = recycler.getViewForPosition(mCurrentPosition);
            mCurrentPosition += mItemDirection;
            return view;
        }
```
这里先忽略从ScrapList获得View的可能性，直接通过recycler.getViewForPosition来获得新的View

3.1 Recycler.getViewForPosition
```js
       /**
         * Obtain a view initialized for the given position.
         *
         * This method should be used by {@link LayoutManager} implementations to obtain
         * views to represent data from an {@link Adapter}.
         * <p>
         * The Recycler may reuse a scrap or detached view from a shared pool if one is
         * available for the correct view type. If the adapter has not indicated that the
         * data at the given position has changed, the Recycler will attempt to hand back
         * a scrap view that was previously initialized for that data without rebinding.
         *
         * @param position Position to obtain a view for
         * @return A view representing the data at <code>position</code> from <code>adapter</code>
         */
        public View getViewForPosition(int position) {
            return getViewForPosition(position, false);
        }
```
3.2 Recycler.getViewForPosition
```js
        View getViewForPosition(int position, boolean dryRun) {
            if (position < 0 || position >= mState.getItemCount()) {
                throw new IndexOutOfBoundsException("Invalid item position " + position
                        + "(" + position + "). Item count:" + mState.getItemCount());
            }
            boolean fromScrap = false;
            ViewHolder holder = null;
            // 0) If there is a changed scrap, try to find from there
            if (mState.isPreLayout()) {
                holder = getChangedScrapViewForPosition(position);
                fromScrap = holder != null;
            }
            // 1) Find from scrap by position
            if (holder == null) {
                holder = getScrapViewForPosition(position, INVALID_TYPE, dryRun);
                ......
            }
            if (holder == null) {
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                ......

                final int type = mAdapter.getItemViewType(offsetPosition);
                // 2) Find from scrap via stable ids, if exists
                if (mAdapter.hasStableIds()) {
                    holder = getScrapViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
                    ......
                }
                if (holder == null && mViewCacheExtension != null) {
                    // We are NOT sending the offsetPosition because LayoutManager does not
                    // know it.
                    final View view = mViewCacheExtension
                            .getViewForPositionAndType(this, position, type);
                    if (view != null) {
                        holder = getChildViewHolder(view);
                        ......
                    }
                }
                if (holder == null) { // fallback to recycler
                    // try recycler.
                    // Head to the shared pool.
                    holder = getRecycledViewPool().getRecycledView(type);
                    ......
                }
                if (holder == null) {
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                }
            }

            // This is very ugly but the only place we can grab this information
            // before the View is rebound and returned to the LayoutManager for post layout ops.
            // We don't need this in pre-layout since the VH is not updated by the LM.
            if (fromScrap && !mState.isPreLayout() && holder
                    .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST)) {
                holder.setFlags(0, ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                if (mState.mRunSimpleAnimations) {
                    int changeFlags = ItemAnimator
                            .buildAdapterChangeFlagsForAnimations(holder);
                    changeFlags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                    final ItemHolderInfo info = mItemAnimator.recordPreLayoutInformation(mState,
                            holder, changeFlags, holder.getUnmodifiedPayloads());
                    recordAnimationInfoIfBouncedHiddenView(holder, info);
                }
            }

            boolean bound = false;
            if (mState.isPreLayout() && holder.isBound()) {
                // do not update unless we absolutely have to.
                holder.mPreLayoutPosition = position;
            } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
                if (DEBUG && holder.isRemoved()) {
                    throw new IllegalStateException("Removed holder should be bound and it should"
                            + " come here only in pre-layout. Holder: " + holder);
                }
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                holder.mOwnerRecyclerView = RecyclerView.this;
                mAdapter.bindViewHolder(holder, offsetPosition);
                attachAccessibilityDelegate(holder.itemView);
                bound = true;
                if (mState.isPreLayout()) {
                    holder.mPreLayoutPosition = position;
                }
            }

            final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
            final LayoutParams rvLayoutParams;
            if (lp == null) {
                rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
                holder.itemView.setLayoutParams(rvLayoutParams);
            } else if (!checkLayoutParams(lp)) {
                rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
                holder.itemView.setLayoutParams(rvLayoutParams);
            } else {
                rvLayoutParams = (LayoutParams) lp;
            }
            rvLayoutParams.mViewHolder = holder;
            rvLayoutParams.mPendingInvalidate = fromScrap && bound;
            return holder.itemView;
```


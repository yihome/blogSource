---
title: RecyclerView源码详解
date: 2018-07-30 16:58:48
tags: [Android,View]
categories: Android
---
[TOC]

**免责声明：** 这篇文章还没有写完，发布的原因是因为我懒的放到草稿箱了，如有错误或者以后请勿怪

# RecyclerView介绍
`RecyclerView` 是Google support-v7包中的控件，用于提供一个列表的显示，和 `ListView` 类似，但是比 `ListView` 更高级也更具扩展性，它更倾向于使用一个模块化的方式来使用多个模块共同实现一个复杂的列表控件，`RecyclerView `主要由如下几个组件构成
* `LayoutManager` 用于控制List子View的测量和布局

* `Adapter` 和 `ListView` 中的Adapter功能类似，但是已经预先添加了 `ViewHolder` ,不需要我们手动添加

* `ItemDecoration ` 用于控制和显示分割线

* `ItemAnimator` 用于实现Item显示消失时动画效果



当然除了上面所说的这些模块之外，`RecyclerView` 还把很多其他的功能也都拆分了出来，这里就不一一列举，文中遇到了在做说明。

# LayoutManager
`LayoutManager` 相当于把本身View应该完成的测量和布局过程抽离出来单独处理，而这两个步骤也是决定一个View显示效果的重要步骤，因此同一个 `RecyclerView` 完全可以通过不同的 `LayoutManager` 实现不同的显示效果，而不是使用不同的控件。

  目前Google已经预先提供好了几种 `LayoutManager` ,分别如下所示

* `LinearLayoutManager` 实现类似 `ListView` 列表样式的布局

* `GridLayoutManager` 实现类似 `GridView` 网格样式的布局

* `StaggeredGridLayoutManager` 实现了瀑布流样式的布局


我们以 `LinearLayoutManager` 为例来说明 `LayoutManager` 在 `RecyclerView` 中的重要作用，首先先说明一下 `LinearLayoutManager` 中的两个辅助类，这两个类在 `LinearLayoutManager` 的工作中也起着一定作用

* `AnchorInfo` 布局的锚点，在测量和布局过程中 `LinearLayoutManager` 都是先确定一个锚点位置，然后根据锚点，分别向上和向下去测量和布局子 `View`
* `LayoutState` 是 `LinearLayoutManager` 在填充整体布局的帮助类，他可以通过一个给定的方向获取下一个需要填充的 `View`



## onMeasure

我们都知道控件的展示主要分为测量、布局、绘制等三个阶段进行，我们首先关注 `RecyclerView` 的测量过程。代码片段会省略部分代码和注释

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    //isAutoMeasureEnabled方法返回false，表示必须要重写LayoutManager的onMeasure方法，完成测量
    //如果设置为true，则表示由RecyclerView的AutoMeasure机制完成测量
    //LinearLayoutManager已经重写了这个方法，永久返回true
    if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);

        //这里看似调用了LayoutManager的onMeasure，但是LinearLayoutManager没有重写这个方法，
        //因此这个最终是调用了RecyclerView自己默认的方法
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        //RecyclerView固定宽高或者Match_Parent的情况下measureSpecModeIsExactly为true
        final boolean measureSpecModeIsExactly =
            widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        //即如果RecyclerView的宽高不是Wrap_Content，到这里测量过程就结束了
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

        //mState是RecyclerView的State对象，这个对象存储了当前RecyclerView的各种状态
        //在RecyclerView中各个组件的交互也是通过State对象来进行的
        //mState初始mLayoutStep为STEP_START
        if (mState.mLayoutStep == State.STEP_START) {
            //dispatchLayoutStep1主要对RecyclerView的动画做了处理，没有绘制流程
            //后续在动画中在做说明
            dispatchLayoutStep1();
        }
        // 这里widthSpec和heightSpec是RecyclerView默认测量的结果
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        //实际的测量，会测量childView
        dispatchLayoutStep2();

        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // if RecyclerView has non-exact width and height and if there is at least one child
        // which also has non-exact width & height, we have to re-measure.
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
    } else {
       ......
    }
}
```

可以看到在 `onMeasure` 方法中，如果我们给 `RecyclerView` 设置了不是 `Wrap_Content` 的宽高，那么 `RecyclerView` 的测量过程很简单，直接使用父控件给定的宽高即可，只有当我们设置为 `Wrap_Content` 时，才需要先去测量子 View 。

测量子 View 的过程事实上是在 `dispatchLayoutStep2` 方法中进行的，这个方法事实上是 Layout 时调用的方法，这个不难理解，因为只有在确定了 Layout 的方式时，才能真正确定子 View 需要多少的屏幕空间

```java
private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    // Step 2: Run layout
    mState.mInPreLayout = false;
    //上述代码基本上就是在往mState中填充RecyclerView的信息，上段代码的注释中也提到State的
    //主要功能就是在RecyclerView各组件间传递信息
    mLayout.onLayoutChildren(mRecycler, mState);
    ......
}
```

`dispatchLayoutStep2` 主要的功能就是为 `mState` 的更新了一些 `RecyclerView` 的最新状态，然后调用了 `LayoutManager` 的 `onLayoutChildren` 方法，值得注意的是这个方法和上个方法一样是用于布局的，所以后面还会调用，这里暂时只说明测量的部分。

```java
 public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
     //计算锚点信息以及获取边缘的extraSpace，锚点初始化时一般是方向上的第一个View（反转的列表则为最后）
     //细节代码有兴趣可以自行研究
     ......
     if (mAnchorInfo.mLayoutFromEnd) {
         // fill towards start
         //通过锚点更新LayoutState信息,更新后LayoutState.next会向上寻找View
         updateLayoutStateToFillStart(mAnchorInfo);
         mLayoutState.mExtra = extraForStart;
         fill(recycler, mLayoutState, state, false);
         startOffset = mLayoutState.mOffset;
         final int firstElement = mLayoutState.mCurrentPosition;
         if (mLayoutState.mAvailable > 0) {
             extraForEnd += mLayoutState.mAvailable;
         }
         // fill towards end
         //通过锚点更新LayoutState信息,更新后LayoutState.next会向下寻找View
         updateLayoutStateToFillEnd(mAnchorInfo);
         mLayoutState.mExtra = extraForEnd;
         mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
         fill(recycler, mLayoutState, state, false);
         endOffset = mLayoutState.mOffset;

         if (mLayoutState.mAvailable > 0) {
             // end could not consume all. add more items towards start
             extraForStart = mLayoutState.mAvailable;
             updateLayoutStateToFillStart(firstElement, startOffset);
             mLayoutState.mExtra = extraForStart;
             fill(recycler, mLayoutState, state, false);
             startOffset = mLayoutState.mOffset;
         }
     } else {
         //这段代码几乎和上段一样，只是填充的方向不同
         ......
     }
     ......
 }
```

上面的代码中可以看出在 `onLayoutChildren` 过程中，`LinearLayoutManager` 是首先确定一个锚点，然后从锚点位置开始向不同方向填充布局。真正的填充部分代码在 `fill` 方法中

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
    // bug修复代码
    ......
    //mAvailable是上面的方法中由锚点更新给layoutState的，为指定指定方向上需要填充的距离
    //mExtra也是前面计算的边缘的extraSpace，
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    //由于最开始onMeasure的时候先调用了RecyclerView默认的onMeasure方法，所以这里remainingSpace
    //就是能Recycler能取到的最大值
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        ......
        //核心代码，用于计算子View需要的空间
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        ......
        //根据layoutChunkResult缩减remainingSpace大小
    }
    ......
    return start - layoutState.mAvailable;
}
```

可以看到在 `fill` 方法中起着比较核心作用的是 `layoutChunk` 方法

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    ......
    //测量ChildView
    measureChildWithMargins(view, 0, 0);
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    int left, top, right, bottom;
    ......
    //布局ChildView
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    ......
}
```

上面的代码其实很清楚了，`layoutChunk` 方法首先获取的 `ChildView` ，对它进行测量，然后布局。恩，你没有看错就是布局 `Layout` ，通过布局方法确定 `ChildView` 的具体位置，`dispatchLayoutStep2`方法的深入展开到此结束，让我们回到 `onMeasure` 的地方在看一下我们没有看完的代码

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    ......
    if (mLayout.isAutoMeasureEnabled()) {
        ......
        //上次的代码到这了
        dispatchLayoutStep2();

        // 通过ChildView的位置综合父控件给的widthSpec和heightSpec选择合适的宽高
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        //重新进行测量，对于LinearLayoutManager来说中如果本身是WARP_CONTENT，
        //同时至少有一个ChildView有不确定（WRAP_CONTENT或者MATCH_PARENT）的宽和高
        if (mLayout.shouldMeasureTwice()) {
            //重新走一遍上面的测量
            ......
        }
    } else {
       ......
    }
}

void setMeasuredDimensionFromChildren(int widthSpec, int heightSpec) {
  	......
    //遍历ChildView，选出坐上右下四个值
    for (int i = 0; i < count; i++) {
        View child = getChildAt(i);
        final Rect bounds = mRecyclerView.mTempRect;
        getDecoratedBoundsWithMargins(child, bounds);
        if (bounds.left < minX) {
            minX = bounds.left;
        }
        if (bounds.right > maxX) {
            maxX = bounds.right;
        }
        if (bounds.top < minY) {
            minY = bounds.top;
        }
        if (bounds.bottom > maxY) {
            maxY = bounds.bottom;
        }
    }
    mRecyclerView.mTempRect.set(minX, minY, maxX, maxY);
    //根据获得的Rect范围综合父控件给的值综合获取最终的宽高
    setMeasuredDimension(mRecyclerView.mTempRect, widthSpec, heightSpec);
}
```

好了到此为止，整个测量过程基本上清楚了，下面我们根据一张时序图回顾一下整个流程

```sequence
participant RecyclerView as A
participant LinearLayoutManager as B
A->A:onMeasure
A->B:isAutoMeasureEnabled
A->A:dispatchLayoutStep2
A->B:onLayoutChildren
B->B:fill
B->B:layoutChunk
B->B:measureChildWithMargins
B->B:layoutDecoratedWithMargins
A->B:setMeasuredDimensionFromChildren
A->B:shouldMeasureTwice
B->B:calculate measurezise......
B->A:setMeasuredDimension

```

可以看出 `RecyclerView` 在整个测量过程还是整个流程的主导，`LayoutManager` 更多像是一个提供具体配置参数的。这个表可以作为我们自定义 `LayoutManager` 时的一个参考。

### onLayout

继续看布局的相关方法，首先从 `RecyclerView` 的 `onLayout` 方法开始

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    //直接调用了dispatchLayout
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}

void dispatchLayout() {
    if (mAdapter == null) {
        return;
    }
    if (mLayout == null) {
        return;
    }
    mState.mIsMeasuring = false;
    //mLayoutStep初始为STEP_START，然后会在dispatchLayoutStep1方法修改为STEP_LAYOUT
    //然后在dispatchLayoutStep2方法中修改为STEP_ANIMATIONS
    //除非若RecyclerView宽高为Wrap_Content，否则此处为STEP_START
    if (mState.mLayoutStep == State.STEP_START) {
        //完成一次完整的测量布局
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
               || mLayout.getHeight() != getHeight()) {
        //测量布局已经但是，种种原因，大小改变了，需要重新测量布局
        //这个方法把RecyclerView的测量宽高以一EXACTLY的形式给了LayoutManager
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

在 `onLayout` 中 `RecyclerView` 并没有做什么多余的操作，而是直接调用了 `dispatchLayout` 方法，这个方法会根据我们在 `onMeasure` 中的行为判断是否需要调用 `dispatchLayoutStep1` 和 `dispatchLayoutStep2` ，但是无论哪种情况都会通过 `setExactMeasureSpecsFrom` 给 `LayoutManager` 设置一个确切的宽高，最后调用 `dispatchLayoutStep3` 方法。

```java
private void dispatchLayoutStep3() {
    //STEP_ANIMATIONS状态就可以看出来，这个方法主要是完成前面dispatchLayoutStep1中
    //设置好的动画
    mState.assertLayoutStep(State.STEP_ANIMATIONS);
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.mLayoutStep = State.STEP_START;
    if (mState.mRunSimpleAnimations) {
        //这个整段代码全都是动画相关的
        ......
    }

    // 一些状态的置位和资源的释放
    ......
    if (mLayout.mPrefetchMaxObservedInInitialPrefetch) {
        // Initial prefetch has expanded cache, so reset until next prefetch.
        // This prevents initial prefetches from expanding the cache permanently.
        mLayout.mPrefetchMaxCountObserved = 0;
        mLayout.mPrefetchMaxObservedInInitialPrefetch = false;
        mRecycler.updateViewCacheSize();
    }

    //这里也是一样，部分状态的初始化
    mLayout.onLayoutCompleted(mState);
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mViewInfoStore.clear();
    if (didChildRangeChange(mMinMaxLayoutPositions[0], mMinMaxLayoutPositions[1])) {
        dispatchOnScrolled(0, 0);
    }
    recoverFocusFromState();
    resetFocusInfo();
}
```

￼![](./153.jpg)

这就结束了？，没错到此为止 `LinearLayoutManager` 或者说 `RecyclerView` 的 auto measure 就是这样的，当然 `LayoutManager` 还有一些其他功能这里还没有涉及到或者没有展开，后面遇到再进行说明。

# Recycler

按照顺序这里应该是本应该是 `Adapter` ，但是我发现如果没有 `Recycler` 存在 `Adapter` 根本玩不下去，看名字 `Recycler` 就感觉像是 `RecyclerView` 的核心，确实它也掌管了所有 `ChildView` 的生成、复用、回收，确实也是核心，首先我们介绍一下 `Recycler` 中的几个缓存队列

* `mChangedScrap` item被标记为更新、有动画且动画支持变化

* `mAttachedScrap` 
* `mCachedViews`

`Recycler` 也还是得从测量过程中的 `LayoutChunk` 说起，虽然这个 `Recycler` 对象在一开始 `onLayoutChildren` 就作为参数传递到了 `LayoutManager` ，但是一直没有正经的使用它。

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
    //从LayoutState中获取下一个View
    View view = layoutState.next(recycler);
    if (view == null) {
       	//说明已经没有更多的View，到底或者到顶了
        result.mFinished = true;
        return;
    }
    LayoutParams params = (LayoutParams) view.getLayoutParams();
    //mScrapList一般情况下都为null，只有在为动画做准备时才会有内存
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                                     == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
        //暂时可以不考虑
        ......
    }
   	measureChildWithMargins(view, 0, 0);
    ......
}

View next(RecyclerView.Recycler recycler) {
    //mScrapList一般情况下都为null，这里暂不考虑
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    //调用了getViewForPosition来获取真正View
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
```

`layoutChunk` 根本意义上说也还没有使用 `Recycler` 对象，但是这里算上获取 `childView` 的一个起点，所以我们从这里开始分析，后续的几个方法也都没有做太多工作。到了 `tryGetViewHolderForPositionByDeadline` 真正的逻辑开始了。代码之前我们先整理一下逻辑，这个方法的的主要工作如下

1. 如果进行了 `PreLayout` 则会优先从 `mChangedScrap` 中查找 `ViewHolder` , `PreLayout` 这个行为发生在 `dispatchLayoutStep1` 中，虽然是 `PreLayout` 但是终究还是会去 `onLayoutChildren` ，然后最后还是会调用 `tryGetViewHolderForPositionByDeadline` ，所以本质上来说第一次来步骤1肯定会跳过 
2. 

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException(...);
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;
    // 0) If there is a changed scrap, try to find from there
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                // recycle holder (and unscrap if relevant) since it can't be used
                if (!dryRun) {
                    // we would like to recycle this but need to make sure it is not used by
                    // animation logic etc.
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException(...);
        }

        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) Find from scrap/cache via stable ids, if exists
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                                               type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            final View view = mViewCacheExtension
                .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException(...);
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException(...);
                }
            }
        }
        if (holder == null) { // fallback to pool
           
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            if (ALLOW_THREAD_GAP_WORK) {
                // only bother finding nested RV if prefetching
                RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                if (innerView != null) {
                    holder.mNestedRecyclerView = new WeakReference<>(innerView);
                }
            }

            long end = getNanoTime();
            mRecyclerPool.factorInCreateTime(type, end - start);
            
        }
    }

    // This is very ugly but the only place we can grab this information
    // before the View is rebound and returned to the LayoutManager for post layout ops.
    // We don't need this in pre-layout since the VH is not updated by the LM.
    if (fromScrapOrHiddenOrCache && !mState.isPreLayout() && holder
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
            throw new IllegalStateException(...);
        }
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
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
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
    return holder;
}
```


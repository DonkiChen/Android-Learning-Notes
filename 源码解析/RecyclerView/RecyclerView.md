# RecyclerView
_version: 1.1.0  默认使用 `LinearLayoutManager` 来分析_
___

## 四级缓存
`RecyclerView.Recycler.getViewForPosition` -> `RecyclerView.Recycler.tryGetViewHolderForPositionByDeadline`
`RecycledViewPool.ScrapData` 根据每个type缓存View, 最大5个
1. mAttachedScrap, mChangedViews
2. mCachedViews
3. ViewCacheExtension
4. RecycledViewPool

## PreLayout

## 缓存流程

### 存缓存
1. `RecyclerView.dispatchLayoutStep2` -> `LayoutManager.onLayoutChildren` -> `LayoutManager.detachAndScrapAttachedViews` -> `LayoutManager.scrapOrRecycleView`
    1. 将(不合法的 && 没有被移除的 && 没有固定id的)从`RecyclerView`中remove掉
        1. 将(数据合法的 && 没有被移除 && 没有更新的 && 位置已知的), 一般是滚动导致的回收和预布局阶段new出来但是不在RecyclerView显示范围内的, 加入到 `mCachedViews`
        2. 将其他的加入 `RecycledViewPool`
    2. 否则从`RecyclerView`中detach掉
        1. 将((移除的 || 数据不合法的) || 没有更新的 || 能回收的) 加入到 `mAttachedScrap`
        2. 将其他加入到 `mChangedScrap`
2. `LayoutManager.removeAndRecycleViewAt` 同 1.1

### 取缓存

`tryGetViewHolderForPositionByDeadline`开始, 先从 `mAttachedScrap` -> `mCachedViews` -> `ViewCacheExtension` -> `RecycledViewPool` -> `createViewHolder`

取缓存要满足
1. `mAttachedScrap`: 
    1. 该ViewHolder未被使用: `!holder.wasReturnedFromScrap()`
    2. 位置相等: `holder.getLayoutPosition() == position`
    3. 数据合法: `!holder.isInvalid()`
    4. 在预布局阶段或者未被移除: `(mState.mInPreLayout || !holder.isRemoved())`
2. `mCachedViews`:
    1. 数据合法: `!holder.isInvalid()`
    2. 位置相等: `holder.getLayoutPosition() == position`
    3. 没有被过渡动画使用: `!holder.isAttachedToTransitionOverlay()`
3. `ViewCacheExtension`: 默认为空
4. `RecycledViewPool`: 
    1. 相同 `viewType`

如果取到的ViewHolder(已经绑定过 || 需要更新 || 数据不合法) 重新绑定 `bindViewHolder`

![取缓存流程图](https://user-gold-cdn.xitu.io/2019/9/8/16d1118c08b4f888?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
[参考链接](https://juejin.im/post/6844903937896611847)

## Item 的五种行为
1. PERSISTENT: 布局前存在, 布局后也存在
2. REMOVED: 布局前存在, 布局后被删除
3. ADDED: 布局前不存在, 布局后添加
4. DISAPPEARING: 布局前后数据存在, 从可见到不可见
5. APPEARING: 布局前后数据存在, 从不可见到可见

## 布局步骤
1. `RecyclerView.dispatchLayoutStep1`
    * 处理adapter的更新
    * 决定应该运行何种动画
    * 保存当前View的信息
    * 如果有必要, 进行预测性布局, 并且保存信息
2. `RecyclerView.dispatchLayoutStep2`: 摆放(未添加时在此时添加)子View
3. `RecyclerView.dispatchLayoutStep3`: 执行动画, 清理信息

## ViewHolder

### Flags

`ViewHolder.mPosition`:位置
`ViewHolder.mItemId`: 当 `RecyclerView.Adapter.setHasStableIds` 传入 true, 并且 `RecyclerView.Adapter.getItemId` 被覆写的时候
`ViewHolder.mItemViewType`:布局类型

1. FLAG_BOUND: 当前ViewHolder被绑定到某个位置了, `ViewHolder.mPosition`, `ViewHolder.mItemId`, `ViewHolder.mItemViewType` 都是对的
2. FLAG_UPDATE: 当前ViewHolder数据是过时的, `ViewHolder.mPosition`, `ViewHolder.mItemId`没变
3. FLAG_INVALID: 当前数据不合法, mPosition 和 mItemId 不可信, 必须被重新绑定
4. FLAG_REMOVED: 数据被删除了, View可能还被用于动画中
5. FLAG_NOT_RECYCLABLE: 当前View不应该被回收, 例如处于动画中
6. FLAG_RETURNED_FROM_SCRAP: 当前ViewHolder是从 scrap 中返回的, 需要重新通过 addView 添加到父视图
7. FLAG_IGNORE: 完全由LayoutManager管理, 不会回收, 除非LayoutManager被替换
8. FLAG_TMP_DETACHED: 从父视图detach
9. FLAG_ADAPTER_POSITION_UNKNOWN: 
10. FLAG_ADAPTER_FULLUPDATE: 全更新, 即没有传入 payload 的更新
11. FLAG_MOVED: ViewHolder变化时, 用于ItemAnimator
12. FLAG_APPEARED_IN_PRE_LAYOUT: ViewHolder在预布局阶段, 用于ItemAnimator

## notify相关

1. `notifyDataSetChanged`, 会给所有ViewHolder加上 ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID
2. `notifyItemRangeChanged`, 会给相应ViewHolder加上 ViewHolder.FLAG_UPDATE
3. `notifyItemRangeRemoved`, 会给相应ViewHolder加上 ViewHolder.FLAG_REMOVED

## 修改单独项目, 调用 `notifyDataSetChanged` 效率低的原因

`notifyDataSetChanged`会将所有View remove掉, 加到最后一级缓存 `RecycledViewPool` 中, 然后取出缓存后每个`ViewHolder`都要进行重新绑定

## 测量

### 单个child的尺寸测量

```Java
    /**
     * 生成子控件的MeasureSpec
     *
     * @param parentSize 父控件的尺寸, 指RecyclerView
     * @param parentMode 父控件的测量模式
     * @param childDimension 子控件的尺寸, 或 MATCH_PARENT/WRAP_CONTENT. 从LayoutParams获取
     * @param canScroll 父控件在该方向上能否滑动
     *
     * @return 返回一个子控件的 MeasureSpec
     */
    public static int getChildMeasureSpec(int parentSize, int parentMode, int padding,
            int childDimension, boolean canScroll) {
        int size = Math.max(0, parentSize - padding);
        int resultSize = 0;
        int resultMode = 0;
        if (canScroll) {
            //如果父控件能滚动
            if (childDimension >= 0) {
                //如果子控件有指定的大小, 那就是 EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //如果子控件为 MATCH_PARENT, 则去看父控件测量模式
                switch (parentMode) {
                    case MeasureSpec.AT_MOST:
                    case MeasureSpec.EXACTLY:
                        //AT_MOST 或者 EXACTLY, 则使用父控件的尺寸和测量模式
                        resultSize = size;
                        resultMode = parentMode;
                        break;
                    case MeasureSpec.UNSPECIFIED:
                        //如果是UNSPECIFIED, 则子控件也是UNSPECIFIED
                        resultSize = 0;
                        resultMode = MeasureSpec.UNSPECIFIED;
                        break;
                }
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //如果子控件是WRAP_CONTENT, 则是UNSPECIFIED
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
        } else {
            //如果父控件不能滑动
            if (childDimension >= 0) {
                //如果子控件有指定的大小, 那就是 EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //如果子控件为 MATCH_PARENT, 则用父控件的尺寸和测量模式
                resultSize = size;
                resultMode = parentMode;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //如果子控件是WRAP_CONTENT, 则用父控件的尺寸
                resultSize = size;
                if (parentMode == MeasureSpec.AT_MOST || parentMode == MeasureSpec.EXACTLY) {
                    resultMode = MeasureSpec.AT_MOST;
                } else {
                    resultMode = MeasureSpec.UNSPECIFIED;
                }
            }
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

### 整个RecyclerView的尺寸测量

```Java

    /**
     * RecyclerView的默认测量方法
     */
    void defaultOnMeasure(int widthSpec, int heightSpec) {
        // calling LayoutManager here is not pretty but that API is already public and it is better
        // than creating another method since this is internal.
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

        setMeasuredDimension(width, height);
    }

    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        if (mLayout == null) {
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }
        if (mLayout.isAutoMeasureEnabled()) {
            //如果开启了isAutoMeasureEnabled, 默认是开启的
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);

            //默认调用 RecyclerView.defaultOnMeasure
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

            final boolean measureSpecModeIsExactly =
                    widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
            if (measureSpecModeIsExactly || mAdapter == null) {
                return;
            }
            //如果 adapter不为空且 宽/高至少一个不确定时

            if (mState.mLayoutStep == State.STEP_START) {
                dispatchLayoutStep1();
            }
            // LayoutManager 存 spec
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;

            // 这一步会进行子控件的测量和放置, 就能拿到宽高了
            dispatchLayoutStep2();

            // 这里会遍历所有子控件 拿到父控件的一个上下左右的坐标, 然后调用setMeasuredDimension设置尺寸
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // 如果宽高都不确定, 并且至少有一个子控件的宽高也是都不确定的, 需要重新测量一次
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();

                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
        } else {
            if (mHasFixedSize) {
                mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
                return;
            }
            // custom onMeasure
            if (mAdapterUpdateDuringMeasure) {
                startInterceptRequestLayout();
                onEnterLayoutOrScroll();
                processAdapterUpdatesAndSetAnimationFlags();
                onExitLayoutOrScroll();

                if (mState.mRunPredictiveAnimations) {
                    mState.mInPreLayout = true;
                } else {
                    // consume remaining updates to provide a consistent state with the layout pass.
                    mAdapterHelper.consumeUpdatesInOnePass();
                    mState.mInPreLayout = false;
                }
                mAdapterUpdateDuringMeasure = false;
                stopInterceptRequestLayout(false);
            } else if (mState.mRunPredictiveAnimations) {
                // If mAdapterUpdateDuringMeasure is false and mRunPredictiveAnimations is true:
                // this means there is already an onMeasure() call performed to handle the pending
                // adapter change, two onMeasure() calls can happen if RV is a child of LinearLayout
                // with layout_width=MATCH_PARENT. RV cannot call LM.onMeasure() second time
                // because getViewForPosition() will crash when LM uses a child to measure.
                setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
                return;
            }

            if (mAdapter != null) {
                mState.mItemCount = mAdapter.getItemCount();
            } else {
                mState.mItemCount = 0;
            }
            startInterceptRequestLayout();
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            stopInterceptRequestLayout(false);
            mState.mInPreLayout = false; // clear
        }
    }
```

### 嵌套RecyclerView为什么会导致内部的RecyclerView全部显示

外部`RecyclerView`宽度和高度`match_parent`, 内部`RecyclerView`宽度`match_parent`和高度`wrap_content`时, 在外部RecyclerView测量内部`RecyclerView`高度的时候, `RecyclerView.LayoutManager.getChildMeasureSpec` 返回了`size = 0 && mode = MeasureSpec.UNSPECIFIED`, 内部`RecyclerView`在测量时就会去放置全部子控件(因为`LinearLayoutManager.LayoutState.mInfinite` 为 true, 详见下方`LinearLayoutManager.resolveIsInfinite`), 然后计算所有子控件的高度作为内部`RecyclerView`的高度

简而言之: `RecyclerView`(在可滑动的方向上)对子控件为`wrap_content`的测量模式是`MeasureSpec.UNSPECIFIED`, `RecyclerView`自身对`MeasureSpec.UNSPECIFIED`的测量方式又是将全部子控件都放置出来, 然后计算总共的宽高

```Java
    boolean resolveIsInfinite() {
        // getMode() 就是 mLayoutManager.getWidthMode() 或者 mLayoutManager.getHeightMode()
        // getEnd() 就是 mLayoutManager.getWidth() 或者 mLayoutManager.getHeight()
        return mOrientationHelper.getMode() == View.MeasureSpec.UNSPECIFIED
                && mOrientationHelper.getEnd() == 0;
    }
```
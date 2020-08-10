# RecyclerView
_version: 1.1.0_
___

## 四级缓存
`RecyclerView.Recycler.getViewForPosition` -> `RecyclerView.Recycler.tryGetViewHolderForPositionByDeadline`
`RecycledViewPool.ScrapData` 根据每个type缓存View, 最大5个
1. mAttachedScrap, mChangedViews
2. mCachedViews
3. ViewCacheExtension
4. RecyclerPool

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
# View
_Version:API 28_
___

## `View#post(Runnable)`中`Runnable`的调用时机

```java
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // Postpone the runnable until we know on which thread it needs to run.
        // Assume that the runnable will be successfully placed after attach.
        getRunQueue().post(action);
        return true;
    }
```

1. 当`View`没有被attach到`Window`上时, 会将传入的`Runnable`存入队列中, 等到`View#dispatchAttachedToWindow(AttachInfo, int)`被调用时执行队列中的`Runnable`

ViewRootImpl会在第一次调用`performTraversals()`时(当Activity第一次resume时, `ActivityThread#handleResumeActivity` -> `WindowManagerGlobal#addView` -> `ViewRootImpl#setView` -> `ViewRootImpl#doTraversal` -> `ViewRootImpl#performTraversals`, 省略部分), 一层层遍历调用`View#dispatchAttachedToWindow(AttachInfo, int)`

```java
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        // Transfer all pending runnables.
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
        //...
    }
```

2. 当`View`已经被attach到`Window`上时, 直接执行`Runnable`

3. 当`View`从`Window`detach时, 这些`Runnable` __不会__ 被移除

## `super#onRestoreInstanceState(Parcelable)` 参数传入`BaseSavedState`的子类时, 在Api <= 22时崩溃

在重写`View#onRestoreInstanceState(Parcelable)`方法时(自定义View的状态保存), 需要调用`super#onRestoreInstanceState(Parcelable)`, 传入的参数最好为`BaseSavedState#getSuperState()`, 因为api <= 22时,`onRestoreInstanceState(Parcelable)`的抛异常条件是`state != BaseSavedState.EMPTY_STATE && state != null`, api > 22才为`state != null && !(state instanceof AbsSavedState)`

```java
    //API 22
    protected void onRestoreInstanceState(Parcelable state) {
        mPrivateFlags |= PFLAG_SAVE_STATE_CALLED;
        if (state != BaseSavedState.EMPTY_STATE && state != null) {
            throw new IllegalArgumentException("...");
        }
    }
```

```java
    //APi 23
    @CallSuper
    protected void onRestoreInstanceState(Parcelable state) {
        mPrivateFlags |= PFLAG_SAVE_STATE_CALLED;
        if (state != null && !(state instanceof AbsSavedState)) {
            throw new IllegalArgumentException("...");
        }
        //...
    }
```

## 在使用没有默认state的selector资源文件时, 无法显示的问题

1. `View#setCompoundDrawablesWithIntrinsicBounds`, `View#setRelativeDrawablesIfNeeded`, 这两个方法里会调用`Drawable#setBounds`, 设置`Drawable`的尺寸. 这两个函数默认会在解析xml时调用.

```java
    //其余类似
    if (left != null) {
        left.setBounds(0, 0, left.getIntrinsicWidth(), left.getIntrinsicHeight());
    }
```

2. 当设置了`android:constantSize = "true"`时, 设置为多个drawable中最大的尺寸, 当没有设置或设置为false时, 返回了 -1 (因为没有设置默认state的drawable)

_Drawable的尺寸计算_

```java
    //DrawableContainer
    @Override
    public int getIntrinsicWidth() {
        if (mDrawableContainerState.isConstantSize()) {
            return mDrawableContainerState.getConstantWidth();
        }
        return mCurrDrawable != null ? mCurrDrawable.getIntrinsicWidth() : -1;
    }

    @Override
    public int getIntrinsicHeight() {
        if (mDrawableContainerState.isConstantSize()) {
            return mDrawableContainerState.getConstantHeight();
        }
        return mCurrDrawable != null ? mCurrDrawable.getIntrinsicHeight() : -1;
    }
```

```java
    //DrawableContainer.DrawableContainerState
    public final int getConstantWidth() {
        if (!mCheckedConstantSize) {
            computeConstantSize();
        }
        return mConstantWidth;
    }

    protected void computeConstantSize() {
        //...
        int count = this.mNumChildren;
        Drawable[] drawables = this.mDrawables;
        this.mConstantWidth = this.mConstantHeight = -1;
        this.mConstantMinimumWidth = this.mConstantMinimumHeight = 0;
        for(int i = 0; i < count; ++i) {
            Drawable dr = drawables[i];
            int s = dr.getIntrinsicWidth();
            if (s > this.mConstantWidth) {
                this.mConstantWidth = s;
            }
            s = dr.getIntrinsicHeight();
            if (s > this.mConstantHeight) {
                this.mConstantHeight = s;
            }
            s = dr.getMinimumWidth();
            if (s > this.mConstantMinimumWidth) {
                this.mConstantMinimumWidth = s;
            }
            s = dr.getMinimumHeight();
            if (s > this.mConstantMinimumHeight) {
                this.mConstantMinimumHeight = s;
            }
        }
    }

```

3. 绘制时的调用栈: `TextView#onDraw` -> `DrawableContainer#draw` -> `BitmapDrawable#draw` -> `BitmapDrawable#updateDstRectAndInsetsIfDirty` -> `#Drawable#copyBounds`. 当没设置android:constantSize = "true"时, mDstRect为[0, 0, -1, -1], 所以没绘制; 反之, mDstRect为[0, 0, width, height], 图片能绘制

```java

    @Override
    public void draw(Canvas canvas) {
        //...
        //设置mDstRect
        updateDstRectAndInsetsIfDirty();
        //...
        //当没设置android:constantSize = "true"时, mDstRect为[0, 0, -1, -1], 所以没绘制
        //反之, mDstRect为[0, 0, width, height], 图片能绘制
        canvas.drawBitmap(bitmap, null, mDstRect, paint);
        //...
    }

    private void updateDstRectAndInsetsIfDirty() {
        if (mDstRectAndInsetsDirty) {
            if (mBitmapState.mTileModeX == null && mBitmapState.mTileModeY == null) {
                final Rect bounds = getBounds();
                final int layoutDirection = getLayoutDirection();
                Gravity.apply(mBitmapState.mGravity, mBitmapWidth, mBitmapHeight,
                        bounds, mDstRect, layoutDirection);
                //...
            } else {
                //复制自身尺寸到mDstRect
                copyBounds(mDstRect);
                mOpticalInsets = Insets.NONE;
            }
        }
        mDstRectAndInsetsDirty = false;
    }
```
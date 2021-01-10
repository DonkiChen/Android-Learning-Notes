# ViewStub

_android: 30_

___

## 是什么

是一个不可见的, 0大小的, 用于在运行时懒加载布局的控件. 当通过调用 `setVisibility()` 或者 `inflate()` 时, 该 `ViewStub` 控件会被其子布局替换.

## 做了哪些优化

1. 测量阶段: 直接设置大小为0

    ```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(0, 0);
    }
    ```

2. 绘制阶段: 跳过了绘制
    ```java
    @Override
    public void draw(Canvas canvas) {
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
    }
    ```

## 属性

1. `layout`: 要加载的子布局id

2. `inflatedId`: 子布局加载后, 它的id

## 渲染过程

1. (未渲染时)调用`inflate()` (其实调用 `setVisibility()` 也是相当于调用了它)后, 会调用 `LayoutInflater.inflate()` (可以通过`setLayoutInflater()` 来自定义)去解析布局, 并且设置id为 `mInflatedId`

2. 然后会调用 `replaceSelfWithView()`, 这里用到了一个平时没怎么用的方法 `removeViewInLayout()` 仅移除, 不触发重新布局和绘制. 之后再通过 `addView()` 将子布局添加进去(但是这里还是触发了 布局和绘制 啊?). 即做了 => 从视图树中将 `ViewStub` 替换成 刚刚渲染出来的子布局.

## `setVisibility()` 和 `inflate()` 的区别

在第一次调用 `inflate()` 后, 会生成一个变量 `mInflatedViewRef`, 为子布局的一个弱引用对象.

### `setVisibility()`

重复调用不会报错, 第一次之后的调用, 会去直接调用子布局的对应方法

```java
    @Override
    @android.view.RemotableViewMethod(asyncImpl = "setVisibilityAsync")
    public void setVisibility(int visibility) {
        if (mInflatedViewRef != null) {
            //如果渲染过了
            View view = mInflatedViewRef.get();
            if (view != null) {
                //去控制子布局的显示
                view.setVisibility(visibility);
            } else {
                throw new IllegalStateException("setVisibility called on un-referenced view");
            }
        } else {
            //没渲染过
            super.setVisibility(visibility);
            if (visibility == VISIBLE || visibility == INVISIBLE) {
                inflate();
            }
        }
    }
```

### `inflate()`

重复调用会抛异常 `IllegalStateException`: "ViewStub must have a non-null ViewGroup viewParent"

```java
    public View inflate() {
        final ViewParent viewParent = getParent();

        if (viewParent != null && viewParent instanceof ViewGroup) {
            //第一次调用
            if (mLayoutResource != 0) {
                final ViewGroup parent = (ViewGroup) viewParent;
                final View view = inflateViewNoAdd(parent);
                replaceSelfWithView(view, parent);

                mInflatedViewRef = new WeakReference<>(view);
                if (mInflateListener != null) {
                    mInflateListener.onInflate(this, view);
                }

                return view;
            } else {
                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
            }
        } else {
            //因为第一次调用后会从父布局中将ViewStub移除, 所以 getParent() 会为 null
            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
        }
    }
```
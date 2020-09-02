# Window, View, Activity
_API 28_
___

## DecorView

## ViewRootImpl

在 `ActivityThread.handleResumeActivity` 第一次调用时, 在`Activity.onResume`之后的 `WindowManager.addView` -> `WindowManagerGlobal.addView` 中创建了 `ViewRootImpl`, 之后调用了 `ViewRootImpl.setView` 设置根布局, 并且同时调用`Choreographer.postCallback`发送了一个`TraversalRunnable`, 申请了VSync中断, 在下一个VSync信号来的时候执行了 `ViewRootImpl.doTraversal` -> `ViewRootImpl.performTraversals` 开始绘制流程.其中, 在第一次绘制时, 会调用`ViewGrou.dispatchAttachedToWindow`, 将所有子View 的 `View.mAttatchInfo` 赋值, 将之前所有用 `View.post` 加入的 `Runnable` 添加到 `MessageQueue`. 然后会调用 `ViewRootImpl.performMeasure`, 然后就是 `View.measure` 的逐级调用测量大小. 然后逐级调用 `View.layout` 放置控件, 逐级调用 `View.draw` 绘制控件

在调用 `ActivityThread.handleDestroyActivity` 会执行 `WindowManager.removeViewImmediate` -> `WindowManagerGlobal.removeView` -> `ViewRootImpl.doDie` -> `View.onDetachedFromWindow`

在 onCreate, onStart, onResume 拿不到宽高的原因: 这3个生命周期是在同一个事件序列里执行的(`TransactionExecutor.cicleToPath`), 所以 View的绘制流程在这之后执行, 所以这几个周期里拿不到宽高

关于测量模式和match_parent，wrap_content: [自定义控件测量模式真的和 match_parent，wrap_content 一一对应吗？](https://www.wanandroid.com/wenda/show/12489) 不对应, 例如 父为wrap_content, 子为wrap_content/match_parent, 最终都是AT_MOST, 与 `ViewGroup.getChildMeasureSpec` 方法相关

阅读资料: [理解VSync](https://www.jianshu.com/p/75139692b8e6)

## Window

`WindowManagerGlobal`(单例), 所有和 WMS 交流都通过该类

`WindowManagerImpl`(非单例) 是一个包装类, 把所有操作都交给了 `WindowManagerGlobal`

在第一个Activity要被启动的时候会调用`WindowManagerGlobal.initialize()`, 初始化 `WindowManagerGlobal`
`Window`是一个抽象类, 唯一的子类是`PhoneWindow`, 在 `Instrumentation.newActivity`创建Activity对象后 -> `Activity.attach`, 这时候创建了一个`PhoneWindow`对象, 并且通过 `WindowManagerImpl.createLocalWindowManager` 创建了一个 `WindowManagerImpl` 对象, 
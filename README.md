# Android-Learning-Notes

记录Android开发过程中遇到的问题

___

## 开发遇到的问题

1. 机型兼容
    1. Samsung SM-G9550 Android 9, API 28   TextView设置 `gravity = "center"` && `layout_width = "wrap_content"` 时, 画布默认向右移动width/2, 导致自定义View绘制出错

2. RadioGroup
    - [x] `RadioGroup#check()` 会导致 `OnCheckedChangeListener#onCheckedChanged()`被调用2次

3. View
    - [x] selector的资源文件的`android:constantSize`属性
    - [x] `View#post(Runnable)`中`Runnable`的调用时机
    - [x] `super#onRestoreInstanceState(Parcelable)` 参数传入`BaseSavedState`的子类时, 在Api <= 22时崩溃

4. Fragment
    - [x] `Fragment#onActivityCreated(Bundle)`, 除非在`Fragment#onSaveInstanceState(Bundle)`中存入了内容, 否则`Bundle`一直为null

## 源码

- [x] EventBus
- [x] Retrofit
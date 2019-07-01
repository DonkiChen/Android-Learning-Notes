# RadioGroup
_Version:Api 28_
___

## 动态添加的RadioButton的id

`RadioGroup.PassThroughHierarchyChangeListener#onChildViewAdded(View, View)`如果加入的子View是`RadioGroup`类型的, 如果没有id, 则会调用`View#generateViewId()`生成id并赋值

```java
    if (parent == RadioGroup.this && child instanceof RadioButton) {
        int id = child.getId();
        // generates an id if it's missing
        if (id == View.NO_ID) {
            id = View.generateViewId();
            child.setId(id);
        }
    }
```

## `RadioGroup#check(int)`导致`OnCheckedChangeListener#onCheckedChanged()`被调用2次

_`mOnCheckedChangeListener#onCheckedChanged`被调用处_
```java
    private void setCheckedId(@IdRes int id) {
        //...
        if (mOnCheckedChangeListener != null) {
            mOnCheckedChangeListener.onCheckedChanged(this, mCheckedId);
        }
    }
```

### 两次的调用方法栈:

1. `RadioGroup#check(int)` -> `RadioGroup#setCheckedStateForView(int, boolean)` -> `RadioButon#setCheck(boolean)` -> `RadioGroup.CheckedStateTracker#onCheckedChanged(CompoundButton, boolean)` -> `RadioGroup#setCheckedId(int)` -> `OnCheckedChangeListener#onCheckedChanged()`

2. `RadioGroup#check(int)` -> `RadioGroup#setCheckedId(int)` -> `OnCheckedChangeListener#onCheckedChanged()`

### 主要原因:

1. `RadioGroup.PassThroughHierarchyChangeListener#onChildViewAdded(View, View)`中, 对加入的每个`RadioButton`都添加了一个`RadioGroup.CheckedStateTracker`, 其中的`onCheckedChanged(CompoundButton, boolean)`调用了`setCheckedId(int)`

```java
    @Override
    public void onChildViewAdded(View parent, View child) {
        if (parent == RadioGroup.this && child instanceof RadioButton) {
            //...
            //mChildOnCheckedChangeListener = new CheckedStateTracker();
            ((RadioButton) child).setOnCheckedChangeWidgetListener(
                    mChildOnCheckedChangeListener);
        }
    }

    private class CheckedStateTracker implements CompoundButton.OnCheckedChangeListener {
        @Override
        public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
            //...
            setCheckedId(id);
        }
    }

```

2. 调用`RadioGroup#check(int)`时, 调用`RadioGroup#setCheckedStateForView`对子`RadioButton`的选中状态修改, 调用了`onCheckedChanged(CompoundButton, boolean)` 从而最终调用`OnCheckedChangeListener#onCheckedChanged()`

```java
    public void check(@IdRes int id) {
        //...
        if (mCheckedId != -1) {
            setCheckedStateForView(mCheckedId, false);
        }
        if (id != -1) {
            setCheckedStateForView(id, true);
        }
    }
```

3. 调用`RadioGroup#check(int)`时, 有一次对`OnCheckedChangeListener#onCheckedChanged()`的直接调用

```java
    public void check(@IdRes int id) {
        //...
        setCheckedId(id);
    }
```


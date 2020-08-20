# Fragment
_Version:Api 28, support-fragment: 28.0.0_
___

## 除非在`Fragment#onSaveInstanceState(Bundle)`中存入了内容, 否则重建后`Fragment#onActivityCreated(Bundle)`的`Bundle`一直为null

`FragmentManagerImpl#saveFragmentBasicState`中, 先new出了一个`Bundle`, 然后调用`Fragment#performSaveInstanceState`,`FragmentManagerImpl#dispatchOnFragmentSaveInstanceState`存入数据, 如果没有存入任何内容, 则将其置为null, 所以导致重建后`Fragment#onActivityCreated(Bundle)`的`Bundle`一直为null

```java
    Bundle saveFragmentBasicState(Fragment f) {
        Bundle result = null;
        if (this.mStateBundle == null) {
            this.mStateBundle = new Bundle();
        }

        f.performSaveInstanceState(this.mStateBundle);
        this.dispatchOnFragmentSaveInstanceState(f, this.mStateBundle, false);
        if (!this.mStateBundle.isEmpty()) {
            result = this.mStateBundle;
            this.mStateBundle = null;
        }
        //...
    }
```
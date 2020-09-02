# LeakCanary 笔记
LeakCanary version:2.4
___

## 不需要调用任何代码就可以运行的原因
`AppWatcherInstaller` 是 `ContentProvider` 的子类, 在 `ContentProvider.onCreate()` 中做了初始化, 调用了 `AppWatcher.manualInstall(application)`, 可以通过重写 `leak_canary_watcher_auto_install` 这个布尔资源来决定是否自动初始化

## 检测Activity的泄露
通过调用`Application.registerActivityLifecycleCallbacks` 在 `onActivityDestroyed` 回调里检测 `Activity`

## 检测Fragment的泄露和 Fragment View 的泄露
检测Fragment泄露有三个类
1. `AndroidOFragmentDestroyWatcher` (检测系统自带Fragment)
2. `AndroidXFragmentDestroyWatcher` (检测androidX的Fragment)
3. `AndroidSupportFragmentDestroyWatcher` (检测support包的Fragment)

其原理都是在 `onFragmentViewDestroyed` 回调里检测 `View`, 在 `onFragmentDestroyed` 回调里检测 `Fragment`,  区别如下:
1. (系统自带Fragment)在 > O的版本
2. (support包的Fragment)没有别的操作
3. (androidX的Fragment)额外加上了 `ViewModel` 泄露的检测初始化, `ViewModelClearedWatcher`
通过 `Application.registerActivityLifecycleCallbacks`的 `onActivityCreated` 回调, 给每一个 `Activity` 调用`FragmentManager.registerFragmentLifecycleCallbacks` 注册以上几个检测类(通过了 `Class.forName` 判断androidX 与 support包是否存在, 避免不必要的注册)

## 检测ViewModel的泄露
如果app依赖了androidX的Fragment, 在 `AndroidXFragmentDestroyWatcher` 里可以看到, 在`ActivityLifecycleCallbacks.onActivityCreated` 或 `FragmentLifecycleCallbacks.onFragmentCreated` 的时候会调用 `ViewModelClearedWatcher.install`

如何检测ViewModel泄露: `ViewModelClearedWatcher` 其实也是 `ViewModel` 的子类, 在回调 `ViewModel.onClear` 的时候(即 `FragmentActivity` 或 `Fragment` 回调 `onDestroy` 时), 拿到 `ViewModelStoreOwner` 中所有注册了的 `ViewModel` , 然后加入检测队列


## InternalLeakCanary

## 如何检测泄露 
通过 `ObjectWatcher.watch` 生成一个包含UUID作为key的弱引用(`KeyedWeakReference`)的对象, 放在map中(key = key, value = 弱引用对象), 然后过5秒把map中被回收的对象删除(创建弱引用时可以传入一个 `ReferenceQueue` , 如果弱引用引用的对象被回收了, 弱引用会被添加到这个队列中, 再根据该对象的key值, 从map中删除), 再主动GC一次, map中剩下的就是泄露的对象. 然后通过 `Debug.dumpHprofData` 生成.hprof文件, 通过Shark解析该文件拿到从GC Root到泄露对象的引用链, 即可知道是什么导致了内存泄漏


## 动态代理的妙用
实现某个有很多个函数的接口但是只需要其中不多的方法时, 可以使用动态代理
```kotlin
inline fun <reified T : Any> noOpDelegate(): T {
  val javaClass = T::class.java
  val noOpHandler = InvocationHandler { _, _, _ ->
    // no op
  }
  return Proxy.newProxyInstance(
      javaClass.classLoader, arrayOf(javaClass), noOpHandler
  ) as T
}

object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
  override fun onActivityDestroyed(activity: Activity) {
    //...
  }
}

```

## 为什么用弱引用而不用其他的引用
强引用肯定是排除的; 软引用只在即将发生OOM的时候才会进行回收, 也是不可以的; 虚引用没有持有对象的引用, 所以没法分析是哪个GC Root持有了引用
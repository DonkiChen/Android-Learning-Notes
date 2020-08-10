# Glide
_version: 4.11.0_
___

## Glide.with()

```Java
  @NonNull
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }

  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    //get() 获取了单例Glide对象
    //getRequestManagerRetriever() 返回了一个 RequestManagerRetriever, 这个类可以通过 RequestManagerFactory.build 生成 RequestManager 对象
    //RequestManagerRetriever 用来生成&管理 RequestManager 对象
    return Glide.get(context).getRequestManagerRetriever();
  }

```

## RequestManagerRetriever

### 如何管理 RequestManager

#### track()

```Java
  synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```

#### get()

Activity, Fragment获取当前的FragmentManager, 生成 SupportRequestManagerFragment 或 RequestManagerFragment; View也类似, 不过是先去获取所在的Fragment或Activity. RequestManagerFragment可以感知到ImageView的生命周期, 在销毁时自动取消加载

```Java
  @SuppressWarnings("deprecation")
  @NonNull
  public RequestManager get(@NonNull View view) {
    //如果是在子线程, 用Application生命周期的RequestManager
    if (Util.isOnBackgroundThread()) {
      return get(view.getContext().getApplicationContext());
    }

    //寻找View所属的Activity
    Activity activity = findActivity(view.getContext());
    // The view might be somewhere else, like a service.
    if (activity == null) {
      return get(view.getContext().getApplicationContext());
    }

    // Support Fragments.
    // Although the user might have non-support Fragments attached to FragmentActivity, searching
    // for non-support Fragments is so expensive pre O and that should be rare enough that we
    // prefer to just fall back to the Activity directly.
    if (activity instanceof FragmentActivity) {
      //androidx.fragment包
      //如果Activity是FragmentActivity, 则去寻找View是在哪个Fragment中 或者 Activity中
      Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
      //如果在fragment中, 用fragment的RequsetManager, 如果在activity中, 则用activity的RequestManager
      return fragment != null ? get(fragment) : get((FragmentActivity) activity);
    }

    // Standard Fragments.
    //同上, 不过这个是 android.app 包, 即系统自带Fragment
    android.app.Fragment fragment = findFragment(view, activity);
    if (fragment == null) {
      return get(activity);
    }
    return get(fragment);
  }

  @Nullable
  private Fragment findSupportFragment(@NonNull View target, @NonNull FragmentActivity activity) {
    //private final ArrayMap<View, Fragment> tempViewToSupportFragment = new ArrayMap<>();
    tempViewToSupportFragment.clear();
    //Fragment的根View作为key, fragment作为value
    findAllSupportFragmentsWithViews(
        activity.getSupportFragmentManager().getFragments(), tempViewToSupportFragment);
    Fragment result = null;
    View activityRoot = activity.findViewById(android.R.id.content);
    View current = target;
    //寻找target在哪里
    while (!current.equals(activityRoot)) {
      //如果在fragments里找到了, 返回fragment
      result = tempViewToSupportFragment.get(current);
      if (result != null) {
        break;
      }
      //没找到, 找current的父控件
      if (current.getParent() instanceof View) {
        current = (View) current.getParent();
      } else {
        break;
      }
    }

    tempViewToSupportFragment.clear();
    return result;
  }

  @NonNull
  public RequestManager get(@NonNull Fragment fragment) {
    if (Util.isOnBackgroundThread()) {
      return get(fragment.getContext().getApplicationContext());
    } else {
      //拿到Fragment的FragmentManager
      FragmentManager fm = fragment.getChildFragmentManager();
      return supportFragmentGet(fragment.getContext(), fm, fragment, fragment.isVisible());
    }
  }

  @SuppressWarnings("deprecation")
  @NonNull
  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      //拿到Activity的FragmentManager
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }

  //supportFragmentGet 与 fragmentGet类似, 只是包名不同
  @NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    //寻找当前FragmentManager的SupportRequestManagerFragment
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      Glide glide = Glide.get(context);
      //生成一个RequestManager
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
  
  @NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      //已经加进去的Fragment如果没有, 去查找即将加入的
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        //如果即将加入的也没有, 那就new一个
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        //放到pending里
        pendingSupportRequestManagerFragments.put(fm, current);
        //不立即添加
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        //添加后从pending中删除
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }

```

### RequestManager

#### load()

会调用 `asDrawable()` -> `as()` 生成新的 `RequestBuilder`

#### into()

##### SingleRequest

1. `begin()`

  ```Java
  @Override
  public void begin() {
    synchronized (requestLock) {
      // 要加载的内容未空, 报错
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }

      // TODO 如果View重新发起同一个已完成的请求, 直接用内存中的缓存 
      if (status == Status.COMPLETE) {
        onResourceReady(resource, DataSource.MEMORY_CACHE);
        return;
      }

      // 对于未完成且未运行的请求, 重新请求
      status = Status.WAITING_FOR_SIZE;
      //如果overrideWidth(Height)合法(> 0 || == Target.SIZE_ORIGINAL)
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        //不合法就得先去获取大小
        target.getSize(this);
      }

      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
    }
  }

  // ViewTarget.SizeDeterminer 获取View尺寸的代码
  void getSize(@NonNull SizeReadyCallback cb) {
    int currentWidth = getTargetWidth();
    int currentHeight = getTargetHeight();
    if (isViewStateAndSizeValid(currentWidth, currentHeight)) {
      cb.onSizeReady(currentWidth, currentHeight);
      return;
    }
    // We want to notify callbacks in the order they were added and we only expect one or two
    // callbacks to be added a time, so a List is a reasonable choice.
    if (!cbs.contains(cb)) {
      cbs.add(cb);
    }
    if (layoutListener == null) {
      // 通过ViewTreeObserver的onPreDraw()回调去拿尺寸, 然后回调SizeReadyCallback
      ViewTreeObserver observer = view.getViewTreeObserver();
      layoutListener = new SizeDeterminerLayoutListener(this);
      observer.addOnPreDrawListener(layoutListener);
    }
  }

  //拿到View尺寸的回调
  @Override
  public void onSizeReady(int width, int height) {
    synchronized (requestLock) {
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      status = Status.RUNNING;
      // 分辨率缩放
      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

      loadStatus = engine.load(/**/);

      // This is a hack that's only useful for testing right now where loads complete synchronously
      // even though under any executor running on any thread but the main thread, the load would
      // have completed asynchronously.
      if (status != Status.RUNNING) {
        loadStatus = null;
      }
    }
  }

  //Engine
  public <R> LoadStatus load(/**/) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    //生成key
    EngineKey key = keyFactory.buildKey(/**/);

    EngineResource<?> memoryResource;
    synchronized (this) {
      //根据key去 当前在使用的资源 -> 内存缓存中 查找有没有缓存
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        //内存中没有缓存
        return waitForExistingOrStartNewJob(/**/);
      }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
  }

  //从内存中获取缓存
  @Nullable
  private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }
    //如果当前资源正在使用, 直接返回
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      return active;
    }
    //如果在内存缓存中找到了, 将他设置为正在使用, 然后返回
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      return cached;
    }

    return null;
  }

  private <R> LoadStatus waitForExistingOrStartNewJob(/**/) {
    //查找当前是否有正在进行的请求
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      //有的话直接加个回调
      current.addCallback(cb, callbackExecutor);
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob = engineJobFactory.build(/**/);
    // 获取图片: 1.资源缓存, 之前被解码过, 转换过的图片的磁盘缓存 2. 磁盘源文件缓存 3. 没有缓存, 根据传入的类型去加载
    DecodeJob<R> decodeJob = decodeJobFactory.build(/**/);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);

    return new LoadStatus(cb, engineJob);
  }
  ```

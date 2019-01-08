# EventBus 笔记

_EventBus version: 3.1.1_
_暂称具有`@Subscribe`注解的类为观察者，称被发送的事件为被观察者)_

---

## 调用`register(object)`后发生了什么

1. 通过`findSubscriberMethods`寻找被注解的方法

    ```java
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        // 寻找被注解的方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                // 产生观察关系
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
    ```
1. `findSubscriberMethods` 寻找，寻找被注解方法的方式：

    1. 反射
    1. 注解生成器生成的文件

    ```java
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        // 是否忽略注解生成器生成的文件
        if (ignoreGeneratedIndex) {
            // 通过反射
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 通过注解生成器生成的文件
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
    ```

1. `subscribe`中，产生观察关系，根据`priority`排序观察者，处理`sticky`事件

    ```java
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                // 将观察者根据 priority 插入
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                // 分发 stikcy 事件
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
    ```

---

## 调用 `post(event)` 之后发生了什么

1. 获取了当前线程的 `PostingThreadState` 对象，将 `event` 加入 `eventQueue` 尾部，循环分发 `postSingleEvent` 直到事件列表为空

    ```java
    public void post(Object event) {
        // 获取当前线程的 PostingThreadState 对象，线程安全的一种实现
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        // 将要分发的事件加入尾部
        eventQueue.add(event);

        if (!postingState.isPosting) {
            // 当前线程是否是主线程
            postingState.isMainThread = isMainThread();
            // 是否正在分发
            postingState.isPosting = true;
            // 分发是否被取消
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
    ```

1. `postSingleEvent` 中，根据 `eventInheritance` 判断是否分发 `event` 的基类和接口，调用 `postSingleEventForEventType` 进行真正的分发

    ```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        // 是否同时分发该类的基类及接口
        if (eventInheritance) {
            // 拿到该类的所有基类及接口
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
    ```

1. `postSingleEventForEventType` 中，根据 `event` 类获取注册了该类的对象，一一进行分发

    ```java
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 获取所有注册了该类的对象
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                // 当前分发的事件
                postingState.event = event;
                // 当前接受者的信息
                postingState.subscription = subscription;
                // 是否被中断
                boolean aborted = false;
                try {
                    // 决定分发的线程
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
    ```

1. `postToSubscription`根据分发的线程，使用不同的 `poster`, [下有解析](#ThreadMode)

    ```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
    ```
1. `invokeSubscriber` 调用被 `@Subscribe` 注解的方法

    ```java
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
    ```

## ThreadMode

1. `POSTING` 默认模式，在调用`post`的线程发放，效率最高，避免进行耗时操作（因为可能由 UI 线程调用)。

1. `MAIN` 主线程，`HandlerPoster` 继承自 `Handler`, 若在主线程 `post` , 则直接分发，否则入队
    ```java
    @Override
    public void enqueue(Subscription subscription, Object event) {
        //PendingPost 中维护了一个回收池，降低对象创建消耗
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //queue 消息队列
            queue.enqueue(pendingPost);
            //handlerActive, 是否正在分发事件
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    // 执行 handleMessage()
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
    @Override
    public void handleMessage(Message msg) {
        // 一次分发事件是否超时，重新排期
        boolean rescheduled = false;
        try {
            // 开始时间
            long started = SystemClock.uptimeMillis();
            while (true) {
                // 类似于双重检查锁的单例，提高性能
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                // 分发
                eventBus.invokeSubscriber(pendingPost);
                // 方法总共用时
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    // 总共用时 >= 10ms, 退出此次分发，减少卡顿
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
    ```

1. `MAIN_ORDERED` 主线程（排到队尾), 类似于 `MAIN`, 优先入队
    ```java
    if (mainThreadPoster != null) {
        // 加入队尾
        mainThreadPoster.enqueue(subscription, event);
    } else {
        invokeSubscriber(subscription, event);
    }
    ```
1. `BACKGROUND` 后台线程，`BackgroundPoster` 实现了 `Runnable`, 只使用了线程池中的一个线程
    ```java
    @Override
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                // 只使用了 1 个线程来分发事件
                executorRunning = true;
                // 默认的线程池是 Executors.newCachedThreadPool()
                eventBus.getExecutorService().execute(this);
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    // 只使用了 1 个线程来分发事件
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }
    ```

1. `ASYNC` 异步，`AsyncPoster`, 直接线程池中执行
    ```java
    @Override
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        // 默认的线程池是 Executors.newCachedThreadPool()
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if (pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }
    ```
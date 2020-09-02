# 启动流程
_Android Api 28_
___

## Application
AMS调用`startProcessLocked`去和`Zygote`(ˈzīˌɡōt)通信, 启动`ActivityThread`, `ActivityThread`再去调AMS的`attachApplication`, AMS去调用 `ActivityThread.bindApplication` 配置app信息, 检查是否有`Activity`需要启动(`ActivityStackSupervisor.attachApplicationLocked` -> `ActivityStackSupervisor.realStartActivityLocked`), 是否有`Service`需要启动(`ActiveServices.attachApplicationLocked` -> `ActiveServices.realStartServiceLocked`), 是否有广播需要发送(`BroadcastQueue.sendPendingBroadcastsLocked`), 然后`ActivityThread`发送一个`BIND_APPLICATION`到`ActivityThread.H`即`Handler`, 然后Looper开始循环 -> `handleBindApplication` 其中new了Application, 调用了`Application.attachBaseContext`, 初始化了`ContentProvider`, 调用了`Application.onCreate` 

## Activity

app A 调用 `startActivity`, 到了 `Instrumentation.execStartActivity`, 然后走到了AMS, AMS 获取到 `ActivityStarter` 判断了 `Intent` 能否被处理, 处理flags, new出栈 或者 重用栈, 然后 `ActivityStack` 把原先栈顶的Activity pause掉(通过Binder调app A的方法`scheduleTransaction`, 传了一个 `PauseActivityItem`过去, pause 掉Activity)
1. 如果目标app没有启动, 先调用AMS的`startProcessLock`启动进程, 在Application调用AMS `attachApplicationLock` 的时候(这里attach是绑定了 `ActivityThread` 代理类 和 `ProcessRecord`), 让`ActivityStackSupervisor`判断当前栈顶的`ActivityRecord`是不是在等待该app的启动, 如果是就调用 `ActivityStackSupervisor.realStartActivityLocked` 启动Activity
2. 如果app启动了, 调用 `ActivityStackSupervisor.realStartActivityLocked` 启动Activity, 获取到`ClientTransaction`, 添加callback `LaunchActivityItem`, 根据是否需要onResume获取`ResumeActivityItem` 或者`PauseActivityItem`, 设置为`ClientTransaction`的最终生命周期状态, 然后调用`ActivityThread.scheduleTransaction`, `LaunchActivityItem.execute` 中 调用了 `ActivityThread.handleLaunchActivity` -> `ActivityThread.performLaunchActivity`, 会从 `Instrumentation.newActivity` 生成Activity对象
3. 如果AMS执行过程中发现无法启动该Activity,因为 `AMS.startActivity`有一个返回值, 会返回给 `Instrumentation.execStartActivity` 一个code, 根据code `Instrumentation`会报相关的错,

`ActivityStackSupervisor.startSpecificActivityLocked` 启动特定 Activity

### 生命周期

`TransactionExecutor`


## Service

### 通用启动流程
调用`ContextImpl.startService`, 判断 Target API >= L 时 不能隐式启动Service, 然后就调用AMS的`startService`, `ActiveServices.startServiceLocked`, 检查各种权限后 调用了`bringUpServiceLocked`, 再判断目标app有没有启动
1. 没启动则先启动进程, 把 `ServiceRecord` 存到 `ActiveServices.mPendingServices` 中, 等到app启动后, 调用AMS的 `attachApplication`时, 再去调用`ActiveServices.realStartServiceLocked`
2. 启动了直接调用 `ActiveServices.realStartServiceLocked`
里面调用了`ActivityThread.scheduleCreateService`, 之后就到了`ActivityThread`, 调用了`ActivityThread.handleCreateService` -> `Service.onCreate` 创建并启动了Service, 如果是前台服务则发送通知

### `startService`
调用了 `ActivityThread.handleServiceArgs` -> `Service.onStartCommand` 传递了参数

### `bindService`
`ActiveServices.realStartServiceLocked` 会调用 `ActiveServices.requestServiceBindingLocked`, 去绑定, 然后会调用 `ActivityThread.scheduleBindService` -> `ActivityThread.handleBindService` -> (`Service.onBind`, `AMS.publishService`) -> `ActiveServices.publishServiceLocked` -> `LoadedApk.ServiceDispatcher.doConnected` -> `ServiceConnection.onServiceConnected`

#### `ServiceDispatcher`

### 在源码中发现了`ActiveServices`返回了 !, !!, ?. 这几种情况的启动失败
1. !: 由于权限不足导致无法启动Service
    1. 启动其他app的 非exported的Service
    2. 没有启动该服务所需要的权限(manifest.xml中<service>节点的"permission"属性)
2. !!: 启动app失败
3. ?: app在后台, 无法启动


## BroadcastReceiver

### `ReceiverDispatcher`

### 本地广播

#### 注册

#### 发送
`BroadcastQueue`

##### 普通广播

##### 有序广播

### 全局广播

#### 注册

#### 发送

## ContentProvider
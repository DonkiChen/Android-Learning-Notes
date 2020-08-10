# okhttp3 笔记
_okhttp version:4.7.2_
___

## 几个重要的类
1. `Dispatcher`: 维护请求队列
2. `ExchangeFinder`: 寻找连接和Http编解码器


## `Call.execute()` 执行过程
1. `execute()`
    ```Kotlin
      override fun execute(): Response {
        synchronized(this) {
          check(!executed) { "Already Executed" }
          executed = true
        }
        //计时, 根据[OkhttpClient.callTimeoutMillis]设置, 超时自动取消 [RealCall.cancel]
        timeout.enter()
        callStart()
        try {
          //添加到[Dispatcher.runningSyncCalls]队列中
          client.dispatcher.executed(this)
          //发起请求
          return getResponseWithInterceptorChain()
        } finally {
          //从[Dispatcher.runningSyncCalls]中移除
          client.dispatcher.finished(this)
        }
      }
    ```
2. `getResponseWithInterceptorChain()`
    ```Kotlin
      @Throws(IOException::class)
      internal fun getResponseWithInterceptorChain(): Response {
        // Build a full stack of interceptors.
        val interceptors = mutableListOf<Interceptor>()
        //添加应用拦截器
        interceptors += client.interceptors
        //添加内置拦截器
        //失败重试和重定向
        interceptors += RetryAndFollowUpInterceptor(client)
        //添加默认的请求头, 设置自定义cookie
        interceptors += BridgeInterceptor(client.cookieJar)
        //缓存拦截器
        interceptors += CacheInterceptor(client.cache)
        //打开连接
        interceptors += ConnectInterceptor
        if (!forWebSocket) {
          //如果不是WebSocket的话, 添加网络拦截器
          interceptors += client.networkInterceptors
        }
        //真正发送请求 以及返回响应
        interceptors += CallServerInterceptor(forWebSocket)
        //...
        val chain = RealInterceptorChain(/*..*/)
    
        var calledNoMoreExchanges = false
        try {
          //依次执行拦截器
          val response = chain.proceed(originalRequest)
          if (isCanceled()) {
            response.closeQuietly()
            throw IOException("Canceled")
          }
          return response
        } catch (e: IOException) {
          calledNoMoreExchanges = true
          //释放连接
          throw noMoreExchanges(e) as Throwable
        } finally {
          if (!calledNoMoreExchanges) {
            //释放连接
            noMoreExchanges(null)
          }
        }
      }
    
    ```
3. `RetryAndFollowUpInterceptor` 管理失败重试和重定向, 其中做了死循环, 直到成功或无法重试
    1. `recover()` 判断能否重试(抛异常情况下)
        ```Kotlin
          private fun recover(
            e: IOException,
            call: RealCall,
            userRequest: Request,
            requestSendStarted: Boolean
          ): Boolean {
            // The application layer has forbidden retries.
            // 是否开启了失败重试
            if (!client.retryOnConnectionFailure) return false

            // We can't send the request body again.
            // 如果请求已经发出, 并且请求是一次性的, 不去重试, 默认[RequestBody.isOneShot]返回false
            // requestIsOneShot = e is FileNotFoundException || RequestBody.isOneShot()
            if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

            // This exception is fatal.
            // 是否是致命的错误, 以下几种是致命的
            // 1. ProtocolException
            // 2. SocketTimeoutException && !requestSendStarted
            // 3. SSLHandshakeException && e.cause is CertificateException
            // 4. SSLPeerUnverifiedException
            if (!isRecoverable(e, requestSendStarted)) return false

            // No more routes to attempt.
            if (!call.retryAfterFailure()) return false

            // For failure recovery, use the same route selector with a new connection.
            return true
          }
        ```
    2. `followUpRequest()` 判断能否重新发起请求, Http Code相关, 不能超过20次
        1. 请求在重用的连接上发起, 该连接被服务器关闭
        2. 客户端超时 Http Code = 408
        3. 需要授权质询 Http Code = 401, 407
        4. 可重试的服务器故障, Http Code = 503 且响应头里带有"Retry-After"
        5. 合并连接上的请求错误, Http Code = 421
        6. 重定向 Http Code = 300, 301, 302, 303, 307, 308

4. `BridgeInterceptor` 添加默认的请求头, 可以设置自定义cookie(默认为空), gzip压缩
5. `CacheInterceptor` 缓存
6. `ConnectInterceptor` 初始化 `Exchange`, 这一个拦截器里做了连接池的复用, dns的解析, 建立Socket, TSL握手, 建立了和服务器的连接
7. `CallServerInterceptor` 真正发送请求 以及返回响应


## `Call.execute()` 与 `Call.enqueue()` 有什么区别
`Call.enqueue()`会生成一个`AsyncCall`放到`Dispatcher.runningAsyncCalls`中, 然后在
1. `Dispatcher.maxRequests` 并发最大请求数量改变时
2. `Dispatcher.maxRequestsPerHost` 同个域名最大并发请求数量改变时
3. `Dispatcher.enqueue()` 有新的异步请求入队时
4. `Dispatcher.finished()` 有请求结束时

这几种情况下会调用`Dispatcher.promoteAndExecute()`判断能否执行异步请求, 能否执行取决于以下两点
1. `runningAsyncCalls.size < this.maxRequests` 正在并发运行的请求数量是否小于限制
2. `asyncCall.callsPerHost.get() < this.maxRequestsPerHost` 当前域名正在并发运行的最大请求数量是否小于限制

如果能够执行, 就调用 `AsyncCall.executeOn`在指定的线程池上执行

这里就有一个需要注意的地方, `Call.execute()`不会判断并发请求数量, 即使你开多个线程同时执行多个`Call.execute()`超过请求数量限制, okhttp也不会阻止你

## 应用拦截器 与 网络拦截器的区别
位于的层级不一样, 应用拦截器处于第一个拦截器, 网络拦截器处于最后一个拦截器, 由于层级的不同所以会有以下的区别

1. 应用拦截器拿到的请求是最原始的请求, 拿到的响应是最终其他拦截器处理过的响应; 网络拦截器拿到的是最终的请求, 与服务器拿到的请求一致, 拿到的响应是最原始的
2. 如果本地有缓存且未失效的情况下, 只有应用拦截器会拿到请求和响应, 因为在`CacheInterceptor`就已经返回了缓存着的响应, 不会再走之后的拦截器
3. 如果遇到失败重试或者重定向等的情况, 应用拦截器只会记录一次请求和响应, 而网络拦截器每次都会记录

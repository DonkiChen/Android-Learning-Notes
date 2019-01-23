# Retrofit 笔记

_Retrofit version:2.5.0_<br/>
_需要用到[Type相关知识](../../Java知识点/反射相关/Type.md)_

---

## 调用`create(class)`后发生了什么

_由于Retrofit有多个类与okhttp3类名相同,okhttp3的类会加上包名.如:`Call<?>`为Retrofit中的类, `okhttp3.Call`为okhttp3中的类_

1. `create(class)`,创建动态代理对象
    ```java
    public <T> T create(final Class<T> service) {
        //验证这个类是接口并且没有继承其他接口
        Utils.validateServiceInterface(service);
        //是否立即加载所有方法
        if (validateEagerly) {
            //加载所有方法
            eagerlyValidateMethods(service);
        }
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
                new InvocationHandler() {
                    //根据特定的类是否存在,获取当前平台
                    //android.os.Build -> Android
                    //java.util.Optional -> Java8
                    private final Platform platform = Platform.get();
                    private final Object[] emptyArgs = new Object[0];

                    @Override
                    public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                            throws Throwable {
                        if (method.getDeclaringClass() == Object.class) {
                            //直接调用定义自Object的方法
                            return method.invoke(this, args);
                        }
                        if (platform.isDefaultMethod(method)) {
                            //Java8的新特性:接口中的default方法
                            //调用该方法
                            //Java8平台中使用了MethodHandles实现
                            //其他平台抛异常:new UnsupportedOperationException()
                            return platform.invokeDefaultMethod(method, service, proxy, args);
                        }
                        //解析注解,调用
                        return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
                    }
                });
    }
    ```

1. `loadServiceMethod` 从缓存中获取`ServiceMethod`对象或重新解析生成
    ```java
    //Retrofit.java
    //缓存
    ServiceMethod<?> loadServiceMethod(Method method) {
        ServiceMethod<?> result = serviceMethodCache.get(method);
        if (result != null) return result;

        synchronized (serviceMethodCache) {
            result = serviceMethodCache.get(method);
            if (result == null) {
                result = ServiceMethod.parseAnnotations(this, method);
                serviceMethodCache.put(method, result);
            }
        }
        return result;
    }

    ```

1. `ServiceMethod.parseAnnotations` 初步过滤返回类型
    ```java
    //ServiceMethod.java
    //返回类型的初步判断
    static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
        //存储请求信息的一个类
        RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

        Type returnType = method.getGenericReturnType();
        //是否包含不支持的返回类型(类型变量,通配符)
        if (Utils.hasUnresolvableType(returnType)) {
            throw methodError(method,
                    "Method return type must not include a type variable or wildcard: %s", returnType);
        }
        //返回类型为空也不支持
        if (returnType == void.class) {
            throw methodError(method, "Service methods cannot return void.");
        }
        //进一步解析
        return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
    }
    ```

1. `RequestFactory.parseAnnotations()` 存储信息,解析注解,检查[请求方式与参数是否符合标准](##关于请求方式)

    ```java
    //RequestFactory
    static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
        return new Builder(retrofit, method).build();
    }

    //RequestFactory.Builder
    Builder(Retrofit retrofit, Method method) {
        this.retrofit = retrofit;
        this.method = method;
        this.methodAnnotations = method.getAnnotations();
        this.parameterTypes = method.getGenericParameterTypes();
        this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

    //RequestFactory.Builder
    RequestFactory build() {
        for (Annotation annotation : methodAnnotations) {
            //解析请求方式注解
            parseMethodAnnotation(annotation);
        }

        if (httpMethod == null) {
            throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
        }

        if (!hasBody) {
            if (isMultipart) {
                throw methodError(method,
                        "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
            }
            if (isFormEncoded) {
                throw methodError(method, "FormUrlEncoded can only be specified on HTTP methods with "
                        + "request body (e.g., @POST).");
            }
        }

        int parameterCount = parameterAnnotationsArray.length;
        parameterHandlers = new ParameterHandler<?>[parameterCount];
        for (int p = 0; p < parameterCount; p++) {
            //ParameterHandler 处理参数的一个基类
            parameterHandlers[p] = parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p]);
        }

        if (relativeUrl == null && !gotUrl) {
            throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
        }
        if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
            throw methodError(method, "Non-body HTTP method cannot contain @Body.");
        }
        if (isFormEncoded && !gotField) {
            throw methodError(method, "Form-encoded method must contain at least one @Field.");
        }
        if (isMultipart && !gotPart) {
            throw methodError(method, "Multipart method must contain at least one @Part.");
        }

        return new RequestFactory(this);
    }

    ```

1. `HttpServiceMethod.parseAnnotations` 进一步解析注解, 生成`HttpServiceMethod`对象
    ```java
    //HttpServiceMethod.java
    //解析注解,生成HttpServiceMethod对象
    static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
            Retrofit retrofit, Method method, RequestFactory requestFactory) {
        //将Call<?>转换为其他类型,[具体过程](##关于CallAdapter)
        CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
        //获取Call<?>中的泛型类型,如果method返回类型不是泛型,则会抛异常
        Type responseType = callAdapter.responseType();
        if (responseType == Response.class || responseType == okhttp3.Response.class) {
            throw methodError(method, "'"
                    + Utils.getRawType(responseType).getName()
                    + "' is not a valid response body type. Did you mean ResponseBody?");
        }
        //使用@HEAD时,返回值必须为Void,[详情](##关于请求方式)
        if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
            throw methodError(method, "HEAD method must use Void as response type.");
        }
        //将返回值解析成java对象,[具体过程](##关于Converter)
        Converter<ResponseBody, ResponseT> responseConverter =
                createResponseConverter(retrofit, method, responseType);
        //用于生成okhttp3.Call的工厂类,典型的是OkHttpClient
        okhttp3.Call.Factory callFactory = retrofit.callFactory;
        return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
    }
    ```
1. `HttpServiceMethod.invoke`
    ```java
    @Override
    ReturnT invoke(Object[] args) {
        //Call<?>经过CallAdapter转换成<ReturnT>
        return callAdapter.adapt(
                new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
    }
    ```

## `OkHttpCall`, 实现了 `Call<?>`, 包装了对okhttp3的请求调用

1. `Call<?>`
    ```java
    public interface Call<T> extends Cloneable {
        //同步请求
        Response<T> execute() throws IOException;
        //异步请求
        void enqueue(Callback<T> callback);
        //是否已请求
        boolean isExecuted();
        //取消请求
        void cancel();
        //是否已取消
        boolean isCanceled();
        Call<T> clone();
        //获取okhttp3请求对象
        okhttp3.Request request();
    }
    ```

1. `execute`同步请求,调用okhttp3的`okhttp3.call.execute`
    ```java
    @Override
    public Response<T> execute() throws IOException {
        okhttp3.Call call;

        synchronized (this) {
            if (executed) throw new IllegalStateException("Already executed.");
            executed = true;
            //creationFailure是createRawCall()过程中抛出的异常
            //因为request()可以调用多次,所以缓存该异常
            if (creationFailure != null) {
                if (creationFailure instanceof IOException) {
                    throw (IOException) creationFailure;
                } else if (creationFailure instanceof RuntimeException) {
                    throw (RuntimeException) creationFailure;
                } else {
                    throw (Error) creationFailure;
                }
            }

            call = rawCall;
            if (call == null) {
                try {
                    call = rawCall = createRawCall();
                } catch (IOException | RuntimeException | Error e) {
                    throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
                    creationFailure = e;
                    throw e;
                }
            }
        }

        if (canceled) {
            call.cancel();
        }
        //解析okhttp3.Response 为 Response<?>
        return parseResponse(call.execute());
    }
    ```

1. `createRawCall` 创建`okhttp3.Call`
    ```java
    //OkHttpCall
    private okhttp3.Call createRawCall() throws IOException {
        okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
        if (call == null) {
            throw new NullPointerException("Call.Factory returned null.");
        }
        return call;
    }

    //RequestFactory 处理参数,生成请求
    okhttp3.Request create(Object[] args) throws IOException {
        @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
                ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

        int argumentCount = args.length;
        if (argumentCount != handlers.length) {
            throw new IllegalArgumentException("Argument count (" + argumentCount
                    + ") doesn't match expected count (" + handlers.length + ")");
        }

        RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
                headers, contentType, hasBody, isFormEncoded, isMultipart);

        List<Object> argumentList = new ArrayList<>(argumentCount);
        for (int p = 0; p < argumentCount; p++) {
            argumentList.add(args[p]);
            //根据不同的注解处理参数
            handlers[p].apply(requestBuilder, args[p]);
        }

        return requestBuilder.get()
                .tag(Invocation.class, new Invocation(method, argumentList))
                .build();
    }
    ```

1. `enqueue`异步请求,逻辑与`execute`类似,调用okhttp3的`okhttp3.call.enqueue`

1. `parseResponse` 解析okhttp3.Response 为 Response<?>
    ```java
     /**
     * [Http状态码](http://www.runoob.com/http/http-status-codes.html)
     * 1** 信息，服务器收到请求，需要请求者继续执行操作
     * 2** 成功，操作被成功接收并处理
     * 3** 重定向，需要进一步的操作以完成请求
     * 4** 客户端错误，请求包含语法错误或无法完成请求
     * 5** 服务器错误，服务器在处理请求的过程中发生了错误
     */
    Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
        ResponseBody rawBody = rawResponse.body();

        // 这里为什么要替换原来的body?
        // Remove the body's source (the only stateful object) so we can pass the response along.
        rawResponse = rawResponse.newBuilder()
                .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
                .build();

        int code = rawResponse.code();
        if (code < 200 || code >= 300) {
            try {
                // 缓存响应体
                // Buffer the entire body to avoid future I/O.
                ResponseBody bufferedBody = Utils.buffer(rawBody);
                return Response.error(bufferedBody, rawResponse);
            } finally {
                rawBody.close();
            }
        }

        if (code == 204 || code == 205) {
            //body都为空
            //204   No Content      无内容。服务器成功处理，但未返回内容。
            //205   Reset Content   重置内容。服务器处理成功，终端应重置文档。
            rawBody.close();
            return Response.success(null, rawResponse);
        }

        ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
        try {
            //解析响应体
            T body = responseConverter.convert(catchingBody);
            return Response.success(body, rawResponse);
        } catch (RuntimeException e) {
            //convert()中,Source.read()捕获的异常,为什么不直接抛出?
            catchingBody.throwIfCaught();
            throw e;
        }
    }

    ```

## Retrofit.Builder

`Retrofit.Builder`中的可配置项有:

1. `client()`, `callFactory()` 用于创建`okhttp3.Call`

1. `baseUrl()` 用于设置api地址前缀

    1. 必须以 ' __/__ ' 结束

        > http://example.com/api 报错, baseUrl must end in ' / ' <br/>
        > http://example.com 正常,原因是`okhttp3.HttpUrl`在结尾补上了' / '

    1. 根据Endpoint有以下情况:

        1. 以 '/'开头,只保留`baseUrl`主机名
            > Base URL: http://example.com/api/<br/>
            > Endpoint: /foo/bar/<br/>
            > Result: http://example.com/foo/bar/

        1. Endpoint是完整地址,无视`baseUrl`
            > Base URL: http://example.com/<br/>
            > Endpoint: https://github.com/square/retrofit/<br/>
            > Result: https://github.com/square/retrofit/

        1. 以 '//'开头,无视`baseUrl`,补全为http
            > Base URL: http://example.com<br/>
            > Endpoint: //github.com/square/retrofit/<br/>
            > Result: http://github.com/square/retrofit/

1. `addConverterFactory()` 传入`Converter.Factory`用于序列化或反序列化,`defaultConverterFactoriesSize()`:
    1. 默认:size = 0

    1. Android:`OptionalConverterFactory`(AndroidSdk>=24), size = AndroidSdk >= 24 ? 1 : 0

    1. Java8:`OptionalConverterFactory`, size = 1

    ```java
    List<Converter.Factory> converterFactories = new ArrayList<>(
            1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
    //先内置,再自定义,最后默认
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());
    ```

1. `addCallAdapterFactory()` 传入`CallAdapter.Factory`用于将`Call<?>`转换为其他返回类型.`defaultCallAdapterFactories()`(按顺序):
    1. 默认:根据`callbackExecutor`是否为null,`DefaultCallAdapterFactory`或`ExecutorCallAdapterFactory`, size = 1

    1. Android:`CompletableFutureCallAdapterFactory`(AndroidSdk>=24),`ExecutorCallAdapterFactory`, size = AndroidSdk >= 24 ? 2 : 1

    1. Java8:`CompletableFutureCallAdapterFactory` + 默认

    ```java
    //先添加自定义的,再添加默认的
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
    ```

1. `callbackExecutor()` 设置`Callback`调用的线程,例如Android默认设置主线程.注意,他不影响`CallAdapter.adapt()`执行的线程

1. `validateEagerly()` 是否在调用 `create()`时就解析接口中所有方法

## 为什么Android中默认回调在主线程

Retrofit中自带有两个Factory: `DefaultCallAdapterFactory`, `ExecutorCallAdapterFactory`(指定线程池).当在Android平台下时,使用传入MainLooper的`ExecutorCallAdapterFactory`

## 方法的注解

### 请求方式

内置请求方式 | 能否有body
:-: | :-:
DELETE | false
GET | false
HEAD | false
PATCH | true
POST | true
PUT | true
OPTIONS | false

`@HTTP` 自定义请求方式

```java
public @interface HTTP {
    //方式名
    String method();
    //Endpoint
    String path() default "";
    //是否有body
    boolean hasBody() default false;
}
```

### 其他

1. `@Headers` 字符串数组, 格式必须是`"key:value"`

1. `@Multipart` 媒体类型`multipart/form-data`,在参数中添加至少1个 `@Part`,`@PartMap`.不能与`@FormUrlEncoded`同时使用

1. `@FormUrlEncoded` 媒体类型`application/x-www-form-urlencoded`,在参数中添加至少1个`@Filed`,`@FiledMap`.不能与`@Multipart`同时使用

## ParameterHandler 参数的注解

1. `@Url` 与baseUrl的拼接规则->[Retrofit.Builder](##Retrofit.Builder)中的baseUrl相关.具体规则:

    1. 不能有多个`@Url`注解

    1. 与`@Path`不能同时使用

    1. 必须在`@Query`前

    1. 必须在`@QueryName`前

    1. 必须在`@QueryMap`前

    1. 与Endpoint 不能同时使用

    1. 参数类型必须是以下之一:`HttpUrl`, `String`, `URI`, `Uri`

1. `@Path`, 替换Endpoint中的 `{...}`,具体规则:

    1. 必须在`@Query`之前

    1. 必须在`@QueryName`之前

    1. 必须在`@QueryMap`之前

    1. 与`@Url`不能同时使用

    1. 与Endpoint 必须同时使用

    1. 参数会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@Query`

    1. 类型可以是`Iterable`,`Array`,`Object`

    1. 参数会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@QueryName`

    1. 类型可以是`Iterable`,`Array`,`Object`

    1. 参数会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@QueryMap`

    1. 参数必须是`Map`

    1. `Map`必须指定类型,例如`Map<String,String>`

    1. key类型必须是`String`

    1. value类型会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@Header`

    1. 类型可以是`Iterable`,`Array`,`Object`

    1. 参数会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@HeaderMap`

    1. 参数必须是`Map`

    1. `Map`必须指定类型,例如`Map<String,String>`

    1. key类型必须是`String`

    1. value类型会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@Field`

    1. 方法必须要有`@FormUrlEncoded`注解

    1. 类型可以是`Iterable`,`Array`,`Object`

    1. 参数会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@FieldMap`

    1. 方法必须要有`@FormUrlEncoded`注解

    1. 参数必须是`Map`

    1. `Map`必须指定类型,例如`Map<String,String>`

    1. key类型必须是`String`

    1. value类型会通过`Converter.Factory.stringConverter`或`toString()`转换成String

1. `@Part`

    1. 方法必须有`@Multipart`注解

    1. value,与`MultipartBody.Part`不能同时存在

        1. 为空时,参数必须是`MultipartBody.Part`, `Iterable<MultipartBody.Part>` 或 `MultipartBody.Part[]`

        1. 不为空时,参数可以是`RequestBody`或`Object`,会通过`Converter.Factory.requestBodyConverter`转换成`RequestBody`

1. `@PartMap`

    1. 方法必须有`@Multipart`注解

    1. 参数必须是`Map`

    1. `Map`必须指定类型,例如`Map<String,String>`

    1. key类型必须是`String`

    1. value类型不能是`MultipartBody.Part`,会通过`Converter.Factory.requestBodyConverter`转换成`RequestBody`

1. `@Body`

    1. 不能与`@FormUrlEncoded`或`@Multipart`同时使用

    1. 只能有一个`@Body`

    1. 参数会通过`Converter.Factory.requestBodyConverter`转换成`RequestBody`
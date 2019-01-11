# Retrofit 笔记

_Retrofit version:2.5.0_<br/>
_需要用到[Type相关知识](../../Java知识点/反射相关/Type.md)_

---

## 调用`create(class)`后发生了什么

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

1. `loadServiceMethod` 解析注解
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
    //ServiceMethod.java
    //返回类型的判断
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

        return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
    }
    //HttpServiceMethod.java
    static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
            Retrofit retrofit, Method method, RequestFactory requestFactory) {
        CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
        Type responseType = callAdapter.responseType();
        if (responseType == Response.class || responseType == okhttp3.Response.class) {
            throw methodError(method, "'"
                    + Utils.getRawType(responseType).getName()
                    + "' is not a valid response body type. Did you mean ResponseBody?");
        }
        if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
            throw methodError(method, "HEAD method must use Void as response type.");
        }

        Converter<ResponseBody, ResponseT> responseConverter =
                createResponseConverter(retrofit, method, responseType);

        okhttp3.Call.Factory callFactory = retrofit.callFactory;
        return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
    }
    ```
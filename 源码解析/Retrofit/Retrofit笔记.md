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
                            return platform.invokeDefaultMethod(method, service, proxy, args);
                        }
                        return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
                    }
                });
    }
    ```
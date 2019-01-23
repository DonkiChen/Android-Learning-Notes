# Type

_Java version:1.8_<br/>
_[示例代码片段](https://gist.github.com/DonkiChen/0007b647f2ab9d6970550885945ce147)_

---

`Type` 是 Java 中所有类型的超接口 (superinterface), 他的直接子类包括:

1. `GenericArrayType` 数组类型
    ```java
    public interface GenericArrayType extends Type {
        // 获取该数组的类型
        Type getGenericComponentType();
    }
    ```
    例子: `Test.Inner<String>[]`, `Test.Inner<String>...`
    > method: private void testGenericArrayType(Test.Inner<String>[] a)<br/>
    >Type: com.chen.Test$Inner<java.lang.String>[] is GenericArrayType<br/>
    >getGenericComponentType() = com.chen.Test$Inner<java.lang.String>

1. `ParameterizedType` 参数化类型，用于泛型使用时
    ```java
    public interface ParameterizedType extends Type {
        // 获取实际类型，即<>内
        Type[] getActualTypeArguments();
        // 获取原始类型，即<>外
        Type getRawType();
        // 获取外部类型，用于内部类时
        Type getOwnerType();
    }
    ```
    例子: `Test.Inner<String>`
    > method: private void testParameterizedType(Test.Inner<String> a)<br/>
    > Type: com.chen.Test$Inner<java.lang.String> is ParameterizedType<br/>
    > getActualTypeArguments() = [class java.lang.String]<br/>
    > getOwnerType() = class com.chen.Test<br/>
    > getRawType() = class com.chen.Test$Inner

1. `TypeVariable` 类型变量，用于泛型定义时
    ```java
    public interface TypeVariable<D extends GenericDeclaration> extends Type, AnnotatedElement {
        //return an array of {@code Type}s representing the upper bound(s) of this type variable, 获取泛型上界
        Type[] getBounds();
        // 获取定义泛型的类
        D getGenericDeclaration();
        // 获取泛型的名字
        String getName();
        //return an array of objects representing the upper bounds of the type variable, 也是获取泛型上界，与 getBounds() 返回的类型不同
        AnnotatedType[] getAnnotatedBounds();
    }
    ```
    例子: `class Test<D extends Number & Serializable>`
    >method: private void testTypeVariable(D d)<br/>
    >Type: D is TypeVariable<br/>
    >getName() = D<br/>
    >getAnnotatedBounds()[0].getType() = class java.lang.Number<br/>
    >getAnnotatedBounds()[1].getType() = interface java.io.Serializable<br/>
    >getBounds() = [class java.lang.Number, interface java.io.Serializable]<br/>
    >getGenericDeclaration() = class com.chen.Test

1. `WildcardType`, 通配符，`?`, `? extends Number`, `? super Integer`
    ```java
    public interface WildcardType extends Type {
        // 获取上界，默认 Object
        Type[] getUpperBounds();
        // 获取下界，默认 null
        Type[] getLowerBounds();
        // one or many? Up to language spec; currently only one, but this API
        // allows for generalization.
        // 一个或多个取决于语言特性，当前是 1 个，API 保证通用性
    }
    ```
    例子: `Test.Inner<? super Number>`
    >method: private void testWildcardType(Test.Inner<? super Number> a)<br/>
    >Type: com.chen.Test$Inner<? super java.lang.Number> is ParameterizedType<br/>
    >getActualTypeArguments() = [? super java.lang.Number]<br/>
    >getOwnerType() = class com.chen.Test<br/>
    >getRawType() = class com.chen.Test$Inner<br/>
    >Type: ? super java.lang.Number is WildcardType<br/>
    >getUpperBounds() = [class java.lang.Object]<br/>
    >getLowerBounds() = [class java.lang.Number]
1. `Class`
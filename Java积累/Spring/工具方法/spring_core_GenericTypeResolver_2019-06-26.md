# 泛型类型解析工具——GenericTypeResolver

> **GenericTypeResolver** 是基于 [ResolvableType](spring_core_ResolvableType_2019-06-26.md) 做的一层封装。提供更加简便的泛型解析。

## 运行示例
`spring-core` 模块中 `test` 下  
* `org.springframework.core.GenericTypeResolverTests`

## 重要方法：

### <span id="resolveTypeArgument">org.springframework.core.GenericTypeResolver#resolveTypeArgument(..)</span>
将 **Class** 解析为指定 **Class** 类型，并返回第一个泛型的 **Class** 类型

### <span id="resolveTypeArguments">org.springframework.core.GenericTypeResolver#resolveTypeArguments(..)</span>
与 [resolveTypeArgument(..)](#resolveTypeArgument) 相对应的，但是解析的是所有的泛型 **Class[]**

### <span id="resolveReturnTypeArgument">org.springframework.core.GenericTypeResolver#resolveReturnTypeArgument(..)</span>
将 **Method** 解析为指定 **Class** 类型，并返回第一个泛型的 **Class** 类型

### <span id="resolveReturnType">org.springframework.core.GenericTypeResolver#resolveReturnType(..)</span>
将 **Method** 以指定的实现类解析，获取其返回值 **Class** 类型

### <span id="getTypeVariableMap">org.springframework.core.GenericTypeResolver#getTypeVariableMap(..)</span>
对指定 **Class** 中的泛型进行解析，并缓存到 `Map<TypeVariable, Type> varMap` 中。如：T -> Integer, V -> String
注意：
* 泛型没有指定时，则不添加到解析 map 中去；
* 当前类 **Class** 中包含内部类时，不会去解析其内部类；
* 但是反过来，内部类 **Class** 会解析其外部类；
* 同时也会解析其**父类**、**实现接口**；

### <span id="resolveType">org.springframework.core.GenericTypeResolver#resolveType(..)</span>
将 `java.lang.reflect.Type` 使用上面 [getTypeVariableMap(..)](#getTypeVariableMap) 解析的结果，来进行解析填充。
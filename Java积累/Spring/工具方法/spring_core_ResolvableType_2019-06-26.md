# 解析类型工具——ResolvableType

> 背景：

## 运行示例：
`spring-core` 模块中 `test` 下  
 * `org.springframework.core.ResolvableTypeTests`。
 * `org.springframework.core.MethodParameterTests`。

## 重要方法：
### `org.springframework.core.MethodParameter`
运行示例：`org.springframework.core.ResolvableTypeTests#forMethodParameterWithNestingAndLevels`
`MethodParameter` 的关键属性:
* executable: `java.lang.reflect.Executable`, `Method` 或 `Constructor`。执行函数
* <span id="parameterIndex">parameterIndex</span>: int
    * -1: 函数的返回值，如：void
    *  0: 第一个入参
    *  1: 第二个入参
    * ...
* <span id="nestingLevel">nestingLevel</span>: int，默认是 1
    * 1: 此类型
    * 2: 第一个泛型
    * 3: 第二个泛型
    * ...
* typeIndexesPerLevel: Map<Integer, Integer>，解析是有顺序的，根据
    * key: [nestingLevel](#nestingLevel)，指定第几个泛型
    * value: [parameterIndex](#parameterIndex)，对应泛型
    * 示例：
        ```java
        @Test
        public void forMethodParameterWithNestingAndLevels_2019_06_26() throws Exception {
            // void nested(Map<Map<String, Integer>, Map<Byte, Long>> p);
            Method method = Methods.class.getMethod("nested", Map.class);
            // nested 方法的 第一个入参，
            MethodParameter methodParameter = new MethodParameter(method, 0, 1);
            // typeIndexesPerLevel
            // (2, 0)
            methodParameter.increaseNestingLevel();
            // Map<String, Integer>
            methodParameter.setTypeIndexForCurrentLevel(0);
            // typeIndexesPerLevel
            // (2, 0)
            // (3, 0)
            methodParameter.increaseNestingLevel();
            // Integer
            methodParameter.setTypeIndexForCurrentLevel(1);
            ResolvableType type = ResolvableType.forMethodParameter(methodParameter);
            assertThat(type.resolve(), equalTo((Class) Integer.class));
        }
        ```
    由上面代码可以发现，`typeIndexesPerLevel` 是在执行 `ResolvableType type = ResolvableType.forMethodParameter(methodParameter);` 前指定好了的，目的是为了在解析为 `ResolvableType` 直接通过偏移量，来指定定位需要解析的泛型
    当然其实也可以直接通过**外层泛型** [`type.getGeneric(i).getGeneric(i)`](#getGeneric) 的方式，一层一层的来获取对应的泛型。

### <span id="getGeneric">`org.springframework.core.ResolvableType#getGeneric(..)`</span>
获取泛型的 `ResolvableType`。
```java
    // 默认获取 index 为 0 的泛型，即第一个泛型
    type.getGeneric();
    // 获取第一个泛型
    type.getGeneric(0);
    // 获取第二个泛型
    type.getGeneric(1);
    ...
    // Map<Integer, List<String>>，获取第二个泛型 List<String> 的第一个泛型，即 String
    type.getGeneric(1, 0);
```

### <span id="resolve">`org.springframework.core.ResolvableType#resolve(..)`</span>
获取解析后的 **Class**

### <span id="resolveGeneric">org.springframework.core.ResolvableType#resolveGeneric(..)</span>
获取指定泛型的解析结果。
[getGeneric](#getGeneric) + [resolve](#resolve)
```java
    @Nullable
    public Class<?> resolveGeneric(int... indexes) {
        return getGeneric(indexes).resolve();
    }
```

### <span id="getType">org.springframework.core.ResolvableType#getType()</span>
返回正在管理的 `java.lang.reflect.Type`。如：
* `org.springframework.core.SerializableTypeWrapper$FieldTypeProvider`
* `org.springframework.core.SerializableTypeWrapper$MethodParameterTypeProvider`
* `org.springframework.core.SerializableTypeWrapper$MethodInvokeTypeProvider`
以上的 **TypeProvider** 会使用 **JDK** 动态代理，来重写某些方法。代理类 `org.springframework.core.SerializableTypeWrapper.TypeProxyInvocationHandler`。

### <span id="toClass">org.springframework.core.ResolvableType#toClass()</span>
[`resolve(Object.class);`](#resolve)，获取解析后的 **Class**，获取不到时，以 **Object.class** 返回。


### <span id="getComponentType">org.springframework.core.ResolvableType#getComponentType()</span>
返回表示数组或组件类型的 `ResolvableType`,如果类型不是数组，则返回 `org.springframework.core.ResolvableType#NONE`


### <span id="forType">org.springframework.core.ResolvableType#forType(...)</span>
将 **owner** `ResolvableType` 以指定支持的 `Type` 类型返回。
如：**owner** 是 `List<String>` 解析的 `ResolvableType`，以 `Type` 为 `ArrayList.class` 类型解析返回。


### class1.isAssignableFrom(class2)
**class2** 是不是 **class1** 的子类或子接口。
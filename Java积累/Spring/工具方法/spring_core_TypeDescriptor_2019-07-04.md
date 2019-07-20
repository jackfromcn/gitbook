# 类型描述工具——TypeDescriptor

> 类型转换的上下文

## 属性字段含义
* <span id="type">type</span>: `Class<?>`，指定的 `MethodParameter`(`nestinglevel` 和 `parameterIndex` 已经确定了唯一的参数)、`Field` 或 `org.springframework.core.convert.Property`，获取其 **class** 类型，包括泛型的 **class** 类型。
* <span id="resolvableType">resolvableType</span>: [`ResolvableType`](spring_core_GenericTypeResolver_2019-06-26.md)，当前入参的解析类型。
* <span id="annotatedElement">annotatedElement</span>: `org.springframework.core.convert.TypeDescriptor.AnnotatedElementAdapter`，注解。

## 运行示例：
`spring-core` 模块中 `test` 下  
 * `org.springframework.core.convert.TypeDescriptorTests`。

 ## 重要方法：

 ### <span id="getType">org.springframework.core.convert.TypeDescriptor#getType()</span>
@link [type](#type)

### <span id="getObjectType">org.springframework.core.convert.TypeDescriptor#getObjectType()</span>
返回 [type](#type) 的包装类型。如：int ==> Integer

### <span id="nested">org.springframework.core.convert.TypeDescriptor#nested(..)</span>
参数列表：
* methodParameter: `MethodParameter`，待解析的方法参数
* <span id="nestingLevel">nestingLevel</span>: `int`，嵌套级别。
1：取下一个泛型的**类型描述**，

**注意：**
这里的 [nestingLevel](#nestingLevel) 与 [`MethodParameter#nestingLevel`](spring_core_GenericTypeResolver_2019-06-26.md#nestingLevel) 含义还是有差别的。
`MethodParameter` 中的 [`nestingLevel`](spring_core_GenericTypeResolver_2019-06-26.md#nestingLevel) 是从 **2** 开始解析下一个泛型的。
但是这里的 [nestingLevel](#nestingLevel) 直接是从 **1** 开始解析的。方法实现如下：
```java
	@Nullable
	private static TypeDescriptor nested(TypeDescriptor typeDescriptor, int nestingLevel) {
		ResolvableType nested = typeDescriptor.resolvableType;
		for (int i = 0; i < nestingLevel; i++) {
			if (Object.class == nested.getType()) {
				// Could be a collection type but we don't know about its element type,
				// so let's just assume there is an element type of type Object...
			}
			else {
                // 循环解析 嵌套结果 的第一个泛型
				nested = nested.getNested(2);
			}
		}
		if (nested == ResolvableType.NONE) {
			return null;
		}
		return getRelatedIfResolvable(typeDescriptor, nested);
	}
```
这里虽然没有指定 [`typeIndexesPerLevel`](spring_core_GenericTypeResolver_2019-06-26.md#typeIndexesPerLevel)，但是通过自己循环调用 `nested = nested.getNested(2);` 方式，达到了简单解析嵌套泛型的目的。
但是也只能简单解析，为什么这么说呢？因为如果**嵌套泛型**中的某个**泛型**，包含了多个**泛型类型**。如：Map<String, Intger>。这样的话，这里解析默认的都是最后一个泛型。通过指定 [`typeIndexesPerLevel`](spring_core_GenericTypeResolver_2019-06-26.md#typeIndexesPerLevel) 可以更加精确的说明，解析每个**泛型类型**时，需要解析其哪一个**泛型类型**。

### <span id="getElementTypeDescriptor">org.springframework.core.convert.TypeDescriptor#getElementTypeDescriptor()</span>
获取**元素**的类型描述
* isArray(): [],数组类型。返回数组的**元素**类型。
* `Stream.class.isAssignableFrom(getType())`: **Stream** 类型，返回 **Stream** 中的元素类型。
* 其他：则以 **Collection** 类型解析，如果是 **Collection** 类型，则返回其元素类型，否则返回 **Null**。

### <span id="getMapKeyTypeDescriptor">org.springframework.core.convert.TypeDescriptor#getMapKeyTypeDescriptor()</span>
获取 **map** 中 key 的**类型描述**。
如果当前类型不是 map 类型，则会抛出 `java.lang.IllegalStateException` 异常。
如果参数中指定了 key 的对象，则将 key 的**类型描述**缩小到其指定类型的类型描述。具体可以见 [narrow](#narrow) 功能。

### <span id="getMapValueTypeDescriptor">org.springframework.core.convert.TypeDescriptor#getMapValueTypeDescriptor()</span>
与上面 [`getMapKeyTypeDescriptor()`](#getMapKeyTypeDescriptor) 相对应，这里返回的是 value 的类型描述

### <span id="forObject">org.springframework.core.convert.TypeDescriptor#forObject(..)</span>
具体调用下面的 [`valueof(..)`](#valueOf) 方法。

### <span id="valueOf">org.springframework.core.convert.TypeDescriptor#valueOf()</span>
以指定 **class** 类型创建 **TypeDescriptor**，基本类型已经缓存好了，无需创建。
```java
	private static final Map<Class<?>, TypeDescriptor> commonTypesCache = new HashMap<>(32);

	private static final Class<?>[] CACHED_COMMON_TYPES = {
			boolean.class, Boolean.class, byte.class, Byte.class, char.class, Character.class,
			double.class, Double.class, float.class, Float.class, int.class, Integer.class,
			long.class, Long.class, short.class, Short.class, String.class, Object.class};

	static {
		for (Class<?> preCachedClass : CACHED_COMMON_TYPES) {
			commonTypesCache.put(preCachedClass, valueOf(preCachedClass));
		}
	}
```


### <span id="getAnnotation">org.springframework.core.convert.TypeDescriptor#getAnnotation(..)</span>
获取指定 **class** 的注解


### <span id="narrow">org.springframework.core.convert.TypeDescriptor#narrow(..)</span>
缩小 **TypeDescriptor** 到指定范围。如：Object --> HashMap


### <span id="upcast">org.springframework.core.convert.TypeDescriptor#upcast(..)</span>
与 `narrow(..)` 方法相对应，这个是放大返回。如：HashMap --> Object


### <span id="elementTypeDescriptor">org.springframework.core.convert.TypeDescriptor#elementTypeDescriptor(..)</span>
将集合的元素类型缩小为指定元素类型，如：`java.util.List<java.lang.Number>`的类型描述，入参是 `Integer`，则返回 `List<Integer>` 类型的描述。`java.util.List<?>`的类型描述，入参是 `Integer`，则也返回 `List<Integer>` 类型的描述。
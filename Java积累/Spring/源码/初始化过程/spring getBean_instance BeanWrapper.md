# Spring 加载Bean的过程_实例化 BeanWrapper

> 流程都是基于 **ObjectFactory** 对象中的内部方法。



## <span id="method_createBeanInstance_main_process">createBeanInstance() 实例化 BeanWrapper</span>

实例化 Bean 对象，是一个复杂的过程，其主要的逻辑为：

- <1> 处，如果存在 Supplier 回调，则调用 `#obtainFromSupplier(Supplier<?> instanceSupplier, String beanName)` 方法，进行初始化。
- <2> 处，如果存在工厂方法，则使用工厂方法进行初始化。
- <3> 处，首先判断缓存，如果缓存中存在，即已经解析过了，则直接使用已经解析了的。根据 constructorArgumentsResolved 参数来判断：
  - <3.1> 处，是使用构造函数自动注入，即调用 `#autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)` 方法。
  - <3.2> 处，还是默认构造函数，即调用 `#instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。
- <4> 处，如果缓存中没有，则需要先确定到底使用哪个构造函数来完成解析工作，因为一个类有多个构造函数，每个构造函数都有不同的构造参数，所以需要根据参数来锁定构造函数并完成初始化。
  - <4.1> 处，如果存在参数，则使用相应的带有参数的构造函数，即调用 `#autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs)` 方法。
  - <4.2> 处，否则，使用默认构造函数，即调用 `#instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。



## <span id="key_methds">关键方法</span>

实例化好的构造方法已经在 **RootBeanDefinition** 中，如果不存在，则使用默认无参构造器进行实例化。



`determineConstructorsFromBeanPostProcessors()`

[`autowireConstructor()`](#method_autowireConstructor_main_process)



### <span id="method_determineConstructorsFromBeanPostProcessors_main_process">determineConstructorsFromBeanPostProcessors() 方法流程</span>





### <span id="method_autowireConstructor_main_process">autowireConstructor() 方法流程</span>

`autowireConstructor()` 方法如下，首先将当前 **ObjectFactory** 作为 **ConstructorResolver** 的构造参数。

```java
protected BeanWrapper autowireConstructor(
		String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

	return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```



真正实现业务逻辑的还是在 **ConstructorResolver** 对象中，调用 **`ConstructorResolver#autowireConstructor()`** 的方法，方法定义如下：<br/>

```java
/**
 * @param mbd bean 合并的 beanDefinition 定义
 * @param chosenCtors 待选择的构造器数组，这个 Class 中定义的所有构造函数
 * @param explicitArgs 通过getBean方法以编程方式传递的参数值
 * @return a BeanWrapper for the new instance
 */
public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs){
}
```

- **RootBeanDefinition**: 合并的 beanDefinition 定义
- **`Constructor<?>[]`**: 待选择的构造器数组，这个 Class 中定义的所有构造函数
- *explicitArgs*: 通过getBean方法以编程方式传递的参数值

剔除一些从缓存中的获取的优化逻辑。只列举一些**初始逻辑**的**关键步骤**。



#### 方法内的重要的临时属性字段

- **constructorToUse**: 构造函数
- **argsHolderToUse**: 构造参数
- **argsToUse**: 构造参数



#### <span id="method_resolvePreparedArguments_main_process">resolvePreparedArguments()解析参数</span>

走到这里肯定是 **RootBeanDefinition** 缓存中获取需要动态参数解析的参数列表。

主要逻辑如下：<br/>

- **if**、**else**、**instanceof** 来判断走哪个类型解析。
- 动态字符串的使用 `BeanExpressionResolver#evaluate()` 来解析，具体实现类有 `StandardBeanExpressionResolver`。

**注意**：

参数为 **AutowiredArgumentMarker** 类型时，实例化对象时，会走 ***AOP*** 切面配置的 bean  实例化获取。并注册 bean 的依赖关系。`ConstructorResolver#resolveAutowiredArgument()`。

参数为 **BeanMetadataElement** 类型时，调用的是 `BeanDefinitionValueResolver#resolveValueIfNecessary ()`。



#### <span id="method_resolveConstructorArguments_main_process">resolveConstructorArguments() 解析构造参数</span>

将该 bean 的构造函数参数解析为 resolvedValues 对象，其中会涉及到其他 bean。

逻辑同上面 [resolvePreparedArguments()解析参数](#method_resolvePreparedArguments_main_process)。

但是只有 [resolvePreparedArguments()解析参数](#method_resolvePreparedArguments_main_process) 中参数类型为 **BeanMetadataElement** 时的情况。



#### <span id="method_createArgumentArray_main_process">createArgumentArray() 创建参数持有者 ArgumentsHolder 对象</span>

根据构造函数和构造参数，创建参数持有者 ArgumentsHolder 对象

- 跟 参数索引、参数类型、参数名 从上面解析到的 **resolvedValues** List 中获取解析好的 **ConstructorArgumentValues.ValueHolder**，并将解析好的 **valueHolder** 内容赋值给当前 **ArgumentsHolder**。
- 如果获取不到，`ConstructorResolver#resolveAutowiredArgument()` 来解析参数，并将 **preparedArguments** 对应索引上的对象标识为 **AutowiredArgumentMarker**，[供下次参数动态解析](#method_resolvePreparedArguments_main_process)时类重新对对象解析。
- 对于需要依赖的 bean，进行依赖关系注入。



#### <span id="method_ArgumentsHolder_storeCache_main_process">ArgumentsHolder#storeCache() 将解析的构造函数加入缓存</span>

- **Lock**: **RootBeanDefinition** 的 **constructorArgumentLock** 作为锁。构造器锁。
- **resolvedConstructorOrFactoryMethod**: 缓存解析好的 **构造函数** 或 **工厂实例方法**。
- **constructorArgumentsResolved**: true，标识已经解析过。
- **preparedConstructorArguments** 和 **resolvedConstructorArguments**: 根据是否需要解析，缓存对应的值。

```java
public void storeCache(RootBeanDefinition mbd, Executable constructorOrFactoryMethod) {
	synchronized (mbd.constructorArgumentLock) {
		mbd.resolvedConstructorOrFactoryMethod = constructorOrFactoryMethod;
		mbd.constructorArgumentsResolved = true;
		if (this.resolveNecessary) {
			mbd.preparedConstructorArguments = this.preparedArguments;
		}
		else {
			mbd.resolvedConstructorArguments = this.arguments;
		}
	}
}
```


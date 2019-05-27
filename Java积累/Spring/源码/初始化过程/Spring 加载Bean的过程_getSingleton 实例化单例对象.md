# Spring 加载Bean的过程_getSingleton 实例化单例对象



## 关键字段

- <span id="factoryBeanInstanceCache_desc">**factoryBeanInstanceCache**</span>: 未完成的FactoryBean实例的缓存，beanName --> BeanWrapper



## <span id="method_getSingleton_main_process">getSingleton() 方法主流程</span>

主流程：<br/>

- **Lock**: [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 作为锁
- **双重判断**，从 [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 中获取实例 bean，如果能获取到，则直接返回。否则，继续实例化。
- **`beforeSingletonCreation`**: 加载前置处理。
- `ObjectFactory#getObject()`: 调用其入参中的 **ObjectFactory** 来实例化对象。其 `getObject()` 方法在前面匿名方法中定义的还是调用 [**`AbstractBeanFactory#createBean()`**](#method_createBean_main_process) 方法。 
- **`afterSingletonCreation`**: 后置处理
- [**`addSingleton()`**](#method_addSingleton_main_process): 加入缓存中



### <span id="method_createBean_main_process">AbstractBeanFactory#createBean() 方法主流程</span>

- 通过 `RootBeanDefinition` 定义解析出实例化的 **Class** 对象，并判断是否需要添加到定义的 `RootBeanDefinition#beanClass` 中。
  - 主要是因为该动态解析的 class 无法保存到到共享的 BeanDefinition
- `AbstractBeanDefinition#prepareMethodOverrides`: 验证和准备覆盖方法
- `resolveBeforeInstantiation`: 实例化的前置处理
- [**`doCreateBean`**](#method_doCreateBean_main_process): 核心逻辑，AOP 的功能就是基于这个地方



#### <span id="method_doCreateBean_main_process">doCreateBean() 方法主流程</span>

整体的思路：

- <1>  如果是单例模式，则清除缓存。
- <2>  调用 `#createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args)` 方法，实例化 bean ，主要是将 BeanDefinition 转换为 org.springframework.beans.BeanWrapper 对象。
- <3>  MergedBeanDefinitionPostProcessor 的应用。
- <4>  单例模式的循环依赖处理。
- <5>  调用 #populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) 方法，进行属性填充。将所有属性填充至 bean 的实例中。
- <6>  调用 #initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) 方法，初始化 bean 。
- <7>  依赖检查。
- <8>  注册 DisposableBean



- [**factoryBeanInstanceCache**](#factoryBeanInstanceCache_desc) 中 **remove**，BeanWrapper ==> 变量 instanceWrapper。
- 如果变量 instanceWrapper 还是为 **Null**，则通过 [**`createBeanInstance()`**](spring getBean_instance BeanWrapper.html#method_createBeanInstance_main_process) 方法实例化一个。
- [`addSingletonFactory()`](#method_addSingletonFactory_main_process): 解决单例模式的循环依赖。
- [`populateBean()`](): 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性，则会递归初始依赖 bean
- [`initializeBean()`](): 调用初始化方法
- 如果是单例的话，则需要解决循环依赖问题。
- `registerDisposableBeanIfNecessary()` 注册 bean。



##### <span id="method_addSingletonFactory_main_process">addSingletonFactory() 逻辑</span>

- **Lock**: [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 作为锁。
-  [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 中不包含改 beanName 的实例，即为初始化完成。
  - [**singletonFactories**](Spring加载Bean的过程.html#singletonFactories_desc) 添加， beanName ——> ObjectFactory。
  - [**earlySingletonObjects**](Spring加载Bean的过程.html#earlySingletonObjects_desc) remove。
  - [**registeredSingletons**](Spring加载Bean的过程.html#registeredSingletons_desc) 添加， beanName。注册。



### <span id="method_addSingleton_main_process">addSingleton() 方法主流程</span>

- **Lock**: [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 作为锁。
- [**singletonObjects**](Spring加载Bean的过程.html#singletonObjects_desc) 添加 beanName ——> instance。
- [**singletonFactories**](Spring加载Bean的过程.html#singletonFactories_desc) 移除 beanName：实例化完成，移除其 **ObjectFactory**。
- [**earlySingletonObjects**](Spring加载Bean的过程.html#earlySingletonObjects_desc) 移除 beanName: 实例化完成，移除其 bean。
- [**registeredSingletons**](Spring加载Bean的过程.html#registeredSingletons_desc) 添加 beanName: 注册单例实例化 beanName



## <span id="method_getObjectForBeanInstance_main_process">getObjectForBeanInstance() 方法主流程</span>
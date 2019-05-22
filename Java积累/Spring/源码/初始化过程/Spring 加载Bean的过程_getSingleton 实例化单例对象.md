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

- **factoryBeanInstanceCache** 中



### <span id="method_addSingleton_main_process">addSingleton() 方法主流程</span>

- **Lock**: [**singletonObjects**](#singletonObjects_desc) 作为锁。
- [**singletonObjects**](#singletonObjects_desc) 添加 beanName ——> instance。
- [**singletonFactories**](#singletonFactories_desc) 移除 beanName：实例化完成，移除其 **ObjectFactory**。
- [**earlySingletonObjects**](#earlySingletonObjects_desc) 移除 beanName: 实例化完成，移除其 bean。
- [**registeredSingletons**](#registeredSingletons_desc) 添加 beanName: 注册单例实例化 beanName



## <span id="method_getObjectForBeanInstance_main_process">getObjectForBeanInstance() 方法主流程</span>
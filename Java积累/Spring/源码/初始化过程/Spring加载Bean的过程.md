# Spring加载Bean的过程

> 单例模式下

## 关键属性字段

关键属性介绍：<br/>

- <span id="singletonObjects_desc">**singletonObjects**</span>: beanName --> beanInstance，存放的是单例 bean 的映射。
- <span id="earlySingletonObjects_desc">**earlySingletonObjects**</span>: beanName --> beanInstance，存放的是【早期】的单例 bean 的映射。

- <span id="singletonFactories_desc">**singletonFactories**</span>:  beanName --> ObjectFactory，存放的是 ObjectFactory 的映射，可以理解为创建单例 bean 的 factory 。
- <span id="singletonsCurrentlyInCreation_desc">**singletonsCurrentlyInCreation**</span>: 正在创建中的单例 Bean 的名字的集合



- <span id="registeredSingletons_desc">**registeredSingletons**</span>： 注册的单例 bean



### <span id="singletonObjects_diff_with_earlySingletonObjects">**singletonObjects** 与 **earlySingletonObjects** 的区别</span>

它与 **singletonObjects** 的区别区别在于 **earlySingletonObjects** 中存放的 bean 不一定是完整的。从 **getSingleton(String)** 方法中，中我们可以了解，bean 在创建过程中就已经加入到 **earlySingletonObjects** 中了，所以当在 bean 的创建过程中就可以通过 `getBean()` 方法获取。这个 Map 也是解决【循环依赖】的关键所在。



## AbstractBeanFactory#doGetBean() 流程

**主流程：**

- `getSingleton`：[从缓存中获取 bean 对象](#method_getSingleton_load_bean_from_cache)。
- 如果 `AbstractBeanFactory#parentBeanFactory != null`，且当前 `AbstractBeanFactory` 不包含其 `BeanDefinition` 的定义。则递归从其父类 `BeanFactory` 中加载其 bean 实例返回。
- 不满足上面条件，即需要从当前 `BeanFactory` 中加载 bean 实例。
- `getMergedLocalBeanDefinition`: [从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象](#method_getMergedLocalBeanDefinition)
- 实例化对象：根据定义的 scope 规则调用其实例化方法，[这里只介绍单例模式下](#singleton_bean_instance_process)。大部分逻辑都是一样的。
- 判断是否需要转换成指定的类型，如果需要，则进行类型转换，并返回。否则，直接返回实例化的 bean。



### <span id="method_getSingleton_load_bean_from_cache">getSingleton() 从缓存中获取 bean 对象流程</span>

#### <span id="method_getSingleton_main_process">方法主流程</span>

- 在 [**singletonObjects**](#singletonObjects_desc) 中获取实例 bean。如果能获取到，则直接返回。否则，根据 [**singletonsCurrentlyInCreation**](#singletonsCurrentlyInCreation_desc) 判断是否正在创建中。如果是，则从下面缓存中继续获取，否则，则直接返回 **Null**。
- **Lock**: [**singletonObjects**](#singletonObjects_desc) 作为锁。
- 在 [**earlySingletonObjects**](#earlySingletonObjects_desc) 中获取实例 bean。如果获取到，则直接返回。否则，继续从缓存中获取。
- 在 [**singletonFactories**](#singletonFactories_desc) 中获取其对象工厂 **ObjectFactory**，如果获取到了，则通过 **ObjectFactory#get()** 方法实例化 bean 实例，并添加到 [**earlySingletonObjects**](#earlySingletonObjects_desc) 中，从 [**singletonFactories**](#singletonFactories_desc) 中移除对应的 **ObjectFactory**。返回实例化的 bean。否则，直接返回 **Null**。



### <span id="method_getMergedLocalBeanDefinition">getMergedLocalBeanDefinition() 获取 RootBeanDefinition</span>





### <span id="getSingleton">单例实例化过程</span>



- 实例化匿名 **ObjectFactory** 对象，作为 `getSingleton()` 的参数。
- [**`getSingleton()`**](#method_getSingleton_main_process) 方法主流程: 获取单例 bean 实例。
- [**`getObjectForBeanInstance()`**](#method_getObjectForBeanInstance_main_process) 方法主流程: 根据指定 beanName 返回需要的实例。(beanName -> "&beanName"，这种这是直接返回工厂类，而不是工厂类创建的 bean)
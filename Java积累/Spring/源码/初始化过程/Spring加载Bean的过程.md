# Spring加载Bean的过程

> 单例模式下



## AbstractBeanFactory#getBean() 流程

**主流程：**

- `getSingleton`：[从缓存中获取 bean 对象](#method_getSingleton_load_bean_from_cache)。

### <span id="method_getSingleton_load_bean_from_cache">getSingleton() 从缓存中获取 bean 对象流程</span>



- <span id="singletonObjects_desc">**singletonObjects**</span>: beanName --> beanInstance，存放的是单例 bean 的映射。
- <span id="earlySingletonObjects_desc">**earlySingletonObjects**</span>: beanName --> beanInstance，存放的是【早期】的单例 bean 的映射。

- <span id="singletonFactories_desc">**singletonFactories**</span>:  beanName --> ObjectFactory，存放的是 ObjectFactory 的映射，可以理解为创建单例 bean 的 factory 。
- <span id="singletonsCurrentlyInCreation_desc">**singletonsCurrentlyInCreation**</span>: 正在创建中的单例 Bean 的名字的集合



#### <span id="singletonObjects_diff_with_earlySingletonObjects">**singletonObjects** 与 **earlySingletonObjects** 的区别</span>

它与 **singletonObjects** 的区别区别在于 **earlySingletonObjects** 中存放的 bean 不一定是完整的。从 **getSingleton(String)** 方法中，中我们可以了解，bean 在创建过程中就已经加入到 **earlySingletonObjects** 中了，所以当在 bean 的创建过程中就可以通过 `getBean()` 方法获取。这个 Map 也是解决【循环依赖】的关键所在。



#### <span id="method_getSingleton_main_process">方法主流程</span>

- 在 [**singletonObjects**](#singletonObjects_desc) 中获取实例 bean。如果能获取到，则直接返回。否则，根据 [**singletonsCurrentlyInCreation**](#singletonsCurrentlyInCreation_desc) 判断是否正在创建中。如果是，则从下面缓存中继续获取，否则，则直接返回 **Null**。
- 在 [**earlySingletonObjects**](#earlySingletonObjects_desc) 中获取实例 bean。如果获取到，则直接返回。否则，继续从缓存中获取。
- 在 [**singletonFactories**](#singletonFactories_desc) 中获取其对象工厂 **ObjectFactory**，如果获取到了，则通过 **ObjectFactory#get()** 方法实例化 bean 实例，并添加到 [**earlySingletonObjects**](#earlySingletonObjects_desc) 中，从 [**singletonFactories**](#singletonFactories_desc) 中移除对应的 **ObjectFactory**。返回实例化的 bean。否则，直接返回 **Null**。
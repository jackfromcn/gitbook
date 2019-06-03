# Spring#getBean() 加载 BeanInstance 主要流程

> BeanDefinition 已经从配置文件中加载完毕。现在是将 spring 内部定义的 BeanDefinition 实例化为需要的 beanInstance。
> doGetBean() 方法主流程。

## <span id="doGetBean">doGetBean(): 方法主流程</span>

* getSingleton() 从缓存中或者实例工厂中获取 Bean 对象
    * [`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String)`](#getSingleton1)
* getSingleton() 实例化单例 bean
    * [`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)`](#getSingleton2)
    * <span id="doGetBean_getSingleton2_ObjectFactory">ObjectFactory 为 匿名内部类</span>
    ```java
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            // 显式从单例缓存中删除 Bean 实例
            // 因为单例模式下为了解决循环依赖，可能他已经存在了，所以销毁它。 TODO
            destroySingleton(beanName);
            throw ex;
        }
    });
    ```
* getObjectForBeanInstance() 实例化 bean，factoryBean 实例化出来的 bean，需要在这里实例化。


 ## <span id="getSingleton1">getSingleton() 从缓存中获取 Bean 实例</span>
* 先从 **singletonObjects** 缓存中获取 完整的 bean，能获取到直接返回
* **singletonObjects** 中获取不到，判断是否在创建中 **singletonsCurrentlyInCreation**，如果不在创建中，则直接返回 Null；否则就是在创建中，继续从缓存中获取
    * **LOCK**，**singletonObjects** 作为锁
    * **earlySingletonObjects** 中获取提前曝光的 不完整的 bean，能获取到则直接返回；获取不到，根据传入的条件 **allowEarlyReference** 是否允许提前引用，这里是 **true**，可以，继续从缓存中获取
    * **singletonFactories** 获取其 **ObjectFactory**
        * 如果 **ObjectFactory** 获取不到，则直接返回 Null，显然是不可能的，已经标识正在创建中，同时也是用了 **singletonObjects** 作为锁，就标识肯定可以从缓存中获取到
        * 调用 **ObjectFactory.getObject();** 方法进行实例化操作
        * 添加 bean 到 **earlySingletonObjects** 中
        * 从 **singletonFactories** 中移除对应的 ObjectFactory



 ## <span id="getSingleton2">getSingleton(): 实例化单例 bean</span>
* 方法全局加锁：
    * `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#singletonObjects`
* beforeSingletonCreation(), 加载前置处理;
    * **singletonsCurrentlyInCreation** 添加 beanName 正在实例化标识
* `singletonFactory.getObject();`, 实例化 bean，ObjectFactory 是 [**doGetBean()** 方法中创建的匿名内部类](#doGetBean_getSingleton2_ObjectFactory)
    * 真正调用的还是 [`org.springframework.beans.factory.support.AbstractBeanFactory#createBean()`](#createBean)
* afterSingletonCreation(), 加载后置处理;
    * **singletonsCurrentlyInCreation** 移除 beanName 正在实例化标识
* addSingleton(), 加入缓存中
    * 方法全局加锁，和 getSingleton() 方法用的同一个对象，**singletonObjects**
    * **singletonObjects**：+ beanName -> singletonObject，singletonObject 可能是对象实例，也可能是 factoryBean
    * **singletonFactories**：- beanName -> ObjectFactory，TODO
    * **earlySingletonObjects**：- beanName -> Object，TODO
    * **registeredSingletons**：+ beanName




 ## <span id="getObjectForBeanInstance">getObjectForBeanInstance(): 实例化为目标 beanInstance</span>
 * 代码逻辑优化，根据获取条件，以及 [getSingleton() 实例化单例 bean](#getSingleton2) 中实例化的共享单例bean **sharedInstance** 的情况，提前返回
    * name 指定需要的是 FactoryBean 时，beanInstance 为 Null，则直接返回 Null
    * name 指定需要的是 FactoryBean 时，beanInstance 是 FactoryBean，直接返回。
    * beanInstance 不是 FactoryBean，肯定是需要的单例对象，直接返回 beanInstance
* 基于上面的逻辑，下面逻辑里的 beanInstance 肯定是 FactoryBean
* getCachedObjectForFactoryBean()：若 BeanDefinition 为 null，则从缓存中加载 Bean 对象，能获取到 beanName 对应的 FactroyBean 生成的 beanInstance，则直接返回。
    * **factoryBeanObjectCache**：beanName -> Object，TODO
* **synthetic**：是否是用户定义的，而不是应用程序本身定义的，判断在下面方法 getObjectFromFactoryBean() 方法中是否需要处理**后续处理**
    * `boolean synthetic = (mbd != null && mbd.isSynthetic());`
    * **synthetic == true**：用户自定义的，需要处理**后续处理**
    * **synthetic == false**：系统内定义的，如：TODO，不需要处理**后续处理**
* **getObjectFromFactoryBean()** 方法：核心处理方法，使用 FactoryBean 获得 Bean 对象
    * factory 为单例模式 且 **singletonObjects** 缓存中存在，下面也只介绍这一种
    * 逻辑加锁，同样也是 **singletonObjects**
    * 在锁逻辑中再从缓存中获取一遍，**factoryBeanObjectCache**，如果获取到，则直接返回；获取不到，为 Null，则自己通过 factory 来获取
    * **doGetObjectFromFactoryBean()**：从 FactoryBean 中获取对象
        * `object = factory.getObject();`：从 FactoryBean 中，获得 Bean 对象
        * 返回 **object** 为 Null，且 **singletonsCurrentlyInCreation** 中已经标识在创建中，则抛出 **BeanCurrentlyInCreationException**
    * 再从缓存中获取一遍，**factoryBeanObjectCache**，如果获取到，则直接返回；获取不到，为 Null，则自己通过 factory 来获取，疑问？？？，TODO
    * 根据前面 **synthetic** 的标识，**shouldPostProcess = !synthetic**，来判断是否需要**后续处理**
        * **singletonsCurrentlyInCreation** 中标识了 beanName 正在创建，则直接返回。不处理下面逻辑
        * **beforeSingletonCreation()**，逻辑同上，在 **singletonsCurrentlyInCreation** 中标识 beanName 正在创建中
        * **postProcessObjectFromFactoryBean()**：对从 FactoryBean 获取的对象进行后处理，生成的对象将暴露给 bean 引用
            * 最终处理逻辑在 `AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization` 中，执行 **BeanPostProcessor#postProcessAfterInitialization()** 方法
        * **afterSingletonCreation()**，逻辑同上，在 **singletonsCurrentlyInCreation** 中移除 beanName 正在实例化标识
    * **singletonObjects** 中包含 beanName，添加到 **factoryBeanObjectCache** 中






## <span id="createBean">createBean(): BeanDefinition 实例化为 beanInstance</span>
* `Class<?> resolvedClass = resolveBeanClass(mbd, beanName)`：
    * 如果获取的class 属性不为null，则克隆该 BeanDefinition，主要是因为该动态解析的 class 无法保存到到共享的 BeanDefinition
* `mbdToUse.prepareMethodOverrides()`：验证和准备覆盖方法
* `Object bean = resolveBeforeInstantiation(beanName, mbdToUse)`：实例化的前置处理
    * 代码实现如下
    ```java
    @Nullable
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            // Make sure bean class is actually resolved at this point.
            if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // 前置
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        // 后置
                        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }
            mbd.beforeInstantiationResolved = (bean != null);
        }
        return bean;
    }

    @Nullable
    protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
                if (result != null) {
                    return result;
                }
            }
        }
        return null;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
            throws BeansException {
        // 遍历 BeanPostProcessor
        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            // 处理
            Object current = processor.postProcessAfterInitialization(result, beanName);
            // 返回空，则返回 result
            if (current == null) {
                return result;
            }
            // 修改 result
            result = current;
        }
        return result;
    }
    ```
    * 主要实现逻辑
        * **applyBeanPostProcessorsBeforeInstantiation()** 方法：`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation()`
        * **applyBeanPostProcessorsAfterInitialization()** 方法：`BeanPostProcessor#postProcessAfterInitialization()`
    * 如果**实例化前的前置处理**返回的 bean 不为 Null，则执行**实例化后的后置处理**
* 如果**resolveBeforeInstantiation()**方法中**实例化前的前置处理**和**实例化后的后置处理**返回的 bean 都不为 Null；即，**resolveBeforeInstantiation()**方法返回的 bean 不为 Null，则直接返回，不执行下面的 bean 实例化方法
* [`Object beanInstance = doCreateBean(beanName, mbdToUse, args)`](#doCreateBean)： 实例化 Bean

## <span id="doCreateBean">doCreateBean(): 实例化 bean</span>
* 单例模式下：`instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);`
    * 从未完成的 FactoryBean 缓存中删除
* **instanceWrapper == null**，**factoryBeanInstanceCache** 中不存在；TODO
    * [**createBeanInstance()**](#createBeanInstance)：使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化
* 判断是否有后置处理，如果有后置处理，则允许后置处理修改 BeanDefinition
    * 逻辑加锁：**RootBeanDefinition#postProcessingLock**
    * `!mbd.postProcessed`，逻辑判断
    * **applyMergedBeanDefinitionPostProcessors()**：后置处理修改 BeanDefinition
    * `mbd.postProcessed = true;`，逻辑标识
* 解决单例模式的循环依赖，从代码逻辑上可以看到，只有 `SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference()` 返回提前曝光的 bean，TODO
    * `boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&isSingletonCurrentlyInCreation(beanName));`
    * **earlySingletonExposure**：需要提前曝光
        * `addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));`：这两有两个主要逻辑**addSingletonFactory()** 和 **getEarlyBeanReference()**
        * 先说 **addSingletonFactory()** 方法逻辑
            * **LOCK**方法全局加锁：**singletonObjects**
            * **singletonFactories**: + beanName -> ObjectFactory
            * **earlySingletonObjects**: - beanName -> Object
            * **registeredSingletons**: + beanName
        * 再聊 **getEarlyBeanReference()** 方法逻辑
            * 代码实现如下
            ```java
            protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
                Object exposedObject = bean;
                if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                    for (BeanPostProcessor bp : getBeanPostProcessors()) {
                        if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                            exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                        }
                    }
                }
                return exposedObject;
            }
            ```
            * 通过代码可以发现是，`SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference()` 方法返回的需要提前曝光的 bean
* [`populateBean(beanName, mbd, instanceWrapper);`](#populateBean): 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性，则会递归初始依赖
* [`exposedObject = initializeBean(beanName, exposedObject, mbd);`](#initializeBean): 调用初始化方法
* **earlySingletonExposure**: true，上面提前曝光了，则需要解决**循环依赖处理**
* `registerDisposableBeanIfNecessary()`: 注册 bean




## <span id="createBeanInstance">createBeanInstance(): 创建 beanInstance</span>
* 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
    ```java
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }
    ```
* 使用 FactoryBean 的 factory-method 来创建，支持静态工厂和实例工厂，[`instantiateUsingFactoryMethod(beanName, mbd, args);`](#instantiateUsingFactoryMethod)
    ```java
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    ```
* 判断**RootBeanDefinition**是否解析过，重新创建同一个bean时，则通过解析过的结果来实例化 beanInstance，并返回
    * autowire 自动注入，调用构造函数自动注入，[`autowireConstructor(beanName, mbd, null, null);`](#autowireConstructor)
    ```java
    if (autowireNecessary) {
        return autowireConstructor(beanName, mbd, null, null);
    }
    ```
    * 使用默认构造函数构造: [`instantiateBean(beanName, mbd);`](#instantiateBean)
* 自动装配的候选构造者
    * 方法：`Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);`
    * 主要是检查已经注册的 **SmartInstantiationAwareBeanPostProcessor**
    * 有参数情况时，创建 Bean 。先利用参数个数，类型等，确定最精确匹配的构造方法。[`autowireConstructor(beanName, mbd, ctors, args);`](#autowireConstructor)
    ```java
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    ```
* 默认构造的首选构造函数
    * 方法：`ctors = mbd.getPreferredConstructors();`
    * 返回 **ctors** 不为 Null，则通过构造函数来实例化 beanInstance, [`autowireConstructor(beanName, mbd, ctors, null);`](#autowireConstructor)
* 有参数时，又没获取到构造方法，则只能调用无参构造方法来创建实例了(兜底方法)
    *  [`instantiateBean(beanName, mbd);`](#instantiateBean)


## <span id="populateBean">populateBean(): 填充 bean 属性值</span>
> 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性
* <1> ，根据 hasInstantiationAwareBeanPostProcessors 属性来判断，是否需要在注入属性之前给 InstantiationAwareBeanPostProcessors 最后一次改变 bean 的机会。此过程可以控制 Spring 是否继续进行属性填充。
    * 遍历 **InstantiationAwareBeanPostProcessor** 集合，如果其中某一个 `ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)` 方法返回 false，则终止填充操作
* 如果没有终止填充操作的标识，那就继续进行填充操作
* 统一存入到 PropertyValues 中，PropertyValues 用于描述 bean 的属性。
* <2> ，根据注入类型( AbstractBeanDefinition#getResolvedAutowireMode() 方法的返回值 )的不同来判断：
    * 是根据名称来自动注入（#autowireByName(...)）
    * 还是根据类型来自动注入（#autowireByType(...)）
* <3> ，进行 BeanPostProcessor 处理。**InstantiationAwareBeanPostProcessor** 类型
* <4> ，依赖检测。
* <5> ，将所有 PropertyValues 中的属性，填充到 BeanWrapper 中。[`applyPropertyValues(beanName, mbd, bw, pvs);`](#applyPropertyValues)




## <span id="initializeBean">initializeBean(): 调用初始化方法</span>
* 激活 Aware 方法。对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
    * `invokeAwareMethods(beanName, bean);`
    ```java
    private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			// BeanNameAware
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			// BeanClassLoaderAware
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			// BeanFactoryAware
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
    ```
* 后置处理器。before: `wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);`
    ```java
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		// 遍历 BeanPostProcessor 数组
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			// 处理
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			// 返回空，则返回 result
			if (current == null) {
				return result;
			}
			// 修改 result
			result = current;
		}
		return result;
	}
    ```
* 激活自定义的 init 方法。`invokeInitMethods(beanName, wrappedBean, mbd);`, `afterPropertiesSet(...)` 和 自定义的 **init-method** 方法
    ```java
	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {
		// 首先会检查是否是 InitializingBean ，如果是的话需要调用 afterPropertiesSet()
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            // 取出不关键的步骤
            // <1> 属性初始化的处理
            ((InitializingBean) bean).afterPropertiesSet();
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			// 判断是否指定了 init-method()，
			// 如果指定了 init-method()，则再调用制定的init-method
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				// <2> 激活用户自定义的初始化方法
				// 利用反射机制执行
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
    ```
* 后处理器，after: `wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);`
    ```java
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {
		// 遍历 BeanPostProcessor
		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			// 处理
			Object current = processor.postProcessAfterInitialization(result, beanName);
			// 返回空，则返回 result
			if (current == null) {
				return result;
			}
			// 修改 result
			result = current;
		}
		return result;
	}
    ```


## <span id="instantiateUsingFactoryMethod">instantiateUsingFactoryMethod(): 使用工厂方法实例化</span>
> `AbstractAutowireCapableBeanFactory#instantiateUsingFactoryMethod()` 和 `AbstractAutowireCapableBeanFactory#autowireConstructor` 方法第一步都是实例化 **ConstructorResolver** 对象
> 代码实现如下：
> ```java
> protected BeanWrapper instantiateUsingFactoryMethod(
>         String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
> 
>     return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
> }
> 
> protected BeanWrapper autowireConstructor(
>         String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {
> 
>     return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
> }
> ```
> 实例化 **ConstructorResolver** 对象代码如下：
> ```java
> public ConstructorResolver(AbstractAutowireCapableBeanFactory beanFactory) {
>     this.beanFactory = beanFactory;
>     this.logger = beanFactory.getLogger();
> }
> ```
因此，可以发现真正处理 [**instantiateUsingFactoryMethod()**](#instantiateUsingFactoryMethod) 和 [**autowireConstructor()**](#autowireConstructor) 逻辑的都在 **ConstructorResolver** 对象中。
下面看看 `ConstructorResolver#instantiateUsingFactoryMethod` 的处理逻辑。
* 初始化 **BeanWrapperImpl**
    ```java
    // 构造 BeanWrapperImpl 对象
    BeanWrapperImpl bw = new BeanWrapperImpl();
    // 初始化 BeanWrapperImpl
    // 向BeanWrapper对象中添加 ConversionService 对象和属性编辑器 PropertyEditor 对象
    this.beanFactory.initBeanWrapper(bw);
    ```
* 获得 factoryBean、factoryClass、isStatic、factoryBeanName 属性
    * **factoryBeanName** 不为 Null
        * `factoryBean = this.beanFactory.getBean(factoryBeanName);`: 会调用 `doGetBean()` 方法进行实例化
		* `factoryClass = factoryBean.getClass();`
		* `isStatic = false;`
	* **factoryBeanName** 为 Null
        * `factoryBean = null;`
        * `factoryClass = mbd.getBeanClass();`
        * `isStatic = true;`
* 获得 factoryMethodToUse、argsHolderToUse、argsToUse 属性
    * 先尝试从缓存中获取，如果 **RootBeanDefinition** 已经解析过，则从缓存中解析即可
        * 逻辑加锁: `mbd.constructorArgumentLock`
        * 从 **RootBeanDefinition** 中获取已经解析好的结果，并对需要动态解析的参数进行解析
    * 缓存中获取不到，`factoryMethodToUse == null || argsToUse == null`，则开始解析配置文件
        * `factoryClass = ClassUtils.getUserClass(factoryClass);`: 获取原始的 **Class** 类型，**factoryClass** 可能是 **CGLB** 生成的子类
        * 检索所有方法
            * 如果**候选方法**中只有 1 个，并且带解析的方法也不需要参数，唯一的**无参方法**来进行缓存
        * `AutowireUtils.sortFactoryMethods(candidates);`: 排序工厂方法
        * 解析构造函数的参数,将该 bean 的构造函数参数解析为 resolvedValues 对象，其中会涉及到其他 bean
            * 方法：`minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);`
        * 遍历 candidates 数组，选择一个最优的工厂方法
            * 构建 **ArgumentsHolder** 对象 **argsHolder**
                * `explicitArgs != null`: `#getBean(...)` 传递了参数，则使用参数列表与需要的一致的
            * 再根据 **argsHolder** 计算 **typeDiffWeight**: 类型差异权重
            * 根据 **typeDiffWeight** 选择最优的工厂方法，**typeDiffWeight** 越小，越优
            * 如果 **typeDiffWeight** 大小一致，判断是否需要添加到 **ambiguousFactoryMethods** 集合中
        * `explicitArgs == null && argsHolderToUse != null`: 解析出了 **argsHolderToUse** 并且 `#getBean(...)` 没有传递参数，即，从配置文件中解析的匹配的结果，将解析结果进行缓存
* 创建 Bean 对象，并设置到 bw 中
    * 方法：`bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, factoryMethodToUse, argsToUse));`
    * 因为上面如果 **factoryBeanName** 为 Null 时，会出现 **factoryBean** 为 Null 的情况，所以调用 **instantiate()** 方法时，**factoryBean** 可能是 Null，但是后面 `java.lang.reflect.Method#invoke(java.lang.Object, java.lang.Object...)` 反射方法中，目标类为 Null 时，会抛出 `java.lang.NullPointerException` 空指针异常。疑问？？？ TODO



## <span id="autowireConstructor">autowireConstructor(): 使用构造函数方法来实例化</span>
主要逻辑和上面 [`instantiateUsingFactoryMethod(...)`](#instantiateUsingFactoryMethod) 方法一致。
逻辑的都在 **ConstructorResolver** 对象中。
下面看看 `ConstructorResolver#autowireConstructor` 的处理逻辑。

* 初始化 **BeanWrapperImpl**
    ```java
    // 构造 BeanWrapperImpl 对象
    BeanWrapperImpl bw = new BeanWrapperImpl();
    // 初始化 BeanWrapperImpl
    // 向BeanWrapper对象中添加 ConversionService 对象和属性编辑器 PropertyEditor 对象
    this.beanFactory.initBeanWrapper(bw);
    ```
* 获得 constructorToUse、argsHolderToUse、argsToUse
* 内部代码前后逻辑有些优化，前后字段可能会导致前后逻辑的优化，因此这里将条件 `explicitArgs != null` 作为一个大的分类进行分析
* `explicitArgs != null`
    * **true**: `getBean(...)` 传递构造参数列表，一定会实时解析配置文件，没有**缓存构造函数**的逻辑，可能效率不是很高，但是也不一定，毕竟参数列表长度和类型确定了，构造函数解析肯定比模糊匹配来的高
        * **argsToUse** 很简单的就确定了，但是 **constructorToUse** 还需要实时取解析，下面就开始从配置文件中解析 **constructorToUse**
        * **candidates**: 获取候选的构造函数
        * `AutowireUtils.sortConstructors(candidates);`: 排序构造函数，public 构造函数优先参数数量降序，非public 构造函数参数数量降序
        * 迭代所有候选构造函数 **candidates**: 选择出匹配的构造函数
            * `constructorToUse != null && argsToUse != null && argsToUse.length > paramTypes.length`: **true**。终止整个循环，不在继续选择构造函数了
                * 从上面 `AutowireUtils.sortConstructors(candidates);` 可以知道，public 构造函数优先参数数量降序，非public 构造函数参数数量降序
                * `argsToUse != null`: 这里是 `getBean(...)` 传递构造参数列表，已经明确了，肯定是 **true**
                * `argsToUse.length > paramTypes.length`: 在构造函数的排序规则中，后面的构造器要么一定比需要的参数离别长度小，要么就是在 非 public 中还存在参数列表一直的构造函数
                * `constructorToUse != null`: 说明已经经过了下面的解析逻辑，已经优先匹配到了构造函数，后面的 **argsHolder** 计算出来的匹配度，可能还不如当前值
            * `paramTypes.length != explicitArgs.length`: 参数列表长度与需要的参数列表长度不一致，则直接跳过
            * 参数列表一致，参数列表 **explicitArgs** 构造成 **ArgumentsHolder** 对象 **argsHolder**
            * 再根据 **argsHolder** 计算 **typeDiffWeight**: 类型差异权重
            * 根据 **typeDiffWeight** 选择最优的构造方法，**typeDiffWeight** 越小，越优
            * 如果 **typeDiffWeight** 大小一致，判断是否需要添加到 **ambiguousConstructors** 集合中
    * **false**: `getBean(...)` 未传递构造参数列表，有**缓存构造函数**的逻辑，解析后会缓存**解析好的构造函数**，下次再实例化的时候就可以从缓存中获取。但是，如果需要实例化的 bean 都是单例的，缓存的结果，事实上并没有用到，除非 bean 的 **scope** 定义为 **prototype** 原型模式，才会有优化的效果。
        * 尝试从缓存中获取，如果 **RootBeanDefinition** 已经解析过，则从缓存中解析即可
        * 逻辑加锁: `mbd.constructorArgumentLock`
        * 从 **RootBeanDefinition** 中获取已经解析好的结果，并对需要动态解析的参数进行解析
            * `constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;`: 构造方法
            * `argsToUse = mbd.resolvedConstructorArguments;`: 已经解析好的构造参数
            * `argsToResolve = mbd.preparedConstructorArguments;`: 需要动态解析的构造参数
                * `argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);`: 动态解析
            * TODO，这里应该已经解析好了构造参数，**constructorToUse** 和 **argsToUse** 应该已经都不为 Null，应该就不会再去执行下面的通过配置文件来解析的逻辑，直接跳过配置文件解析逻辑，进行实例化操作
        * `constructorToUse == null || argsToUse == null`: **true**，说明在缓存中没有获取到，需要通过配置文件来进行解析
            * **candidates**: 获取候选的构造函数
            * 如果候选构造函数 **candidates** 只有**唯一**且是**无参无参构造器**，并且 **RootBeanDefinition** 中也没有定义构造参数，则说明已经匹配上了，直接将当前的无参构造器缓存，并使用无参构造器实例化对象，由此可见，定义 bean 的时候使用无参构造器实例化，可以更快实例化对象
            * 从 **RootBeanDefinition** 中获取构造参数，并解析成 **resolvedValues**
            `AutowireUtils.sortConstructors(candidates);`: 排序构造函数，public 构造函数优先参数数量降序，非public 构造函数参数数量降序
            * 迭代所有候选构造函数 **candidates**: 选择出匹配的构造函数
                * `constructorToUse != null && argsToUse != null && argsToUse.length > paramTypes.length`: **true**。终止整个循环，不在继续选择构造函数了
                    * 从上面 `AutowireUtils.sortConstructors(candidates);` 可以知道，public 构造函数优先参数数量降序，非public 构造函数参数数量降序
                    * `argsToUse.length > paramTypes.length`: 在构造函数的排序规则中，后面的构造器要么一定比需要的参数离别长度小，要么就是在 非 public 中还存在参数列表一直的构造函数
                    * `constructorToUse != null`: 说明已经经过了下面的解析逻辑，已经优先匹配到了构造函数，后面的 **argsHolder** 计算出来的匹配度，可能还不如当前值
                    * `argsToUse != null`: 逻辑同 **constructorToUse** 一样
                * `paramTypes.length < minNrOfArgs` 时，则跳过当前构造器
                    * 跳过可以理解，当前构造器参数比最少需要的构造参数列表还少，肯定需要跳过
                    * 但是和上面的 **break** 终止循环的差异在哪呢？？？
                        * 猜想1: 上面 **break** 逻辑中，**argsToUse** 已经判断出了不为 Null，**argsToUse** 已经确定了，只要再确定出 **constructorToUse** 即可，但是这里循环判断中 **argsToUse** 可能还是 Null，还需要从多个构造参数列表中选择一个更优的构造函数，所以还需要进行下去
                * `argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);`: 构造出 **argsHolder**，这里单独在看，TODO
                * 参数列表一致，参数列表 **explicitArgs** 构造成 **ArgumentsHolder** 对象 **argsHolder**
                * 再根据 **argsHolder** 计算 **typeDiffWeight**: 类型差异权重
                * 根据 **typeDiffWeight** 选择最优的构造方法，**typeDiffWeight** 越小，越优
                * 如果 **typeDiffWeight** 大小一致，判断是否需要添加到 **ambiguousConstructors** 集合中
* 创建 Bean 对象，并设置到 bw 中
    * 方法：`bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));`


## <span id="instantiateBean">instantiateBean(): 默认无参构造函数来实例化</span>
代码实现逻辑如下：
```java
beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
```
真正实现逻辑的还是 `SimpleInstantiationStrategy#instantiate(...)`
* `!bd.hasMethodOverrides()`: 是否包含重写的方法
    * **true**: 不包含，则直接使用无参构造器进行实例化，并将 **resolvedConstructorOrFactoryMethod** 指定为**无参构造器**
    * **false**: 使用 CGLIB 创建的子类对象，`instantiateWithMethodInjection(bd, beanName, owner);`





## <span id="ConstructorResolver_instantiate1">`ConstructorResolver#instantiate(...)`: 实例化工厂方法</span>
内部调用的还是 `SimpleInstantiationStrategy#instantiate(...)` 方法，代码如下:
```java
return this.beanFactory.getInstantiationStrategy().instantiate(
    mbd, beanName, this.beanFactory, factoryBean, factoryMethod, args);
```



## <span id="ConstructorResolver_instantiate2">`ConstructorResolver#instantiate(...)`: 实例化构造方法</span>
TODO...


## <span id="ConstructorResolver_createArgumentArray">`ConstructorResolver#createArgumentArray(...)`: 包装成 ArgumentsHolder</span>
* 获取类型转换器 **converter**: 后面参数类型需要转换的时候使用其进行转换，有限使用用户自定义的类型转换器，用户没有自定义时，使用 **BeanWrapper** 作为类型转换器
* 实例化 **ArgumentsHolder**
    * **rawArguments**: 记录原始 value
    * **arguments**: 记录转换后的 value
    * **preparedArguments**: TODO，值有以下两个来源
        * **ConstructorArgumentValues.ValueHolder** 中 **source** 也是 **ConstructorArgumentValues.ValueHolder** 类型，其 **source** 中的 value 值，也就是原始值
        * 类型为 **AutowiredArgumentMarker** 的标识对象
* 遍历参数列表
    * 获取 **valueHolder**: 类型 **ConstructorArgumentValues.ValueHolder**
        * 条件 `resolvedValues != null`
        * `valueHolder = resolvedValues.getArgumentValue(paramIndex, paramType, paramName, usedValueHolders);`
            * 返回的 **valueHolder** 是在将配置文件解析 **BeanDefinition** 过程中，包装好的，TODO，后面讲跳转链接在这里配一下
    * `valueHolder != null`
        * 这里聊聊 `valueHolder == null`， 什么情况下才会为 null
            * 如果 `resolvedValues == null`，则不会进行上一步获取 **valueHolder** 的逻辑，**valueHolder** 肯定为 Null
            * 但是如果 `resolvedValues != null`，只有根据 **index**、**type** 和 **name** 都匹配不到时，才会为 Null，但是在 xml 文件解析判断的时候，已经做了判断，不可能都不写，因此可以理解为，如果 `resolvedValues != null`，走了上面接下 **valueHolder** 的逻辑，**valueHolder** 就一定不会为 Null
            ```xml
            <constructor-arg type="" index="" name="" value="" ref=""></constructor-arg>
            ```
        * **true**: **resolvedValues** 不为 Null
            * 将 **valueHolder** 添加到 **usedValueHolders** Set 集合中，注意，这里 **ConstructorArgumentValues.ValueHolder** 没有重写 hashCode 和 equals 方法，疑问？？？ TODO
            * 填充 **ArgumentsHolder** 中的参数
                * `valueHolder.isConverted()`: 逻辑优化，已经经过解析，直接从 **valueHolder** 中取出 **convertedValue** 来填充到 **preparedArguments** 和 **arguments** 中
                * 但是从代码的赋值引用中，**converted** 字段似乎一直都没有被修改过，因此就是默认值 **false**，这样是不是说这个逻辑永远都不会出现，疑问？？？待验证，TODO
                * **valueHoder** 中 **value** 没有转换时
                    * `MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);`: **executable** 传递的**构造器函数**或**方法函数**作为执行上线问，获取到对应的**方法参数** methodParam
                    * `convertedValue = converter.convertIfNecessary(originalValue, paramType, methodParam);`: 根据需要将原始值转换成需要的参数类型
                    * 如果 `valueHolder.source instanceof ConstructorArgumentValues.ValueHolder` 时，则将 **source.value** 取出，赋值给 **preparedArguments** 对应的索引，并标识方法返回值 **ArgumentsHolder.resolveNecessary = true** 
        * **false**: **resolvedValues** 为 Null
            * `MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);`: 逻辑同上，获取 **methodParam**
            * `Object autowiredArgument = resolveAutowiredArgument(methodParam, beanName, autowiredBeanNames, converter, fallback);`: 解析需要注入的参数，会涉及到 bean 的依赖关系，这个方法比较重要，后面再看看，TODO
            * **rawArguments** 和 **arguments** 上对应的索引都赋值为 **autowiredArgument**
            * **preparedArguments** 对应的索引上标识为 **AutowiredArgumentMarker**
            * **resolveNecessary** 标识为 **true**
* 对于需要注册依赖关系的 bean 进行循环依赖注册 `this.beanFactory.registerDependentBean(autowiredBeanName, beanName);`





## <span id="SimpleInstantiationStrategy_instantiate">`SimpleInstantiationStrategy#instantiate(...)`: 实例化方法</span>
* 设置 Method 可访问
    * 方法：`ReflectionUtils.makeAccessible(factoryMethod);`
* 创建 Bean 对象
    * 方法：`Object result = factoryMethod.invoke(factoryBean, args);`
    * 在调用方法前后，**currentlyInvokedFactoryMethod** 会将线程上下文的方法获取出来，调用后又填充回去







## <span id="applyPropertyValues">applyPropertyValues(...): 将属性应用到 bean 中</span>
* `pvs instanceof MutablePropertyValues` && `mpvs.isConverted()`: 已经解析了，并且缓存了，则直接调用 [`bw.setPropertyValues(mpvs);`] set 值
* 获取 **converter**
* 遍历 **PropertyValue**，将转换的值添加到 **deepCopy** 中
    * 如果 **pv** 已经转换，则直接添加到 **deepCopy** 中
    * **pv** 没有转换，则进行转换工作
        * 转换属性值，例如将引用转换为IoC容器中实例化对象引用 ！！！！！ 对属性值的解析！！
            * `Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);`
        * `convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);`
        * 添加到 **deepCopy**  中
* 标识 **pvs** 已经转换完成: 疑问？？？这里设置是有条件的，但是没看懂，会面再看看。TODO
* 进行属性依赖注入，依赖注入的真真正正实现依赖的注入方法在此！！！: `bw.setPropertyValues(new MutablePropertyValues(deepCopy));`



## <span id="setPropertyValues">setPropertyValues(...): 为实例化对象设置属性值 ，依赖注入真真正正地实现在此！！！！！</span>

最后反射填充值得时候，一种是通过 **field** 来填充，如下：
```java
ReflectionUtils.makeAccessible(this.field);
this.field.set(getWrappedInstance(), value);
```

另一种是通过方法填充，如下：
```java
ReflectionUtils.makeAccessible(writeMethod);
writeMethod.invoke(getWrappedInstance(), value);
```

在通过反射填充值之前，还是会进两个方法 
* `processKeyedProperty(...)`: 对数组、List 和 Map 类型进行填充，否则抛出错误
* `processLocalProperty(...)`: 直接注入转换值
    * 比较 **valueToApply** 和 **originalValue** 是否相等，标识出是否需要转换 **conversionNecessary**
        * 如果 **pv** 已经转换过了，则，直接取叫可以了
        * 否则，还需要再进行一次转换工作，`valueToApply = convertForProperty(tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());`
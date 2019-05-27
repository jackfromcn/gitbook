# (转)Spring源码解析(十二)Spring扩展接口SmartInstantiationAwareBeanPostProcessor解析

之前我们分析了 InstantiationAwareBeanPostProcessor、BeanPostProcessor、今天来分析一下SmartInstantiationAwareBeanPostProcessor的用法;

SmartInstantiationAwareBeanPostProcessor 继承自 InstantiationAwareBeanPostProcessor； 
但是SmartInstantiationAwareBeanPostProcessor多了一个三个方法
```java
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

// 预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
    Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;
// 选择合适的构造器，比如目标对象有多个构造器，在这里可以进行一些定制化，选择合适的构造器
// beanClass参数表示目标实例的类型，beanName是目标实例在Spring容器中的name
// 返回值是个构造器数组，如果返回null，会执行下一个PostProcessor的determineCandidateConstructors方法；否则选取该PostProcessor选择的构造器
    Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;
// 获得提前暴露的bean引用。主要用于解决循环引用的问题
// 只有单例对象才会调用此方法
    Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;
}
```

## getEarlyBeanReference
这个方法见名思意就是获取提前引用的意思了，Spring中解决循环引用的时候有调用这个方法, 
关于循环引用请看 [分析一个Spring循环引用失败的问题](springSourceCode_analysis_10_80301135.md)

但是我还是想再分析一下它的调用时机

## getEarlyBeanReference调用时机
准备两个类，让他们相互引用
```xml
 <bean id="circulationa" class="src.bean.CirculationA">
    <property name="circulationB" ref="circulationb"/>
  </bean>
  <bean id="circulationb" class="src.bean.CirculationB" >
    <property name="circulationA" ref="circulationa"/>
  </bean>
```

启动； 
1. 加载circulationa,然后将调用了代码,提前将singleto暴露出去，但是这个时候只是getEarlyBeanReference还没有被调用; 因为没有出现循环引用的情况;现在放入缓存是为了预防有循环引用的情况可以通过这个getEarlyBeanReference获取对象;
```java
// Eagerly cache singletons to be able to resolve circular references
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isDebugEnabled()) {
                logger.debug("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            addSingletonFactory(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    return getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }
```

2. 然后填充属性值;调用下面的方法，在填充属性的时候发现引用了circulationb；然后就去获取circulationb来填充
```java
populateBean(beanName, mbd, instanceWrapper);
```

3. 加载circulationb, 执行的操作跟 1,2一样; circulationb发现了引用了circulationa；然后直接调用getSingleton获取circulationa;
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        //这个地方就是调用getEarlyBeanReference的地方了;
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
```

这一步返回的就是 getEarlyBeanReference得到的值； 

4. 执行getEarlyBeanReference方法
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
        Object exposedObject = bean;
        if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                    SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                    exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                    if (exposedObject == null) {
                        return null;
                    }
                }
            }
        }
        return exposedObject;
    }
```

4.1 一般情况下，如果系统中没有SmartInstantiationAwareBeanPostProcessor接口；就是直接返回exposedObject什么也不做; 
4.2.所以利用SmartInstantiationAwareBeanPostProcessor可以改变一下提前暴露的对象;

5. 拿到引用了之后….就不分析了…..

## determineCandidateConstructors调用时机
检测Bean的构造器，可以检测出多个候选构造器，再有相应的策略决定使用哪一个，如AutowiredAnnotationBeanPostProcessor实现将自动扫描通过@Autowired/@Value注解的构造器从而可以完成构造器注入

### predictBeanType
预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null；当你调用BeanFactory.getType(name)时当通过Bean定义无法得到Bean类型信息时就调用该回调方法来决定类型信息；BeanFactory.isTypeMatch(name, targetType)用于检测给定名字的Bean是否匹配目标类型（如在依赖注入时需要使用）；

## 利用SmartInstantiationAwareBeanPostProcessor做点啥？
在Spring中默认实现了它的有两个实现类; 
AbstractAutoProxyCreator 
InstantiationAwareBeanPostProcessorAdapter；这个只是但是的实现了一下所有接口，但是都是直接返回并没有做什么事情； 
那我们主要分析一下AbstractAutoProxyCreator做了啥？

## AbstractAutoProxyCreator
TODO…..涉及到AOP 等下次分析AOP的时候再回来分析
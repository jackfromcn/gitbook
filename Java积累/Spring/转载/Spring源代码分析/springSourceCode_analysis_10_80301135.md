# (转)Spring源码解析(十)分析一个Spring循环引用失败的问题

## 前言:
之前我们有分析过Spring是怎么解决循环引用的问题,主要思路就是三级缓存；Spring在加载beanA的时候会先调用默认的空构造函数(在没有指定构造函数实例化的前提下)得到一个空的实例引用对象，这个时候没有设置任何值，但是Spring会用缓存把它给提前暴露出来，让其他依赖beanA的bean可以持有它提前暴露的引用；比如 a 依赖b ，b依赖a，并且他们都是通过默认方法实例化，那么简单流程是这样的： 
1. ioc实例化a,a提前暴露自己的，然后填充属性值，在填充属性值的时候发现有个对象b,这个时候去容器里面取到b的引用，发现b还没有被创建，那么就走实例化b的流程； 
2. 实例化b；流程跟a一样；但是不同的是b填充属性的时候，发现有引用a的实例，这个时候a已经提前暴露了自己了，所以b可以直接在容器里面拿到a的引用；那么b就实例化并且也初始化完成了; 
3. 拿到b了之后，a就可以持有b的引用 ，整个流程就走完了;

具体详细一点可以看这篇文章 [Spring-bean的循环依赖以及解决方式](https://blog.csdn.net/u010853261/article/details/77940767)

Spring不能解决“A的构造方法中依赖了B的实例对象，同时B依赖了A的实例对象”这类问题

这篇文章我想从源码的角度来分析一下整个流程；并且分析一下Spring为什么不能解决“A的构造方法中依赖了B的实例对象，同时B依赖了A的实例对象”这类问题

## 例子
首先创建两个bean类; 
CirculationA 
有个属性circulationB，并且有个构造函数给circulationB赋值；
```java
public class CirculationA {

    private CirculationB circulationB;

    public CirculationA(CirculationB circulationB) {
        this.circulationB = circulationB;
    }
}
```

CirculationB 
有个属性circulationA，然后set方法
```java
public class CirculationB {
    private CirculationA circulationA;

    public CirculationA getCirculationA() {
        return circulationA;
    }

    public void setCirculationA(CirculationA circulationA) {
        this.circulationA = circulationA;
    }
}
```

### SpringContextConfig.xml 
circulationa 用给定的构造函数实例化; 
circulationb 就用默认的实例化方法(默认的空构造函数)
```xml
  <bean id="circulationa" class="src.bean.CirculationA">
    <constructor-arg name="circulationB" ref="circulationb"/>
  </bean>
  <bean id="circulationb" class="src.bean.CirculationB" >
    <property name="circulationA" ref="circulationa"/>
  </bean>
```

好，例子准完毕，上面的例子是 circulationa的构造函数里面有circulationb；然后circulationb属性里面有circulationa; 
#### 启动容器！结果如下:
```bash
警告: Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'circulationa' defined in class path resource [config.xml]: Cannot resolve reference to bean 'circulationb' while setting constructor argument; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'circulationb' defined in class path resource [config.xml]: Cannot resolve reference to bean 'circulationa' while setting bean property 'circulationA'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'circulationa': Requested bean is currently in creation: Is there an unresolvable circular reference?
Exception in thread "main" org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'circulationa' defined in class path resource [config.xml]: Cannot resolve reference to bean 'circulationb' while setting constructor argument; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'circulationb' defined in class path resource [config.xml]: Cannot resolve reference to bean 'circulationa' while setting bean property 'circulationA'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'circulationa': Requested bean is currently in creation: Is there an unresolvable circular reference?
Disconnected from the target VM, address: '127.0.0.1:64128', transport: 'socket'
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:359)
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveValueIfNecessary(BeanDefinitionValueResolver.java:108)
    at org.springframework.beans.factory.support.ConstructorResolver.resolveConstructorArguments(ConstructorResolver.java:648)
    at org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:145)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1193)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1095)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:513)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:483)
    at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:306)
    at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
    at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
    at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:761)
    at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:867)
    at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543)
    at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:139)
    at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:83)
    at StartIOCUseDefaultListAbleBeanFactory.main(StartIOCUseDefaultListAbleBeanFactory.java:30)
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'circulationb' defined in class path resource [config.xml]: Cannot resolve reference to bean 'circulationa' while setting bean property 'circulationA'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'circulationa': Requested bean is currently in creation: Is there an unresolvable circular reference?
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:359)
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveValueIfNecessary(BeanDefinitionValueResolver.java:108)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyPropertyValues(AbstractAutowireCapableBeanFactory.java:1531)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1276)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:553)
    at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:483)
    at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:306)
    at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
    at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
    at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:351)
    ... 17 more
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'circulationa': Requested bean is currently in creation: Is there an unresolvable circular reference?
    at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.beforeSingletonCreation(DefaultSingletonBeanRegistry.java:347)
    at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:223)
    at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
    at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
    at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:351)
    ... 27 more
```

**报错了,Spring它解决不了这种情况**
**Ok,源码走起来**： 为了节省篇幅我只贴重要代码 
第一步,加载circulationa 
### AbstractBeanFactory
```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {

    protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            throws BeansException {
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            try {
                                return createBean(beanName, mbd, args);
                            }

                        }
                    });
                }
    }

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        //在创建之前把beanName加入到正在创建中的属性中singletonsCurrentlyInCreation；
        //但是这个是一个set，如果之前已经加进去了，再进去就抛异常BeanCurrentlyInCreationException
        //Requested bean is currently in creation: Is there an unresolvable circular reference?")提示可能存在循环引用
        beforeSingletonCreation(beanName);
    }
    protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }

    @Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    }
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
            throws BeanCreationException {
            instanceWrapper = createBeanInstance(beanName, mbd, args);
//.......
// Initialize the bean instance.
        Object exposedObject = bean;
        try {
            populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
                exposedObject = initializeBean(beanName, exposedObject, mbd);
            }
        }

}
    protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    //因为circulationa是有构造函数的，所以使用autowireConstructor
        if (ctors != null ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
                mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
            return autowireConstructor(beanName, mbd, ctors, args);
        }
    }
//最终执行
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
            Constructor<?>[] chosenCtors, final Object[] explicitArgs) {
            //解析构造函数参数值
                minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
                //......
//选择对应的策略来实例化对象；这里是生成正在的实例了。
//但是在这之前，构造参数要拿到
                beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
                        mbd, beanName, this.beanFactory, constructorToUse, argsToUse);

}


}
```

省略….最终调用BeanDefinitionValueResolver
```java
/**
     * Resolve a reference to another bean in the factory.
     */
    private Object resolveReference(Object argName, RuntimeBeanReference ref) {
        try {
            String refName = ref.getBeanName();
            refName = String.valueOf(doEvaluate(refName));
            if (ref.isToParent()) {
                if (this.beanFactory.getParentBeanFactory() == null) {
                    throw new BeanCreationException(
                            this.beanDefinition.getResourceDescription(), this.beanName,
                            "Can't resolve reference to bean '" + refName +
                            "' in parent factory: no parent factory available");
                }
                //!!!这里,要先去查找refName的实例
                return this.beanFactory.getParentBeanFactory().getBean(refName);
            }
            else {
                Object bean = this.beanFactory.getBean(refName);
                this.beanFactory.registerDependentBean(refName, this.beanName);
                return bean;
            }
        }
        catch (BeansException ex) {
            throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
        }
    }
```

跟着上面的顺序我们整理一下; 
1. 启动容器,加载circulationa,因为是构造函数生成，所以要先解析构造函数的属性，这时候发现有引用circulationb，那么通过getBean(circulationb)先拿到circulationb的实例； 
2. 如果拿到了，则生成circulationa的实例对象返回;但是这个时候代码执行circulationb的加载过程了；

然后我们分析一下circulationb加载 
circulationb跟circulationa差不多 
加载circulationb,把它加入到正在创建的属性中
```java
protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }
```

然后用默认的方式创建实例; 
circulationa 是rautowireConstructor(beanName, mbd, ctors, args)创建的；这个方法需要先拿到构造函数的值;所以执行了调用getBean(circulationb)

circulationa是调用了instantiateBean；这个方法不需要提前知道属性；它用默认的构造函数生成实例；这时候的实例是没有设置任何属性的；
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
                beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
```

不过生成了实例之后，在doCreateBean方法中有一个populateBean；这个方法就是专门填充属性值的，因为circulationb有circulationa的属性; 
所以会去容器里面取circulationa的引用；但是circulationa这个时候还没有成功创建实例啊；因为它还一直在等circulationb创建成功之后返回给它引用呢，返回了circulationa才能创建实例啊；

这个时候circulationb没有拿到circulationa，那么又会去调用getBean(circulationa); 
大家想一想如果这样下去就没完没了了啊; 
所以Spring就抛出异常了 
**那么在哪里抛出异常呢?**
在第二次调用getBean(circulationa)的时候会走到下面
```java
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }
```

因为circulationa之前加进来过一次啊，而且没有创建成功是不会删除的啊；现在又add一次，因为this.singletonsCurrentlyInCreation是一个set；已经存在的再次add会返回false;那么这段代码就会抛出异常了;
```bash
Error creating bean with name 'circulationa': Requested bean is currently in creation: Is there an unresolvable circular reference?
```
情况就是这样,只要是用构造函数创建一个实例，并且构造函数里包含的值存在循环引用，那么spring就会抛出异常; 
所以如果有循环引用的情况请避免使用构造函数的方式
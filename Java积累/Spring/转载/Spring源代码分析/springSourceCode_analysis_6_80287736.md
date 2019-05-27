# (转)Spring源码分析(六)FactoryBean 接口解析

说道FactoryBean，不少人会拿它跟BeanFactory作比较，但是实际上他们没有多大关系；我们简单介绍一下两者

## 一、BeanFactory和FactoryBean区别
## BeanFactory
> BeanFactory：这就是一个Factory，是一个IOC容器或者叫对象工厂，它里面存着很多的bean。例如默认的实现方式DefaultListableBeanFactory；我们把IOC容器可以比作一个水桶，IOC容器里面的所有bean就是装的水；

## FactoryBean
> FactoryBean：是一个Java Bean，但是它是一个能生产对象的工厂Bean，把IOC容器比作水桶，那么Java Bean就是水桶里面的水，但是这个FactoryBean一种比较特殊的水，可以把它看成是一个水球，这个水球里面也包含了水，我们可以通过IOC取这个水球，但是也可以直接取水球里面的水;(用法就是 用&表示取水球，取水球里的水用正常的获取方式就行了；如果我们用&获取到了水球之后，可以通过这个水球的 getObject方法获取水球里面的水;)

在之前的文章中我们已经分析过BeanFactory的源码了，今天我们单独分析FactoryBean

## 代码入口
```java
  public static void main(String[] args){
         ClassPathResource resource = new ClassPathResource("SpringContextConfig.xml");
        DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
        reader.loadBeanDefinitions(resource);
        MyFactoryBean beanA = (MyFactoryBean)factory.getBean("&fb");
        TestFb beanB = (TestFb)factory.getBean("fb");
        TestFb beanC =beanA.getObject();
        System.out.print(beanB==beanC);
        System.out.print(beanA);
    }
```

### AbstractBeanFactory
> AbstractBeanFactory extends FactoryBeanRegistrySupport

断点进去最后访问到AbstractBeanFactory的getObjectForBeanInstance方法；方法中的几个参数分别表示 
beanInstance: 
通过Object sharedInstance = getSingleton(beanName);获取到的，这个就是实例化的对象，就算用我们getBean的时候beanName传的是fb，不是&fb，这里返回的就是FactoryBean的实例对象;就是上面比喻的水球; 
name:就是传进来的name ；没有过滤&字符的； 
beanName:过滤了&字符的
<span id="getObjectForBeanInstance"></span>
```java
    /**专门缓存从FactoryBeans中生成的对象的；除了第一次要getObject()，后面直接从这个mao中取就行了 
    Cache of singleton objects created by FactoryBeans: FactoryBean name --> object */
    private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<String, Object>(16);


protected Object getObjectForBeanInstance(
            Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

        // 如果name 是&开头的，但是beanInstance又不是FactoryBean类型的就抛出异常
        if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
        }


        //如果不是FactoryBean 例如name=fb；直接返回
        //如果name是&开头并且FactoryBean类型的例如name=&fb,fb也是FactoryBean类型； 直接返回
        if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
            return beanInstance;
        }
    // beanInstance是FactoryBean类型的并且name不是是以&开头的;例如name=fb 就要走下面的流程
    //beanInstance不是FactoryBean类型并且name
        Object object = null;
        if (mbd == null) {
        //检查之前是否已经生成放入缓存了，如果有 直接返回
            object = getCachedObjectForFactoryBean(beanName);
        }
        if (object == null) {
            // Return bean instance from factory.
            FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
            // 获取合并之后的BeanDefinition
            if (mbd == null && containsBeanDefinition(beanName)) {
                mbd = getMergedLocalBeanDefinition(beanName);
            }
            //TODO....暂时不知道这里是怎么设置的，回头分析
            boolean synthetic = (mbd != null && mbd.isSynthetic());
            /**
            *1.返回FactoryBean.getObject()对象
            *2.执行后置处理器接口的方法BeanPostProcessor中的
            *postProcessAfterInitialization(result, beanName);方法；
            */
            object = getObjectFromFactoryBean(factory, beanName, !synthetic);
        }
        return object;
    }
```

### FactoryBeanRegistrySupport
```java
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry{

    /**
     * 从FactoryBean中获取Bean
     * @param factory the FactoryBean instance
     * @param beanName the name of the bean
     * @param shouldPostProcess whether the bean is subject to post-processing
     * @return the object obtained from the FactoryBean
     * @throws BeanCreationException if FactoryBean object creation failed
     * @see org.springframework.beans.factory.FactoryBean#getObject()
     */
    protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    //判断factory是单例的并且beanName已经被创建
        if (factory.isSingleton() && containsSingleton(beanName)) {
            synchronized (getSingletonMutex()) {
            //如果已经创建过直接返回
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    object = doGetObjectFromFactoryBean(factory, beanName);
                    // Only post-process and store if not put there already during getObject() call above
                    // (e.g. because of circular reference processing triggered by custom getBean calls)
                    //TODO....这里回头分析...
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    }
                    else {
                        if (object != null && shouldPostProcess) {
                            try {
                            //执行所有后置处理器接口的方法BeanPostProcessor中的postProcessAfterInitialization(result, beanName);方法；
                            //这里单独写一篇文章将后置处理器BeanPostProcessor
                                object = postProcessObjectFromFactoryBean(object, beanName);
                            }
                            catch (Throwable ex) {
                                throw new BeanCreationException(beanName,
                                        "Post-processing of FactoryBean's singleton object failed", ex);
                            }
                        }
    //将通过factoryBean.getObject方法得到的对象存到 factoryBeanObjectCache中         this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
                    }
                }
                return (object != NULL_OBJECT ? object : null);
            }
        }
        else {//如果不是单例对象
        //调用FactoryBean的getObject方法返回实例，也就是从水球里面取出水；
            Object object = doGetObjectFromFactoryBean(factory, beanName);
            if (object != null && shouldPostProcess) {
                try {
                //执行后置处理器
                    object = postProcessObjectFromFactoryBean(object, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
                }
            }
            return object;
        }
    }

    //调用FactoryBean的getObject放法
    private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
            throws BeanCreationException {

        Object object;
        try {
            if (System.getSecurityManager() != null) {
                AccessControlContext acc = getAccessControlContext();
                try {
                    object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        @Override
                        public Object run() throws Exception {
                                return factory.getObject();
                            }
                        }, acc);
                }
                catch (PrivilegedActionException pae) {
                    throw pae.getException();
                }
            }
            else {
                //可以看到最终调用了getObject方法；这是需要实现类自己实现
                object = factory.getObject();
            }
        }
        catch (FactoryBeanNotInitializedException ex) {
            throw new BeanCurrentlyInCreationException(beanName, ex.toString());
        }
        catch (Throwable ex) {
            throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
        }

        // Do not accept a null value for a FactoryBean that's not fully
        // initialized yet: Many FactoryBeans just return null then.
        if (object == null && isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(
                    beanName, "FactoryBean which is currently in creation returned null from getObject");
        }
        return object;
    }
```

二、总结
总结一下整个FactoryBean源码的流程 
1.判断是否需要调用FactoryBean的getObject方法来获取实例 
判断依据是:①.如果name有前缀&，直接返回 
②.如果没有前缀&，调用getObject方法获取实例 
2.如果是通过调用GetObject方法获取实例，则还要执行一下后置处理器 
执行后置处理器接口的方法BeanPostProcessor中的 
postProcessAfterInitialization(result, beanName);方法；

## 三、分析过程有一些需要单独分析，占个坑
1. mbd.isSynthetic()中的值是什么时候设置的？有什么用？
TODO…

2. BeanPostProcessor后置处理器讲解
[BeanPostProcessor解析](springSourceCode_analysis_7_80289441.md)

3. 上面Object alreadyThere = this.factoryBeanObjectCache.get(beanName);这段代码的意思？什么情况下要先查这个？
TODO…..

[说明文字](#getObjectForBeanInstance)
# (转)Spring源码分析(九)lazy-init 在Spring中是怎么控制加载的

## 一、lazy-init说明:
ApplicationContext实现的默认行为就是在启动时将所有singleton bean提前进行实例化（也就是依赖注入）。提前实例化意味着作为初始化过程的一部分，ApplicationContext实例会创建并配置所有的singleton bean。通常情况下这是件好事，因为这样在配置中的任何错误就会即刻被发现（否则的话可能要花几个小时甚至几天）。
```xml
<bean id="testBean" class="com.fhx.TestBean">
```

该bean默认的设置为：
```xml
<bean id="testBean" class="com.fhx.TestBean" lazy-init="false">
```
lazy-init="false" 立退加载， 表示spring启动时，立刻进行实例化。
（lazy-init 设置只对scop属性为singleton的bean起作用）

有时候这种默认处理可能并不是你想要的。如果你不想让一个singleton bean在ApplicationContext实现在初始化时被提前实例化，那么可以将bean设置为延迟实例化。

, lazy-init=”true”> 延迟加载 ,设置为lazy的bean将不会在ApplicationContext启动时提前被实例化，而是在第一次向容器通过getBean索取bean时实例化的。

如果一个设置了立即加载的bean1,引用了一个延迟加载的bean2,那么bean1在容器启动时被实例化，而bean2由于被bean1引用，所以也被实例化，这种情况也符合延迟加载的bean在第一次调用时才被实例化的规则。 
在容器层次中通过在元素上使用’default-lazy-init’属性来控制延迟初始化也是可能的。如下面的配置：
```xml
<beans default-lazy-init="true"><!-- no beans will be eagerly pre-instantiated... --></beans>
```

一般beans 和 bean 层次配置的默认值都是false;并且bean的优先级>beans的优先级 
如果一个bean的scope属性为scope=“pototype“时，即使设置了lazy-init=”false”，容器启动时不实例化bean，而是调用getBean方法是实例化的； 
现在我们通过源码来分析一下;

## 二、lazy-init 属性被设置的地方，并且优先级 bean>beans;
如果想看所有属性被设置的地方请看博文 
[Spring是如何解析xml中的属性到BeanDefinition中的](springSourceCode_analysis_3_80223871.md)

```java
//解析bean的属性值
 public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName, BeanDefinition containingBean, AbstractBeanDefinition bd) {
 // 省略
 //如果当前元素没有设置 lazyInit 懒加载；则去 this.defaults.getLazyInit()；这个defaults是上一篇分析过的；整个xml文件全局的默认值；
        String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
        if (DEFAULT_VALUE.equals(lazyInit)) {
            lazyInit = this.defaults.getLazyInit();
        }
//省略....
}
```

## 三、lazy-init发挥作用的地方
```java
@Override
    public void refresh() throws BeansException, IllegalStateException {
    // 忽略..
    // 实例化所有剩余非 lazy-init 为true的单例对象
        finishBeanFactoryInitialization(beanFactory);
    // 忽略..     
    }
```

最终执行了 
beanFactory.preInstantiateSingletons();
```java
    @Override
    public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Pre-instantiating singletons in " + this);
        }

        // Iterate over a copy to allow for init methods which in turn register new bean definitions.
        // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
        List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

        // 遍历所有bean
        for (String beanName : beanNames) {
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            //1.bd的abstract属性是false；<bean abstract="false"> 不能被实例化，它主要作用是被用作被子bean继承属性用的；
            //2.单例对象并且 lazy-init为false
            //满足上面条件才行
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            /**
            *如果实现了FactoryBean接口
            *1.先将FactoryBean的实现类实例化；
            *2.判断是否将FactoryBean实现类的getObject方法返回的实例对象也实例化；判断依据
            *  2.1如果当前bean实现了SmartFactoryBean接口，并且isEagerInit()返回true；才会调用工厂类的方法
            */
                if (isFactoryBean(beanName)) {
                    final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                            @Override
                            public Boolean run() {
                                return ((SmartFactoryBean<?>) factory).isEagerInit();
                            }
                        }, getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
                else {//如果不是FactoryBean接口之间实例化
                    getBean(beanName);
                }
            }
        }

        // 调用所有SmartInitializingSingleton类型的实现类的afterSingletonsInstantiated方法；通过名字可以知道它表示 单例对象实例化后需要做的操作
        for (String beanName : beanNames) {
            Object singletonInstance = getSingleton(beanName);
            if (singletonInstance instanceof SmartInitializingSingleton) {
                final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged(new PrivilegedAction<Object>() {
                        @Override
                        public Object run() {
                            smartSingleton.afterSingletonsInstantiated();
                            return null;
                        }
                    }, getAccessControlContext());
                }
                else {
                    smartSingleton.afterSingletonsInstantiated();
                }
            }
        }
    }
```

## 四、问答
1. Ioc容器在实例化bean的时候，Ioc会主动调用FactoryBean类型的的getObject方法来为我们生成对象吗？ 
答: 一般情况下是不会的，一般情况碰到FactoryBean类型的是调用 getBean(&beanName),但是有一种情况例外，如果这个FactoryBean还实现了SmartInitializingSingleton接口的话，IOC就会帮我们主动调用getBean(beanName)来实例化；

# (转)Spring源码解析(十四)Spring调用初始化方法initializeBean

在执行完填充属性的方法populateBean(beanName, mbd, instanceWrapper)之后,就要执行初始化initializeBean方法了; 
show the code:
```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    invokeAwareMethods(beanName, bean);
                    return null;
                }
            }, getAccessControlContext());
        }
        else {
            //调用所有BeanNameAware、BeanClassLoaderAware、BeanFactoryAware接口方法
            invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //执行初始化之前的 前置操作  https://blog.csdn.net/u010634066/article/details/80291728
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }

        try {
            //调用初始化方法
            invokeInitMethods(beanName, wrappedBean, mbd);
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }

        if (mbd == null || !mbd.isSynthetic()) {
        //执行初始化之后的 后置操作  https://blog.csdn.net/u010634066/article/details/80291728
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
        return wrappedBean;
    }
```

## Aware接口
这只是一个标志接口,没有任何方法和属性；它表示bean对IOC容器的一个感知能力

## BeanNameAware
## BeanClassLoaderAware
## BeanFactoryAware
这三个都继承Aware接口，并且分别对应 
`void setBeanName(String name);`
`void setBeanClassLoader(ClassLoader classLoader);`
`void setBeanFactory(BeanFactory beanFactory) throws BeansException;`
都是spring将数据暴露出去的一种方式;我们在自己的bean中实现这个方法；就可以拿到对应的引用了，例如：
```java
public class TestBeanFactoryAware implements BeanFactoryAware {
    private BeanFactory beanFactory ;
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory ;
    }
    public BeanFactory getBeanFactory(){
        return beanFactory;
    }
}
```

主要分析一下invokeInitMethods方法

## InitializingBean接口
这个接口只有一个方法 
`void afterPropertiesSet() throws Exception;`
它的主要调用时机 
Aware感知接口 - > postProcessBeforeInitialization ->afterPropertiesSet
```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
            throws Throwable {

        boolean isInitializingBean = (bean instanceof InitializingBean);
            //如果当前bean是 InitializingBean类型的&&afterPropertiesSet这个方法没有注册为外部管理的初始化方法
            //就回调afterPropertiesSet方法
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            if (logger.isDebugEnabled()) {
                logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
            }
            if (System.getSecurityManager() != null) {
                try {
                    AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        @Override
                        public Object run() throws Exception {
                            ((InitializingBean) bean).afterPropertiesSet();
                            return null;
                        }
                    }, getAccessControlContext());
                }
                catch (PrivilegedActionException pae) {
                    throw pae.getException();
                }
            }
            else {
                ((InitializingBean) bean).afterPropertiesSet();
            }
        }

        if (mbd != null) {

            String initMethodName = mbd.getInitMethodName();
            //如果设置了initMethod方法的话也会执行用户配置的初始话方法
            //并且这个类不是 InitializingBean类型和不是afterPropertiesSet方法 ；
            //才能执行用户配置的方法
            if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {
                invokeCustomInitMethod(beanName, bean, mbd);
            }
        }
    }
```

执行用户配置的初始化方法
```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
            throws Throwable {
        //省略部分代码...
        String initMethodName = mbd.getInitMethodName();
        //获取初始化方法名的  Method对象，拿到这个对象就可以invoke调用了
        final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
                BeanUtils.findMethod(bean.getClass(), initMethodName) :
                ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
            try {
                //给权限
                ReflectionUtils.makeAccessible(initMethod);
                //执行
                initMethod.invoke(bean);
            }
            catch (InvocationTargetException ex) {
                throw ex.getTargetException();
            }

    }
```

## 总结
整个初始化流程如下 
1. 调用各类感知Aware接口 
2. 执行applyBeanPostProcessorsBeforeInitialization初始化前的 处置操作 
3. 调用InitializingBean接口初始化 
4. 如果配置了method-init，则调用其方法初始化 
5. 调用applyBeanPostProcessorsAfterInitialization 初始化之后的处置操作

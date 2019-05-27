# (转)Spring源码分析(十五)Spring中常用注解使用以及源码分析

从Java5.0开始，Java开始支持注解。Spring做为Java生态中的领军框架，从2.5版本后也开始支持注解。相比起之前使用xml来配置Spring框架，使用注解提供了更多的控制Spring框架的方式。

现在越来越多的项目也都在使用注解做相关的配置，但Spring的注解非常多，相信很多注解大家都没有使用过。本文就尽量全面地概括介绍一下Spring中常用的注解。 
[JAVA注解了解一下](https://blog.csdn.net/u010634066/article/details/80384125)

## 1.@Required
> 此注解用于bean的setter方法上。表示此属性是必须的，必须在配置阶段注入，否则会抛出BeanInitializationExcepion。

相关代码:

### RequiredAnnotationBeanPostProcessor
```java
public class RequiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {

@Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

        if (!this.validatedBeanNames.contains(beanName)) {
        //如果是是用的工厂方法 例如<bean id ="a" factory-bean="testAnnotFactory" factory-method="getAnnotInstance" />形式;会直接跳过不检查
        //如果配置了  <meta         key="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.skipRequiredCheck"
            value="true" />则也跳过不检查
            if (!shouldSkip(this.beanFactory, beanName)) {
                List<String> invalidProperties = new ArrayList<String>();
                for (PropertyDescriptor pd : pds) {
                //如果是set方法并且有Require注解，则抛出异常
                    if (isRequiredProperty(pd) && !pvs.contains(pd.getName())) {
                        invalidProperties.add(pd.getName());
                    }
                }
                if (!invalidProperties.isEmpty()) {
                    throw new BeanInitializationException(buildExceptionMessage(invalidProperties, beanName));
                }
            }
            this.validatedBeanNames.add(beanName);
        }
        return pvs;
    }
}
```

RequiredAnnotationBeanPostProcessor最终是实现了InstantiationAwareBeanPostProcessor接口的postProcessPropertyValues方法; 
就是在populateBean()方法里面被调用的
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {

        if (hasInstAwareBpps || needsDepCheck) {
            PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            if (hasInstAwareBpps) {
                for (BeanPostProcessor bp : getBeanPostProcessors()) {
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                        pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        if (pvs == null) {
                            return;
                        }
                    }
                }
            }
            if (needsDepCheck) {
                checkDependencies(beanName, mbd, filteredPds, pvs);
            }
        }
        //所以的属性准备就绪 进行填充
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
```

使用说明： 
1. 只能作用于setter方法上面 
2. 被@Require修饰了之后,该属性必须要被设置值 
3. 如果该类使用了工厂方式则跳过验证 
[关于factory-bean factory-method 了解一下](https://blog.csdn.net/nvd11/article/details/51542360)
```xml
 <bean id ="testAnnotFactory" class="src.testannot.TestAnnotFactory"/>
  <bean id ="a" factory-bean="testAnnotFactory" factory-method="getAnnotInstance" />
```

```java
public class TestAnnotFactory {
    public TestAnnot getAnnotInstance(){
        return new TestAnnot();
    }
}
public class TestAnnot {
    private String needRequire;
    public String getNeedRequire() {
        return needRequire;
    }
    @Required
    public void setNeedRequire(String needRequire) {
        this.needRequire = needRequire;
    }
}
```

上面的情况下 会跳过验证 
4. 如果配置了属性skipRequiredCheck也会跳过验证
```xml
  <bean class="src.testannot.TestAnnot" >
    <meta
            key="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.skipRequiredCheck"
            value="true" />
  </bean>
```

5 . 配置文件中要引入下面的后置处理器才会生效
```xml
<bean class="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor"/> 
```

## 2.@Autowired
> 在传统的spring注入方式中，我们对类变量都要求实现get与set的方法。在pring 2.5 引入了 @Autowired 注释，它可以对类成员变量、方法及构造函数进行标注，完成自动装配的工作。 通过 @Autowired的使用来消除 set ，get方法。不过在引及@Autowired注释后，要在spring的配置文件 applicationContext.xml中加入：如下代码

```xml
<!-- 该 BeanPostProcessor 将自动对标注 @Autowired 的 Bean 进行注入 -->     
  <bean class="org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor"/> 
```

看看注解的描述
```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {

    /**
     * Declares whether the annotated dependency is required.
     * <p>Defaults to {@code true}.
     */
    boolean required() default true;

}
```

[@Autowire使用了解一下](https://blog.csdn.net/u013257679/article/details/52295106)
怎么使用，点击上面的链接讲解的很清楚 ，我们来看下源码的实现
```java
public class AutowiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {


}
```

## 3.@Qualifier
> 此注解是和@Autowired一起使用的。使用此注解可以让你对注入的过程有更多的控制。@Qualifier可以被用在单个构造器或者方法的参数上。当上下文有几个相同类型的bean, 使用@Autowired则无法区分要绑定的bean，此时可以使用@Qualifier来指定名称。

他们的使用跟@Resource差不多
```java
   /*@Autowired
    @Qualifier(value = "t2")*/
    @Resource(name = "t2")
    public TestAnnot testAnnot;
```

都是根据名称来注入

## 4.@Configuration @Bean
> @Configuration标注在类上，相当于把该类作为spring的xml配置文件中的beans>，作用为：配置spring容器(应用上下文) 
> @Bean标注在方法上(返回某个实例的方法)，等价于spring的xml配置文件中的bean>，作用为：注册bean对象

[@Configuration注解、@Bean注解以及配置自动扫描、bean作用域](https://blog.csdn.net/javaloveiphone/article/details/52182899)

## 5.@ComponentScan @Lazy
> @ComponentScan 指定Spring扫描注解的package。如果没有指定包，那么默认会扫描此配置类所在的package。 
> @Lazy 此注解使用在Spring的组件类上。默认的，Spring中Bean的依赖一开始就被创建和配置。如果想要延迟初始化一个bean，那么可以在此类上使用Lazy注解，表示此bean只有在第一次被使用的时候才会被创建和初始化。此注解也可以使用在被@Configuration注解的类上，表示其中所有被@Bean注解的方法都会延迟初始化。

```java
@Component(value = "ttt")
@Lazy
public class TTT {
}
```

或者
```java
 @Bean(name = "ttt")
    @Lazy
    public TTT getTTT(){
        return new TTT();
    }
```

## 6.@Value
> 此注解使用在字段、构造器参数和方法参数上。@Value可以指定属性取值的表达式，支持通过#{}使用SpringEL来取值，也支持使用${}来将属性来源中(Properties文件、本地环境变量、系统属性等)的值注入到bean的属性中。此注解值的注入发生在AutowiredAnnotationBeanPostProcessor类中。

AutowiredAnnotationBeanPostProcessor可以解析 @Autowired和@Value
```java
public AutowiredAnnotationBeanPostProcessor() {
        this.autowiredAnnotationTypes.add(Autowired.class);
        this.autowiredAnnotationTypes.add(Value.class);
        try {
            this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                    ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
            logger.info("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
        }
        catch (ClassNotFoundException ex) {
            // JSR-330 API not available - simply skip.
        }
    }
```
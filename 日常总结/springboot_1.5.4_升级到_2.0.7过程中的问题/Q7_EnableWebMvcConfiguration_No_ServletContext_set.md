# Q7_EnableWebMvcConfiguration No ServletContext set

> 背景：项目将 `spring-boot-starter-parent` 的版本从 `1.5.4.RELEASE` 升级到了 `2.0.7.RELEASE`。项目中 `graphql starter` 版本从 `4.0.0` 升级到了 `5.0.2`。

## 问题发现

基于以上背景，本以为项目配置已经没有问题了。

但是在一次合并代码后，发现了如下错误提示。<br/>

```bash
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'resourceHandlerMapping' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.HandlerMapping]: Factory method 'resourceHandlerMapping' threw exception; nested exception is java.lang.IllegalStateException: No ServletContext set
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:591)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1246)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1096)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:535)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:495)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:317)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:315)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:759)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:867)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:548)
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:142)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:754)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:386)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:307)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1242)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1230)
	at com.ke.ehr.personnel.EhrKeApplication.main(EhrKeApplication.java:22)
```



提示关键字 `resourceHandlerMapping`、`WebMvcAutoConfiguration$EnableWebMvcConfiguration.class`、**No ServletContext set**。

**咋一看，提示有点吓人。**(事实上，确实是，在排查过程中，从日志提示来排查，总是抓不住关键点)



## 问题排查

从日志提示上，猜想可能是 **WebMvcConfigurerAdapter** 过时导致的，问题链接。[Q2_WebMvcConfigurerAdapter 过时](Q2_WebMvcConfigurerAdapter 过时.md)

但是不管怎么按标准修改，还是提示同样的错误提示。

从 **No ServletContext set** 关键字提示，能猜想到是 **Servlet** 初始化时，那出了问题。**Spring MVC** 处理请求的是 `DispatcherServlet`，而 `graphql starter` 中实现也是通过 **Servlet**，主要核心逻辑处理是 `graphql.servlet.AbstractGraphQLHttpServlet`。因此，两个 **Servlet** 能有交集的只有 **filter** 定义了。(从最后结果上看，这个也不是关键点)



## 问题关键

当将解决好版本升级的问题分支回退到合并其他分支的代码版本时，其运行时正常的。

当时虽然很快发现了这个问题。但是一直怀疑因为默写细节没有处理到位，**springboot** `2.0.*` 与 `1.5.*` 的版本有了较大改动，`graphql starter` 的 `4.0.0` 与 `5.0.2` 也有了较大调整。所以一直怀疑有默写潜在的 **bug**。

在不断的分支代码一点一点的回退与对比的过程中，最后发现了在某个 **Service** 中依赖了某个 **Service** 后，就会提示上面的错误。但是 **spring IOC** 已经将循环依赖的问题解决了(只要不是通过构造器注入)，而且项目中都是通过 `@Autowired` 注解来注入的。应该不会出现循环依赖的问题，而且提示也不对。

通过开发过程中，对最有可能的问题猜测，就是**两个Service** 都通过 `@Autowired` 注解注入了 [`IGraphQL`](Q5_Graphql_starter-4.0.0升级到5.0.2中的问题.md)  对象。于是通过 **ApplicationContext** 获取对应的 **Bean**，避免 `GraphQL` 内部加载到导致的依赖注入问题。


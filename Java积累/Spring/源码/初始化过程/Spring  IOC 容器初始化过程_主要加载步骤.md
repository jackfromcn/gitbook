# Spring  IOC 容器初始化过程_主要加载步骤



## 方式1



```java
XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
Person p = (Person) factroy.getBean("person");
```



## 方式二



```java
ClassPathResource resource = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factroy);
reader.loadBeanDefinitions(resource);
```



## Spring 加载资源并装配对象的过程

1. 定义好 `Spring` 的配置文件。
2. 通过 `Resource` 对象将 `Spring` 配置文件进行抽象。抽象成一个 `Resource` 对象。
3. 定义好 `Bean` 工厂(各种 `BeanFactory`)。
4. 定义好 `XmlBeanDefinitionReader` 对象，并将工厂作为参数传递进入供后续回调使用。
5. 通过 `XmlBeanDefinitionReader` 对象读取之前抽象出的 `Resource` 对象(包含 **XML** 文件的解析过程)。
6. 本质上，**XML**文件的解析是有 `XmlBeanDefinitionReader` 对象交由 `BeanDefinitionParserDelegate` 委托来完成的，实质上这里面使用到了委托模式。
7. `IOC` 容器创建完毕，用户可以通过容器获取到所需要的对象信息。

**重要：**在 `DefaultBeanDefinitionDocumentReader` 类中的 `doRegisterBeanDefinitions` 方法使用了经典的模板方法设置模式，子类可以重写 `preProcessXml` 与 `postProcessXml` 方法，实现对**XML**配置文件的自定义扩展，类似于 `Junit` 的 `setUp`，`testXXX` 与 `tearDown` 方法。



1. `Spring` 的 `Bean` 实际上是缓存在 `CurrentHashMap` 对象中。
2. 在创建 `Bean` 之前，首先需要将该 `Bean` 的创建标识设定好，标识该 `Bean` 已经或即将被创建，为的是增强缓存的效率。
3. 根据 `Bean` 的 `scope` 属性来确定是 `singleton` 还是 `prototype` 等返回，然后创建相应的 `Bean` 对象。
4. 通过**Java**反射来创建 `Bean` 的实例，在创建之前首先检查访问修饰符，如果不是**public**的，则调用 `setAccessible(true)` 来突破**Java**的语法限制，使得可以通过如私有的构造器方法来创建对象实例。
5. 接下来寻找 `Bean` 的属性值，来完成属性的注入。
6. 将所创建出的 `singleton` 对象添加到缓存中，供


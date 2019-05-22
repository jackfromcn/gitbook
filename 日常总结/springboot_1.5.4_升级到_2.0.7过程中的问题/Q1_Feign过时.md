# Q1_Feign 过时

> 项目背景：项目基础架构就是个普通的 **springboot** 项目，没有整合到 **spring cloud** 中，但是为了对外 **http** 接口对接方便，于是单独引入了 `feign` 组件作为 **http** 请求的包装。



## [项目初期引入 `fegin` 过程](/feign不使用eureka调用外部系统.md)



## 项目初期定义

1、`spring-boot-starter-parent` 版本定义

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.14.RELEASE</version>
</parent>
```

2、`fegin` 版本定义

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
    <version>1.4.4.RELEASE</version>
</dependency>
```

`@EnableFeignClients` 注解应用路径：`org.springframework.cloud.netflix.feign.EnableFeignClients`

`@FeignClient` 注解应用路径：`org.springframework.cloud.netflix.feign.FeignClient`



## 项目改造定义

1、`spring-boot-starter-parent` 版本定义

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.7.RELEASE</version>
</parent>
```

2、`fegin` 版本定义

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
```



## 注解应用路径修改

| 注解                | old path                                                   | new path                                               |
| ------------------- | ---------------------------------------------------------- | ------------------------------------------------------ |
| @EnableFeignClients | org.springframework.cloud.netflix.feign.EnableFeignClients | org.springframework.cloud.openfeign.EnableFeignClients |
| @FeignClient        | org.springframework.cloud.netflix.feign.FeignClient        | org.springframework.cloud.openfeign.FeignClient        |



这里只需要将包依赖修改一下，并将引用类的路径修改一下即可。并不会有大的问题。
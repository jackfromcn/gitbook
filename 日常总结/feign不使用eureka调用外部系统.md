# feign不使用eureka 调用外部系统

> 背景：项目中各服务的 *http* 接口，都是自己内部系统维护一套加密授权列表，规则类似，但是授权的 `appId`、`appKey` 是不同的，系统之间使用的不是 `spring cloud` 全家桶，也没有统一的授权校验的 `oAuth` 服务。对接的系统服务、接口增多后不好维护。

考虑使用 `@FeignClient` 来进行接口维护，其内部封装了一套调用前后处理的规则，规则定义类似组件一样，可插拔，根据各个系统服务，定制不同的加密、解析规则。



## 尝试前的问题

1. 本系统是 `spring boot` 项目，`spring-boot-starter-parent` 版本是 `1.5.14.RELEASE`，系统中维护了内部服务需要的各组件，`jar` 极其容易冲突。
2. `spring cloud` 服务通过 `eureka` 来获取服务，配置比较简单，之前也没有调用外部服务的相关经验。但是 `feign` 底层最后也是通过 `http` 请求实现的。需要对 `fegin` 有一定了解，或者能从网上找到相关 *demo*。



## 尝试中的问题

参考文章：

- **Github ** 代码 `demo`（**spring cloud** 版本依赖有些问题，*pull* 下来后，无法正常启动，可以参考）： <https://github.com/dangnianchuntian/springcloud>
- **FeignClient外部http请求**：https://blog.csdn.net/sincy09/article/details/82979093
- **FeignClient 调用**：https://blog.csdn.net/qq_36763236/article/details/82024039
- **springboot调用外部接口FeignClient**：https://blog.csdn.net/gisam/article/details/72757620



问题：

- `spring cloud` 依赖的版本，有的 `demo` 中不是稳定版本，无法拉取下来。
- 最开始跟着网上 *demo* 尝试，项目启动很快大概在 *3 ~ 4 s* 内就会启动，失败，说明项目启动类哪里配置错误了，所以将 `@EnableFeignClients` 注解给去掉了，后面虽然理解 `demo` 中实践的主要步骤，但是一直没有将 `@EnableFeignClients` 注释给加回来，也不确定问题主要是在哪。最初失败的 `demo` 依赖的 `fegin` ***jar*** 包应该是 `netflix` 原生的实现。



## 问题解决的 `demo`

最终问题解决所需要的几个姿势很简单。

1. 添加 `feginclient` 所需要的依赖

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-feign</artifactId>
       <version>1.4.4.RELEASE</version>
   </dependency>
   ```

2. 在启动类上注释 `@org.springframework.cloud.netflix.feign.EnableFeignClients`

   ```java
   package com;
   
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.netflix.feign.EnableFeignClients;
   
   /**
    * @author wencheng
    * @date 2018/11/30
    */
   @Slf4j
   @EnableFeignClients
   @SpringBootApplication
   public class IApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(IApplication.class, args);
       }
   
   }
   ```

3. 在 `@FeignClient` 注释中 `url` 指定直连服务的 **http** 地址

   ```java
   import org.springframework.cloud.netflix.feign.FeignClient;
   import org.springframework.http.MediaType;
   import org.springframework.web.bind.annotation.GetMapping;
   
   /**
    *
    * @author wencheng
    * @date 2019/4/8
    */
   @FeignClient(name = "provider",
           url = "http://localhost:8090",
           configuration = FeignClientConfiguration.class)
   public interface Ewaytec2001API {
   
       @GetMapping(value = "/actuator/metrics.json", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
       String metrics();
   
   }
   ```

   ```java
   import com.exception.ServiceException;
   import feign.Feign;
   import feign.Request;
   import feign.RequestInterceptor;
   import feign.Retryer;
   import feign.auth.BasicAuthRequestInterceptor;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Scope;
   
   import java.util.List;
   import java.util.concurrent.TimeUnit;
   
   @Configuration
   public class FeignClientConfiguration {
       @Value("${key:username}")
       private String consumerKey;
   
       @Value("${password:password}")
       private String consumerPassword;
       @Bean
       public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
           return new BasicAuthRequestInterceptor(consumerKey, consumerPassword);
       }
   
       public static int connectTimeOutMillis = 150000;//超时时间
       public static int readTimeOutMillis = 150000;
       @Bean
       public Request.Options options() {
           return new Request.Options(connectTimeOutMillis, readTimeOutMillis);
       }
   
       @Bean("FeignRetryer")
       @ConditionalOnMissingBean
       public Retryer feignRetryer() {
           return new Retryer.Default(TimeUnit.MINUTES.toMillis(5), TimeUnit.MINUTES.toMillis(10), 2);
       }
   
       @Bean
       @Scope("prototype")
       public Feign.Builder feignBuilder(List<RequestInterceptor> requestInterceptors) {
           Feign.Builder builder = Feign.builder().decode404()
                   .requestInterceptors(requestInterceptors).requestInterceptor(requestTemplate -> {
   
                   }).errorDecoder((s, response) -> {
   //                        if (response.status() >= 400 && response.status() <= 499) {
   ////                            if (response.body() != null) {
   ////                                val error = utils.mapper.readValue(response.body().asInputStream(), ErrorEntity::class.java)
   ////                                throw AppException(error.message, HttpStatus.valueOf(error.status))
   ////                            }
   ////                            val status = HttpStatus.valueOf(response.status())
   ////                            throw AppException(status.reasonPhrase, status)
   //                        } else {
   //                            throw Exception("$s 出现异常：" + response.body().asReader().read()))
   //                        }
                       throw new ServiceException(response.toString());
                   });
           return builder;
       }
   }
   ```

   


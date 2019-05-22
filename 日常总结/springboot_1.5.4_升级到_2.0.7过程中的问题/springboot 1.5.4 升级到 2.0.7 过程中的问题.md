# springboot 1.5.4 升级到 2.0.7 过程中的问题

> 项目背景：项目中使用的 `spring-boot-starter-parent` 版本是 `1.5.14.RELEASE`，系统中也整合了 `spring-boot-starter-parent`，其版本是 `4.0.0`，依赖的 `graphql-java-tools` 的版本是 `4.3.0`。
>
> 
>
> 需求背景：基本功能都提供了。其实不升级也没多大问题。但是 `graphql tools` 的版本比较低，支持解析的 `schema` 格式少了些，尤其是定义的 `Query` 和 `Mutation` 不能拆分在多个文件中，接口定义都必须在一个 `schema` 文件中。

为了解决将 ***graphql*** 的 **schema** 文件拆分到多个文件中，在按模块开发过程中，有效降低接口定义的冲突，于是将 **springboot** 升级到 `2.0.7.RELEASE`。

为了解决上面的问题，于是就有了这次的总结。

**springboot 2.0** 本身就有了比较多的调整和优化，所以将升级过程中发现的问题和其调整的部分进行记录。



## **SpringBoot** `2.0` 对比 `1.5` 的调整(发现的)



- Spring Boot Maven plugin

- 配置文件的定义的改动
- 数据库相关的改动

- Actuator的相关变动
- spring security的相关变动
- DataSource的相关变动
- redis的相关改动
- Elasticsearch的的相关变动
- 视图模板引擎



## 升级过程中遇到的问题列表



- `Fegin` 过时
- `WebMvcConfigurerAdapter` 过时
- `Actuator` 和 `Prometheus` 的调整
- `Spring security` 的调整，以及与 `Actuator` 安全校验的整合
- `Graphql starter` **4.0.0 ** 升级到 **5.0.2** 的问题
- `graphiql` 和 `voyager` 的整合
- 运行提示 **EnableWebMvcConfiguration No ServletContext set** ，问题原因



## 参考

- springboot的官方wiki的文档[Springboot Migration文档地址](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Fspring-projects%2Fspring-boot%2Fwiki%2FSpring-Boot-2.0-Migration-Guide)
- [记录springboot从1.5.10升级到2.0.0过程中遇到的问题（上）](https://www.jianshu.com/p/81e2798d6dd1)
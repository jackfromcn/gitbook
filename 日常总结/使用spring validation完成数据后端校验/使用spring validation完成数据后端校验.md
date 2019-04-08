# 使用spring validation完成数据后端校验

参考链接([ 徐靖峰|个人博客](https://www.cnkirito.moe/))：<https://www.cnkirito.moe/spring-validation/>

>背景：项目准备将 `restful` 请求方式改造成 `graphql` 请求方式，项目是面向 B 端的，PC 端提交表单参数内容非常多，参数校验也比较多。

## `restful` 与 `graphql` 请求参数校验对比

使用 `spring mvc` 的 `restful` 请求，`spring mvc`  框架 中 `WebDataBinder`  可以将 `request` 中的参数自动绑定到 `POJO` 对象中，可以使用 `spring validation` 完成后端校验。

但是 `graphql` 请求方式，使用的是 `graphql` 官方语法定义的 `graphql-tools` 来进行参数解析和绑定。示例如下：<br/>![graphql请求参数](./graphql请求参数.png)

![graphql解析入参](./graphql解析入参.png)
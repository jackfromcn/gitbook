# 使用spring validation完成数据后端校验

参考链接([ 徐靖峰|个人博客](https://www.cnkirito.moe/))：<https://www.cnkirito.moe/spring-validation/>

>背景：项目准备将 `restful` 请求方式改造成 `graphql` 请求方式，项目是面向 B 端的，PC 端提交表单参数内容非常多。参数校验也比较多。使用 `spring mvc` 的 `restful` 请求，可以使用 `spring validation` 完成后端校验，但是 `graphql` 请求
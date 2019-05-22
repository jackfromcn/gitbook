# Spring Boot Actuator监控-案例1

> 背景：

## Spring Boot 项目配置

## pom.xml

### spring boot 版本

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.14.RELEASE</version>
</parent>
```



### spring 监控所需依赖

```xml
<!-- spring boot actuator-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>

<!-- 监控 start -->
<!-- 监控 规范 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-spring-legacy</artifactId>
    <version>1.0.6</version>
</dependency>

<!-- prometheus 输入依赖 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.0.6</version>
</dependency>

<!-- influxdb 输入依赖 -->
<!--<dependency>-->
    <!--<groupId>io.micrometer</groupId>-->
    <!--<artifactId>micrometer-registry-influx</artifactId>-->
    <!--<version>1.0.6</version>-->
<!--</dependency>-->

<!-- jvm 输出规范 -->
<dependency>
    <groupId>io.github.mweirauch</groupId>
    <artifactId>micrometer-jvm-extras</artifactId>
    <version>0.1.2</version>
</dependency>
<!-- 监控 end -->
```



### application.yml

```yaml
management:
  context-path: /actuator
  security:
    enabled: true
  info:
    git:
      mode: full
  metrics:
    export:
      influx:
        enabled: true
        db: springboot
        uri: http://tx-node1:8086
#        user-name:
#        password:
        connect-timeout: 1s
        read-timeout: 10s
        auto-create-db: true
        step: 1m
        num-threads: 2
        consistency: one
        compressed: true
        batch-size: 10000
security:
  user:
    name: username
    password: user_password
  basic:
    enabled: false
endpoints:
  enabled: true
  sensitive: true
  health:
    sensitive: false
  info:
    sensitive: false
```



### 代码配置

`prometheus` 抓取对应信息需要定义其 **application** 的标签。一般对应为 `${spring.application.name}`。

```java
@Bean
MeterRegistryCustomizer<MeterRegistry> configurer(
        @Value("${spring.application.name}") String applicationName) {
    return registry -> registry.config().commonTags("application", applicationName);
}
```



### [prometheus、altermanager、grafana 相关配置](../../docker-compose/prometheus、alertmanager、grafana/prometheus、alertmanager、grafana.md)

#### Prometheus 配置

添加 `Prometheus` 的监控 ***Job***。

```yaml
  - job_name: 'actuator-demo'
    metrics_path: '/actuator/prometheus'
    basic_auth:
      username: kehr
      password: kehr123456
    static_configs:
      - targets: ['localhost:8080']
```



#### Grafana 配置

登录 `Grafana UI`(admin/admin)

```http
http://localhost:3000
```

通过Grafana的**+**图标导入(**Import**) `JVM (Micrometer)` dashboard：

- grafana id = **4701**
- 注意选中`prometheus`数据源

查看`JVM (Micormeter)` dashboard。



#### 参考

Grafana Dashboard仓库

```http
https://grafana.com/dashboards
```

Micrometer Prometheus支持

```http
https://micrometer.io/docs/registry/prometheus
```

Micrometer Springboot 1.5支持

```http
https://micrometer.io/docs/ref/spring/1.5
```


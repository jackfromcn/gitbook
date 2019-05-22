# Q3_监控 Actuator&Prometheus 调整

> 背景：



## 1.5.4.RELEASE 的配置

### **`pom.xml`** 依赖配置

```xml
<!-- 监控 start -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-spring-legacy</artifactId>
    <version>1.0.6</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.0.6</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-influx</artifactId>
    <version>1.0.6</version>
</dependency>

<dependency>
    <groupId>io.github.mweirauch</groupId>
    <artifactId>micrometer-jvm-extras</artifactId>
    <version>0.1.2</version>
</dependency>
<!-- 监控 end -->
```



### **`application.yml`** 配置

```yaml
#安全认证 https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-monitoring.html
management:
  context-path: /actuator
  security:
    enabled: true
  info:
    git:
      mode: full
#  metrics:
#    export:
#      influx:
#        enabled: true
#        db: ${spring.application.name}
#        uri: http://tx-node1:8086
##        user-name:
##        password:
#        connect-timeout: 1s
#        read-timeout: 10s
#        auto-create-db: true
#        step: 1m
#        num-threads: 2
#        consistency: one
#        compressed: true
#        batch-size: 10000
security:
  user:
    name: kehr
    password: kehr123456
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



## 2.0.7.RELEASE 的配置

### **`pom.xml`** 依赖配置

```xml
<!-- 监控 start -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<!-- 监控 end -->
```



### **`application.yml`** 配置

```yaml
#安全认证 https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-monitoring.html
management:
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: WHEN_AUTHORIZED
      roles: "ADMIN"
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m
        descriptions: true
```


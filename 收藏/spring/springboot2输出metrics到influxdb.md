# springboot2输出metrics到influxdb

## 序

本文主要研究一下如何将springboot2的metrics输出到influxdb

## maven

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-influx</artifactId>
        </dependency>
```

## 配置

```yaml
management:
  metrics:
    export:
      influx:
        enabled: true
        db: springboot
        uri: http://192.168.99.100:8086
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
```

## influx

```bash
docker run -d --name influx -p 8086:8086 influxdb
```

> 启动之后创建数据库

- 命令行创建

```bash
docker exec -it influx influx
create database springboot
```

- rest接口创建

```bash
curl -i -X POST http://192.168.99.100:8086/query --data-urlencode "q=CREATE DATABASE springboot"
```

返回

```bash
HTTP/1.1 200 OK
Content-Type: application/json
Request-Id: f3ce7449-7227-11e8-8002-000000000000
X-Influxdb-Build: OSS
X-Influxdb-Version: 1.5.3
X-Request-Id: f3ce7449-7227-11e8-8002-000000000000
Date: Sun, 17 Jun 2018 12:14:15 GMT
Transfer-Encoding: chunked

{"results":[{"statement_id":0}]}
```

> 或者直接配置文件指定auto-create-db=true，就无需额外创建

## 查看

- 命令行查看

```sql
docker exec -it influx influx
> use springboot
> show MEASUREMENTS
name: measurements
name
----
jvm.buffer.count
jvm.buffer.memory.used
jvm.buffer.total.capacity
jvm.classes.loaded
jvm.classes.unloaded
jvm.gc.live.data.size
jvm.gc.max.data.size
jvm.gc.memory.allocated
jvm.gc.memory.promoted
jvm.gc.pause
jvm.memory.committed
jvm.memory.max
jvm.memory.used
jvm.threads.daemon
jvm.threads.live
jvm.threads.peak
logback.events
process.cpu.usage
process.files.max
process.files.open
process.start.time
process.uptime
system.cpu.count
system.cpu.usage
system.load.average.1m
```

> 查看具体指标

```sql
> show series from "http.server.requests"
key
---
http.server.requests,exception=None,method=GET,metric_type=histogram,status=200,uri=/actuator/health
> select * from "http.server.requests"
name: http.server.requests
time                count exception mean      method metric_type status sum       upper     uri
----                ----- --------- ----      ------ ----------- ------ ---       -----     ---
1529238292912000000 0     None      0         GET    histogram   200    0         72.601487 /actuator/health
1529238352888000000 2     None      39.154634 GET    histogram   200    78.309267 72.601487 /actuator/health
1529238412886000000 0     None      0         GET    histogram   200    0         72.601487 /actuator/health
1529238472885000000 0     None      0         GET    histogram   200    0         0         /actuator/health
1529238532882000000 0     None      0         GET    histogram   200    0         0         /actuator/health
1529238592879000000 0     None      0         GET    histogram   200    0         0         /actuator/health
```

> 注意这里表名要加引号

- rest接口查看

```bash
curl -G 'http://192.168.99.100:8086/query?pretty=true' --data-urlencode "db=springboot" --data-urlencode "q=SELECT \"*\" FROM \"http.server.requests\""
{
    "results": [
        {
            "statement_id": 0
        }
    ]
}
```

## 小结

springboot2使用micrometer作为metrics组件，其提供了对influxdb的支持，只需要引入micrometer-registry-influx，然后进行配置即可。

## doc

- [Exporting metrics to InfluxDB and Prometheus using Spring Boot Actuator](https://hk.saowen.com/a/ba843d5d9fc69db72381fc259ac4a6bc7f642d80097459433ad9dc1650eb3649)
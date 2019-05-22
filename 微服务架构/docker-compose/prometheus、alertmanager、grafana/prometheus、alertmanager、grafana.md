# prometheus、alertmanager、grafana



### docker-compose.yaml

```yaml
version: '2'

networks:
  monitor:
    driver: bridge

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    hostname: prometheus
    restart: always
    volumes:
      - prometheus.yml:/etc/prometheus/prometheus.yml
      - node_down.yml:/etc/prometheus/node_down.yml
      - memory_over.yml:/etc/prometheus/memory_over.yml
    ports:
      - "9090:9090"
    networks:
      - monitor

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    hostname: alertmanager
    restart: always
    volumes:
      - alertmanager.yml:/etc/alertmanager/config.yml
    ports:
      - "9093:9093"
    networks:
      - monitor

  grafana:
    image: grafana/grafana
    container_name: grafana
    hostname: grafana
    restart: always
    ports:
      - "3000:3000"
    networks:
      - monitor
```



###  prometheus.yml

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["localhost:9093"]
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "node_down.yml"
  - "memory_over.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'actuator-demo'
    metrics_path: '/actuator/prometheus'
    basic_auth:
      username: kehr
      password: kehr123456
    static_configs:
      - targets: ['10.33.138.195:8090']
```



### 添加报警规则

#### prometheus targets 监控报警参考配置(node_down.yml)：

```yaml
groups:
- name: example
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      user: caizh
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
```



#### 节点内存使用率监控报警参考配置（memory_over.yml）

```yaml
groups:
- name: example
  rules:
  - alert: NodeMemoryUsage
    expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 100 > 80
    for: 1m
    labels:
      user: caizh
    annotations:
      summary: "{{$labels.instance}}: High Memory usage detected"
      description: "{{$labels.instance}}: Memory usage is above 80% (current value is:{{ $value }})"
```



#### 修改prometheus配置文件prometheus.yml，开启报警功能，添加报警规则配置文件

```yaml
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["localhost:9093"]
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "node_down.yml"
  - "memory_over.yml"
```



#### alertmanager.yml

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:587'
  smtp_from: '邮箱账号'
  smtp_auth_username: '邮箱账号'
  smtp_auth_password: '邮箱密码'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10s
  receiver: 'team-X-mails'
receivers:
- name: 'team-X-mails'
  email_configs:
  - to: '邮箱账号'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```


# K8s 集群报警规则

## 背景说明

目前 Kubernetes 集群采用的监控工具为 Prometheus Operator(包含 Prometheus，ServiceMonitor，Alertmanager，Grafana 等组件) 平台进行数据收集及监控报警。

## 版本信息

文档 v1-20190326 Operator 0.25 Prometheus v2.5.0 Grafana 5.2.4 Alertmanager v0.15.3 Kubernetes 1.8.9

## 配置详情

当前警报规则配置如下(持续更新)：

prometheus-rules.yaml

### Kubernetes集群监控数据二次合成

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: prometheus-k8s-rules
  namespace: monitoring
spec:
  groups:
  - name: k8s.rules
    rules:
    - expr: |
        sum(rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=""}[5m])) by (namespace)
      record: namespace:container_cpu_usage_seconds_total:sum_rate
    - expr: |
        sum by (namespace, pod_name, container_name) (
          rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=""}[5m])
        )
      record: namespace_pod_name_container_name:container_cpu_usage_seconds_total:sum_rate
    - expr: |
        sum(container_memory_usage_bytes{job="kubelet", image!="", container_name!=""}) by (namespace)
      record: namespace:container_memory_usage_bytes:sum
    - expr: |
        sum by (namespace, label_name) (
           sum(rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=""}[5m])) by (namespace, pod_name)
         * on (namespace, pod_name) group_left(label_name)
           label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod_name", "$1", "pod", "(.*)")
        )
      record: namespace_name:container_cpu_usage_seconds_total:sum_rate
    - expr: |
        sum by (namespace, label_name) (
          sum(container_memory_usage_bytes{job="kubelet",image!="", container_name!=""}) by (pod_name, namespace)
        * on (namespace, pod_name) group_left(label_name)
          label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod_name", "$1", "pod", "(.*)")
        )
      record: namespace_name:container_memory_usage_bytes:sum
    - expr: |
        sum by (namespace, label_name) (
          sum(kube_pod_container_resource_requests_memory_bytes{job="kube-state-metrics"}) by (namespace, pod)
        * on (namespace, pod) group_left(label_name)
          label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod_name", "$1", "pod", "(.*)")
        )
      record: namespace_name:kube_pod_container_resource_requests_memory_bytes:sum
    - expr: |
        sum by (namespace, label_name) (
          sum(kube_pod_container_resource_requests_cpu_cores{job="kube-state-metrics"} and on(pod) kube_pod_status_scheduled{condition="true"}) by (namespace, pod)
        * on (namespace, pod) group_left(label_name)
          label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod_name", "$1", "pod", "(.*)")
        )
      record: namespace_name:kube_pod_container_resource_requests_cpu_cores:sum
    - expr: |
        histogram_quantile(0.99, sum(rate(scheduler_e2e_scheduling_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.99"
      record: cluster_quantile:scheduler_e2e_scheduling_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.99, sum(rate(scheduler_scheduling_algorithm_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.99"
      record: cluster_quantile:scheduler_scheduling_algorithm_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.99, sum(rate(scheduler_binding_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.99"
      record: cluster_quantile:scheduler_binding_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.9, sum(rate(scheduler_e2e_scheduling_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.9"
      record: cluster_quantile:scheduler_e2e_scheduling_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.9, sum(rate(scheduler_scheduling_algorithm_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.9"
      record: cluster_quantile:scheduler_scheduling_algorithm_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.9, sum(rate(scheduler_binding_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.9"
      record: cluster_quantile:scheduler_binding_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.5, sum(rate(scheduler_e2e_scheduling_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.5"
      record: cluster_quantile:scheduler_e2e_scheduling_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.5, sum(rate(scheduler_scheduling_algorithm_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.5"
      record: cluster_quantile:scheduler_scheduling_algorithm_latency:histogram_quantile
    - expr: |
        histogram_quantile(0.5, sum(rate(scheduler_binding_latency_microseconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.5"
      record: cluster_quantile:scheduler_binding_latency:histogram_quantile
  - name: kube-apiserver.rules
    rules:
    - expr: |
        histogram_quantile(0.99, sum(rate(apiserver_request_latencies_bucket{job="apiserver"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.99"
      record: cluster_quantile:apiserver_request_latencies:histogram_quantile
    - expr: |
        histogram_quantile(0.9, sum(rate(apiserver_request_latencies_bucket{job="apiserver"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.9"
      record: cluster_quantile:apiserver_request_latencies:histogram_quantile
    - expr: |
        histogram_quantile(0.5, sum(rate(apiserver_request_latencies_bucket{job="apiserver"}[5m])) without(instance, pod)) / 1e+06
      labels:
        quantile: "0.5"
      record: cluster_quantile:apiserver_request_latencies:histogram_quantile
    - alert: CoreDNSDown
      annotations:
        message: CoreDNS has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-corednsdown
      expr: |
        absent(up{job="kube-dns"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: KubeAPIDown
      annotations:
        message: KubeAPI has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapidown
      expr: |
        absent(up{job="apiserver"} == 1)
      for: 15m
      labels:
        severity: critical
#    - alert: KubeControllerManagerDown

<!-- toc -->

- [annotations:](#annotations)
- [message: KubeControllerManager has disappeared from Prometheus target discovery.](#message-kubecontrollermanager-has-disappeared-from-prometheus-target-discovery)
- [expr: |](#expr-)
- [absent(up{job="kube-controller-manager"} == 1)](#absentupjobkube-controller-manager--1)
- [for: 15m](#for-15m)
- [labels:](#labels)
- [severity: critical](#severity-critical)
- [- alert: KubeSchedulerDown](#-alert-kubeschedulerdown)
- [annotations:](#annotations-1)
- [message: KubeScheduler has disappeared from Prometheus target discovery.](#message-kubescheduler-has-disappeared-from-prometheus-target-discovery)
- [expr: |](#expr--1)
- [absent(up{job="kube-scheduler"} == 1)](#absentupjobkube-scheduler--1)
- [for: 15m](#for-15m-1)
- [labels:](#labels-1)
- [severity: critical](#severity-critical-1)
- [- alert: KubeletTooManyPods](#-alert-kubelettoomanypods)
- [annotations:](#annotations-2)
- [message: Kubelet {{ $labels.instance }} is running {{ $value }} Pods, close](#message-kubelet--labelsinstance--is-running--value--pods-close)
- [to the limit of 110.](#to-the-limit-of-110)
- [runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubelettoomanypods](#runbookurl-httpsgithubcomkubernetes-monitoringkubernetes-mixintreemasterrunbookmdalert-name-kubelettoomanypods)
- [expr: |](#expr--2)
- [kubelet_running_pod_count{job="kubelet"} > 110 * 0.9](#kubeletrunningpodcountjobkubelet--110--09)
- [for: 15m](#for-15m-2)
- [labels:](#labels-2)
- [severity: warning](#severity-warning)
    + [Prometheus 集群自身警报及数据二次合成](#prometheus-集群自身警报及数据二次合成)
    + [Kubernetes 集群内部运行应用监控警报](#kubernetes-集群内部运行应用监控警报)
    + [Kubernetes集群计算资源警报](#kubernetes集群计算资源警报)
    + [Kubernetes集群存储资源警报](#kubernetes集群存储资源警报)
    + [Java 应用警报规则](#java-应用警报规则)

<!-- tocstop -->

#      annotations:
#        message: KubeControllerManager has disappeared from Prometheus target discovery.
#      expr: |
#        absent(up{job="kube-controller-manager"} == 1)
#      for: 15m
#      labels:
#        severity: critical
#    - alert: KubeSchedulerDown
#      annotations:
#        message: KubeScheduler has disappeared from Prometheus target discovery.
#      expr: |
#        absent(up{job="kube-scheduler"} == 1)
#      for: 15m
#      labels:
#        severity: critical
    - alert: KubeNodeNotReady
      annotations:
        message: '{{ $labels.node }} has been unready for more than an hour.'
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubenodenotready
      expr: |
        kube_node_status_condition{job="kube-state-metrics",condition="Ready",status="true"} == 0
      for: 1h
      labels:
        severity: warning
    - alert: KubeVersionMismatch
      annotations:
        message: There are {{ $value }} different versions of Kubernetes components
          running.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeversionmismatch
      expr: |
        count(count(kubernetes_build_info{job!="kube-dns"}) by (gitVersion)) > 1
      for: 1h
      labels:
        severity: warning
    - alert: KubeClientErrors
      annotations:
        message: Kubernetes API server client '{{ $labels.job }}/{{ $labels.instance
          }}' is experiencing {{ printf "%0.0f" $value }}% errors.'
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclienterrors
      expr: |
        (sum(rate(rest_client_requests_total{code!~"2..|404"}[5m])) by (instance, job)
          /
        sum(rate(rest_client_requests_total[5m])) by (instance, job))
        * 100 > 1
      for: 15m
      labels:
        severity: warning
    - alert: KubeClientErrors
      annotations:
        message: Kubernetes API server client '{{ $labels.job }}/{{ $labels.instance
          }}' is experiencing {{ printf "%0.0f" $value }} errors / second.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclienterrors
      expr: |
        sum(rate(ksm_scrape_error_total{job="kube-state-metrics"}[5m])) by (instance, job) > 0.1
      for: 15m
      labels:
        severity: warning
#    - alert: KubeletTooManyPods
#      annotations:
#        message: Kubelet {{ $labels.instance }} is running {{ $value }} Pods, close
#          to the limit of 110.
#        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubelettoomanypods
#      expr: |
#        kubelet_running_pod_count{job="kubelet"} > 110 * 0.9
#      for: 15m
#      labels:
#        severity: warning
    - alert: KubeAPILatencyHigh
      annotations:
        message: The API server has a 99th percentile latency of {{ $value }} seconds
          for {{ $labels.verb }} {{ $labels.resource }}.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapilatencyhigh
      expr: |
        cluster_quantile:apiserver_request_latencies:histogram_quantile{job="apiserver",quantile="0.99",subresource!="log",verb!~"^(?:LIST|WATCH|WATCHLIST|PROXY|CONNECT)$"} > 1
      for: 10m
      labels:
        severity: warning
    - alert: KubeAPILatencyHigh
      annotations:
        message: The API server has a 99th percentile latency of {{ $value }} seconds
          for {{ $labels.verb }} {{ $labels.resource }}.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapilatencyhigh
      expr: |
        cluster_quantile:apiserver_request_latencies:histogram_quantile{job="apiserver",quantile="0.99",subresource!="log",verb!~"^(?:LIST|WATCH|WATCHLIST|PROXY|CONNECT)$"} > 4
      for: 10m
      labels:
        severity: critical
    - alert: KubeAPIErrorsHigh
      annotations:
        message: API server is returning errors for {{ $value }}% of requests.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
      expr: |
        sum(rate(apiserver_request_count{job="apiserver",code=~"^(?:5..)$"}[5m])) without(instance, pod)
          /
        sum(rate(apiserver_request_count{job="apiserver"}[5m])) without(instance, pod) * 100 > 10
      for: 10m
      labels:
        severity: critical
    - alert: KubeAPIErrorsHigh
      annotations:
        message: API server is returning errors for {{ $value }}% of requests.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
      expr: |
        sum(rate(apiserver_request_count{job="apiserver",code=~"^(?:5..)$"}[5m])) without(instance, pod)
          /
        sum(rate(apiserver_request_count{job="apiserver"}[5m])) without(instance, pod) * 100 > 5
      for: 10m
      labels:
        severity: warning
    - alert: KubeClientCertificateExpiration
      annotations:
        message: Kubernetes API certificate is expiring in less than 7 days.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclientcertificateexpiration
      expr: |
        histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 604800
      labels:
        severity: warning
    - alert: KubeClientCertificateExpiration
      annotations:
        message: Kubernetes API certificate is expiring in less than 24 hours.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclientcertificateexpiration
      expr: |
        histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 86400
      labels:
        severity: critical
```

### Kubernetes Node节点数据合成和警报

```yaml
  - name: node.rules
    rules:
    - expr: sum(min(kube_pod_info) by (node))
      record: ':kube_pod_info_node_count:'
    - expr: |
        max(label_replace(kube_pod_info{job="kube-state-metrics"}, "pod", "$1", "pod", "(.*)")) by (node, namespace, pod)
      record: 'node_namespace_pod:kube_pod_info:'
    - expr: |
        count by (node) (sum by (node, cpu) (
          node_cpu_seconds_total{job="node-exporter"}
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        ))
      record: node:node_num_cpu:sum
    - expr: |
        1 - avg(rate(node_cpu_seconds_total{job="node-exporter",mode="idle"}[1m]))
      record: :node_cpu_utilisation:avg1m
    - expr: |
        1 - avg by (node) (
          rate(node_cpu_seconds_total{job="node-exporter",mode="idle"}[1m])
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:)
      record: node:node_cpu_utilisation:avg1m
    - expr: |
        sum(node_load1{job="node-exporter"})
        /
        sum(node:node_num_cpu:sum)
      record: ':node_cpu_saturation_load1:'
    - expr: |
        sum by (node) (
          node_load1{job="node-exporter"}
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
        /
        node:node_num_cpu:sum
      record: 'node:node_cpu_saturation_load1:'
    - expr: |
        1 -
        sum(node_memory_MemFree_bytes{job="node-exporter"} + node_memory_Cached_bytes{job="node-exporter"} + node_memory_Buffers_bytes{job="node-exporter"})
        /
        sum(node_memory_MemTotal_bytes{job="node-exporter"})
      record: ':node_memory_utilisation:'
    - expr: |
        sum(node_memory_MemFree_bytes{job="node-exporter"} + node_memory_Cached_bytes{job="node-exporter"} + node_memory_Buffers_bytes{job="node-exporter"})
      record: :node_memory_MemFreeCachedBuffers_bytes:sum
    - expr: |
        sum(node_memory_MemTotal_bytes{job="node-exporter"})
      record: :node_memory_MemTotal_bytes:sum
    - expr: |
        sum by (node) (
          (node_memory_MemFree_bytes{job="node-exporter"} + node_memory_Cached_bytes{job="node-exporter"} + node_memory_Buffers_bytes{job="node-exporter"})
          * on (namespace, pod) group_left(node)
            node_namespace_pod:kube_pod_info:
        )
      record: node:node_memory_bytes_available:sum
    - expr: |
        sum by (node) (
          node_memory_MemTotal_bytes{job="node-exporter"}
          * on (namespace, pod) group_left(node)
            node_namespace_pod:kube_pod_info:
        )
      record: node:node_memory_bytes_total:sum
    - expr: |
        (node:node_memory_bytes_total:sum - node:node_memory_bytes_available:sum)
        /
        scalar(sum(node:node_memory_bytes_total:sum))
      record: node:node_memory_utilisation:ratio
    - expr: |
        1e3 * sum(
          (rate(node_vmstat_pgpgin{job="node-exporter"}[1m])
         + rate(node_vmstat_pgpgout{job="node-exporter"}[1m]))
        )
      record: :node_memory_swap_io_bytes:sum_rate
    - expr: |
        1 -
        sum by (node) (
          (node_memory_MemFree_bytes{job="node-exporter"} + node_memory_Cached_bytes{job="node-exporter"} + node_memory_Buffers_bytes{job="node-exporter"})
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
        /
        sum by (node) (
          node_memory_MemTotal_bytes{job="node-exporter"}
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
      record: 'node:node_memory_utilisation:'
    - expr: |
        1 - (node:node_memory_bytes_available:sum / node:node_memory_bytes_total:sum)
      record: 'node:node_memory_utilisation_2:'
    - expr: |
        1e3 * sum by (node) (
          (rate(node_vmstat_pgpgin{job="node-exporter"}[1m])
         + rate(node_vmstat_pgpgout{job="node-exporter"}[1m]))
         * on (namespace, pod) group_left(node)
           node_namespace_pod:kube_pod_info:
        )
      record: node:node_memory_swap_io_bytes:sum_rate
    - expr: |
        avg(irate(node_disk_io_time_seconds_total{job="node-exporter",device=~"(sd|xvd|nvme).+"}[1m]))
      record: :node_disk_utilisation:avg_irate
    - expr: |
        avg by (node) (
          irate(node_disk_io_time_seconds_total{job="node-exporter",device=~"(sd|xvd|nvme).+"}[1m])
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
      record: node:node_disk_utilisation:avg_irate
    - expr: |
        100 - round((node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) / node_memory_MemTotal_bytes * 100 )
      record: NodeMemUsage
    - expr: |
        avg(irate(node_disk_io_time_weighted_seconds_total{job="node-exporter",device=~"(sd|xvd|nvme).+"}[1m]) / 1e3)
      record: :node_disk_saturation:avg_irate
    - expr: |
        avg by (node) (
          irate(node_disk_io_time_weighted_seconds_total{job="node-exporter",device=~"(sd|xvd|nvme).+"}[1m]) / 1e3
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
      record: node:node_disk_saturation:avg_irate
    - expr: |
        max by (namespace, pod, device, instance) ((node_filesystem_size_bytes{fstype=~"ext[234]|btrfs|xfs|zfs"}
        - node_filesystem_avail_bytes{fstype=~"ext[234]|btrfs|xfs|zfs"})
        / node_filesystem_size_bytes{fstype=~"ext[234]|btrfs|xfs|zfs"})
      record: 'node:node_filesystem_usage:'
    - expr: |
        max by (namespace, pod, device) (node_filesystem_avail_bytes{fstype=~"ext[234]|btrfs|xfs|zfs"} / node_filesystem_size_bytes{fstype=~"ext[234]|btrfs|xfs|zfs"})
      record: 'node:node_filesystem_avail:'
    - expr: |
        sum(irate(node_network_receive_bytes_total{job="node-exporter",device="eth0"}[1m])) +
        sum(irate(node_network_transmit_bytes_total{job="node-exporter",device="eth0"}[1m]))
      record: :node_net_utilisation:sum_irate
    - expr: |
        sum by (node) (
          (irate(node_network_receive_bytes_total{job="node-exporter",device="eth0"}[1m]) +
          irate(node_network_transmit_bytes_total{job="node-exporter",device="eth0"}[1m]))
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
      record: node:node_net_utilisation:sum_irate
    - expr: |
        sum(irate(node_network_receive_drop_total{job="node-exporter",device="eth0"}[1m])) +
        sum(irate(node_network_transmit_drop_total{job="node-exporter",device="eth0"}[1m]))
      record: :node_net_saturation:sum_irate
    - expr: |
        sum by (node) (
          (irate(node_network_receive_drop_total{job="node-exporter",device="eth0"}[1m]) +
          irate(node_network_transmit_drop_total{job="node-exporter",device="eth0"}[1m]))
        * on (namespace, pod) group_left(node)
          node_namespace_pod:kube_pod_info:
        )
      record: node:node_net_saturation:sum_irate
    - expr: sum(rate(node_cpu{mode!="idle",mode!="iowait"}[3m])) BY (instance)
      record: instance:node_cpu:rate:sum
    - expr: sum((node_filesystem_size{mountpoint="/"} - node_filesystem_free{mountpoint="/"}))
          BY (instance)
      record: instance:node_filesystem_usage:sum
    - expr: sum(rate(node_network_receive_bytes[3m])) BY (instance)
      record: instance:node_network_receive_bytes:rate:sum
    - expr: sum(rate(node_network_transmit_bytes[3m])) BY (instance)
      record: instance:node_network_transmit_bytes:rate:sum
    - expr: sum(rate(node_cpu{mode!="idle",mode!="iowait"}[5m])) WITHOUT (cpu, mode)
          / ON(instance) GROUP_LEFT() count(sum(node_cpu) BY (instance, cpu)) BY (instance)
      record: instance:node_cpu:ratio
    - expr: sum(rate(node_cpu{mode!="idle",mode!="iowait"}[5m]))
      record: cluster:node_cpu:sum_rate5m
    - expr: cluster:node_cpu:rate5m / count(sum(node_cpu) BY (instance, cpu))
      record: cluster:node_cpu:ratio
    - expr: round(1 - node_filesystem_free_bytes{device=~"/dev/.+",device!~"/dev/mapper/docker.+", mountpoint="/etc/hostname"} / node_filesystem_size_bytes{device=~"/dev/.+",device!~"/dev/mapper/docker.+", mountpoint="/etc/hostname"},0.01)
      record: NodeDiskUsage
    - alert: NodeDiskRunningFull
      annotations:
        message: Device {{ $labels.device }} of node-exporter {{ $labels.namespace
          }}/{{ $labels.pod }}/{{$labels.instance}} will be full within the next 24 hours.
      expr: |
        (node:node_filesystem_usage: > 0.85) and (predict_linear(node:node_filesystem_avail:[6h], 3600 * 24) < 0)
      for: 30m
      labels:
        severity: warning
    - alert: NodeDiskRunningFull
      annotations:
        message: Device {{ $labels.device }} of node-exporter {{ $labels.namespace
          }}/{{ $labels.pod }}/{{$labels.instance}} will be full within the next 2 hours.
      expr: |
        (node:node_filesystem_usage: > 0.85) and (predict_linear(node:node_filesystem_avail:[30m], 3600 * 2) < 0)
      for: 10m
      labels:
        severity: critical
    - alert: NodeAvailMemLow
      annotations:
        message: 'Memory usage on {{$labels.instance}} has been {{ $value }}%  for 10 mins.'
      expr: NodeMemUsage > 80
      for: 10m
      labels:
        severity: warning
    - alert: NodeDiskUsageHigh
      annotations:
        message: 'Disk usage on {{$labels.instance}} {{$labels.device}} is {{ $value }}%.'
      expr: NodeDiskUsage > 80
      for: 10m
      labels:
        severity: warning
    - alert: KubeNodeAbnormal
      annotations:
        message: Pod {{ $labels.pod }} detected a node kernel error /unregister_netdevice/ in {{$labels.cluster}} Cluster.
      expr: |
        round(increase(fluentd_node_kernel_error_detector[5m])) > 0
      for: 0s
      labels:
        severity: warning
```

### Prometheus 集群自身警报及数据二次合成

```yaml
  - name: kube-prometheus-node-recording.rules
    rules:
    - alert: BlackboxChecks
      annotations:
        message: Instance {{ $labels.instance }} http_2xx check failed {{$labels.namespace}} by blackbox.
      expr: |
        probe_success < 1
      for: 2m
      labels:
        severity: warning
  - name: kubernetes-absent
    rules:
    - alert: AlertmanagerDown
      annotations:
        message: Alertmanager has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-alertmanagerdown
      expr: |
        absent(up{job="alertmanager-main"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: KubeStateMetricsDown
      annotations:
        message: KubeStateMetrics has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatemetricsdown
      expr: |
        absent(up{job="kube-state-metrics"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: KubeletDown
      annotations:
        message: Kubelet has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeletdown
      expr: |
        absent(up{job="kubelet"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: NodeExporterDown
      annotations:
        message: NodeExporter has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodeexporterdown
      expr: |
        absent(up{job="node-exporter"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusDown
      annotations:
        message: Prometheus has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-prometheusdown
      expr: |
        absent(up{job="prometheus-k8s"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusOperatorDown
      annotations:
        message: PrometheusOperator has disappeared from Prometheus target discovery.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-prometheusoperatordown
      expr: |
        absent(up{job="prometheus-operator"} == 1)
      for: 15m
      labels:
        severity: critical
    - alert: PrometheusOperatorReconcileErrors
      annotations:
        message: Errors while reconciling {{ $labels.controller }} in {{ $labels.namespace
          }} Namespace.
      expr: |
        rate(prometheus_operator_reconcile_errors_total{job="prometheus-operator"}[5m]) > 0.1
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusOperatorNodeLookupErrors
      annotations:
        message: Errors while reconciling Prometheus in {{ $labels.namespace }} Namespace.
      expr: |
        rate(prometheus_operator_node_address_lookup_errors_total{job="prometheus-operator"}[5m]) > 0.1
      for: 10m
      labels:
        severity: warning
    - alert: AlertmanagerConfigInconsistent
      annotations:
        message: The configuration of the instances of the Alertmanager cluster `{{$labels.service}}`
          are out of sync.
      expr: |
        count_values("config_hash", alertmanager_config_hash{job="alertmanager-main"}) BY (service) / ON(service) GROUP_LEFT() label_replace(prometheus_operator_spec_replicas{job="prometheus-operator",controller="alertmanager"}, "service", "alertmanager-$1", "name", "(.*)") != 1
      for: 5m
      labels:
        severity: critical
    - alert: AlertmanagerFailedReload
      annotations:
        message: Reloading Alertmanager's configuration has failed for {{ $labels.namespace
          }}/{{ $labels.pod}}.
      expr: |
        alertmanager_config_last_reload_successful{job="alertmanager-main"} == 0
      for: 10m
      labels:
        severity: warning
    - alert: AlertmanagerMembersInconsistent
      annotations:
        message: Alertmanager has not found all other members of the cluster.
      expr: |
        alertmanager_cluster_members{job="alertmanager-main"}
          != on (service) GROUP_LEFT()
        count by (service) (alertmanager_cluster_members{job="alertmanager-main"})
      for: 5m
      labels:
        severity: critical
    - alert: PrometheusConfigReloadFailed
      annotations:
        message: Reloading Prometheus' configuration has failed for {{$labels.namespace}}/{{$labels.pod}}
        summary: Reloading Prometheus' configuration failed
      expr: |
        prometheus_config_last_reload_successful{job="prometheus-k8s"} == 0
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusNotificationQueueRunningFull
      annotations:
        message: Prometheus' alert notification queue is running full for {{$labels.namespace}}/{{
          $labels.pod}}
        summary: Prometheus' alert notification queue is running full
      expr: |
        predict_linear(prometheus_notifications_queue_length{job="prometheus-k8s"}[5m], 60 * 30) > prometheus_notifications_queue_capacity{job="prometheus-k8s"}
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusErrorSendingAlerts
      annotations:
        message: Errors while sending alerts from Prometheus {{$labels.namespace}}/{{
          $labels.pod}} to Alertmanager {{$labels.Alertmanager}}
        summary: Errors while sending alert from Prometheus
      expr: |
        rate(prometheus_notifications_errors_total{job="prometheus-k8s"}[5m]) / rate(prometheus_notifications_sent_total{job="prometheus-k8s"}[5m]) > 0.01
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusErrorSendingAlerts
      annotations:
        message: Errors while sending alerts from Prometheus {{$labels.namespace}}/{{
          $labels.pod}} to Alertmanager {{$labels.Alertmanager}}
        summary: Errors while sending alerts from Prometheus
      expr: |
        rate(prometheus_notifications_errors_total{job="prometheus-k8s"}[5m]) / rate(prometheus_notifications_sent_total{job="prometheus-k8s"}[5m]) > 0.03
      for: 10m
      labels:
        severity: critical
    - alert: PrometheusNotConnectedToAlertmanagers
      annotations:
        message: Prometheus {{ $labels.namespace }}/{{ $labels.pod}} is not connected
          to any Alertmanagers
        summary: Prometheus is not connected to any Alertmanagers
      expr: |
        prometheus_notifications_alertmanagers_discovered{job="prometheus-k8s"} < 1
      for: 10m
      labels:
        severity: warning
    - alert: PrometheusTSDBReloadsFailing
      annotations:
        message: '{{$labels.job}} at {{$labels.instance}} had {{$value | humanize}}
          reload failures over the last four hours.'
        summary: Prometheus has issues reloading data blocks from disk
      expr: |
        increase(prometheus_tsdb_reloads_failures_total{job="prometheus-k8s"}[2h]) > 0
      for: 12h
      labels:
        severity: warning
    - alert: PrometheusTSDBCompactionsFailing
      annotations:
        message: '{{$labels.job}} at {{$labels.instance}} had {{$value | humanize}}
          compaction failures over the last four hours.'
        summary: Prometheus has issues compacting sample blocks
      expr: |
        increase(prometheus_tsdb_compactions_failed_total{job="prometheus-k8s"}[2h]) > 0
      for: 12h
      labels:
        severity: warning
    - alert: PrometheusTSDBWALCorruptions
      annotations:
        message: '{{$labels.job}} at {{$labels.instance}} has a corrupted write-ahead
          log (WAL).'
        summary: Prometheus write-ahead log is corrupted
      expr: |
        tsdb_wal_corruptions_total{job="prometheus-k8s"} > 0
      for: 4h
      labels:
        severity: warning
    - alert: PrometheusNotIngestingSamples
      annotations:
        message: Prometheus {{ $labels.namespace }}/{{ $labels.pod}} isn't ingesting
          samples.
        summary: Prometheus isn't ingesting samples
      expr: |
        rate(prometheus_tsdb_head_samples_appended_total{job="prometheus-k8s"}[5m]) <= 0
      for: 10m
      labels:
        severity: warning
#    - alert: PrometheusTargetScrapesDuplicate
#      annotations:
#        message: '{{$labels.namespace}}/{{$labels.pod}} has many samples rejected
#          due to duplicate timestamps but different values'
#        summary: Prometheus has many samples rejected
#      expr: |
#        increase(prometheus_target_scrapes_sample_duplicate_timestamp_total{job="prometheus-k8s"}[5m]) > 0
#      for: 10m
#      labels:
#        severity: warning
```

### Kubernetes 集群内部运行应用监控警报

```yaml
  - name: kubernetes-apps

    rules:
    - alert: KubePodCrashLooping
      annotations:
#        message: Pod {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container
#          }}) is restarting {{ printf "%.2f" $value }} times / 5 minutes.
        message: Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting {{ $value }} times / 5 minutes.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepodcrashlooping
      expr: |
        round(rate(kube_pod_container_status_restarts_total{job="kube-state-metrics"}[5m]),1) > 0
      for: 5m
      labels:
        severity: critical
    - alert: KubePodNotReady
      annotations:
        message: Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready
          state for longer than 5m.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepodnotready
      expr: |
        sum by (namespace, pod) (kube_pod_status_phase{job="kube-state-metrics", phase=~"Pending|Unknown"}) > 0
      for: 5m
      labels:
        severity: critical
    - alert: KubeDeploymentGenerationMismatch
      annotations:
        message: Deployment generation for {{ $labels.namespace }}/{{ $labels.deployment
          }} does not match, this indicates that the Deployment has failed but has
          not been rolled back.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedeploymentgenerationmismatch
      expr: |
        kube_deployment_status_observed_generation{job="kube-state-metrics"}
          !=
        kube_deployment_metadata_generation{job="kube-state-metrics"}
      for: 15m
      labels:
        severity: critical
#    - alert: KubeDeploymentReplicasMismatch
#      annotations:
#        message: Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has not
#          matched the expected number of replicas for longer than 5m.
#        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedeploymentreplicasmismatch
#      expr: |
#        kube_deployment_spec_replicas{job="kube-state-metrics"}
#          !=
#        kube_deployment_status_replicas_available{job="kube-state-metrics"}
#      for: 5m
#      labels:
#        severity: critical
    - alert: KubeStatefulSetReplicasMismatch
      annotations:
        message: StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} has
          not matched the expected number of replicas for longer than 15 minutes.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetreplicasmismatch
      expr: |
        kube_statefulset_status_replicas_ready{job="kube-state-metrics"}
          !=
        kube_statefulset_status_replicas{job="kube-state-metrics"}
      for: 15m
      labels:
        severity: critical
    - alert: KubeStatefulSetGenerationMismatch
      annotations:
        message: StatefulSet generation for {{ $labels.namespace }}/{{ $labels.statefulset
          }} does not match, this indicates that the StatefulSet has failed but has
          not been rolled back.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetgenerationmismatch
      expr: |
        kube_statefulset_status_observed_generation{job="kube-state-metrics"}
          !=
        kube_statefulset_metadata_generation{job="kube-state-metrics"}
      for: 15m
      labels:
        severity: critical
    - alert: KubeStatefulSetUpdateNotRolledOut
      annotations:
        message: StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} update
          has not been rolled out.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetupdatenotrolledout
      expr: |
        max without (revision) (
          kube_statefulset_status_current_revision{job="kube-state-metrics"}
            unless
          kube_statefulset_status_update_revision{job="kube-state-metrics"}
        )
          *
        (
          kube_statefulset_replicas{job="kube-state-metrics"}
            !=
          kube_statefulset_status_replicas_updated{job="kube-state-metrics"}
        )
      for: 15m
      labels:
        severity: critical
    - alert: KubeDaemonSetRolloutStuck
      annotations:
        message: Only {{ $value }}% of the desired Pods of DaemonSet {{ $labels.namespace
          }}/{{ $labels.daemonset }} are scheduled and ready.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetrolloutstuck
      expr: |
        kube_daemonset_status_number_ready{job="kube-state-metrics"}
          /
        kube_daemonset_status_desired_number_scheduled{job="kube-state-metrics"} * 100 < 100
      for: 15m
      labels:
        severity: critical
    - alert: KubeDaemonSetNotScheduled
      annotations:
        message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
          }} are not scheduled.'
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetnotscheduled
      expr: |
        kube_daemonset_status_desired_number_scheduled{job="kube-state-metrics"}
          -
        kube_daemonset_status_current_number_scheduled{job="kube-state-metrics"} > 0
      for: 10m
      labels:
        severity: warning
    - alert: KubeDaemonSetMisScheduled
      annotations:
        message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
          }} are running where they are not supposed to run.'
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetmisscheduled
      expr: |
        kube_daemonset_status_number_misscheduled{job="kube-state-metrics"} > 0
      for: 10m
      labels:
        severity: warning
    - alert: KubeCronJobRunning
      annotations:
        message: CronJob {{ $labels.namespace }}/{{ $labels.cronjob }} is taking more
          than 1h to complete.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecronjobrunning
      expr: |
        time() - kube_cronjob_next_schedule_time{job="kube-state-metrics"} > 3600
      for: 1h
      labels:
        severity: warning
    - alert: KubeJobCompletion
      annotations:
        message: Job {{ $labels.namespace }}/{{ $labels.job_name }} is taking more
          than one hour to complete.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubejobcompletion
      expr: |
        kube_job_spec_completions{job="kube-state-metrics"} - kube_job_status_succeeded{job="kube-state-metrics"}  > 0
      for: 1h
      labels:
        severity: warning
    - alert: KubeJobFailed
      annotations:
        message: Job {{ $labels.namespace }}/{{ $labels.job_name }} failed to complete.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubejobfailed
      expr: |
        kube_job_status_failed{job="kube-state-metrics"}  > 0
      for: 1h
      labels:
        severity: warning
#    - alert: KubeIngressPerformance
#      annotations:
#        message: Performance of ingress {{$labels.server_zone}} status_code 2xx is degraded, responsed time {{$value}} threshold is 2s.
#      expr: |
#        round(sum by (server_zone)(rate(nginx_responses_total{status_code="2xx",server_zone != "*", server_zone != "_"}[1m])), 0.01) > 2
#      labels:
#        severity: warning
    - alert: KubeUpstreamResponseTime
      annotations:
        message: '响应时间 {{ $labels.upstream }}: {{$value}}s/2m, 阈值: 1s/2m.'
      expr: |
        round(sum by (upstream) (nginx_upstream_response_msecs_avg)/1000, 0.01) > 1
      for: 2m
      labels:
        severity: warning
    - alert: TargetDown
      annotations:
        message: '{{ $value }}% of the {{ $labels.job }} targets are down.'
      expr: 100 * (count(up == 0) BY (job) / count(up) BY (job)) > 10
      for: 10m
      labels:
        severity: warning
    - alert: NetworkPodPortPingFailed
      annotations:
        message: 'Network check was failed from {{$labels.instance}} on node {{$labels.src_node}} to {{$labels.dest_ip_port}} on node {{$labels.dest_node}}.'
      expr: networktest_pod_port_fail > 0
      for: 0s
      labels:
        severity: warning

    - alert: Ingressxxx
      annotations:
        message: '{{$labels.server_zone}} got {{ $value }} {{$labels.status_code}} hits.'
      expr: round(increase(nginx_responses_total{status_code=~"5xx|4xx",server_zone!="*",server_zone!~"_|aroute.ke.com|i.aroute.ke.com"}[5m]),1) > 50
      for: 1s
      labels:
        severity: warning
    - alert: KubePodAbnormal
      annotations:
        message: Pod {{ $labels.pod }} in {{$labels.namespace}} was {{$labels.reason}}.
      expr: |
        sum_over_time(kube_pod_container_status_terminated_reason{reason!~"Completed|Error"}[5m]) > 0
      for: 1m
      labels:
        severity: warning
```

### Kubernetes集群计算资源警报

```yaml
  - name: kubernetes-resources
    rules:
    - alert: KubeCPUOvercommit
      annotations:
        message: Cluster has overcommitted CPU resource requests for Pods and cannot
          tolerate node failure.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecpuovercommit
      expr: |
        sum(namespace_name:kube_pod_container_resource_requests_cpu_cores:sum)
          /
        sum(node:node_num_cpu:sum)
          >
        (count(node:node_num_cpu:sum)-1) / count(node:node_num_cpu:sum)
      for: 5m
      labels:
        severity: warning
    - alert: KubeMemOvercommit
      annotations:
        message: Cluster has overcommitted memory resource requests for Pods and cannot
          tolerate node failure.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubememovercommit
      expr: |
        sum(namespace_name:kube_pod_container_resource_requests_memory_bytes:sum)
          /
        sum(node_memory_MemTotal_bytes)
          >
        (count(node:node_num_cpu:sum)-1)
          /
        count(node:node_num_cpu:sum)
      for: 5m
      labels:
        severity: warning
    - alert: KubeCPUOvercommit
      annotations:
        message: Cluster has overcommitted CPU resource requests for Namespaces.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecpuovercommit
      expr: |
        sum(kube_resourcequota{job="kube-state-metrics", type="hard", resource="requests.cpu"})
          /
        sum(node:node_num_cpu:sum)
          > 1.5
      for: 5m
      labels:
        severity: warning
    - alert: KubeMemOvercommit
      annotations:
        message: Cluster has overcommitted memory resource requests for Namespaces.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubememovercommit
      expr: |
        sum(kube_resourcequota{job="kube-state-metrics", type="hard", resource="requests.memory"})
          /
        sum(node_memory_MemTotal_bytes{job="node-exporter"})
          > 1.5
      for: 5m
      labels:
        severity: warning
    - alert: KubeQuotaExceeded
      annotations:
        message: Namespace {{ $labels.namespace }} is using {{ printf "%0.0f" $value
          }}% of its {{ $labels.resource }} quota.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubequotaexceeded
      expr: |
        100 * kube_resourcequota{job="kube-state-metrics", type="used"}
          / ignoring(instance, job, type)
        (kube_resourcequota{job="kube-state-metrics", type="hard"} > 0)
          > 90
      for: 15m
      labels:
        severity: warning
#    - alert: CPUThrottlingHigh
#      annotations:
#        message: '{{ printf "%0.0f" $value }}% throttling of CPU in namespace {{ $labels.namespace
#          }} for container {{ $labels.container_name }} in pod {{ $labels.pod_name
#          }}.'
#        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-cputhrottlinghigh
#      expr: "100 * sum(increase(container_cpu_cfs_throttled_periods_total{}[5m]))
#        by (container_name, pod_name, namespace) \n  / \nsum(increase(container_cpu_cfs_periods_total{}[5m]))
#        by (container_name, pod_name, namespace)\n  > 25 \n"
#      for: 15m
#      labels:
#        severity: warning
```

### Kubernetes集群存储资源警报

```yaml
  - name: kubernetes-storage
    rules:
    - alert: KubePersistentVolumeUsageCritical
      annotations:
        message: The PersistentVolume claimed by {{ $labels.persistentvolumeclaim
          }} in Namespace {{ $labels.namespace }} is only {{ printf "%0.2f" $value
          }}% free.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumeusagecritical
      expr: |
        100 * kubelet_volume_stats_available_bytes{job="kubelet"}
          /
        kubelet_volume_stats_capacity_bytes{job="kubelet"}
          < 3
      for: 1m
      labels:
        severity: critical
    - alert: KubePersistentVolumeFullInFourDays
      annotations:
        message: Based on recent sampling, the PersistentVolume claimed by {{ $labels.persistentvolumeclaim
          }} in Namespace {{ $labels.namespace }} is expected to fill up within four
          days. Currently {{ printf "%0.2f" $value }}% is available.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumefullinfourdays
      expr: |
        100 * (
          kubelet_volume_stats_available_bytes{job="kubelet"}
            /
          kubelet_volume_stats_capacity_bytes{job="kubelet"}
        ) < 15
        and
        predict_linear(kubelet_volume_stats_available_bytes{job="kubelet"}[6h], 4 * 24 * 3600) < 0
      for: 5m
      labels:
        severity: critical
    - alert: KubePersistentVolumeErrors
      annotations:
        message: The persistent volume {{ $labels.persistentvolume }} has status {{
          $labels.phase }}.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumeerrors
      expr: |
        kube_persistentvolume_status_phase{phase=~"Failed|Pending",job="kube-state-metrics"} > 0
      for: 5m
      labels:
        severity: critical
```

### Java 应用警报规则

```yaml
  - name: java-app-alert
    rules:
    - alert: JavaComponentHealthLagTooLong
      expr: component_healthcheck_lag_mseconds{compType="system",compName="component-healthcheck-lag"} > 1800000
      for: 30m
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "1800000"
        description: "组件健康状态已超过30分钟（{{$value}}）未成功更新，请及时排查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

      # 组件健康
    - alert: JavaComponentDown
      expr: health_status_total{job="jvm",compType="component.health"} == 1000
      for: 0s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "1000"
        description: "组件：{{$labels.compName}}，状态：Down(不可用) ，请及时排查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"
    # 内存报警
    - alert: JavaFreeMemoryInsufficient
      expr: (mem_free_bytes_total{job="jvm",compType="system.memory",compName="system"} / mem_bytes_total) < 0.09
      for: 1m
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "0.09"
        description: "空闲内存不足9%，空闲内存 / 总内存 = {{$value}}，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"
    # 线程死锁报警 ，有些JVM收集不到这个参数
    - alert: JavaThreadDeadlock
      expr: threads_deadlock_total{job="jvm",compType="system.thread",compName="system"}> 0
      for: 0s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "1"
        description: "死锁的线程数：{{$value}}，请及时排查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Old(full) GC 报警
    - alert: JavaG1OldGCOccur
      expr: increase(gc_count_total{job="jvm",compType="system.gc",compName="g1_old_generation"}[15m])> 2
      for: 15m
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "2"
        description: "近15分钟Full GC次数：{{$value}}，请及时排查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # 30秒内Error级别的日志超过60条
    - alert: JavaErrorLogEventIncrease
      expr:  increase(error_total{job="jvm",compType="logging",compName="logback"}[35s])> 80
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "80"
        description: "Error级别的日志在增加，30秒内已累计产生{{$value}}条，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # 生产环境禁止输出Debug级别的日志
    - alert: JavaDebugLogEventIncrease
      expr:  increase(debug_total{job="jvm",compType="logging",compName="logback"}[35s])> 50
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "50"
        description: "30秒内已累计产生{{$value}}条Debug日志，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"


    # 数据库连接池使用率过高（> 80%）
    - alert: JavaDataSourcePoolUtilizationExcessive
      expr: (pool_active_total{job="jvm",compType="db.datasource",category="pool"} / pool_max_total)> 0.8
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "0.8"
        description: "数据库连接池（{{$labels.compName}}）使用率(活跃连接数 / 最大连接数)超过80% ，当前值：{{$value}}，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # MongoDB 连接池使用率过高（> 80%)
    - alert: JavaMongoDBPoolUtilizationExcessive
      expr:  (pool_active_total{job="jvm",compType="db.mongo",category="pool"} / pool_max_total)> 0.8
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "0.8"
        description: "MongoDB连接池（{{$labels.compName}}）使用率(已用连接数 / 最大连接数)超过80% ，当前值：{{$value}}，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Tomcat线程池使用率过高（> 80%)
    - alert: JavaTomcatThreadPoolUtilizationExcessive
      expr:  (threads_total{job="jvm",compType="web.tomcat",compName="tomcat.default",category="threadpool"} / threads_max_total)> 0.8
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "0.8"
        description: "Tomcat 线程池使用率(当前线程数 / 最大线程数)超过80% ，当前值：{{$value}}，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Tomcat 30秒内失败请求数超过 50个
    - alert: JavaTomcatErrorRequestIncrease
      expr:  increase(request_error_total{job="jvm",compType="web.tomcat",compName="tomcat.default",category="request.stats"}[35s])> 100
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "100"
        description: "Tomcat的失败请求数(httpstatus>=400)激增，30秒内失败请求数已达{{$value}}个，请及时检查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"


    # 线程池使用率过高（> 80%)
    - alert: JavaThreadPoolUtilizationExcessive
      expr: (threadpool_current_total{job="jvm",compType="threadpool"} / threadpool_max_total ) > 0.8
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "0.8"
        description: "线程池({{$labels.compName}})使用率(当前线程数/最大线程数)过高(> 80%)，当前值：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # 线程池处理不及时，队列中任务数超过20个了
    - alert: JavaThreadPoolTaskQueued
      expr: threadpool_task_queued_total{job="jvm",compType="threadpool"}  > 20
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "20"
        description: "线程池({{$labels.compName}})处理不及时，当前已有{{$value}}个任务入队等待执行中。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Redis连接池使用率过高（ > 80%)
    - alert: JavaRedisPoolUtilizationExcessive
      expr: (con_total{job="jvm",compType="cache.redis.jedis",compName="Jedis",category="stats.pool"} / con_max_total ) > 0.8
      for: 25s
      labels:
        severity: warning
      annotations:
        value: "{{$value}}"
        threshold: "0.8"
        description: "Jedis连接池使用率(已用连接/最大连接数)过高(> 80%)，目前使用率：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Redis连接池已耗尽，线程开始阻塞等待
    - alert: JavaRedisPoolExhausted
      expr: (threads_waits{job="jvm",compType="cache.redis.jedis",compName="Jedis",category="stats.pool"}) > 0
      for: 0s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "1"
        description: "Jedis连接池已耗尽，目前阻塞等待的线程数：{{$value}}，请排查。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Hystrix调用第三方服务时30秒内失败（断路、超时、接口异常）请求数超过10个
    - alert: JavaHystrixErrorRequestIncrease
      expr: increase(request_error_total{job="jvm",compType="circuitbreaker.hystrix",category="stats"}[35s])> 10
      for: 0s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "10"
        description: "Hystrix第三方调用失败, command: {{$labels.compName}}，group：{{$labels.commandGroup}} ， 累计失败请求数：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Kafka 生产者 TOPIC 发送记录出错
    - alert: JavaKafkaProducerTopicSendError
      expr: sum_over_time(record_errorRate_total{job="jvm",compType="mq.kafka",category="producer.topic.stats"}[35s])> 5
      for: 0s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "5"
        description: "Kafka Topic: {{$labels.compName}}，近30秒累计发送失败：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

    # Kafka 生产者因buffer耗尽而丢弃的消息数
    - alert: JavaKafkaProducerBufferExhaustDroppedRecords
      expr: sum_over_time(buffer_exhaustedRate_total{job="jvm",compType="mq.kafka",category="producer.stats"}[35s])> 3
      for: 10s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "3"
        description: "应用: {{$labels.compName}}，30秒内因Buffer耗尽丢弃的记录数：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"

     # Kafka 生产者正在阻塞等待入队Buffer的线程数
    - alert: JavaKafkaProducerBufferWaitingThreads
      expr: buffer_waitingThread_total{job="jvm",compType="mq.kafka",category="producer.stats"} > 10
      for: 25s
      labels:
        severity: critical
      annotations:
        value: "{{$value}}"
        threshold: "10"
        description: "应用: {{$labels.compName}}，正在阻塞等待入队Buffer的线程数：{{$value}}。应用：{{$labels.service}}，节点：{{$labels.pod}}，命名空间：{{$labels.namespace}}。"
    # Kafka 消费者
```
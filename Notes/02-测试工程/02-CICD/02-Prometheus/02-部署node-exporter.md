Node Exporter 是 Prometheus 官方生态中的指标采集组件。
它负责读取 Linux 系统信息，并通过 HTTP `/metrics` 接口暴露指标。
整体工作流程为：
宿主机 Linux → Node Exporter → Prometheus → Grafana。

Node Exporter 默认监听 `9100` 端口。
例如，它可以暴露以下指标：

|指标|含义|
|---|---|
|`node_cpu_seconds_total`|CPU 各种运行状态累计时间|
|`node_memory_MemTotal_bytes`|宿主机总内存|
|`node_memory_MemAvailable_bytes`|宿主机可用内存|
|`node_filesystem_size_bytes`|文件系统总容量|
|`node_filesystem_avail_bytes`|文件系统可用空间|
|`node_network_receive_bytes_total`|网络接收累计字节数|
|`node_network_transmit_bytes_total`|网络发送累计字节数|

这些指标大多是 Counter 或 Gauge。

例如 `node_cpu_seconds_total` 是 Counter，因此后续通常使用 PromQL 的 `rate()` 计算 CPU 使用速率，而不是直接查看累计值。

部署配置：`compose`
```yml
  node-exporter:
      image: prom/node-exporter:v1.10.2
      container_name: node-exporter
      restart: unless-stopped
      pid: host
      command:
        - "--path.rootfs=/host"
      volumes:
        - "/:/host:ro"
      ports:
        - "127.0.0.1:9100:9100"
      security_opt:
        - no-new-privileges:true
```


## 计算宿主机 CPU 使用率

Node Exporter 提供：
```
node_cpu_seconds_total
```
它是一个 Counter，并且带有 `mode` Label，用于区分 CPU 状态。

常见状态包括：

|Mode|含义|
|---|---|
|idle|空闲|
|user|用户态|
|system|内核态|
|iowait|等待 I/O|
|irq|硬件中断|
|softirq|软中断|

### 1. 查看 CPU 空闲时间增长速率

```
rate(node_cpu_seconds_total{
  job="node-exporter",
  mode="idle"
}[5m])
```

### 2. 计算 CPU 使用率

CPU 使用率可以理解为：`CPU使用率 = 1 - CPU空闲率`

PromQL：
```
100 * (
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{
      job="node-exporter",
      mode="idle"
    }[5m])
  )
)
```
返回结果单位是百分比。
这里的 `avg by (instance)` 是对同一宿主机所有 CPU 核心的空闲比例求平均。

例如：
```
25.6
```
表示宿主机整体 CPU 使用率约为 25.6%。

注意：这是基于非 idle 时间计算的总体 CPU 使用率，与不同操作系统工具对 iowait 等状态的统计口径可能存在差异。
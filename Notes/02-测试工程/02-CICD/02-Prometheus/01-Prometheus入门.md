# 理解Prometheus
**Prometheus** 是一个开源监控系统，其核心工作方式是定期主动访问监控目标，抓取目标暴露的指标，并按照时间序列存储。
例如，我们希望监控 FastAPI 容器的 CPU 使用率。

整个过程是：
1. FastAPI 或 Exporter 暴露 HTTP 指标接口，例如 `/metrics`。
2. Prometheus 定期访问该接口。
3. Prometheus 获取指标数据。
4. Prometheus 将指标按照时间戳存储。
5. Grafana 从 Prometheus 查询数据并展示。
### 什么是 Pull 模型？
Prometheus 默认使用 Pull 模型，也就是 Prometheus 主动抓取数据，而不是由被监控服务主动推送数据。

例如：

```
Prometheus
    |
    | HTTP GET /metrics
    v
Node Exporter
    |
    v
返回 CPU、内存、磁盘指标
```

这意味着 Prometheus 必须能够通过网络访问被监控目标。

这也是后续配置 Docker Compose 网络时需要重点关注的地方。

### 什么是 Metric？
Metric 就是监控指标。
例如：
```
process_cpu_seconds_total 125.5
```
表示进程累计消耗的 CPU 时间。

```
process_resident_memory_bytes 104857600
```
表示进程当前使用的常驻内存大小。

Prometheus 中常见的指标类型有：

|类型|用途|示例|
|---|---|---|
|Counter|只增不减的累计值|HTTP 请求总数|
|Gauge|可以上升或下降的瞬时值|当前内存使用量|
|Histogram|统计数值分布|HTTP 请求延迟分布|
|Summary|统计分位数和观测值|请求延迟分位数|

### 什么是 Label？
Prometheus 使用 Label 区分同一个指标的不同维度。

例如：
```promql
http_requests_total{method="GET",status="200"}
```

这里的 `method` 和 `status` 就是 Label。

以后我们可以通过 Label 区分不同 HTTP 方法、状态码和 API 路由。

需要注意，不能随意使用用户 ID、请求 UUID 等高基数 Label，否则会导致时间序列数量急剧增长。

# 部署Prometheus
暂时不引入 Node Exporter、cAdvisor 和 Grafana，先把 Prometheus 自身运行、数据持久化和指标抓取机制掌握好。

## 理解三个核心概念

1. Scrape Interval：采集间隔
	Prometheus 定期访问目标的 `/metrics` 接口。例如设置为 `15s`，意味着每 15 秒采集一次。

2. Scrape Target：采集目标
	每一个需要采集指标的服务都是一个 Target。例如 Prometheus 自身的指标接口是 `prometheus:9090`。

3. TSDB：时序数据库
	Prometheus 内置时序数据库，用于保存采集到的指标及其时间戳。我们会使用 Docker Volume 持久化数据，避免容器重建后丢失监控历史。

`prometheus.yml` 配置文件
```yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - localhost:9090
```

|配置|含义|
|---|---|
|scrape_interval|每 15 秒采集一次指标|
|evaluation_interval|每 15 秒执行一次告警规则计算|
|scrape_configs|定义需要采集的目标|
|job_name|当前采集任务的名称|
|static_configs|使用静态方式配置目标|
|targets|需要访问的指标地址|
这里的 `localhost:9090` 表示 Prometheus 容器自身的 9090 端口，而不是宿主机的 localhost

`compose`
```yml
  prometheus:
    image: prom/prometheus:v3.13.5
    ports:
      # Prometheus Web UI
      - "127.0.0.1:9090:9090" 
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
```


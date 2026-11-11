PromQL 是 Prometheus 的查询语言。

例如：
```
container_memory_working_set_bytes
```
表示查询所有容器的工作集内存。

增加 Label 过滤条件：
```
container_memory_working_set_bytes{
  name="platform-dev-fastapi-1"
}
```


PromQL 常见 Label 匹配方式：

|运算符|含义|
|---|---|
|`=`|精确匹配|
|`!=`|不等于|
|`=~`|正则匹配|
|`!~`|正则不匹配|
## 理解 Counter 与 Gauge

这是 PromQL 中非常重要的基础。

### **Gauge**：瞬时值。

例如：
```
container_memory_working_set_bytes
```
内存使用量可以增加，也可以减少，所以它属于 Gauge。

查询时通常直接使用，不需要 `rate()`。

### **Counter**：累计值。

例如：
```
container_cpu_usage_seconds_total
```

CPU 使用时间只会不断累计，容器重启后可能归零。
如果直接查看它，只能看到累计消耗时间，并不能直观反映当前 CPU 使用率。

因此需要使用 
#### `rate()`

```
rate(container_cpu_usage_seconds_total[5m])
```
它表示过去 5 分钟内，CPU 累计使用时间每秒增加多少秒。

例如返回：
```
0.25
```
表示该容器平均每秒消耗了 0.25 秒 CPU 时间，也就是使用了一个逻辑 CPU 核心的 25%。


一些指标计算
[[02-部署node-exporter#计算宿主机 CPU 使用率]]
[[03-部署cAdvisor容器资源监控#计算容器 CPU 使用率]]







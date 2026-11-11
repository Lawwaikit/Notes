cAdvisor 通过 Linux 内核提供的 cgroups 等机制获取容器资源统计数据，并将其暴露为 Prometheus 格式的指标。

主要指标如下：

| 指标                                       | 含义            |
| ---------------------------------------- | ------------- |
| `container_cpu_usage_seconds_total`      | 容器累计 CPU 使用时间 |
| `container_memory_usage_bytes`           | 容器当前内存使用量     |
| `container_memory_working_set_bytes`     | 容器工作集内存       |
| `container_network_receive_bytes_total`  | 容器网络接收字节数     |
| `container_network_transmit_bytes_total` | 容器网络发送字节数     |
| `container_fs_reads_bytes_total`         | 容器文件系统读取字节数   |
| `container_fs_writes_bytes_total`        | 容器文件系统写入字节数   |

部署配置：`compose`
```yml
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.55.1
    container_name: platform-dev-cadvisor
    restart: unless-stopped

    privileged: true

    devices:
      - /dev/kmsg:/dev/kmsg

    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro

    ports:
      - "127.0.0.1:8080:8080"
```


# 常见指标计算
## 计算容器 CPU 使用率

### 1. 查询容器 CPU 使用速率
```
rate(container_cpu_usage_seconds_total{
  image!=""
}[5m])
```
它返回每个容器使用 CPU 的速率，单位是 CPU 核心数。

### 2. 转换成百分比

```
100 * rate(container_cpu_usage_seconds_total{
  image!=""
}[5m])
```

例如返回 `35`，表示该容器使用了一个逻辑 CPU 核心的 35%。

这里有一个容易混淆的地方：这个百分比是以一个逻辑 CPU 核心为 100% 的口径。

如果容器使用了 2 个完整 CPU 核心，则结果可能达到 200%。

### 3. 按容器聚合

```
100 *
sum by (name) (
  rate(container_cpu_usage_seconds_total{
    image!=""
  }[5m])
)
```

`sum by (name)` 将同一个容器不同 CPU 核心的指标合并。

如果想重点观察 FastAPI：

```
100 *
sum by (name) (
  rate(container_cpu_usage_seconds_total{
    image!="",
    name=~".*fastapi.*"
  }[5m])
)
```

## 计算容器内存使用量

内存属于 Gauge，可以直接查询。
### 1. 查询容器工作集内存
```
container_memory_working_set_bytes{
  image!=""
}
```
单位是 Byte。

### 2. 转换成 MB
```
container_memory_working_set_bytes{
  image!=""
} / 1024 / 1024
```

### 3. 查询 FastAPI 内存
```
container_memory_working_set_bytes{
  image!="",
  name=~".*fastapi.*"
} / 1024 / 1024
```

这里我们使用 `container_memory_working_set_bytes`，而不是简单使用 `container_memory_usage_bytes`，因为工作集更适合观察实际活跃内存使用情况。

## 计算容器网络吞吐量

cAdvisor 提供网络累计字节数：
```
container_network_receive_bytes_total
```
它是 Counter。

### 1. 计算网络接收速率
```
rate(container_network_receive_bytes_total{
  image!=""
}[5m])
```
单位是 Bytes/s。

### 2. 转换成 MB/s
```
rate(container_network_receive_bytes_total{
  image!=""
}[5m]) / 1024 / 1024
```

发送速率：
```
rate(container_network_transmit_bytes_total{
  image!=""
}[5m]) / 1024 / 1024
```
注意，容器可能有多个网络接口，因此需要根据实际 Label 判断是否需要聚合。

## 计算容器磁盘 I/O 速率

读取速率：
```
rate(container_fs_reads_bytes_total{
  image!=""
}[5m])
```

写入速率：
```
rate(container_fs_writes_bytes_total{
  image!=""
}[5m])
```

单位都是 Bytes/s。
需要注意，cAdvisor 的文件系统指标并不等于容器所有存储操作的完整统计，尤其不能简单地将它们当作 MySQL Volume 的全部物理磁盘 I/O。

后续我们会专门区分容器写入层、Volume 和宿主机文件系统。
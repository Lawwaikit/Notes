Grafana 是一个可观测性可视化平台，能够连接 Prometheus 等数据源，并将监控指标展示为图表、仪表盘和告警。
Grafana 不负责采集 Docker 指标，也不直接读取容器的 cgroups。它通过 Prometheus API 查询已经采集好的数据。
### 增加 Compose 服务
Compose 配置中增加：
```yml
services:
  grafana:
    image: grafana/grafana:13.2
    restart: unless-stopped

    ports:
      - "127.0.0.1:3000:3000"

    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  grafana_data:
```

部署之后 访问http://localhost:3000/



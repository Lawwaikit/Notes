# Grafana Provisioning：自动化配置管理
通过 Grafana Web 页面手动添加 Prometheus 数据源、创建 Dashboard，再将 Dashboard 导出为 JSON 纳入 Git 管理。这解决了配置文件的版本管理问题，但还没有解决自动部署问题：如果删除并重建 Grafana 容器，仍然需要手动配置数据源和导入 Dashboard。

### 一、理解 Provisioning 的目录结构

Grafana 容器默认从 `/etc/grafana/provisioning/` 读取 Provisioning 配置。我们在项目中建立对应的配置目录：

```r
monitoring/
└── grafana/
    ├── provisioning/
    │   ├── datasources/
    │   │   └── prometheus.yml
    │   └── dashboards/
    │       └── dashboards.yml
    └── dashboards/
        └── docker-test-platform-dev.json
```

这里有两个容易混淆的目录：`provisioning/dashboards/` 存放 Dashboard 加载规则，而 `dashboards/` 存放真正的 Dashboard JSON 文件。

### 二、配置 Prometheus 数据源

在项目根目录创建文件：

`monitoring/grafana/provisioning/datasources/prometheus.yml`

内容如下：

```yml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

关键配置说明：
- `apiVersion: 1`：Provisioning 配置文件格式版本。
- `name`：Grafana 中显示的数据源名称。
- `uid`：数据源的稳定标识，Dashboard 可以通过它引用数据源。
- `type`：数据源类型，这里使用 Prometheus。
- `access: proxy`：由 Grafana 后端代理请求 Prometheus。
- `url`：使用 Docker Compose 服务名访问 Prometheus，而不是 `localhost`。
- `isDefault: true`：将 Prometheus 设置为默认数据源。
- `editable: false`：禁止通过 Web 页面修改这个由文件管理的数据源。

特别注意：`editable: false` 意味着数据源配置应当通过修改 YAML 文件管理，而不是通过 Grafana 页面修改。

### 三、配置 Dashboard 自动加载

创建文件：
`monitoring/grafana/provisioning/dashboards/dashboards.yml`

内容如下：
```yml
apiVersion: 1

providers:
  - name: DockerTestPlatform
    orgId: 1
    folder: DockerTestPlatform
    type: file
    disableDeletion: true
    editable: false
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

关键配置说明：
`providers` 定义 Dashboard 加载规则
`type: file` 表示从本地文件系统读取 JSON；
`folder` 指定 Dashboard 在 Grafana 页面中的文件夹；
`path` 指向容器内部的 JSON 文件目录；
`disableDeletion: true` 表示文件被移除时不自动删除已经加载的 Dashboard；
`editable: false` 表示 Dashboard 由配置文件管理，不允许在页面中直接编辑。

### 四、修改 Compose 挂载配置

接下来需要让 Grafana 容器能够读取刚才创建的文件。

在 `compose.dev.yml` 的 `grafana` 服务中增加以下挂载配置：

```yml
services:
  grafana:
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
```

这里保留原来的 `grafana_data` named volume，不要删除或替换。两个新增的 bind mount 分别用于提供 Provisioning 配置和 Dashboard JSON 文件，`:ro` 表示容器只能读取这些文件。

### 五、启动并验收

先检查 Compose 配置是否有效：

```bash
docker compose \
  -p platform-dev \
  -f compose.yml \
  -f compose.dev.yml \
  config -q
```

没有报错，说明 Compose 配置解析成功。

然后只重建 Grafana 服务：

```bash
docker compose \
  -p platform-dev \
  -f compose.yml \
  -f compose.dev.yml \
  up -d --no-deps grafana
```

`--no-deps` 表示不启动或重建 Grafana 的依赖服务，因此不会主动重建 Prometheus、MySQL、Redis 等服务。

查看 Grafana 日志：

```bash
docker compose \
  -p platform-dev \
  -f compose.yml \
  -f compose.dev.yml \
  logs --tail=100 grafana
```

确认没有 Provisioning 相关错误，然后打开 Grafana 页面检查
1. 进入 Connections → Data sources，确认 Prometheus 数据源存在，URL 为 `http://prometheus:9090`。
2. 进入 Dashboards，确认 `DockerTestPlatform Dev Monitoring` 已自动加载。
3. 打开 Dashboard，确认原有 Panel 能正常查询 Prometheus 数据。

如果 Dashboard 原本已经存在，Provisioning 可能会接管同一个 Dashboard UID。若出现 UID 冲突或重复 Dashboard，先不要删除数据卷，检查导出 JSON 中的 `uid` 和现有 Dashboard 的 UID 是否一致。


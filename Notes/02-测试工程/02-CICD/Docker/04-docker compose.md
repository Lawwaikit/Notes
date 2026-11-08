# Docker Compose

Compose 是**定义和运行多容器应用**的工具：用一份 YAML 描述所有服务（镜像、端口、卷、依赖、网络），一条命令批量创建、启动、销毁。

`docker run` 的一长串参数，正好对应 YAML 里的一个 service：

```text
docker run -d --name web -p 5000:5000 -v ./app:/app --env-file .env.dev web:dev
        ↓ 等价于
services:
  web:
    image: web:dev
    container_name: web
    ports: ["5000:5000"]
    volumes: ["./app:/app"]
    env_file: [.env.dev]
```

## 1. 命令速查

```bash
# 构建并启动（开发最常用）
docker compose up -d --build

# 查看所有服务状态 / 日志
docker compose ps
docker compose logs -f flask

# 扩容 Flask 到 3 个副本（注意 §9 的坑）
docker compose up -d --scale flask=3
```

| 命令 | 作用 | 备注 |
| --- | --- | --- |
| `up -d` | 创建并后台启动 | 容器已存在且配置未变则复用 |
| `up -d --build` | 先重新构建镜像再启动 | 改了 Dockerfile / 依赖后必备 |
| `up -d --force-recreate` | 强制重建容器 | 改了 env、挂载但没重建时用 |
| `ps` | 服务与容器状态 | 看 STATUS 列判断 healthy |
| `logs -f flask` | 跟踪某服务日志 | 不写服务名 = 全部服务 |
| `exec flask bash` | 进运行中的容器 | 比 `docker exec` 好在自动定位项目 |
| `restart flask` | 重启容器 | 不重建、不重新读镜像 |
| `stop` / `start` | 停 / 启已创建的容器 | 不删容器 |
| `down` | 停并删除容器 + 默认网络 | **不删命名卷** |
| `down -v` | 同时删除命名卷 | 数据全没，重置环境用 |
| `config` | 校验语法 + 打印**合并后**的最终配置 | 排查变量/覆盖是否生效 |
| `config -q` | 只校验不输出 | CI 里用 |
| `ls` | 列出本机所有 compose 项目 | 按项目名分组 |
| `build` / `pull` / `top` / `events` | 构建 / 拉镜像 / 看进程 / 看事件 | 排障常用 |

两个易混点：

- `up` vs `start`：`up` 会**对齐配置**（重建有变化的容器、补建网络卷），`start` 只是把已存在的容器拉起来。改完 compose 文件用 `start` 是不生效的。
- `down` 删不删数据：命名卷默认保留，只有 `-v` 才删；**bind mount 的宿主目录永远不受影响**。

## 2. 项目名与命名前缀

`docker compose config` 输出的第一行 `name: dockertestplatform` 就是项目名，来源优先级：

1. `docker compose -p 名字`
2. 环境变量 `COMPOSE_PROJECT_NAME`
3. YAML 顶层 `name:` 字段
4. 默认 = compose 文件所在的**目录名**

作用：**服务、网络、卷都会被加上 `<项目名>_` 前缀**（如 `dockertestplatform_mysql_data_dev`）。所以同一个 compose 文件在不同目录跑两次，会得到两套互不干扰的环境；`docker compose ls` 也按这个项目名来分组。

例外是 `container_name`：写死后**不加前缀**，于是

- 同一台宿主机上不能重名（两个项目/两份 compose 会打架）
- 不能用 `--scale`（副本容器名必须唯一）

## 3. 文件结构

```yaml
name: dockertestplatform    # 可选，项目名
services:                   # 必填
  flask: ...
networks:                   # 可选，自定义网络
  backend: ...
volumes:                    # 可选，命名卷
  mysql_data_dev: ...
```

- 文件名推荐 `compose.yaml`（`docker-compose.yml` 也识别，v2 优先前者）。
- `version:` 字段**已废弃**，写了只会得到一条 warning，别再照抄旧教程。
- **只用空格缩进，禁止 Tab**，层级错一格就报 `yaml: line X: did not find expected key`。

## 4. services 常用字段

| 字段                              | 作用                     | 写法 / 注意                                                                |
| ------------------------------- | ---------------------- | ---------------------------------------------------------------------- |
| `image`                         | 直接用现成镜像                | `mysql:8.0`；与 `build` 同写时作为构建产物名                                       |
| `build`                         | 从 Dockerfile 构建        | `build: ./flask-app` 或长语法 context/dockerfile/args/target               |
| `ports`                         | 发布到宿主机                 | `"5000:5000"`（宿主:容器）；只写 `"5000"` = 随机宿主端口；`"127.0.0.1:5000:5000"` 只绑本机 |
| `expose`                        | 仅容器间可见                 | 不发布到宿主机，纯声明                                                            |
| `environment`                   | 注入环境变量                 | 列表或键值对；同名 key **优先于** `env_file`                                       |
| `env_file`                      | 从文件读变量                 | `.env.dev`；只注入容器，不参与插值                                                 |
| `volumes`                       | 挂载                     | `./src:/app` bind；`data:/var/lib/mysql` named；`:ro` 只读                 |
| `depends_on`                    | 启动依赖                   | 短语法只排顺序；长语法配 `condition` 才等就绪                                          |
| `healthcheck`                   | 健康探测                   | test/interval/timeout/retries/start_period                             |
| `restart`                       | 重启策略                   | no / always / unless-stopped / on-failure                              |
| `networks`                      | 接入自定义网络                | 不写 = 进默认网络 `<项目名>_default`                                             |
| `command` / `entrypoint`        | 覆盖镜像的 CMD / ENTRYPOINT | 只改这次运行，不改镜像                                                            |
| `logging`                       | 日志驱动与轮转                | `json-file` + max-size/max-file，防日志写爆磁盘                                |
| `deploy` / `mem_limit` / `cpus` | 资源限制                   | `deploy.resources.limits` 在 `compose up` 也生效；老写法 `mem_limit`           |
| `user` / `working_dir`          | 运行身份与目录                | 生产建议 `user: "1000:1000"`                                               |

`restart` 策略怎么选：

- `no`（默认）：不自动重启。
- `on-failure`：非 0 退出才重启。
- `always`：崩了重启，**手动 stop 的容器在 docker 重启后也会被拉起**。
- `unless-stopped`：手动 stop 后就保持停止，调试期最顺手。

### build 长语法

```yaml
flask:
  build:
    context: ./flask-app      # 构建上下文（相对 compose 文件所在目录）
    dockerfile: Dockerfile    # 换 Dockerfile 文件名
    args:                     # 对应 Dockerfile 的 ARG
      PIP_INDEX_URL: https://pypi.tuna.tsinghua.edu.cn/simple
    target: builder           # 多阶段构建里的某一阶段
```

`build` 就是"跑一次 `docker build`"：`context` 对应上下文，`dockerfile` 对应 `-f`，`target` 对应 `--target`。

### 三种挂载

| 写法 | 类型 | 生命周期 | 典型用途 |
| --- | --- | --- | --- |
| `./logs:/var/log/app` | bind mount | 跟宿主机目录走 | 日志、配置文件、开发期源码 |
| `mysql_data_dev:/var/lib/mysql` | named volume | Docker 管理，`down -v` 才删 | 数据库数据（跨容器重建保留） |
| `./init.sql:/docker-entrypoint-initdb.d/init.sql:ro` | 单文件 bind | 跟宿主机文件走 | 初始化脚本、配置，建议只读 |

相对路径是**相对 compose 文件所在目录**，不是执行命令的当前目录（这点和 `docker run -v` 必须写绝对路径不同）。

### depends_on + healthcheck

```yaml
depends_on:
  mysql:
    condition: service_healthy     # 等 mysql 健康检查通过
  redis:
    condition: service_started     # 只等它启动（等价短语法）
```

- 短语法 `depends_on: [mysql]` **只保证启动顺序**，不等服务就绪。应用起得比库快就会连不上——这是最常见的"启动即失败"原因。
- `condition` 可选值：`service_started` / `service_healthy` / `service_completed_successfully`（一次性初始化任务跑完）。
- `depends_on` 只影响**启动**，不影响重启；运行时挂掉不会连锁重启。
- 即使配了 healthcheck，应用侧仍建议自带重试（降低概率 ≠ 保证）。

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:5000/healthz || exit 1"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 20s      # 启动期内的失败不计入 retries
```

- `test` 写 `CMD`（exec 形式，不过 shell）或 `CMD-SHELL`（走 shell，能写 `||`、`$VAR`）。
- 探测地址写 `127.0.0.1`（容器自己），不要写别的容器名。
- 状态在 `docker compose ps` 的 STATUS 列看：`healthy` / `unhealthy` / `starting`。

## 5. 网络与端口

- 不声明 `networks` 时，所有服务自动进默认网络，**服务名就是 DNS**：容器里直接连 `mysql:3306`、`redis:6379`。
- 别在容器里用 `localhost` 连其他容器——localhost 是容器自己。
- 按用途分组隔离：

```yaml
services:
  flask:
    networks: [frontend, backend]
  mysql:
    networks: [backend]
networks:
  frontend:
  backend:
```

- `ports` 和容器互访是两件独立的事：内部互访**不需要** `ports`，写 `expose` 就够；`ports` 只给宿主机/外部用。
- 接入已有网络（跨项目互通）：

```yaml
networks:
  shared:
    external: true
    name: dockertestplatform_default
```

- 容器访问宿主机上的服务：Mac/Windows 用 `host.docker.internal`，Linux 需要加 `extra_hosts: ["host.docker.internal:host-gateway"]`。

## 6. 环境变量管理与插值
Compose 中可以同时出现 `.env`、`env_file` 和 `environment` 三种语法

变量有`插值`和`容器环境变量`两种

### `.env` 文件：主要为 Compose 提供变量插值

例如你的配置中：
```yml
environment:
  APP_ENV: ${APP_ENV}
  APP_VERSION: ${APP_VERSION}
```

Compose 在创建容器之前，会先解析 `${APP_ENV}` 和 `${APP_VERSION}`。默认情况下，它会读取项目目录中的 `.env` 文件，也会使用运行 Compose 命令时 Shell 中已经存在的环境变量。

### `env_file`：负责向容器注入环境变量

你的配置中：
```yml
env_file:
  - ./fastapi/.env
```

`.env`
```ini
APP_ENV=development
APP_VERSION=1.0.0
```

这表示 Compose 创建 FastAPI 容器时，会读取 `fastapi/.env`，将文件中的变量都注入容器。

### `environment`：显式声明容器需要注入的环境变量

例如：
```yml
environment:
  APP_ENV: ${APP_ENV}
  TEST_ENV: 2
```

### 容器环境变量
容器里面能读到的环境变量，注入方式有 

| 优先级 | 配置来源                    | 说明            |
| --- | ----------------------- | ------------- |
| 1高  | `docker compose run -e` | 临时运行时指定       |
| 2   | `environment`           | Compose 中显式配置 |
| 3   | `env_file`              | 文件注入          |
| 4低  | Dockerfile 的 `ENV`      | 镜像默认环境变量      |

`env_file` 和 `environment` **(environment优先级高)

当 `environment` 里面变量值 配置了 `${}`，就是需要插值

### 变量插值
compose解析时需要获取的变量，来源有
1. 运行 `docker compose` 命令的 Shell 环境变量
2. `--env-file` ，注意这个是compose命令行参数，和yml文件的`env-file`服务配置不是同一个
3. `.env` ，docker compose文件所在目录的`.env`文件

#### `${VAR}` 的默认值和错误检查
Compose 支持几种常见插值表达式：

|写法|含义|
|---|---|
|`${VAR}`|使用变量值；未设置时通常给出警告并替换为空字符串|
|`${VAR:-default}`|变量未设置或为空字符串时，使用默认值|
|`${VAR-default}`|只有变量未设置时，才使用默认值|
|`${VAR:?error}`|变量未设置或为空字符串时，报错并停止|
|`${VAR?error}`|只有变量未设置时，才报错|

其中最容易混淆的是 `:-` 和 `-`。前者同时检查变量是否为空，后者只检查变量是否未设置。


## 7. 多环境：override 与多文件

- 默认自动合并 `compose.yaml` + `compose.override.yaml`（后者存在就生效，不用加参数）。
- 显式指定多个文件，**后面的覆盖前面的**：

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

合并规则：

- 标量（`image`、`command`、`restart`）→ 后者**覆盖**
- 列表/映射（`ports`、`volumes`、`environment`）→ **追加或按键合并**，不是替换（端口会两个都映射，env 同名 key 覆盖）
- Compose 2.24+ 可用 `!reset []` 清空、`!override` 强制替换

生产覆盖文件只写差异：

```yaml
# compose.prod.yaml
services:
  flask:
    build:
      context: ./flask-app
      target: prod
    volumes: !reset []          # 生产不挂源码
    environment:
      FLASK_ENV: production
    restart: always
```

密钥放 `.env.prod`，用 `--env-file .env.prod` 传入，**别把密钥提交进仓库**。

## 8. 完整示例（开发环境）

```yaml
# compose.yaml —— 开发环境
services:
  # Flask 应用
  flask:
    build:
      context: ./flask-app
      dockerfile: Dockerfile
    image: flask-app:dev            # 构建产物打 tag，方便复用
    container_name: flask-dev
    ports:
      - "5000:5000"
    env_file:
      - .env.dev
    environment:
      FLASK_ENV: development        # 同名 key 覆盖 env_file
      MYSQL_HOST: mysql             # 用服务名，不用 localhost
      REDIS_HOST: redis
    volumes:
      - ./flask-app:/app            # 代码热加载（仅开发）
      - ./logs/flask:/var/log/flask
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:5000/healthz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    networks: [backend]

  # MySQL 数据库
  mysql:
    image: mysql:8.0
    container_name: mysql-dev
    env_file:
      - .env.dev                    # 提供 MYSQL_ROOT_PASSWORD 等
    ports:
      - "3306:3306"
    volumes:
      - mysql_data_dev:/var/lib/mysql
      - ./mysql-init/dev-init.sql:/docker-entrypoint-initdb.d/dev-init.sql:ro   # 初始化 SQL
      - ./logs/mysql:/var/log/mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 30s             # 首次初始化数据目录较慢
    networks: [backend]

  # Redis 缓存
  redis:
    image: redis:7.2
    container_name: redis-dev
    ports:
      - "6379:6379"
    volumes:
      - redis_data_dev:/data
    restart: always
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [backend]

volumes:
  mysql_data_dev:
  redis_data_dev:

networks:
  backend:
    driver: bridge
```

配套 `.env.dev`（只放容器要用的变量）：

```dotenv
MYSQL_ROOT_PASSWORD=dev123456
MYSQL_DATABASE=dockertest
MYSQL_USER=devuser
MYSQL_PASSWORD=devpass
```

启动与验证：`docker compose up -d --build` → `docker compose ps` 看到 `healthy` → `docker compose logs -f flask` 跟日志。

## 9. 常见坑

- **`depends_on` 不等于"等就绪"**：短语法只排启动顺序，应用连库失败先查这里；要等就绪必须 `healthcheck` + `condition: service_healthy`。
- **端口冲突**：宿主上已经装了 MySQL，`"3306:3306"` 直接起不来。只给容器间通信用 `expose`，别写 `ports`。
- **`--scale` 与 `container_name`/固定端口冲突**：副本数大于 1 时容器名不能重复、宿主端口也不能重复 → 去掉 `container_name`，`ports` 写 `"5000"` 让 Docker 随机分配。
- **改代码不生效**：开发期靠 volume 挂载（改代码即生效）；改了 `requirements.txt` 或 Dockerfile 必须 `up -d --build`。
- **`down` 之后数据还在**：命名卷默认保留，想彻底重置用 `down -v`；bind mount 的宿主目录不受影响。
- **容器里连不上另一个容器的 `localhost`**：用服务名，如 `mysql:3306`。
- **`$` 被吞**：写 `$$`。
- **Tab 缩进**：YAML 只认空格，报错 `did not find expected key`。
- **`environment` 悄悄覆盖了 `env_file`**：同名 key 以 `environment` 为准，用 `docker compose config` 看最终值。
- **bind mount 把镜像内容盖住了**：`./flask-app:/app` 会覆盖镜像里 `/app` 的全部内容（依赖装在 site-packages，一般不受影响，但镜像内预置的文件会"消失"）。
- **`:ro` 漏写**：初始化 SQL、配置文件被容器改写或删除，排查半天找不到原因。


---
---

# 进阶
# 多文件配置
Compose 提供了配置合并机制。
假设你的项目需要支持开发环境和生产环境。

开发环境可能需要 `LOG_LEVEL=DEBUG`、较高的 CPU 和内存限制，而生产环境则需要 `LOG_LEVEL=INFO`、严格的资源限制。

如果维护两份完整的 Compose 文件，会产生大量重复配置。以后修改 MySQL 或 Redis 的配置时，还需要同步修改两份文件。
```bash
# 开发环境
docker compose \
  -f compose.yaml \
  -f compose.dev.yaml \
  up -d

# 生产环境
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d
```

Compose 合并配置时，主要涉及以下规则：

| 配置类型                     | 合并行为              |
| ------------------------ | ----------------- |
| 普通标量，例如 `image`、`cpus`   | 后面的值覆盖前面的值        |
| Mapping，例如 `environment` | 按键合并，相同键以后面的值为准   |
| Sequence，例如 `ports`      | 通常追加合并，但部分字段有特殊规则 |
| `volumes`                | 按容器挂载目标等唯一性规则合并   |
## 普通标量覆盖
**cpus, mem_limit**
`compose.yml`
```yml
services:
  fastapi:
    cpus: 1.0
    mem_limit: 512m
	environment:
      LOG_LEVEL: INFO
```

`compose.dev.yml`
```yml
services:
  fastapi:
	cpus: 2.0  # 标量覆盖
    mem_limit: 1g  # 标量覆盖
    environment:  # mapping覆盖
      LOG_LEVEL: DEBUG
```

## Sequence类型覆盖
`ports`、`volumes` 等列表不能简单理解为后面的配置完全覆盖前面的配置。
常规配置会增加列表项
### 使用 !reset 和 !override

Compose 提供了两个特殊标签，用于控制合并行为。

#### `!reset`：清空基础配置中的某个字段。

例如：
```yml
services:
  nginx:
    ports: !reset []
```
合并之后，Nginx 不再保留基础文件中的端口映射。

#### `!override`：完全替换基础配置中的字段。

例如：
```yml
services:
  nginx:
    ports: !override
      - "8080:80"
```
最终只保留 `8080:80`，不再保留原来的 `80:80`。

# YAML 锚点与复用
例如，FastAPI、MySQL、Redis、Nginx 都重复声明了相同的日志配置：
```yml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "5"
```
如果未来需要把日志文件数量从 5 改成 10，就需要修改四处。
YAML 支持锚点（Anchor）和别名（Alias），可以复用配置。

例如：
```yml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "5"

services:
  fastapi:
    logging: *default-logging

  mysql:
    logging: *default-logging

  redis:
	logging:
      <<: *default-logging
      options:
	    max-file: "3" # 浅合并，options会被替换,max-size失效
```
这样，四个服务可以共用一份日志配置。

锚点机制主要涉及三个符号：

1. **定义锚点（`&`）**：在某个配置块前加上 `&名称`，为其创建一个“锚点”。
2. **引用别名（`*`）**：在需要复用该配置的地方，使用 `*名称` 即可完整复制锚点处的值。
3. **合并覆盖（`<<`）**：如果你希望在复用锚点配置的基础上，再修改或增加个别属性，可以使用 YAML 的合并键 `<<`。

**`x-` 使用**：锚点定义在Compose 文件的顶层（`services`、`volumes` 等之外）时，必须用 `x-` 开头，不会被 Compose 误认为是无效配置。当锚点直接定义在某个合法配置项内部时，则不需要用 `x-`
	原因是 Docker Compose 只认识 `services`、`volumes`、`networks` 等固定的顶级键。如果你写一个没有 `x-` 前缀的 `logging:` 在顶层，Compose 会把它当成一个无法识别的非法字段而报错。而 `x-` 前缀是 Compose 规范中**唯一允许被静默忽略**的例外，Compose 看到以 `x-` 开
	的字段会直接跳过，这样就刚好可以用来存放供锚点引用的“游离”配置，不影响正常解析。

如果在compose内本来就合法的配置上，定义锚点，就不需要x-前缀
```yml
services:
  service-a:
    image: my-app
    logging: &my-logging   # 这里不需要 x-，因为 logging 是合法的服务配置
      driver: json-file
      options:
        max-size: "10m"
  service-b:
    image: my-app
    logging: *my-logging   # 直接引用
```


**`<<`合并的限制**： 
1. 合并键**仅适用于映射（键值对）**，不能用于列表（序列）。例如，你不能用它来合并两个 `volumes` 列表，如果直接写 `volumes` 键会覆盖整个列表，而不是将列表项合并。
2. 合并只做**顶层键的替换**，不会递归合并嵌套的映射（mapping）。
```yml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "5"

services:
  redis:
	logging:
      <<: *default-logging
      options:
	    max-file: "3" # 浅合并，options会被替换,max-size失效
```
使用docker compose config查看redis的配置
```yml
  nginx:
    logging:
      driver: json-file
      options:
        max-file: "3"
```


# depends_on、healthcheck 与服务就绪判断
这一节非常重要。它解决的是多容器项目中一个常见问题：容器已经启动，不代表里面的应用已经准备好提供服务。

结合你的 DockerTestPlatform，目前有四个服务：
- **FastAPI**：需要连接 MySQL 和 Redis。
- **MySQL**：需要完成初始化，才能正常接受数据库请求。
- **Redis**：需要启动并能够响应命令。
- **Nginx**：需要将请求转发给 FastAPI。

## depends_on：控制服务启动依赖
短格式
```yml
nginx:
  depends_on:
    - fastapi
```

长格式
```yml
fastapi:
  depends_on:
    mysql:
      condition: service_healthy
```

`condition` 主要有三个条件：

|condition|含义|
|---|---|
|`service_started`|依赖服务已经启动|
|`service_healthy`|依赖服务健康检查通过|
|`service_completed_successfully`|依赖服务正常执行完成，退出码为 0|

## healthcheck：定义服务健康状态

`healthcheck` 是 Docker 提供的健康检查机制。Docker 会按照指定的周期，在容器内部执行检查命令，根据命令的退出码判断容器是否健康。

比如
```yml
healthcheck:
  test:
    - CMD-SHELL
    - 'mysqladmin ping -h localhost -uroot -p"$$(cat /run/secrets/mysql_root_password)"'
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s
```

各参数含义如下：

|参数|当前值|作用|
|---|---|---|
|`test`|mysqladmin ping|实际执行的检查命令|
|`interval`|10s|两次检查之间的间隔|
|`timeout`|5s|单次检查最长等待时间|
|`retries`|5|连续失败多少次后判定为 Unhealthy|
|`start_period`|30s|容器启动后的健康检查宽限期|
这里有一个容易误解的地方：`start_period` 并不是延迟 30 秒才开始第一次检查，而是给服务初始化预留宽限期。

在宽限期内，如果检查失败，通常不会计入连续失败次数；如果检查成功，则认为服务已经启动，后续失败会正常计数。

### 3.2 healthcheck 的执行结果

健康检查主要有三种状态：


| 状态            | 含义                   | 说明                     |
| ------------- | -------------------- | ---------------------- |
| **starting**  | 容器刚启动，尚未获得足够的健康检查结果  | 健康检查尚未积累到足够的结果，处于初始阶段  |
| **healthy**   | 健康检查已经成功             | 检查通过，容器状态正常            |
| **unhealthy** | 连续检查失败次数达到 `retries` | 连续失败达到重试次数上限，容器被标记为不健康 |

需要特别注意：Unhealthy 不等于容器退出，也不代表 Docker 会自动重启容器。

## Liveness, Readiness
这两个概念经常出现在 Kubernetes 中，但同样适用于 Docker 容器应用的设计。
### Liveness：应用是否还活着

Liveness 关注的是应用自身是否仍然正常运行。

例如 FastAPI：

```
GET /health/live
```

只要 Python 应用的事件循环正常工作，HTTP 服务能够响应，就返回 200。
如果 Liveness 失败，说明应用可能已经陷入死锁、无法响应或发生严重故障。
需要注意，Docker 原生 healthcheck 并没有独立的 Liveness 类型。这里是我们通过健康检查端点的设计来实现这种语义。
### Readiness：应用是否准备好提供业务

Readiness 关注的是应用是否具备处理业务请求的条件。
例如：
```
GET /health/ready
```

这个接口可以检查：
- FastAPI 是否正常响应。
- MySQL 是否能够执行简单查询。
- Redis 是否能够正常执行 PING。
如果 MySQL 不可用，Readiness 就返回失败。

![[Pasted image 20261106171357.png]]
 
重点：Docker Compose 不能自动完成故障恢复
- `healthcheck` 负责检查健康状态。
- `restart` 负责处理容器主进程退出后的重启行为。
- `depends_on` 负责服务启动时的依赖控制。
当容器服务运行中 发生故障时，只能知道unhealth，不能做故障恢复
需要应用层做识别，比如对于`fastapi`来说
- 用连接池的失效连接检测
- 对于数据库连接初始化、Redis 连接初始化等操作，可以使用带退避机制的重试策略

# nginx负载均衡和服务发现

`nginx.conf` 配置如下：
```bash
resolver 127.0.0.11 valid=10s ipv6=off;

upstream fastapi_backend {
    zone fastapi_backend 64k;
    server fastapi:8000 resolve;
}   

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://fastapi_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
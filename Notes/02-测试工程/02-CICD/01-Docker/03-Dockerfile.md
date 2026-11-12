# Dockerfile

[参考资料：# Docker Dockerfile](https://www.runoob.com/docker/docker-dockerfile.html)

Dockerfile 是**镜像构建脚本**：Docker 从上到下逐行执行，每条指令生成一个**镜像层（Layer）**。

```bash
docker build -t app:1.0 .                            # . 是构建上下文
docker build -t app:1.0 -f docker/Dockerfile.prod .  # 指定 Dockerfile 文件，上下文仍是最后的 .
docker build --no-cache -t app:1.0 .                 # 禁用缓存，排查"改了没生效"
docker build --build-arg VERSION=2.0 -t app:2.0 .    # 传 ARG
```

`-t 名称:标签`：不写标签默认 `latest`，建议始终显式带版本（见第 9 节）。

## 1. 构建上下文（build context）

Docker 是 C/S 架构：本机是 C（client），docker 引擎是 S（server）。**构建实际在引擎侧完成，引擎拿不到本机文件**，所以 `docker build` 会先把指定目录打包上传，这个目录就是"构建上下文"。

- `.` = 用当前目录作上下文。**只有上下文内的文件才能被 `COPY`/`ADD` 引用**，`../xxx` 这类越界路径直接报错。
- `-f` 只改变 Dockerfile 的位置，**不改变上下文**；上下文永远是最后那个参数。
- 上下文越小，传输和构建越快 → 用 `.dockerignore` 排除无关文件。

```gitignore
# .dockerignore
.git
.venv
__pycache__
*.log
.env
node_modules
```

- 验证是否生效：构建日志里 `load build context → transferring context: 2.1MB`（旧构建器是 `Sending build context to Docker daemon 2.1MB`）。体积异常大，说明 `.dockerignore` 漏了东西。

## 2. 指令速查

| 指令          | 用法                                             | 说明                | 注意点                            |
| ----------- | ---------------------------------------------- | ----------------- | ------------------------------ |
| FROM        | `FROM python:3.12-slim`                        | 指定基础镜像            | 必须是第一条指令（只允许注释、ARG 在它前面）      |
| WORKDIR     | `WORKDIR /app`                                 | 设置容器工作目录，类似 `cd`  | 后续 COPY、RUN、CMD 默认在此目录执行，不存在自动创建 |
| COPY        | `COPY . .`                                     | 拷贝宿主机文件到容器        | 源是构建上下文路径；每次改动都会生成新层           |
| ADD         | `ADD file.tar.gz /app`                         | 功能类似 COPY，但可自动解压  | 用途少，除非真要解压 tar，否则用 COPY        |
| RUN         | `RUN apt-get update && apt-get install -y vim` | 构建镜像时执行命令         | 每条 RUN 生成新层，按"变更频率"合并           |
| ENV         | `ENV MESSAGE="Hello"`                          | 定义环境变量            | 构建期和运行时都可见，会持久化进镜像             |
| ARG         | `ARG PY_VERSION=3.12`                          | 构建期参数             | 只在构建期可见，FROM 之后需重新声明           |
| EXPOSE      | `EXPOSE 5000`                                  | 声明端口              | 仅文档用途，不自动映射端口                  |
| ENTRYPOINT  | `ENTRYPOINT ["python"]`                        | 固定容器入口            | 配 CMD 传默认参数，run 参数会追加           |
| CMD         | `CMD ["python", "app.py"]`                     | 容器启动默认命令          | 最后一条 CMD 生效，可被 docker run 参数整体覆盖 |
| USER        | `USER app`                                     | 指定运行用户            | 生产别用 root                      |
| HEALTHCHECK | `HEALTHCHECK CMD curl -f http://localhost/`    | 健康探测              | compose/编排靠它判断是否就绪             |
| LABEL       | `LABEL version="1.0"`                          | 元数据               | 可用 `docker inspect` 查询         |

推荐书写顺序：

```dockerfile
FROM python:3.12-slim        # 1. 基础镜像
WORKDIR /app                 # 2. 固定工作目录
COPY requirements.txt .      # 3. 先拷"变化少"的依赖清单
RUN pip install -r ...       # 4. 装依赖（缓存友好）
COPY . .                     # 5. 最后拷源码（变化最频繁）
ENV PYTHONUNBUFFERED=1       # 6. 运行配置
EXPOSE 5000                  # 7. 声明端口（文档性质）
USER app                     # 8. 降权运行
ENTRYPOINT ["python"]        # 9. 入口
CMD ["app.py"]               #    默认参数
```

这套顺序不是语法要求，而是**为了缓存命中**（见第 4 节）。

## 3. 两种路径：上下文路径 vs 容器内路径

Dockerfile 里的路径分两套参照系，混淆是最常见的坑：

| 位置                     | 参照系        | 说明                                         |
| ---------------------- | ---------- | ------------------------------------------ |
| `COPY` 的**源**          | 构建上下文（宿主机） | `COPY app/requirements.txt .` 里的 `app/` 是上下文里的目录 |
| `COPY` 的**目标**、`RUN` 路径 | 容器内路径      | 相对路径都基于 `WORKDIR`                           |

### WORKDIR

- 设置的是**容器内**工作目录，后续 `RUN`/`COPY`/`CMD`/`ENTRYPOINT` 都以此为基准。
- 目录不存在会**自动创建**，比 `RUN cd /app` 可靠（`cd` 只对当前那条 RUN 有效）。
- 相对路径会拼接：`WORKDIR /app` + `WORKDIR src` → `/app/src`。
- 一个镜像用一个 `WORKDIR` 就够，反复横跳只会让后面的指令难读。

### 业务代码的 `app/` 和 `WORKDIR /app` 是同一个吗？

不是，纯属重名：

- `WORKDIR /app` 是**容器内的绝对路径**，容器里会真的建一个 `/app` 目录。
- 宿主机上的 `app/` 是**构建上下文里的目录**，只作为 `COPY` 的来源。
- 改成 `WORKDIR /code` + `COPY app/ /code`，效果完全一样。

### `COPY app/requirements.txt .` 怎么读

`app/requirements.txt` 是**宿主机（构建上下文）路径**，末尾的 `.` 是**容器内目标**，即当前 `WORKDIR`，也就是 `/app`。

### COPY 的其他细节

- 目录作源时，拷的是**目录内容**而不是目录本身：`COPY app/ /app` → `/app/main.py`，不是 `/app/app/main.py`。
- 源路径末尾 `/` 没意义（`app` 等价 `app/`）；目标末尾 `/` 表示"拷进这个目录"。
- `COPY --chown=1000:1000 . .` 一步设属主，省一次 `RUN chown`（那层会白胖）。
- `COPY --from=builder /wheels /wheels` 从别的构建阶段或别的镜像取文件。

### ADD vs COPY

| 场景        | COPY  | ADD                             |
| --------- | ----- | ------------------------------- |
| 普通文件/目录   | 推荐    | 行为一样                            |
| 本地 `.tar.gz` | 原样拷入  | 自动解压到目标目录                       |
| 远程 URL    | 不支持   | 支持下载，但不解压、不校验、缓存不可控             |

结论：**默认一律 COPY**，只有"确实要解压 tar 包"才用 ADD；下载文件用 `RUN curl -fsSL` 更可控。

## 4. 分层与缓存

三条核心规则：

1. **层只增不减**：`RUN rm` 只是在新层打"删除标记"，上一层的文件依然躺在镜像里占体积。
2. **缓存逐层失效**：某一层变了，它之后的**所有层**缓存全部作废。
3. **"变没变"的判断**：指令字符串变了，或 `COPY`/`ADD` 的文件内容校验和变了（BuildKit 按内容，旧构建器按元数据）。

所以：

```dockerfile
# ❌ 体积不会变小：删除发生在下一层
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# ✅ 删在同一层
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*

# ✅ 依赖先装、源码后拷：改代码不会触发重装
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

- `apt-get update` 和 `install` **必须在同一个 RUN**，否则缓存会让 update 复用几天前的索引，install 直接 404。
- `--no-install-recommends` 少装一批无关包；pip 用 `--no-cache-dir` 或 `ENV PIP_NO_CACHE_DIR=1`，别留 `~/.cache/pip`。
- 合并 RUN 能减少层数，但**不是越合越好**：按"变更频率"分组（系统包 / 语言依赖 / 源码），否则改一行代码就要重装全部依赖。
- `--no-cache` 全量重建；只想让某一步失效，改那一步的字符串或涉及的文件即可。

## 5. CMD vs ENTRYPOINT

两者都是启动命令，区别在于"谁会被 `docker run` 后面的参数覆盖"。

| 写法                          | 形式        | 特点                              |
| --------------------------- | --------- | ------------------------------- |
| `CMD ["python", "app.py"]`  | exec（推荐）  | 进程直接是 PID 1，能收到 SIGTERM         |
| `CMD python app.py`         | shell     | 实际是 `/bin/sh -c "..."`，应用成了 sh 的子进程 |
| `ENTRYPOINT ["python"]`     | exec（推荐）  | 固定入口，run 参数追加在后面                |
| `ENTRYPOINT python app.py`  | shell     | 多包一层 sh，**run 参数无法追加**          |

覆盖规则：

| Dockerfile 里写了    | `docker run img --port 8000` 的效果  |
| ---------------- | ------------------------------- |
| 只有 CMD           | **整体替换** CMD                    |
| 只有 ENTRYPOINT    | 参数**追加**到 ENTRYPOINT 后面          |
| ENTRYPOINT + CMD | CMD 是默认参数，被 run 参数**整体替换**       |

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "5000"]      # 不传参时的默认值
```

- `docker run img` → `python app.py --port 5000`
- `docker run img --port 8000` → `python app.py --port 8000`

shell 形式的实际后果：

- 应用不是 PID 1 → 收不到 SIGTERM → `docker stop` 干等 10 秒后 SIGKILL，优雅退出、测试 teardown 全部失效。
- 既要变量展开又要 exec 形式：`ENTRYPOINT ["sh", "-c", "python app.py --port $PORT"]`。
- 需要初始化脚本再启主进程：写 `entrypoint.sh`，最后一行 `exec "$@"` 让主进程接管 PID 1。

## 6. ARG vs ENV

| | ARG | ENV |
| --- | --- | --- |
| 生效期 | 仅构建期 | 构建期 + 容器运行时 |
| 谁可见 | `RUN` 可见，`CMD`/运行时不可见 | `RUN`、`CMD`、容器内进程都可见 |
| 传值方式 | `--build-arg` | `docker run -e` / compose `environment` |
| 是否留在镜像 | 记录在 `docker history` | 写进镜像配置 |

```dockerfile
ARG PY_VERSION=3.12          # FROM 之前可用
FROM python:${PY_VERSION}-slim
ARG PY_VERSION               # FROM 之后必须重新声明才能继续用
RUN python -V

ENV APP_ENV=production       # 运行时也能读到
```

- 版本号、镜像源、编译开关这类"构建参数"用 ARG；端口、路径、运行环境用 ENV。
- **别用 ARG/ENV 存密钥**：ARG 会出现在 `docker history`，ENV 会持久化进镜像。敏感值走 BuildKit secret：`RUN --mount=type=secret,id=pip_conf ...`。

## 7. 多阶段构建

编译期需要 gcc/JDK/node_modules，运行期只需要产物 → 把"编译器"留在构建阶段，运行镜像只留成品。

```dockerfile
# ---- 阶段 1：构建 ----
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip wheel --no-cache-dir -r requirements.txt -w /wheels

# ---- 阶段 2：运行 ----
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/* \
 && rm -rf /wheels
COPY . .
USER app
CMD ["python", "app.py"]
```

- 阶段之间**不共享文件系统**，只有显式 `COPY --from=` 的文件才会过去。
- 调试某个阶段：`docker build --target builder -t app:builder .`。
- 常见变体：先用沉重镜像跑 `pytest`（`FROM builder AS test`），再把产物 COPY 进精简的运行镜像——测试依赖不进生产镜像。

## 8. 完整示例

```dockerfile
FROM python:3.12-slim

# 1. 运行期环境变量
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 2. 系统依赖与缓存清理写在同一条 RUN
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

# 3. 依赖先装：改源码不重装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 源码最后拷
COPY . .

# 5. 非 root 运行
RUN useradd -m -u 1000 app && chown -R app:app /app
USER app

EXPOSE 5000

# 6. 健康检查：compose / 编排靠它判断"可用"
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://127.0.0.1:5000/healthz || exit 1

# 7. 入口固定，参数可覆盖
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0", "--port", "5000"]
```

配套的 `.dockerignore`：

```gitignore
.git
.venv
__pycache__
*.log
.env
```

## 9. 常见坑

- **`EXPOSE` 不映射端口**：只是声明，对外访问还得 `docker run -p 8080:5000`；用 compose 时靠 `ports` 才真正映射。
- **`COPY . .` 打包了脏东西**：`.env`、`.git`、本地 `.venv` 一起进镜像 → 必须写 `.dockerignore`。
- **镜像莫名很大**：`docker history` 看每层大小，多半是 apt 缓存、pip 缓存，或者多阶段没做。
- **Tab 缩进**：续行符是 `\`，别用 Tab 对齐，编辑器设成空格。
- **改了代码容器没变**：镜像不会自动更新，要么重新 `docker build`，要么开发期用 volume 挂源码热更新（见 compose 笔记）。
- **开发镜像 vs 生产镜像**：开发挂 volume + `debug=True`；生产不挂源码、非 root、不开调试、依赖用多阶段瘦身。
- **tag 别只用 `latest`**：`app:1.0` → `app:2.0`，回滚时 `docker run app:1.0` 就行；同一 tag 反复覆盖会让部署版本无法追溯。
- **`localhost` 的含义**：容器内 `localhost` 是容器自己。`HEALTHCHECK` 探自己写 `127.0.0.1` 没问题，但连其他容器必须用**服务名**（如 `mysql:3306`）。

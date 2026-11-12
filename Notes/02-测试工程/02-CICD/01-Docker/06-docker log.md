# 基本概念和参数
Docker 提供了一套日志管理机制，负责收集容器主进程输出的标准输出和标准错误，并交给对应的 Logging Driver 处理。
### stdout 和 stderr 是什么？

Linux 进程默认有三个标准文件描述符：

|文件描述符|名称|用途|
|---|---|---|
|0|stdin|标准输入|
|1|stdout|标准输出|
|2|stderr|标准错误|

其中：
- `stdin`：程序接收输入的地方。
- `stdout`：程序输出正常运行信息的地方。
- `stderr`：程序输出错误信息的地方。

例如：

```python
import sys

print("Application started")

print("Database connection failed", file=sys.stderr)
```

第一条信息写入 stdout，第二条信息写入 stderr。

对于 Docker 来说，这两个输出流非常重要。

**Docker 默认通过容器主进程的 stdout 和 stderr 收集日志。**

其基本关系是：

![[Pasted image 20261104173152.png]]

需要注意：Docker 并不要求应用程序必须使用某个特定日志框架。只要应用程序将日志写入 stdout 或 stderr，Docker 就可以按照 Logging Driver 的配置进行处理。

常见参数

|参数|作用|
|---|---|
|`-f`|持续跟踪日志，类似 Linux 的 `tail -f`|
|`--tail N`|只显示最后 N 行日志|
|`--since`|显示指定时间之后的日志|
|`--timestamps`|在日志前显示时间戳|


```
# 查看日志
docker logs <name_or_id> 
# 查看实时日志
docker logs -f <name_or_id> 
```

# Logging Driver
Logging Driver 解决的是 日志由谁收集、以什么格式存储、存储在哪里，以及如何管理日志。
### 1. 什么是 Logging Driver？
Docker 提供了 Logging Driver（日志驱动）机制，用来处理容器主进程产生的 stdout 和 stderr。

可以把它理解为 Docker 的日志存储后端。应用程序负责输出日志，Logging Driver 负责接收和处理日志。

Docker 支持多种 Logging Driver，不同驱动的存储方式不同。

目前重点学习三种：

| Logging Driver | 日志存储位置                | 主要特点                 |
| -------------- | --------------------- | -------------------- |
| `json-file`    | Docker 管理的 JSON 日志文件  | 默认驱动，兼容性好            |
| `local`        | Docker 管理的本地日志文件      | 存储效率较高，支持自动轮转        |
| `journald`     | Linux systemd Journal | 可以使用 journalctl 统一查询 |

### 2. json-file **重点掌握**
`json-file` 是 Docker 常见的默认日志驱动。

它会将容器的 stdout 和 stderr 输出保存为 JSON 格式的日志文件。

例如，日志内容可能是：

```bash
{"log":"Application started\n","stream":"stdout","time":"2026-11-04T09:00:00.000000000Z"}
{"log":"Database connection failed\n","stream":"stderr","time":"2026-11-04T09:00:01.000000000Z"}
```

每条日志记录包含三个重要字段：

- `log`：实际日志内容。
- `stream`：日志来自 stdout 还是 stderr。
- `time`：Docker 接收到日志的时间。

日志文件通常保存在宿主机 Docker 数据目录下的容器目录中，例如：

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

但要注意，WSL2、Docker Desktop 或自定义 Docker data-root 环境下，实际位置可能不同。

它的优点是兼容性好，能够直接配合 `docker logs` 使用。缺点是如果不配置日志轮转，日志文件可能持续增长，最终占满磁盘。

### 3. local

`local` 是 Docker 提供的本地日志驱动，专门针对本地容器日志存储进行了优化。

它与 `json-file` 的主要区别是存储格式和管理机制。

| 对比项         | json-file | local            |
| ----------- | --------- | ---------------- |
| 存储格式        | JSON      | Docker 自身管理的日志格式 |
| 日志压缩        | 默认不压缩     | 默认启用压缩           |
| 自动轮转        | 需要配置      | 默认启用             |
| 存储效率        | 相对较低      | 通常更高             |
| docker logs | 支持        | 支持               |

`local` 默认启用日志轮转，而 `json-file` 默认不会自动轮转，需要手动配置。

`local` 的一个重要优势是默认就会进行日志轮转，降低日志无限增长的风险。

Docker 默认的 `local` 驱动配置通常为单个日志文件最大 20 MB，最多保留 5 个文件，并启用压缩。

因此，Docker 官方也推荐在不需要其他日志系统集成的情况下考虑使用 `local`。

### 4. journald

`journald` 是 Linux systemd 提供的日志管理系统。

当 Docker 使用 `journald` 作为 Logging Driver 时，容器日志会被发送到宿主机的 systemd Journal，而不是按照 `json-file` 的方式存储。

例如，可以使用：

```
journalctl CONTAINER_NAME=test-platform-fastapi
```

查看指定容器的日志。

也可以使用：

```
journalctl -f CONTAINER_NAME=test-platform-fastapi
```

实时跟踪日志。

它的优势是可以将 Docker 容器日志与宿主机其他系统服务日志统一管理。不过，它依赖宿主机的 systemd Journal 环境，存储容量和保留策略也需要结合 journald 自身配置管理。

### 5. 三种驱动应该怎么选？

结合实际运维场景：
- `json-file`：适合希望使用传统 JSON 日志文件、兼容现有日志采集工具的环境。
- `local`：适合普通 Docker 主机，希望降低日志磁盘占用并减少手动配置的环境。
- `journald`：适合使用 systemd 的 Linux 服务器，希望统一管理系统和容器日志的环境。

# 配置日志轮转
### 1. Compose 中 logging 的配置结构

Docker Compose 使用 `logging` 配置容器日志驱动。

```yml
services:
  fastapi:
  
	environment:
	  LOG_LEVEL: INFO  # 可以通过环境变量告诉里面的服务 采用什么级别

    logging:
      driver: json-file
      options:
        max-size: "10m"  # 单个日志文件达到多大时触发轮转
        max-file: "3"    # 最多保留多少个日志文件
```

日志级别按**严重程度从低到高**排序，最常见的是：
`TRACE < DEBUG < INFO < WARN < ERROR < FATAL`

- **TRACE**：最细粒度，追踪每一步（很少开启）
- **DEBUG**：调试信息
- **INFO**：常规运行信息（默认生产级别附近）

- **WARN**：警告，不影响运行但需关注
- **ERROR**：错误，功能受影响
- **FATAL / CRITICAL**：致命错误，系统可能无法继续

补充说明：

1. **级别越高越严重**，配置日志级别时，**设置某个级别会输出该级别及以上的日志**。例如设为 `INFO`，则输出 INFO、WARN、ERROR、FATAL，不输出 DEBUG、TRACE。

2. **不同框架命名略有差异**：
    - Python `logging`：`DEBUG < INFO < WARNING < ERROR < CRITICAL`
    - Log4j / SLF4J：`TRACE < DEBUG < INFO < WARN < ERROR < FATAL`
    - Syslog：`debug < info < notice < warning < err < crit < alert < emerg`
3. **记忆口诀**：从“最啰嗦”到“最严重”——TRACE → DEBUG → INFO → WARN → ERROR → FATAL。

所以一般排序就是：**TRACE < DEBUG < INFO < WARN < ERROR < FATAL**。



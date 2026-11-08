# 第一阶段：Docker 资源管理的基本原理

本阶段首先需要理解一个核心问题：Docker 容器为什么能够限制 CPU 和内存？

答案是 Linux 内核提供的 cgroups（Control Groups，控制组）机制。

### 1. 为什么容器需要资源限制？

先结合你的 DockerTestPlatform 项目来看。

目前项目包含四个主要服务：FastAPI、MySQL、Redis 和 Nginx。它们虽然运行在不同的容器中，但实际上共享宿主机的 CPU、内存、磁盘和网络资源。

例如，假设你的 WSL2 虚拟机分配了 4GB 内存，运行情况如下：

|服务|内存使用量|
|---|---|
|FastAPI|300MB|
|MySQL|800MB|
|Redis|200MB|
|Nginx|100MB|
|宿主机及其他进程|1.5GB|
|合计|2.9GB|

假设 FastAPI 因为程序 Bug 出现内存泄漏，内存使用量不断增长：

```
FastAPI: 300MB → 500MB → 1GB → 2GB
```

最终可能导致整个系统内存不足。

这会带来几个问题：

1. FastAPI 占用大量内存，挤压 MySQL 和 Redis 的可用内存。
    
2. MySQL 可能因为内存不足而无法正常处理请求。
    
3. Linux 内核可能触发 OOM Killer，选择进程杀死以释放内存。
    

这就是资源限制的意义：防止某个容器无限制地消耗宿主机资源，影响其他容器和整个系统。

例如，我们可以给 FastAPI 设置 512MB 内存上限。这样，即使 FastAPI 出现内存泄漏，也不能无限制地占用宿主机内存。

需要注意，资源限制并不意味着容器拥有独立的 CPU 或内存，而是 Linux 内核在多个进程之间执行资源管理。

### 2. 什么是 Linux cgroups？

cgroups：Control Groups

cgroups 是 Linux 内核提供的一种资源管理机制，可以将多个进程划分到不同的控制组，并对这些进程的资源使用进行限制、统计和控制。

Docker 正是利用 cgroups 实现容器的资源限制。

可以把它理解为一个资源管理器：

|资源|cgroups 能做什么|
|---|---|
|CPU|限制进程能够使用多少 CPU 时间|
|Memory|限制进程能够使用多少内存|
|I/O|控制进程的块设备读写资源|
|PIDs|限制进程数量|

例如，我们创建三个容器：

```
Linux Kernel
    │
    ├── cgroup: FastAPI
    │      CPU: 最多 1 核
    │      Memory: 最多 512MB
    │
    ├── cgroup: MySQL
    │      CPU: 最多 2 核
    │      Memory: 最多 2GB
    │
    └── cgroup: Redis
           CPU: 最多 0.5 核
           Memory: 最多 256MB
```

这里每个容器对应一个独立的资源控制组。

当 FastAPI 尝试使用超过 512MB 的内存时，内核会根据内存控制策略进行处理，而不是允许它无限制地增长。

注意：cgroups 不负责实现容器的文件系统隔离、网络隔离或进程命名空间隔离。 这些分别涉及 Docker 使用的其他 Linux 内核机制。

### 3. cgroups v1 和 v2

Linux cgroups 目前主要有两个版本。

|对比|cgroups v1|cgroups v2|
|---|---|---|
|设计|不同资源控制器相对独立|统一层级结构|
|资源管理|多个控制器可以有不同层级|统一管理|
|使用情况|较早期 Linux 系统|现代 Linux 系统普遍采用|

你目前使用 Ubuntu / WSL2，具体采用哪个版本可以通过下面的命令查看：

```
stat -fc %T /sys/fs/cgroup
```

判断结果：

```
cgroup2fs
```

代表使用 cgroups v2。

如果输出：

```
tmpfs
```

通常代表使用 cgroups v1。

### 4. cgroups 如何与 Docker 配合？

整个过程可以理解为：

![[Pasted image 20261104182832.png]]

因此，Docker 本身并不是直接控制 CPU 或内存的底层机制，它主要负责配置，由 Linux 内核通过 cgroups 执行。

# 使用 docker stats 监控容器资源
## 1. docker stats 是什么？

`docker stats` 是 Docker 提供的实时资源监控命令，可以查看容器当前的资源使用情况。

它主要监控以下指标：

|指标|含义|可以发现的问题|
|---|---|---|
|CPU %|CPU 使用率|CPU 是否过高|
|MEM USAGE / LIMIT|内存使用量及限制|内存是否接近上限|
|MEM %|内存使用百分比|内存使用压力|
|NET I/O|网络接收和发送数据量|网络流量是否异常|
|BLOCK I/O|块设备读写数据量|磁盘 I/O 是否异常|
|PIDS|容器内进程数量|进程数量是否异常增长|

这里需要特别注意：docker stats 主要用于观察资源使用情况，并不直接告诉你资源使用异常的原因。

```bash
# 查看所有容器的实时资源使用情况
docker stats
# 只查看 FastAPI
docker stats test-platform-fastapi
# 只获取一次监控数据
docker stats --no-stream
```
## 2. 限制CPU


### 三个参数：`--cpus`、`--cpu-shares` 和 `--cpuset-cpus`

| 参数              | 核心作用            | 是否限制 CPU 上限 |
| --------------- | --------------- | ----------- |
| `--cpus`        | 限制 CPU 总使用量     | 是           |
| `--cpu-shares`  | 控制 CPU 竞争时的相对权重 | 否           |
| `--cpuset-cpus` | 指定允许使用的 CPU 核心  | 间接限制        |

### 查看容器CPU限制
```bash
docker inspect cpu-test \
  --format 'NanoCpus={{.HostConfig.NanoCpus}}'
  
# 预期结果，1,000,000,000 纳秒 CPU 时间的额度对应一个 CPU 核心的计算能力配置。
NanoCpus=1000000000
```

### CPU排查思路：
1. 使用 `docker stats` 查看容器 CPU 使用率。如果长期接近配置的 CPU 上限，需要进一步检查。
2. 使用 `docker inspect` 查看容器是否配置了 CPU 限制。
3. 查看 cgroups 信息的 `cpu.max`，确认实际生效的 CPU 配额。
4. 查看 `cpu.stat`，重点比较 `nr_throttled` 和 `throttled_usec` 的增量，判断是否持续发生 CPU 节流。

## 3. 限制内存
### 三个核心参数：`--memory`、`--memory-swap` 和 `--memory-reservation`

三个参数的区别

| 参数                     | 类型  | 主要作用         |
| ---------------------- | --- | ------------ |
| `--memory`             | 硬限制 | 限制容器最大内存额度   |
| `--memory-swap`        | 硬限制 | 限制内存与Swap总额度 |
| `--memory-reservation` | 软限制 | 内存压力下的资源管理参考 |
### 检查容器内存限制：
```bash
docker inspect memory-test \
  --format 'Memory={{.HostConfig.Memory}} MemorySwap={{.HostConfig.MemorySwap}} MemoryReservation={{.HostConfig.MemoryReservation}}'
  
# 预期结果
Memory=536870912 MemorySwap=1073741824 MemoryReservation=268435456
```
 
### 内存排查思路：
第一步：查看容器退出状态。
 ```bash
 docker inspect test-platform-fastapi \
  --format 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}'
 ```
如果 `OOMKilled=true`，说明 Docker 确认容器进程因 OOM 被杀死。

第二步：查看容器日志。
```bash
docker logs --tail 100 test-platform-fastapi
```

注意，OOM 发生时进程可能来不及输出错误日志，因此没有异常日志并不能排除 OOM。

第三步：查看内存限制和实际使用量。
```bash
docker stats test-platform-fastapi

docker inspect test-platform-fastapi \
  --format 'Memory={{.HostConfig.Memory}} MemorySwap={{.HostConfig.MemorySwap}}'
```

第四步：查看内核 OOM 记录。

```bash
sudo dmesg -T | grep -i -E 'oom|killed process|out of memory'
```

如果需要进一步确认 cgroups v2 的记录，可以查看对应容器 cgroup 中的 `memory.events`，重点关注 `oom` 和 `oom_kill`。

```bash
# 获取pid
container=xxx
PID=$(docker inspect -f '{{.State.Pid}}' $container)
# 获取cgroup
CGROUP=$(awk -F: '$1=="0"{print $3}' /proc/$PID/cgroup)
# 进入cgroup目录
cd /sys/fs/cgroup${CGROUP}/

# 查看内存事件
cat memory.events
# `max` 表示内存使用触及上限次数
# `oom` 发生了OOM 状态事件的次数
# `oom_kill`内核执行OOM Kill的次数
```


# Docker OOM、OOMKilled 与 Exit Code 137
## 什么是 OOM？

OOM 是 Out Of Memory 的缩写，表示系统内存不足。

结合 DockerTestPlatform 项目，假设 FastAPI 设置了 512MiB 内存上限：

```
--memory=512m
```

当 FastAPI 不断申请内存，最终达到限制时，Linux 内核会尝试回收可回收的内存。

如果仍然无法满足内存申请，就可能触发 OOM Killer。

### OOM Killer 是什么？

OOM Killer 是 Linux 内核的一种内存保护机制。

当内存分配无法满足时，内核可以选择并杀死某些进程，释放内存，避免系统因内存耗尽而无法正常运行。

需要区分两种情况：

情况一：容器自身达到内存限制。
	例如 FastAPI 的内存上限是 512MiB，它触发了 cgroup OOM，内核可能杀死 FastAPI 容器中的进程。

情况二：整个宿主机内存不足。
	例如 WSL2 总共只有 4GB 内存，FastAPI、MySQL 和其他进程共同消耗了大量内存，最终触发宿主机级别的 OOM。

这两种情况都可能导致进程被杀死，但影响范围不同。

## 2. 什么是 OOMKilled？

Docker 提供了一个字段 `OOMKilled`，用于记录容器是否因 OOM 被杀死。

执行：

```BASH
docker inspect test-platform-fastapi \
  --format '{{.State.OOMKilled}}'
  
可能输出true  则表示 Docker 记录到该容器发生了 OOM Kill。
如果输出false 则表示没有记录到该容器因 OOM 被杀死（不代表没有发生）
```

注意，`OOMKilled=false` 不代表宿主机从未发生过 OOM，也不能单凭这个字段排除所有内存相关故障。

## 3. 什么是 Exit Code 137？

这是排查 Docker 容器异常退出时经常遇到的退出码。

Linux 进程被信号杀死时，通常可以通过退出状态反映信号编号。

计算方式：128+9=137

其中：
- 128 是信号退出状态的基础值。
- 9 对应 SIGKILL。
- 137 表示进程以 SIGKILL 终止。

例如：

```bash
docker inspect test-platform-fastapi \
  --format '{{.State.ExitCode}}'
# 输出137
```

这表示容器主进程以 `SIGKILL` 结束。

非常重要：**Exit Code 137 不等于 OOM**。

因为 SIGKILL 不仅可能由 OOM Killer 发送，也可能来自人工执行 `kill -9`、`docker kill`，或者其他系统管理操作。

因此，排查时需要结合多个信息
```bash
docker inspect test-platform-fastapi \
  --format 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}'
```



# 查看容器的 cgroups 信息

```bash
# 获取pid
container= 
PID=$(docker inspect -f '{{.State.Pid}}' $container)
# 获取cgroup
CGROUP=$(awk -F: '$1=="0"{print $3}' /proc/$PID/cgroup)
# 进入cgroup目录
cd /sys/fs/cgroup${CGROUP}/

# 查看CPUmax
cat cpu.max
# 假如输出 50000 100000  
含义是每 100000 微秒的调度周期，最多允许使用 50000 微秒 CPU 时间，也就是半个 CPU 核心。


# 查看cpu状态
cat cpu.stat
# `nr_throttled` 发生 CPU 节流的周期数量
# `throttled_usec` 的被节流的累计时间
# 看这两个增量 判断是否持续发生 CPU 节流

# 查看内存事件
cat memory.events
# `max` 表示内存使用触及上限次数
# `oom` 发生了OOM 状态事件的次数
# `oom_kill`内核执行OOM Kill的次数
```

# 查看内核 oom 相关的日志
```bash
sudo dmesg -T | grep -i -E 'oom|killed process|out of memory'
```
正式版本 Tag 应当代表一个确定的发布版本，例如 `20261109-01`。发布后不应该再将其他镜像覆盖到这个 Tag 上。

开发测试 Tag 则用于开发过程中反复构建和测试，例如 `developer-test`。它可以随着开发进度不断更新。

适合当前项目的简单规范。

|Tag 类型|示例|用途|
|---|---|---|
|开发版本|`developer-test`|日常开发测试|
|正式版本|`20261109-01`|正式发布|
|下一次正式版本|`20261110-01`|下一次发布|

工程规范：正式版本 Tag 创建后，不允许覆盖。

另外，Tag 只是人类可读的版本标识。真正用于精确识别镜像内容的是 Digest，例如 `sha256:xxxx`。后续我们会在发布验收中使用 Digest 确认镜像身份。


对于目前的项目，我们先建立以下流程：
![[mermaid-diagram.png]]


# 正式环境和开发环境隔离
新建`compose.prod.yml`, `.env.prod` 等文件
注意以下配置：
`compose.prod.yml` 使用正式版的镜像仓库的镜像，而不是`latest`
```yml
services:
  fastapi:
    build: !reset null
    image: localhost:5000/platform-fastapi:20261109-01
```

```bash
# 部署正式环境
docker compose -p platform-prod --env-file .env.prod \
  -f compose.yml -f compose.prod.yml
  up -d
```


# 滚动更新
滚动更新的核心思想是：先启动新版本实例，确认新实例健康后，再逐步淘汰旧版本实例。

![[Pasted image 20261110175406.png]]

这里有三个关键条件：
1. 新版本实例必须通过健康检查，才能接收流量。
2. 旧版本实例必须在停止前完成正在处理的请求。
3. 流量入口必须能够识别实例变化，并将请求分配给健康实例。

但是 Docker Compose 本身没有 Kubernetes 式的原生滚动更新控制器

# 蓝绿部署（Blue-Green Deployment）


# 数据库变更与应用版本兼容性
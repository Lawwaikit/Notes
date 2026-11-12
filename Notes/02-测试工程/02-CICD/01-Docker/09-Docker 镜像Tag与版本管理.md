### 常见的镜像 Tag 规范

常见的版本管理方式有以下几种：

| Tag 示例         | 含义            | 适用场景   |
| -------------- | ------------- | ------ |
| `1.0.0`        | 正式版本          | 生产部署   |
| `1.0.1`        | 补丁版本          | Bug 修复 |
| `1.1.0`        | 功能更新          | 新增功能   |
| `2.0.0`        | 重大版本          | 不兼容变更  |
| `dev`          | 开发版本          | 开发测试   |
| `20261108-001` | 日期及构建序号       | 持续集成   |
| `git-a1b2c3d`  | Git Commit 标识 | 追踪代码版本 |
### 为镜像增加Tag
```bash
docker tag test-platform-fastapi:latest test-platform-fastapi:1.0.0

Usage:  docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

### Image Digest
例如：
```text
test-platform-fastapi@sha256:abcdef123456...
```

Digest 是基于镜像内容计算得到的内容寻址标识。
与 Tag 不同，Digest 不会因为给镜像重新打 Tag 而发生变化。

例如，Compose 可以这样指定镜像：
```yml
services:
  fastapi:
    image: test-platform-fastapi@sha256:abcdef123456...
```
这样部署时，Docker 会按照指定 Digest 获取镜像，而不是依赖某个可能发生变化的 Tag。

第一步：查看当前镜像的 Digest 信息。

执行：
```bash
docker image inspect test-platform-fastapi:1.0.0 \
	--format '{{json .RepoDigests}}'

输出
["test-platform-fastapi@sha256:16b4989de8b5f7dec971e6c540f47ff1562c35a3044ac19074958ce537503e5c"]
```
注意：如果镜像尚未从仓库拉取或推送，`RepoDigests` 可能为空，这是正常的。

第二步：查看镜像 Image ID。
```bash
docker image inspect test-platform-fastapi:1.0.0 --format '{{.Id}}'

输出
sha256:16b4989de8b5f7dec971e6c540f47ff1562c35a3044ac19074958ce537503e5c
```

第三步：理解两种标识的区别。

这里需要特别注意：
- Image ID 标识本地镜像配置对象。
- RepoDigest 标识仓库中的镜像 Manifest 内容。
- Tag 是指向镜像的可变名称。

# 私有镜像仓库

镜像仓库（Registry）负责存储和分发镜像。

![[Pasted image 20261109160954.png]]

## Docker Registry：Docker 官方提供的基础 Registry 实现
搭建本地Registry：使用 HTTP 和无认证配置，只为了学习镜像上传、下载的工作机制。这不是生产环境的安全配置。


### 部署 registry 容器
```bash
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2
```

### 镜像仓库地址
接下来要理解一个重要概念：镜像名称实际上可以包含 Registry 地址。
例如：
```
test-platform-fastapi:1.0.0
```
这是一个没有明确指定 Registry 地址的镜像名称，Docker 默认使用 Docker Hub。

而：
```
localhost:5000/test-platform-fastapi:1.0.0
```
则明确表示：
- Registry 地址：`localhost:5000`
- 镜像名称：`test-platform-fastapi`
- Tag：`1.0.0`

这意味着 Docker 会将镜像上传到本机 5000 端口对应的 Registry，而不是 Docker Hub。

将本地镜像推到私有仓库
```bash
docker tag \
  test-platform-fastapi:20261109-01 \
  localhost:5000/test-platform-fastapi:20261109-01
  
docker images

docker push localhost:5000/test-platform-fastapi:20261109-01
```
Docker 会执行以下操作：
1. 检查本地镜像的各个 Layer。
2. 与 Registry 比较已有 Layer。
3. 上传 Registry 尚不存在的 Layer。
4. 上传镜像 Manifest，完成镜像发布。
如果某些 Layer 已经存在，Docker 通常会复用，不需要重复上传。

### 验证 Registry 中的镜像
```bash
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/test-platform-fastapi/tags/list
```

### 验证pull
```bash
# 确认有镜像
docker images localhost:5000/test-platform-fastapi

# 删除本地引用
docker image rm localhost:5000/test-platform-fastapi:20261109-01
docker image rm test-platform-fastapi:20261109-01 # 引用同一个id

# pull
docker pull localhost:5000/test-platform-fastapi:20261109-01

# 查看镜像
docker images localhost:5000/test-platform-fastapi

# 检查镜像id（要和原来一致）
docker image inspect \
  localhost:5000/test-platform-fastapi:20261109-01 \
  --format '{{.Id}}'
```

# HTTPS
HTTP 本身不提供传输加密。如果通过网络传输镜像，网络中的其他人可能截获传输内容。
例如，镜像可能包含应用程序代码、内部配置文件，甚至意外打包进去的敏感信息。
## Regisrty配置https
#### 基础理解
Registry 本身支持 TLS，只需要提供证书和私钥。
Registry 通过环境变量加载它们：

```yaml
environment:
  REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
  REGISTRY_HTTP_TLS_KEY: /certs/domain.key
```
同时将宿主机证书目录挂载到容器的 `/certs`。

#### 配置自签名证书
```bash
mkdir -p registry/certs

openssl version # 确认安装了openssl

openssl req \
  -newkey rsa:2048 \
  -nodes \
  -sha256 \
  -keyout registry/certs/domain.key \
  -x509 \
  -days 365 \
  -out registry/certs/domain.crt \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

参数说明：
- `-newkey rsa:2048`：生成 RSA 私钥。
- `-nodes`：私钥不使用密码加密，方便 Registry 自动启动。
- `-x509`：直接生成自签名证书。
- `-days 365`：证书有效期为一年。
- `-addext`：指定证书 SAN，支持 localhost 和 127.0.0.1。
这里生成的是自签名证书，适合本次实验，不建议直接作为面向公网的生产证书。

#### 创建容器
```bash
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)/registry/certs:/certs:ro" \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

#### 验证容器
```bash
# 容器正常
docker logs registry

# 验证 https请求
python3 -c "import ssl, urllib.request; c=ssl.create_default_context(cafile='registry/certs/domain.crt'); print(urllib.request.urlopen('https://localhost:5000/v2/', context=c).status)"
```

## Docker信任证书
### 1. Docker 如何管理 Registry 证书？

Docker 支持通过特定目录管理每个 Registry 的证书。
Linux Docker Engine 默认使用：
```bash
/etc/docker/certs.d/
```

目录结构如下：
```bash
/etc/docker/certs.d/
└── localhost:5000/
    └── ca.crt
```

这里有三个重要约定：
- 目录名称必须与 Registry 地址一致，包括端口。
- `ca.crt` 用于提供客户端信任的 CA 证书。
- Docker 会自动读取对应目录中的证书，不需要在 Compose 中配置。

本次实验使用自签名证书，因此直接将 `domain.crt` 作为信任证书安装。

注意：这是因为当前证书是自签名证书，证书本身同时承担服务器证书和信任锚的角色。实际企业环境通常由内部 CA 签发服务器证书，客户端应该信任 CA 根证书，而不是直接信任每台服务器的叶子证书。

### 2. 配置 Docker 证书目录

首先确认 Docker Engine 的运行环境。

你目前使用的是 WSL，因此需要特别注意：证书必须安装在运行 Docker Engine 的 Linux 环境中，而不一定是你执行命令的终端所在环境。

先执行：`docker context show`
然后执行：`docker info --format '{{.DockerRootDir}}'`
如果输出 `/var/lib/docker`, 通常说明当前 Docker Engine 使用 Linux 环境。

接下来查看 Docker 是否运行在当前 WSL 发行版中：
```
ps -ef | grep '[d]ockerd'
```

如果能看到 `dockerd` 进程，说明 Docker Engine 就运行在当前 Linux 环境。

### 3. 安装证书

如果确认 Docker Engine 运行在当前 WSL 环境，执行：
```
sudo mkdir -p /etc/docker/certs.d/localhost:5000
```

复制证书：
```
sudo cp registry/certs/domain.crt /etc/docker/certs.d/localhost:5000/ca.crt
```
这里不需要复制 `domain.key`。

特别注意：不要把服务器私钥复制到 Docker 客户端。 客户端只需要信任证书，不需要拥有服务器私钥。

### 4. 验证 Docker 是否信任证书

执行：
```
docker pull localhost:5000/test-platform-fastapi:20261109-01
```
预期能够正常 Pull，不再出现证书验证错误。

再执行：
```
docker push localhost:5000/test-platform-fastapi:20261109-01
```
预期能够正常 Push。

## 配置 Basic Authentication
目前 Registry 虽然启用了 HTTPS，但没有身份认证。任何能够访问 Registry 的客户端，都可能直接执行镜像 Push、Pull 等操作。

### 1. 理解认证流程
Basic Authentication 的核心是客户端提供用户名和密码，Registry 验证凭据后决定是否允许访问。
需要注意，Basic Authentication 本身并不加密用户名和密码，因此必须与 HTTPS 配合使用。否则凭据可能在网络传输过程中被窃取。

### 2. 准备认证文件
Registry 官方镜像支持通过 `htpasswd` 文件配置 Basic Authentication。我们先使用临时容器生成认证文件，不需要在宿主机安装额外软件。

在项目根目录执行：
```bash
mkdir -p registry/auth

docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn admin '你的实验密码' \
  > registry/auth/htpasswd
```

这里有几个关键点：
- `-B` 表示使用 bcrypt 算法哈希密码。
- `-b` 表示通过命令行参数传入密码。
- `-n` 表示将结果输出到标准输出，而不是直接写入文件。
- `>` 将生成的认证记录保存到 `registry/auth/htpasswd`。

### 3. 验证认证文件
执行：
```
cat registry/auth/htpasswd
```
预期会看到类似：
```
admin:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

其中 `admin` 是用户名，后面的内容是 bcrypt 密码哈希，而不是明文密码。

### 4. 启用 Registry 身份认证
Registry 官方镜像通过以下三个环境变量启用 Basic Authentication：
`REGISTRY_AUTH`：指定认证类型，设置为 `htpasswd` 
`REGISTRY_AUTH_HTPASSWD_REALM`：设置认证领域名称              
`REGISTRY_AUTH_HTPASSWD_PATH`：指定 htpasswd 文件路径      

启动容器
```bash
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)/registry/certs:/certs:ro" \
  -v "$(pwd)/registry/auth:/auth:ro" \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

验证未登录
```bash
python3 -c "import ssl, urllib.request, urllib.error; c=ssl.create_default_context(cafile='registry/certs/domain.crt'); req=urllib.request.Request('https://localhost:5000/v2/'); exec('try:\n urllib.request.urlopen(req, context=c)\n print(\"Unexpected success\")\nexcept urllib.error.HTTPError as e:\n print(\"HTTP Status:\", e.code)\n print(\"WWW-Authenticate:\", e.headers.get(\"WWW-Authenticate\"))')"

# 预期结果
HTTP Status: 401
WWW-Authenticate: Basic realm="Registry Realm"

# 尝试pull
docker pull localhost:5000/test-platform-fastapi:20261109-01
```

验证已登录
```bash
docker login localhost:5000
cat ~/.docker/config.json

docker pull localhost:5000/test-platform-fastapi:20261109-01
docker push localhost:5000/test-platform-fastapi:20261109-01
```

# Registry 安全加固
身份认证不等于权限控制。 Basic Authentication **只能验证用户身份**，不能直接实现不同用户只能 Pull 或只能 Push 的**细粒度权限管理**。
目前使用的 Docker Distribution Registry（`registry:2`）通过 `htpasswd` 实现 Basic Authentication。它能够识别不同用户，但原生 htpasswd 认证并不提供完善的 RBAC（基于角色的访问控制）

# Registry 镜像清理与存储回收
首先理解 Registry 的存储结构
Docker Registry 并不是简单地按照 Tag 保存一份完整镜像。它主要通过以下几个概念组织镜像数据：
- **Manifest**：记录镜像配置和各层 Layer 的摘要信息。
- **Blob**：实际存储镜像配置和文件系统层数据。
- **Tag**：指向某个 Manifest 的可变名称。
![[Pasted image 20261109195700.png]]

**删除Tag不等于释放磁盘空间**
Registry 的镜像清理需要区分两个动作。
第一步是删除 Manifest，使对应的镜像版本不再被 Registry 引用。第二步是执行 Garbage Collection（垃圾回收），清理已经没有任何 Manifest 引用的 Blob

删除镜像版本和真正释放磁盘空间是两个独立操作。

还有一个重要的安全限制：Registry 执行 Garbage Collection 时，不应该同时接受 Push 请求，否则可能误删正在上传或使用的数据。

开启Registry 删除功能，显式启用 `REGISTRY_STORAGE_DELETE_ENABLED`=true
```bash
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)/registry/certs:/certs:ro" \
  -v "$(pwd)/registry/auth:/auth:ro" \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry:2
```

通过HTTP API删除manifest
```
GET /v2/<name>/manifests/<tag>  获取manifest
GET /v2/<name>/tags/list
DELETE /v2/<name>/manifests/<digest>
```

垃圾回收模拟
```bash
docker run --rm \
  -v registry-data:/var/lib/registry \
  registry:2 \
  garbage-collect \
  --dry-run \
  /etc/docker/registry/config.yml
```
这里有几个关键点：
- `registry-data` 挂载的是实际存储镜像数据的 Volume。
- `garbage-collect` 启动 Registry 的垃圾回收命令。
- `--dry-run` 只模拟删除，不会真正删除 Blob。
- `/etc/docker/registry/config.yml` 是 Registry 镜像内置的配置文件。

重点关注最后几行，可能出现类似：
```
4 blobs marked, 3 blobs eligible for deletion

blob eligible for deletion: sha256:xxxx
blob eligible for deletion: sha256:xxxx
blob eligible for deletion: sha256:xxxx
```
其中：
- `blobs marked` 表示仍然被有效 Manifest 引用的 Blob。
- `blobs eligible for deletion` 表示已经没有有效引用、符合回收条件的 Blob。
注意，Dry Run 统计的是 Blob 数量，不是镜像数量，也不代表对应数量的镜像版本。


# Registry 数据备份与恢复
### Registry 需要备份什么

你目前的 Registry 使用 Docker Volume `registry-data` 保存镜像数据，另外还有证书和认证文件。

生产环境中需要考虑备份以下内容：

|数据|当前存放位置|是否需要备份|
|---|---|---|
|镜像数据|`registry-data`|必须|
|TLS 证书|`registry/certs/domain.crt`|建议|
|TLS 私钥|`registry/certs/domain.key`|必须安全备份|
|用户认证文件|`registry/auth/htpasswd`|必须|
|Registry 配置|Docker 启动参数|必须记录|

这里有一个重要原则：备份必须保证数据一致性。
Registry 在 Push 镜像时，可能同时写入 Manifest 和 Blob。如果在写入过程中直接复制 Volume，备份可能不完整。因此，我们先停止 Registry，再备份数据。


### 实际操作：
#### 备份 Registry Volume
```bash
mkdir -p backups/registry
docker stop registry
# 使用 `:ro` 将源 Volume 以只读方式挂载，避免备份容器修改原始数据。
docker run --rm \
  -v registry-data:/source:ro \
  -v "$(pwd)/backups/registry:/backup" \
  alpine:latest \
  tar -czf /backup/registry-data.tar.gz -C /source .
# 检查备份文件
ls -lh backups/registry/registry-data.tar.gz
docker start registry
```

验证备份内容
```
tar -tzf backups/registry/registry-data.tar.gz | head -30
```


#### 恢复备份测试
```bash
# 创建恢复测试 Volume
docker volume create registry-restore-test
# 将备份解压到新 Volume
docker run --rm \
  -v registry-restore-test:/restore \
  -v "$(pwd)/backups/registry:/backup:ro" \
  alpine:latest \
  tar -xzf /backup/registry-data.tar.gz -C /restore
  
# 检查恢复后的目录
docker run --rm \
  -v registry-restore-test:/restore:ro \
  alpine:latest \
  ls -lah /restore/docker/registry/v2
```

验证恢复后的Registry
```bash
docker run -d \
  --name registry-restore-test \
  -p 5001:5000 \
  -v registry-restore-test:/var/lib/registry \
  -v "$(pwd)/registry/certs:/certs:ro" \
  -v "$(pwd)/registry/auth:/auth:ro" \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

验证API
```bash
curl -k -u "admin:admin" https://localhost:5001/v2/
curl -k -u "admin:admin" https://localhost:5001/v2/test-platform-fastapi/tags/list
```



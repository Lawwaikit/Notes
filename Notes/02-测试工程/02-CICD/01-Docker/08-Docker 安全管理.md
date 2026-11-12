### Docker 如何实现隔离？

Docker 主要依赖 Linux 内核提供的机制。

|技术|主要作用|安全意义|
|---|---|---|
|Namespace|隔离进程、网络、挂载点等资源视图|限制容器能看到的资源|
|cgroups|限制 CPU、内存等资源使用|防止资源滥用|
|Capabilities|拆分传统 root 权限|限制容器进程的特权操作|
|Seccomp|过滤系统调用|减少可调用的内核接口|
|AppArmor / SELinux|强制访问控制|限制进程访问文件及其他资源|
**Namespace** 解决的是「我能看到什么」
**cgroups** 解决的是「我能使用多少资源」
**Capabilities** 解决的是「我能执行哪些特权操作」。

# Root 用户与非 Root 用户
容器应该尽量使用非 Root 用户，在 dockerfile里面可以指定用户
```dockerfile
xxxx

# 后续容器进程以 fastapi 用户身份运行
RUN groupadd --system fastapi && useradd --system --gid fastapi --create-home fastapi

# 复制文件时直接指定文件所有者
COPY --chown=fastapi:fastapi app/ .

# 5. 复制应用代码
# COPY app/ .

# 6. 声明容器服务端口
EXPOSE 8000

# 切换到app用户，避免使用root用户运行应用
USER fastapi

# 7. 容器启动命令
ENTRYPOINT ["uvicorn"]

CMD ["main:app", "--host", "0.0.0.0", "--port", "8000"]
```

# Linux Capabilities
传统 Linux 权限体系中，root 拥有大量特权操作能力，而普通用户通常不具备这些能力。

Capabilities 将 root 的特权拆分成多个独立的权限。
例如：

|Capability|权限作用|
|---|---|
|CAP_CHOWN|修改文件所有者|
|CAP_NET_BIND_SERVICE|绑定 1024 以下的端口|
|CAP_NET_RAW|使用 RAW 和 PACKET Socket|
|CAP_NET_ADMIN|管理网络接口、路由等|
|CAP_SYS_ADMIN|涉及大量系统管理操作|
|CAP_DAC_OVERRIDE|绕过常规文件读写权限检查|
Docker 可以通过 `cap-drop` 删除容器不需要的能力，也可以通过 `cap-add` 添加特定能力。
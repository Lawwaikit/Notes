[参考资料：# Docker Dockerfile](https://www.runoob.com/docker/docker-dockerfile.html)

dockerfile 是一份“镜像构建脚本”，Docker 会 **从上到下逐行执行**，每条指令都会生成一个新的 **镜像层 (Layer)**。

**顺序很重要**：

1. FROM → 定义基础镜像
2. WORKDIR → 设置工作目录
3. COPY / ADD → 拷贝文件
4. RUN → 构建阶段命令（安装依赖等）
5. ENV → 定义环境变量
6. EXPOSE → 声明端口
7. CMD / ENTRYPOINT → 容器启动命令

| 指令       | 用法                                           | 说明              | 注意点                            |
| ---------- | ---------------------------------------------- | ----------------- | ------------------------------ |
| FROM       | `FROM python:3.10-slim`                        | 指定基础镜像            | 必须在第一行                         |
| WORKDIR    | `WORKDIR /app`                                 | 设置容器工作目录，类似 `cd`  | 后续 COPY、RUN 默认在此目录执行           |
| COPY       | `COPY . .`                                     | 拷贝宿主机文件到容器        | 每次改动都会生成新层，尽量先 COPY 不常改的文件     |
| ADD        | `ADD file.tar.gz /app`                         | 功能类似 COPY，但可以自动解压 | 用途少，推荐 COPY                    |
| RUN        | `RUN apt-get update && apt-get install -y vim` | 构建镜像时执行命令         | 每条 RUN 都生成新层，合并多条命令可减少层数       |
| ENV        | `ENV MESSAGE="Hello"`                          | 定义环境变量            | 可被 CMD 或程序读取                   |
| EXPOSE     | `EXPOSE 5000`                                  | 声明端口              | 仅文档用途，不自动映射端口                  |
| CMD        | `CMD ["python", "app.py"]`                     | 容器启动默认命令          | 最后一条 CMD 生效，可被 docker run 命令覆盖 |
| ENTRYPOINT | `ENTRYPOINT ["python"]`                        | 固定容器入口            | 配合 CMD 可传参数，增强灵活性              |

## 3️⃣ 实践技巧与优化

1. **缓存优化**
    - Docker 会缓存每一层，如果上一层没变，下次构建会复用缓存。
    - 优化顺序：先 COPY 依赖文件 → RUN 安装依赖 → 再 COPY 源码
    ```
    COPY requirements.txt . 
    RUN pip install --no-cache-dir -r requirements.txt 
    COPY . .
    ```
    - 这样修改源代码不会重新安装依赖，加快构建速度。  
2. **减少镜像体积**
    - 使用轻量镜像，如 `python:3.10-slim`
    - 合并 RUN 命令，删除缓存：
    ```
    RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
    ```
3. **热更新开发**
  - 使用 Volume 挂载代码目录
  - CMD 中启动程序时开启调试模式（Flask `debug=True`）
4. **版本迭代**
    - 每次改动生成新镜像打 tag，如 `mypythonapp:1.0 → mypythonapp:2.0`
    - 方便回滚和管理多版本容器


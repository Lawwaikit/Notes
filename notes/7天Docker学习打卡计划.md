## 🗓️ 7 天 Docker 学习打卡计划

### **Day 1：Docker 基础命令**

目标：熟悉镜像和容器的管理

- 安装 Docker 并验证
    
    ```bash
    docker --version
    docker run hello-world
    ```
    
- 学习查看镜像和容器
    
    ```bash
    docker images
    docker ps
    docker ps -a
    ```
    
- 启动、停止、删除容器
    
    ```bash
    docker run -d --name test-nginx -p 8080:80 nginx
    docker stop test-nginx
    docker rm test-nginx
    ```
    
- 实操：运行一个 Nginx 或 Python 容器，访问页面
    

---

### **Day 2：Dockerfile 入门**

目标：学会写 Dockerfile 并构建镜像

- 复习 Dockerfile 核心指令：`FROM`、`WORKDIR`、`COPY`、`RUN`、`CMD`
    
- 实操：
    
    1. 写一个简单 `app.py` 输出一句话
        
    2. 编写 Dockerfile
        
    3. 构建镜像：
        
        ```bash
        docker build -t mypythonapp:1.0 .
        ```
        
    4. 运行容器并验证输出
        
- 练习修改 `app.py` 内容，重新 build 并运行
    

---

### **Day 3：Volume & 数据持久化**

目标：理解挂载宿主机目录和 Volume

- Bind Mount：
    
    ```bash
    docker run -v $(pwd):/app -p 8080:5000 mypythonapp:1.0
    ```
    
- Docker Volume：
    
    ```bash
    docker volume create myvolume
    docker run -v myvolume:/data mypythonapp:1.0
    ```
    
- 实操：
    
    - 修改代码在宿主机目录，刷新容器页面验证热更新
        
    - 创建 Volume 存储日志或数据库文件，停止容器后数据仍存在
        

---

### **Day 4：环境变量 & 配置**

目标：掌握传递环境变量控制容器行为

- Docker run 设置环境变量：
    
    ```bash
    docker run -e MESSAGE="Hello Docker v2" -p 8080:5000 mypythonapp:2.0
    ```
    
- Dockerfile ENV 指令：
    
    ```dockerfile
    ENV MESSAGE="Default Message"
    ```
    
- 实操：
    
    - 修改 Flask 或 Python 程序读取环境变量并返回
        
    - 测试不同 MESSAGE 值运行效果
        

---

### **Day 5：网络与端口映射**

目标：理解容器网络和端口映射

- 端口映射：
    
    ```bash
    docker run -p 8080:5000 mypythonapp:2.0
    ```
    
- 容器间通信：
    
    - 创建自定义网络
        
        ```bash
        docker network create mynet
        docker run -d --name web --network mynet mypythonapp:2.0
        docker run -d --name redis --network mynet redis:7
        ```
        
- 实操：
    
    - Flask 容器连接 Redis，测试能否读写
        

---

### **Day 6：镜像优化**

目标：学会减少镜像体积，加快构建

- 使用轻量镜像：`python:3.10-slim`
    
- 利用缓存层优化 Dockerfile：
    
    ```dockerfile
    COPY requirements.txt .
    RUN pip install -r requirements.txt
    COPY . .
    ```
    
- 多阶段构建（可选）：
    
    - Build Stage 安装依赖
        
    - Production Stage 只保留最终程序
        
- 实操：优化你的 Flask Dockerfile，构建并运行，比较镜像体积
    

---

### **Day 7：迭代更新 & 调试技巧**

目标：掌握代码更新、热重载和容器调试

- 修改代码 → 重新 build → run 新容器：
    
    ```bash
    docker build -t mypythonapp:3.0 .
    docker run -d -p 8080:5000 mypythonapp:3.0
    ```
    
- 调试技巧：
    
    - 查看容器状态：`docker inspect <container_id>`
        
    - 临时进入容器：`docker exec -it <container_id> bash`
        
    - 查看资源：`docker stats`
        
- 实操：
    
    - 挂载 Volume + debug 模式实现 Flask 热更新
        
    - 测试容器内调试命令
        

---

完成这个 7 天计划，你就能：

- 熟练管理镜像和容器
    
- 写 Dockerfile 并打包程序
    
- 挂载 Volume 热更新代码
    
- 使用环境变量和网络配置
    
- 优化镜像体积
    
- 迭代更新和调试容器
    

---

如果你愿意，我可以帮你 **在每一天加上具体操作步骤 + 验证输出**，保证打卡时每一步都有结果，学习效果最大化。

你希望我帮你加吗？
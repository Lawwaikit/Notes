## docker image 镜像操作
```bash
docker images # 查看镜像

docker image ls #查看镜像，等同于docker images

docker iamge rm <image_id>  # 删除镜像
docker rmi <image_id>  # 删除镜像

#其他操作查看 docker image COMMAND --help
```

## docker container 容器操作
```bash
docker ps  # 列出正在运行的容器
docker ps -a  # 列出所有容器

docker start <container>  # 启动容器
docker stop <container>  # 停止容器
docker restart <container>  # 重启容器
docker rm <container>  # 删除容器

docker logs <container>  # 查看容器日志

docker exec -it <container> bash # 进入容器命令行


```
### 启动容器命令详解
```bash
docker run -d     \
  --name flask-demo
  -p 8081:5000    \
  -v /root/flask_docker_demo:/app  \
  
  -e MESSAGE="Hello, Docker Nginx"    \

  --memory=256m \
  --cpus=0.5 \

  nginx:stable
```

| 参数           | 作用               |
| ------------ | ---------------- |
| -d           | 后台运行             |
| --name       | 指定容器名称           |
| -e           | 注入环境变量           |
| -p 8080:80   | 宿主机 8080 映射容器 80 |
| nginx:stable | 使用的镜像            |
## docker volume 卷管理
```bash
docker volume ls      # List volumes 

docker volume create  # Create a volume 

docker volume inspect # Display detailed information on one or more volumes 

docker volume prune   # Remove unused local volumes 
docker volume rm      # Remove one or more volumes
```

## docker network 网络管理
```bash
docker network ls

docket network create <name>

docket network inspect <name_or_id>

# 创建网络并将现有容器接入, app和db是容器
docker network create app-net
docker network connect app-net app
docker network connect app-net db

docker network connect <network> <container>
docker network disconnect <network> <container>

docker network prune # Remove all unused networks
docker network rm 
```

## docker排障常用
```bash

# 资源使用 CPU / 内存 / 网络 / I/O
docker stats <name_or_id> 
# 查看容器事件
docker events --filter container=xxx  
# 查看端口映射
docker port <name_or_id> 
# 查看容器详细信息配置
docker inspect <name_or_id>  
# 查看日志
docker logs <name_or_id> 
# 查看实时日志
docker logs -f <name_or_id> 
# 进入容器命令行
docker exec -it <name_or_id> sh 
#创建新的交互式容器(排障)
docker run -it <image> bash 

# 查看所有正在运行的容器的实时资源使用情况
docker stats

```

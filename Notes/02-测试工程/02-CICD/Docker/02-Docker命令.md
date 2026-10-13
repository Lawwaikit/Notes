## docker image 镜像操作
```bash
docker images # 查看镜像

docker image ls #查看镜像，等同于docker images

docker iamge rm <image_id>  # 删除镜像
docker rmi <image_id>  # 删除镜像

#其他操作查看 docker image COMMAND --help
```

## 容器操作
```bash
docker ps  # 列出正在运行的容器
docker ps -a  # 列出所有容器

docker start <con_id>  # 启动容器
docker stop <con_id>  # 停止容器
docker restart <con_id>  # 重启容器
docker rm <con_id>  # 删除容器

docker logs <con_id>  # 查看容器日志

docker exec -it <container> bash # 进入容器命令行

#创建容器
docker run -d     \
  -p 8081:5000    \
  -v /root/flask_docker_demo:/app  \
  -e MESSAGE="Hello, Docker v3"    \
  --name flask-demo  flask-demo:1.4
```

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
docker stats  # 实时查看 CPU / 内存 / 网络 / I/O

docker events --filter container=xxx  # 查看事件

docker port <name_or_id> # 查看端口映射

docker inspect <name_or_id>  # 查看容器配置

docker logs <name_or_id> # 查看日志

docker exec -it <name_or_id> sh # 进入容器命令行

docker run -it <image> bash #创建新的交互式容器(排障)

# 查看正在运行的容器的实时资源使用情况
docker stats

# 查看指定容器的资源使用情况
docker stats app
```

compose命令
```bash
docker compose ls # 查看compose项目
```
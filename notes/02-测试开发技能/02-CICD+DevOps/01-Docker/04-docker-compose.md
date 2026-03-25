Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。


```bash
# 构建并启动
docker-compose up --build -d

# 查看日志
docker-compose logs -f flask

# 扩容 Flask 到 3 个副本
docker-compose up -d --scale flask=3
```

## yml文件
以下是一个yml文件的编写
```yml
# 开发环境配置

services:
  # Flask应用
  flask:
    build: ./flask-app
    container_name: flask-dev
    ports:
      - "5000:5000"
    env_file:
      - .env.dev
    volumes:
      - ./flask-app:/app  # 热加载代码
      - ./logs/flask:/var/log/flask
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5
    depends_on:
      - mysql
      - redis
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # MySQL 数据库
  mysql:
    image: mysql:8.0
    container_name: mysql-dev
    env_file:
      - .env.dev
    ports:
      - "3306:3306"
    volumes:
      - mysql_data_dev:/var/lib/mysql
      - ./mysql-init/dev-init.sql:/docker-entrypoint-initdb.d/dev-init.sql # 初始化 SQL
      - ./logs/mysql:/var/log/mysql
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis 缓存
  redis:
    image: redis:7.2
    container_name: redis-dev
    ports:
      - "6379:6379"
    volumes:
      - redis_data_dev:/data
    restart: always
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  mysql_data_dev:
  redis_data_dev:

```
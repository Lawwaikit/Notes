Docker Secrets 用于管理敏感信息，例如数据库密码、API Token、TLS 私钥等。

它的核心思想是：**不再把敏感信息直接作为普通环境变量传递，而是以文件形式提供给服务。**

用普通环境变量的方式，攻击者可以通过docker inspect的方式读取密码`
`docker inspect test-platform-mysql`

配置compose，让服务使用secret
```yml
services:
  fastapi:
    container_name: test-platform-fastapi
    ports:
      - "8000"
    secrets:
      # 声明哪些容器可以使用对应 Secret
      - mysql_password
        
secrets:
  mysql_password:
    # 定义 Secret 文件来源
    file: ./secrets/mysql_password.txt

```

服务内部使用
```python
from pathlib import Path
MYSQL_PASSWORD_FILE = Path("/run/secrets/mysql_password")
pwd = MYSQL_PASSWORD_FILE.read_text().strip()
```

注意权限：
宿主机文件的权限传入容器内部，如果容器的用户权限不够，会导致服务读取失败

当前 FastAPI  容器内用户是 UID 999、GID 999。
宿主机 Secret 文件原本属于 UID 1000、GID 1000
因此使用 `chmod 600` 时容器无法读取。

最终通过调整文件所属组和权限解决：

```bash
sudo chgrp 999 secrets/mysql_password.txt
sudo chgrp 999 secrets/mysql_root_password.txt

sudo chmod 640 secrets/mysql_password.txt
sudo chmod 640 secrets/mysql_root_password.txt
```
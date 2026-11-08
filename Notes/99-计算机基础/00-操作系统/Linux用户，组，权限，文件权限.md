
## 一、Linux 用户与用户组基础

### 用户（User）

每个能登录/运行进程的身份。系统靠 **UID** 识别，`/etc/passwd` 存基本信息：

```bash
用户名:密码占位:UID:GID:描述:家目录:登录shell
lawwaikit:x:1000:1000::/home/lawwaikit:/bin/bash
```


### 用户组（Group）

一组用户的集合，方便批量授权。系统靠 **GID** 识别，`/etc/group` 存信息。

- **主组（primary group）**：用户创建文件时，文件默认属于这个组，记在 `/etc/passwd` 第 4 列。
- **附加组（supplementary groups）**：用户额外加入的组，用 `groups` 或 `id` 查看。

### 权限判断的核心逻辑

当你访问一个文件时，内核按这个顺序判断你属于哪一类：

1. **你是文件属主（owner）吗？** → 用"**属主权限**"那一组。
2. **否则，你属于文件属组（group）吗？** → 用"**属组权限**"那一组。
3. **否则（other）** → 用"**其他人权限**"那一组。

## 二、`ll` 输出的每一列

以你贴的第一行为例：

```text
drwxrwxr-x 7 lawwaikit lawwaikit 4096   Nov  4 13:56 ./
│└──┬───┘  │ └───┬───┘ └───┬───┘ └──┬─┘ └────┬─────┘ └┬┘
│   │      │     │          │        │        │        │
│   │      │     │          │        │        │        └─ 文件名
│   │      │     │          │        │        └─ 修改时间
│   │      │     │          │        └─ 大小（字节）
│   │      │     │          └─ 属组名
│   │      │     └─ 属主名
│   │      └─ 硬链接数（目录则是子目录数+2）
│   └─ 权限位（10 个字符）
└─ 文件类型（1 个字符）
```


逐列说明：

|列|示例|含义|
|---|---|---|
|1|`d`|文件类型|
|2|`rwxrwxr-x`|权限位，分 3 组×3 位|
|3|`7`|硬链接数（对目录=子目录数+2）|
|4|`lawwaikit`|**属主**（owner）|
|5|`lawwaikit`|**属组**（group）|
|6|`4096`|大小（字节；目录通常显示块大小 4096）|
|7|`Nov 4 13:56`|最后修改时间（mtime）|
|8|`./`|文件名|

### 第 1 列：文件类型

| 字符  | 类型     |
| --- | ------ |
| `-` | 普通文件   |
| `d` | 目录     |
| `l` | 符号链接   |
| `c` | 字符设备   |
| `b` | 块设备    |
| `s` | socket |
| `p` | 命名管道   |

### 第 2 列：权限位（9 位 = 3 组）

```text
rwx  rwx   r-x
属主  属组  其他人
```

- `r` = 读（4），`w` = 写（2），`x` = 执行（1）
- 对**目录**：`r`=能列目录，`w`=能在目录内创建/删除文件，`x`=能进入目录（cd）
- 对**文件**：`x`=可执行


用数字表示：`rwx=7, rw-=6, r-x=5, r--=4`，所以 `rwxrwxr-x` = `775`。

### 第 4、5 列：属主和属组

你的文件属主和属组都是 `lawwaikit`，说明这个目录是 `lawwaikit` 用户创建的，且它的主组也叫 `lawwaikit`（这就是之前说的 UID=GID 同名组的约定）。

### 第 3 列：硬链接数

- 普通文件：通常 `1`（你贴的文件都是 1）
- 目录：至少 `2`（`.` 和 `..` 各算一个），加上每个子目录各贡献一个。所以 `drwxrwxr-x 7` 表示这个目录下有 **5 个子目录**（7 - 2 = 5），和你看到 fastapi、mysql、nginx、redis、secrets 五个子目录吻合。

---

## 三、权限数字与符号对照

|数字|符号|含义|
|---|---|---|
|7|rwx|读+写+执行|
|6|rw-|读+写|
|5|r-x|读+执行|
|4|r--|只读|
|0|---|无|

你的文件模式速查：

|文件|权限|数字|说明|
|---|---|---|---|
|`./`|drwxrwxr-x|775|属主/属组可读写执行，其他人可读+进入|
|`../`|drwxr-x---|750|其他人完全无权|
|`.env` 等普通文件|-rw-rw-r--|664|属主/属组可读写，其他人只读|
|子目录|drwxrwxr-x|775|同上|

---

## 四、结合你的 Docker 场景

之前你提到容器内 `fastapi`/`mysql` 用户都是 UID 999。把这些文件挂进容器后：

- 容器内进程若以 UID **非** `lawwaikit`（比如 999）运行，它既不是属主也不是属组 → 落到"其他人"权限。
    
- 对 `775` 目录：其他人有 `r-x`，能读能进，但**不能写**（没有 `w`）——所以容器往里写文件会 Permission denied。
    
- 对 `664` 文件：其他人只读，不能改。
    

这就是为什么很多 compose 配置里要 `user: "1000:1000"` 或 `chown` 目录，让容器进程的 UID 匹配宿主机文件属主，否则写不进去。

---

## 五、常用命令

bash

id                          # 看自己的 UID/GID/附加组
ls -l / ll -a               # 看权限
chmod 664 file              # 数字改权限
chmod g+w file              # 符号改权限
chown user:group file       # 改属主属组
chgrp group file            # 改属组
umask                       # 看新建文件的默认权限掩码
getfacl file                # 看 ACL（更细的权限）

新建文件默认权限 = `666 - umask`，目录 = `777 - umask`。常见 umask `022` → 文件 `644`、目录 `755`；你这里是 `664/775`，说明 umask 可能是 `002`（同组可写），常见于需要团队协作的环境。

---

# `cat /etc/passwd` 逐行解读
## 一、每行格式

```text
用户名:密码占位:UID:GID:描述(gecos):家目录:登录shell
比如
lawwaikit:x:1000:1000::/home/lawwaikit:/bin/bash
```

|字段|示例|含义|
|---|---|---|
|1 用户名|`lawwaikit`|登录名|
|2 密码占位|`x`|密码实际存在 `/etc/shadow`（`x` 表示去那查）|
|3 UID|`1000`|用户数字 ID|
|4 GID|`1000`|**主组** GID|
|5 描述|空 / `root`|GECOS 字段，全名/备注|
|6 家目录|`/home/lawwaikit`|登录后的初始目录|
|7 登录 shell|`/bin/bash`|登录后启动的程序|

**第 7 字段决定账户能否登录**：`/usr/sbin/nologin` 或 `/bin/false` 表示禁止交互登录（只能被服务以特定身份调用，不能 ssh/login 进来）。

---
## 二、按类别分组看

### 1. 超级用户

```text
root:x:0:0:root:/root:/bin/bash
```
UID=0、GID=0、家目录 `/root`、能登录。所有权限检查的终极后门。

### 2. 传统 Unix 系统账户（UID 1–10 等小号）

```
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
```

这些是 Unix 早期留下的标准服务账户，现在多数已不实际使用，但为兼容保留。注意几个细节：

- `sync` 的 GID 是 65534（nogroup），shell 是 `/bin/sync`——登录即执行 sync 然后退出。
- `games`、`man` 的 GID 与 UID 不同（`games` UID 5 / GID 60，`man` UID 6 / GID 12），印证之前说的 **UID 和 GID 是两个独立命名空间**，不必相等。

### 3. Web / 应用服务账户（UID 33–42）

```
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
```

- `www-data`：Nginx/Apache 的运行身份，家目录 `/var/www`。你的 nginx 容器内很可能就是它或 root。
- `_apt`：APT 下载包时降权用的账户，家目录 `/nonexistent`，GID 65534。

### 4. "无身份"账户

```
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

UID=65534、GID=65534，约定俗成的"最低权限"身份，NFS、沙箱、匿名服务常用。家目录 `/nonexistent`，不能登录。

### 5. 系统服务账户（UID 900+ 动态分配）

```
systemd-network:x:998:998:...:/:/usr/sbin/nologin
dhcpcd:x:996:996:...:/usr/lib/dhcpcd:/bin/false
messagebus:x:995:995:...:/nonexistent:/usr/sbin/nologin
syslog:x:100:101:...:/nonexistent:/usr/sbin/nologin
systemd-resolve:x:989:989:...:/:/usr/sbin/nologin
_chrony:x:988:988:...:/var/lib/chrony:/usr/sbin/nologin
landscape:x:101:103:...:/var/lib/landscape:/usr/sbin/nologin
polkitd:x:987:987:...:/:/usr/sbin/nologin
sshd:x:985:65534:...:/run/sshd:/usr/sbin/nologin
```

对应 `/etc/group` 里的同名组。特点：

- 家目录多为 `/` 或 `/nonexistent`。
- shell 为 `nologin`/`false`，不能交互登录。
- **注意 UID 与 GID 不一定相同**：`syslog` UID=100 但 GID=101；`landscape` UID=101 但 GID=103。再次说明 UID/GID 独立。
- `sshd` 的 GID 是 65534（nogroup），这是 Debian/Ubuntu 的常见配置，sshd 降权后不需要额外组权限。
### 6. 你的账户

```
lawwaikit:x:1000:1000::/home/lawwaikit:/bin/bash
```

- UID=1000，**主组 GID=1000**（即 `lawwaikit` 组）。
- GECOS 为空（两个冒号之间什么都没写）。
- 家目录 `/home/lawwaikit`，能登录（`/bin/bash`）。
- 结合 `/etc/group`，你还额外属于 `adm,sudo,docker,dialout,...` 等附加组——那些**不在 passwd 里**，只在 group 文件第 4 列。

---
# `cat /etc/group` 逐行解读
## 一、每行格式

```text
组名:密码占位:GID:组成员列表

root:x:0:
adm:x:4:syslog,lawwaikit
lawwaikit:x:1000:
docker:x:986:lawwaikit
```

|字段|示例|含义|
|---|---|---|
|1 组名|`adm`|组的名字|
|2 密码占位|`x`|组密码（基本不用，`x` 表示密码在 `/etc/gshadow`）|
|3 GID|`4`|组的数字 ID|
|4 成员列表|`syslog,lawwaikit`|**附加组**成员（注意：主组用户不列在这里）|

**关键点**：第 4 列只列出"把该组当作附加组"的用户。用户的主组只记在 `/etc/passwd`，不会出现在这里。比如 `lawwaikit:x:1000:` 后面是空的，但 `lawwaikit` 用户的主组就是它——因为主组关系在 `/etc/passwd` 里写明了。

# 把 passwd 和 group 对照起来

两个文件的分工：

|       | `/etc/passwd`         | `/etc/group`       |
| ----- | --------------------- | ------------------ |
| 回答    | 用户是谁、UID、**主组**、shell | 组有哪些、GID、**附加组**成员 |
| 主组关系  | ✅ 记在第 4 列             | ❌ 不重复记             |
| 附加组关系 | ❌ 不记                  | ✅ 记在第 4 列          |

所以看 `lawwaikit` 的完整身份要两个文件都看：
- passwd → `lawwaikit:x:1000:1000:...` → 主组 = GID 1000 = `lawwaikit` 组
- group → 搜 `lawwaikit` 出现在 `adm:4`、`sudo:27`、`docker:986` 等行的第 4 列 → 这些是附加组

用 `id lawwaikit` 一条命令就能合并显示。

---
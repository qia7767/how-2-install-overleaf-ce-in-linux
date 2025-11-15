# Docker 安装 Overleaf CE 教程

 <p align="center"><strong><span style="color:#ff5722">Author: Ro</span></strong></p>

##   教程说明


###  目标与读者定位

  - **目标**：在一台 Linux 服务器上用 **Docker** 部署 **Overleaf CE**，并开启临时公网访问。
  - **读者**：对 Linux 有基础认知的用户。

### 文中符号与路径约定

  - **项目路径**：`${HOME}/docker/overleaf`（请确保当前用户的 `$HOME` 正确）

  - **服务端口**：Overleaf 宿主端口 **7643/tcp** 暴露到容器 **80/tcp** 

  - **容器名**：`Overleaf`、`Overleaf-DB`（Mongo）、`Overleaf-REDIS`（Redis）

  - **Compose** 文件所在目录：`${HOME}/docker/overleaf`

> 注意：Docker Compose **不会自动展开 `~`**，请使用 `${HOME}`。

------

##  前置条件

### 必备软件

  - **Docker 引擎**（建议 20.10+）
  - **Docker Compose v2**（`docker compose version` 可用）
  - **curl/wget** 与基本编译工具非必需，但建议安装。

  快速自检：

  ```bash
  docker version
  docker compose version
  ```

### 端口规划

  - **内部端口（容器内/宿主暴露）**：
    - Overleaf：`7643/tcp → 80/tcp`（宿主对外暴露 7643）
    - MongoDB：`27017/tcp`（**不建议**对外暴露）
    - Redis：`6379/tcp`（**不建议**对外暴露）
  - **外网访问**（三选一）：
    1. 反向代理（Nginx/Traefik）→ `80/443`
    2. Cloudflare Tunnel（无需开放 80/443）
    3. 直接映射 `7643`（临时或内网用）

### 域名与证书选项

  - **公网域名 + 证书**：生产首选。反代到 `127.0.0.1:7643`，用 **Let’s Encrypt** 自动签发。

  - **内网自签**：内网使用可接受；需信任根证书。

  - **Cloudflare Tunnel**：快且省事；适合零公网/防火墙严格场景。

    > 隧道临时域名会变动，生产建议 **命名 Tunnel + 自有域名**。

------

## Overleaf CE 架构速览

### 核心组件

  - **Overleaf(ShareLaTeX)**：Web 应用（Node.js），提供项目管理、在线编辑、编译触发等。
  - **MongoDB**：存储用户、项目信息、元数据。
  - **Redis**：会话缓存、队列等（提升并发与实时协作稳定度）。

### 数据流与依赖关系

  1. 浏览器访问 Overleaf（HTTP/WS）
  2. Overleaf 与 **Mongo** 读写元数据
  3. Overleaf 与 **Redis** 进行会话/队列交互
  4. Overleaf 宿主卷 **`/var/lib/overleaf`** 存放项目文件、上传内容、编译缓存与日志

### 持久化与关键目录

  - **项目文件与上传**：`/var/lib/overleaf`  存放 Overleaf 项目源码、历史版本以及用户上传的文件，是最核心的数据目录。
  - **Mongo 数据**：`/data/db`  存放 MongoDB 的数据文件，记录项目元数据、用户信息等结构化数据。
  - **系统字体**：`/usr/share/fonts`  提供 XeTeX / LuaTeX 渲染时可用的系统字体，可通过挂载方式加入额外字体。
  - **TeX Live**：`/usr/local/texlive`  存放 TeX Live 程序与宏包。

  > **原则**：所有“会变化/需保留”的内容都应挂载到宿主机目录，避免容器重建导致数据丢失。

------

## 安装 Docker（各发行版）

> 目标：安装 Docker Engine + Compose v2（`docker compose`）。
>  安装后做 3 件事：**加入用户组**、**设置开机自启**、**运行 Hello** 验证。

### Ubuntu 22.04+/24.04 & Debian 12

```bash
# 卸旧
sudo apt remove -y docker docker-engine docker.io containerd runc || true
# 依赖与 keyring
sudo apt update && sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo $ID)/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/$(. /etc/os-release; echo $ID) \
$(. /etc/os-release; echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# 安装
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# 组与服务
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
# 验证
docker --version
docker compose version
docker run --rm hello-world
```

### Fedora 39+/40+

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
docker --version
docker compose version
docker run --rm hello-world
```

### RHEL / AlmaLinux / Rocky 8–9

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
docker --version
docker compose version
docker run --rm hello-world
```

### Arch Linux

```bash
sudo pacman -Syu --noconfirm
sudo pacman -S --noconfirm docker docker-compose
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
docker --version
docker compose version || docker-compose --version
docker run --rm hello-world
```

### openSUSE Leap / Tumbleweed

```bash
sudo zypper refresh
sudo zypper install -y docker docker-compose
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
docker --version
docker compose version || docker-compose --version
docker run --rm hello-world
```

**常见问题**

- 权限不足：重新登录或执行 `newgrp docker`。
- 受限网络：设置代理或使用镜像源。
- SELinux（RHEL 系）挂载：卷挂载加 `:Z` 或预先 `chcon`。

------

### Docker 常用语法

#### 镜像

```bash
# 拉取
docker pull mongo:6
# 查看
docker images
# 改 tag
docker tag images/image:oldtag images/image:newtag
# 删除（确保未被使用）
docker rmi IMAGE_ID_OR_NAME
```

#### 容器

```bash
# 一次性运行
docker run --rm alpine:latest uname -a
# 守护进程、映射端口、命名
docker run -d --name my-redis -p 6379:6379 redis:7
# 启停与状态
docker ps
docker ps -a
docker stop my-redis && docker start my-redis
docker restart my-redis
# 日志与进入
docker logs -f my-redis
docker exec -it my-redis sh
# 停掉所有正在运行的容器
docker stop $(docker ps -q)
# 删除所有容器（包括 Exited 的）
docker rm $(docker ps -aq)
```

#### 卷与网络

```bash
docker volume create overleaf_data
docker volume ls
docker network create overleaf_net
docker network ls
```

#### 复制与导入导出

```bash
docker cp ./localfile.txt my-redis:/tmp/localfile.txt
docker cp my-redis:/tmp/localfile.txt ./localfile.txt
docker export my-redis -o my-redis.tar
docker import my-redis.tar my-redis:exported
```

#### 清理

```bash
docker system prune -af           # 清理未用资源（谨慎）
docker image prune -f             # 仅清理悬空镜像
```

#### Compose

如使用的是本教程的目录结构，配置文件位于：`${HOME}/docker/overleaf/docker-compose.yml`。请先进入该目录：

```bash
cd "${HOME}/docker/overleaf"
```

在包含 `docker-compose.yml` 的目录中：

- **后台启动（或更新）所有服务容器**

```bash
docker compose up -d
```

若镜像不存在，会自动从远程仓库拉取；

若 `docker-compose.yml` 配置有变（端口、环境变量、卷等），会按新配置重建对应容器；

`-d` 表示后台运行，不会占用当前终端。

- **查看当前 compose 项目中各服务的运行状态、端口等**

```bash
docker compose ps
```
列出当前项目下所有服务容器的名称、状态以及端口映射，方便检查服务是否正常运行。

- **持续查看所有服务的实时日志**

```bash
docker compose logs -f
```

持续输出当前 compose 项目中所有服务的日志，方便排查问题；按 **Ctrl + C** 退出日志界面，容器本身不会停止运行。

- **停止本项目的所有容器**

```bash
docker compose stop
```

停止当前 compose 项目中的所有运行中容器，但**不会删除容器与网络**；容器状态从「运行中」变为「已停止」，之后可通过 `docker compose start` 再次启动。

- **启动本项目中已停止的容器**

```bash
docker compose start
```

启动当前 compose 项目中已存在、处于「已停止」状态的容器，不会新建容器、也不会应用对 `docker-compose.yml` 的新改动。

- **停止并删除本项目的容器与网络**

```bash
docker compose down
```

停止当前 compose 项目中的所有容器，并**删除对应容器以及所属网络**（不删除镜像，也不删除挂载到宿主机的目录 / 卷）。

- **重启本项目的所有服务容器**

```bash
docker compose restart
```

对当前 Compose 项目中所有服务执行「先停止再启动」；适用于服务出现异常、或修改了挂载目录内的配置文件后，需要通过重启使其生效的场景。

> 注意：如果修改了 `docker-compose.yml` 本身，`start` / `restart` 不会应用新配置，建议使用：
>
> ```bash
> docker compose up -d
> ```
>
> Compose 会自动对比配置，只重建有变动的服务容器。

- **手动从远程仓库拉取镜像（更新为最新版本）**

```bash
docker compose pull
```

根据当前 `docker-compose.yml` 中声明的镜像名称与标签，从远程仓库拉取最新版镜像，适用于升级前先预拉镜像。


## 拉取镜像

### 需要的镜像

- `sharelatex/sharelatex:latest`
- `mongo:6.0`
- `redis:7-alpine`

### 直接从官方仓库拉取（能通就用它）

```bash
docker pull sharelatex/sharelatex:latest
docker pull mongo:6.0
docker pull redis:7-alpine
```

### 官方受限时的镜像源（拉完记得 retag）

- **sharelatex**

>目的：从镜像站拉取  `sharelatex/sharelatex:latest`，随后本地 retag 为 `sharelatex/sharelatex:latest`。
> 下面 **A/B**任选其一执行。

```bash
# =============================
# A / B 选一
# =============================

### 方案 A：unsee（推荐）

docker pull docker-0.unsee.tech/sharelatex/sharelatex:latest 
docker tag  docker-0.unsee.tech/sharelatex/sharelatex:latest  sharelatex/sharelatex:latest
docker rmi  docker-0.unsee.tech/sharelatex/sharelatex:latest  #可选，删除旧镜像

### --- B：hlmirror ---
docker pull docker.hlmirror.com/sharelatex/sharelatex:latest 
docker tag  docker.hlmirror.com/sharelatex/sharelatex:latest  sharelatex/sharelatex:latest 
docker rmi  docker.hlmirror.com/sharelatex/sharelatex:latest  #可选，删除旧镜像
```

>- `docker-compose.yml` 里请保持：`image: sharelatex/sharelatex:latest`。
>- 若后续想更新到新的 latest，只需**重复上述流程**（拉最新 → 覆盖性 retag 到 `latest`），Compose 仍指向同一固定名。

- **Mongo 与 Redis（DaoCloud 版）**

 ```bash
 # 下载mongo 6.0
 docker pull docker.m.daocloud.io/library/mongo:6.0
 docker tag  docker.m.daocloud.io/library/mongo:6.0  mongo:6.0
 docker rmi  docker.m.daocloud.io/library/mongo:6.0  #可选，删除旧镜像
 # 下载redis:7-alpine
 docker pull docker.m.daocloud.io/library/redis:7-alpine
 docker tag  docker.m.daocloud.io/library/redis:7-alpine  redis:7-alpine
 docker rmi  docker.m.daocloud.io/library/redis:7-alpine  #可选，删除旧镜像
 ```

>- `rmi` 只是清理中间源镜像，节省磁盘；不影响已改名的本地镜像使用。
>- 改完标签后，Compose 里写 `mongo:6.0`、`redis:7(-alpine)` 就能离线复用。

### 验证镜像是否就绪

```bash
docker images |  grep -E 'sharelatex|mongo|redis'
```

## 创建overleaf目录

- **目录结构**

```bash
${HOME}/docker/overleaf/
├─ data/      # Overleaf 数据目录 (/var/lib/overleaf)
├─ db/        # MongoDB 数据 (/data/db)
├─ fonts/     # 系统字体，会挂到 /usr/share/fonts
├─ texlive/   # TeX Live 根目录，会挂到 /usr/local/texlive
└─ redis/     # Redis 持久化 (/data) — 仅当启用 AOF/RDB 时才会产生文件
```

- **一键创建**

```bash
# 1) 统一根目录
export OVERLEAF_ROOT="${HOME}/docker/overleaf"

# 2) 创建目录
mkdir -p "${OVERLEAF_ROOT}"/{data,db,fonts,texlive,redis}

# 3) 基本权限（容器内常用 1000:1000 运行，先给到）
sudo chown -R 1000:1000 "${OVERLEAF_ROOT}"
sudo chmod -R 775 "${OVERLEAF_ROOT}"

# 4)（可选）查看结果
tree -L 1 "${OVERLEAF_ROOT}" || ls -al "${OVERLEAF_ROOT}"
```

> 说明：ShareLaTeX 容器里主要进程通常不是 root 运行；用 `1000:1000` 能避免权限报错。

## 部署overleaf ce

### 创建 compose文件

初次部署时，**只挂数据目录，不挂 TeX Live / 字体**，避免空目录覆盖镜像自带环境。

在终端执行下面的命令，创建 `${HOME}/docker/overleaf/docker-compose.yml` 配置文件：

```yaml
cat > "${HOME}/docker/overleaf/docker-compose.yml" << 'EOF'

services:
  mongo:
    image: mongo:6.0                 # 本地镜像
    container_name: Overleaf-DB
    restart: no
    command: ["--replSet","overleaf","--bind_ip_all"]
    volumes:
      - ${HOME}/docker/overleaf/db:/data/db:rw
    healthcheck:
      test: ["CMD-SHELL","echo 'db.runCommand({ping:1}).ok' | mongosh --quiet mongodb://localhost:27017/admin || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 18
      start_period: 120s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  mongoinit:
    image: mongo:6.0                 # 本地镜像
    container_name: mongo-init
    restart: no
    depends_on:
      mongo:
        condition: service_healthy
    entrypoint:
      - mongosh
      - --host
      - mongo:27017
      - --eval
      - >
        try{
          rs.initiate({ _id: "overleaf", members: [ { _id: 0, host: "mongo:27017" } ] });
        }catch(e){ print("rs.initiate may already be done:", e); }

  redis:
    image: redis:7-alpine            # 本地镜像
    container_name: Overleaf-REDIS
    restart: no
    command: ["redis-server","--save","","--appendonly","no"]
    volumes:
      - ${HOME}/docker/overleaf/redis:/data:rw
    healthcheck:
      test: ["CMD","redis-cli","ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options: { max-size: "5m", max-file: "2" }

  overleaf:
    image: sharelatex/sharelatex:latest   # 本地镜像（若你用 latest 已重打标签）
    container_name: Overleaf
    restart: "on-failure:5"
    depends_on:
      mongo:
        condition: service_healthy
      mongoinit:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    ports:
      - "7643:80"                      # 反代时指向 127.0.0.1:7643
    volumes:
      - ${HOME}/docker/overleaf/data:/var/lib/overleaf:rw
      # ↓↓ 先不要挂，等复制完再启用 ↓↓
      # - ${HOME}/docker/overleaf/texlive:/usr/local/texlive:rw
      # - ${HOME}/docker/overleaf/fonts:/usr/share/fonts:rw   # 字体持久化（稍后开启）
    environment:
      OVERLEAF_APP_NAME: "Overleaf Community Edition"
      OVERLEAF_MONGO_URL: "mongodb://mongo:27017/sharelatex?replicaSet=overleaf"
      OVERLEAF_REDIS_HOST: "redis"
      ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file"
      ENABLE_CONVERSIONS: "true"
      EMAIL_CONFIRMATION_DISABLED: "true"   # 暂无 SMTP 时务必开启
      OVERLEAF_ALLOW_EMAIL_SIGNUP: "true"   #
      OVERLEAF_ALLOW_ANONYMOUS_READ_AND_WRITE_SHARING: "true"    # 开放匿名分享
      OVERLEAF_CONTACT_EMAIL: "admin@example.com"
      OVERLEAF_SITE_URL: "http://localhost:7643"
      OVERLEAF_NAV_TITLE: "Overleaf on Home"
      OVERLEAF_PROXY_LEARN: "true"
      OVERLEAF_MAX_UPLOAD_SIZE: "150MB"
      TEXMFVAR: "/var/lib/sharelatex/tmp/texmf-var"
      TZ: "Asia/Macau"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:80/ || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
            
EOF
```

### 启动部署

在宿主机上执行：

```bash
cd ${HOME}/docker/overleaf
docker compose -f docker-compose.yml up -d
# 或：docker compose up -d
```

在后台启动三个服务：

- `Overleaf-DB`（Mongo）
- `Overleaf-REDIS`（Redis）
- `Overleaf`（Overleaf Web）

### 停止并清理 Overleaf 容器

下面的命令**仅作示例**，理解整个「关停 → 删除容器 → 清空数据」的流程。

- **正常关闭 compose 项目（推荐的标准做法）**

这一步会停止并删除当前 compose 项目中的所有容器和网络，但**不会删除镜像，也不会动挂载目录里的数据**：

```bash
cd "${HOME}/docker/overleaf"
docker-compose down
```
- **强制删除指定容器**


如果之前做过各种测试、手动 docker run 过容器，存在一些残留容器，也可以点名强制删除：

```bash
docker rm -f Overleaf Overleaf-DB Overleaf-REDIS mongo-init
```

- **删除挂载的持久化目录**

如果你确认这次部署只是测试，**不再需要里面的项目 / 用户 / 数据库等任何数据**，可以手动删除挂载目录，实现“完全重置”：

```bash
export OVERLEAF_ROOT="${HOME}/docker/overleaf"
sudo rm -rf "${OVERLEAF_ROOT}"/{data,db,fonts,texlive,redis}
```

### 查看容器与查看健康状态

- **总览所有容器状态**

``` bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

> `docker ps`：列出当前正在运行的容器
>
> `--format ...`：只显示**容器名称 / 状态 / 使用镜像**，以表格形式输出，方便快速查看：
>
> - `NAMES`：容器名（例如 `Overleaf`、`Overleaf-DB`、`Overleaf-REDIS`）
> - `STATUS`：运行状态（Up / Exited / healthy / unhealthy 等）
> - `IMAGE`：所用镜像版本（例如 `sharelatex/sharelatex:latest`）
>

- **查看关键容器日志**

当某个服务 unhealthy 或无法访问时，可以跟日志排查问题：

```bash
docker logs -f Overleaf-DB    # Mongo
docker logs -f mongo-init     # 副本集初始化结果
docker logs -f Overleaf-REDIS # Redis
docker logs -f Overleaf       # Overleaf
```

> `docker logs -f 容器名`：
>
> - 显示该容器的日志输出，
> - `-f` 表示持续跟随（类似 `tail -f`），按 **Ctrl + C** 退出查看日志，容器本身不会停止。
>
> 一般排错优先看：
>
> - `Overleaf`：应用是否能连上 Mongo / Redis，有无配置错误
> - `Overleaf-DB` + `mongo-init`：副本集是否初始化成功
> - `Overleaf-REDIS`：Redis 是否正常响应 `PING`（健康检查）

### 通用环境变量

> 从 **Overleaf CE/Server Pro 5.0.1** 起，环境变量前缀由 `SHARELATEX_*` **更名**为 `OVERLEAF_*`。
>
> 如果使用 **4.x 或更早版本**，请把相应变量改回 `SHARELATEX_*`（例如用 `SHARELATEX_SITE_URL` 而不是 `OVERLEAF_SITE_URL`）。

- `OVERLEAF_SITE_URL`： Overleaf 实例对外可访问的地址。用于生成公开链接以及 WebSocket 连接。
- `OVERLEAF_ADMIN_EMAIL`：站点管理员的联系邮箱（用户可通过该邮箱联系你）。
- `OVERLEAF_APP_NAME`：**应用显示名称**（默认：`Overleaf (Community Edition)`）。
- `OVERLEAF_MONGO_URL`：Mongo 数据库的连接 URL。
- `OVERLEAF_REDIS_HOST` 与 `REDIS_HOST`：Redis 主机名。**两者都需要设置**（见发行说明）。
- `OVERLEAF_REDIS_PORT` 与 `REDIS_PORT`：Redis 端口。**两者都需要设置**（见发行说明）。
- `OVERLEAF_REDIS_PASS` 与 `REDIS_PASSWORD`：连接 Redis 的密码（如适用）。
- `OVERLEAF_NAV_TITLE`：浏览器标签页标题。
- `OVERLEAF_SESSION_SECRET`：用于签名会话/令牌的随机字符串。若做负载均衡，多个实例必须使用**相同**的值；单实例可不手动设置。
- `OVERLEAF_COOKIE_SESSION_LENGTH`：覆盖默认会话 Cookie 过期时间（默认 5 天）。单位毫秒，例如 1 小时：`COOKIE_SESSION_LENGTH=3600000`。（自 Server Pro 4.2）
- `OVERLEAF_BEHIND_PROXY`：若运行在 Nginx/Apache 等反向代理之后，设为 `true`，以正确识别来源 IP。
- `OVERLEAF_SECURE_COOKIE`：设为非零启用 “Secure Cookie”。**仅在反代层已启用 SSL** 时使用。
- `OVERLEAF_RESTRICT_INVITES_TO_EXISTING_ACCOUNTS`：设为 `true` 时，仅允许邀请已存在账户对应邮箱的用户。
- `OVERLEAF_ALLOW_PUBLIC_ACCESS`：设为 `true` 允许未登录用户访问站点内容。默认 `false`（未登录用户会被统一重定向到登录页）。注意：这不会关闭认证或安全机制；若要让未登录用户查看公开项目，**必须**开启此项。
- `OVERLEAF_ALLOW_ANONYMOUS_READ_AND_WRITE_SHARING`：设为 `true` 允许通过新式“链接分享”的匿名访问者**查看并编辑**项目。
- `EMAIL_CONFIRMATION_DISABLED`：设为 `true` 时，不显示“请验证邮箱”的横幅提示。
- `ADDITIONAL_TEXT_EXTENSIONS`：配置额外可编辑的文本扩展名的字符串数组，例如：
   `ADDITIONAL_TEXT_EXTENSIONS='["abc","xyz"]'`
- `OVERLEAF_STATUS_PAGE_URL`：自定义状态页地址（自 3.4.0），例如 `status.example.com`。
- `OVERLEAF_FPH_INITIALIZE_NEW_PROJECTS`：设为 `false` 时，新建项目**不再**初始化“完整项目历史”（Full Project History）（自 3.5.0）。
- `OVERLEAF_FPH_DISPLAY_NEW_PROJECTS`：设为 `false` 时，新建项目默认**不显示**“完整项目历史”，而显示旧版历史视图（自 3.5.0）。
- `ENABLE_CRON_RESOURCE_DELETION`：设为 `true` 启用自动清理：删除的项目与用户在 90 天后被自动清除。
- `OVERLEAF_bCSP_ENABLED`：设为 `false` 以禁用内容安全策略（CSP）。
- `OVERLEAF_DISABLE_CHAT`：设为 `true` 禁用站内聊天功能（对所有用户生效）。

### 首次初始化与验证

- 验证 Mongo 副本集：

```bash
docker exec -it Overleaf-DB mongosh --quiet --eval 'rs.status().ok'
```

若输出为 1，表示副本集初始化成功

- Overleaf 页面：

浏览器访问 `http://<服务器IP>:7643/`（容器 80 暴露到宿主 7643）或者`127.0.0.1:7643/`，应出现 **Overleaf** 登录页。

## 进入容器并创建管理员

进入容器

```bash
docker exec -it Overleaf bash
```

执行以下命令，在容器内创建 **admin@example.com** 的管理员账号，几秒钟后，终端会输出一个激活链接。

```bash
cd /overleaf/services/web && node modules/server-ce-scripts/scripts/create-user --admin --email=admin@example.com
```
复制**生成的链接**，然后将其粘贴到新的浏览器选项卡中。输入**您自己的密码**，然后单击**激活**。

##   Overleaf 容器内的用户操作

进入容器后，首先切换到 Web 服务目录：

```bash
docker exec -it Overleaf bash
cd /overleaf/services/web
```

 - **创建 / 升级管理员账号**

```bash
node modules/server-ce-scripts/scripts/create-user --admin --email="admin@example.com"
```

为新的邮箱创建管理员，或将已有普通用户升级为管理员。

 -  **创建普通用户**


```bash
node modules/server-ce-scripts/scripts/create-user --email="user@example.com"
```

为新的邮箱创建普通用户，或将已有管理员调整为普通用户。

 -  **删除用户（CLI）**

```bash
node modules/server-ce-scripts/scripts/delete-user.mjs --email="user@example.com"
```

### 更多可用脚本

**所有可用的用户 / 管理脚本都位于容器内目录：**

 ```bash
/overleaf/services/web/modules/server-ce-scripts/scripts
 ```
可以进入该目录后使用 `ls` 查看所有脚本名称，并按需求调用对应的 `.js` / `.mjs` 脚本。

- `change-compile-timeout.mjs`：调整 Overleaf 的 LaTeX 编译超时阈值。


  ```bash
  node modules/server-ce-scripts/scripts/change-compile-timeout.mjs --seconds 180
  ```

- `create-user.mjs`：创建用户；支持管理员标志。


  ```bash
  # 管理员
  node modules/server-ce-scripts/scripts/create-user.mjs --admin --email "admin@example.com"
  # 普通用户
  node modules/server-ce-scripts/scripts/create-user.mjs --email "user@example.com"
  ```

- `rename-tag.mjs`：重命名系统内“标签”（项目/文件的标记）。

```bash
node modules/server-ce-scripts/scripts/rename-tag.mjs --from "OldTag" --to "NewTag"
```

- `check-mongodb.mjs`：检查 MongoDB 连接与健康状态。

```bash
node modules/server-ce-scripts/scripts/check-mongodb.mjs --uri "mongodb://mongo:27017/sharelatex"
```

- `delete-user.mjs`：删除用户（**谨慎使用**，更推荐先备份或采用软删除方案）。

```bash
node modules/server-ce-scripts/scripts/delete-user.mjs --email "user@example.com"
```

- `transfer-all-projects-to-user.mjs`：将某用户的**全部项目**迁移给另一个用户。

```bash
node modules/server-ce-scripts/scripts/transfer-all-projects-to-user.mjs \
  --from "old@example.com" --to "new@example.com"
```

- `check-redis.mjs`：检查 Redis 的连接状态与可用性。

```bash
node modules/server-ce-scripts/scripts/check-redis.mjs --host "redis" --port 6379
```

- `export-legacy-user-projects.mjs`：导出用户的“旧格式”项目数据（用于迁移或归档）。

```bash
node modules/server-ce-scripts/scripts/export-legacy-user-projects.mjs \
  --email "user@example.com" --out "/tmp/user_legacy_export.tgz"
```

- `upgrade-user-features.mjs`：升级用户的功能/配额（如历史版本、空间、权限套餐等）。

```bash
node modules/server-ce-scripts/scripts/upgrade-user-features.mjs \
  --email "user@example.com" --plan "pro"
```

- `check-texlive-images.mjs`：检查 TeX Live 镜像源的可用性与版本信息。

```bash
node modules/server-ce-scripts/scripts/check-texlive-images.mjs --mirror "ctan"
```

- `export-user-projects.mjs`：导出指定用户的全部项目（新格式或常规导出方式）。

```bash
node modules/server-ce-scripts/scripts/export-user-projects.mjs \
  --email "user@example.com" --out "/tmp/user_projects.tgz"
```

- `create-user.js`：与 `create-user.mjs` 功能相同的 CommonJS 版本（兼容旧环境）。

```bash
node modules/server-ce-scripts/scripts/create-user.js --email "user@example.com"
```

- `migrate-user-emails.mjs`：批量迁移/替换用户邮箱（例如更换邮箱域名）。

```bash
node modules/server-ce-scripts/scripts/migrate-user-emails.mjs \
  --from-domain "example.com" --to-domain "newexample.com"
```

------

使用前后的小建议：

- **先干跑/查看帮助**：`node <script> --help`；或 `sed -n '1,80p' <script>` 看头部注释与参数说明。
- **备份优先**：涉及删除/迁移前，先导出目标用户项目或快照数据库。

##  Mongo 容器内操作

进入 Mongo 容器并连接数据库

```bash
docker exec -it Overleaf-DB bash
mongosh sharelatex
```

第一行进入 `Overleaf-DB` 容器的交互式 Shell；
第二行启动 `mongosh`，并连接到 `sharelatex` 数据库（Overleaf 默认使用的库）。

- **提升/取消管理员**

```bash
// 提升为管理员
db.users.updateOne({email:"user1@example.com"}, {$set:{isAdmin:true}})

// 取消管理员
db.users.updateOne({email:"user1@example.com"}, {$set:{isAdmin:false}})
```

- **列表/检索用户**

```sh
// 列出关键信息
db.users.find({}, {email:1, isAdmin:1, deleted:1, createdAt:1}).sort({createdAt:1}).toArray()

// 按邮箱检索
db.users.find({email:"user1@example.com"}, {email:1, isAdmin:1, deleted:1}).toArray()
```

## 字体安装与管理

### 字体持久化

当确认 Overleaf 运行正常、容器内自带的 TeX Live 与系统字体可用后，可以将其拷贝到宿主机，实现持久化。

在**宿主机**执行：

```bash
# 1）准备两个持久化目录（之前已准备过）
mkdir -p "${HOME}/docker/overleaf/texlive"
mkdir -p "${HOME}/docker/overleaf/fonts"

# 2）从 Overleaf 容器拷贝 TeX Live 到宿主机
sudo docker cp Overleaf:/usr/local/texlive/. "${HOME}/docker/overleaf/texlive"

# 3）从 Overleaf 容器拷贝系统字体到宿主机
sudo docker cp Overleaf:/usr/share/fonts/. "${HOME}/docker/overleaf/fonts"
```

> 注意两个 `.`：
>  `.../texlive/.` 和 `.../fonts/.` 都是“复制目录内容”，避免出现 `/texlive/texlive` / `/fonts/fonts` 多套一层的情况。

简单检查拷贝结果：

```bash
ls "${HOME}/docker/overleaf/texlive"
ls "${HOME}/docker/overleaf/fonts"
```

若能看到年份目录、`tlpkg`，以及大量 `.ttf` / `.otf` 字体文件，说明拷贝成功。

**启用 TeX Live / 字体持久化并重新部署**

接下来，将刚才拷出的目录通过 Docker 挂载回容器，使后续对 TeX Live / 字体的修改都持久化到宿主机。

编辑 `${HOME}/docker/overleaf/docker-compose.yml` 中 `overleaf` 服务的 `volumes` 段，将 TeX Live 与字体挂载补上（在原有数据挂载基础上添加两行）：

```yaml
  overleaf:
    ...
    volumes:
      - ${HOME}/docker/overleaf/data:/var/lib/overleaf:rw
      - ${HOME}/docker/overleaf/texlive:/usr/local/texlive:rw
      - ${HOME}/docker/overleaf/fonts:/usr/share/fonts:rw
```

或者直接执行：

```yaml
cat > "${HOME}/docker/overleaf/docker-compose.yml" << 'EOF'

services:
  mongo:
    image: mongo:6.0                 # 本地镜像
    container_name: Overleaf-DB
    restart: no
    command: ["--replSet","overleaf","--bind_ip_all"]
    volumes:
      - ${HOME}/docker/overleaf/db:/data/db:rw
    healthcheck:
      test: ["CMD-SHELL","echo 'db.runCommand({ping:1}).ok' | mongosh --quiet mongodb://localhost:27017/admin || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 18
      start_period: 120s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  mongoinit:
    image: mongo:6.0                 # 本地镜像
    container_name: mongo-init
    restart: no
    depends_on:
      mongo:
        condition: service_healthy
    entrypoint:
      - mongosh
      - --host
      - mongo:27017
      - --eval
      - >
        try{
          rs.initiate({ _id: "overleaf", members: [ { _id: 0, host: "mongo:27017" } ] });
        }catch(e){ print("rs.initiate may already be done:", e); }

  redis:
    image: redis:7-alpine            # 本地镜像
    container_name: Overleaf-REDIS
    restart: no
    command: ["redis-server","--save","","--appendonly","no"]
    volumes:
      - ${HOME}/docker/overleaf/redis:/data:rw
    healthcheck:
      test: ["CMD","redis-cli","ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options: { max-size: "5m", max-file: "2" }

  overleaf:
    image: sharelatex/sharelatex:latest   # 本地镜像（若你用 latest 已重打标签）
    container_name: Overleaf
    restart: "on-failure:5"
    depends_on:
      mongo:
        condition: service_healthy
      mongoinit:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    ports:
      - "7643:80"                      # 反代时指向 127.0.0.1:7643
    volumes:
      - ${HOME}/docker/overleaf/data:/var/lib/overleaf:rw
      - ${HOME}/docker/overleaf/texlive:/usr/local/texlive:rw
      - ${HOME}/docker/overleaf/fonts:/usr/share/fonts:rw   
    environment:
      OVERLEAF_APP_NAME: "Overleaf Community Edition"
      OVERLEAF_MONGO_URL: "mongodb://mongo:27017/sharelatex?replicaSet=overleaf"
      OVERLEAF_REDIS_HOST: "redis"
      ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file"
      ENABLE_CONVERSIONS: "true"
      EMAIL_CONFIRMATION_DISABLED: "true"   # 暂无 SMTP 时务必开启
      OVERLEAF_ALLOW_EMAIL_SIGNUP: "true"   #
      OVERLEAF_ALLOW_ANONYMOUS_READ_AND_WRITE_SHARING: "true"    # 开放匿名分享
      OVERLEAF_CONTACT_EMAIL: "admin@example.com"
      OVERLEAF_SITE_URL: "http://localhost:7643"
      OVERLEAF_NAV_TITLE: "Overleaf on Home"
      OVERLEAF_PROXY_LEARN: "true"
      OVERLEAF_MAX_UPLOAD_SIZE: "150MB"
      TEXMFVAR: "/var/lib/sharelatex/tmp/texmf-var"
      TZ: "Asia/Macau"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:80/ || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
            
EOF
```

之后，在宿主机执行重新部署：

```bash
cd "${HOME}/docker/overleaf"
# 关停当前容器
docker compose down
# 按新配置重新启动
docker compose up -d
```

并在容器内验证 TeX Live 与字体是否正常：

```bash
docker exec -it Overleaf bash

which kpsewhich
tlmgr --version

fc-list | head
ls /usr/share/fonts | head
```

此时：

- `/usr/local/texlive` 与 `/usr/share/fonts` 已由宿主机目录提供（可持久化）；
- 在容器内运行 `tlmgr update`、安装额外宏包，或通过 `apt` / 手工复制安装字体，都会同步写入 `${HOME}/docker/overleaf/texlive` / `${HOME}/docker/overleaf/fonts`，实现持久化部署。

### 更新 tlmgr 

进入 Overleaf 容器：

```bash
docker exec -it Overleaf bash
```

在容器内执行：

```bash
# 可选：进入 TeX Live 目录（非必须）
cd /usr/local/texlive

# 升级 tlmgr 引导脚本
wget http://mirror.ctan.org/systems/texlive/tlnet/update-tlmgr-latest.sh
sh update-tlmgr-latest.sh -- --upgrade

# 切换清华镜像
tlmgr option repository https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/tlnet/

# 升级 tlmgr 本体与已安装包
tlmgr update --self --all
```

**安装常用字体 / 语言集合**

若你没有安装 `scheme-full`，可以额外装一些常用字体与中文相关集合：

```bash
# 常用字体集合（西文字体/数学字体）
tlmgr install collection-fontsrecommended

# 中文 / 日文 / 韩文语言支持（用于 CJK 文档）
tlmgr install collection-langchinese collection-langjapanese collection-langkorean
```

> - `collection-fontsrecommended`：TeX Live 推荐的一批通用字体/数学字体集合。
>- `collection-langchinese` 等：补充 CJK 相关宏包与字体支持，适合 XeLaTeX / LuaLaTeX / CJK 文档。

**若需要完整 TeX Live，执行：**

```bash
# 时间较长，请确保会话不中断（建议在 tmux / screen 中运行）
tlmgr install scheme-full
```

###  安装常用 CJK 系统字体

> 容器内 `/usr/share/fonts` 已挂载到宿主机 `${HOME}/docker/overleaf/fonts`，安装的字体会被持久化。

```bash
# 1）安装常用 CJK 字体
apt-get update && apt-get install -y fonts-noto-cjk fonts-noto-cjk-extra

# 2）刷新系统与 LuaTeX 字体缓存
fc-cache -f -v
luaotfload-tool -u
```

### A) 安装「最大化开源字体合集」

> 覆盖：Noto 全家桶（含 CJK / Emoji）、文泉驿、文鼎、IPA / VL Gothic、Nanum、DejaVu / Liberation / FreeFont、
>  以及泰文 / 高棉 / 老挝 / 缅文 / 埃塞俄比亚 / 阿拉伯 / 科学排版常用西文等。
>  如只需 CJK，可缩减为：`fonts-noto-cjk* + fonts-noto-color-emoji + fonts-wqy-* + fonts-arphic-* + ipaex + nanum`。

```bash
# 建议先确保容器有必要源（Ubuntu 基础一般已带 main/universe）
apt-get update

# 尽可能全的开源字体合集（不含微软核心字体）
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  fonts-noto fonts-noto-extra fonts-noto-unhinted \
  fonts-noto-cjk fonts-noto-cjk-extra fonts-noto-color-emoji \
  fonts-dejavu fonts-dejavu-extra fonts-liberation2 fonts-freefont-ttf \
  fonts-wqy-zenhei fonts-wqy-microhei \
  fonts-arphic-uming fonts-arphic-ukai \
  fonts-ipaexfont fonts-vlgothic \
  fonts-nanum fonts-nanum-coding \
  fonts-thai-tlwg fonts-khmeros fonts-lao \
  fonts-sil-abyssinica fonts-sil-padauk \
  fonts-kacst-one fonts-kacst \
  fonts-stix fonts-stix-two \
  fonts-lmodern \
  fonts-roboto fonts-roboto-unhinted

# 刷新系统与 LuaTeX 字体缓存
fc-cache -f -v
luaotfload-tool -u
```

**说明与选择：**

- `fonts-noto*`：几乎覆盖全球语言脚本；`fonts-noto-cjk*` 即思源黑/宋（Noto CJK）。
- `fonts-noto-color-emoji`：彩色 Emoji。
- `fonts-wqy-*`（文泉驿）、`fonts-arphic-*`（文鼎）：兼容老文档与中文界面的常见回退。
- `fonts-ipaexfont`/`fonts-vlgothic`：日文常用。
- `fonts-nanum*`：韩文。
- `fonts-stix*`/`fonts-lmodern`：理工/数理排版常见西文字体。
- 其余（Thai/Khmer/Lao/Abyssinica/Padauk/KACST 等）用于东南亚、非洲、阿拉伯语系等文档的 fallback。

------

### B) 安装微软核心字体

> 需要启用 `multiverse` 并接受 EULA；仅当你确实需要 Word/PowerPoint 常见字体兼容性时再装。若镜像最小化，可能需 `software-properties-common`。

```bash
apt-get update && apt-get install -y --no-install-recommends \
  software-properties-common debconf-utils

add-apt-repository -y multiverse
apt-get update

# 预先接受 EULA，避免交互阻塞
echo "ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true" \
  | debconf-set-selections

DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  ttf-mscorefonts-installer

# 刷新缓存
fc-cache -f -v
luaotfload-tool -u
```

### 放入自带字体

将 Windows 的 `C:\Windows\Fonts` 拷到容器的 `/usr/share/fonts/mc-font` 目录后，在容器内执行：

```bash
chown -R root:root /usr/share/fonts/mc-font
find /usr/share/fonts/mc-font -type d -exec chmod 755 {} +
find /usr/share/fonts/mc-font -type f -exec chmod 644 {} +
# 刷新缓存
fc-cache -f -v
luaotfload-tool -u
```

### 验证字体是否可用

```bash
# 检查 Noto CJK 包是否安装
dpkg -l | grep -E 'fonts-noto-cjk($|-extra)'

# 检查系统是否识别到 Noto CJK 字体
fc-list | grep -i 'noto .* cjk' | head
# 看到 “Noto Sans CJK SC / Noto Serif CJK SC” 等即正常
```

> 提示：
>
> - XeTeX 主要依赖 `fc-cache`。
> - LuaTeX 额外使用 `luaotfload-tool -u` 刷新索引。

### Fontconfig 字体查询速查

- **基础查询**

```bash
# 1) 列出：家族 / 样式 / 文件
fc-list : family style file | sort

# 2) 仅字体家族（去重）
fc-list : family | tr ',' '\n' | sed 's/^[[:space:]]*//' | sort -u

# 3) 名称模糊搜索（不区分大小写）
fc-list | grep -i "SimSun\|Songti\|Times New Roman\|Noto Sans CJK"

# 4) 查看某名字实际匹配（含回退链）
fc-match "Times New Roman" -v | sed -n '1,12p'
# 简版：
fc-match "Times New Roman"

# 5) 仅看中文（或指定语言）字体
fc-list :lang=zh family file | sort
# 也可试：:lang=zh-cn、:lang=ja、:lang=ko 等
```

- **目录扫描与缓存**

```bash
# 列出被识别到的字体文件路径（抽样）
fc-list : file | sort -u | head

# 查看/刷新缓存（也会显示扫描目录）
fc-cache -v | sed -n '1,80p'
```

------

### 清单 + 汇总字体报告

下面命令生成当前系统可用字体的**汇总报告**

```bash
mkdir -p /usr/share/fonts/_reports

# A) 家族名清单（去重）
fc-list : family \
| tr ',' '\n' | sed 's/^[[:space:]]*//' | sort -u \
> /usr/share/fonts/_reports/fonts_family.txt

# B) 详细清单（TSV：Family/Style/File）
fc-list -f '%{family}\t%{style}\t%{file}\n' | sort \
> /usr/share/fonts/_reports/fonts_detail.tsv
```

下面命令生成更友好的汇总与 LaTeX 调用名提示

```python
python3 - <<'PY'
import os, re, collections, sys

IN = "/usr/share/fonts/_reports/fonts_detail.tsv"
OUTDIR = os.path.dirname(IN)

if not os.path.isfile(IN):
    print(f"[ERR] 找不到 {IN}，请先生成 fonts_detail.tsv")
    sys.exit(1)

rows = []
with open(IN, 'r', encoding='utf-8', errors='ignore') as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        parts = line.split('\t')
        if len(parts) < 3:
            parts = re.split(r'\s{2,}', line)
        if len(parts) >= 3:
            fam, sty, path = parts[0].strip(), parts[1].strip(), parts[2].strip()
            rows.append((fam, sty, path))

def split_families(fam):
    return [x.strip() for x in fam.split(',') if x.strip()]

FamilyInfo = collections.defaultdict(lambda: {"styles": set(), "files": set(), "examples": []})

for fam_raw, style, path in rows:
    for fam in split_families(fam_raw):
        d = FamilyInfo[fam]
        d["styles"].add(style or "Regular")
        d["files"].add(path)
        if len(d["examples"]) < 3:
            d["examples"].append((style or "Regular", path))

def has_style(styles, key):
    P = key.lower()
    return any(P in (s or "").lower() for s in styles)

# 1) 汇总 TSV
sum_path = os.path.join(OUTDIR, "fonts_summary.tsv")
with open(sum_path, "w", encoding="utf-8") as fo:
    fo.write("Family\tStyles\tN_Styles\tExample_File\n")
    for fam in sorted(FamilyInfo):
        styles = sorted(FamilyInfo[fam]["styles"])
        n = len(styles)
        example_file = FamilyInfo[fam]["examples"][0][1] if FamilyInfo[fam]["examples"] else ""
        fo.write(f"{fam}\t{', '.join(styles)}\t{n}\t{example_file}\n")
print(f"[OK] 写出 {sum_path}")

# 2) LaTeX 调用名建议
latex_path = os.path.join(OUTDIR, "fonts_latex_hints.txt")
with open(latex_path, "w", encoding="utf-8") as fo:
    fo.write("# XeLaTeX/LuaLaTeX 可尝试的调用名（按 family + 常见样式）\n")
    common = ["Regular","Light","Book","Medium","Semibold","Bold","Italic","Bold Italic"]
    for fam in sorted(FamilyInfo):
        styles = FamilyInfo[fam]["styles"]
        names = []
        for s in common:
            if has_style(styles, s):
                if s.lower() == "regular":
                    names.append(f"{fam}")            # 仅 family 名
                names.append(f"{fam} {s}".strip())    # family + 样式
        # 去重保序
        seen = set(); names2 = []
        for x in names:
            if x not in seen:
                names2.append(x); seen.add(x)
        fo.write(f"\n[{fam}]\n  " + ", ".join(names2) + "\n")
print(f"[OK] 写出 {latex_path}")
PY
```

**生成结果与位置：**

- 家族清单：`/usr/share/fonts/_reports/fonts_family.txt`
- 详细清单：`/usr/share/fonts/_reports/fonts_detail.tsv`
- 汇总表：`/usr/share/fonts/_reports/fonts_summary.tsv`
- LaTeX 名称提示：`/usr/share/fonts/_reports/fonts_latex_hints.txt`

------

## 公网访问 Overleaf（Cloudflare Quick Tunnel）

假设已经通过 Docker / Compose 部署好了 Overleaf，并在本机监听 **7643/tcp（HTTP）**。

---

### 前提检查

```bash
# 确认 Docker 正常工作
docker version

# 确认 Overleaf 已在本机 7643 端口监听
curl -I http://127.0.0.1:7643
# 看到 HTTP/1.1 200 / 302 等响应即可（不是 200 也没关系，只要有响应）
```

> 提示：
>
> - Overleaf 容器通常通过 `-p 7643:80` 暴露到宿主机。
> - 若你映射的端口不是 `7643`，请把下文中所有 `7643` 替换为你的实际端口。

### 镜像下载

**目标：**
 先从国内可用镜像源拉取 **cloudflared**，再将其打上官方标签 `cloudflare/cloudflared:latest`，这样后续命令就可以直接使用官方写法。

- **DaoCloud 源**

```bash
docker pull m.daocloud.io/docker.io/cloudflare/cloudflared:latest
docker tag  m.daocloud.io/docker.io/cloudflare/cloudflared:latest cloudflare/cloudflared:latest
# 可选：清理原镜像
docker rmi  m.daocloud.io/docker.io/cloudflare/cloudflared:latest

# 验证是否已有官方名
docker images | grep cloudflared
```

- **GHCR 源**

```bash
docker pull ghcr.io/cloudflare/cloudflared:latest
docker tag  ghcr.io/cloudflare/cloudflared:latest cloudflare/cloudflared:latest
```

- **dockerproxy 源**

```bash
docker pull dockerproxy.com/cloudflare/cloudflared:latest || \
docker pull dockerproxy.com/r/cloudflare/cloudflared:latest
  
# 成功哪个就 tag 哪个
docker tag dockerproxy.com/cloudflare/cloudflared:latest cloudflare/cloudflared:latest || true
docker tag dockerproxy.com/r/cloudflare/cloudflared:latest cloudflare/cloudflared:latest || true
```

------

### 运行 Cloudflare Quick Tunnel

**关键点：** 使用 **host 网络**，让容器内的 `127.0.0.1` 直接指向宿主机本机，从而访问本机 `7643` 端口。

```bash
docker run -d --name cftmp --restart unless-stopped \
  --network host \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate --url http://127.0.0.1:7643
```

参数说明：

- `--restart unless-stopped`：开机自动拉起，只要你没有手动停止。
- `--network host`：使用宿主机网络命名空间，避免“容器内 127.0.0.1 无法访问宿主机服务”的问题。

> 如果你**不使用** host 网络（不推荐），必须将 URL 改为宿主机的局域网 IP，例如：
>  `... --url http://192.168.x.x:7643`

------

### 获取公网地址并验证

查看 cloudflared 容器日志：

```bash
docker logs -f cftmp
```

几秒钟内会看到类似输出：

```bash
https://xxxxx.trycloudflare.com
```

该 URL 即为公网访问地址，浏览器访问该地址，能看到 Overleaf 登录页面即表示成功。

快速提取公网地址的示例命令：

```bash
CF_CONTAINER=cftmp sh -c '
  U=$(whoami); D="docker"; [ "$U" != "root" ] && D="sudo docker";
  URL=$($D logs "$CF_CONTAINER" 2>&1 | grep -Eo "https://[a-z0-9-]+\.trycloudflare\.com" | tail -1);
  echo "$URL"
'
```

> 小提示：
>
> - 新生成的 `trycloudflare.com` 链接有时需要 **10–60 秒** 才在所有边缘节点完全可用，如首次访问失败，可稍等片刻再试。
>
> - 日志中若出现类似：
>
>   ```bash
>   ERR Cannot determine default origin certificate path ...
>   ```
>
>   这与回源 HTTPS + mTLS 相关，你当前使用的是回源 HTTP，可直接忽略，不影响 Quick Tunnel 工作。

------

### 日常管理命令

```bash
# 查看运行状态 / 公网地址（日志中会显示 trycloudflare 域名）
docker logs -f cftmp

# 停止并删除 cloudflared 容器（当前“临时域名”将失效）
docker rm -f cftmp

# 需要再次启用时，重新执行前文 docker run 命令（通常会获得一个新的 trycloudflare 域名）
```

说明：

- `xxxxx.trycloudflare.com` 这个**临时域名**在容器持续运行期间一般保持不变；
- 删除容器或服务重建后，新的容器通常会分配到一个**新的** `trycloudflare.com` 域名；
- 这套方案非常适合作为 **临时公网访问 / 远程演示** 的简易方案，无需域名与证书配置。

------

### 配置 systemd 服务（可选）

如果你的系统使用 systemd（如 Ubuntu / Debian / CentOS 等），可以为 cloudflared 创建一个 systemd 服务，统一由系统管理。

创建文件：`/etc/systemd/system/cloudflared-quick.service`

```bash
[Unit]
Description=Cloudflare Quick Tunnel for Overleaf (port 7643)
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=always
RestartSec=3
ExecStartPre=-/usr/bin/docker rm -f cftmp
ExecStart=/usr/bin/docker run --name cftmp --network host \
  cloudflare/cloudflared:latest tunnel --no-autoupdate --url http://127.0.0.1:7643
ExecStop=/usr/bin/docker rm -f cftmp

[Install]
WantedBy=multi-user.target
```

加载并启用服务：

```bash
systemctl daemon-reload
systemctl enable --now cloudflared-quick.service
journalctl -u cloudflared-quick.service -f
```

这样就无需手动输入 `docker run`，开机会自动拉起 Quick Tunnel，日志也统一交给 systemd 管理。

# 附录1 Docker 开机启动与容器自动启动机制说明

## 查看 Docker 是否启用开机自启

系统使用 systemd 管理 Docker 服务，当 Docker 设置为 **开机自动启动** 时，系统一开机 Docker 就会运行。可通过以下命令查询：

```bash
systemctl is-enabled docker
```

返回 `enabled` ＝ 开机自启
返回 `disabled` ＝ 不会随开机启动

### 启用 Docker 开机自启

```bash
sudo systemctl enable docker
```

### 禁用 Docker 开机自启

```bash
sudo systemctl disable docker
```

### 手动启动 / 停止 Docker

```bash
sudo systemctl start docker
sudo systemctl stop docker
```

------

## 容器是否随 Docker 自动启动

当 Docker 服务启动后，容器是否自动启动 **完全取决于容器的 `restart` 策略**。

------

### 常见 `restart` 策略说明

`restart` 是写在 **docker-compose.yml 每个服务（service）下面的一个字段**，例如：

```yaml
services:
  overleaf:
    image: sharelatex/sharelatex:latest
    container_name: Overleaf
    restart: always      # ← 就写在这里
    ports:
      - "7643:80"
```

不同的取值会决定容器在 Docker 重启 / 系统重启 / 异常退出时的自动重启行为：

| restart 策略     | 行为描述                                                     |
| ---------------- | ------------------------------------------------------------ |
| `no`（默认）     | Docker 启动时不会自动运行容器                                |
| `on-failure[:N]` | 容器异常退出时自动重启；Docker 重启时通常也会拉起之前正在运行的容器 |
| `always`         | Docker 服务启动时总是自动启动容器                            |
| `unless-stopped` | 类似 always，但如果你手工 stop，则不会自动重启               |

### 推荐配置

- **希望 Docker 启动时 Overleaf 自动启动：**
   `restart: always`
- **希望 Docker 启动时容器不自动启动，需要手动运行：**
   `restart: "no"`

------

##  示例：容器的不同启动方式

### A）Docker 不随开机启动，容器自动启动（推荐）

适合日常个人服务器（HomeLab）。

1. 禁用 Docker 开机启动：

   ```bash
   sudo systemctl disable docker
   ```

2. 给容器设置：

   ```bash
   restart: always
   ```

3. 当你需要使用 Overleaf 时，只需：

   ```bash
   sudo systemctl start docker
   ```

所有容器会自动恢复运行。

------

### B）Docker 随开机启动，容器也自动启动

适合业务服务器、长期在线服务。

配置：

```bash
sudo systemctl enable docker
```

容器设置：

```bash
restart: always
```

系统开机 → Docker 服务启动 → 所有容器自动启动。

------

### C）Docker 启动后，容器也必须手动启动

适合完全手动控制容器的情况。

容器设置：

```bash
restart: no
```

即使 Docker 启动，容器也不会自动跑，必须：

```bash
docker compose up -d
# 或
docker start <container>
```

------

# 附录2 秒杀！一键部署overleaf CE

下面是一键部署脚本，一条龙完成拉镜像，部署，持久化字体，临时公网访问。

在执行前请先确认：**Docker / docker compose 已安装并可用**

```bash
#!/usr/bin/env bash
# ======================================================
# Overleaf Community Edition 一键部署脚本（单机版）
# - 拉取镜像：Overleaf / MongoDB / Redis / Cloudflared
# - 创建并持久化：项目数据、Mongo、TeX Live、字体、Redis
# - 首次启动后复制容器内 TeX Live / 字体到宿主机
# - 重新挂载持久化目录并按新配置启动
# - 可选：后台升级 TeX Live & 安装多语种字体
# - 可选：通过 Cloudflare Tunnel 暴露临时公网访问链接
#
# 适用环境：
# - 已安装 Docker、docker compose（插件或老版 docker-compose）
# - Linux（Debian/Ubuntu 系列优先），宿主机具备 sudo 权限
# ======================================================

set -e

# =========================================
# 0. 环境检查 & sudo 预热
# =========================================

# 0.1 检查 docker 是否存在
if ! command -v docker >/dev/null 2>&1; then
  echo "错误：未找到 docker 命令，请先安装 Docker 再运行本脚本。" >&2
  exit 1
fi

# 0.2 提示 docker compose 状态（仅提示，不强制退出）
if ! docker compose version >/dev/null 2>&1; then
  echo "提示：未检测到 'docker compose' 子命令。"
  echo "      如你使用的是旧版 'docker-compose'，请将脚本中的"
  echo "      'docker compose ...' 手工替换为 'docker-compose ...' 后再执行。"
fi

# 0.3 如果不是 root，预热 sudo
if [ "$(id -u)" -ne 0 ]; then
  echo "需要使用 sudo 执行部分操作（如 chown / chmod），正在预热 sudo..."
  sudo -v || { echo "获取 sudo 权限失败，请检查 sudo 配置。"; exit 1; }
fi

# 0.4 统一根目录：可提前在外部 export OVERLEAF_ROOT 覆盖
: "${OVERLEAF_ROOT:=${HOME}/docker/overleaf}"
echo "Overleaf 根目录：${OVERLEAF_ROOT}"
# =========================================
# 1. 拉取并重打标签：Overleaf / MongoDB / Redis 镜像
# =========================================

# 1.1 拉取 Overleaf（sharelatex）镜像，并重打标签为 sharelatex/sharelatex:latest
docker pull docker-0.unsee.tech/sharelatex/sharelatex:latest
docker tag  docker-0.unsee.tech/sharelatex/sharelatex:latest  sharelatex/sharelatex:latest

# 1.2 拉取 MongoDB 6.0 并重打标签为 mongo:6.0
docker pull docker.m.daocloud.io/library/mongo:6.0
docker tag  docker.m.daocloud.io/library/mongo:6.0  mongo:6.0

# 1.3 拉取 Redis 7(alpine) 并重打标签为 redis:7-alpine
docker pull docker.m.daocloud.io/library/redis:7-alpine
docker tag  docker.m.daocloud.io/library/redis:7-alpine  redis:7-alpine

# =========================================
# 2. 创建 Overleaf 根目录与子目录，并设置权限
# =========================================


# 2.1 创建数据目录：项目数据、数据库、字体、TeX Live、Redis
mkdir -p "${OVERLEAF_ROOT}"/{data,db,fonts,texlive,redis}

# 2.2 基本权限设置
# Overleaf 容器内通常以 UID:GID = 1000:1000 运行，这里先将宿主机目录拥有者设为该用户
sudo chown -R 1000:1000 "${OVERLEAF_ROOT}"
# 目录权限 775：拥有者和同组可读写执行，其他用户可读执行
sudo chmod -R 775 "${OVERLEAF_ROOT}"

# =========================================
# 3. 第一次生成 docker-compose.yml（不挂载 texlive / fonts）并启动
#    目的：先用容器自带的 TeX Live / 字体跑起来，再完整拷贝到宿主机
# =========================================
cat > "${HOME}/docker/overleaf/docker-compose.yml" << 'EOF'

services:
  mongo:
    image: mongo:6.0                 # 本地镜像
    container_name: Overleaf-DB
    restart: no
    command: ["--replSet","overleaf","--bind_ip_all"]
    volumes:
      - ${HOME}/docker/overleaf/db:/data/db:rw
    healthcheck:
      test: ["CMD-SHELL","echo 'db.runCommand({ping:1}).ok' | mongosh --quiet mongodb://localhost:27017/admin || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 18
      start_period: 120s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  mongoinit:
    image: mongo:6.0                 # 本地镜像
    container_name: mongo-init
    restart: no
    depends_on:
      mongo:
        condition: service_healthy
    entrypoint:
      - mongosh
      - --host
      - mongo:27017
      - --eval
      - >
        try{
          rs.initiate({ _id: "overleaf", members: [ { _id: 0, host: "mongo:27017" } ] });
        }catch(e){ print("rs.initiate may already be done:", e); }

  redis:
    image: redis:7-alpine            # 本地镜像
    container_name: Overleaf-REDIS
    restart: no
    command: ["redis-server","--save","","--appendonly","no"]
    volumes:
      - ${HOME}/docker/overleaf/redis:/data:rw
    healthcheck:
      test: ["CMD","redis-cli","ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options: { max-size: "5m", max-file: "2" }

  overleaf:
    image: sharelatex/sharelatex:latest   # 本地镜像（若你用 latest 已重打标签）
    container_name: Overleaf
    restart: "on-failure:5"
    depends_on:
      mongo:
        condition: service_healthy
      mongoinit:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    ports:
      - "7643:80"                      # 反代时指向 127.0.0.1:7643
    volumes:
      - ${HOME}/docker/overleaf/data:/var/lib/overleaf:rw
      # ↓↓ 先不要挂，等复制完再启用 ↓↓
      # - ${HOME}/docker/overleaf/texlive:/usr/local/texlive:rw
      # - ${HOME}/docker/overleaf/fonts:/usr/share/fonts:rw   # 字体持久化（稍后开启）
    environment:
      OVERLEAF_APP_NAME: "Overleaf Community Edition"
      OVERLEAF_MONGO_URL: "mongodb://mongo:27017/sharelatex?replicaSet=overleaf"
      OVERLEAF_REDIS_HOST: "redis"
      ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file"
      ENABLE_CONVERSIONS: "true"
      EMAIL_CONFIRMATION_DISABLED: "true"   # 暂无 SMTP 时务必开启
      OVERLEAF_ALLOW_EMAIL_SIGNUP: "true"   #
      OVERLEAF_ALLOW_ANONYMOUS_READ_AND_WRITE_SHARING: "true"    # 开放匿名分享
      OVERLEAF_CONTACT_EMAIL: "admin@example.com"
      OVERLEAF_SITE_URL: "http://localhost:7643"
      OVERLEAF_NAV_TITLE: "Overleaf on Home"
      OVERLEAF_PROXY_LEARN: "true"
      OVERLEAF_MAX_UPLOAD_SIZE: "150MB"
      TEXMFVAR: "/var/lib/sharelatex/tmp/texmf-var"
      TZ: "Asia/Macau"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:80/ || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
            
EOF

# 进入目录并启动（第一次）
cd "${OVERLEAF_ROOT}"
docker compose -f docker-compose.yml up -d
# 如使用旧版 docker-compose，可替换为：
# docker-compose -f docker-compose.yml up -d

# =========================================
# 4. 从 Overleaf 容器中拷贝 TeX Live / 字体到宿主机
# =========================================

# 4.1 拷贝 TeX Live 完整目录到宿主机（用于后续持久化与自定义包）
sudo docker cp Overleaf:/usr/local/texlive/. "${OVERLEAF_ROOT}/texlive"

# 4.2 拷贝系统字体目录到宿主机（用于自定义 / 持久化字体）
sudo docker cp Overleaf:/usr/share/fonts/. "${OVERLEAF_ROOT}/fonts"

# =========================================
# 5. 用挂载后的路径重新生成 docker-compose.yml，并按新配置重启
# =========================================

cat > "${HOME}/docker/overleaf/docker-compose.yml" << 'EOF'

services:
  mongo:
    image: mongo:6.0                 # 本地镜像
    container_name: Overleaf-DB
    restart: no
    command: ["--replSet","overleaf","--bind_ip_all"]
    volumes:
      - ${HOME}/docker/overleaf/db:/data/db:rw
    healthcheck:
      test: ["CMD-SHELL","echo 'db.runCommand({ping:1}).ok' | mongosh --quiet mongodb://localhost:27017/admin || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 18
      start_period: 120s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  mongoinit:
    image: mongo:6.0                 # 本地镜像
    container_name: mongo-init
    restart: no
    depends_on:
      mongo:
        condition: service_healthy
    entrypoint:
      - mongosh
      - --host
      - mongo:27017
      - --eval
      - >
        try{
          rs.initiate({ _id: "overleaf", members: [ { _id: 0, host: "mongo:27017" } ] });
        }catch(e){ print("rs.initiate may already be done:", e); }

  redis:
    image: redis:7-alpine            # 本地镜像
    container_name: Overleaf-REDIS
    restart: no
    command: ["redis-server","--save","","--appendonly","no"]
    volumes:
      - ${HOME}/docker/overleaf/redis:/data:rw
    healthcheck:
      test: ["CMD","redis-cli","ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options: { max-size: "5m", max-file: "2" }

  overleaf:
    image: sharelatex/sharelatex:latest   # 本地镜像（若你用 latest 已重打标签）
    container_name: Overleaf
    restart: "on-failure:5"
    depends_on:
      mongo:
        condition: service_healthy
      mongoinit:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    ports:
      - "7643:80"                      # 反代时指向 127.0.0.1:7643
    volumes:
      - ${HOME}/docker/overleaf/data:/var/lib/overleaf:rw
      - ${HOME}/docker/overleaf/texlive:/usr/local/texlive:rw
      - ${HOME}/docker/overleaf/fonts:/usr/share/fonts:rw   
    environment:
      OVERLEAF_APP_NAME: "Overleaf Community Edition"
      OVERLEAF_MONGO_URL: "mongodb://mongo:27017/sharelatex?replicaSet=overleaf"
      OVERLEAF_REDIS_HOST: "redis"
      ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file"
      ENABLE_CONVERSIONS: "true"
      EMAIL_CONFIRMATION_DISABLED: "true"   # 暂无 SMTP 时务必开启
      OVERLEAF_ALLOW_EMAIL_SIGNUP: "true"   #
      OVERLEAF_ALLOW_ANONYMOUS_READ_AND_WRITE_SHARING: "true"    # 开放匿名分享
      OVERLEAF_CONTACT_EMAIL: "admin@example.com"
      OVERLEAF_SITE_URL: "http://localhost:7643"
      OVERLEAF_NAV_TITLE: "Overleaf on Home"
      OVERLEAF_PROXY_LEARN: "true"
      OVERLEAF_MAX_UPLOAD_SIZE: "150MB"
      TEXMFVAR: "/var/lib/sharelatex/tmp/texmf-var"
      TZ: "Asia/Macau"
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:80/ || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
            
EOF

# 按新配置重启（此时已挂载 TeX Live / 字体）
docker compose -f docker-compose.yml up -d


# =========================================
# 6.（可选）后台升级 TeX Live 并安装字体（非交互）
#    命令会在 Overleaf 容器内后台执行，终端不会被长时间占用
# =========================================

docker exec -d Overleaf bash -lc '
set -e

# 1) 升级 tlmgr 并切换到清华镜像
cd /usr/local/texlive || true
wget http://mirror.ctan.org/systems/texlive/tlnet/update-tlmgr-latest.sh
sh update-tlmgr-latest.sh -- --upgrade
tlmgr option repository https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/tlnet/
tlmgr update --self --all

# 2) 在容器内部后台安装完整 TeX Live（scheme-full），不阻塞后续步骤
tlmgr install scheme-full &

# 3) 安装常用字体（开源字体 + 微软核心字体）
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  fonts-noto fonts-noto-extra fonts-noto-unhinted \
  fonts-noto-cjk fonts-noto-cjk-extra fonts-noto-color-emoji \
  fonts-dejavu fonts-dejavu-extra fonts-liberation2 fonts-freefont-ttf \
  fonts-wqy-zenhei fonts-wqy-microhei \
  fonts-arphic-uming fonts-arphic-ukai \
  fonts-ipaexfont fonts-vlgothic \
  fonts-nanum fonts-nanum-coding \
  fonts-thai-tlwg fonts-khmeros fonts-lao \
  fonts-sil-abyssinica fonts-sil-padauk \
  fonts-kacst-one fonts-kacst \
  fonts-stix fonts-stix-two \
  fonts-lmodern \
  fonts-roboto fonts-roboto-unhinted \
  software-properties-common debconf-utils

add-apt-repository -y multiverse
apt-get update
echo "ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  ttf-mscorefonts-installer

# 4) 刷新字体缓存
fc-cache -f -v
luaotfload-tool -u
'

# =========================================
# 7. 建立临时公网访问（Cloudflare Tunnel）
#    - 使用 cloudflared 暴露本地 7643 端口
#    - 自动输出一个临时的 trycloudflare 链接
# =========================================

# 7.1 拉取并重打标签（使用国内镜像源）
docker pull m.daocloud.io/docker.io/cloudflare/cloudflared:latest
docker tag  m.daocloud.io/docker.io/cloudflare/cloudflared:latest cloudflare/cloudflared:latest

# 7.2 启动临时 Cloudflare Tunnel 容器（指向本机 7643 端口）
docker run -d --name cftmp --restart unless-stopped \
  --network host \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate --url http://127.0.0.1:7643

# 7.3 从日志中自动提取临时公网访问链接
CF_CONTAINER=cftmp sh -c '
  U=$(whoami); D="docker"; [ "$U" != "root" ] && D="sudo docker";

  echo "等待 Cloudflare Tunnel 分配临时访问地址..."

  # 最多等 ~60 秒，每 2 秒检查一次日志
  for i in $(seq 1 30); do
    URL=$($D logs "$CF_CONTAINER" 2>&1 | grep -Eo "https://[a-z0-9-]+\.trycloudflare\.com" | tail -1)
    [ -n "$URL" ] && break
    sleep 2
  done

  if [ -n "$URL" ]; then
    echo
    echo "===== Overleaf 临时公网访问地址 ====="
    echo "$URL"
    echo "===================================="
  else
    echo "未在预期时间内从日志中获取到 trycloudflare 链接，请稍后手动执行："
    echo "  $D logs $CF_CONTAINER | grep -E \"https://[a-z0-9-]+\\.trycloudflare\\.com\""
    exit 1
  fi
'
```

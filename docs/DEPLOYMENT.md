# 📦 FoamV2 Docker 部署教程

[← 返回项目介绍](../README.md) · [🖼️ 查看功能截图](SCREENSHOTS.md) · [💬 Telegram 群组](https://t.me/FoamHub)

FoamV2 推荐使用 Docker Compose 部署。下面的配置会启动 Foam Web、Foam API、MySQL 和 Redis，适合在一台 Linux 服务器上快速搭建。

> FoamV2 为授权软件。部署完成后仍需获取有效授权，具体方式请查看官网或加入 Telegram 群组咨询。

## 1. 环境准备

- 一台 Linux 服务器，建议 2 核 CPU、4 GB 内存及以上
- Docker 24+ 与 Docker Compose v2
- 开放 Web 访问端口 `8081`
- 准备 TMDB API Token 与 API Key

确认 Docker 已安装：

```bash
docker --version
docker compose version
```

## 2. 创建部署目录

```bash
mkdir -p foam-v2
cd foam-v2
```

将项目提供的 `docker-compose.yml` 上传到这个目录。仓库中的原文件作为部署模板保留；实际部署时，请只修改服务器上的副本。

## 3. 核对 Compose 配置

当前 `docker-compose.yml` 的结构如下。不要直接照搬其中的示例密码、地址或 API 凭据，启动前请按代码块后的说明逐项替换。

```yaml
version: '3'
services:
  foam-api-v2:
    image: ciwei123321/foam-api-v2:latest
    privileged: true
    ports:
      - "8080:8080"
    volumes:
      - ./data:/data
      - /etc/hosts:/etc/hosts
    container_name: foam-api-v2
    restart: always
    environment:
      #db:3306 使用的是容器内部的端口 不是映射完的端口
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/foam-api-v2?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=GMT%2B8&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=78FRC#5BqnOk0ppk
      # 需要配置tmdb接口hosts
      - TMDB_APITOKEN=tmdb api token
      - TMDB_APIKEY=tmdb api key
      - TMDB_IMAGE_URL=https://image.tmdb.org/t/p/original
      - TZ=Asia/Shanghai
      # 代理地址
      - HTTP_PROXY_ENABLED=false
      - HTTP_PROXY=http://ip:port
      - HTTPS_PROXY=http://ip:port
      - NO_PROXY=172.17.0.1,127.0.0.1,localhost
      # 搜索接口地址 pansou地址
      - EMBY_HUB_SEARCH_URL=
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PWD=123456
      - REDIS_DB=0
      # 浏览器实际访问的前端 Origin；生产请改成你的域名或服务器 IP:端口
      - FOAM_CORS_ALLOWED_ORIGINS=你的http://ip:8081访问地址
    networks:
      - foam-network
    links:
      - db
      - redis
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.4.6
    container_name: mysql_container
    environment:
      MYSQL_ROOT_PASSWORD: 78FRC#5BqnOk0ppk
      MYSQL_DATABASE: foam-api-v2
      TZ: "Asia/Shanghai"
      LANG: en_US.UTF-8
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --group_concat_max_len=102400
    ports:
      - "3306:3306"
    volumes:
      - ./mysql-data:/var/lib/mysql
    restart: always
    networks:
      - foam-network

  foam-web:
    image: ciwei123321/foam-web:latest
    container_name: foam-web
    restart: always
    ports:
      - "8081:80"
    environment:
      API_BASE_URL: "http://foam-api-v2:8080"
      TZ: Asia/Shanghai
      IMAGE_URL: https://image.tmdb.org/t/p/
    networks:
      - foam-network
    links:
      - foam-api-v2
    depends_on:
      - foam-api-v2

  redis:
    image: redis:7.4
    container_name: redis_container
    restart: always
    command:
      - redis-server
      - --appendonly
      - "yes"
      - --save
      - "60"
      - "1"
      - --requirepass
      - "123456"
    volumes:
      - ./redis-data:/data
    networks:
      - foam-network

networks:
  foam-network:
```

### ✅ 必须替换

> 无论模板当前显示什么值，以下内容都应换成你自己的配置。密码必须使用新生成的强密码，不要继续使用模板值。

| YML 中的位置 | 替换成什么 | 为什么要替换 |
| --- | --- | --- |
| `SPRING_DATASOURCE_PASSWORD` | 你的 MySQL Root 密码 | Foam API 使用它连接 MySQL；必须与下面的 `MYSQL_ROOT_PASSWORD` 完全一致，否则 API 会因数据库认证失败而反复重启。 |
| `MYSQL_ROOT_PASSWORD` | 与上一项相同的 MySQL Root 密码 | 这是 MySQL 容器初始化 Root 用户时使用的密码。 |
| `TMDB_APITOKEN` | 你自己的 TMDB API Read Access Token | 用于 TMDB 媒体搜索、详情及元数据请求。 |
| `TMDB_APIKEY` | 你自己的 TMDB API Key | 部分 TMDB 请求仍需要 API Key，不能填写 Token 或随意字符串。 |
| `REDIS_PWD` | 你的 Redis 密码 | Foam API 使用它连接 Redis；必须与 Redis 的 `--requirepass` 密码一致。 |
| Redis `--requirepass` 后面的值 | 与上一项相同的 Redis 密码 | Redis 服务端使用这个值校验连接，两个密码不一致会导致缓存、登录状态等功能异常。 |
| `FOAM_CORS_ALLOWED_ORIGINS` | 浏览器实际访问 Foam 的 Origin | 用于放行 Web 跨域请求。填写协议、域名或 IP 及端口，例如 `http://192.168.1.10:8081` 或 `https://foam.example.com`，末尾不要加 `/`；多个地址用英文逗号分隔。 |

MySQL 与 Redis 密码需要各自保持两处一致，可以按下面的对应关系检查：

```text
SPRING_DATASOURCE_PASSWORD = MYSQL_ROOT_PASSWORD
REDIS_PWD = Redis --requirepass 后面的值
```

### ⚙️ 按需替换

| YML 中的位置 | 什么时候需要改 | 备注 |
| --- | --- | --- |
| `HTTP_PROXY_ENABLED` | 服务器访问 TMDB 需要代理时改为 `true` | 设为 `false` 时不会启用代理。 |
| `HTTP_PROXY` / `HTTPS_PROXY` | 启用代理时填写真实的 `http://代理IP:端口` | 不要保留无法访问的局域网代理地址。 |
| `NO_PROXY` | 启用代理且内部容器通信异常时 | 建议包含 `foam-api-v2,db,redis,127.0.0.1,localhost`，避免 Docker 内部请求绕到代理。 |
| `EMBY_HUB_SEARCH_URL` | 已部署 Pansou/搜索服务时 | 填写搜索服务地址；未部署就保持为空。 |
| `8080:8080` / `8081:80` / `3306:3306` | 主机端口冲突或需要调整外部访问端口时 | 只改冒号左侧的主机端口。Web 端口改变后，也要同步修改 CORS Origin。 |
| `./data`、`./mysql-data`、`./redis-data` | 希望把数据保存到其他磁盘时 | 修改冒号左侧的宿主机路径，冒号右侧的容器路径不要改。 |

### 🧭 通常不要替换

- `SPRING_DATASOURCE_URL` 中的 `db:3306` 是 MySQL 容器的服务名和内部端口，不是服务器公网 IP。
- `REDIS_HOST=redis` 是 Redis 的 Docker 服务名。
- `API_BASE_URL=http://foam-api-v2:8080` 是 Web 容器访问 API 的内部地址。
- `MYSQL_DATABASE=foam-api-v2`、`REDIS_PORT=6379`、`TMDB_IMAGE_URL` 一般保持原值。
- `/etc/hosts:/etc/hosts` 表示把宿主机 Hosts 挂载进 API 容器。TMDB 域名无法解析时，应按实际网络情况修改服务器的 `/etc/hosts`，不是替换这条容器路径。

当前模板没有向公网映射 Redis 端口；MySQL 映射了主机 `3306` 端口。如果不需要远程连接 MySQL，请在实际部署副本中移除该端口映射，或至少通过防火墙限制访问来源。

## 4. 启动服务

```bash
docker compose pull
docker compose up -d
docker compose ps
```

首次启动需要等待数据库初始化。可通过日志确认服务状态：

```bash
docker compose logs -f foam-api-v2 foam-web
```

服务正常后访问：

```text
http://YOUR_SERVER_IP:8081
```

按页面提示初始化管理员并登录，然后在后台完成授权、Emby 服务器、Telegram、TMDB 和通知渠道等配置。

## 5. 域名与 HTTPS

生产环境建议使用 Nginx、Caddy 或其他反向代理，将域名转发到 `http://127.0.0.1:8081`，并启用 HTTPS。

切换为域名后，记得在实际部署的 `docker-compose.yml` 中同步修改：

```yaml
- FOAM_CORS_ALLOWED_ORIGINS=https://foam.example.com
```

然后重新创建 API 容器使配置生效：

```bash
docker compose up -d
```

## 6. 更新与备份

更新到最新镜像：

```bash
docker compose pull
docker compose up -d
```

升级前建议备份以下目录：

- `data/`：Foam 数据与上传文件
- `mysql-data/`：MySQL 数据
- `redis-data/`：Redis 持久化数据
- `docker-compose.yml`：部署配置

设置请参考：[FoamV2官网](https://foamhub.cc.cd)

## 7. 常见问题

### 页面可以打开，但接口请求失败

检查 `FOAM_CORS_ALLOWED_ORIGINS` 是否与浏览器实际访问地址完全一致，并查看 API 日志：

```bash
docker compose logs --tail=200 foam-api-v2
```

### 海报、TMDB 搜索或图片无法加载

确认 TMDB Token/API Key 正确，并检查服务器是否可以访问 TMDB。需要代理时，请按自己的网络环境为 API 容器补充 HTTP/HTTPS 代理配置。

### 容器反复重启

```bash
docker compose ps
docker compose logs --tail=200 db redis foam-api-v2 foam-web
```

重点检查数据库密码是否一致、数据目录权限是否正常，以及服务器内存是否充足。

## 🔐 安全建议

- 使用高强度且互不相同的 MySQL、Redis 密码
- 不要将含有真实密码、授权信息、Bot Token 或 API Key 的 `docker-compose.yml` 提交到 GitHub
- 不要向公网暴露 MySQL、Redis 和内部 API 端口
- 正式使用时配置 HTTPS，并定期备份持久化目录

---

[← 返回项目介绍](../README.md) · [🖼️ 查看功能截图](SCREENSHOTS.md)

---
title: "Docker 核心知识点总结：从原理到实战"
description: "Docker 运行原理、核心概念、Dockerfile、Docker Compose、数据管理与网络 — 后端开发者的容器化知识地图"
pubDatetime: 2026-07-01T00:00:00.000Z
featured: true
tags:
  - Docker
  - 容器化
  - DevOps
  - 实战
---

后端开发绕不开 Docker。这篇文章把 Docker 的核心知识点从原理到实战梳理一遍，争取看完就能上手。

---

## 1. Docker 解决什么问题

一句话：**"我的电脑能跑，你的跑不了"。**

以前部署一个后端服务，要在服务器上装 JDK、装 Maven、装 MySQL、配环境变量、配端口……每台机器都来一遍。Docker 把应用和它依赖的一切打包成一个标准化的"集装箱"，在任何装了 Docker 的机器上都能一键跑起来。

**Docker vs 虚拟机**

| | 虚拟机 | Docker 容器 |
|---|---|---|
| 隔离方式 | Hypervisor + 完整 Guest OS | 共享宿主机内核 |
| 启动速度 | 分钟级 | 秒级 |
| 资源占用 | GB 级（每个 VM 跑一个完整系统） | MB 级（只包含应用和依赖） |
| 性能损耗 | 较大 | 接近原生 |

容器直接用宿主机的内核，不需要虚拟化一个完整的操作系统，所以更轻、更快。

---

## 2. 运行原理

Docker 不是虚拟机，它靠 Linux 内核的三大技术实现隔离：

### Namespace — 隔离

Linux 内核把系统资源隔离成独立的"空间"，每个容器以为自己是唯一的：

| Namespace | 隔离什么 |
|---|---|
| PID | 进程（容器里 PID 1 是自己的主进程） |
| NET | 网络（每个容器有自己的 IP、端口） |
| MNT | 文件系统（每个容器看到自己的目录树） |
| UTS | 主机名 |

### Cgroups — 限制资源

控制每个容器能用多少 CPU、内存、磁盘 I/O，防止某个容器把整台机器的资源吃光。

### UnionFS — 镜像分层

镜像是一层一层叠起来的，可以用"透明胶片叠画"来理解：

```text
┌──────────────────────┐
│  可写层（容器运行时）  │  ← 容器里写的文件都在这一层，删容器就没了
├──────────────────────┤
│  CMD ["uvicorn"...]   │  ← Dockerfile 最后一行
├──────────────────────┤
│  COPY app ./app       │  ← 复制代码
├──────────────────────┤
│  RUN pip install ...  │  ← 安装依赖
├──────────────────────┤
│  python:3.12-slim     │  ← 基础镜像
└──────────────────────┘
```

每一层是**只读**的，容器启动时在最上面加一个**可写层**。删除容器只是扔掉可写层，镜像本身不变。

好处：如果两个项目都用 `python:3.12-slim` 作为基础镜像，Docker 只存一份底层，不会重复占用空间。

---

## 3. 核心概念：镜像与容器

| 概念 | 是什么 | 后端类比 |
|---|---|---|
| **镜像 Image** | 静态模板，包含应用和依赖 | Java 的 class |
| **容器 Container** | 镜像运行后的进程实例 | Java 的 `new` 一个对象 |

一个镜像可以启动多个容器，就像一个 class 可以 new 出多个对象。

```bash
docker images   # 看镜像
docker ps       # 看正在运行的容器
docker ps -a    # 看所有容器（包括已停止的）
```

删除镜像用 `docker rmi`，删除容器用 `docker rm`，别混了。

---

## 4. 基础命令

高频命令速查表：

| 命令 | 作用 | 示例 |
|---|---|---|
| `docker run` | 创建并运行容器 | `docker run -d -p 8080:80 nginx` |
| `docker ps` | 查看运行中的容器 | `docker ps` |
| `docker logs` | 查看容器日志 | `docker logs -f 容器名` |
| `docker exec` | 在容器里执行命令 | `docker exec -it 容器名 bash` |
| `docker stop` | 停止容器 | `docker stop 容器名` |
| `docker rm` | 删除容器 | `docker rm 容器名` |
| `docker images` | 查看本地镜像 | `docker images` |
| `docker rmi` | 删除镜像 | `docker rmi 镜像名` |

### 端口映射

```bash
docker run -d -p 8080:80 nginx
```

`-p 8080:80` 的格式是 **宿主机端口:容器端口**。

意思是：访问宿主机的 8080 端口 → 转发到容器的 80 端口。方向容易记反，注意**左边是宿主机，右边是容器**。

---

## 5. Dockerfile

Dockerfile 是构建镜像的说明书。核心指令只有 5 个：

| 指令 | 作用 | 类比 |
|---|---|---|
| `FROM` | 基于哪个基础镜像 | 继承一个父类 |
| `WORKDIR` | 设置工作目录 | `cd` 到某个目录 |
| `COPY` | 把本机文件复制进容器 | 拷贝文件 |
| `RUN` | 构建时执行命令（装依赖等） | 编译期操作 |
| `CMD` | 容器启动后执行的命令 | 运行时操作 |

一个真实的 Python 后端 Dockerfile：

```dockerfile
FROM python:3.12-slim

RUN groupadd -g 1001 appuser && \
    useradd -u 1001 -g appuser -m appuser

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

COPY app ./app

RUN mkdir -p uploads logs && chown -R appuser:appuser /app
USER appuser

EXPOSE 8001

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8001/health')" || exit 1

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

几个要点：

- **先 COPY 依赖文件，再 COPY 代码**：利用 Docker 分层缓存，代码变了但依赖没变时不用重新装包。
- **USER appuser**：安全实践，不用 root 跑应用。
- **0.0.0.0**：容器里必须监听 `0.0.0.0`，如果监听 `127.0.0.1`，外部无法访问。
- **HEALTHCHECK**：定期检测应用是否存活。

### 构建上下文（Context）

Docker Compose 里经常看到这样的配置：

```yaml
backend:
  build:
    context: .
    dockerfile: deploy/Dockerfile.backend
```

`context: .` 是**构建上下文**——构建时 Docker 能看到的文件范围。Dockerfile 里的 `COPY` 路径都是相对于 context 目录的。

如果 context 是项目根目录 `.`，那 `COPY backend/app ./app` 就是从项目根目录下找 `backend/app`。

---

## 6. 数据管理：Volume

容器是"用完即扔"的，删了容器，容器内的数据也跟着没了。Volume 把数据存到容器外面，由 Docker 管理。

```yaml
services:
  mysql:
    image: mysql:8
    volumes:
      - mysql_data:/var/lib/mysql    # 使用数据卷

volumes:
  mysql_data:                        # 声明数据卷
```

**为什么要两个地方都写？**

- **顶层 `volumes:`** = 声明有哪些数据卷（类似声明变量）
- **服务里的 `volumes:`** = 实际使用这些数据卷（类似使用变量）

`mysql_data:/var/lib/mysql` 的意思：把名叫 `mysql_data` 的数据卷挂载到容器内的 `/var/lib/mysql` 目录。

```text
宿主机                          容器
┌────────────────┐            ┌────────────────┐
│  mysql_data    │ ◄────────► │ /var/lib/mysql  │
│  (Docker管理)  │   挂载      │ (MySQL数据目录)  │
└────────────────┘            └────────────────┘
```

Volume 的实际存储位置由 Docker 管理，可以用 `docker volume inspect mysql_data` 查看。

**重要**：`docker compose down` 默认**不删** volume，数据安全。但 `docker compose down -v` 会删 volume，数据库数据直接没了，千万小心。

---

## 7. 网络

### 容器间通信

Docker Compose 可以定义自定义网络，同一网络里的容器可以**用服务名互相访问**：

```yaml
services:
  mysql:
    image: mysql:8
    networks:
      - backend

  app:
    build: .
    environment:
      MYSQL_HOST: mysql       # 用服务名，不是 IP
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

Docker 会自动把服务名解析成对应容器的 IP（类似 DNS），所以后端配置数据库地址时写 `mysql` 而不是 `localhost` 或 IP。

### 为什么容器里不能用 localhost？

在容器里，`localhost` 指的是**容器自己**，不是宿主机。

```text
宿主机的 localhost = 你的电脑
容器里的 localhost = 容器自己
```

所以后端容器连数据库容器时，不能写 `localhost:3306`，要写 `mysql:3306`（服务名）。

---

## 8. Docker Compose

Docker Compose 用一个 yml 文件管理多个容器。

### 两种服务来源

```yaml
services:
  db:
    image: mysql:8         # 直接用 Docker Hub 的现成镜像

  app:
    build: .               # 用当前目录的 Dockerfile 构建镜像
```

### 环境变量与 .env 文件

```yaml
environment:
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}     # 从 .env 读取
  JWT_EXPIRE_HOURS: ${JWT_EXPIRE_HOURS:-24}       # 从 .env 读取，没有就默认 24
  UPLOAD_DIR: /app/uploads                        # 直接写死
```

`${MYSQL_ROOT_PASSWORD}` 这种写法，Docker Compose 会**自动**去当前目录找 `.env` 文件读取。这是 Docker Compose 的内置约定，不需要额外配置。

### 启动顺序

```yaml
backend:
  depends_on:
    mysql:
      condition: service_healthy
```

- 普通的 `depends_on` 只保证启动顺序，不保证服务 ready。
- `condition: service_healthy` 配合 `healthcheck` 使用，等 MySQL 真正可用后才启动后端。

### 常用命令

```bash
docker compose up -d            # 后台启动所有服务
docker compose down             # 停止并删除容器（不删 volume）
docker compose down -v          # 停止并删除容器 + volume（慎用！）
docker compose logs -f          # 实时查看日志
docker compose ps               # 查看服务状态
docker compose restart           # 重启所有服务
```

---

## 9. 实战案例

一个典型的前后端 + 数据库项目的 Docker Compose 配置：

```yaml
services:
  mysql:
    image: mysql:8
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: mydb
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: .
      dockerfile: deploy/Dockerfile.backend
    environment:
      MYSQL_HOST: mysql
      MYSQL_PORT: 3306
    volumes:
      - uploads:/app/uploads
      - logs:/app/logs
    networks:
      - backend
    depends_on:
      mysql:
        condition: service_healthy

  frontend:
    build:
      context: .
      dockerfile: deploy/Dockerfile.frontend
    ports:
      - "8080:80"
    networks:
      - backend
    depends_on:
      - backend

networks:
  backend:
    driver: bridge

volumes:
  mysql_data:
  uploads:
  logs:
```

启动流程：

```text
docker compose up -d
       │
       ├── mysql 先启动，healthcheck 每 10 秒检测
       │
       ├── 等 mysql 通过 healthcheck → backend 启动
       │
       └── 等 backend 容器创建 → frontend 启动
```

请求流程：

```text
浏览器 → localhost:8080 → frontend(nginx:80) → 反代到 backend:8001 → 查询 mysql:3306
```

注意 backend **没有暴露 ports**，它不直接对外，只通过 frontend 的 nginx 反向代理访问。

容器日志可以在 Docker Desktop 的 Logs 标签页查看，也可以用 `docker compose logs -f` 命令行看。Docker 自动捕获容器进程输出到 stdout/stderr 的内容，不需要额外配置。

---

## 10. 常见坑

| 坑 | 说明 |
|---|---|
| **localhost 指谁** | 容器里 `localhost` 指容器自己，连其他容器要用服务名 |
| **端口方向** | `-p 8080:80` 是 `宿主机:容器`，左宿右容 |
| **镜像 vs 容器** | `docker images` 看镜像，`docker ps` 看容器，删除也是 `rmi` vs `rm` |
| **volume 误删** | `docker compose down` 不删 volume，`docker compose down -v` 才删 |
| **监听地址** | 容器里应用要监听 `0.0.0.0`，监听 `127.0.0.1` 外部访问不到 |
| **container_name** | 可选配置，写了方便操作，但会阻止 `--scale` 扩容 |
| **depends_on** | 普通 depends_on 不等服务 ready，要配合 `condition: service_healthy` |

# Docker 核心知识点总结博客 — 设计规格

## 元信息

- **文件名**: `docker-knowledge-summary.md`
- **路径**: `src/content/posts/docker-knowledge-summary.md`
- **标题**: Docker 核心知识点总结：从原理到实战
- **description**: Docker 运行原理、核心概念、Dockerfile、Docker Compose、数据管理与网络 — 后端开发者的容器化知识地图
- **tags**: Docker, 容器化, DevOps, 实战
- **featured**: true
- **draft**: false

## 写作风格

参照"后端视角学前端"系列的风格：

- 开头一段简短的导语
- 用 `---` 分隔导语与正文
- 用 `## 编号. 标题` 作章节
- 大量使用**表格**做概念对比
- 用后端概念做类比（Java class = 镜像，new = 容器）
- 代码块展示关键命令和配置
- 每个知识点讲清"是什么"和"为什么"
- 避免纯命令罗列，注重理解

## 章节大纲

### 导语（2-3 句）

后端开发为什么要学 Docker，一句话定义。

### 1. Docker 解决什么问题

- "我的电脑能跑，你的跑不了"
- Docker 之前的部署流程 vs Docker 之后
- Docker vs 虚拟机的本质区别（共享内核 vs 完整 Guest OS）

### 2. 运行原理

三大核心技术，每个用 1-2 段话讲清楚：

| 技术 | 作用 | 一句话 |
|------|------|--------|
| Namespace | 隔离 | 每个容器以为自己是独立的 |
| Cgroups | 限制资源 | 限制 CPU、内存等 |
| UnionFS | 镜像分层 | 多层只读 + 一层可写 |

UnionFS 用"透明胶片叠画"的类比。

### 3. 核心概念：镜像与容器

- 镜像 = 静态模板 / Java 的 class
- 容器 = 运行实例 / Java 的 new
- 一个镜像可以启动多个容器
- `docker images` vs `docker ps`
- 镜像分层与缓存优化（先 COPY 依赖文件再 COPY 代码）

### 4. 基础命令

用表格整理高频命令：

| 命令 | 作用 |
|------|------|
| docker run | 创建并运行容器 |
| docker ps | 查看运行中的容器 |
| docker logs | 查看日志 |
| docker exec | 进入容器 |
| docker stop/rm | 停止/删除容器 |

端口映射 `-p 宿主机:容器` 方向解释。

### 5. Dockerfile

- 5 个核心指令：FROM, WORKDIR, COPY, RUN, CMD
- 用真实的 Python 后端 Dockerfile 作为示例
- 构建上下文（context）是什么：COPY 的根目录
- 安全实践：创建非 root 用户（USER appuser）
- HEALTHCHECK 配置

### 6. 数据管理：Volume

- 为什么需要 Volume（容器删了数据不丢）
- 两处写法解释：顶层声明 + 服务里使用
- Volume 实际存储位置
- `docker compose down` vs `docker compose down -v`

### 7. 网络

- Docker 自定义网络（bridge 模式）
- 容器间通过服务名通信（Docker DNS）
- 为什么容器里不能用 localhost 连其他容器
- 为什么应用要监听 0.0.0.0

### 8. Docker Compose

- 一个 yml 管理多个容器
- 两种服务来源：`image:` vs `build:`
- 环境变量：`${VAR}` 和 `.env` 文件（自动读取的约定）
- `${VAR:-default}` 默认值语法
- `depends_on` + `condition: service_healthy`
- 常用命令：up/down/logs/ps

### 9. 实战案例

用 docker-compose.test.yml 作为完整案例：
- 三服务架构：MySQL + Backend + Frontend
- 请求流程：浏览器 → nginx → backend → mysql
- 容器日志：stdout 自动收集

### 10. 常见坑

用表格或列表整理：
- localhost 在容器里指的是容器自己
- `-p 8080:80` 方向：宿主机:容器
- docker images 看镜像，docker ps 看容器
- `docker compose down -v` 会删 volume
- 容器里监听 127.0.0.1 外面访问不到

## 预估长度

约 2000-2500 字（不含代码块），10 个章节。

## 不包含的内容

- Kubernetes / Docker Swarm
- CI/CD 流水线
- 私有镜像仓库
- 多阶段构建优化（可后续单独写）
- Docker 安装配置

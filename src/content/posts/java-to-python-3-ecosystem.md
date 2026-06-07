---
title: "Java 转 Python（三）：生态与部署"
description: "AI Agent、生态总览、代码分层、Uvicorn/Gunicorn 部署 — Python 工程化实践"
pubDatetime: 2026-06-07T00:00:00.000Z
featured: false
tags:
  - python
  - java
  - 部署
  - AI
---

系列第三篇。前两篇覆盖了语法、FastAPI、数据库；本篇聚焦 **AI Agent、生态选型、分层架构、部署与工程文化**。

---

## 1. AI Agent 对话

PydanticAI = Spring AI 的 Python 版。`@agent.tool` 注册函数，AI 自动决定何时调用。

```python
# 1. 创建 Agent
agent = Agent(
    OpenAIModel("deepseek-v4-flash", provider=...),
    instructions="你是 PayReach BD 助手...",
)

# 2. 注册 Tool（AI 可调用的函数）
@agent.tool
async def query_exhibitions(ctx, keyword="", region=""):
    """查询展会库。按关键词搜索。"""
    repo = get_exhibition_repo()
    exhibitions, total = await repo.find_paginated(keyword=keyword)
    return json.dumps({"exhibitions": [...], "total": total})

# 3. 运行对话
result = await agent.run("帮我查上海的展会", message_history=history)
print(result.output)  # AI 的自然语言回复
```

```text
用户: "帮我查上海的展会"
  ↓
Agent 分析意图 → 决定调用 query_exhibitions(keyword="上海")
  ↓
Tool 查数据库 → 返回展会列表 JSON
  ↓
Agent 组织自然语言 → "上海地区共有 5 个展会..."
```

| 概念 | Spring AI | PydanticAI |
|------|-----------|------------|
| Agent 创建 | `ChatClient.builder()` | `Agent(model, instructions)` |
| 注册工具 | `@Bean Function` | `@agent.tool` |
| 对话 | `chatClient.call(msg)` | `agent.run(msg)` |
| 工具描述 | `@Description` | docstring（函数注释） |

---

## 2. Python 生态总览

### Web 框架

| 框架 | 类比 Java | 特点 | 适合场景 |
|------|-----------|------|----------|
| Django | Spring Boot 全家桶 | ORM+Admin+Auth 全内置 | 传统 Web、后台管理 |
| FastAPI | Spring Boot 轻量版 | 异步原生、Pydantic 集成 | API 服务、AI 项目 |
| Flask | Vert.x / Spark Java | 极简，啥都自己选 | 简单 API、微服务 |

### ORM

| 框架 | 类比 Java | 特点 |
|------|-----------|------|
| SQLAlchemy | Hibernate | Python ORM 标准，功能最全 |
| Django ORM | Spring Data JPA | Django 内置 |
| SQLModel | MyBatis-Plus | FastAPI 作者做的，较新 |
| 手写 SQL | JdbcTemplate | 灵活，本项目早期做法 |

### Pydantic 生态

| 项目 | 作者 | 用途 |
|------|------|------|
| Pydantic | Samuel Colvin | 数据验证（= Lombok + Jackson + Validator） |
| PydanticAI | Pydantic 公司 | AI Agent 框架 |
| Logfire | Pydantic 公司 | 可观测性平台 |
| FastAPI | Sebastián Ramírez | Web 框架（**不是** Pydantic 公司的） |
| SQLModel | Sebastián Ramírez | ORM |

> FastAPI **依赖** Pydantic，但作者是不同的人。

---

## 3. 核心代码分层架构

和 Spring Boot 一样：Route → Service → Repository → Model。

```text
┌──────────────────────────────────────────────────┐
│  Route (Controller)                               │
│  routes/login.py, routes/companies.py ...         │
│  只做参数校验 + 调 Service + 返回响应              │
├──────────────────────────────────────────────────┤
│  Service                                          │
│  services/auth.py, services/article_service.py    │
│  业务编排 + session_scope() 事务管理               │
├──────────────────────────────────────────────────┤
│  Repository (Mapper/DAO)                          │
│  repositories/user_repo.py ...                    │
│  继承 BaseRepository, _use_session 透传 session    │
├──────────────────────────────────────────────────┤
│  Model 层                                         │
│  models/user.py (DTO) ←→ db/tables/user.py (Entity) │
├──────────────────────────────────────────────────┤
│  db/engine.py                                     │
│  AsyncEngine + session_scope + init_db            │
└──────────────────────────────────────────────────┘
```

| Python 文件 | Java 对应物 | 职责 |
|-------------|-------------|------|
| `db/engine.py` | application.yml + DataSource | 引擎配置 + 事务管理 |
| `db/tables/user.py` | `@Entity` / `@TableName` | 表结构映射 |
| `models/user.py` | UserDTO / UserVO | 业务传输对象 |
| `repositories/base.py` | `BaseMapper<T>` | 通用 session 管理 |
| `repositories/user_repo.py` | UserMapper | CRUD 数据操作 |
| `services/auth.py` | `@Service AuthService` | 业务逻辑编排 |
| `routes/login.py` | `@RestController LoginCtrl` | HTTP 路由入口 |

---

## 4. Session 透传机制

= Java 的 `@Transactional` 事务传播。Service 创建 session，传给多个 Repository，实现多表原子操作。

```python
# Python — 手动传 session
async with session_scope() as session:
    await article_repo.save_analysis(data, session=session)
    await exhibition_repo.upsert(exhibition, session=session)
    # 两个写操作在同一个事务中，要么都成功，要么都回滚
```

```java
// Java — 声明式传播
@Transactional
public void analyze(int articleId) {
    articleMapper.saveAnalysis(data);        // 自动在同一事务
    exhibitionMapper.upsert(exhibition);     // 自动在同一事务
}
```

```python
# BaseRepository — 有外部 session 就共享，没有就独立事务
class BaseRepository:
    @asynccontextmanager
    async def _use_session(self, session=None):
        if session is not None:
            yield session            # 有外部 session → 共享事务
        else:
            async with session_scope() as s:
                yield s              # 无外部 session → 独立事务
```

> Java `@Transactional` 是声明式（框架自动管理），Python 显式传 `session` 参数（手动但透明）。

---

## 5. 部署：Uvicorn & Gunicorn

= Java 的 Tomcat。Uvicorn 干活，Gunicorn 管进程。

| Python | Java 对应 | 职责 |
|--------|-----------|------|
| FastAPI | Spring MVC | 写路由、处理请求 |
| Uvicorn | Tomcat（内嵌） | HTTP 服务器，处理网络 I/O |
| Gunicorn | Tomcat 线程池管理 | 管理多个 Worker 进程 |

```bash
# 开发环境 — 单进程 + 热重载
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

# 生产环境方式1 — Uvicorn 多进程（简单够用）
uvicorn app.main:app --host 0.0.0.0 --port 8001 --workers 4

# 生产环境方式2 — Gunicorn + Uvicorn（更成熟）
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8001
```

```text
                    ┌─ Worker 1 (PID 1001) ─ 自己的 GIL ─ 处理请求
                    │
Gunicorn (Master) ──┼─ Worker 2 (PID 1002) ─ 自己的 GIL ─ 处理请求
                    │
                    ├─ Worker 3 (PID 1003) ─ 自己的 GIL ─ 处理请求
                    │
                    └─ Worker 4 (PID 1004) ─ 自己的 GIL ─ 处理请求
```

| | Java (Tomcat) | Python (Gunicorn+Uvicorn) |
|--|---------------|---------------------------|
| 进程 | 通常 1 个 JVM 进程 | 多个 Worker 进程 |
| 并发 | 1进程 × 200线程 | 4进程 × 每进程数千协程 |
| CPU | 200线程跑满多核 | 4进程跑满4核 |
| 内存 | 共享堆内存 | 每进程独立内存 |

> 多进程时连接池翻倍：4 进程 × `pool_size=10` = 40 个连接。Worker 数通常设为 CPU 核心数 × 2 + 1。

---

## 6. 注释文化 & 生态读音

### 注释差异

| | Java | Python |
|--|------|--------|
| 方法注释 | **必须**，Javadoc 全套 | **看情况**，函数名清晰就不写 |
| 字段注释 | 实体类每个字段 | 几乎不写，类型注解代替 |
| 模块注释 | 类级别 Javadoc | 文件顶部 docstring |
| 行内注释 | 较多 | 只注释"为什么"，不注释"做什么" |
| 检查工具 | SonarQube 强制 | pylint 不检查注释覆盖率 |

```python
# ❌ 坏注释（解释做什么 — Python 社区不推荐）
user = await repo.find_by_email(email)  # 查找用户

# ✅ 好注释（解释为什么 — Python 推荐）
# bcrypt 在事务外执行，避免 DB 连接长时间占用
new_hash = hash_password(new_password)
```

### 生态读音

| 单词 | 读法 | 来源 |
|------|------|------|
| Uvicorn | 尤维康 (you-vi-corn) | UV + Unicorn |
| Gunicorn | 咕尼康 (goo-ni-corn) | Green Unix Unicorn |
| FastAPI | 法斯特 API | Fast + API |
| Pydantic | 派丹提克 (pie-DAN-tick) | Py + pedantic |
| SQLAlchemy | S-Q-L 阿尔凯米 | SQL + Alchemy |
| asyncio | 阿辛克IO (a-SINK-ee-oh) | async + I/O |
| pytest | 派泰斯特 (pie-test) | Py + test |
| ASGI | A-S-G-I 四个字母 | Async Server Gateway Interface |
| WSGI | 威斯基 (wiz-ghee) | Web Server Gateway Interface |

> Python Web 服务器爱用独角兽命名（Gunicorn、Uvicorn），就像 Java 生态爱用动物（Tomcat、Kafka、Camel）。

### 命名规范速记

| 元素 | Java（驼峰） | Python（蛇形） |
|------|-------------|----------------|
| 方法 | `findByEmail()` | `find_by_email()` |
| 变量 | `userName` | `user_name` |
| 常量 | `MAX_RETRY` | `MAX_RETRY`（一样） |
| 类名 | `UserRepository` | `UserRepository`（一样） |
| 文件名 | `UserRepository.java` | `user_repo.py` |

> 转换记忆法：驼峰大写字母 → `_小写` → `findByEmail` → `find_by_email`

---

## 系列总结

| 篇 | 主题 | 核心收获 |
|----|------|----------|
| 一 | 语法与数据结构 | 无类型声明、snake_case、list/dict 推导式 |
| 二 | 框架与数据库 | FastAPI 路由/DI、Pydantic、SQLModel、async/await |
| 三 | 生态与部署 | AI Agent、分层架构、Uvicorn/Gunicorn、工程文化 |

三篇下来，Java 开发者应该能读懂 PayReach 这类 Python 后端项目，并独立完成 API 开发与部署。

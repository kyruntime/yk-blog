---
title: "Python 速查手册：Java 开发者版"
description: "从 Java 转 Python 的完整对照指南：覆盖语法、类、模块、数据结构、FastAPI、数据库、async/await、装饰器、GIL 等 16 个主题"
pubDatetime: 2026-06-07T00:00:00.000Z
featured: true
tags:
  - python
  - java
  - 速查手册
  - 后端开发
---

基于 PayReach 项目代码，用 Java 对照学 Python。面试前快速复习用。

整理日期：2026 年 6 月 3 日 · 来源：项目实战对话 · 更新：第二轮深入学习

---

## 01 · 基础语法差异

### 变量声明

Python 不需要声明变量类型，直接赋值就创建变量。

```text
# Python — 直接赋值
now = time.time()
name = "CES"
score = 95

// Java — 必须声明类型
double now = System.currentTimeMillis();
String name = "CES";
int score = 95;
```

### 类型注解（可选 vs 必须）

Python 的类型注解是**可选的**，加不加都能运行。

```text
# Python — 以下两种都合法
def greet(name: str) -> str:    # 有类型注解
    return f"Hello {name}"

def greet(name):                # 无类型注解，也能跑
    return f"Hello {name}"

// Java — 类型必须写
public String greet(String name) {
    return "Hello " + name;
}
```

> **提示：** FastAPI 特例：在 FastAPI 路由中，类型注解不只是文档——FastAPI 会用它做参数验证和自动文档生成。

### 命名规范

| 概念 | Java | Python |
| --- | --- | --- |
| 私有变量 | `private` 关键字 | `_` 前缀（约定，非强制） |
| 常量 | `static final` | 全大写 `MAX_SIZE = 100` |
| 类名 | PascalCase | PascalCase（相同） |
| 方法名 | camelCase | snake_case |
| 包名 | com.example.pkg | app.services.xxx |

### 字符串格式化

```text
# Python f-string
name, score = "CES", 95
msg = f"展会 {name} 评分 {score} 分"

// Java
String msg = String.format("展会 %s 评分 %d 分", name, score);
// 或 Java 21+
String msg = STR."展会 \{name} 评分 \{score} 分";
```

---

## 02 · 函数与类

### Python 不强制用 class

Java 里一切必须放在 class 里。Python 的函数可以独立存在。

```text
# Python — 直接写函数，不需要 class
async def analyze_article(title: str, text: str) -> dict:
    client = _get_client()
    response = await client.chat(...)
    return response

// Java — 必须有 class
public class AiAnalyzer {
    public Map<String, Object> analyzeArticle(String title, String text) {
        // ...
    }
}
```

> **提示：** 原则：有状态（需要记住东西）→ 用 class。无状态（纯函数）→ 直接写函数。

### self = this

Python 的 `self` 必须显式写在方法参数里。

```text
# Python — self 必须写
class WeChatClient:
    def __init__(self, token: str):  # 构造器
        self.token = token           # self 必须写

    async def close(self):           # self 必须写
        await self._client.aclose()

// Java — this 隐式存在
public class WeChatClient {
    private String token;
    public WeChatClient(String token) {
        this.token = token;          // this 可写可不写
    }
    public void close() { ... }
}
```

| 特性 | Java this | Python self |
| --- | --- | --- |
| 声明位置 | 不出现在参数列表 | **必须**写在参数第一位 |
| 使用时 | 可写可不写 | **必须**写 self.x |
| 调用方 | 不传 | 不传（自动传入） |

### @property = getter

Python 不需要 getter/setter，直接 `.属性名` 访问。需要逻辑时用 `@property`。

```text
# Python — 直接访问 + @property 计算属性
class Exhibition(BaseModel):
    name: str
    score: int = 0

    @property
    def recommendation_level(self) -> str:
        if self.score >= 90: return "强烈推荐"
        if self.score >= 70: return "值得关注"
        return "一般"

e = Exhibition(name="CES", score=95)
e.name                    # 直接访问，不需要 .getName()
e.recommendation_level    # @property，看起来也像属性

// Java
e.getName();
e.getRecommendationLevel();
```

### 一个文件可以有多个 class

```text
# Python wechat.py — 一个文件里 4 个类 + 常量
USER_AGENT = "Mozilla/5.0 ..."
BASE_URL = "https://mp.weixin.qq.com"

class WeChatAPIError(Exception): ...
class SessionExpiredError(WeChatAPIError): ...
class WeChatClient: ...
class WeChatLoginFlow: ...

// Java — 一个文件只能有一个 public class
// WeChatClient.java
// WeChatLoginFlow.java
// WeChatAPIError.java
// SessionExpiredError.java  → 4 个文件
```

### 内部类：Python 几乎不用

Java 有四种内部类且很常用。Python 语法支持但社区不用——相关类直接并列写在同一文件里。

---

## 03 · 模块与导入

### import 可以导入任何东西

```text
# Python — 可以导入 class、函数、变量、常量
from app.services.wechat import WeChatClient          # class
from app.services.wechat import SessionExpiredError    # 异常类
from app.services.ai_analyzer import analyze_article   # 函数
from app.services.wechat import USER_AGENT             # 常量

// Java — 只能导入类
import com.example.services.WeChatClient;
```

| 特性 | Java | Python |
| --- | --- | --- |
| import 什么 | 只能 class | class / 函数 / 变量 / 常量 |
| 一个文件 | 一个 public class | 多个 class + 函数 + 变量 |
| 路径映射 | com.example.Foo → com/example/Foo.java | app.services.wechat → app/services/wechat.py |

### 函数内导入（避免循环依赖）

```python
# Python — 可以在函数内部 import
async def _get_system_prompt() -> str:
    from app.deps import get_config_repo    # 延迟导入
    val = await get_config_repo().get("ai_analyzer_prompt", "")
    return val
```

> **提示：** 原因：Python 的模块在 import 时就会执行顶层代码。如果 A import B，B 又 import A，就循环了。放在函数内部可以延迟导入时机。

---

## 04 · 数据结构

### list（= ArrayList）

```text
# 创建
items = [1, 2, 3]                    # Java: List.of(1, 2, 3)
items = []                           # Java: new ArrayList<>()

# 增删查
items.append(4)                      # Java: items.add(4)
items.pop()                          # 删最后一个
items[0]                             # Java: items.get(0)
items[-1]                            # 最后一个！Java 没有
3 in items                           # Java: items.contains(3)
len(items)                           # Java: items.size()

# 切片（Python 独有）
items[1:3]                           # 取索引 1~2
items[:3]                            # 前 3 个
items[-3:]                           # 最后 3 个
items[::-1]                          # 反转！

# 排序
sorted(items, key=lambda x: x.name) # Java: items.sort(Comparator.comparing(X::getName))
```

### dict（= HashMap）

```text
# 创建
d = {"name": "CES", "score": 95}    # Java: Map.of(...)

# 增改查
d["location"] = "上海"               # Java: d.put("location", "上海")
d["name"]                            # KeyError if missing!
d.get("name")                        # None if missing
d.get("name", "默认值")              # Java: d.getOrDefault(...)
"name" in d                          # Java: d.containsKey("name")

# 遍历
for key, value in d.items():         # Java: for (Map.Entry e : d.entrySet())
```

### 列表推导式（Python 最强语法）

```text
# 基本形式
names = [e.name for e in exhibitions]

# 带条件
high = [e for e in exhibitions if e.score > 80]

# dict 推导式
pairs = {e.id: e.name for e in exhibitions}

# Java Stream 等价
exhibitions.stream()
    .filter(e -> e.getScore() > 80)
    .collect(Collectors.toList());
```

### ** 解包语法

```python
data = {"name": "张三", "age": 25}

# 以下两行完全等价
user = CreateUser(**data)
user = CreateUser(name="张三", age=25)

# 项目实际用法：数据库行 → 对象
rows = await cur.fetchall()
return [Exhibition(**r) for r in rows]
```

> **提示：** ** 就是把 dict 展开成关键字参数。Java 没有等价语法。

### tuple 多返回值

```text
# Python 函数可以返回多个值
def find_paginated(...) -> tuple[list[Exhibition], int]:
    return exhibitions, total

# 调用方拆包
exhibitions, total = await repo.find_paginated(page=1)

// Java 做不到，需要包装类
// public class PageResult<T> { List<T> items; int total; }
```

---

## 05 · Pydantic 数据模型

### Pydantic = Lombok + Jackson + Validator

```text
# Python Pydantic
class Exhibition(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    date_range: str = ""            # 有默认值 = 选填
    score: int = 0
    first_article_id: Optional[int] = None  # 可以是 int 或 None

// Java 等价
@Data                               // Lombok
public class Exhibition {
    private int id;
    @NotNull private String name;
    private String dateRange = "";
    private int score = 0;
    @Nullable private Integer firstArticleId;
}
```

| 特性 | Java | Pydantic |
| --- | --- | --- |
| getter/setter | Lombok @Data | 自动有，直接 .name |
| JSON 序列化 | Jackson | 内置 .model_dump() |
| 数据验证 | @Valid + @NotNull | 运行时自动验证 |
| 默认值 | 字段赋值 | `field: type = 默认值` |
| 必填 | @NotNull | 不写默认值 = 必填 |

### Optional[int] 含义

```python
first_article_id: Optional[int] = None
# 拆解：
# first_article_id → 字段名
# Optional[int]    → 类型：int 或 None
# = None           → 默认值是 None（选填）

# 等价新写法（Python 3.10+）
first_article_id: int | None = None
```

### BaseModel vs dataclass

```python
# Pydantic BaseModel — 有运行时验证
class Exhibition(BaseModel):
    name: str
    score: int
Exhibition(name=123, score="abc")  # ❌ ValidationError!

# dataclass — 无验证
@dataclass
class NewsSource:
    name: str = ""
    id: int = 0
NewsSource(name=123, id="abc")     # ✅ 不报错，但数据是错的
```

> **提示：** 选择原则：需要验证外部输入 → BaseModel。内部使用、数据可信 → dataclass。

### field(default_factory=list)

```python
# ❌ 错误！所有实例共享同一个 list
class Bad:
    keywords: list[str] = []

# ✅ 正确！每次创建新 list
class Good:
    keywords: list[str] = field(default_factory=list)
```

> **提示：** Python 的坑：可变对象（list/dict）不能直接当默认值，否则所有实例共享同一个对象。

---

## 06 · FastAPI 框架

### 路由定义（= @GetMapping）

```text
# Python FastAPI
router = APIRouter(prefix="/api/wechat", tags=["登录"])

@router.get("/login/qrcode")
async def get_qrcode():
    return {"qrcode": "base64data..."}

// Java Spring Boot
@RestController
@RequestMapping("/api/wechat")
public class LoginController {
    @GetMapping("/login/qrcode")
    public Map<String, Object> getQrcode() {
        return Map.of("qrcode", "base64data...");
    }
}
```

### 依赖注入（= @Autowired）

```text
# Python — deps.py 手动工厂
@lru_cache                           # 单例（只创建一次）
def get_article_repo() -> ArticleRepository:
    return ArticleRepository()

@lru_cache
def get_article_service() -> ArticleService:
    return ArticleService(
        article_repo=get_article_repo(),      # 手动注入
        exhibition_repo=get_exhibition_repo(),
    )

# 路由中使用
@router.get("/articles")
async def list_articles(
    service: ArticleService = Depends(get_article_service),  # 注入
):
    return await service.list_all()

// Java Spring — 自动注入
@Service
public class ArticleService {
    @Autowired private ArticleRepository articleRepo;  // 自动
}
```

| 概念 | Spring | FastAPI |
| --- | --- | --- |
| 定义 Bean | @Bean / @Component | @lru_cache 工厂函数 |
| 注入 | @Autowired（自动） | Depends()（显式） |
| 单例 | @Scope 默认单例 | @lru_cache |
| 配置类 | @Configuration class | deps.py 模块 |

### 中间件（= Filter / Interceptor）

```text
# Python FastAPI
class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.monotonic()
        response = await call_next(request)   # 放行到下一层
        elapsed = (time.monotonic() - start) * 1000
        logger.info("%s %s status=%d elapsed=%.0fms",
            request.method, request.url.path,
            response.status_code, elapsed)
        return response

# 注册
app.add_middleware(AccessLogMiddleware)

// Java
@Component
public class AccessLogFilter extends OncePerRequestFilter {
    protected void doFilterInternal(request, response, chain) {
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);
        long elapsed = System.currentTimeMillis() - start;
        log.info(...)；
    }
}
```

### 启动流程（= Application 主类）

| 启动步骤 | Spring Boot | FastAPI main.py |
| --- | --- | --- |
| 读配置 | application.yml | load_dotenv() |
| 创建应用 | SpringApplication.run() | FastAPI() |
| 扫描 Controller | @ComponentScan（自动） | app.include_router()（手动） |
| 注册中间件 | @Bean Filter | app.add_middleware() |
| 启动初始化 | @PostConstruct | @app.on_event("startup") |
| 关闭清理 | @PreDestroy | @app.on_event("shutdown") |
| 数据库迁移 | Flyway | ensure_table()（手写 DDL） |

---

## 07 · 数据库操作

### 引擎与连接池（= HikariCP + DataSource）

已迁移至 SQLModel + SQLAlchemy AsyncEngine，不再使用原始 aiomysql。

```text
# Python SQLModel — db/engine.py
DATABASE_URL = URL.create("mysql+aiomysql", username=..., password=..., host=..., database=...)

engine = create_async_engine(DATABASE_URL, pool_size=10, max_overflow=5, pool_recycle=3600)

async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

// Java HikariCP
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://...");
config.setMaximumPoolSize(10);
DataSource ds = new HikariDataSource(config);
```

### 事务管理（= @Transactional）

```text
# Python — session_scope 上下文管理器
@asynccontextmanager
async def session_scope():
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# Service 层使用（等价 Java 的 @Transactional）
async def change_password(user_id, old_pwd, new_pwd):
    current_hash = await repo.get_password_hash(user_id)  # 事务外
    new_hash = hash_password(new_pwd)                      # 事务外（CPU密集）
    async with session_scope() as session:                 # ← 开启事务
        await repo.update_password(user_id, new_hash, session=session)
    # 退出 async with → 自动 commit

// Java
@Transactional
public void changePassword(int userId, String oldPwd, String newPwd) {
    // Spring 自动处理 commit/rollback
}
```

### 表模型（= @Entity / @TableName）

```text
# Python SQLModel — db/tables/user.py
class UserTable(SQLModel, table=True):
    __tablename__ = "users"
    id: Optional[int] = Field(default=None, primary_key=True)
    email: str = Field(max_length=100, sa_column_kwargs={"unique": True})
    password_hash: str = Field(max_length=255)
    name: str = Field(max_length=50)
    role: Optional[str] = Field(default="user",
        sa_column=Column(SAEnum("admin", "user"), server_default="user"))

// Java MyBatis-Plus
@TableName("users")
public class UserEntity {
    @TableId(type = IdType.AUTO) private Integer id;
    private String email;
    private String passwordHash;
    private String name;
    private String role;
}
```

### Repository 层（= BaseMapper + Service）

```text
# Python — BaseRepository 提供 session 管理
class BaseRepository:
    @asynccontextmanager
    async def _use_session(self, session=None):
        if session is not None:
            yield session            # 有外部 session → 共享事务
        else:
            async with session_scope() as s:
                yield s              # 无外部 session → 独立事务

# 具体 Repository
class UserRepository(BaseRepository):
    async def find_by_email(self, email, session=None):
        async with self._use_session(session) as s:
            stmt = select(UserTable).where(UserTable.email == email)
            row = (await s.exec(stmt)).first()
            return User.model_validate(row) if row else None

    async def update(self, user_id, name=None, session=None):
        async with self._use_session(session) as s:
            row = await s.get(UserTable, user_id)
            if name is not None:
                row.name = name      # 直接改属性 = UPDATE（脏检查）
            s.add(row)
            await s.flush()

// Java MyBatis-Plus
User user = userMapper.selectOne(new QueryWrapper<User>().eq("email", email));
user.setName(newName);
userMapper.updateById(user);
```

| 概念 | Java | Python（本项目 SQLModel） |
| --- | --- | --- |
| 连接池 | HikariCP | SQLAlchemy AsyncEngine |
| 事务 | @Transactional | session_scope() |
| 查询 | QueryWrapper | select(Table).where(...) |
| 更新 | updateById() | 改属性 + s.add() + flush() |
| 表定义 | @TableName Entity | SQLModel(table=True) |
| 建表 | 手动 DDL / Flyway | SQLModel.metadata.create_all() |

---

## 08 · 配置管理

### Pydantic Settings（= application.yml）

```python
# Python config.py
class Settings(BaseSettings):
    mysql_host: str = "22.50.21.10"     # 有默认值 = 选填
    mysql_app_password: str              # 无默认值 = 必填！
    ai_chat_api_key: str                 # 必填

    model_config = {"env_file": ".env"}  # 从 .env 读取

settings = Settings()  # 自动读取环境变量

# 使用
from app.config import settings
host = settings.mysql_host
```

#### 完整链路

```text
.env 文件                    →  config.py Settings()    →  其他代码
MYSQL_APP_PASSWORD=xxx           settings.mysql_app_password    base.py: 连数据库
AI_CHAT_API_KEY=sk-xxx           settings.ai_chat_api_key       ai_analyzer.py: 调 AI
```

| 特性 | Spring Boot | Pydantic Settings |
| --- | --- | --- |
| 配置文件 | application.yml | .env（键值对） |
| 必填检查 | @NotNull | 无默认值 = 必填 |
| 注入 | @Value / @ConfigurationProperties | from app.config import settings |
| 多环境 | application-dev.yml | .env.local（override） |

---

## 09 · AI Agent 对话

### PydanticAI + Function Calling

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

#### 对话流程

```text
用户: "帮我查上海的展会"
  ↓
Agent 分析意图 → 决定调用 query_exhibitions(keyword="上海")
  ↓
Tool 查数据库 → 返回展会列表 JSON
  ↓
Agent 组织自然语言 → "上海地区共有 5 个展会..."
  ↓
保存对话历史到 MySQL
```

| 概念 | Spring AI | PydanticAI |
| --- | --- | --- |
| Agent 创建 | ChatClient.builder() | Agent(model, instructions) |
| 注册工具 | @Bean Function | @agent.tool |
| 对话 | chatClient.call(msg) | agent.run(msg) |
| 工具描述 | @Description | docstring（函数注释） |

---

## 10 · Python 生态总览

### Web 框架对比

| 框架 | 类比 Java | 特点 | 适合场景 |
| --- | --- | --- | --- |
| Django | Spring Boot 全家桶 | ORM+Admin+Auth 全内置 | 传统 Web、后台管理 |
| FastAPI | Spring Boot 轻量版 | 异步原生、Pydantic 集成 | API 服务、AI 项目 |
| Flask | Vert.x / Spark Java | 极简，啥都自己选 | 简单 API、微服务 |

### ORM 对比

| 框架 | 类比 Java | 特点 |
| --- | --- | --- |
| SQLAlchemy | Hibernate | Python ORM 标准，功能最全 |
| Django ORM | Spring Data JPA | Django 内置 |
| SQLModel | MyBatis-Plus | FastAPI 作者做的，较新 |
| 手写 SQL | JdbcTemplate | 本项目的做法 |

### Pydantic 生态帝国

| 项目 | 作者 | 用途 |
| --- | --- | --- |
| Pydantic | Samuel Colvin | 数据验证（= Lombok + Jackson + Validator） |
| PydanticAI | Pydantic 公司 | AI Agent 框架 |
| Logfire | Pydantic 公司 | 可观测性平台（商业） |
| FastAPI | Sebastián Ramírez | Web 框架（不是 Pydantic 公司的！） |
| SQLModel | Sebastián Ramírez | ORM |

> **提示：** 注意：FastAPI 和 Pydantic 不是同一个团队做的！FastAPI **依赖** Pydantic，但作者是不同的人。

### Django Admin：内置后台管理

Django 最强卖点——几行代码自动生成完整的数据管理后台（有 UI、有表单、有权限），不需要写前端。

```python
# Django admin.py — 就这两行
from django.contrib import admin
admin.site.register(Exhibition)
# → 自动得到：登录页、数据列表、搜索、筛选、增删改查、权限管理
```

> **提示：** AI 时代的思考：Django Admin 的"自动生成"优势正在被 AI 辅助开发削弱。现在用 AI 直接写 Vue/React 前端，速度差不多，但灵活性高得多。

---

## 11 · 核心代码分层架构

### 项目分层 = Spring Boot 分层

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
| --- | --- | --- |
| db/engine.py | application.yml + DataSource | 引擎配置 + 事务管理 |
| db/tables/user.py | @Entity / @TableName | 表结构映射 |
| models/user.py | UserDTO / UserVO | 业务传输对象 |
| repositories/base.py | BaseMapper\<T\> | 通用 session 管理 |
| repositories/user_repo.py | UserMapper | CRUD 数据操作 |
| services/auth.py | @Service AuthService | 业务逻辑编排 |
| routes/login.py | @RestController LoginCtrl | HTTP 路由入口 |

### Session 透传机制（= 事务传播）

Service 创建 session，传给多个 Repository，实现多表原子操作：

```text
# Python — 手动传 session
async with session_scope() as session:
    await article_repo.save_analysis(data, session=session)
    await exhibition_repo.upsert(exhibition, session=session)
    # 两个写操作在同一个事务中，要么都成功，要么都回滚

// Java — 声明式传播
@Transactional
public void analyze(int articleId) {
    articleMapper.saveAnalysis(data);        // 自动在同一事务
    exhibitionMapper.upsert(exhibition);     // 自动在同一事务
}
```

> **提示：** 核心区别：Java 的 @Transactional 是声明式的（框架自动管理），Python 需要显式传递 session 参数（手动但透明）。

---

## 12 · async/await 协程

### Java 多线程 vs Python 协程

```text
// Java — 一个请求 = 一个线程（Tomcat 默认 200 线程）
@GetMapping("/articles")
public List<Article> getArticles() {
    List<Article> articles = articleMapper.selectList(null);  // 线程卡住等 DB
    return articles;
}

# Python — 一个请求 = 一个协程（一个线程管数千协程）
@router.get("/articles")
async def get_articles():
    articles = await article_repo.find_list()   # 让出 CPU → 处理别的 → DB返回 → 继续
    return articles
```

| | Java 多线程 | Python async |
| --- | --- | --- |
| 等 DB 时 | 线程**阻塞**，啥也不干 | 协程**让出**，去处理别的请求 |
| 并发模型 | 200 个线程 = 200 并发 | 1 个线程 + 数千协程 = 数千并发 |
| 内存开销 | 每线程 ~1MB（200线程=200MB） | 每协程 ~几KB（5000协程≈几十MB） |
| CPU 密集 | 多核并行 | **单线程**，CPU密集会卡住 |

### 做饭比喻

**Java（多线程）：** 你有 200 个厨师（线程），每个厨师煮一碗面，等水烧开时就**站在锅边干等**。想多做面？再雇厨师。

**Python（async）：** 你有 1 个超级厨师（事件循环），煮面时水还没开？去切另一碗面的菜！水开了回来下面，面煮着？去准备第三碗！1 个人同时管 5000 碗面。

### await 的含义

```python
# await s.exec(stmt) 的意思是：
# 1. 把 SQL 发给 MySQL
# 2. 让出 CPU（不阻塞！去处理别的请求）
# 3. MySQL 返回结果后，回来继续执行

# 如果不用 await（同步代码），等 DB 时会卡住整个 Python 进程
```

### asyncio.gather — 并行查询

```text
# Python — 5 个查询并行执行！
articles, total, tags, global_total, new_count = await asyncio.gather(
    article_repo.find_list(...),
    article_repo.count(...),
    article_repo.get_keyword_tags(...),
    article_repo.count(),
    article_repo.count_new(),
)

// Java 等价 — 需要 CompletableFuture
CompletableFuture.allOf(
    CompletableFuture.supplyAsync(() -> articleMapper.selectList(...)),
    CompletableFuture.supplyAsync(() -> articleMapper.count(...)),
    // ... 复杂得多
);
```

---

## 13 · 装饰器 vs 注解

### 外观像，本质不同

```text
// Java 注解 — 只是标签，Spring 扫描后才生效
@Transactional
@RequestMapping("/api/users")
public class UserService { ... }

# Python 装饰器 — 就是函数调用，定义时立即执行
@router.get("/api/users")
@staticmethod
async def get_users(): ...
```

| | Java 注解 | Python 装饰器 |
| --- | --- | --- |
| 本质 | **元数据**（贴标签） | **函数**（包装函数） |
| 何时生效 | 运行时靠**框架扫描** | **定义时立即执行** |
| 需要框架 | 必须（Spring等） | 不需要，纯 Python 语法 |
| 能修改行为 | 不能，只提供信息 | 能，直接包裹/替换函数 |

### 装饰器 = 套娃

```python
# 最简单的理解
@router.get("/health")
async def health():
    return {"status": "ok"}

# 等价于：
async def health():
    return {"status": "ok"}
health = router.get("/health")(health)  # ← 装饰器就是函数调用！
```

> **提示：** 比喻：Java 注解 = 在快递箱上贴"易碎"标签（快递员看到才轻拿轻放）。Python 装饰器 = 给快递箱外面套气泡膜（不需要快递员配合，本身就防摔）。

### 项目中常见的 4 种装饰器

#### 1. @router.get / @router.post — 路由注册

```python
@router.get("/health")      # 访问 GET /health 时执行此函数
async def health(): ...
```

#### 2. @staticmethod — 静态方法

```python
class ArticleService:
    @staticmethod               # 不需要 self，等价 Java 的 static
    def _to_item(art): ...
```

#### 3. @asynccontextmanager — 上下文管理器

```python
@asynccontextmanager
async def session_scope():
    async with async_session_factory() as session:
        try:
            yield session        # ← 暂停，把 session 交出去用
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# yield 的意思：暂停在这里，等 async with 块执行完再回来
```

#### 4. @property — 把方法伪装成属性

```python
class User:
    @property
    def full_name(self):
        return f"{self.first} {self.last}"

user.full_name    # 不用加括号！看起来像属性
# Java: user.getFullName()  // 必须加括号
```

### 执行顺序

```python
@A
@B
@C
def func(): pass

# 等价于：func = A(B(C(func)))
# 从下往上套娃，调用时从外往内执行
# 像穿衣服：先穿内衣(C)，再穿衬衫(B)，最后穿外套(A)
```

### 自己写一个装饰器（= Java AOP @Around）

```text
# Python — 5 行搞定
import functools

def log_method(func):
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        result = await func(*args, **kwargs)
        print(f"{func.__name__} 完成")
        return result
    return wrapper

@log_method
async def analyze(article_id): ...

// Java 同样效果需要：
// 1. 定义 @LogMethod 注解
// 2. 写 @Aspect 切面类
// 3. 配置 AOP 扫描
// 4. 确保 Bean 被 Spring 管理
```

### 面试答法

> **提示：** 面试一句话：Java 注解是元数据，本身不执行逻辑，需要框架（如 Spring AOP）在运行时读取并处理。Python 装饰器是函数，在定义时就直接包裹原函数，不依赖任何框架。

---

## 14 · GIL 全局解释器锁

### GIL 是什么？

Python 进程里有一把全局锁，同一时刻**只有一个线程**能执行 Python 代码。

```text
// Java — 4 个线程真正同时跑在 4 个 CPU 核心上
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> compute(1));  // 核心 1
pool.submit(() -> compute(2));  // 核心 2
pool.submit(() -> compute(3));  // 核心 3
pool.submit(() -> compute(4));  // 核心 4 → 4 任务并行

# Python — 4 个线程，但同一时间只有 1 个执行
t1 = threading.Thread(target=compute, args=(1,))
t2 = threading.Thread(target=compute, args=(2,))
t3 = threading.Thread(target=compute, args=(3,))
t4 = threading.Thread(target=compute, args=(4,))
# 实际在 1 个核心上轮流跑，时间 ≈ 4 倍
```

### 为什么有 GIL？（图书馆比喻）

Python 用**引用计数**管理内存——每个对象记录有多少变量指向它，为 0 就删。

比如图书馆每本书有借阅计数器。如果两人同时还书：

```python
# 张三和李四同时还书，同时看到计数器是 2
# 张三：2 - 1 = 1，写入 → 1
# 李四：2 - 1 = 1，写入 → 1  ← 应该是 0！（数据竞争）

# 解决方案：在大门口放保安（GIL），同一时间只能一个人操作计数器
```

**Java 没这个问题**，因为 Java 用 GC（垃圾回收），不用引用计数。

### GIL 的影响

| 场景 | GIL 影响 | 解决方案 |
| --- | --- | --- |
| **CPU 密集**（计算、加密） | 致命！多线程没用 | 用多进程 multiprocessing |
| **I/O 密集**（网络、DB） | 没影响！等I/O时释放GIL | asyncio 或多线程都行 |

> **提示：** 我们项目不受影响：Web 服务 99% 的时间在等 I/O（等 MySQL、等 HTTP），且我们用的是 async（协程），压根没用多线程，GIL 完全不相关。

### CPU 并行的方案

```python
# 方案 1：多进程（每个进程有自己的 GIL，互不影响）
from multiprocessing import Pool
with Pool(4) as pool:
    results = pool.map(compute, [1, 2, 3, 4])  # 真正并行！

# 方案 2：Uvicorn 多 Worker
uvicorn app.main:app --workers 4  # 4 个独立进程

# 方案 3：Python 3.13 实验性去掉 GIL（--disable-gil）
```

### 面试答法

> **提示：** 面试标准回答：GIL 是 CPython 的全局锁，同一时刻只有一个线程能执行 Python 字节码。对 I/O 密集型任务没影响（等 I/O 时会释放 GIL），对 CPU 密集型有影响，解决方案是 multiprocessing 多进程或 C 扩展。Python 3.13 开始实验性支持 free-threaded 模式。

---

## 15 · 部署：Uvicorn & Gunicorn

### Python 的 Tomcat

| Python | Java 对应 | 职责 |
| --- | --- | --- |
| FastAPI | Spring MVC | 写路由、处理请求 |
| Uvicorn | Tomcat（内嵌） | HTTP 服务器，处理网络 I/O |
| Gunicorn | Tomcat 线程池管理 | 管理多个 Worker 进程 |

> **提示：** 一句话：Uvicorn 是干活的（处理请求），Gunicorn 是管人的（管理进程）。开发用 Uvicorn 就够，生产加 Gunicorn 或 `--workers`。

### 启动方式

```bash
# 开发环境 — 单进程 + 热重载
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

# 生产环境方式1 — Uvicorn 多进程（简单够用）
uvicorn app.main:app --host 0.0.0.0 --port 8001 --workers 4

# 生产环境方式2 — Gunicorn + Uvicorn（更成熟）
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8001
```

### 多进程架构

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
| --- | --- | --- |
| 进程 | 通常 1 个 JVM 进程 | 多个 Worker 进程 |
| 并发 | 1进程 × 200线程 | 4进程 × 每进程数千协程 |
| CPU | 200线程跑满多核 | 4进程跑满4核 |
| 内存 | 共享堆内存 | 每进程独立内存 |

> **提示：** 注意：多进程时数据库连接池翻倍。4 进程 × pool_size=10 = 40 个连接。Worker 数通常设为 CPU 核心数 × 2 + 1。

---

## 16 · 注释文化 & 生态读音

### 注释文化差异

| | Java | Python |
| --- | --- | --- |
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

> **提示：** Python 黄金法则：好的代码不需要注释来解释它**做什么**，只需要注释来解释它**为什么这样做**。

### Python 生态读音指南

| 单词 | 读法 | 来源 |
| --- | --- | --- |
| Uvicorn | 尤维康 (you-vi-corn) | UV + Unicorn（独角兽） |
| Gunicorn | 咕尼康 (goo-ni-corn) | Green Unix Unicorn |
| FastAPI | 法斯特 API | Fast + API |
| Pydantic | 派丹提克 (pie-DAN-tick) | Py + pedantic（学究的） |
| SQLAlchemy | S-Q-L 阿尔凯米 | SQL + Alchemy（炼金术） |
| asyncio | 阿辛克IO (a-SINK-ee-oh) | async + I/O |
| pytest | 派泰斯特 (pie-test) | Py + test |
| ASGI | 四个字母分开读 A-S-G-I | Async Server Gateway Interface |
| WSGI | 威斯基 (wiz-ghee) | Web Server Gateway Interface |

> **提示：** 独角兽文化：Python Web 服务器爱用独角兽命名（Gunicorn=Green Unicorn, Uvicorn=UV Unicorn）。就像 Java 生态爱用动物（Tomcat猫、Kafka卡夫卡、Camel骆驼）。

### 命名规范速记

| 元素 | Java（驼峰） | Python（蛇形） |
| --- | --- | --- |
| 方法 | findByEmail() | find_by_email() |
| 变量 | userName | user_name |
| 常量 | MAX_RETRY | MAX_RETRY（一样！） |
| 类名 | UserRepository | UserRepository（一样！） |
| 文件名 | UserRepository.java | user_repo.py |

> **提示：** 转换记忆法：把驼峰大写字母变成 _小写 → findByEmail → find_by_email → getUserById → get_user_by_id

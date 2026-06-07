---
title: "Java 转 Python（二）：框架实践"
description: "Pydantic、FastAPI、数据库操作、配置管理 — 用 Python 写后端的核心工具链"
pubDatetime: 2026-06-07T00:00:00.000Z
featured: false
tags:
  - Python
  - 实战
---

系列第二篇。第一篇讲了基础语法差异，这篇聚焦**框架与开发实践**。代码来自 PayReach 项目实战。

---

## 1. 函数与类

Java 里一切必须放在 class 里；Python 无状态逻辑直接写函数即可。

```text
# Python — 直接写函数
async def analyze_article(title: str, text: str) -> dict:
    client = _get_client()
    response = await client.chat(...)
    return response

// Java — 必须有 class
public class AiAnalyzer {
    public Map<String, Object> analyzeArticle(String title, String text) { ... }
}
```

> 有状态 → class；无状态纯函数 → 直接写函数。

### self = this

```text
# Python — self 必须写在参数第一位
class WeChatClient:
    def __init__(self, token: str):
        self.token = token
    async def close(self):
        await self._client.aclose()

// Java — this 隐式存在，可写可不写
public class WeChatClient {
    private String token;
    public WeChatClient(String token) { this.token = token; }
    public void close() { ... }
}
```

| 特性 | Java `this` | Python `self` |
|------|-------------|---------------|
| 声明位置 | 不出现在参数列表 | **必须**写在参数第一位 |
| 使用时 | 可写可不写 | **必须**写 `self.x` |

### @property = getter

```text
class Exhibition(BaseModel):
    name: str
    score: int = 0

    @property
    def recommendation_level(self) -> str:
        if self.score >= 90: return "强烈推荐"
        return "一般"

e.name                    # 直接访问，不需要 .getName()
e.recommendation_level    # @property，看起来像属性

// Java: e.getName(); e.getRecommendationLevel();
```

### 一个文件多个 class

```text
# Python wechat.py — 4 个类 + 常量，一个文件
USER_AGENT = "Mozilla/5.0 ..."
class WeChatAPIError(Exception): ...
class SessionExpiredError(WeChatAPIError): ...
class WeChatClient: ...
class WeChatLoginFlow: ...

// Java — 一个 public class 一个文件 → 4 个文件
```

---

## 2. 模块与导入

Python 的 `import` 粒度更细，可以导入函数、常量，不只是类。

```text
# Python
from app.services.wechat import WeChatClient          # class
from app.services.ai_analyzer import analyze_article   # 函数
from app.services.wechat import USER_AGENT             # 常量

// Java — 只能导入类
import com.example.services.WeChatClient;
```

| 特性 | Java | Python |
|------|------|--------|
| import 什么 | 只能 class | class / 函数 / 变量 / 常量 |
| 路径映射 | `com.example.Foo` → `Foo.java` | `app.services.wechat` → `wechat.py` |

### 函数内导入（避免循环依赖）

```python
async def _get_system_prompt() -> str:
    from app.deps import get_config_repo    # 延迟导入，打破 A↔B 循环
    val = await get_config_repo().get("ai_analyzer_prompt", "")
    return val
```

> 模块 import 时执行顶层代码；函数内 import 可延迟时机。

---

## 3. Pydantic 数据模型

Pydantic = Lombok + Jackson + Validator，三合一。

```text
# Python Pydantic
class Exhibition(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    date_range: str = ""            # 有默认值 = 选填
    score: int = 0
    first_article_id: Optional[int] = None

// Java 等价
@Data
public class Exhibition {
    private int id;
    @NotNull private String name;
    private String dateRange = "";
    private int score = 0;
    @Nullable private Integer firstArticleId;
}
```

| 特性 | Java | Pydantic |
|------|------|----------|
| getter/setter | Lombok `@Data` | 自动有，直接 `.name` |
| JSON 序列化 | Jackson | 内置 `.model_dump()` |
| 数据验证 | `@Valid` + `@NotNull` | 运行时自动验证 |
| 必填 | `@NotNull` | 不写默认值 = 必填 |

### Optional[int] 含义

```python
first_article_id: Optional[int] = None
# Optional[int] → 类型是 int 或 None
# = None        → 默认值 None，选填
# Python 3.10+: first_article_id: int | None = None
```

### BaseModel vs dataclass

```python
# BaseModel — 有运行时验证
class Exhibition(BaseModel):
    name: str; score: int
Exhibition(name=123, score="abc")  # ❌ ValidationError!

# dataclass — 无验证，内部可信数据用
@dataclass
class NewsSource:
    name: str = ""; id: int = 0
NewsSource(name=123, id="abc")     # ✅ 不报错，但数据是错的
```

> 可变默认值（list/dict）用 `field(default_factory=list)`，否则所有实例共享同一对象。

---

## 4. FastAPI 框架

FastAPI 对标 Spring Boot 轻量版。

### 路由（= @GetMapping）

```text
router = APIRouter(prefix="/api/wechat", tags=["登录"])

@router.get("/login/qrcode")
async def get_qrcode():
    return {"qrcode": "base64data..."}

// Java
@RestController @RequestMapping("/api/wechat")
public class LoginController {
    @GetMapping("/login/qrcode")
    public Map<String, Object> getQrcode() {
        return Map.of("qrcode", "base64data...");
    }
}
```

### 依赖注入（= @Autowired）

```text
@lru_cache
def get_article_repo() -> ArticleRepository:
    return ArticleRepository()

@lru_cache
def get_article_service() -> ArticleService:
    return ArticleService(article_repo=get_article_repo())  # 手动注入

@router.get("/articles")
async def list_articles(
    service: ArticleService = Depends(get_article_service),
):
    return await service.list_all()

// Java
@Service
public class ArticleService {
    @Autowired private ArticleRepository articleRepo;  // 自动
}
```

| 概念 | Spring | FastAPI |
|------|--------|---------|
| 定义 Bean | `@Bean` / `@Component` | `@lru_cache` 工厂函数 |
| 注入 | `@Autowired`（自动） | `Depends()`（显式） |
| 配置类 | `@Configuration` | `deps.py` 模块 |

### 中间件（= Filter）

```text
class AccessLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.monotonic()
        response = await call_next(request)
        elapsed = (time.monotonic() - start) * 1000
        logger.info("%s %s status=%d elapsed=%.0fms",
            request.method, request.url.path, response.status_code, elapsed)
        return response

app.add_middleware(AccessLogMiddleware)

// Java: OncePerRequestFilter → chain.doFilter(request, response)
```

| 启动步骤 | Spring Boot | FastAPI |
|----------|-------------|---------|
| 读配置 | `application.yml` | `load_dotenv()` |
| 注册路由 | `@ComponentScan` | `app.include_router()` |
| 生命周期 | `@PostConstruct` | `@app.on_event("startup")` |

---

## 5. 数据库操作

SQLModel + SQLAlchemy AsyncEngine，分层与 Spring Boot 一致。

### 引擎与连接池（= HikariCP）

```text
# db/engine.py
engine = create_async_engine(DATABASE_URL, pool_size=10, max_overflow=5, pool_recycle=3600)
async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

// Java
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(10);
DataSource ds = new HikariDataSource(config);
```

### 事务管理（= @Transactional）

```text
@asynccontextmanager
async def session_scope():
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback(); raise

async def change_password(user_id, old_pwd, new_pwd):
    new_hash = hash_password(new_pwd)          # 事务外（CPU密集）
    async with session_scope() as session:     # ← 开启事务
        await repo.update_password(user_id, new_hash, session=session)

// Java: @Transactional → Spring 自动 commit/rollback
```

### 表模型（= @Entity）

```text
class UserTable(SQLModel, table=True):
    __tablename__ = "users"
    id: Optional[int] = Field(default=None, primary_key=True)
    email: str = Field(max_length=100, sa_column_kwargs={"unique": True})
    password_hash: str = Field(max_length=255)
    role: Optional[str] = Field(default="user",
        sa_column=Column(SAEnum("admin", "user"), server_default="user"))

// Java MyBatis-Plus
@TableName("users")
public class UserEntity {
    @TableId(type = IdType.AUTO) private Integer id;
    private String email; private String passwordHash; private String role;
}
```

### Repository 层

```text
class UserRepository(BaseRepository):
    async def find_by_email(self, email, session=None):
        async with self._use_session(session) as s:
            stmt = select(UserTable).where(UserTable.email == email)
            row = (await s.exec(stmt)).first()
            return User.model_validate(row) if row else None

    async def update(self, user_id, name=None, session=None):
        async with self._use_session(session) as s:
            row = await s.get(UserTable, user_id)
            if name is not None: row.name = name  # 脏检查
            s.add(row); await s.flush()

// Java: userMapper.selectOne(...); user.setName(...); userMapper.updateById(user);
```

| 概念 | Java | Python（SQLModel） |
|------|------|---------------------|
| 连接池 | HikariCP | AsyncEngine |
| 事务 | `@Transactional` | `session_scope()` |
| 查询 | QueryWrapper | `select(Table).where(...)` |
| 更新 | `updateById()` | 改属性 + `s.add()` + `flush()` |
| 表定义 | `@TableName` | `SQLModel(table=True)` |

---

## 6. 配置管理

Pydantic Settings = `application.yml` + `@ConfigurationProperties`。

```python
class Settings(BaseSettings):
    mysql_host: str = "22.50.21.10"     # 有默认值 = 选填
    mysql_app_password: str              # 无默认值 = 必填
    ai_chat_api_key: str
    model_config = {"env_file": ".env"}

settings = Settings()  # 自动读取环境变量 + .env
```

```text
.env 文件                    →  Settings()              →  业务代码
MYSQL_APP_PASSWORD=xxx           settings.mysql_app_password    连数据库
AI_CHAT_API_KEY=sk-xxx           settings.ai_chat_api_key       调 AI
```

| 特性 | Spring Boot | Pydantic Settings |
|------|-------------|-------------------|
| 配置文件 | `application.yml` | `.env` |
| 必填检查 | `@NotNull` | 无默认值 = 必填 |
| 注入 | `@Value` | `from app.config import settings` |
| 多环境 | `application-dev.yml` | `.env.local` |

---

**下一篇**：[Java 转 Python（三）：进阶话题](/posts/java-to-python-3-advanced) — async/await 协程、装饰器 vs 注解、GIL、部署、AI Agent 等。

---
title: "Java 转 Python（一）：语言特性"
description: "基础语法、数据结构、装饰器、生成器、async/await、GIL — Python 和 Java 的核心差异"
pubDatetime: 2026-06-07T00:00:00.000Z
featured: true
tags:
  - Python
  - 实战
---

本篇用代码对比讲清六个核心差异。

---

## 1. 基础语法

Python 不需要声明类型，直接赋值即创建变量。

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

### 类型注解：可选 vs 必须

```text
# Python — 两种都合法
def greet(name: str) -> str:
    return f"Hello {name}"

def greet(name):                # 无注解也能跑
    return f"Hello {name}"

// Java — 类型必须写
public String greet(String name) {
    return "Hello " + name;
}
```

> FastAPI 特例：路由参数的类型注解不只是文档，FastAPI 用它做参数验证和 OpenAPI 生成。

### 命名规范

| 概念 | Java | Python |
|------|------|--------|
| 私有 | `private` 关键字 | `_` 前缀（约定，非强制） |
| 常量 | `static final` | 全大写 `MAX_SIZE = 100` |
| 类名 | PascalCase | PascalCase（相同） |
| 方法名 | camelCase | snake_case |
| 包/模块 | `com.example.pkg` | `app.services.xxx` |

### 字符串格式化

```text
# Python f-string
name, score = "CES", 95
msg = f"展会 {name} 评分 {score} 分"

// Java
String msg = String.format("展会 %s 评分 %d 分", name, score);
// Java 21+
String msg = STR."展会 \{name} 评分 \{score} 分";
```

---

## 2. 数据结构

### list = ArrayList

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
items[::-1]                          # 反转
```

### dict = HashMap

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

### 列表推导式

```text
# 基本形式
names = [e.name for e in exhibitions]

# 带条件
high = [e for e in exhibitions if e.score > 80]

# dict 推导式
pairs = {e.id: e.name for e in exhibitions}

// Java Stream 等价
exhibitions.stream()
    .filter(e -> e.getScore() > 80)
    .collect(Collectors.toList());
```

### `**` 解包

```text
data = {"name": "张三", "age": 25}

# 以下两行完全等价
user = CreateUser(**data)
user = CreateUser(name="张三", age=25)

# 项目实际：数据库行 → 对象
rows = await cur.fetchall()
return [Exhibition(**r) for r in rows]
```

`**` 把 dict 展开成关键字参数，Java 没有等价语法。

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

## 3. 装饰器 vs 注解

外观像，**本质完全不同**：Java 注解是元数据标签，Python 装饰器是函数调用。

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
|---|-----------|---------------|
| 本质 | **元数据**（贴标签） | **函数**（包装函数） |
| 何时生效 | 运行时靠**框架扫描** | **定义时立即执行** |
| 需要框架 | 必须（Spring 等） | 不需要，纯 Python 语法 |
| 能修改行为 | 不能，只提供信息 | 能，直接包裹/替换函数 |

### 装饰器 = 套娃

```text
@router.get("/health")
async def health():
    return {"status": "ok"}

# 等价于：
async def health():
    return {"status": "ok"}
health = router.get("/health")(health)  # 装饰器就是函数调用！
```

### 项目中常见的 4 种

```python
@router.get("/health")           # 1. 路由注册
async def health(): ...

class ArticleService:
    @staticmethod                # 2. 静态方法，不需要 self
    def _to_item(art): ...

@asynccontextmanager             # 3. 上下文管理器（事务）
async def session_scope():
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

class User:
    @property                    # 4. 方法伪装成属性
    def full_name(self):
        return f"{self.first} {self.last}"
# user.full_name  不用加括号，Java 要 getFullName()
```

### 自己写一个（= Java AOP @Around）

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

> 面试一句话：Java 注解是元数据，本身不执行逻辑，需要框架在运行时读取并处理。Python 装饰器是函数，定义时就直接包裹原函数，不依赖任何框架。

---

## 4. 生成器 / yield

`return` 一次性返回全部数据；`yield` 逐个产出，调用方按需消费，内存友好。

```text
# return — 一次性返回
def get_all():
    return db.query("SELECT * FROM articles")  # 全部加载到内存

# yield — 逐个产出，等价 Iterator 语法糖
def get_all():
    for row in db.stream("SELECT * FROM articles"):
        yield Article(**row)

for i in countdown(5): print(i)   # for 循环自动调用 __next__()

// Java 需要实现 Iterator 接口，hasNext() + next()

# 生成器表达式 — (x**2 for x in range(1_000_000)) 惰性求值
# 列表推导     — [x**2 for x in range(1_000_000)] 立即占内存
```

### 实际场景：SSE 推送

PayReach 抓取进度用 SSE 长连接，生成器逐条推送事件：

```python
@router.get("/crawl/events")
async def crawl_events():
    q = crawler.subscribe()

    async def event_stream():
        try:
            yield f"event: state\ndata: {json.dumps(crawler.crawl_state)}\n\n"
            while True:
                try:
                    msg = await asyncio.wait_for(q.get(), timeout=30)
                    yield f"event: {msg['event']}\ndata: {json.dumps(msg)}\n\n"
                except asyncio.TimeoutError:
                    yield ": keepalive\n\n"    # 心跳保活
        finally:
            crawler.unsubscribe(q)

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

| | return | yield |
|---|--------|-------|
| 返回时机 | 函数结束，一次性 | 每次 yield 暂停并产出 |
| 内存 | 全部数据在内存 | 按需，一条一条 |
| 适用 | 小数据集 | 大文件、流式 API、SSE |

---

## 5. async/await

Java 一个请求占一个线程；Python 一个请求是一个协程，单线程管数千并发。

```text
// Java — 一个请求 = 一个线程（Tomcat 默认 200 线程）
@GetMapping("/articles")
public List<Article> getArticles() {
    return articleMapper.selectList(null);  // 线程卡住等 DB
}

# Python — 一个请求 = 一个协程
@router.get("/articles")
async def get_articles():
    articles = await article_repo.find_list()  # 让出 CPU，DB 返回后继续
    return articles
```

| | Java 多线程 | Python async |
|---|------------|--------------|
| 等 DB 时 | 线程**阻塞**，啥也不干 | 协程**让出**，处理别的请求 |
| 并发模型 | 200 线程 = 200 并发 | 1 线程 + 数千协程 |
| 内存开销 | 每线程 ~1MB | 每协程 ~几 KB |
| CPU 密集 | 多核并行 | **单线程**，会卡住 |

`await` = 发请求 → 让出 CPU 处理别的协程 → 结果返回后继续。`asyncio.gather` 并行多个 await：

```text
# Python — 5 个查询并行执行
articles, total, tags, global_total, new_count = await asyncio.gather(
    article_repo.find_list(...),
    article_repo.count(...),
    article_repo.get_keyword_tags(...),
    article_repo.count(),
    article_repo.count_new(),
)

// Java 等价 — CompletableFuture
CompletableFuture.allOf(
    CompletableFuture.supplyAsync(() -> articleMapper.selectList(...)),
    CompletableFuture.supplyAsync(() -> articleMapper.count(...)),
    // ...
);
```

I/O 密集用 `async/await`；CPU 密集用 `multiprocessing`；简单脚本直接写同步函数。

---

## 6. GIL 全局解释器锁

CPython 进程里有一把全局锁，**同一时刻只有一个线程**能执行 Python 字节码。

```text
// Java — 4 个线程真正同时跑在 4 个核心上
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> compute(1));  // 核心 1
pool.submit(() -> compute(2));  // 核心 2
pool.submit(() -> compute(3));  // 核心 3
pool.submit(() -> compute(4));  // 核心 4

# Python — 4 个线程，但同一时间只有 1 个执行
t1 = threading.Thread(target=compute, args=(1,))
t2 = threading.Thread(target=compute, args=(2,))
# 实际在 1 个核心上轮流跑，时间 ≈ 4 倍
```

Python 用**引用计数**管理内存，多线程同时改计数器会数据竞争，GIL 保证同一时刻只有一个线程执行字节码。Java 用 GC，无此问题。

### 影响与解决方案

| 场景 | GIL 影响 | 解决方案 |
|------|----------|----------|
| **CPU 密集**（计算、加密） | 致命，多线程没用 | `multiprocessing` 多进程 |
| **I/O 密集**（网络、DB） | 没影响，等 I/O 时释放 GIL | `asyncio` 或多线程都行 |

```text
# 多进程 — 每个进程独立 GIL
from multiprocessing import Pool
with Pool(4) as pool: results = pool.map(compute, [1,2,3,4])

# 多 Worker — uvicorn app.main:app --workers 4
# Python 3.13 — 实验性 --disable-gil
```

---

## 下一篇

语言特性打底之后，下一篇进入实战层：**[Java 转 Python（二）：Web 框架与分层架构](/posts/java-to-python-2-framework)** — FastAPI 路由、依赖注入、SQLModel 数据库、Pydantic 数据模型。

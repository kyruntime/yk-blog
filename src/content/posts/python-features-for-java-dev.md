---
author: YK
pubDatetime: 2026-06-07T00:00:00.000Z
title: Python 语言特性：Java 开发者视角
slug: python-features-for-java-dev
featured: true
draft: false
tags:
  - python
  - java
  - 语言特性
description: 从 Java 转 Python，真正需要理解的 5 个核心特性：鸭子类型、装饰器、生成器、async/await、GIL。
---

王垠在《如何掌握所有的程序语言》中说：**重视语言特性，而不是语言**。作为一个 Java 转 Python 的开发者，我深以为然。

以下是 Python 相对 Java **真正不同的** 5 个核心特性。其余（变量、函数、类、循环）都只是语法换了件衣服。

## 1. 鸭子类型

Java 思考方式：这个对象**是什么类型**？→ 编译器检查接口

Python 思考方式：这个对象**能做什么**？→ 运行时有方法就行

```python
# 不需要接口声明，有 export() 方法就能传进来
def process(item):
    print(item.export())
```

Python 提供三种约束方式（由松到严）：
- 纯鸭子类型：零约束，有方法就能调
- `Protocol`：静态鸭子，类型检查器验证
- `ABC + @abstractmethod`：强约束，忘了实现直接报错

继承语法：`class Child(Parent):`（括号里放父类，没有 `extends` 关键字）

## 2. 装饰器

Java 的 `@注解` 只是元数据标记，需要框架反射读取。Python 的 `@装饰器` 直接执行，改变函数行为。

```python
@timer
def fetch_data():
    ...

# 等价于：fetch_data = timer(fetch_data)
```

装饰器 = 接收函数、返回新函数的高阶函数。5 行代码实现 Java 需要 AOP + 注解 + 切面类才能做到的事。

## 3. 生成器 / yield

`yield` = `return` 的分期付款版。

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a      # 给出一个值，暂停
        a, b = b, a + b
```

本质是 Java Iterator 的语法糖，省掉手写 `hasNext()`/`next()`/状态变量。

实际场景：SSE 流式推送、大文件逐行读取、上下文管理器。

## 4. async/await

Java 用多线程并发（1000 请求 = 1000 线程）。Python 用单线程协程（1000 请求 = 1 线程 + 1000 协程轮流跑）。

```python
async def get_user(user_id):
    user = await db.query(user_id)        # 等 IO 时让出 CPU
    orders = await order_api.get(user_id)  # 别的协程趁机跑
    return combine(user, orders)
```

`await` = "我要等一个耗时操作，等的时候把 CPU 让给别人"。

## 5. GIL（全局解释器锁）

Python 多线程**不能真正并行执行** Python 代码。任意时刻只有 1 个线程在跑。

所以 Python Web 开发选了 async 协程这条路，绕过 GIL，用单线程也能高并发。

| 场景 | 方案 |
|------|------|
| Web 请求（IO 多） | async/await |
| CPU 密集计算 | 多进程 |

---

**结论**：设计模式是跨语言的，Python 只是语法更简洁。唯一真正的新思维是鸭子类型和协程模型。

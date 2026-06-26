---
title: "后端视角学前端（三）：Vue 工程实战"
description: "项目结构、Router、Pinia、前后端通信、Tailwind 布局、踩坑记录 — 从 0 到可维护的 Vue 项目"
pubDatetime: 2026-06-28T00:00:00.000Z
featured: true
tags:
  - 前端
  - Vue
  - 实战
---

第一篇讲生态，第二篇讲概念，这篇讲怎么把它们组合成一个可维护的项目。

---

## 1. 项目最佳结构

```text
src/
  views/           页面级组件（和路由一一对应）
  components/      可复用 UI 组件
  composables/     逻辑函数（useXxx）
  stores/          Pinia 全局状态
  router/          路由配置
  utils/           工具函数（日期格式化、HTTP 封装）
  types/           TypeScript 类型定义
  assets/          静态资源
  App.vue          根组件
  main.ts          入口文件
```

**各目录职责对比**：

| 目录 | 放什么 | 后端类比 |
|---|---|---|
| views | 页面组件（对应路由） | Controller |
| components | 可复用 UI 组件 | — |
| composables | 业务逻辑函数 | Service |
| stores | 全局共享状态 | 全局缓存 |
| utils | 工具函数 | utils 包 |
| types | 类型定义 | Model / Schema |

层级关系：`路由 → views → components → composables`。页面组装组件，组件调用逻辑。

---

## 2. Vue Router

把 URL 映射到组件，前端的"路由表"。

```typescript
const routes = [
  { path: '/login',       component: LoginPage, meta: { public: true } },
  { path: '/accounts',    component: AccountsPage },
  { path: '/exhibitions/:id', component: ExhibitionDetailPage },
]
```

**路由守卫** — 每次跳转前检查权限：

```typescript
router.beforeEach(async (to) => {
  if (to.meta.public) return true       // 公开页直接放行
  if (!auth.isLoggedIn) return '/login'  // 未登录跳登录页
  if (to.meta.requiresAdmin && !isAdmin) return '/'  // 权限不足
})
```

**后端类比**：路由守卫 ≈ FastAPI 的 `Depends()` 中间件做权限校验。

**导航方式**：

```typescript
router.push('/accounts')                              // 跳转
router.push({ path: '/articles', query: { id: '1' } }) // 带参数
router.back()                                          // 后退
```

---

## 3. Pinia Store vs Composable

| | Pinia Store | Composable |
|---|---|---|
| 状态 | **全局单例**，所有组件共享 | **每次调用创建新实例** |
| 适用 | 登录用户信息、全局列表 | 搜索筛选、表单逻辑 |
| 持久性 | 应用生命周期 | 组件生命周期 |
| 后端类比 | Redis 缓存 | Service 实例 |

```typescript
// Store — 全局唯一，哪里调都是同一份数据
const store = useArticleStore()
store.accounts  // 页面 A 和页面 B 看到的是同一个列表

// Composable — 每个组件独立状态
const { filterAddedBy } = useArticleSearch()  // 每个组件有自己的筛选条件
```

**判断标准**：数据需要跨组件/跨页面共享 → Store。否则 → Composable。

---

## 4. Tailwind CSS 布局

90% 的布局靠 Flex 解决：

```html
<!-- 水平排列 + 居中 + 间距 -->
<div class="flex items-center gap-4">
  <span>左边</span>
  <span>右边</span>
</div>

<!-- 垂直排列 -->
<div class="flex flex-col gap-2">
  <div>上</div>
  <div>下</div>
</div>
```

**典型的全局布局**：

```html
<div class="flex h-screen">
  <Sidebar class="w-52" />
  <div class="flex flex-1 flex-col">
    <TabBar />
    <main class="flex-1 overflow-auto">
      <RouterView />
    </main>
  </div>
</div>
```

Tailwind 颜色系统：`{属性}-{颜色}-{深浅}`，数字越大越深。

```text
bg-blue-50   → 最浅蓝
bg-blue-500  → 标准蓝
bg-blue-900  → 最深蓝
text-gray-600 → 深灰文字
```

---

## 5. 前后端通信

三种方式，各有适用场景：

**普通请求** — 最常用，一问一答：

```typescript
const data = await http.get('/api/articles')
```

**轮询** — 定时请求，适合非实时场景：

```typescript
onMounted(() => {
  loadStatus()
  timer = setInterval(loadStatus, 10000)  // 每 10 秒
})
onUnmounted(() => clearInterval(timer))
```

**SSE（Server-Sent Events）** — 后端主动推送，适合实时场景：

```typescript
// 前端
const es = new EventSource('/api/crawl/progress')
es.onmessage = (e) => updateProgress(JSON.parse(e.data))

// 后端（FastAPI）
async def event_stream():
    while True:
        data = await queue.get()
        yield f"data: {json.dumps(data)}\n\n"

return StreamingResponse(event_stream(), media_type="text/event-stream")
```

| 方式 | 方向 | 实时性 | 适用 |
|---|---|---|---|
| 普通请求 | 前端主动 | 按需 | 列表加载、表单提交 |
| 轮询 | 前端主动 | 秒级 | 队列状态面板 |
| SSE | 后端主动 | 实时 | 抓取进度、日志流 |

---

## 6. 测试策略

| 层级 | 工具 | 测什么 | ROI |
|---|---|---|---|
| 单元测试 | Vitest | 纯函数/逻辑 | 高（简单场景） |
| 组件测试 | Vue Test Utils | 单个组件交互 | 中 |
| E2E 测试 | Playwright | 完整用户流程 | 最高 |

**中小项目推荐**：只写 E2E 测试。一个 E2E 测试覆盖前后端完整链路，比 5 个单元测试更有效。

---

## 7. 踩坑记录

### watch 执行顺序

`watch(A)` 里修改了 B，但 B 上也有 `watch(B)` 会清空 A 的关联值——两个 watch 打架。

**解决**：用一个 flag 临时禁止某个 watch 执行：

```typescript
let _skip = false
watch(filterAddedBy, () => {
  if (_skip) return
  // 检查并清空 filterAccountId
})

function setFilter(accountId) {
  _skip = true
  filterAddedBy.value = lookupAddedBy(accountId)
  filterAccountId.value = accountId
  nextTick(() => { _skip = false })
}
```

### v-show 组件的状态保持

`v-show` 隐藏的组件不会销毁，`onMounted` 只执行一次。如果需要在 tab 切换时启停逻辑（如轮询），用 `active` prop + `watch` 控制：

```typescript
const props = defineProps<{ active?: boolean }>()

watch(() => props.active, (visible) => {
  visible ? startPoll() : stopPoll()
})
```

### 跨页面传参

组件 emit 在 `v-show` 环境下不可靠（组件始终存在，事件绑定时机有坑）。**用 `router.push` + query 参数**更稳定：

```typescript
// 发送方
router.push({ path: '/articles', query: { account_id: '123' } })

// 接收方
watch(() => route.fullPath, () => {
  if (route.query.account_id) {
    setFilter(route.query.account_id)
  }
})
```

URL 参数是最可靠的跨组件通信方式——不依赖组件生命周期，刷新页面也不丢失。

---
title: "后端视角学前端（二）：Vue 核心概念"
description: "响应式系统、组件化、Composition API、生命周期 — 用后端类比理解 Vue 的设计哲学"
pubDatetime: 2026-06-27T00:00:00.000Z
featured: true
tags:
  - 前端
  - Vue
  - 实战
---

上篇讲了前端生态全貌，这篇深入 Vue 的核心概念。所有示例来自真实项目。

---

## 1. 响应式系统

Vue 最核心的能力：**改了数据，UI 自动更新。**

三个基本构件：

```typescript
// ref — 创建响应式变量
const count = ref(0)
count.value++  // UI 自动更新

// computed — 自动计算（依赖变了就重算）
const double = computed(() => count.value * 2)

// watch — 监听变化后执行副作用
watch(count, (newVal) => {
  console.log('count 变了:', newVal)
})
```

**后端类比**：`ref` 像数据库字段，`computed` 像视图（View），`watch` 像触发器（Trigger）。

**实际场景**：用户选了"张三" → `filterAddedBy` 变了 → `filteredAccounts`（computed）自动重算 → 下拉列表 UI 自动更新。整条链路是自动的。

---

## 2. 组件化

页面拆成独立的小块，像搭积木。

```text
AccountsPage（容器）
  ├── ArticlesTab（文章列表）
  │     ├── Pagination（分页器）
  │     └── ArticleListItem（文章卡片）
  └── AccountManageTab（数据来源）
        └── Pagination（分页器）
```

组件通信只有三种方式：

| 方式 | 方向 | 适用场景 |
|---|---|---|
| **props** | 父 → 子 | 传配置给子组件 |
| **emit** | 子 → 父 | 子组件通知父组件 |
| **Store** | 任意 → 任意 | 跨组件/跨页面共享数据 |

```html
<!-- 父组件传 props -->
<Pagination :total="100" :page="1" />

<!-- 子组件发 emit -->
<button @click="$emit('change', 2)">下一页</button>
```

**黄金法则**：能用 props/emit 解决的，不用 Store。Store 是"大炮"，父子通信用"小刀"就够。

---

## 3. Composition API

Vue 3 的核心写法，按**功能**而非**类型**组织代码。

**Options API（Vue 2）** — 按类型分：data 一块、methods 一块、computed 一块，搜索相关的代码散落各处。

**Composition API（Vue 3）** — 按功能聚合，封装成可复用函数：

```typescript
// composables/useArticleSearch.ts
export function useArticleSearch() {
  const filterAddedBy = ref('')
  const filterAccountId = ref('')

  const filteredAccounts = computed(() => {
    if (!filterAddedBy.value) return store.accounts
    return store.accounts.filter(a => a.added_by === filterAddedBy.value)
  })

  function doSearch() { /* ... */ }

  return { filterAddedBy, filterAccountId, filteredAccounts, doSearch }
}
```

组件里一行调用就能用：

```typescript
const { filterAddedBy, filteredAccounts, doSearch } = useArticleSearch()
```

**后端类比**：Composable ≈ Python 的 Service 类，封装一组相关的业务逻辑。

---

## 4. 单文件组件（SFC）

一个 `.vue` 文件包含三部分：

```html
<script setup lang="ts">
// 逻辑
const count = ref(0)
</script>

<template>
  <!-- UI -->
  <button @click="count++">{{ count }}</button>
</template>

<style scoped>
/* 样式（只作用于当前组件） */
button { color: blue; }
</style>
```

逻辑、UI、样式放一起，开发体验好。用 Tailwind 后 `<style>` 块通常省略。

---

## 5. 生命周期

组件从创建到销毁经历的阶段，只需记住两个：

| 钩子 | 时机 | 用途 |
|---|---|---|
| `onMounted` | 组件渲染完成 | 调 API 加载数据 |
| `onUnmounted` | 组件被销毁 | 清理定时器、关闭连接 |

```typescript
onMounted(async () => {
  await store.load()              // 页面加载好了，拿数据
})

onUnmounted(() => {
  eventSource?.close()            // 离开页面，关闭 SSE 连接
})
```

**后端类比**：`onMounted` ≈ `@app.on_event("startup")`，`onUnmounted` ≈ `@app.on_event("shutdown")`。

---

## 6. 条件渲染：v-if vs v-show

| | v-if | v-show |
|---|---|---|
| false 时 | 组件销毁，移出 DOM | 组件还在，CSS 隐藏 |
| true 时 | 重新创建 | 取消隐藏 |
| 状态保留 | 不保留 | 保留 |

**选择依据**：频繁切换用 `v-show`（避免反复创建销毁），很少切换用 `v-if`（减少初始渲染量）。

Tab 页切换用 `v-show`——切走再切回来，之前的筛选条件还在。

---

## 7. 双向绑定：v-model

输入框和变量自动双向同步：

```html
<input v-model="searchQuery" />
<!-- 等价于 -->
<input :value="searchQuery" @input="searchQuery = $event.target.value" />
```

用户输入 → 变量更新 → 用到变量的 computed/watch 自动触发 → UI 更新。

---

## 8. 虚拟 DOM

Vue 不直接操作真实页面。数据变了 → 先在 JS 对象（虚拟 DOM）上算出差异 → 只把变化的部分更新到真实页面。

```text
旧虚拟 DOM：[文章A, 文章B(未读), 文章C]
新虚拟 DOM：[文章A, 文章B(已读), 文章C]
                         ↑ 只有这里变了

Diff 结果：只改文章 B 的 class → 一行 DOM 操作搞定
```

好处：避免整个列表重建，性能好。Vue 3 还做了编译时优化——静态节点标记后跳过 diff。

# 健身动作百科设计 Spec

## 基本信息

| 项目 | 内容 |
| --- | --- |
| 名称 | 健身动作百科（fitness-exercises） |
| 定位 | 双人私人健身房动作查询工具 |
| 用户 | 本人 + 老婆，健身房手机查阅 |
| 数据源 | [exercises-dataset](https://github.com/hasaneyldrm/exercises-dataset)（1,324 动作） |
| 部署 | Vercel（前端）+ Supabase（账号 + 收藏） |

## 核心需求

> 在健身房用手机快速查动作、看动图、读中文说明，各自收藏常练动作。

### 做什么

- 浏览、搜索、筛选 1,324 个健身动作
- 查看动图 GIF 和中文步骤说明
- 轻量账号登录，各自独立收藏列表

### 不做什么（YAGNI）

- 训练计划编排
- 组数/重量记录
- 离线/PWA 缓存
- 社交、分享
- 多语言界面（界面中文，数据用 `instructions.zh`）
- 暗色模式（MVP）

---

## 技术方案

### 选型：Nuxt 3 + Vercel + Supabase

| 层 | 技术 | 理由 |
| --- | --- | --- |
| 前端框架 | Nuxt 3 | SSR/路由成熟，Vercel 一等支持，移动端体验好 |
| 样式 | Tailwind CSS | 快速实现清爽简约 UI |
| 部署 | Vercel | 用户已有 Vercel 使用经验，Git push 自动部署 |
| 账号 + 数据 | Supabase Auth + Postgres | 免费额度够用，Auth + RLS 开箱即用，无需自建登录 |
| 动作数据 | 静态 JSON + public 媒体 | 1,324 条只读数据，无需数据库 |

### 架构

```
手机浏览器
    ↓
Vercel（Nuxt 3 前端）
    ├── 静态资源：exercises.json + images/ + videos/
    └── @supabase/supabase-js
            ├── Auth（邮箱+密码登录）
            └── Postgres favorites 表（RLS 隔离）
```

### 数据流

1. **动作数据**：构建时从 exercises-dataset 导入 `public/`，首页 fetch 一次 JSON 缓存在内存，搜索/筛选纯前端
2. **收藏**：登录后通过 Supabase Client 读写 `favorites` 表，用 `exercise_id` 回查本地 JSON
3. **媒体**：列表页只加载 JPG 缩略图；详情页懒加载 GIF

### 媒体策略

- 1,324 张 JPG + 1,324 个 GIF 放入 `public/images/` 和 `public/videos/`
- 列表：`loading="lazy"` 缩略图
- 详情：进入页面后再加载 GIF，缩略图作占位
- 保留 `attribution` 字段展示：© Gym visual — https://gymvisual.com/

---

## UI/UX 设计

### 视觉风格：清爽简约

| 元素 | 值 |
| --- | --- |
| 背景 | `#F5F5F7` |
| 卡片 | 白色，圆角 12px |
| 文字 | `#1D1D1F` |
| 强调色 | `#0071E3` |
| 字体 | `-apple-system` 系统字体栈 |
| 间距 | 移动端边距 16px，卡片间距 12px |

### 页面结构（4 页 + 底部导航）

#### 1. 首页 `/`

```
┌──────┬──────────────────────────┐
│ 全部 │  🔍 搜索框    [筛选⏷]   │
│ 胸   ├──────────────────────────┤
│ 背   │  ┌─────┐  ┌─────┐       │
│ 腿   │  │ 🏋️  │  │ 🏋️  │       │
│ 肩   │  │卧推  │  │深蹲  │       │
│ 手臂 │  └─────┘  └─────┘       │
│ 核心 │  ...无限滚动...          │
│ 有氧 │                          │
└──────┴──────────────────────────┘
│  [首页]    [收藏]    [我的]      │
└─────────────────────────────────┘
```

- **左侧分类栏**：宽 72–80px，固定；当前项浅蓝底 + 左侧色条高亮
- **右侧列表**：搜索框 sticky；双列卡片网格，每页 20 条下拉加载
- **器械筛选**：搜索框旁筛选图标，点击弹出底部抽屉（杠铃/哑铃/徒手/绳索…）
- **分类映射**（英文 category → 中文显示）：

| category | 中文 |
| --- | --- |
| chest | 胸 |
| back | 背 |
| upper legs | 腿 |
| shoulders | 肩 |
| upper arms | 手臂 |
| waist | 核心 |
| lower legs | 小腿 |
| lower arms | 前臂 |
| cardio | 有氧 |
| neck | 颈 |

#### 2. 详情页 `/exercise/[id]`

- 顶部：GIF 动图自动循环（居中，约 60% 屏宽），懒加载
- 动作名称（大号标题）
- 标签行：部位 · 器械 · 目标肌
- 右上角心形收藏按钮（未登录弹窗引导登录）
- 中文步骤说明（编号列表）
- 底部：Gym visual 署名
- 左上角返回，保留列表滚动位置

#### 3. 收藏页 `/favorites`

- 需登录，未登录显示引导
- 布局同首页（左分类 + 右列表），数据为收藏子集
- 空状态：「还没有收藏，去首页看看吧」

#### 4. 我的 `/profile`

- 邮箱 + 密码登录表单
- 登录后显示邮箱、登出按钮
- 无注册入口（账号手动创建）

### 底部导航

- 固定底栏，高度 56px + `safe-area-inset-bottom`
- 三个 Tab：首页 / 收藏 / 我的
- 图标 + 文字标签

### 移动端适配

| 要点 | 做法 |
| --- | --- |
| 视口 | `width=device-width, initial-scale=1` |
| 主屏添加 | `apple-mobile-web-app-capable` meta |
| 触控区域 | 最小 44×44px |
| 搜索 | 输入即过滤，300ms 防抖 |
| 筛选 | 部位左侧单选；器械底部抽屉多选 |
| 收藏 | 乐观更新，失败回滚 + Toast |
| 加载 | 骨架屏占位 |

### 交互细节

- 搜索：匹配 `name`（英文）和 `instructions.zh`（中文关键词）
- 筛选组合：左侧部位 AND 器械抽屉选择 AND 搜索关键词
- 收藏：点击心形 → `insert`/`delete` → 即时 UI 反馈

---

## 数据模型

### 静态数据：`exercises.json`

每条记录使用 exercises-dataset 原始结构，关键字段：

| 字段 | 用途 |
| --- | --- |
| `id` | 唯一标识，如 `"0001"` |
| `name` | 英文动作名，搜索用 |
| `category` / `body_part` | 部位筛选 |
| `equipment` | 器械筛选 |
| `target` | 目标肌，详情展示 |
| `secondary_muscles` | 协同肌，详情展示 |
| `instructions.zh` | 中文步骤说明 |
| `image` | 缩略图路径 |
| `gif_url` | 动图路径 |
| `attribution` | 版权署名 |

### Supabase：`favorites` 表

```sql
create table favorites (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users(id) on delete cascade,
  exercise_id text not null,
  created_at  timestamptz default now(),
  unique(user_id, exercise_id)
);

alter table favorites enable row level security;

create policy "select_own" on favorites
  for select using (auth.uid() = user_id);

create policy "insert_own" on favorites
  for insert with check (auth.uid() = user_id);

create policy "delete_own" on favorites
  for delete using (auth.uid() = user_id);
```

### 账号策略

1. Supabase Dashboard 手动创建两个账号
2. 关闭公开注册（`Enable email signups` = off）
3. 登录页仅邮箱 + 密码，无注册按钮
4. 密码重置走 Supabase 邮件

### 环境变量

```env
NUXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NUXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
```

仅使用 `anon key`；数据隔离由 RLS 保证。

---

## 项目结构

独立仓库 `fitness-exercises/`（与 blog 项目无关）：

```
fitness-exercises/
├── public/
│   ├── data/exercises.json
│   ├── images/              # 1324 张缩略图
│   └── videos/              # 1324 个 GIF
├── pages/
│   ├── index.vue            # 首页
│   ├── exercise/[id].vue    # 详情
│   ├── favorites.vue        # 收藏
│   └── profile.vue          # 我的
├── components/
│   ├── ExerciseCard.vue
│   ├── CategorySidebar.vue
│   ├── SearchBar.vue
│   ├── FilterDrawer.vue
│   └── BottomNav.vue
├── composables/
│   ├── useExercises.ts      # JSON 加载/搜索/筛选
│   └── useFavorites.ts      # Supabase 收藏 CRUD
├── nuxt.config.ts
├── package.json
└── .env.example
```

---

## 错误处理

| 场景 | 处理 |
| --- | --- |
| JSON 加载失败 | 全页错误提示 + 重试按钮 |
| 未登录点收藏 | 弹窗「请先登录」→ 跳转 `/profile` |
| 收藏网络失败 | Toast 提示，回滚 UI 状态 |
| 搜索/筛选无结果 | 空状态「没有找到相关动作」 |
| GIF 加载慢 | 缩略图占位，GIF 加载完淡入 |
| 登录失败 | 表单下方红色错误文案 |

---

## 测试策略

| 范围 | 方法 |
| --- | --- |
| 搜索/筛选逻辑 | 单元测试 `useExercises` |
| 收藏 CRUD | mock Supabase client |
| 移动端布局 | Playwright E2E，375px 视口 |
| 部署验证 | 手机真机：搜索 → 详情 → 收藏全流程 |

---

## 部署流程

1. `git clone` exercises-dataset，复制 `data/`、`images/`、`videos/` 到 `public/`
2. 创建 GitHub 仓库 `fitness-exercises`，推送代码
3. Vercel 连接仓库，框架自动识别 Nuxt
4. 配置环境变量 `NUXT_PUBLIC_SUPABASE_URL` 和 `NUXT_PUBLIC_SUPABASE_ANON_KEY`
5. Supabase 建表 + RLS + 手动创建两个账号
6. Push 触发自动部署
7. 手机浏览器打开网址 →「添加到主屏幕」

---

## 许可证注意

- exercises-dataset 代码/数据结构：MIT
- 图片和 GIF：© [Gym visual](https://gymvisual.com/)，需在详情页保留 attribution
- 私人使用，不对外公开分发媒体

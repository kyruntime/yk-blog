---
title: "Playwright 实战指南：从 E2E 测试到浏览器自动化"
description: "多浏览器支持、自动等待、Codegen 录制、网络拦截、Trace Viewer 调试 — 一篇搞定现代 Web 测试"
pubDatetime: 2026-06-22T00:00:00.000Z
featured: true
tags:
  - 测试
  - 实战
---

## 为什么是 Playwright

| 对比项 | Playwright | Cypress | Puppeteer |
|--------|-----------|---------|-----------|
| 多浏览器 | Chromium + Firefox + WebKit | 仅 Chromium（实验性 Firefox） | 仅 Chromium |
| 语言 | TS/JS/Python/Java/C# | JS/TS | JS/TS |
| 并发 | 原生 worker sharding | 串行（付费并发） | 需自己管理 |
| 网络拦截 | 内置 `page.route()` | `cy.intercept()` | 内置 |
| 组件测试 | 支持 React/Vue/Svelte | 支持 | 不支持 |

Playwright 由 Microsoft 维护，核心优势是**跨浏览器 + 自动等待 + 完整的调试工具链**。

## 5 分钟起步

```bash
# 初始化项目
npm init playwright@latest

# 运行测试
npx playwright test

# 打开 UI 模式（实时观看）
npx playwright test --ui
```

生成的目录结构：

```
tests/
  example.spec.ts
playwright.config.ts
```

## CLI 常用参数

`npx playwright test` 后面可以跟路径和选项，灵活组合：

```bash
npx playwright test [测试路径] [选项]
```

### 运行范围

```bash
# 跑指定目录
npx playwright test tests/e2e/ui/

# 跑单个文件
npx playwright test tests/e2e/ui/settings.spec.ts

# 按测试名称过滤（-g = grep）
npx playwright test -g "登录成功"

# 跑指定 project（对应 config 里的 projects 配置）
npx playwright test --project=chromium
```

### 输出控制

| 参数 | 效果 | 适用场景 |
|------|------|----------|
| `--reporter=list` | 逐行列出每个测试状态 | **日常开发推荐** |
| `--reporter=html` | 生成可交互的 HTML 报告 | 详细查看失败原因 |
| `--reporter=dot` | 每个测试一个点，极简 | CI 或大量测试 |
| `--reporter=line` | 单行覆盖式刷新进度 | 终端直接看（不要管道到 tail） |
| `--reporter=json` | JSON 格式输出 | 程序化处理结果 |

> **避坑**：`--reporter=line | tail -60` 会因管道缓冲导致看不到实时进度，用 `--reporter=list` 替代。

### 调试利器

```bash
# 打开浏览器窗口，能看到操作过程
npx playwright test --headed

# 启动调试器，可以单步执行、设断点
npx playwright test --debug

# 打开图形界面，交互式选择和运行测试
npx playwright test --ui
```

### 执行控制

```bash
# 失败后重试 N 次
npx playwright test --retries=1

# 控制并发 worker 数
npx playwright test --workers=4

# 更新截图基准
npx playwright test --update-snapshots

# CI 分片（4 台机器各跑 1/4）
npx playwright test --shard=1/4
```

### 实战组合示例

```bash
# 日常开发：跑 UI 测试，实时看进度
npx playwright test tests/e2e/ui/ --reporter=list

# 调试单个失败的测试
npx playwright test tests/e2e/ui/settings.spec.ts --headed --debug

# CI 环境
npx playwright test --reporter=github --retries=1

# 快速验证某个功能
npx playwright test -g "展会列表" --reporter=list
```

## 核心概念

### 跨进程架构

理解 Playwright 的架构有助于理解为什么一切操作都是 `async/await`。

```
测试代码 (Node.js 进程)
      ↕ WebSocket
Playwright 内核
      ↕ Chrome DevTools Protocol
浏览器 (独立 OS 进程)
```

三个独立进程通过网络协议通信。当你写 `await page.click('#btn')` 时：

1. Node.js 进程通过 WebSocket 发送指令给浏览器
2. 浏览器执行 actionability checks（下面详述），然后点击
3. 浏览器通过 WebSocket 返回结果
4. Node.js 进程收到结果，继续下一行

这就是为什么 **每个操作都必须 `await`**——两个进程之间有网络往返。漏写 `await` 不会报错，但会导致操作顺序混乱，这是 flaky test 的常见根源。

### 自动等待 & Actionability Checks

Playwright 在执行操作前自动等待元素满足一组条件（称为 Actionability Checks），不需要手动 `waitForSelector`：

```typescript
await page.click('button[type="submit"]')
```

这一行背后，Playwright 会持续轮询直到以下全部满足：

| 检查项 | 含义 |
|--------|------|
| **Attached** | 元素存在于 DOM 中 |
| **Visible** | 不是 `display: none`、`visibility: hidden`、零尺寸 |
| **Stable** | 连续两帧位置不变（没在动画中） |
| **Receives Events** | 没被其他元素（overlay、modal）遮挡 |
| **Enabled** | 没有 `disabled` 属性 |

实际行为取决于元素状态：

| 场景 | 行为 |
|------|------|
| 元素不存在 | 持续等待直到超时 |
| 元素存在但 hidden | 等待它变可见，超时则报错 |
| 元素被 modal 遮住 | 等待遮挡物消失 |
| 元素 disabled | 等待它变 enabled |
| 元素 2 秒后才渲染 | 等 2 秒后立即操作 |
| 元素已就绪 | **毫秒级完成** |

关键认知：**选择器写对时操作极快，选择器写错时会等满超时才报错**。这就是为什么 E2E 测试有时一秒完事，有时卡半分钟。

### 语义化定位器

优先使用语义化定位，测试不会因为 class 名变化而崩：

```typescript
// 推荐：语义化
await page.getByRole('button', { name: '提交' }).click()
await page.getByLabel('邮箱').fill('test@example.com')
await page.getByPlaceholder('搜索...').fill('keyword')
await page.getByText('登录成功').isVisible()

// 可用但不推荐：CSS 选择器
await page.locator('.btn-primary').click()
```

### 超时层级

Playwright 有**四层独立超时**，很容易混淆：

| 超时类型 | 控制范围 | 默认值 | 配置项 |
|----------|---------|--------|--------|
| **Test Timeout** | 单个测试用例的总时限 | 30 秒 | `timeout` |
| **Expect Timeout** | `expect()` 断言等待 | 5 秒 | `expect.timeout` |
| **Action Timeout** | `click()`/`fill()` 等操作 | 无（继承 test） | `use.actionTimeout` |
| **Navigation Timeout** | `goto()`/`waitForURL()` | 无（继承 test） | `use.navigationTimeout` |

```typescript
// 这个最多等 5 秒（expect timeout）
await expect(page.getByText('成功')).toBeVisible()

// 这个最多等 30 秒（继承 test timeout）
await page.click('#submit')
```

可以按需独立配置，缩短反馈时间：

```typescript
export default defineConfig({
  timeout: 30_000,
  expect: { timeout: 5_000 },
  use: {
    actionTimeout: 10_000,
    navigationTimeout: 15_000,
  },
})
```

不设 actionTimeout/navigationTimeout 时它们共享 test timeout 的 30 秒上限。选择器写错时的等待时间 = 对应的超时值。

### 断言

Playwright 的断言是 **web-first** 的——会自动重试直到条件满足或超时（默认 5 秒）：

```typescript
import { expect } from '@playwright/test'

// ✅ web-first：自动等待 + 重试
await expect(page.getByRole('heading')).toHaveText('Dashboard')
await expect(page.locator('.item')).toHaveCount(5)
await expect(page).toHaveURL(/.*dashboard/)
await expect(page.getByRole('alert')).not.toBeVisible()

// ❌ 不要这样写：isVisible() 不会等待，立即返回 true/false
expect(await page.getByText('welcome').isVisible()).toBe(true)
```

**Soft Assertions**：普通 `expect()` 失败即终止测试。用 `expect.soft()` 可以收集所有失败，测试结束后一次性报告：

```typescript
await expect.soft(page.getByTestId('username')).toHaveText('Alice')
await expect.soft(page.getByTestId('role')).toHaveText('Admin')
await expect.soft(page.getByTestId('status')).toHaveText('Active')
// 即使第一个失败，后面两个也会执行，最后统一报告
```

## Codegen：录制生成代码

```bash
npx playwright codegen https://your-app.com
```

打开浏览器后你的每次点击、输入都会实时生成测试代码。适合快速生成定位器，然后手动优化。

## Page Object 模式

将页面操作封装为类，测试代码只关心业务逻辑：

```typescript
// pages/login.ts
export class LoginPage {
  constructor(private page: Page) {}

  async login(email: string, password: string) {
    await this.page.getByLabel('邮箱').fill(email)
    await this.page.getByLabel('密码').fill(password)
    await this.page.getByRole('button', { name: '登录' }).click()
  }
}

// tests/auth.spec.ts
test('登录成功跳转首页', async ({ page }) => {
  const loginPage = new LoginPage(page)
  await page.goto('/login')
  await loginPage.login('admin@test.com', '123456')
  await expect(page).toHaveURL('/dashboard')
})
```

## 网络拦截

`page.route()` 是**提前注册拦截规则**——在打开页面之前设好"路障"，页面发出请求时会被劫持。这不是在等待请求，而是预设陷阱。

**为什么需要：** 你想测试"列表为空时页面显示什么"，但数据库里有真实数据删不掉。Mock 让你在不改后端的情况下，控制前端看到的数据。

```typescript
// 场景 1：测试空数据状态
await page.route('**/api/exhibitions', route => {
  route.fulfill({ body: JSON.stringify({ items: [], total: 0 }) })
})
await page.goto('/exhibitions')
await expect(page.getByText('暂无展会')).toBeVisible()

// 场景 2：测试服务端报错时前端的错误提示
await page.route('**/api/companies', route => {
  route.fulfill({ status: 500, body: 'Internal Server Error' })
})

// 场景 3：测试接口很慢时的 loading 状态
await page.route('**/api/accounts', async route => {
  await new Promise(r => setTimeout(r, 3000))
  route.fulfill({ body: JSON.stringify([]) })
})

// 拦截并修改请求头
await page.route('**/api/**', async route => {
  const headers = { ...route.request().headers(), 'x-test': 'true' }
  await route.continue({ headers })
})

// 录制 HAR 文件（首次录制真实请求，后续从文件回放）
await page.routeFromHAR('tests/data/api.har', { update: true })
```

> `page.route()` = 事先设置拦截规则（伪造返回）  
> `page.waitForResponse()` = 等待真实请求完成（不拦截，只观察）

## Trace Viewer：调试利器

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // 失败时自动录制 trace
  },
})
```

查看 trace：

```bash
npx playwright show-trace test-results/xxx/trace.zip
```

Trace 包含每一步的 DOM 快照、网络请求、Console 日志，可以逐帧回放。

## 截图与视觉对比

```typescript
// 全页面截图
await page.screenshot({ path: 'screenshot.png', fullPage: true })

// 视觉回归（对比基准图）
await expect(page).toHaveScreenshot('homepage.png', {
  maxDiffPixelRatio: 0.01,
})
```

首次运行生成基准图，后续运行自动对比差异。

## 多上下文 & 多用户

```typescript
test('两个用户同时操作', async ({ browser }) => {
  const adminContext = await browser.newContext()
  const userContext = await browser.newContext()

  const adminPage = await adminContext.newPage()
  const userPage = await userContext.newPage()

  await adminPage.goto('/admin')
  await userPage.goto('/dashboard')
  // 两个用户互不干扰，各自独立的 cookie/storage
})
```

## CI 集成（GitHub Actions）

```yaml
- name: Run Playwright
  run: npx playwright test
- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: playwright-report
    path: playwright-report/
```

## 更多能力速查

| 功能 | 用法 | 场景 |
|------|------|------|
| 移动端模拟 | `devices['iPhone 14']` | 响应式测试 |
| 文件上传 | `page.setInputFiles()` | 表单测试 |
| 文件下载 | `page.waitForEvent('download')` | 导出功能 |
| iframe | `page.frameLocator('#frame')` | 嵌入内容 |
| Shadow DOM | `locator` 自动穿透 | Web Components |
| 视频录制 | `video: { mode: 'on' }` | 调试回放 |
| 并发分片 | `--shard=1/4` | CI 加速 |
| 组件测试 | `@playwright/experimental-ct-vue` | 单组件隔离测试 |
| 认证复用 | `storageState` | 跳过重复登录 |
| 重试 | `retries: 2` | 抗 flaky |

## 总结

Playwright 的设计哲学：**让测试像用户一样操作页面**。自动等待消除了 90% 的 flaky test，语义化定位让测试不再脆弱，Trace Viewer 让调试不再痛苦。

从 `npm init playwright@latest` 开始，10 分钟你就能跑起第一个测试。

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

## 核心概念

### 自动等待（Auto-waiting）

Playwright 在执行操作前自动等待元素满足条件（可见、可交互、稳定）：

```typescript
// 不需要手动 waitFor，直接操作
await page.click('button[type="submit"]')
// Playwright 会自动等待按钮可见 + 可点击 + 没有被遮挡
```

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

### 断言

```typescript
import { expect } from '@playwright/test'

await expect(page.getByRole('heading')).toHaveText('Dashboard')
await expect(page.locator('.item')).toHaveCount(5)
await expect(page).toHaveURL(/.*dashboard/)
await expect(page.getByRole('alert')).not.toBeVisible()
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

```typescript
// Mock API 返回
await page.route('**/api/users', async route => {
  await route.fulfill({
    status: 200,
    body: JSON.stringify([{ id: 1, name: 'Mock User' }]),
  })
})

// 拦截并修改请求
await page.route('**/api/**', async route => {
  const headers = { ...route.request().headers(), 'x-test': 'true' }
  await route.continue({ headers })
})

// 录制 HAR 文件
await page.routeFromHAR('tests/data/api.har', { update: true })
```

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

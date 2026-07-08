# 前端 e2e 探针脚本模板

> loop-eng ③建闸「运行时闸门」层的标准落地形式。前端/框架集成/部署配置类切片的运行时闸门，必须沉淀为**可重复跑的 e2e 脚本**（不只临时 browser-use 手动探针），断言可先红。
>
> 由来：某 V1 联调暴露 crypto-js ESM + 框架 baseURL `/mock` 白屏——build/tsc/3 轮 verifier 全漏，因运行时探针只临时 browser-use 没沉淀成可重复闸门。本模板把运行时探针固化成切片级闸门。

## 位置
`tests/e2e/<切片>.spec.ts`（或 `e2e/`，随项目约定）

## 工具选择
- **playwright**（推荐）：标准化、可重复跑、断言丰富、跨浏览器、CI 友好
- **browser-use**：联调时手动探针，**不算闸门**（不可重复跑、无断言沉淀）——仅用于调试/一次性验证
- **httpx/curl**：API 层探针（前端切片需配合渲染层断言，单独不够）

## 脚本模板（playwright）

```ts
// tests/e2e/<切片>.spec.ts
import { test, expect } from '@playwright/test';

const BASE = 'http://localhost:5174'; // 前端 dev，proxy /api → 后端

test('<切片> 页面渲染 + 数据加载', async ({ page }) => {
  await page.goto(`${BASE}/<route>`);
  // ① 非白屏：#app 挂载（Vue/React 真启动，防 crypto-js/baseURL 这类崩溃）
  await expect(page.locator('#app')).not.toBeEmpty();
  // ② 关键文本：页面标题/指标值（替换为该切片期望渲染的内容）
  await expect(page.locator('body')).toContainText('<期望文本，如「首页驾驶舱」>');
  // ③ API 经 proxy 返数据（前后端连通 + 包装格式正确）
  const res = await page.request.get(`${BASE}/api/v1/<endpoint>`);
  const body = await res.json();
  expect(body.code).toBe('0');
  expect(body.result).toBeDefined();
});
```

## 闸门三断言（缺一不算运行时闸门）
1. **非白屏**：`#app` 非空 —— 防 main.ts 崩溃（依赖 ESM、框架配置、style 路径）
2. **关键文本**：body 含期望内容 —— 防数据没加载（API 失败/walRequest 解包错/baseURL 不对）
3. **API 经 proxy**：`code:'0'` + result —— 防前后端断连/包装格式不对

## 闸门要求
- **先红**：先写期望文本（页面还没渲染该内容时），`npx playwright test` 失败
- **后绿**：实现后跑通
- **可重复**：CI/本地任意时刻可跑，不依赖手动 browser-use
- **随切片沉淀**：每个前端切片⑤验证时写/更新对应 e2e 脚本，不只联调时临时探

## 何时用
- **前端切片**（页面/组件/路由）③建闸：必须叠加
- **框架集成配置**（vite/proxy/依赖/baseURL/主题）改动：必须叠加——正是 crypto-js/`/mock` 这类 bug 的防线
- **纯后端切片**：不需要（后端有单元/集成测试强闸门）

## 与联调闸门的区别
- **e2e 探针脚本** = 切片级运行时闸门（每个前端切片⑤，可重复跑）
- **联调闸门** = loop 级端到端（终止前主链路走通，可手动 browser-use 驱动全链路）

两者互补：e2e 探针保证每个切片运行时绿，联调闸门保证整体链路通。

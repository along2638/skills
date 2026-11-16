# 测试规范

## 1. 测试分层

| 层级 | 工具 | 范围 | 耗时 | 是否进 CI |
|---|---|---|---|---|
| 单元测试 | Vitest | 工具函数、composable、store | 毫秒 | 是 |
| 组件测试 | Vitest + @vue/test-utils + jsdom | 组件渲染、交互、事件 | 百毫秒 | 是 |
| 接口层测试 | Vitest + MSW | 请求封装、错误映射 | 百毫秒 | 是 |
| 端到端 | Playwright | 登录、下单等主流程 | 秒级 | 是（独立阶段） |

原则：

- 分层的目的是**让大部分逻辑用最快的方式测到**。业务逻辑放在 composable 与 store 里，就能用毫秒级单测覆盖，而不必为每个判断启动浏览器。
- 组件测试只验证"渲染正确、事件发出、状态显示正确"，不测 Element Plus 自身行为。
- E2E 只覆盖少数关键链路，多了会变成半小时跑不完的负担，且失败原因难定位。

## 2. 什么必须测

| 对象 | 必须覆盖 |
|---|---|
| 工具函数 | 正常值、边界值（空、0、超长）、异常输入 |
| composable | 状态变化、清理逻辑、参数变化的竞态 |
| store action | 成功、业务错误、网络错误、分页参数正确性 |
| 表单校验 | 必填、长度、格式、跨字段校验 |
| 权限指令 | 有权限渲染、无权限移除 |
| 请求层 | 解包 `data`、`code !== 0` 抛错、401 处理、超时映射 |
| E2E | 登录、核心业务主流程、关键错误提示 |

## 3. 什么不必测

- 纯样式与布局（视觉回归另行考虑，不用单测）。
- 第三方组件的内部行为（`el-table` 排序、`el-form` 校验本身）。
- 只做透传的包装组件（`<slot />` 转发），价值极低。
- 常量与类型定义。

判断标准：**这条测试失败时，能明确指向一个业务规则被破坏吗？** 不能，就不写。

## 4. Vitest 配置

```ts
// vitest.config.ts
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/__tests__/**/*.spec.ts'],
    exclude: ['e2e/**', 'node_modules/**'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{ts,vue}'],
      exclude: ['src/**/*.d.ts', 'src/main.ts', 'src/types/**'],
    },
  },
})
```

```ts
// src/test/setup.ts
import { config } from '@vue/test-utils'
import { beforeEach, vi } from 'vitest'

// Element Plus 的部分组件依赖 ResizeObserver，jsdom 没有
globalThis.ResizeObserver = class {
  observe() {}
  unobserve() {}
  disconnect() {}
}

config.global.stubs = {
  transition: false,
  'transition-group': false,
}

beforeEach(() => {
  localStorage.clear()
  vi.clearAllMocks()
})
```

`globals: true` 让 `describe` / `it` / `expect` 无需导入，但需要同步在 `tsconfig` 的 `types` 中加入 `vitest/globals`，否则类型检查报错。

## 5. 组件测试

```ts
// src/components/order/__tests__/OrderStatusTag.spec.ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import OrderStatusTag from '../OrderStatusTag.vue'

describe('OrderStatusTag', () => {
  it('待支付状态显示对应文案与类型', () => {
    const wrapper = mount(OrderStatusTag, {
      props: { status: 'WAIT_PAY' },
    })

    expect(wrapper.text()).toBe('待支付')
    expect(wrapper.find('.el-tag--warning').exists()).toBe(true)
  })

  it('未知状态兜底显示', () => {
    const wrapper = mount(OrderStatusTag, {
      props: { status: 'UNKNOWN' as never },
    })

    expect(wrapper.text()).toBe('-')
  })
})
```

带交互与异步的组件：

```ts
// src/views/order/__tests__/OrderListView.spec.ts
import { flushPromises, mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import OrderListView from '../OrderListView.vue'
import { orderApi } from '@/api/order'

vi.mock('@/api/order')

describe('OrderListView', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('加载成功后渲染订单行', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({
      records: [
        { id: '1', orderNo: 'SO202601001', amount: '99.00', status: 'PAID', createTime: '2026-01-01 10:00:00' },
      ],
      total: 1,
      pageNum: 1,
      pageSize: 10,
      pages: 1,
    })

    const wrapper = mount(OrderListView, { global: { plugins: [createPinia()] } })
    await flushPromises()

    expect(wrapper.text()).toContain('SO202601001')
    expect(wrapper.text()).toContain('¥99.00')
  })

  it('加载失败时展示错误态与重试按钮', async () => {
    vi.mocked(orderApi.fetchOrderList).mockRejectedValue(new Error('网络异常'))

    const wrapper = mount(OrderListView)
    await flushPromises()

    expect(wrapper.text()).toContain('重新加载')
  })
})
```

要点：

- `flushPromises()` 等异步队列清空，不要用 `setTimeout` 等固定时间。
- 断言用户可见的结果（文本、`aria` 属性、是否存在），不要断言内部 `data`。
- 需要真实 Element Plus 组件时不要全局 stub；只 stub 掉与测试目标无关的重组件（如 `el-dialog` 的 teleport）。
- 触发事件用 `await wrapper.find('button').trigger('click')`，不要直接调用组件内部方法。

## 6. store 测试

```ts
// src/stores/__tests__/order.spec.ts
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { useOrderStore } from '../order'
import { orderApi } from '@/api/order'

vi.mock('@/api/order')

describe('useOrderStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('拉取列表后写入数据与总数', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({
      records: [{ id: '1', orderNo: 'SO1', amount: '1.00', status: 'PAID', createTime: '' }],
      total: 1,
      pageNum: 1,
      pageSize: 10,
      pages: 1,
    })

    const store = useOrderStore()
    await store.fetchList()

    expect(store.list).toHaveLength(1)
    expect(store.total).toBe(1)
    expect(store.loading).toBe(false)
  })

  it('失败时保留旧数据并记录错误', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({
      records: [{ id: '1', orderNo: 'SO1', amount: '1.00', status: 'PAID', createTime: '' }],
      total: 1,
      pageNum: 1,
      pageSize: 10,
      pages: 1,
    })
    const store = useOrderStore()
    await store.fetchList()

    vi.mocked(orderApi.fetchOrderList).mockRejectedValue(new Error('超时'))
    await store.fetchList({ pageNum: 2 })

    expect(store.list).toHaveLength(1)
    expect(store.error).toBe('超时')
  })

  it('翻页时传递正确的分页参数', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({ records: [], total: 0, pageNum: 3, pageSize: 20, pages: 0 })
    const store = useOrderStore()

    await store.fetchList({ pageNum: 3, pageSize: 20 })

    expect(orderApi.fetchOrderList).toHaveBeenCalledWith(
      expect.objectContaining({ pageNum: 3, pageSize: 20 }),
    )
  })
})
```

## 7. 接口层 Mock（MSW）

单元测试里直接 `vi.mock` 接口模块即可；需要验证"请求层如何解析响应"时用 MSW 拦截真实 `fetch`/`xhr`：

```ts
// src/test/msw.ts
import { http, HttpResponse } from 'msw'
import { setupServer } from 'msw/node'

export const server = setupServer(
  http.get('/api/v1/orders', ({ request }) => {
    const url = new URL(request.url)
    const pageNum = Number(url.searchParams.get('pageNum') ?? 1)
    return HttpResponse.json({
      code: 0,
      message: 'success',
      traceId: 'test-trace',
      data: { records: [], total: 0, pageNum, pageSize: 10, pages: 0 },
    })
  }),
  http.post('/api/v1/orders/:id/cancel', () =>
    HttpResponse.json({ code: 30102, message: '订单已发货，无法取消', traceId: 'test-trace', data: null }),
  ),
)
```

```ts
// src/test/setup.ts 中挂载
import { server } from './msw'
import { afterAll, afterEach, beforeAll } from 'vitest'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

`onUnhandledRequest: 'error'` 很重要：避免测试因为漏 mock 而静默失败。

## 8. E2E（Playwright）

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  use: {
    baseURL: 'http://127.0.0.1:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  webServer: {
    command: 'pnpm dev',
    url: 'http://127.0.0.1:5173',
    reuseExistingServer: !process.env.CI,
  },
})
```

```ts
// e2e/order.spec.ts
import { expect, test } from '@playwright/test'

test('已发货订单不能取消', async ({ page }) => {
  await page.goto('/order/list')

  const row = page.getByRole('row', { name: /SO202601001/ })
  await expect(row.getByRole('button', { name: '取消订单' })).toHaveCount(0)
})
```

规则：

- 选择器优先 `getByRole` / `getByLabel` / `getByText`，其次 `data-testid`；禁止用 CSS class 或 `nth-child`，样式一改就全挂。
- 禁止 `waitForTimeout`。用 `expect(locator).toBeVisible()` 这类自动等待断言。
- 登录态用 `storageState` 复用，每个用例不必重复走登录流程。
- 用例之间互不依赖，数据由接口或 fixture 准备，不靠"上一条用例留下的数据"。
- E2E 失败要能定位：开启 `trace` 与失败截图，CI 保留制品。

## 9. 覆盖率

- 语句与分支覆盖率目标 70%，`utils` 与 `stores` 核心逻辑 ≥ 80%。
- 不追求 100%：为覆盖而写的断言往往没有业务含义，反而增加维护成本。
- 新增代码的覆盖率不低于整体水平；覆盖率下降超过 2% 需要在评审中说明原因。
- 覆盖率是参考指标，不是质量证明：大量 `expect(x).toBeTruthy()` 的测试能把覆盖率做高但毫无价值。

## 10. 测试数据与稳定性

- 用工厂函数构造测试数据，避免每个用例手写完整对象：

```ts
// src/test/factories.ts
import type { OrderItem } from '@/types/order'

export function createOrder(overrides: Partial<OrderItem> = {}): OrderItem {
  return {
    id: '1',
    orderNo: 'SO202601001',
    amount: '99.00',
    status: 'PAID',
    createTime: '2026-01-01 10:00:00',
    ...overrides,
  }
}
```

- 时间相关逻辑用 `vi.setSystemTime(new Date('2026-01-01T10:00:00+08:00'))` 固定，不依赖机器当前时间。
- 不使用真实手机号、身份证、姓名；用明显虚构的数据。
- 测试必须稳定：不访问网络、不依赖执行顺序、不产生共享副作用（`beforeEach` 清理 `localStorage` 与 pinia）。
- 禁止提交 `it.skip` / `describe.skip`，除非附工单号与恢复计划。
- 断言失败信息要有业务含义：`expect(store.error).toBe('超时')` 优于 `expect(store.error).toBeTruthy()`。

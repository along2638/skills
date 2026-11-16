# 状态管理与数据层规范

## 1. 状态放哪里

| 状态范围 | 载体 | 示例 |
|---|---|---|
| 只在一个组件内用 | `ref` / `reactive` | 弹窗显隐、表单输入、局部 loading |
| 父子/兄弟组件 | props / emits / `defineModel` | 筛选条与列表 |
| 局部多层嵌套 | `provide` / `inject` | 表单上下文、只读模式开关 |
| 跨页面共享 | Pinia store | 登录用户、权限、全局字典 |
| 服务端数据 | 请求层 + 页面/ store | 列表、详情、统计数据 |

判断标准：**能否用"组件树 + props"表达**。能，就不要进 Pinia。Pinia 的代价是引入全局可变状态：谁都能改、难以追溯、页面间会互相污染（A 页面改了筛选条件，B 页面打开时带着旧条件）。

常见误用：把每个页面的列表数据都塞进全局 store，导致进入页面必须先 `reset()`，忘记重置就串数据。这类数据留在页面里，只在"需要跨页面保留"时才放 store。

## 2. Pinia 规范

```ts
// src/stores/order.ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'
import { orderApi, type OrderQuery } from '@/api/order'
import type { OrderItem } from '@/types/order'

export const useOrderStore = defineStore('order', () => {
  const list = ref<OrderItem[]>([])
  const total = ref(0)
  const loading = ref(false)
  const error = ref<string | null>(null)
  const query = ref<OrderQuery>({ pageNum: 1, pageSize: 10 })

  const isEmpty = computed(() => !loading.value && !error.value && list.value.length === 0)

  async function fetchList(patch: Partial<OrderQuery> = {}) {
    query.value = { ...query.value, ...patch }
    loading.value = true
    error.value = null
    try {
      const result = await orderApi.fetchOrderList(query.value)
      list.value = result.list
      total.value = result.total
    } catch (e) {
      // 失败时保留旧数据，只标记错误，避免页面闪空
      error.value = e instanceof Error ? e.message : '加载失败'
    } finally {
      loading.value = false
    }
  }

  async function cancel(id: string) {
    await orderApi.cancelOrder(id)
    await fetchList()
  }

  function reset() {
    list.value = []
    total.value = 0
    error.value = null
    query.value = { pageNum: 1, pageSize: 10 }
  }

  return { list, total, loading, error, query, isEmpty, fetchList, cancel, reset }
})
```

规则：

- **一个文件一个 store**，文件名 `useXxxStore.ts`，`defineStore` 的第一个参数与文件名一致，方便 devtools 定位。
- 用 setup 语法（`ref` + 函数），不用 options 语法。前者类型推断更好，也便于抽 composable。
- state 只存"最小事实"：列表与总数，不存 `pageCount`（由 `total / pageSize` 派生）、不存 `hasNext`。
- action 内**不发 toast、不跳路由、不弹确认框**。UI 反馈是页面的事，store 里弹 toast 会让单测无法断言，也会在不需要提示的场景误弹。
- action 内捕获错误时，要么写进 `error` 状态，要么往上抛，不要既捕获又静默。
- 提供 `reset()`：跨页面共享的 store 在被复用前必须能回到初始态。
- 需要持久化时用 `pinia-plugin-persistedstate`，只持久化白名单字段，禁止整个 store 落 `localStorage`。

```ts
// 持久化白名单
persist: {
  key: 'order-filter',
  pick: ['query'],
}
```

## 3. 请求层

### 3.1 统一实例

```ts
// src/api/request.ts
import axios, { AxiosError, type AxiosRequestConfig } from 'axios'
import { ElMessage } from 'element-plus'
import { storage } from '@/utils/storage'
import type { ApiResponse } from '@/types/api'

export class BizError extends Error {
  readonly code: number
  readonly traceId: string

  constructor(code: number, message: string, traceId = '') {
    super(message)
    this.name = 'BizError'
    this.code = code
    this.traceId = traceId
  }
}

export function isBizError(error: unknown): error is BizError {
  return error instanceof BizError
}

export function isAbortError(error: unknown): boolean {
  return error instanceof DOMException && error.name === 'AbortError'
}

const instance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
})

instance.interceptors.request.use((config) => {
  const token = storage.getToken()
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  config.headers['X-Request-Id'] = crypto.randomUUID()
  return config
})

instance.interceptors.response.use(
  (response) => {
    const body = response.data as ApiResponse<unknown>
    // 文件下载等非标准响应直接返回
    if (!body || typeof body.code !== 'number') {
      return response.data
    }
    if (body.code !== 0) {
      throw new BizError(body.code, body.message || '请求失败', body.traceId)
    }
    return body.data
  },
  async (error: AxiosError<ApiResponse<unknown>>) => {
    if (isAbortError(error)) {
      return Promise.reject(error)
    }

    const status = error.response?.status
    const body = error.response?.data

    if (status === 401) {
      await handleUnauthorized()
      return Promise.reject(new BizError(401, '登录已过期，请重新登录'))
    }

    if (status === 403) {
      ElMessage.error('没有操作权限')
      return Promise.reject(new BizError(403, '没有操作权限'))
    }

    if (body && typeof body.code === 'number') {
      return Promise.reject(new BizError(body.code, body.message || '请求失败', body.traceId))
    }

    if (error.code === 'ECONNABORTED') {
      return Promise.reject(new BizError(-1, '请求超时，请稍后重试'))
    }

    return Promise.reject(new BizError(-1, '网络异常，请检查网络后重试'))
  },
)

let redirecting = false

async function handleUnauthorized() {
  if (redirecting) {
    return
  }
  redirecting = true
  storage.clearToken()
  // 动态导入避免 request ←→ router 的循环依赖
  const { router } = await import('@/router')
  const current = router.currentRoute.value
  await router.replace({ path: '/login', query: { redirect: current.fullPath } })
  redirecting = false
}

export async function request<T>(config: AxiosRequestConfig): Promise<T> {
  return instance.request<unknown, T>(config)
}
```

要点：

- 响应拦截器**解包 `data`**，业务层只拿数据，不再写 `res.data.data`。
- 业务错误统一抛 `BizError`，页面只需判断 `isBizError` 就能拿到可展示的 `message`。
- 401 用一个 `redirecting` 标记防止并发请求触发多次跳转（这是最常见的线上问题：页面同时发 5 个请求，跳 5 次登录页）。
- 请求层**不 import router 的静态依赖**，用动态 `import()` 打破循环依赖。
- 不在这里统一弹所有错误 toast：列表页希望就地展示错误态，写操作希望 toast，策略应由页面决定。这里只在 403 这类明确场景给出提示。

### 3.2 接口文件

```ts
// src/api/order.ts
import { request } from './request'
import type { PageResult } from '@/types/api'
import type { OrderDetail, OrderItem, OrderQuery } from '@/types/order'

export const orderApi = {
  fetchOrderList(params: OrderQuery, signal?: AbortSignal) {
    return request<PageResult<OrderItem>>({ url: '/order/list', method: 'get', params, signal })
  },
  fetchOrderDetail(id: string) {
    return request<OrderDetail>({ url: `/order/${id}`, method: 'get' })
  },
  cancelOrder(id: string, reason: string) {
    return request<void>({ url: `/order/${id}/cancel`, method: 'post', data: { reason } })
  },
}
```

- 一个业务域一个文件，导出一个 `xxxApi` 对象；不导出零散函数，调用处能看出归属。
- 入参类型与后端契约一一对应，字段名不改写、不做驼峰/下划线转换（统一由后端返回驼峰，或由 `request` 层统一转换）。
- 需要取消的请求接受 `signal` 参数，页面负责创建与销毁 `AbortController`。

### 3.3 类型三段式

```ts
// src/types/order.ts
export type OrderStatus = 'WAIT_PAY' | 'PAID' | 'SHIPPED' | 'COMPLETED' | 'CANCELLED'

export interface OrderItem {
  id: string
  orderNo: string
  amount: string
  status: OrderStatus
  createTime: string
}

export interface OrderDetail extends OrderItem {
  receiverName: string
  receiverPhone: string
  remark: string
}

export interface OrderQuery {
  pageNum: number
  pageSize: number
  keyword?: string
  status?: OrderStatus
}
```

列表项与详情分开：列表接口通常只返回部分字段，用同一个类型会导致详情页字段"看起来有、运行时 undefined"。详情用 `extends` 复用公共字段。

## 4. 错误处理分层

| 层 | 职责 | 写法 |
|---|---|---|
| 请求层 | 网络错误、HTTP 状态、业务码 → `BizError` | 拦截器统一转换 |
| store | 记录 `error` 状态、保留旧数据 | `try/catch` 写状态 |
| 页面 | 展示错误态、提供重试、写操作 toast | `isBizError` 判断后提示 |
| 全局 | 未捕获异常上报 | `app.config.errorHandler` |

```ts
// 页面写法
async function handleCancel(id: string) {
  try {
    await orderStore.cancel(id)
    ElMessage.success('已取消')
  } catch (error) {
    ElMessage.error(isBizError(error) ? error.message : '取消失败，请稍后重试')
  }
}
```

规则：

- 业务错误给用户看 `message`（后端已写好可读文案），技术细节只进控制台与日志。
- 同一错误不要提示两次（store 里 toast + 页面再 toast）。
- 写操作失败必须保留用户已填内容，不清空表单。

## 5. 三态处理

任何异步数据都有三种状态，必须全部处理：

```vue
<template>
  <div v-if="store.loading" class="order-list__skeleton">
    <el-skeleton :rows="6" animated />
  </div>
  <el-result v-else-if="store.error" icon="error" :title="store.error">
    <template #extra>
      <el-button type="primary" @click="store.fetchList()">重新加载</el-button>
    </template>
  </el-result>
  <el-empty v-else-if="store.isEmpty" description="暂无订单" />
  <el-table v-else :data="store.list" row-key="id">
    <!-- columns -->
  </el-table>
</template>
```

反模式：

- 只有 `v-loading`，接口失败时页面一片空白，用户不知道该刷新还是等待。
- 空列表直接用表格的"暂无数据"，没有引导（如"调整筛选条件"）。
- 错误态用 `ElMessage` 提示后不留任何可重试入口。

## 6. 缓存与刷新

- `keep-alive` 缓存的列表页，返回时必须刷新：在 `onActivated` 中调用 `fetchList()`，并用标记避免首次进入重复请求。

```ts
let mounted = false

onMounted(async () => {
  await store.fetchList()
  mounted = true
})

onActivated(() => {
  if (mounted) {
    store.fetchList()
  }
})
```

- 列表详情往返时，优先用"页面内状态 + `keep-alive`"保留滚动位置与筛选条件，而不是把筛选条件塞进全局 store。
- 需要跨页面立即同步的数据（如全局未读数），由 store 统一维护，改动后刷新而非各处缓存。
- 不做前端自建缓存层：接口本身有缓存头，重复请求用请求合并或防抖控制，避免"本地缓存与服务端不一致"这类难查的问题。

## 7. 表单数据

```ts
const formRef = ref<FormInstance>()
const form = reactive<OrderUpdatePayload>({ remark: '', status: 'PAID' })
const submitting = ref(false)

async function handleSubmit() {
  if (submitting.value) {
    return
  }
  const valid = await formRef.value?.validate().catch(() => false)
  if (!valid) {
    return
  }
  submitting.value = true
  try {
    await orderApi.update(form)
    ElMessage.success('保存成功')
    close()
  } catch (error) {
    ElMessage.error(isBizError(error) ? error.message : '保存失败')
  } finally {
    submitting.value = false
  }
}
```

规则：

- 提交前统一 `validate()`，并处理校验失败分支（`catch` 返回 `false`），不能 `await validate()` 后直接提交。
- `submitting` 状态同时用于按钮 `:loading` 与函数入口判断，双保险防重复提交。
- 表单数据用 `reactive`，编辑场景用 `Object.assign(form, detail)`；不要直接指向接口返回对象（否则取消编辑时原数据已被改）。
- 重置用显式函数重新赋值，不依赖 `resetFields()` 的隐式行为（它只重置初始值，动态改过的字段不生效）。

## 8. 时间与金额

```ts
import dayjs from 'dayjs'

export function formatDateTime(value?: string): string {
  return value ? dayjs(value).format('YYYY-MM-DD HH:mm:ss') : '-'
}

export function formatAmount(value?: string): string {
  if (!value) {
    return '-'
  }
  return new Intl.NumberFormat('zh-CN', {
    style: 'currency',
    currency: 'CNY',
    minimumFractionDigits: 2,
  }).format(Number(value))
}
```

- 后端返回时间字符串，前端只格式化不改语义；需要时区转换时明确标注，默认按后端返回的本地时间展示。
- 金额后端返回字符串（避免精度丢失），前端只做展示格式化，**不做加减乘除**；需要统计时由后端出结果。
- 禁止手写 `date.getMonth() + 1` 拼接日期，闰年与补零问题都会出错。

## 9. 本地存储

```ts
// src/utils/storage.ts
const PREFIX = 'admin:'

export const storage = {
  getToken(): string {
    return localStorage.getItem(`${PREFIX}token`) ?? ''
  },
  setToken(token: string) {
    localStorage.setItem(`${PREFIX}token`, token)
  },
  clearToken() {
    localStorage.removeItem(`${PREFIX}token`)
  },
  get<T>(key: string, fallback: T): T {
    const raw = localStorage.getItem(PREFIX + key)
    if (!raw) {
      return fallback
    }
    try {
      return JSON.parse(raw) as T
    } catch {
      return fallback
    }
  },
  set<T>(key: string, value: T) {
    localStorage.setItem(PREFIX + key, JSON.stringify(value))
  },
  remove(key: string) {
    localStorage.removeItem(PREFIX + key)
  },
}
```

- 键名统一加项目前缀，避免同域多应用互相覆盖。
- `localStorage` 只存 token、主题、表格列配置这类非敏感偏好；禁止存用户隐私、手机号、身份信息。
- `sessionStorage` 用于多标签页隔离的场景（如不同账号调试），默认不用。
- token 存储方案的安全权衡见 `references/security.md`。

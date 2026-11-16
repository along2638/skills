# 完整功能切片：订单列表页

本示例覆盖一个页面从类型定义到测试的完整实现，可直接作为新增业务模块的模板。基线：Vue 3.4 + TypeScript 5.4 + Vite 5 + Pinia 2 + Element Plus 2.6。

## 1. 需求与文件清单

需求：订单列表，支持关键字搜索、状态筛选、时间范围筛选、分页、行内查看详情与取消订单（已发货订单不可取消）。

| 文件 | 职责 |
|---|---|
| `src/types/order.ts` | 契约类型：状态、列表项、详情、查询参数 |
| `src/api/order.ts` | 接口函数，唯一与后端交互的地方 |
| `src/stores/order.ts` | 列表数据、加载/错误状态、取消动作 |
| `src/composables/useOrderFilters.ts` | URL 查询参数 ↔ 筛选条件，参数解析与边界兜底 |
| `src/components/order/OrderStatusTag.vue` | 状态到文案与颜色的映射 |
| `src/views/order/OrderListView.vue` | 页面编排：筛选区、表格、分页、三态 |
| `src/router/modules/order.ts` | 路由注册与权限元信息 |
| `src/stores/__tests__/order.spec.ts` | store 单元测试 |
| `src/composables/__tests__/useOrderFilters.spec.ts` | 参数解析测试 |
| `src/components/order/__tests__/OrderStatusTag.spec.ts` | 组件渲染测试 |

数据流：URL 是筛选条件的唯一来源 → 页面监听 `route.query` → 调 store → store 调 api → 渲染。**不在页面、store、URL 三处各存一份筛选条件**，这是列表页最常见的 bug 来源。

## 2. 类型定义

`src/types/order.ts`。状态用字面量联合而非 `enum`，与后端返回的字符串 code 一一对应。

```ts
export type OrderStatus = 'WAIT_PAY' | 'PAID' | 'SHIPPED' | 'COMPLETED' | 'CANCELLED'

export const ORDER_STATUS_LIST: readonly OrderStatus[] = [
  'WAIT_PAY',
  'PAID',
  'SHIPPED',
  'COMPLETED',
  'CANCELLED',
]

export function isOrderStatus(value: unknown): value is OrderStatus {
  return typeof value === 'string' && (ORDER_STATUS_LIST as readonly string[]).includes(value)
}

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
  startTime?: string
  endTime?: string
}

export interface PageResult<T> {
  records: T[]
  total: number
  pageNum: number
  pageSize: number
  pages: number
}
```

`isOrderStatus` 不是多余的：URL 参数是用户可改的字符串，直接 `as OrderStatus` 会让非法值进入业务逻辑，请求后端返回 400 却在前端看不出原因。

## 3. 接口层

`src/api/order.ts`。唯一的 HTTP 调用出口，页面与组件都不出现 `axios`。

```ts
import { request } from './request'
import type { OrderDetail, OrderItem, OrderQuery, PageResult } from '@/types/order'

export const orderApi = {
  fetchOrderList(params: OrderQuery, signal?: AbortSignal) {
    return request<PageResult<OrderItem>>({
      url: '/orders',
      method: 'get',
      params,
      signal,
    })
  },

  fetchOrderDetail(id: string) {
    return request<OrderDetail>({
      url: `/orders/${id}`,
      method: 'get',
    })
  },

  cancelOrder(id: string, reason: string) {
    return request<void>({
      url: `/orders/${id}/cancel`,
      method: 'post',
      data: { reason },
    })
  },
}
```

`request.ts` 的完整实现见 `references/state-and-data.md`，其关键职责是：注入 token、解包 `data`、把 `code !== 0` 与网络错误统一转成 `BizError`、集中处理 401。

## 4. Store

`src/stores/order.ts`。只存展示数据，不存筛选条件（筛选在 URL）。`requestId` 用于丢弃过期响应。

```ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'
import { orderApi } from '@/api/order'
import type { OrderItem, OrderQuery } from '@/types/order'

export const useOrderStore = defineStore('order', () => {
  const list = ref<OrderItem[]>([])
  const total = ref(0)
  const loading = ref(false)
  const error = ref<string | null>(null)

  let requestId = 0

  const isEmpty = computed(() => !loading.value && !error.value && list.value.length === 0)

  async function fetchList(params: OrderQuery) {
    const current = ++requestId
    loading.value = true
    error.value = null

    try {
      const result = await orderApi.fetchOrderList(params)
      // 快速切换筛选时，旧请求可能后返回，丢弃它
      if (current !== requestId) {
        return
      }
      list.value = result.records
      total.value = result.total
    } catch (e) {
      if (current !== requestId) {
        return
      }
      // 保留旧数据，只标记错误，避免列表闪空
      error.value = e instanceof Error ? e.message : '订单加载失败'
    } finally {
      if (current === requestId) {
        loading.value = false
      }
    }
  }

  async function cancel(id: string, reason: string) {
    await orderApi.cancelOrder(id, reason)
  }

  function clear() {
    list.value = []
    total.value = 0
    error.value = null
  }

  return { list, total, loading, error, isEmpty, fetchList, cancel, clear }
})
```

为什么 `cancel` 里不刷新列表：取消成功后需要刷新，但刷新用什么参数由页面决定（当前筛选条件）。store 里硬编码 `fetchList()` 会丢掉分页与筛选。页面拿到成功结果后带着当前 `filters` 再调一次。

## 5. 筛选参数 composable

`src/composables/useOrderFilters.ts`。集中处理三件事：URL ↔ 查询对象转换、非法参数兜底、写入 URL 时不污染历史记录。

```ts
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useDebounceFn } from '@vueuse/core'
import { isOrderStatus, type OrderQuery } from '@/types/order'

const PAGE_SIZES = [10, 20, 50] as const
const DEFAULT_PAGE_SIZE = 10

export function parseOrderQuery(raw: Record<string, unknown>): OrderQuery {
  const rawPageNum = Number(raw.pageNum)
  const rawPageSize = Number(raw.pageSize)

  return {
    pageNum: Number.isInteger(rawPageNum) && rawPageNum > 0 ? rawPageNum : 1,
    pageSize: PAGE_SIZES.includes(rawPageSize as 10 | 20 | 50) ? rawPageSize : DEFAULT_PAGE_SIZE,
    keyword: typeof raw.keyword === 'string' && raw.keyword.trim() ? raw.keyword.trim() : undefined,
    status: isOrderStatus(raw.status) ? raw.status : undefined,
    startTime: typeof raw.startTime === 'string' ? raw.startTime : undefined,
    endTime: typeof raw.endTime === 'string' ? raw.endTime : undefined,
  }
}

export function useOrderFilters() {
  const route = useRoute()
  const router = useRouter()

  const filters = computed(() => parseOrderQuery(route.query as Record<string, unknown>))

  function apply(patch: Partial<OrderQuery>) {
    const next: Record<string, string> = {}
    const merged = { ...filters.value, ...patch }

    Object.entries(merged).forEach(([key, value]) => {
      if (value !== undefined && value !== null && value !== '') {
        next[key] = String(value)
      }
    })

    // replace 而非 push：筛选变化不应产生一堆历史记录
    router.replace({ query: next })
  }

  // 输入框边打边搜，防抖 300ms
  const applyKeyword = useDebounceFn((keyword: string) => {
    apply({ keyword, pageNum: 1 })
  }, 300)

  function reset() {
    router.replace({ query: {} })
  }

  return { filters, apply, applyKeyword, reset }
}
```

`parseOrderQuery` 单独导出为纯函数，是为了能直接单测——不用挂载组件、不用 mock 路由。

## 6. 状态标签组件

`src/components/order/OrderStatusTag.vue`。映射关系用 `Record<OrderStatus, ...>`，新增状态时忘记补文案会编译报错。

```vue
<script setup lang="ts">
import { computed } from 'vue'
import type { OrderStatus } from '@/types/order'

interface Props {
  status: OrderStatus
}

const props = defineProps<Props>()

const STATUS_MAP: Record<OrderStatus, { label: string; type: 'warning' | 'success' | 'info' | 'danger' }> = {
  WAIT_PAY: { label: '待支付', type: 'warning' },
  PAID: { label: '已支付', type: 'success' },
  SHIPPED: { label: '已发货', type: 'info' },
  COMPLETED: { label: '已完成', type: 'info' },
  CANCELLED: { label: '已取消', type: 'danger' },
}

const current = computed(() => STATUS_MAP[props.status])
</script>

<template>
  <el-tag :type="current.type" disable-transitions>{{ current.label }}</el-tag>
</template>
```

映射表放在模块作用域常量里，不要在模板中写 `v-if` 链：状态一多，模板会变成不可读的分支树。

## 7. 页面

`src/views/order/OrderListView.vue`。三态齐全，筛选走 URL，分页参数在 URL 中保留。

```vue
<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { useOrderStore } from '@/stores/order'
import { useOrderFilters } from '@/composables/useOrderFilters'
import { isBizError } from '@/api/request'
import OrderStatusTag from '@/components/order/OrderStatusTag.vue'
import { ORDER_STATUS_LIST, type OrderStatus } from '@/types/order'

const route = useRoute()
const router = useRouter()
const store = useOrderStore()
const { filters, apply, applyKeyword, reset } = useOrderFilters()

const keywordInput = ref(filters.value.keyword ?? '')
const statusInput = ref<OrderStatus | undefined>(filters.value.status)
const dateRange = ref<[string, string] | null>(
  filters.value.startTime && filters.value.endTime
    ? [filters.value.startTime, filters.value.endTime]
    : null,
)

const columns = [
  { prop: 'orderNo', label: '订单号', minWidth: 180 },
  { prop: 'amount', label: '金额', width: 120 },
  { prop: 'status', label: '状态', width: 120 },
  { prop: 'createTime', label: '创建时间', width: 180 },
] as const

const isEmpty = computed(() => !store.loading && !store.error && store.list.length === 0)

// URL 是筛选条件的唯一来源：任何变化都从这里触发请求
watch(
  () => route.query,
  () => {
    keywordInput.value = filters.value.keyword ?? ''
    statusInput.value = filters.value.status
    dateRange.value =
      filters.value.startTime && filters.value.endTime
        ? [filters.value.startTime, filters.value.endTime]
        : null
    void store.fetchList(filters.value)
  },
  { immediate: true },
)

function handleSearch() {
  apply({
    keyword: keywordInput.value.trim() || undefined,
    status: statusInput.value,
    startTime: dateRange.value?.[0],
    endTime: dateRange.value?.[1],
    pageNum: 1,
  })
}

function handleReset() {
  keywordInput.value = ''
  statusInput.value = undefined
  dateRange.value = null
  reset()
}

function handlePageChange(pageNum: number) {
  apply({ pageNum })
}

function handleSizeChange(pageSize: number) {
  apply({ pageSize, pageNum: 1 })
}

function goDetail(id: string) {
  void router.push({ name: 'OrderDetail', params: { id } })
}

async function handleCancel(id: string) {
  try {
    const { value } = await ElMessageBox.prompt('请填写取消原因', '取消订单', {
      inputPattern: /\S+/,
      inputErrorMessage: '取消原因不能为空',
    })
    await store.cancel(id, value)
    ElMessage.success('订单已取消')
    // 直接重新拉取：URL 参数没变，改 query 不会触发路由变化
    await store.fetchList(filters.value)
  } catch (error) {
    // 用户点击取消按钮关闭弹窗，不是错误
    if (error === 'cancel' || error === 'close') {
      return
    }
    ElMessage.error(isBizError(error) ? error.message : '取消失败，请稍后重试')
  }
}

function canCancel(status: OrderStatus): boolean {
  return status === 'WAIT_PAY' || status === 'PAID'
}
</script>

<template>
  <div class="order-list">
    <el-form :inline="true" class="order-list__filters" @submit.prevent>
      <el-form-item label="关键字">
        <el-input
          v-model="keywordInput"
          placeholder="订单号 / 收货人"
          clearable
          class="order-list__keyword"
          @input="applyKeyword($event)"
          @keyup.enter="handleSearch"
        />
      </el-form-item>
      <el-form-item label="状态">
        <el-select v-model="statusInput" placeholder="全部" clearable class="order-list__status">
          <el-option v-for="item in ORDER_STATUS_LIST" :key="item" :label="item" :value="item" />
        </el-select>
      </el-form-item>
      <el-form-item label="创建时间">
        <el-date-picker
          v-model="dateRange"
          type="daterange"
          value-format="YYYY-MM-DD"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
        />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="handleSearch">查询</el-button>
        <el-button @click="handleReset">重置</el-button>
      </el-form-item>
    </el-form>

    <el-skeleton v-if="store.loading" :rows="6" animated class="order-list__skeleton" />

    <el-result v-else-if="store.error" icon="error" :title="store.error">
      <template #extra>
        <el-button type="primary" @click="store.fetchList(filters)">重新加载</el-button>
      </template>
    </el-result>

    <el-empty v-else-if="isEmpty" description="没有符合条件的订单">
      <el-button @click="handleReset">清空筛选条件</el-button>
    </el-empty>

    <template v-else>
      <el-table :data="store.list" row-key="id" border>
        <el-table-column
          v-for="col in columns"
          :key="col.prop"
          :prop="col.prop"
          :label="col.label"
          :min-width="col.minWidth"
          :width="col.width"
        >
          <template v-if="col.prop === 'status'" #default="{ row }">
            <OrderStatusTag :status="row.status" />
          </template>
          <template v-else-if="col.prop === 'amount'" #default="{ row }">
            <span class="order-list__amount">¥{{ row.amount }}</span>
          </template>
        </el-table-column>

        <el-table-column label="操作" width="160" fixed="right">
          <template #default="{ row }">
            <el-button link type="primary" @click="goDetail(row.id)">详情</el-button>
            <el-button
              v-if="canCancel(row.status)"
              v-permission="'order:cancel'"
              link
              type="danger"
              @click="handleCancel(row.id)"
            >
              取消
            </el-button>
          </template>
        </el-table-column>
      </el-table>

      <el-pagination
        class="order-list__pagination"
        :current-page="filters.pageNum"
        :page-size="filters.pageSize"
        :total="store.total"
        :page-sizes="[10, 20, 50]"
        layout="total, sizes, prev, pager, next, jumper"
        background
        @current-change="handlePageChange"
        @size-change="handleSizeChange"
      />
    </template>
  </div>
</template>

<style scoped lang="scss">
.order-list {
  padding: var(--spacing-lg);

  &__filters {
    margin-bottom: var(--spacing-md);
  }

  &__keyword {
    width: 240px;
  }

  &__status {
    width: 160px;
  }

  &__amount {
    font-weight: 500;
    color: var(--color-text-primary);
  }

  &__pagination {
    margin-top: var(--spacing-md);
    justify-content: flex-end;
  }
}
</style>
```

几个刻意的写法：

- `watch(route.query)` 是唯一的数据入口，`immediate: true` 保证首屏加载，不需要额外的 `onMounted` 请求。
- 输入框使用防抖 `applyKeyword`，但回车与"查询"按钮立即生效——用户明确表达意图时不该再等 300ms。
- `handleCancel` 里区分"用户取消弹窗"与"接口失败"：`ElMessageBox` 关闭时 reject 的是字符串 `'cancel'`，不区分会把正常取消操作提示成错误。
- 表格列配置放在模块作用域，避免每次渲染重建数组。

## 8. 路由注册

`src/router/modules/order.ts`。懒加载 + 权限元信息 + 面包屑标题。

```ts
import type { RouteRecordRaw } from 'vue-router'

const orderRoutes: RouteRecordRaw = {
  path: '/order',
  name: 'Order',
  component: () => import('@/layouts/BasicLayout.vue'),
  redirect: '/order/list',
  meta: { title: '订单管理', icon: 'Tickets' },
  children: [
    {
      path: 'list',
      name: 'OrderList',
      component: () => import('@/views/order/OrderListView.vue'),
      meta: { title: '订单列表', permission: 'order:list', keepAlive: true },
    },
    {
      path: ':id',
      name: 'OrderDetail',
      component: () => import('@/views/order/OrderDetailView.vue'),
      meta: { title: '订单详情', permission: 'order:detail', hidden: true },
    },
  ],
}

export default orderRoutes
```

配置了 `keepAlive: true` 的页面，从详情返回时列表不会重新请求；如果需要刷新，在页面中加 `onActivated`，并注意首次进入会同时触发 `onMounted` 与 `onActivated`。

## 9. Store 测试

`src/stores/__tests__/order.spec.ts`。覆盖成功、失败保留旧数据、过期响应被丢弃三种情况。

```ts
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { useOrderStore } from '../order'
import { orderApi } from '@/api/order'
import { createOrder } from '@/test/factories'

vi.mock('@/api/order')

const baseQuery = { pageNum: 1, pageSize: 10 }

describe('useOrderStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('加载成功后写入列表与总数', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({
      records: [createOrder({ id: '1', orderNo: 'SO202601001' })],
      total: 1,
      pageNum: 1,
      pageSize: 10,
      pages: 1,
    })

    const store = useOrderStore()
    await store.fetchList(baseQuery)

    expect(store.list).toHaveLength(1)
    expect(store.total).toBe(1)
    expect(store.loading).toBe(false)
    expect(store.error).toBeNull()
  })

  it('失败时保留旧数据并记录错误', async () => {
    vi.mocked(orderApi.fetchOrderList).mockResolvedValue({
      records: [createOrder()],
      total: 1,
      pageNum: 1,
      pageSize: 10,
      pages: 1,
    })
    const store = useOrderStore()
    await store.fetchList(baseQuery)

    vi.mocked(orderApi.fetchOrderList).mockRejectedValue(new Error('请求超时'))
    await store.fetchList({ pageNum: 2, pageSize: 10 })

    expect(store.list).toHaveLength(1)
    expect(store.error).toBe('请求超时')
    expect(store.loading).toBe(false)
  })

  it('先发后至的过期响应不会覆盖新数据', async () => {
    const store = useOrderStore()

    let resolveFirst: (value: unknown) => void = () => undefined
    vi.mocked(orderApi.fetchOrderList)
      .mockImplementationOnce(
        () => new Promise((resolve) => { resolveFirst = resolve }) as Promise<never>,
      )
      .mockResolvedValueOnce({ records: [createOrder({ id: '2' })], total: 1, pageNum: 2, pageSize: 10, pages: 1 })

    const first = store.fetchList({ pageNum: 1, pageSize: 10 })
    await store.fetchList({ pageNum: 2, pageSize: 10 })

    resolveFirst({ records: [createOrder({ id: '1' })], total: 1, pageNum: 1, pageSize: 10, pages: 1 })
    await first

    expect(store.list[0]?.id).toBe('2')
  })
})
```

## 10. 参数解析与组件测试

`src/composables/__tests__/useOrderFilters.spec.ts`：

```ts
import { describe, expect, it } from 'vitest'
import { parseOrderQuery } from '../useOrderFilters'

describe('parseOrderQuery', () => {
  it('缺省时给出默认分页', () => {
    expect(parseOrderQuery({})).toEqual({
      pageNum: 1,
      pageSize: 10,
      keyword: undefined,
      status: undefined,
      startTime: undefined,
      endTime: undefined,
    })
  })

  it('非法参数回落到安全值', () => {
    const result = parseOrderQuery({ pageNum: 'abc', pageSize: '999', status: 'HACKED' })

    expect(result.pageNum).toBe(1)
    expect(result.pageSize).toBe(10)
    expect(result.status).toBeUndefined()
  })

  it('合法参数被保留并去除关键字首尾空格', () => {
    const result = parseOrderQuery({ pageNum: '3', pageSize: '20', keyword: '  SO1  ', status: 'PAID' })

    expect(result).toMatchObject({ pageNum: 3, pageSize: 20, keyword: 'SO1', status: 'PAID' })
  })
})
```

`src/components/order/__tests__/OrderStatusTag.spec.ts`：

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import OrderStatusTag from '../OrderStatusTag.vue'

describe('OrderStatusTag', () => {
  it('待支付显示 warning 样式与中文文案', () => {
    const wrapper = mount(OrderStatusTag, { props: { status: 'WAIT_PAY' } })

    expect(wrapper.text()).toBe('待支付')
    expect(wrapper.classes()).toContain('el-tag--warning')
  })

  it('已取消显示 danger 样式', () => {
    const wrapper = mount(OrderStatusTag, { props: { status: 'CANCELLED' } })

    expect(wrapper.text()).toBe('已取消')
    expect(wrapper.classes()).toContain('el-tag--danger')
  })
})
```

## 11. 照抄检查点

1. 状态、查询参数、返回结构都有独立类型，业务代码中无 `any`。
2. 所有请求经过 `api/`，页面与组件不出现 `axios`。
3. 筛选条件只有一处真相（本项目为 URL），没有页面与 store 双写。
4. `loading` / `error` / `empty` 三态齐全，错误态提供重试入口。
5. 表格 `row-key` 为业务 ID，`v-for` 的 `key` 不为 index。
6. 分页大小限定在允许集合内，非法参数有兜底。
7. 取消等写操作有确认、有防重复、失败有明确提示，且区分"用户关闭弹窗"与"接口失败"。
8. 权限控制用 `v-permission`，同时后端接口鉴权（前端不视为安全边界）。
9. 异步请求有竞态保护，过期响应被丢弃。
10. store 至少覆盖成功、失败、竞态三条测试；纯函数与关键组件有单测。

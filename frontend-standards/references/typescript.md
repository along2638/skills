# TypeScript 规范

## 1. tsconfig

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "preserve",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["vite/client"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.vue", "vite.config.ts", "vitest.config.ts"]
}
```

几个开关的作用：

- `strict`：开启全部严格检查，包括 `strictNullChecks`。关掉它等于放弃 TypeScript 的主要价值。
- `noUncheckedIndexedAccess`：`arr[0]`、`record[key]` 的结果带 `undefined`。初上手会觉得啰嗦，但它拦住的正是"数组越界访问导致 `Cannot read properties of undefined`"这类线上错误。
- `verbatimModuleSyntax`：要求类型导入写 `import type`，保证编译后不残留无用 import，也让 tree-shaking 更干净。
- `skipLibCheck: true`：跳过 `node_modules` 里 `.d.ts` 的检查，构建速度明显提升，业务类型仍全部检查。
- 不使用 `allowJs`：新项目不混入 `.js`，`vite.config.ts` 这类配置也用 TS。

## 2. 类型放在哪

| 类型 | 位置 | 说明 |
|---|---|---|
| 接口请求/响应 | `src/api/xxx.ts` 顶部 | 与接口函数同文件，改动时一起改 |
| 页面内部状态 | 页面组件内 | 只被一个页面使用，不上升为全局 |
| 组件 props / emits | 组件内 `interface Props` | 与组件同生命周期 |
| 跨模块共享类型 | `src/types/xxx.ts` | 如 `UserProfile`、`Permission` |
| 环境与全局声明 | `src/types/env.d.ts` | `ImportMetaEnv`、`*.vue` 声明 |
| 自动生成 | `src/types/auto-imports.d.ts`、`components.d.ts` | 入库但不手改 |

不要把类型集中到一个 `types/index.ts` 里当仓库：任何改动都会牵动所有引用，且无法判断某个类型还有没有人用。

## 3. 命名

- 类型/接口 `PascalCase`：`OrderItem`、`PageQuery`。
- 不加 `I` 前缀（`IOrder`）、不加 `Type` 后缀（`OrderType`），这些来自其他语言的残留习惯。
- 泛型参数用有含义的短名：`T`、`TData`、`TItem`、`TKey extends keyof T`。
- 布尔类型属性用 `is/has/can` 前缀：`isDeleted`、`hasPermission`、`canCancel`。
- 联合类型成员用大写下划线或字符串字面量，与后端 code 对齐。

## 4. 向 `any` 宣战

`any` 的问题是它会让类型检查沿调用链失效：一个 `any` 传进函数，函数内所有属性访问都不再校验。项目中只要出现几个核心 `any`，类型系统就名存实亡。

### 4.1 替代写法

| 场景 | 不要写 | 改写 |
|---|---|---|
| 未知结构的外部数据 | `data: any` | `data: unknown` + 类型守卫 |
| 任意对象 | `Record<string, any>` | `Record<string, unknown>` |
| 泛型容器 | `function wrap(v: any): any` | `function wrap<T>(v: T): T` |
| 可能是多种类型 | `value: any` | `value: string \| number` |
| 函数参数 | `(e: any) => void` | `(e: Event) => void` |
| 第三方无类型 | `const sdk: any = window.sdk` | 自己写最小声明文件 |

### 4.2 确实无法避免时

```ts
// types/qywx.d.ts —— 为无类型 SDK 写最小声明，而不是 any 满天飞
declare global {
  interface Window {
    wx?: {
      config(options: Record<string, unknown>): void
      ready(callback: () => void): void
    }
  }
}

export {}
```

豁免必须注释原因，且限定单行：

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- 三方支付 SDK 回调签名无类型声明
window.payCallback = (payload: any) => handlePayResult(payload)
```

`@ts-ignore` 一律禁止，用 `@ts-expect-error`：

```ts
// @ts-expect-error 组件库 Table 的 row-key 类型声明缺失，运行时支持函数
<el-table :row-key="(row: OrderItem) => row.id" />
```

`@ts-expect-error` 的好处是：一旦上游修好类型，这行会立刻报"未使用的忽略"，强制你删掉豁免。

## 5. 类型断言的使用边界

`as` 会让编译器闭嘴，但不改变运行时。只允许在两个边界使用：

1. **接口响应**：`request<OrderListResult>` 在 `api/` 层声明一次，业务层不再断言。
2. **环境变量**：`import.meta.env.VITE_ENABLE_MOCK as string | undefined`（更好的是在 `env.d.ts` 声明类型）。

```ts
// 反例：在业务代码里层层断言
const list = (res.data as any).list as OrderItem[]

// 正例：请求函数带泛型
const list = await orderApi.fetchOrderList(query) // 返回 OrderListResult
```

`as unknown as T` 属于强制转换，除迁移历史代码外禁止使用。

## 6. interface 与 type 的选择

| 场景 | 用哪个 | 原因 |
|---|---|---|
| 对象结构、组件 props、接口响应 | `interface` | 可声明合并、错误信息更友好 |
| 联合、交叉、映射、条件类型 | `type` | `interface` 无法表达 |
| 函数类型 | `type` | `type Fn = (id: string) => void` 更直观 |
| 需要被 `implements` | `interface` | 类实现时更清晰 |

项目内保持一致的默认：**对象用 `interface`，其余用 `type`**，不要同一类结构两种混用。

## 7. 用字面量联合代替 enum

```ts
// 推荐
export const ORDER_STATUS = {
  WAIT_PAY: 'WAIT_PAY',
  PAID: 'PAID',
  SHIPPED: 'SHIPPED',
  COMPLETED: 'COMPLETED',
  CANCELLED: 'CANCELLED',
} as const

export type OrderStatus = (typeof ORDER_STATUS)[keyof typeof ORDER_STATUS]
```

```ts
// 不推荐
enum OrderStatus {
  WAIT_PAY = 'WAIT_PAY',
}
```

原因：

- `enum` 会生成运行时代码，影响 tree-shaking；`as const` 对象是纯数据结构。
- 后端返回的是字符串 code，字面量联合可以直接比较，不需要 `as` 转换。
- 字面量联合在 `switch` 中有穷尽性检查，漏分支时 `never` 兜底能报错。
- 需要遍历渲染下拉项时，`as const` 对象可以直接 `Object.entries`，`enum` 要用反向映射且字符串枚举不支持。

穷尽性检查写法：

```ts
function assertNever(value: never): never {
  throw new Error(`未处理的状态: ${String(value)}`)
}

function statusText(status: OrderStatus): string {
  switch (status) {
    case 'WAIT_PAY':
      return '待支付'
    case 'PAID':
      return '已支付'
    case 'SHIPPED':
      return '已发货'
    case 'COMPLETED':
      return '已完成'
    case 'CANCELLED':
      return '已取消'
    default:
      return assertNever(status)
  }
}
```

新增状态时忘记处理，`assertNever` 会直接编译报错——这比运行时发现白屏强得多。

## 8. 泛型实战

### 8.1 后端契约类型

```ts
// src/types/api.ts
export interface ApiResponse<T> {
  code: number
  message: string
  data: T
  traceId: string
}

export interface PageQuery {
  pageNum: number
  pageSize: number
}

export interface PageResult<T> {
  list: T[]
  total: number
  pageNum: number
  pageSize: number
}

export type Nullable<T> = T | null
```

### 8.2 请求函数

```ts
// src/api/order.ts
import { request } from './request'
import type { PageQuery, PageResult } from '@/types/api'

export interface OrderQuery extends PageQuery {
  keyword?: string
  status?: OrderStatus
  startTime?: string
  endTime?: string
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

export const orderApi = {
  fetchOrderList(params: OrderQuery) {
    return request<PageResult<OrderItem>>({ url: '/order/list', method: 'get', params })
  },
  fetchOrderDetail(id: string) {
    return request<OrderDetail>({ url: `/order/${id}`, method: 'get' })
  },
  cancelOrder(id: string) {
    return request<void>({ url: `/order/${id}/cancel`, method: 'post' })
  },
}
```

关键点：泛型 `T` 在接口层指定一次，调用方拿到的是精确类型，不需要再断言，也不会把 `any` 漏进业务代码。

## 9. 工具类型实战

```ts
// 从实体派生表单类型：去掉后端生成字段
type OrderForm = Omit<OrderItem, 'id' | 'orderNo' | 'createTime'>

// 更新接口只传部分字段
type OrderUpdatePayload = Partial<Pick<OrderItem, 'remark' | 'status'>>

// 选项映射
type StatusOption = { label: string; value: OrderStatus }
const STATUS_OPTIONS: Record<OrderStatus, string> = {
  WAIT_PAY: '待支付',
  PAID: '已支付',
  SHIPPED: '已发货',
  COMPLETED: '已完成',
  CANCELLED: '已取消',
}

// 只读配置，防止运行时被改
const TABLE_COLUMNS: ReadonlyArray<{ prop: string; label: string }> = [
  { prop: 'orderNo', label: '订单号' },
  { prop: 'amount', label: '金额' },
]
```

`Record<OrderStatus, string>` 的价值：新增状态时若忘记补文案，编译直接报错。

## 10. 类型守卫与收窄

```ts
// 可辨识联合表达页面状态，避免三个布尔值互相矛盾
type PageState<T> =
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; data: T }

function isApiError(error: unknown): error is ApiError {
  return typeof error === 'object' && error !== null && 'code' in error && 'message' in error
}

try {
  await load()
} catch (error) {
  if (isApiError(error)) {
    message.error(error.message)
  } else {
    message.error('系统异常，请稍后重试')
  }
}
```

收窄优先使用 `typeof`、`instanceof`、`in`、字面量判断，而不是 `as` 强转。`catch (error)` 中的 `error` 在 `useUnknownInCatchVariables`（`strict` 已包含）下是 `unknown`，这是好事。

## 11. Vue 相关类型

```ts
// props + 默认值
interface Props {
  order: OrderItem
  showActions?: boolean
}
const props = withDefaults(defineProps<Props>(), { showActions: true })

// emits
const emit = defineEmits<{
  cancel: [id: string]
  change: [payload: { id: string; status: OrderStatus }]
}>()

// v-model
const keyword = defineModel<string>({ default: '' })

// ref 的类型推断与显式标注
const list = ref<OrderItem[]>([])
const formRef = ref<FormInstance>() // 需要调用 validate/expose 方法时必须标注实例类型

// 模板 ref
const tableRef = ref<InstanceType<typeof ElTable>>()
```

注意：

- `ref([])` 推断为 `never[]`，必须写 `ref<OrderItem[]>([])`。
- `ref<FormInstance>()` 初始是 `undefined`，使用前判断 `formRef.value`。
- 不要用 `ref<any>` 图省事，组件实例类型用 `InstanceType<typeof Xxx>` 或官方导出的 `XxxInstance`。

## 12. 常见坑

1. **`Object.keys` 丢类型**：返回 `string[]` 而非 `keyof T`，需要时写 `(Object.keys(obj) as Array<keyof T>)`，并在注释说明原因。
2. **`JSON.parse` 返回 `any`**：包一层 `parseJson<T>(text: string): T | null`，内部 `try/catch`。
3. **可选属性与 `undefined` 混用**：接口中用 `field?: string`，不要用 `field: string | undefined`，除非确实要传 `undefined` 覆盖默认值。
4. **数组 `find` 结果**：返回 `T | undefined`，使用前判空，不要 `!` 断言。
5. **非空断言 `!`**：只允许在已确定 DOM 挂载完成后使用（`onMounted` 内），且需紧邻判断逻辑。
6. **`defineProps` 解构**：Vue 3.4 之前解构会丢响应性；3.4+ 支持响应式解构，但仍建议用 `props.xxx` 保持明确。

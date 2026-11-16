# 前后端接口契约（前端视角）

后端契约的权威定义在后端规范的 `references/api-contract.md`。本文件是**同一份契约的前端承接侧**：前端按什么结构定义类型、如何解包与处理错误、哪些变更必须容忍。两份文件出现分歧时，以后端实现与本文档的「对齐表」为准，并立即修正另一方。

## 1. 契约总则

- 契约由后端定义并在 OpenAPI 文档中固化；前端不发明字段，只消费。
- 前端对**新增**字段、新增枚举值、新增可选参数必须容忍；对**删除/重命名/类型变更**不做兼容，由后端升版本。
- 字段命名统一 lowerCamelCase，前端不做下划线转换（不做 `snake_case` → 驼峰的后处理）。
- 任何"前端先按自己想的字段写，联调时再改"的做法都会产生返工；联调前先对齐第 12 节清单。

## 2. 统一响应包

```ts
// src/types/api.ts
export interface ApiResponse<T> {
  code: number
  message: string
  data: T
  traceId: string
}
```

| 字段 | 类型 | 说明 | 前端用法 |
|---|---|---|---|
| `code` | number | `0` 成功，非 0 失败 | **唯一**的业务判断依据 |
| `message` | string | 面向用户的提示 | 直接展示，不做文案匹配 |
| `data` | T \| null | 业务数据；失败时通常为 `null` | 成功分支才读取 |
| `traceId` | string | 与后端日志一致 | 错误上报与问题排查时带上 |

规则：

- **禁止用 `message` 做逻辑判断**（`if (message.includes('已支付'))`）。文案会改，错误码不会。
- 请求层负责解包：拦截器返回 `data`，业务代码拿到的是 `T` 而不是 `ApiResponse<T>`。
- `data` 允许为 `null`（如 `void` 成功、失败响应），类型上写 `T | null`，调用处判空。
- `traceId` 同时出现在响应体和 `X-Trace-Id` 响应头，两者取其一（优先 body），用于错误提示的"复制追踪号"。

## 3. 分页契约

请求参数：

| 参数 | 类型 | 约束 | 默认 |
|---|---|---|---|
| `pageNum` | number | ≥ 1，超过总页数由后端返回最后一页 | 1 |
| `pageSize` | number | 1 ~ 100 | 10 |
| `sortBy` | string | 后端白名单内的字段名 | 后端默认排序 |
| `sortOrder` | `'asc' \| 'desc'` | 与 `sortBy` 配对使用 | `desc` |

响应 `data`：

```ts
export interface PageResult<T> {
  records: T[]
  total: number
  pageNum: number
  pageSize: number
  pages: number
}
```

**注意字段名是 `records` 而不是 `list`**。这是前后端最容易错的一处：后端 `PageResp.records` 一旦被前端按 `list` 取值，得到的就是 `undefined`，页面表现为"接口有数据但列表空白"，且不报错。

前端 store 内部变量叫 `list` 没问题（那是本地状态命名），但赋值必须来自 `data.records`：`list.value = result.records`。

规则：

- 列表为空时 `records` 是 `[]`，不会是 `null`、不会是 `undefined`。
- `pages` 由后端计算，前端不自己算（除非需要本地分页预览，那属于另一种场景）。
- 前端请求 `pageNum` 超出范围时后端会回落到最后一页，前端应以响应里的 `pageNum` 为准回填当前页，而不是用请求参数。
- 排序字段必须走白名单参数，禁止把前端任意字段名传过去——后端会拒绝或忽略，前端需按响应实际顺序渲染，不要本地重排。
- 批量操作不要塞进 URL：URL 参数总量上限 2048 字节，超过会被截断导致漏处理，批量 ID 改用 `POST` + body。

## 4. 字段类型约定

| 类型 | 后端返回 | 前端处理 |
|---|---|---|
| 主键 / 外键 ID | 字符串 `"10001"` | 类型写 `string`，**不做 `Number()` 转换**，路由参数直接用 |
| 超长单号（订单号、流水号） | 字符串 | 同上，禁止参与数值运算 |
| 金额 | 字符串 `"99.00"` | 只格式化展示，禁止 `+ - * /`；比较用字符串相等或转 `BigInt` 分单位 |
| 时间 | `"2026-10-25 10:00:00"` | dayjs 解析后格式化；`safari` 不接受带空格的格式时用 `dayjs(value, 'YYYY-MM-DD HH:mm:ss')` 显式指定 |
| 日期 | `"2026-10-25"` | 同上 |
| 枚举 | 大写蛇形字符串 `"WAIT_PAY"` | 字面量联合 + 映射表，未知值必须有兜底文案 |
| 布尔 | `true` / `false` | 类型 `boolean`，不要与 `0/1` 混用 |
| 空列表 | `[]` | 直接遍历，不需要判 `null` |
| 空对象 | `{}` | 同上 |
| 可选字段 | 字段可能不存在或为 `null` | 类型写 `field?: T \| null`，展示层统一兜底 `-` |

容易踩的坑：

1. **雪花 ID 精度**：后端已按字符串返回。若某接口漏配序列化返回了数字，超过 2^53 会变成 `...000`，前端会发现"详情查不到"。遇到时按第 11 节排查，不要自己在前端 `String()` 补救。
2. **金额**：`"99.00"` 直接渲染；需要千分位用 `Intl.NumberFormat`，需要相加请让后端返回汇总值。
3. **时区**：统一东八区。前端 `dayjs` 不做时区转换，直接按字符串展示，避免"少 8 小时"。

## 5. 错误码与前端处理策略

| 区间 | 含义 | 前端行为 |
|---|---|---|
| `0` | 成功 | 正常渲染 |
| `1xxxx` | 参数/请求格式 | 400 时优先映射到表单字段（见第 6 节）；无字段信息则 toast |
| `2xxxx` | 认证、权限、限流 | 20001 清 token 跳登录（防重入）；20002 提示无权限；20005 提示稍后重试 |
| `3xxxx` | 业务错误 | 展示 `message`，保持页面状态，不做整页错误 |
| `4xxxx` | 第三方依赖失败 | 展示 `message`，提示"请稍后重试"，不暴露第三方名称 |
| `5xxxx` | 系统错误 | 统一文案"系统繁忙"，附带 `traceId` 便于反馈 |

对应的常量集中定义，禁止在页面里写裸数字：

```ts
// src/types/error-code.ts
export const ERROR_CODE = {
  SUCCESS: 0,
  PARAM_INVALID: 10001,
  REQUEST_BODY_INVALID: 10002,
  UNAUTHORIZED: 20001,
  FORBIDDEN: 20002,
  NOT_FOUND: 20003,
  TOO_MANY_REQUESTS: 20005,
  SYSTEM_ERROR: 50000,
} as const

export type ErrorCodeValue = (typeof ERROR_CODE)[keyof typeof ERROR_CODE]

export function isAuthError(code: number): boolean {
  return code === ERROR_CODE.UNAUTHORIZED
}
```

`3xxxx` 业务码不在前端枚举里逐个登记（后端会持续新增），前端只处理区间语义，具体提示直接用 `message`。

## 6. 参数校验错误

后端校验失败固定返回 HTTP 400 + `code = 10001`，`data` 是字段与提示的数组：

```json
{
  "code": 10001,
  "message": "参数校验失败",
  "data": [
    { "field": "amount", "message": "金额必须大于0" },
    { "field": "receiverPhone", "message": "手机号格式不正确" }
  ],
  "traceId": "9f2c1a7b4e6d"
}
```

前端应把它回填到表单项，而不是只弹一个笼统的 toast：

```ts
export interface FieldError {
  field: string
  message: string
}

export function isFieldErrorList(value: unknown): value is FieldError[] {
  return (
    Array.isArray(value) &&
    value.every(
      (item) =>
        typeof item === 'object' &&
        item !== null &&
        typeof (item as FieldError).field === 'string' &&
        typeof (item as FieldError).message === 'string',
    )
  )
}
```

```ts
function applyFieldErrors(errors: FieldError[]) {
  errors.forEach(({ field, message }) => {
    // 只提示第一条即可，避免一次弹十几条
    formRef.value?.validateField(field).catch(() => undefined)
    ElMessage.warning(`${message}`)
  })
}
```

约定：

- `field` 使用后端字段名（与前端表单 `prop` 一致）。若不一致，在前端维护一张映射表，不要改后端字段名。
- 前端已有校验规则时，后端校验是兜底；两边规则必须一致（长度、格式、范围），否则会出现"前端放行、后端拒绝"。
- 嵌套字段用 `items[0].amount` 这类路径形式，前端映射时需要按路径拆分回填到子表单。

## 7. 请求头

| Header | 方向 | 前端职责 |
|---|---|---|
| `Authorization` | 请求 | `Bearer ${token}`，由请求拦截器统一注入 |
| `Content-Type` | 请求 | JSON 写 `application/json;charset=UTF-8`；上传用 `multipart/form-data`（让浏览器自动带 boundary，不要手写） |
| `X-Request-Id` | 请求 | 每个请求生成 `crypto.randomUUID()`，便于链路对齐 |
| `Idempotency-Key` | 请求 | 创建/支付/退款类接口必须带，见第 8 节 |
| `X-Trace-Id` | 响应 | 读取并附加到错误对象，用户反馈问题时提供 |

```ts
// 请求拦截器片段
instance.interceptors.request.use((config) => {
  const token = storage.getToken()
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  config.headers['X-Request-Id'] = crypto.randomUUID()
  return config
})
```

## 8. 幂等

后端的幂等方案是基于 `Idempotency-Key` 的 Redis 记录：相同 key + 相同请求体返回首次结果，相同 key + 不同请求体返回 409。

前端职责：

- 创建类请求（下单、支付、退款、批量导入）在**用户点击时生成一次** key，重试沿用同一 key，成功后作废。
- key 用 UUID，长度 8~64，只含 `[A-Za-z0-9-]`。
- 按钮置灰与请求中标记只解决误触，不解决网络重试与重复提交，二者都要做。

```ts
export function createIdempotencyKey(): string {
  return crypto.randomUUID()
}

// 一次业务动作内复用同一个 key
async function submitOrder(payload: OrderCreatePayload) {
  const key = createIdempotencyKey()
  return request<OrderDetail>({
    url: '/orders',
    method: 'post',
    data: payload,
    headers: { 'Idempotency-Key': key },
  })
}
```

## 9. 文件上传与下载

上传：

- `POST /api/v1/files`，`multipart/form-data`，字段名与后端约定一致（如 `file`）。
- 响应仍包在 `Result` 里，`data` 返回文件标识与访问地址（字段以后端接口文档为准，通常含 `fileId`、`url`、`name`、`size`）。
- 前端只做体验校验（类型、大小），后端必须重新校验。

下载：

- 下载接口**不包 `Result`**，直接返回二进制流，HTTP 200 + `Content-Disposition`。
- 前端必须 `responseType: 'blob'`，并从响应头解析文件名；出错时后端可能返回 JSON，需要判断类型。

```ts
export async function downloadFile(url: string, params?: Record<string, unknown>) {
  const response = await axios.get(url, {
    params,
    responseType: 'blob',
    baseURL: import.meta.env.VITE_API_BASE_URL,
    headers: { Authorization: `Bearer ${storage.getToken()}` },
  })

  const blob = response.data as Blob
  // 后端出错时返回的是 JSON，被当成 blob 收下了
  if (blob.type.includes('application/json')) {
    const text = await blob.text()
    const body = JSON.parse(text) as ApiResponse<unknown>
    throw new BizError(body.code, body.message, body.traceId)
  }

  const disposition = String(response.headers['content-disposition'] ?? '')
  const matched = /filename\*=UTF-8''([^;]+)/.exec(disposition)
  const filename = matched ? decodeURIComponent(matched[1]!) : 'download.xlsx'

  const objectUrl = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = objectUrl
  link.download = filename
  link.click()
  URL.revokeObjectURL(objectUrl)
}
```

注意 `Content-Disposition` 的中文文件名按 RFC 5987 用 `filename*=UTF-8''` 传递，`decodeURIComponent` 后再使用。

## 10. 版本与向后兼容

| 后端变更 | 是否破坏性 | 前端要求 |
|---|---|---|
| 新增响应字段 | 否 | 忽略即可，不影响渲染 |
| 新增枚举值 | 否 | **必须有兜底**：未知 code 显示原始值或"未知状态"，不能空白或报错 |
| 新增可选请求参数 | 否 | 不传时行为与之前一致 |
| 删除/重命名字段 | 是 | 后端升 `/api/v2`，前端不猜测兼容 |
| 字段类型变更（数字 ↔ 字符串） | 是 | 同上 |
| 收紧校验（长度、必填） | 是 | 同上 |

未知枚举的兜底写法：

```ts
const STATUS_TEXT: Record<string, string> = {
  WAIT_PAY: '待支付',
  PAID: '已支付',
  SHIPPED: '已发货',
  COMPLETED: '已完成',
  CANCELLED: '已取消',
}

export function statusText(status: string): string {
  return STATUS_TEXT[status] ?? status
}
```

用 `Record<string, string>` 而不是 `Record<OrderStatus, string>`，是有意的取舍：前者会失去"新增状态忘记补文案"的编译期检查，后者会让后端新增状态时前端直接渲染空白。**建议主链路用严格类型 + `assertNever` 兜底，展示型字段用宽松映射 + 原值兜底**，见 `references/typescript.md` 第 7 节。

## 11. 契约不一致的排查清单

| 症状 | 最可能的原因 | 处理 |
|---|---|---|
| 接口有数据，列表空白且无报错 | 分页字段名不符（`records` vs `list`） | 打印响应体，核对第 3 节 |
| 详情页字段全 `undefined` | 前端按 `data.xxx` 取值，但实际是 `data.detail.xxx`；或字段名拼写不同 | 对比 OpenAPI 定义 |
| ID 后几位变 `000` | 后端漏配字符串序列化，JS 精度丢失 | 让后端修，不在前端 `String()` 补救 |
| 金额算出来 `0.30000000000000004` | 前端做了浮点运算 | 改成展示后端汇总值 |
| 时间少 8 小时 | 前端按 UTC 解析，或后端返回了带 `Z` 的 ISO 串 | 统一 `Asia/Shanghai`，显式指定格式解析 |
| 错误提示总是"系统繁忙" | 响应体不是 `Result` 结构，拦截器走了兜底分支 | 检查 HTTP 状态与 body 结构 |
| 401 反复跳登录、页面闪烁 | 并发请求各自触发跳转 | 用 `redirecting` 标记防重入 |
| 下载得到损坏文件 | 未设 `responseType: 'blob'`，或把 JSON 错误当文件保存 | 按第 9 节判断 `blob.type` |
| 重复下单产生两条数据 | 未传 `Idempotency-Key`，或每次重试都新建 key | 按第 8 节 |

## 12. 联调前置清单

接入一个新接口前，逐条确认后再写代码：

1. 路径是否为 `/api/v1/{复数资源}`，动作用子路径（`/orders/{id}/cancel`）。
2. 成功与失败的响应体是否都是 `Result` 结构，`code` 语义是否为 `0` 表示成功。
3. 分页响应字段是否为 `records` / `total` / `pageNum` / `pageSize` / `pages`。
4. ID、金额、时间、枚举的实际类型与格式是否与第 4 节一致。
5. 错误码区间与前端处理策略是否匹配（尤其 401、403、429、409）。
6. 校验失败时 `data` 是否为 `{ field, message }[]`，字段名能否与表单 `prop` 对应。
7. 创建/支付类接口是否要求 `Idempotency-Key`，长度与字符集约定是否一致。
8. 下载接口是否返回流、文件名放在哪个响应头。
9. 是否有新增枚举值会让前端显示空白。
10. 接口文档示例是否为脱敏数据，字段命名有无 `snake_case` 残留。

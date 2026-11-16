# Vue 组件编码规范

## 1. 单文件组件结构

固定顺序：`<script setup lang="ts">` → `<template>` → `<style scoped lang="scss">`。`script` 在最上，是因为读代码时先看数据与逻辑，再看模板。

```vue
<script setup lang="ts">
import { computed, ref } from 'vue'
import type { OrderItem } from '@/types/order'

interface Props {
  order: OrderItem
  showActions?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showActions: true,
})

const emit = defineEmits<{
  cancel: [id: string]
  view: [id: string]
}>()

const cancelling = ref(false)

const amountText = computed(() => `¥${props.order.amount}`)

async function handleCancel() {
  if (cancelling.value) {
    return
  }
  cancelling.value = true
  try {
    emit('cancel', props.order.id)
  } finally {
    cancelling.value = false
  }
}
</script>

<template>
  <div class="order-item">
    <span class="order-item__no">{{ order.orderNo }}</span>
    <span class="order-item__amount">{{ amountText }}</span>
    <el-button v-if="showActions" :loading="cancelling" @click="handleCancel">取消</el-button>
  </div>
</template>

<style scoped lang="scss">
.order-item {
  display: flex;
  align-items: center;
  gap: var(--spacing-sm);

  &__no {
    color: var(--color-text-primary);
  }
}
</style>
```

规则：

- 一个 SFC 一个组件，文件名与组件名一致，PascalCase 多单词。
- 单文件超过 400 行必须拆：先看模板能否拆子组件，再看逻辑能否抽 composable。
- 模板嵌套超过 4 层必须拆，嵌套深了改一个字段要滚三屏。
- 不在 SFC 里写 `export default { ... }` 与 `<script setup>` 并存，除非需要 `inheritAttrs` 这类必须用选项式声明的配置（此时用第二个 `<script>` 块，只写 `export default { inheritAttrs: false }`）。

## 2. 命名

| 对象 | 规则 | 示例 |
|---|---|---|
| 组件文件 / 组件名 | PascalCase，多单词 | `OrderList.vue`、`OrderStatusTag.vue` |
| 页面 | `XxxListView` / `XxxDetailView` / `XxxFormView` | `OrderListView.vue` |
| composable | `use` + 大驼峰，文件同名 | `useOrderFilters.ts` |
| store | `use` + 领域 + `Store` | `useOrderStore` |
| 事件 | kebab-case，动词开头，过去式或动作 | `@row-click`、`@submit-success` |
| props | camelCase，布尔用形容词或 `is/has` 前缀 | `showActions`、`isLoading`、`hasPermission` |
| CSS 类 | BEM，块名与组件名对应 | `.order-list__item--active` |
| 常量 | UPPER_SNAKE_CASE | `ORDER_STATUS_MAP` |
| 类型 | PascalCase，不加 `I` 前缀、不加 `Type` 后缀 | `OrderItem` |

模板中组件用 PascalCase 引用（`<OrderStatusTag />`），HTML 原生标签保持小写，避免与自定义元素混淆。

## 3. props / emits / slots

### 3.1 props

- 必须用类型声明（`defineProps<Props>()` 或运行时对象），并提供默认值 `withDefaults`。
- 对象/数组默认值必须是工厂函数：`items: () => []`，否则所有实例共享同一个引用。
- 禁止在子组件里直接修改 props；需要"可编辑副本"时用 `const local = ref(props.value)` 或 `computed` + `emit`。
- 布尔 props 不写 `:show-actions="true"`，直接 `show-actions`；默认值设为 `false` 时更符合直觉。
- 避免"万能 props"：超过 7 个 props 通常意味着该拆组件或改传配置对象。

### 3.2 emits

```ts
const emit = defineEmits<{
  change: [value: string]
  'update:modelValue': [value: string]
  submit: [payload: { id: string; remark: string }]
}>()
```

- 必须显式声明，不要依赖 `$emit` 透传；声明后事件名与签名都有类型检查。
- 事件名在模板里用 kebab-case，在 `emit()` 里用声明的名字（带冒号的 `update:modelValue` 保持原样）。
- 不要把整个对象实例抛给父组件再让父组件改内部字段；抛最小必要数据。
- 事件命名表达"发生了什么"，不表达"父组件该做什么"：`submit` 优于 `doSubmit`。

### 3.3 v-model 与 defineModel

Vue 3.4+ 用 `defineModel` 简化双向绑定：

```ts
// 单值
const modelValue = defineModel<string>({ default: '' })

// 多值
const keyword = defineModel<string>('keyword', { default: '' })
const pageSize = defineModel<number>('pageSize', { default: 10 })
```

父组件：

```vue
<OrderFilterBar v-model:keyword="keyword" v-model:page-size="pageSize" />
```

注意：`defineModel` 返回的是 `Ref`，可以在子组件内直接赋值；不要再手动 `emit('update:xxx')`，两者混用会触发两次更新。

### 3.4 slots

- 内容可替换的位置用 slot，不要用 `v-if` 堆一堆 props 变体。
- 具名插槽用于结构固定的区域（`#header`、`#footer`），作用域插槽用于"父组件需要子组件数据"（`#default="{ row }"`）。
- 提供默认内容，让调用方不传也能用。
- 插槽名 kebab-case：`<template #extra-actions>`。

## 4. 响应式

### 4.1 ref 与 reactive

| 场景 | 选择 | 原因 |
|---|---|---|
| 基本类型、需要整体替换 | `ref` | `.value` 赋值语义清晰 |
| 表单对象（字段固定） | `reactive` | 模板里不用写 `.value` |
| 需要整体重置的对象 | `ref` | `obj.value = { ...initial }` 一步到位 |
| 大列表 / 只读大数据 | `shallowRef` | 不深度代理，性能更好 |
| 第三方实例（图表、地图、组件实例） | `markRaw` / `shallowRef` | 避免被代理导致异常 |

经验规则：**默认用 `ref`**。`reactive` 只在"表单对象、字段较多且不需要整体替换"时使用，因为 `reactive` 解构会丢失响应性，重置也要逐个字段赋值。

```ts
// 反例：解构 reactive 丢响应性
const form = reactive({ name: '', phone: '' })
const { name } = form // name 是普通字符串，后续修改不触发更新

// 正例：用 ref 或 toRefs
const form = ref({ name: '', phone: '' })
```

### 4.2 computed 与 watch

- 一个值由其他值推导 → `computed`，不要用 `watch` 回写第二个变量。`watch` 回写会产生两个真相，容易不同步。
- `computed` 内不做副作用（发请求、改其他状态），只做纯计算。
- `watch` 用于"值变化后要执行副作用"：发请求、写 localStorage、埋点。

```ts
// 反例：两份状态手动同步
const total = ref(0)
watch(items, () => {
  total.value = items.value.reduce((sum, item) => sum + Number(item.amount), 0)
})

// 正例
const total = computed(() => items.value.reduce((sum, item) => sum + Number(item.amount), 0))
```

`watch` 的写法要求：

```ts
watch(
  () => props.orderId,
  async (id) => {
    if (!id) {
      return
    }
    await loadDetail(id)
  },
  { immediate: true },
)
```

- 监听具体来源（`() => props.orderId`），不要监听整个 `props`。
- `deep: true` 只用于确实需要的小对象；监听大数组的深层变化会带来明显开销，改用版本号或 `watch` 特定字段。
- 异步 watch 存在竞态：连续触发时旧请求可能后返回。必须用 `AbortController` 或请求序号丢弃过期结果。
- `flush: 'post'` 用于需要 DOM 已更新的场景（如滚动到某元素）。

### 4.3 响应性丢失的四个常见坑

1. 解构 `reactive` 对象或 `props`：`const { id } = props` 后 `id` 不再更新，用 `toRef(props, 'id')` 或 `computed(() => props.id)`。
2. 给 `ref` 数组用下标整体替换：`list.value[0] = newItem` 在 Vue 3 是可响应的，但 `list.value.length = 0` 后重新 `push` 会触发多次更新，优先整体替换 `list.value = []`。
3. `defineExpose` 暴露的普通变量不是响应式的，暴露 `ref` 或方法。
4. 在 `setup` 顶层把 `ref` 值存进普通对象再传给子组件，后续更新不生效。

## 5. 生命周期与副作用

| 钩子 | 用途 | 注意 |
|---|---|---|
| `onMounted` | 首次拉数据、初始化第三方实例 | 不保证子组件已全部挂载完成，需要时用 `nextTick` |
| `onActivated` | `keep-alive` 页面被激活时刷新 | 缓存页面必须处理，否则返回时数据是旧的 |
| `onDeactivated` | 离开缓存页面时暂停轮询 / 定时器 | 与 `onActivated` 成对出现 |
| `onUnmounted` | 清理定时器、监听、`AbortController` | 漏清理会导致内存泄漏与报错 |
| `onBeforeUnmount` | 需要 DOM 仍存在时收尾（销毁图表实例） | 图表库通常在此时 `dispose` |

规则：

- 在 `setup` 顶层只写同步的声明与注册，异步请求放 `onMounted`。
- 所有 `setInterval` / `addEventListener` / `IntersectionObserver` / `ResizeObserver` / `AbortController` 必须配对清理。
- 页面卸载后仍在跑的请求会触发"组件已卸载但还在 setState"的告警，统一在 `onUnmounted` 中 `abort()`。

```ts
const controller = ref<AbortController>()

async function load() {
  controller.value?.abort()
  controller.value = new AbortController()
  try {
    list.value = await orderApi.fetchList(query.value, controller.value.signal)
  } catch (error) {
    if (isAbortError(error)) {
      return
    }
    throw error
  }
}

onUnmounted(() => controller.value?.abort())
```

## 6. 模板写法

- 属性顺序固定：`v-if` 或 `v-for` → `v-bind` → `v-model` → `v-on` → 其他自定义指令，便于扫读。
- `v-if` 与 `v-for` 不同时使用：`v-for` 优先级更高，`v-if` 会作用于每一项，语义容易错，也浪费渲染。需要过滤时用 `computed` 先算好。
- `v-for` 的 `key` 必须用业务唯一 ID；无 ID 时才用 `:key="index"` 并在注释中说明原因（仅限静态列表）。
- 模板里只放表达式，不放复杂逻辑；超过一个 `&&` 或三元嵌套就抽 `computed` 或方法。
- 禁止在模板里调用会重新生成对象的方法：`v-for="user in getUsers()"` 每次渲染都新建数组，触发无限更新，改用 `computed`。
- 空态、加载态、错误态在模板里显式分支，不要用 `v-if="list.length"` 一句话带过导致空白页面。
- 文本插值 `{{ }}` 不会转义 HTML，本身安全；渲染富文本必须走 `v-html` + `DOMPurify`，见 `references/security.md`。

## 7. 组件通信方式选择

| 场景 | 方式 | 说明 |
|---|---|---|
| 父 → 子 | props | 默认方式 |
| 子 → 父 | emits | 事件名表达发生了什么 |
| 双向绑定 | `defineModel` | 表单类组件 |
| 结构复用 | slots | 内容由父组件决定 |
| 跨多层（3 层以上） | `provide` / `inject` | 用 `InjectionKey` 提供类型 |
| 跨页面共享 | Pinia store | 需要持久或跨路由 |
| 全局事件总线 | **禁止** | 用 store 或 provide/inject 替代 |

`provide` / `inject` 必须配类型键，避免字符串魔法：

```ts
// src/composables/useOrderContext.ts
import type { InjectionKey } from 'vue'

export interface OrderContext {
  readonlyMode: boolean
}

export const orderContextKey: InjectionKey<OrderContext> = Symbol('orderContext')
```

## 8. 组件边界与错误处理

- 组件内部错误不要吞掉：网络错误抛给页面统一处理，渲染错误由 `app.config.errorHandler` 兜底并上报。
- `app.config.errorHandler` 中记录 `error` + 组件名 + 路由，但不上报完整堆栈到用户可见区域。
- 可预期的业务失败（余额不足、库存不足）用 `ElMessage` 提示并保留页面状态，不要抛异常导致白屏。
- 异步组件 / 图表 / 富文本等易崩模块外包一层错误边界组件（`onErrorCaptured`），失败时展示降级 UI。
- 表单组件必须处理"校验失败"分支：`await formRef.value?.validate()` 抛错时中止提交，不要 catch 后继续。

## 9. 性能陷阱（组件层面）

- 大列表交给虚拟滚动，见 `references/performance.md`；不要在 `v-for` 里写 `v-if` 或复杂方法调用。
- 只读的大对象用 `shallowRef` + `markRaw`，避免深层代理开销。
- 频繁变化的局部状态不要提升到全局，减少无谓的跨组件更新。
- `v-show` 用于频繁切换、`v-if` 用于条件渲染且初始可能为假；不要用 `v-show` 包裹重组件（首次渲染成本仍在）。
- 组件库的表格列定义、表单规则等配置对象放模块作用域常量或 `computed`，不要每次渲染重建。
- 图片、图标、图表按需加载；重型组件（富文本、地图、PDF 预览）用 `defineAsyncComponent` 异步引入。

## 10. 可维护性红线

1. 禁止在组件里写 `setTimeout` 做"等接口返回"的 hack，用 `await` 或事件。
2. 禁止用 `nextTick` 堆叠规避响应式问题，先排查为什么没更新。
3. 禁止 `$parent` / `$refs` 深层调用另一组件的方法，改为 props/emits 或 store。
4. 禁止在 `watch` 里修改被监听的源，形成循环更新。
5. 禁止把 `props` 直接 `v-model` 绑定到输入框（Vue 3 会警告，且违反单向数据流）。
6. 禁止提交调试代码：`console.log`、`debugger`、注释掉的整段逻辑。
7. 复杂模板必须加注释说明业务含义（`<!-- 已发货订单不支持取消 -->`），不要注释语法本身。

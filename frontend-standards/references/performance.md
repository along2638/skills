# 性能规范

## 1. 指标与目标

| 指标 | 含义 | 后台系统目标 | C 端页面目标 |
|---|---|---|---|
| LCP | 最大内容渲染 | < 2.5s | < 2.0s |
| INP | 交互到下一帧绘制 | < 200ms | < 150ms |
| CLS | 累计布局偏移 | < 0.1 | < 0.05 |
| TTFB | 首字节时间 | < 500ms | < 300ms |
| 首屏 JS（gzip） | 首屏必需脚本 | < 300KB | < 200KB |
| 单路由 chunk | 单个懒加载块 | < 300KB | < 250KB |

后台系统允许首屏稍慢（用户停留时间长、网络环境可控），但**交互响应必须快**——表格操作卡顿比首屏慢一秒更影响效率。

测量方式：

- 本地：Chrome DevTools Performance 面板 + Lighthouse，固定"移动端 4G 降速"条件。
- 线上：`web-vitals` 采集真实用户数据（RUM），不要只看实验室数据。

```ts
// src/utils/vitals.ts
import { onCLS, onINP, onLCP, onTTFB } from 'web-vitals'

export function reportVitals() {
  const send = (metric: { name: string; value: number; id: string }) => {
    navigator.sendBeacon?.(
      '/api/v1/metrics/vitals',
      JSON.stringify({ name: metric.name, value: metric.value, id: metric.id, path: location.pathname }),
    )
  }

  onLCP(send)
  onINP(send)
  onCLS(send)
  onTTFB(send)
}
```

用 `sendBeacon` 而不是 `fetch`：页面卸载时请求不被中断，且不阻塞主线程。

## 2. 首屏与分包

- **路由必须懒加载**：`component: () => import('@/views/order/OrderListView.vue')`。静态 import 会把所有页面打进首屏 chunk。
- `manualChunks` 按"变动频率"分包：Vue 全家桶、组件库各成一块，业务代码一块。发版后第三方库 hash 不变，用户只下载业务 chunk。
- 单个 chunk 超过 500KB（gzip 前）必须处理，处理顺序：
  1. 检查是否引入了整库（`moment`、全量 `lodash`、全量 `echarts`）；
  2. 改成按需引入（`echarts/core` + 具体图表类型）；
  3. 拆分路由；
  4. 重型模块用 `defineAsyncComponent` 延迟到真正需要时加载。
- 首屏不需要的第三方能力（导出 Excel、PDF 预览、地图）一律异步加载。

```ts
const ExportDialog = defineAsyncComponent(() => import('@/components/order/OrderExportDialog.vue'))
```

- 预加载只对"下一步极可能访问"的路由使用，滥用 `<link rel="prefetch">` 会抢首屏带宽。

## 3. 图片与媒体

- 格式优先级：AVIF / WebP → PNG（需要透明）→ JPG。构建时不内联大图（`assetsInlineLimit` 保持默认 4KB）。
- 所有 `<img>` 声明 `width` / `height` 或使用 `aspect-ratio`，避免图片加载后布局跳动（CLS 的主要来源）。
- 首屏外的图片加 `loading="lazy"` 与 `decoding="async"`；首屏主图不要 lazy，会拖慢 LCP。
- 大图走 CDN 缩放参数（`?w=400`），不要上传 3000px 原图再用 CSS 缩小。
- 图标用 SVG 组件或字体图标，不用多套 PNG 精灵图。
- 视频不自动播放，需要时 `preload="none"` + `poster`。

## 4. 长列表

| 数据量 | 方案 |
|---|---|
| < 200 条 | 普通 `v-for` / `el-table` |
| 200 ~ 5000 条 | 虚拟滚动（`el-table-v2`、`vue-virtual-scroller`） |
| > 5000 条 | 后端分页 + 虚拟滚动，禁止一次性拉全量 |

规则：

- 虚拟滚动要求每行等高；不定高时设置 `estimatedSize` 并接受少量偏差。
- 虚拟列表内不要用 `el-tooltip` 包裹每个单元格（实例数量爆炸），改用原生 `title`。
- 分页大小上限 100，见 `references/state-and-data.md`；不允许"加载全部"按钮。
- 无限滚动必须防抖 + 加载中互斥，否则滚动到底会连发 5 个请求。

## 5. 渲染优化

- `computed` 缓存派生值，禁止在模板里调用生成新对象的方法：

```vue
<!-- 反例：每次渲染都新建数组，触发无限更新 -->
<div v-for="item in getFilteredList()" :key="item.id" />

<!-- 正例 -->
<div v-for="item in filteredList" :key="item.id" />
```

- `v-if` 与 `v-show` 的选择：频繁切换用 `v-show`，初始可能为假且渲染成本高用 `v-if`。
- `key` 用业务 ID，保证 diff 正确复用；用 index 在排序/删除时会导致状态错位与多余渲染。
- 大列表项拆成独立子组件，让更新范围收敛到变化的行；不要把所有行渲染逻辑写在一个大模板里。
- `v-memo` 只在确实存在"大量静态行 + 个别变化"时使用，起手就加会让代码难读且收益不确定。
- 频繁更新的数值（倒计时、进度、鼠标位置）用 `shallowRef` 或直接操作 DOM，不要触发整棵子树更新。

## 6. 响应式开销

```ts
import { markRaw, shallowRef } from 'vue'

// 图表实例、地图实例、富文本编辑器：不能被代理
const chart = shallowRef<echarts.ECharts>()
const mapInstance = markRaw(new Map())

// 只读的大配置
const columns = shallowRef(TABLE_COLUMNS)
```

- 深层响应式会为每个嵌套属性建 `Proxy`，几千行数据全量代理会带来明显的初始化与内存开销。
- 只读数据用 `shallowRef`，替换时整体赋值。
- 第三方实例用 `markRaw` 或 `shallowRef`，否则库内部的对象比较会失效（典型症状：图表不更新、地图交互失灵）。
- 不要把所有状态都提升到 store：全局状态的每次变更都会触发所有订阅组件更新。

## 7. 请求优化

- 并发请求用 `Promise.all` 而非串行 `await`；但要控制并发数（浏览器同域 6 个连接，超出的会排队）。
- 搜索、筛选、输入联想用防抖（300ms）：

```ts
import { useDebounceFn } from '@vueuse/core'

const handleSearch = useDebounceFn((keyword: string) => {
  store.fetchList({ keyword, pageNum: 1 })
}, 300)
```

- 滚动、resize 用节流（100ms）。
- 参数变化时取消上一个请求，避免竞态与浪费带宽，写法见 `references/vue-coding.md`。
- 同一份数据被多个组件同时请求时，在 store 层做请求合并（记录 in-flight Promise），不要每个组件各发一次。
- 轮询只在页面可见时进行（`document.visibilityState`），离开页面或切到后台必须暂停。

## 8. 缓存策略

- 静态资源文件名带 hash，配 `Cache-Control: public, max-age=31536000, immutable`。
- `index.html` 不缓存（`no-cache`），否则用户拿到旧版本引用已删除的 hash 文件，出现白屏。
- 接口缓存由后端控制 `ETag` / `Cache-Control`，前端不另建缓存层。
- 需要离线可用时用 Service Worker，但必须有版本升级与"新版本可用，请刷新"的提示，不能静默换缓存导致新旧资源混用。

## 9. 监控与上报

```ts
// src/main.ts
app.config.errorHandler = (error, instance, info) => {
  reportError({ error, component: instance?.$options.name, info, route: router.currentRoute.value.fullPath })
}

window.addEventListener('unhandledrejection', (event) => {
  reportError({ error: event.reason, info: 'unhandledrejection' })
})
```

规则：

- 上报失败不影响业务：包在 `try/catch` 里，且不 `await`。
- 上报做采样（如 10%），避免流量与费用失控；错误类上报可以全量，性能指标采样。
- 上报内容不含用户敏感信息（手机号、身份证、token），见 `references/security.md`。
- 资源加载失败（`window.addEventListener('error', ..., true)`）单独上报，这类错误不会进 Vue 的 errorHandler。
- 每个上报带 `traceId` 或 `requestId`，便于与后端日志对齐。

## 10. 反模式清单

1. 首屏静态 import 所有页面。
2. `import _ from 'lodash'`、整包引入 `echarts`、`moment`。
3. 在 `watch` 里对大数组做 `map`/`filter` 再把结果写回另一个响应式变量。
4. 模板里调用 `formatDate(new Date())` 这类每次渲染都变的方法。
5. 表格里每一行都包 `el-tooltip` / `el-popover`。
6. 用 `setInterval` 轮询而不判断页面可见性。
7. 列表用 index 作 key 又支持排序/删除。
8. 图片不写宽高，导致 CLS 超标。
9. 为"性能"过早加 `v-memo`、`Object.freeze`，代码可读性下降但收益可忽略。
10. 只在开发机测性能，不看真实用户数据。

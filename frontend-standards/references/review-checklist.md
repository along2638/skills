# 前端代码评审清单

## 1. 直接阻断合并的情况

出现以下任一项，不进入细节讨论，先改：

- [ ] 页面或组件里直接调用 `axios` / `fetch`，绕过 `api/request.ts`。
- [ ] 使用 `any`，或用 `as any`、`@ts-ignore` 绕过类型错误且无说明。
- [ ] 直接修改 `props`，或用 `$parent` / `$refs` 跨组件改变子组件内部状态。
- [ ] `v-for` 用 `index` 作 `key`，且列表存在增删或排序。
- [ ] `v-html` 渲染用户输入且未经过 `DOMPurify`。
- [ ] 引入第二套 UI 组件库，或新增基线外重型依赖（图表、富文本、地图）未说明体积与替代方案。
- [ ] 提交了 `node_modules`、`dist`、`.env.local`、含密钥的文件。
- [ ] 存在未清理的 `setInterval`、`addEventListener`、`AbortController`、`ResizeObserver`。
- [ ] 异步请求无竞态处理，快速切换参数时旧结果覆盖新结果。
- [ ] 表单提交无防重复，连点产生多条数据。
- [ ] 新增 composable / store action / 工具函数没有单测，或存在 `.skip` 的测试。
- [ ] `pnpm lint` 或 `pnpm typecheck` 未通过。

## 2. 结构与分层

- [ ] 页面在 `views/`，可复用组件在 `components/`，一个页面的私有组件在其 `views/{domain}/components/`。
- [ ] `components/base/` 下的组件不 import `stores`、`api`、`router`。
- [ ] 依赖方向单向：`views → components / composables → api`，无反向 import。
- [ ] `utils/` 内是纯函数，不依赖 Vue 运行时。
- [ ] store 之间没有 action 互相编排，跨域逻辑在 composable 或页面。
- [ ] 单文件组件 ≤ 400 行，模板嵌套 ≤ 4 层。
- [ ] 没有"万能组件"：props 未超过 7 个，没有用大量 `v-if` 应对变体。

## 3. 路由与权限

- [ ] 路由组件全部懒加载，`router` 文件顶部没有静态 import 页面。
- [ ] 路由 `name` 唯一，跳转用 `name` 而非硬编码路径。
- [ ] 需要鉴权的路由配置了 `meta.permission`，`meta.title` 不为空。
- [ ] 按钮级权限用 `v-permission`，权限点命名与后端一致。
- [ ] 守卫中无死循环风险（`fetchProfile` 失败会退出，不会无限重试）。
- [ ] `redirect` 回跳参数做了同站相对路径校验。
- [ ] `keep-alive` 缓存的页面实现了 `onActivated` 刷新，离开时暂停轮询。

## 4. Vue 编码

- [ ] 全部使用 `<script setup lang="ts">`，无 mixin、无 Options API 混写。
- [ ] 组件名与文件名 PascalCase 多单词，模板中引用一致。
- [ ] props 有类型和默认值，对象默认值是工厂函数。
- [ ] emits 显式声明，事件名表达"发生了什么"。
- [ ] `defineModel` 与手动 `emit('update:xxx')` 没有同时使用。
- [ ] `v-if` 与 `v-for` 不在同一元素；过滤逻辑在 `computed` 中完成。
- [ ] 模板中无方法调用生成数组/对象的写法，无多层三元嵌套。
- [ ] `watch` 监听具体来源，`deep` 只用于小对象，异步 watch 有竞态处理。
- [ ] 用 `computed` 派生而非 `watch` 回写第二份状态。
- [ ] `onUnmounted` / `onScopeDispose` 中清理了所有副作用。
- [ ] 没有 `console.log`、`debugger`、注释掉的整段逻辑。

## 5. TypeScript

- [ ] 接口函数返回值类型显式声明，`request<T>` 的 `T` 与后端契约一致。
- [ ] 类型断言只出现在 `api/` 边界与 `import.meta.env`，业务代码内无 `as`。
- [ ] 状态与枚举用 `as const` 字面量联合，未用 `enum`。
- [ ] 表单类型从接口类型用 `Pick` / `Omit` 派生，未重复定义。
- [ ] 可辨识联合处理加载/错误状态，未用多个互不约束的布尔值。
- [ ] `vue-tsc --noEmit` 无错误（Vite build 不检查类型，不能只看构建通过）。

## 6. 状态与请求

- [ ] 只有跨页面共享的数据进 Pinia，页面内状态用 `ref`。
- [ ] store 的 state 无冗余派生数据，派生值用 getter 或 `computed`。
- [ ] store 中无 DOM、组件实例、路由实例。
- [ ] action 内不弹 toast、不跳路由，UI 反馈由页面负责。
- [ ] 请求层统一处理 HTTP 错误与业务错误码，401 集中跳登录且不会重复跳转。
- [ ] 请求可取消：切换筛选、离开页面时 `abort()`。
- [ ] 页面三态齐全：`loading` 有骨架屏或 loading，`error` 有重试入口，`empty` 有引导文案。
- [ ] 同一份列表数据只在一处维护，没有 store 与页面各存一份再手动同步。
- [ ] 金额只做格式化展示，未在前端做加减乘除。
- [ ] `localStorage` 只存 token 与非敏感偏好，键名带项目前缀，读写有封装。

## 7. 接口契约对齐

- [ ] 路径符合 `/api/v1/{复数资源}`，动作用子路径（`/orders/{id}/cancel`），无 `/order/list` 这类动词式路径。
- [ ] `baseURL` 只配一次（`/api/v1`），接口文件里不重复写版本前缀。
- [ ] 响应包字段为 `code` / `message` / `data` / `traceId`，业务判断只用 `code === 0`。
- [ ] 分页响应按 `records` / `total` / `pageNum` / `pageSize` / `pages` 取值，未使用 `list`。
- [ ] 当前页以响应返回的 `pageNum` 为准回填，不用请求参数硬覆盖。
- [ ] ID 与金额按字符串处理，未做 `Number()` 转换，金额未参与前端运算。
- [ ] 时间格式与后端一致（`yyyy-MM-dd HH:mm:ss`，东八区），解析时显式指定格式。
- [ ] 枚举字符串与后端 code 一致，未知值有兜底展示，不会渲染空白。
- [ ] 空列表按 `[]`、空对象按 `{}` 处理，未到处写 `?.` 与 `|| {}` 防御性代码。
- [ ] 校验失败（400 / 10001）时读取 `data` 的 `{ field, message }[]` 并回填表单。
- [ ] 401 / 403 / 429 / 409 有独立分支处理，未统一按"网络异常"提示。
- [ ] 创建、支付、退款类请求带 `Idempotency-Key`，重试复用同一个 key。
- [ ] 下载接口用 `responseType: 'blob'`，并处理"后端返回 JSON 错误被当文件"的情况。
- [ ] 错误对象保留 `traceId`，用户反馈问题时可提供。
- [ ] 后端新增字段 / 新增枚举值时前端不报错（可兼容性已确认）。

## 8. 样式与交互

- [ ] `<style scoped lang="scss">`，穿透只用 `:deep()`。
- [ ] 颜色、间距、圆角、字号取自 CSS 变量，无裸色值。
- [ ] 嵌套不超过 3 层，无 `!important`（组件库覆盖除外且已收口到 `styles/element.scss`）。
- [ ] 组件库定制通过 CSS 变量覆盖，未修改 `node_modules`。
- [ ] 布局用 flex / grid，无 `float` 做常规布局。
- [ ] 响应式断点使用统一 mixin，未在组件里散落魔法数字。
- [ ] 图片有 `alt`，表单控件有 `label`，可键盘操作，焦点样式未被移除。
- [ ] 文案无硬编码中文（有国际化需求时），日期与金额走 dayjs / `Intl`。
- [ ] 加载、禁用、hover、focus、错误五种视觉状态都已设计。

## 9. 质量门禁

- [ ] `pnpm lint` 通过，无 `eslint-disable` 批量注释。
- [ ] `pnpm typecheck` 通过。
- [ ] `pnpm test` 通过，新增逻辑有对应单测。
- [ ] 提交信息符合 Conventional Commits（`feat` / `fix` / `refactor` / `docs` / `test` / `chore`）。
- [ ] 未使用 `--no-verify` 绕过 hook。
- [ ] 依赖变更后 `pnpm-lock.yaml` 同步更新，无重复依赖副本（`pnpm why` 检查）。
- [ ] 构建产物可正常访问：`base` 路径、history 模式 fallback 均已确认。

## 10. 性能

- [ ] 路由懒加载；新增页面未被打进首屏 chunk。
- [ ] 单 chunk > 500KB 已用 `manualChunks` 拆分或按需加载。
- [ ] 列表超过 200 条使用虚拟滚动，未一次性渲染。
- [ ] 大对象使用 `shallowRef` / `markRaw`，未把第三方实例放进深层响应式。
- [ ] 图片使用合适格式与尺寸，`loading="lazy"`，有固定宽高避免 CLS。
- [ ] 搜索、滚动等高频操作有防抖/节流。
- [ ] 未在 `watch` 中做重计算，未在模板中调用生成对象的方法。
- [ ] 新增第三方脚本（统计、客服）异步加载且不影响首屏。

## 11. 安全

- [ ] 无 `v-html` 未净化，无 `eval` / `new Function`。
- [ ] token 不写死在代码里，密钥不出现在 `VITE_` 变量中。
- [ ] 生产构建去除 `console` 与 sourcemap 公开路径已确认。
- [ ] `target="_blank"` 均带 `rel="noopener noreferrer"`。
- [ ] 上传文件前端校验类型与大小，预览后 `revokeObjectURL`。
- [ ] 外部跳转地址走白名单校验。
- [ ] 权限判断只用于展示，未当作安全措施（接口鉴权由后端保证）。
- [ ] `pnpm audit` 无高危漏洞，或已记录处置方案。

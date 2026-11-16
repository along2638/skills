---
name: frontend-standards
description: 前端开发规范（Vue 3 + TypeScript + Vite + Pinia + Element Plus）。用于新建或改造前端项目的目录结构与分层、依赖与构建配置、Vue 组件与 Composition API 写法、TypeScript 类型设计、Pinia 状态与请求层、样式与组件库定制、路由与权限、代码质量工具链、测试、性能与安全；也用于按规范审查前端代码、排查不符合项或回答团队规范问题。
---

# 前端开发规范

## 适用范围（先确认）

- 本规范基线为 **Vue 3.4 + TypeScript 5.4 + Vite 5 + Pinia 2 + Element Plus 2**，包管理 pnpm，测试 Vitest + Playwright。
- 若项目是 **Vue 2.x + Options API + webpack**：不要直接套用组合式 API、`<script setup>`、Pinia、Vite 配置；只借用目录分层、请求层、类型设计、质量门禁的思想，并按实际版本重写。
- 若项目是 **React / 小程序 / uni-app**：本规范不适用，只可参考分层与工程化部分。
- 若项目是 **Nuxt**：目录约定换成 Nuxt 约定（`pages/`、`server/`、`composables/` 自动导入），`onMounted` 相关规则需按 SSR 语义调整。
- 生成代码必须能通过 `tsc --noEmit` 与 ESLint；不确定项目版本时先读 `package.json`。

## 基线（不可协商）

| 项 | 规定 |
|---|---|
| 运行时 | Node.js 20 LTS（>= 20.11），`engines` 与 `.nvmrc` 锁定 |
| 包管理 | pnpm 9+，只允许 `pnpm-lock.yaml` |
| 框架 | Vue 3.4.x，Composition API + `<script setup lang="ts">` |
| 语言 | TypeScript 5.4，`strict: true`（含 `noUncheckedIndexedAccess`） |
| 构建 | Vite 5.2 |
| 路由 | Vue Router 4.3，history 模式 |
| 状态 | Pinia 2.1（禁用 Vuex） |
| UI | Element Plus 2.6（主选）；换库需全项目一致，禁止两套并存 |
| 请求 | Axios 1.7，统一封装在 `src/api/request.ts` |
| 样式 | Sass（dart-sass）+ CSS 变量（禁用 Less） |
| 工具库 | VueUse 10.9、lodash-es 4.17（按需引入） |
| 日期 | dayjs（Element Plus 已内置，禁止 moment） |
| 代码质量 | ESLint 9 flat config + Prettier 3 + Stylelint 16 |
| 提交 | husky 9 + lint-staged 15 + commitlint 19（Conventional Commits） |
| 测试 | Vitest 1.6 + @vue/test-utils 2.4 + Playwright 1.45 |

## 目录结构骨架

```text
src
├── api              接口定义（按业务域分文件）+ request.ts 请求实例
├── assets           静态资源（字体、图片，需经过构建）
├── components       跨页面复用的组件，不含业务接口调用
│   └── order        OrderStatusTag / OrderFilterBar 等业务通用组件
├── composables      组合式函数（useXxx.ts），承载可复用逻辑
├── directives       自定义指令（v-permission、v-debounce）
├── layouts          布局壳层（BasicLayout、BlankLayout）
├── router           路由表与守卫
│   └── modules      按业务域拆分的路由模块
├── stores           Pinia store（useXxxStore.ts）
├── styles           全局样式、变量、mixin、组件库覆盖
├── types            全局类型与接口契约（api 类型就近放 api 目录亦可）
├── utils            无状态纯函数（format、validate、storage）
├── views            页面，按路由层级组织
├── App.vue
└── main.ts
```

`views` 反映路由结构，`components` 反映复用关系，两者不要混。只有一个业务域时业务组件直接放 `components/{domain}/`，出现第三个业务域时再考虑抽 `packages/` 或微前端，不要提前拆分。

## 工作流程

### 新建项目

1. 阅读 `references/architecture.md`、`references/tech-stack.md`、`references/quality-tooling.md`。
2. 按顺序输出：目录树 → `package.json` → `vite.config.ts` → `tsconfig.json` → ESLint/Prettier/Stylelint 配置 → 请求层 → 路由与布局 → 一个示例页面。
3. 所有文件给完整内容，不给省略号、不写 `TODO`；import 必须齐全，依赖版本与基线一致。

### 新增一个页面 / 功能（垂直切片）

1. 阅读 `references/architecture.md`、`references/vue-coding.md`、`references/state-and-data.md`。
2. 实现顺序：类型定义 → 接口函数 → store（需要时）→ 子组件 → 页面 → 路由注册 → 单元测试。
3. 同步补齐：三态（loading / error / empty）、表单校验、按钮权限、错误兜底、骨架屏或空态文案。
4. 输出后逐条对照 `references/review-checklist.md` 自检。
5. 需要端到端参照时，直接看 `references/example-feature.md` 的订单列表页切片。

### 代码审查

1. 以 `references/review-checklist.md` 为主，按需读对应专题文件。
2. 输出按严重级别分组：**阻断**（违反硬规则 / 有缺陷）、**建议**（可读性 / 优化）。
3. 每条给出 `文件:行号`、违反的规则、最小修复方案；不要无故重写整个文件。
4. 检查是否存在"能跑但会埋雷"的写法：内存泄漏、竞态、响应式丢失、`any` 扩散。

### 规范问答

- 先给结论，再引用本 skill 的具体规则与参考文件。
- 项目现状与规范冲突时，说明偏差、风险与迁移成本，不默认项目已合规。

## 不可违反的硬规则

**结构**

1. 页面放 `views/`，复用组件放 `components/`；请求只写在 `api/`，页面和组件都不写 `axios`。
2. 依赖方向 `views → components / composables → api`；禁止兄弟业务模块互相 import 形成循环依赖。
3. 一个 SFC 只放一个组件；单文件 ≤ 400 行，模板嵌套 ≤ 4 层，超出必须拆子组件或抽 composable。
4. 全局状态只放 `stores/`，跨组件共享逻辑优先 composable；不把 `stores/` 当万能缓存。

**技术栈与工程化**

5. 只用 pnpm，`pnpm-lock.yaml` 必须入库；禁止 npm / yarn 混用产生多份 lock。
6. 不引入基线外依赖；UI 组件库只能有一套；日期统一 dayjs；禁用 moment、Vuex、jQuery、`vue-resource`。
7. 环境变量只允许 `VITE_` 前缀，且**不得包含任何密钥**——前端变量会原样打进产物。

**Vue 编码**

8. 新代码一律 `<script setup lang="ts">` + Composition API；禁止 mixin、禁止 Options API 混写。
9. 组件文件名 PascalCase 且多单词（`OrderList.vue`）；`components/index.ts` 不导出默认对象，按需具名导出。
10. props 必须声明类型与默认值（`withDefaults`）；禁止直接修改 props；子传父必须用 `defineEmits` 显式声明。
11. `v-for` 必须带稳定 `key`（业务 ID），禁用 index 作 key；`v-if` 与 `v-for` 不写在同一个元素上。
12. 禁止用 `document.querySelector` 直接操作 DOM；必须操作时用模板 `ref`，且在 `onMounted` 之后。
13. 数据请求放在 `onMounted` 或显式动作中，`setup` 顶层不发起不可控的异步副作用。
14. `watch` 必须明确是否需要 `immediate` / `deep`；禁止深层 watch 大对象，优先 `computed`。
15. `v-html` 默认禁用；确需渲染富文本必须先 `DOMPurify.sanitize`。

**TypeScript**

16. 禁止 `any`；需要逃生时用 `unknown` + 类型收窄；禁 `@ts-ignore`，必须用 `@ts-expect-error` 并写明原因。
17. 请求函数必须泛型化（`request<T>`）并在调用处声明响应类型；禁止 `res.data as any` 扩散到业务层。
18. 状态与枚举用 `as const` 的字面量联合，不用 `enum`；与后端 code 字符串一一对应。

**状态与请求**

19. 所有请求走 `src/api/request.ts`；业务错误统一包装为 `BizError`，401 集中处理，不在每个页面各写一套。
20. `loading` / `error` / `empty` 三态必须处理，禁止只写 loading，或错误时页面白屏无反馈。
21. 同一份列表数据只在页面或 store 一处维护，禁止两处并存再手动同步。
22. Pinia 中禁止存 DOM、组件实例、路由实例；state 只存必要数据，派生值用 getter 或 `computed`。

**样式与交互**

23. 样式默认 `<style scoped lang="scss">`；穿透只用 `:deep()`，禁用 `::v-deep`、`/deep/`。
24. 颜色、间距、圆角、字号必须用 CSS 变量或 Sass 变量，禁止在业务组件写裸色值。
25. 提交类操作必须防重复：请求中禁用按钮 + 状态标记，不能只靠用户不连点。
26. 表单前端校验规则必须与后端一致（必填、长度、格式、范围），并在提交前统一触发。

**质量、测试、性能与安全**

27. 提交前必须通过 `pnpm lint`、`pnpm typecheck`（`vue-tsc --noEmit`）、单测；Vite 构建不做类型检查，不能靠 `build` 通过判定类型正确。
28. 新增 composable、store action、工具函数必须有单元测试；主流程（登录、下单）必须有 E2E。
29. 路由必须懒加载；单 chunk > 500KB 必须拆分；列表超过 200 条必须虚拟滚动。
30. token 与密钥不进源码；生产构建禁用 `console.log`；`target="_blank"` 必须带 `rel="noopener noreferrer"`。

## 组件与 store 约定（强制）

- 组件分三类，职责不混：
  - **基础组件**（`components/base/`）：只接收 props / 抛事件，不发请求、不读 router、不读 store。
  - **业务组件**（`components/{domain}/`）：可读 store、可调 api，但不包含页面级布局与路由跳转。
  - **页面**（`views/`）：负责编排、路由参数、权限、三态与错误提示。
- 组件对外只暴露 props / emits / slots，不暴露内部方法；父组件要控制子组件时，改为受控 props。
- store 用 setup 语法，命名 `useXxxStore`；一个文件一个 store；action 内不发 toast、不跳路由，UI 反馈由页面决定。
- 组合式函数命名 `useXxx`，返回值必须是对象或元组；内部创建的定时器、监听、`AbortController` 必须在 `onScopeDispose` / `onUnmounted` 中清理。

## 关键决策速查

| 场景 | 规定 |
|---|---|
| 组件写法 | `<script setup lang="ts">`，顺序 `script` → `template` → `style` |
| 响应式 | 基本类型 `ref`，对象表单 `reactive`，只读大对象 `shallowRef` + `markRaw` |
| 派生的派生 | 用 `computed`，禁止用 `watch` 回写第二个变量 |
| v-model | 单值用 `defineModel()`；多值用 `defineModel('xxx')` 或 `props + emit('update:xxx')` |
| 父调子方法 | 改为受控 props 或 `defineExpose` 暴露最小接口，不暴露整个实例 |
| 组件通信 | 父子 props/emits → 跨层 provide/inject → 跨页面 store → 全局事件禁用 |
| 请求取消 | 组件卸载或参数变化时 `AbortController.abort()`，避免竞态覆盖 |
| 列表 key | 业务 ID（`row.id`），无 ID 时用后端返回的唯一键 |
| 表单校验 | Element Plus `rules` + `formRef.validate()`，提交前统一触发 |
| 金额展示 | 后端返回字符串，前端只格式化展示，不做加减乘除 |
| 时间展示 | dayjs 格式化，禁止手写 `getMonth() + 1` 拼接 |
| 权限 | 路由级守卫 + 按钮级 `v-permission`，接口鉴权仍由后端负责 |
| 全局状态边界 | 跨页面共享才进 Pinia；单页面内部用 `ref` / composable |
| 环境变量 | `VITE_API_BASE_URL` 等非敏感配置，密钥一律走服务端代理 |
| 组件库定制 | 覆盖 CSS 变量（`--el-color-primary`），不改 node_modules 源码 |
| 图标 | 统一一套图标库（Element Plus 图标或 `unplugin-icons`） |
| 打包 | 路由懒加载 + `manualChunks` 拆 vendor；`dist` 不入库 |

## 输出要求

- 使用中文说明，代码标识符、文件名、路由路径用英文。
- 生成代码给出完整文件路径与可直接落盘的内容；新增依赖说明版本与用途，并确认在 `references/tech-stack.md` 的版本矩阵内。
- 涉及接口时同时给出请求/响应类型定义，字段与后端契约一致（`code`、`message`、`data`、`traceId`）。
- 用户项目现状未知时，先读 `package.json`、`vite.config.ts` 与相邻页面再动手，不凭空假设目录结构与别名。
- 不要主动引入状态管理以外的重型依赖（图表库、富文本、地图、微前端），确需时先说明体积与替代方案。

## 参考文件

| 文件 | 何时阅读 |
|---|---|
| `references/architecture.md` | 建项目、目录分层、路由与权限、布局、模块边界 |
| `references/tech-stack.md` | 选型、版本矩阵、依赖治理、vite 配置、环境变量 |
| `references/vue-coding.md` | 组件写法、props/emits、响应式、生命周期、模板指令 |
| `references/typescript.md` | 类型设计、泛型、类型收窄、tsconfig、禁用 any 的替代写法 |
| `references/state-and-data.md` | Pinia、请求层封装、错误处理、三态、缓存与表单 |
| `references/ui-and-styles.md` | 样式分层、设计令牌、组件库定制、响应式、可访问性、国际化 |
| `references/quality-tooling.md` | ESLint/Prettier/Stylelint、husky、commit 规范、CI |
| `references/testing.md` | 单元测试、组件测试、接口 Mock、E2E、覆盖率 |
| `references/performance.md` | 首屏、分包、长列表、渲染优化、监控指标 |
| `references/security.md` | XSS、token 存储、权限边界、依赖安全、上传与重定向 |
| `references/review-checklist.md` | 提交前自检与代码评审 |
| `references/example-feature.md` | 需要完整可照抄的订单列表页垂直切片 |

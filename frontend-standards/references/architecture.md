# 架构与目录规范

## 1. 目录结构

```text
project
├── public                 不参与构建的静态文件（favicon、robots.txt）
├── src
│   ├── api                接口层：一个业务域一个文件，另有 request.ts
│   │   ├── request.ts     Axios 实例、拦截器、BizError
│   │   ├── order.ts
│   │   └── user.ts
│   ├── assets             参与构建的资源（图片、字体、svg）
│   ├── components         跨页面复用组件
│   │   ├── base           Button/Table/Modal 等无业务的基础封装
│   │   └── order          OrderStatusTag 等业务通用组件
│   ├── composables        组合式函数，useXxx.ts
│   ├── directives         自定义指令，index.ts 统一注册
│   ├── layouts            布局壳层
│   │   ├── BasicLayout.vue
│   │   └── BlankLayout.vue
│   ├── router
│   │   ├── index.ts       创建 router 实例、注册守卫
│   │   ├── routes.ts      静态路由（登录、404、首页）
│   │   └── modules        按业务域拆分的动态路由
│   ├── stores             Pinia store
│   ├── styles
│   │   ├── variables.scss   Sass 变量（断点、mixin 依赖）
│   │   ├── tokens.scss      CSS 变量（颜色、间距、圆角、阴影）
│   │   ├── reset.scss
│   │   ├── global.scss
│   │   └── element.scss     组件库覆盖
│   ├── types              全局类型、接口契约、env.d.ts
│   ├── utils              无状态纯函数
│   ├── views              页面，目录结构与路由层级对应
│   │   └── order
│   │       ├── OrderListView.vue
│   │       └── OrderDetailView.vue
│   ├── App.vue
│   └── main.ts
├── e2e                    Playwright 用例
├── .env.development
├── .env.production
├── eslint.config.js
├── vite.config.ts
├── vitest.config.ts
└── package.json
```

约定：

- `views` 的目录层级与路由路径保持一致：`views/order/OrderListView.vue` 对应 `/order/list`。
- `views` 目录下可以有 `components/` 子目录，只放该页面专用的组件；一旦被两个页面使用，就上移到 `src/components/{domain}/`。
- `utils` 只放纯函数：给定输入必定得到确定输出，不读 store、不发请求、不碰 DOM。有副作用的逻辑放 `composables`。
- `types` 只放跨模块类型；单个接口的请求/响应类型就近定义在 `api/xxx.ts` 顶部，避免全局类型文件膨胀成杂物间。

## 2. 分层与职责

| 层 | 位置 | 职责 | 不该做的事 |
|---|---|---|---|
| 页面 | `views/` | 编排、路由参数、权限判断、三态、错误提示 | 直接调 `axios`、写复杂业务计算 |
| 业务组件 | `components/{domain}/` | 展示 + 局部交互，可读 store | 页面级布局、路由跳转、全局提示 |
| 基础组件 | `components/base/` | 纯展示与交互封装 | 发请求、读 store、读 router |
| 组合式函数 | `composables/` | 可复用逻辑（筛选、分页、上传、权限） | 返回 DOM、持有全局单例状态 |
| 状态 | `stores/` | 跨页面共享数据与动作 | 存组件实例、发 toast、跳路由 |
| 接口 | `api/` | URL、入参、响应类型、请求实例 | 写业务分支、做数据加工 |
| 工具 | `utils/` | 格式化、校验、存储封装 | 依赖 Vue 运行时 |

依赖方向必须单向：

```text
views → components → composables → stores → api → request.ts
  ↘________________________________↗
```

反向依赖是常见事故源：`api` 里 import store 会导致循环依赖与测试困难，接口层拿到 token 应该由 `request.ts` 从 `storage` 读取，而不是读 store。

## 3. 路由与权限

### 3.1 路由表组织

静态路由（无需登录）与业务路由分离，业务路由按域拆到 `router/modules/`，由 `import.meta.glob` 自动收集：

```ts
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'
import { staticRoutes } from './routes'

const modules = import.meta.glob<{ default: RouteRecordRaw }>('./modules/*.ts', { eager: true })

const dynamicRoutes: RouteRecordRaw[] = Object.values(modules).map((mod) => mod.default)

export const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [...staticRoutes, ...dynamicRoutes],
  scrollBehavior: () => ({ top: 0 }),
})

export default router
```

```ts
// src/router/modules/order.ts
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
      meta: { title: '订单列表', permission: 'order:list' },
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

规则：

- 路由必须懒加载（`() => import()`），禁止在 `routes.ts` 顶部静态 import 页面组件。
- `name` 必须唯一且有意义，跳转优先用 `name` 而不是拼路径，路径改动时不会漏改。
- `meta` 至少包含 `title`，需要权限的加 `permission`，不在菜单展示的加 `hidden`。
- 路由 `path` 用小写 kebab-case：`/order/list`、`/user-center/profile`。

### 3.2 权限模型

| 层级 | 实现 | 作用 | 边界 |
|---|---|---|---|
| 路由级 | 全局前置守卫校验 `meta.permission` | 未授权页面不进入，避免白屏 | 只是体验控制 |
| 按钮级 | `v-permission` 指令移除 DOM | 隐藏无权操作 | 只是展示控制 |
| 接口级 | 后端鉴权 | 真正的安全边界 | 前端无法替代 |

```ts
// src/router/guard.ts
import type { Router } from 'vue-router'
import { useUserStore } from '@/stores/user'

const WHITE_LIST = ['/login', '/404', '/403']

export function setupGuard(router: Router) {
  router.beforeEach(async (to) => {
    const userStore = useUserStore()

    if (WHITE_LIST.includes(to.path)) {
      return true
    }

    if (!userStore.token) {
      return { path: '/login', query: { redirect: to.fullPath } }
    }

    if (!userStore.profile) {
      await userStore.fetchProfile()
    }

    const permission = to.meta.permission as string | undefined
    if (permission && !userStore.hasPermission(permission)) {
      return { path: '/403' }
    }

    return true
  })

  router.afterEach((to) => {
    const title = to.meta.title as string | undefined
    document.title = title ? `${title} - 运营后台` : '运营后台'
  })
}
```

注意：

- 守卫里只做判断与跳转，不做重业务；`fetchProfile` 失败要清 token 并跳登录，不能无限重试。
- `redirect` 参数回跳前必须校验是同站相对路径，见 `references/security.md` 的重定向一节。
- 不要在前端根据权限动态增删路由来实现"安全"，接口鉴权才是边界。

## 4. 布局与壳层

- 布局放 `layouts/`，只负责页面骨架（侧栏、顶栏、标签页、内容区 `<router-view>`），不放业务逻辑。
- 多套布局用 `route.meta.layout` 或路由层级区分：登录页用 `BlankLayout`，后台用 `BasicLayout`。
- 布局里的页面切换动画、`keep-alive` 缓存配置集中在这一层，页面不各自处理。
- `keep-alive` 只缓存明确需要的页面（列表页常见），通过 `route.meta.keepAlive` 声明；缓存页面必须实现 `onActivated` 刷新逻辑，见 `references/vue-coding.md`。

## 5. 组件分类与边界

| 类型 | 位置 | 判断标准 | 示例 |
|---|---|---|---|
| 基础组件 | `components/base/` | 换个项目也能用 | `BaseTable`、`ConfirmButton` |
| 业务组件 | `components/{domain}/` | 依赖本业务的数据结构 | `OrderStatusTag`、`OrderFilterBar` |
| 页面私有组件 | `views/{domain}/components/` | 只被一个页面使用 | `OrderExportDialog` |

三条边界：

1. 基础组件不 import `stores`、`api`、`router`，只通过 props/emits 与外界交互。
2. 业务组件可以读 store，但不做路由跳转；跳转由页面传入回调或在页面中监听事件。
3. 页面不为了"复用"把自身拆成 props 巨多的万能组件；先看能否抽 composable 复用逻辑，而不是复用模板。

## 6. composables 规范

```ts
// src/composables/usePagination.ts
import { computed, ref } from 'vue'

export interface UsePaginationOptions {
  pageSize?: number
  onChange?: () => void
}

export function usePagination(options: UsePaginationOptions = {}) {
  const pageNum = ref(1)
  const pageSize = ref(options.pageSize ?? 10)
  const total = ref(0)

  const pageCount = computed(() => Math.ceil(total.value / pageSize.value))

  function setTotal(value: number) {
    total.value = value
  }

  function changePage(next: number) {
    if (next < 1 || (pageCount.value > 0 && next > pageCount.value)) {
      return
    }
    pageNum.value = next
    options.onChange?.()
  }

  function reset() {
    pageNum.value = 1
    total.value = 0
  }

  return { pageNum, pageSize, total, pageCount, setTotal, changePage, reset }
}
```

规则：

- 命名 `useXxx`，文件名与函数名一致；一个文件一个 composable。
- 返回值用对象（便于按需解构），不要返回裸数组。
- 内部创建的定时器、事件监听、`AbortController`、`requestAnimationFrame` 必须在 `onScopeDispose` 或 `onUnmounted` 中清理，否则页面切换后仍在跑。
- composable 不返回 `ref` 包裹的 DOM 之外的浏览器对象；需要 DOM 时通过参数传入 `Ref<HTMLElement | null>`。
- 不要在 composable 里调用 `useRouter()` 后再在内部随意跳转，导航由调用方决定。

## 7. 模块边界与循环依赖

- 业务域之间不直接 import：订单页面需要用户信息时，读 `userStore` 或调用 `userApi`，不 import `views/user/...`。
- 出现循环依赖的信号：`pnpm build` 报 `Circular dependency`、某个模块初始化时为 `undefined`、单测里 mock 不生效。用 `madge --circular src` 排查。
- `stores` 之间允许互相读，但禁止 A 的 action 里调用 B 的 action 做编排；跨 store 编排放到 composable 或页面。
- `utils` 不允许 import 任何 Vue 相关模块，保证可在 Node 环境直接单测。

## 8. 多环境与构建入口

| 环境 | 命令 | 配置文件 | 用途 |
|---|---|---|---|
| 本地 | `pnpm dev` | `.env.development` | 代理到本地或测试后端 |
| 测试 | `pnpm build:test` | `.env.staging` | 提交测试验收 |
| 生产 | `pnpm build` | `.env.production` | 正式发布 |

- 环境差异只通过 `VITE_` 变量表达，不在代码里写 `if (process.env.NODE_ENV === 'production')` 分支处理业务逻辑。
- `main.ts` 只做装配：创建应用、注册插件、注册指令、挂载。任何业务初始化（拉用户信息）放路由守卫或 `App.vue` 的 `onMounted`。
- `dist` 与 `node_modules` 不入库；构建产物路径 `base` 若部署在子目录必须显式配置，否则静态资源 404。

## 9. 新建业务模块检查项

1. `views/{domain}/` 目录创建，页面文件按 `XxxListView.vue` / `XxxDetailView.vue` 命名。
2. `api/{domain}.ts` 定义入参、列表项、详情三个类型与对应函数。
3. 需要跨页面共享的数据才建 `stores/{domain}.ts`，否则用页面内 `ref`。
4. `router/modules/{domain}.ts` 注册路由，配齐 `title` / `permission` / 懒加载。
5. 权限点在后端与前端命名一致（`order:list`），并同步到 `v-permission` 使用的常量。
6. 至少一个页面级单测或 E2E 用例，覆盖主流程与错误分支。
7. 检查未引入新的重复依赖，`pnpm why` 无多余副本。

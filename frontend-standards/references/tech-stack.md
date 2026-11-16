# 技术栈与工程化规范

## 1. 版本矩阵

| 依赖 | 版本 | 用途 | 必选 |
|---|---|---|---|
| vue | 3.4.x | 框架 | 是 |
| vue-router | 4.3.x | 路由 | 是 |
| pinia | 2.1.x | 状态管理 | 是 |
| pinia-plugin-persistedstate | 3.2.x | store 持久化 | 按需 |
| element-plus | 2.6.x | UI 组件库 | 是（主选） |
| @element-plus/icons-vue | 2.3.x | 图标 | 是 |
| axios | 1.7.x | HTTP 客户端 | 是 |
| dayjs | 1.11.x | 日期处理 | 是 |
| @vueuse/core | 10.9.x | 组合式工具集 | 是 |
| lodash-es | 4.17.x | 通用工具（按需引入） | 按需 |
| sass | 1.77.x | 样式预处理 | 是 |
| typescript | 5.4.x | 类型系统 | 是 |
| vite | 5.2.x | 构建 | 是 |
| @vitejs/plugin-vue | 5.0.x | Vue 单文件组件支持 | 是 |
| vue-tsc | 2.0.x | 类型检查 | 是 |
| unplugin-auto-import | 0.17.x | API 自动导入 | 是 |
| unplugin-vue-components | 0.27.x | 组件按需引入 | 是 |
| eslint | 9.x | 代码检查 | 是 |
| typescript-eslint | 7.x | TS 规则 | 是 |
| eslint-plugin-vue | 9.26.x | Vue 规则 | 是 |
| prettier | 3.2.x | 格式化 | 是 |
| stylelint | 16.x | 样式检查 | 是 |
| husky | 9.x | Git hooks | 是 |
| lint-staged | 15.x | 暂存区检查 | 是 |
| @commitlint/cli | 19.x | 提交信息校验 | 是 |
| vitest | 1.6.x | 单元测试 | 是 |
| @vue/test-utils | 2.4.x | 组件测试 | 是 |
| jsdom | 24.x | 测试 DOM 环境 | 是 |
| @vitest/coverage-v8 | 1.6.x | 覆盖率 | 是 |
| msw | 2.x | 接口 Mock | 按需 |
| @playwright/test | 1.45.x | E2E | 是 |

版本策略：`package.json` 用 `^` 声明，真实版本由 `pnpm-lock.yaml` 锁定。跨 minor 升级（如 Vue 3.4 → 3.5）必须单独提交并跑完整回归，不与功能改动混在一起。

## 2. 依赖选型

新增依赖前先过四问：

1. 现有依赖能否覆盖？（`lodash-es`、`@vueuse/core` 已覆盖大量场景，别再引 `ramda`、`moment`）
2. 体积成本多少？（用 `bundlephobia` 看 gzip 后大小，超过 30KB 要说明理由）
3. 是否仍在维护？（最近一年有提交、issue 有回应、没有长期未修复的高危漏洞）
4. 是否支持 Vue 3 + TypeScript？（类型声明是否自带，不要靠 `@types/xxx` 猜实现）

禁止引入：

| 依赖 | 原因 | 替代 |
|---|---|---|
| moment | 体积大、已停止新功能开发 | dayjs |
| Vuex | Vue 3 已由 Pinia 取代 | pinia |
| jQuery | 与响应式渲染冲突 | 原生 API / VueUse |
| vue-resource | 停止维护 | axios |
| 第二套 UI 组件库 | 样式冲突、包体积翻倍 | 统一用 Element Plus |
| `request` / `superagent` | 与 axios 职责重复 | axios |
| 大型工具库全量引入（`lodash` 而非 `lodash-es`） | 无法 tree-shaking | `lodash-es` 按需导入 |

`lodash-es` 只允许具名导入：

```ts
import { debounce, cloneDeep } from 'lodash-es'
```

禁止 `import _ from 'lodash-es'`，那会把整个库打进产物。

## 3. 包管理

- 只用 pnpm，仓库只保留 `pnpm-lock.yaml`。发现 `package-lock.json` / `yarn.lock` 立即删除，混用会导致依赖树不一致与"本地能跑、CI 报错"。
- `package.json` 声明运行时约束：

```json
{
  "name": "admin-web",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=20.11.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "build:test": "vite build --mode staging",
    "preview": "vite preview",
    "lint": "eslint . --max-warnings 0",
    "lint:style": "stylelint \"src/**/*.{vue,scss,css}\"",
    "format": "prettier --write \"src/**/*.{ts,vue,scss,json,md}\"",
    "typecheck": "vue-tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "prepare": "husky"
  }
}
```

- `.nvmrc` 写 `20.11.1`，CI 与本地都按它选 Node 版本，避免 `node-sass` 这类二进制依赖的版本坑（本项目用 dart-sass，无此问题，但 Node 版本仍需一致）。
- 提交 `pnpm-lock.yaml`；CI 用 `pnpm install --frozen-lockfile`，本地不要随手 `--no-frozen-lockfile` 绕过。

## 4. 依赖治理

- 查依赖来源：`pnpm why axios`，确认没有两份不同版本同时存在。
- 去重：`pnpm dedupe`，多份副本会让体积和类型冲突同时出现。
- 过期检查：`pnpm outdated`，安全补丁随迭代升级，不必等大版本。
- 漏洞扫描：`pnpm audit --prod`，高危必须处置，中危记录工单。
- 禁止生产依赖指向 git 仓库或本地路径（`git+https://...`、`file:../`）：无法审计、无法离线构建、CI 不可复现。
- 禁止 `latest` 与 `*` 版本号；`alpha` / `beta` / `rc` 只能临时验证，不得进主干。

## 5. vite.config.ts

```ts
import { fileURLToPath, URL } from 'node:url'
import { defineConfig, loadEnv } from 'vite'
import vue from '@vitejs/plugin-vue'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_')

  return {
    base: env.VITE_PUBLIC_PATH || '/',
    plugins: [
      vue(),
      AutoImport({
        imports: ['vue', 'vue-router', 'pinia'],
        resolvers: [ElementPlusResolver()],
        dts: 'src/types/auto-imports.d.ts',
        eslintrc: { enabled: false },
      }),
      Components({
        resolvers: [ElementPlusResolver({ importStyle: 'sass' })],
        dts: 'src/types/components.d.ts',
        dirs: ['src/components'],
      }),
    ],
    resolve: {
      alias: {
        '@': fileURLToPath(new URL('./src', import.meta.url)),
      },
    },
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: `@use "@/styles/variables.scss" as *;`,
        },
      },
    },
    server: {
      port: 5173,
      open: false,
      proxy: {
        '/api': {
          target: env.VITE_PROXY_TARGET || 'http://127.0.0.1:8080',
          changeOrigin: true,
        },
      },
    },
    build: {
      sourcemap: false,
      chunkSizeWarningLimit: 500,
      rollupOptions: {
        output: {
          manualChunks: {
            vue: ['vue', 'vue-router', 'pinia'],
            element: ['element-plus'],
            vendor: ['axios', 'dayjs', 'lodash-es'],
          },
        },
      },
    },
  }
})
```

要点：

- `dts` 生成的类型文件要入库，否则 CI 首次类型检查会因缺少声明失败。
- `changeOrigin` 用于本地代理解决跨域，生产环境由网关统一处理，不靠前端代理。
- `manualChunks` 按"很少变动的库"分包，业务代码单独成块，发版后用户只需重新下载业务 chunk。
- 生产 `sourcemap: false`；需要排查线上问题时临时开启并上传到内网，不要留在公开路径。
- 不写 `define: { 'process.env': env }`，Vite 的 `import.meta.env` 已够用，注入整个 env 会泄露配置。

## 6. 环境变量

| 变量 | 示例 | 说明 |
|---|---|---|
| `VITE_APP_TITLE` | `订单后台` | 页面标题 |
| `VITE_API_BASE_URL` | `/api/v1` | 接口前缀（含版本），生产由网关转发 |
| `VITE_PROXY_TARGET` | `http://127.0.0.1:8080` | 仅本地代理使用 |
| `VITE_PUBLIC_PATH` | `/admin/` | 部署子目录时的 base |
| `VITE_ENABLE_MOCK` | `false` | 是否启用 MSW |

```bash
# .env.development
VITE_APP_TITLE=订单后台
VITE_API_BASE_URL=/api/v1
VITE_PROXY_TARGET=http://127.0.0.1:8080
VITE_ENABLE_MOCK=true
```

```ts
// src/types/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string
  readonly VITE_API_BASE_URL: string
  readonly VITE_PROXY_TARGET?: string
  readonly VITE_PUBLIC_PATH?: string
  readonly VITE_ENABLE_MOCK?: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

规则：

- 只有 `VITE_` 前缀的变量会被注入客户端，其余变量（含 `NODE_ENV`）在业务代码中不可用。
- **所有 `VITE_` 变量都会出现在产物里**，任何人打开 DevTools 都能看到。密钥、内部服务地址、第三方 secret 一律不得写入；需要调第三方时走后端代理。
- `.env.local` 只放个人本地配置，加入 `.gitignore`；`.env.development` / `.env.production` 入库以便团队一致。

## 7. 构建与部署

- 部署到子目录必须设置 `base`（或 `VITE_PUBLIC_PATH`），否则 `/assets/xxx.js` 404。
- `history` 模式需要服务端 fallback，否则刷新 404：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}

location /api/ {
  proxy_pass http://backend:8080/;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
}
```

- 静态资源按文件名 hash 做长缓存，`index.html` 不缓存：

```nginx
location ~* \.(js|css|woff2?|png|jpg|svg)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}

location = /index.html {
  add_header Cache-Control "no-cache";
}
```

- `dist/` 不入库，由 CI 构建并上传制品；构建产物不手工修改。
- 需要压缩时由 Nginx 或 CDN 做 gzip/brotli，不在前端插件里部署压缩逻辑。

# 代码质量与工程化

## 1. 工具链职责

| 工具 | 管什么 | 不管什么 |
|---|---|---|
| ESLint | 代码质量：未使用变量、错误的响应式用法、`any`、潜在 bug | 格式、样式 |
| Prettier | 格式：缩进、引号、换行、行宽 | 代码质量 |
| Stylelint | 样式：属性顺序、未知属性、废弃语法 | 逻辑 |
| vue-tsc | 类型 | 运行时行为 |
| Vitest | 行为 | 类型与风格 |

分工必须清楚：**ESLint 与 Prettier 的格式规则不允许重叠**（不安装 `eslint-config-prettier` 之外的格式化插件），否则同一条代码两个工具来回改。

## 2. ESLint 9 flat config

```js
// eslint.config.js
import js from '@eslint/js'
import ts from 'typescript-eslint'
import vue from 'eslint-plugin-vue'
import globals from 'globals'

export default ts.config(
  {
    ignores: ['dist', 'coverage', 'playwright-report', 'node_modules', 'src/types/auto-imports.d.ts', 'src/types/components.d.ts'],
  },
  js.configs.recommended,
  ...ts.configs.recommendedTypeChecked,
  ...vue.configs['flat/recommended'],
  {
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: 'module',
      globals: {
        ...globals.browser,
        ...globals.node,
      },
    },
    rules: {
      'no-console': ['error', { allow: ['warn', 'error'] }],
      'no-debugger': 'error',
      'no-duplicate-imports': 'error',
      'prefer-const': 'error',
      eqeqeq: ['error', 'always'],
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/await-thenable': 'error',
      'vue/multi-word-component-names': 'error',
      'vue/component-name-in-template-casing': ['error', 'PascalCase'],
      'vue/block-order': ['error', { order: ['script', 'template', 'style'] }],
      'vue/no-mutating-props': 'error',
      'vue/no-v-html': 'error',
      'vue/require-explicit-emits': 'error',
    },
  },
  {
    // .vue 文件需要 vue-eslint-parser + TS parser
    files: ['**/*.vue'],
    languageOptions: {
      parserOptions: {
        parser: ts.parser,
        extraFileExtensions: ['.vue'],
      },
    },
  },
  {
    // 配置文件自身不需要类型检查规则
    files: ['*.config.{js,ts}', 'eslint.config.js'],
    ...ts.configs.disableTypeChecked,
  },
)
```

要点：

- 用 `recommendedTypeChecked` 而不是 `recommended`：`no-floating-promises`、`await-thenable` 这类规则能拦住大量异步 bug，代价是要求 `parserOptions.projectService`（或 `project`）配置正确。
- `vue/no-v-html` 设为 `error`，确需渲染富文本时按 `references/security.md` 显式关闭并写明原因。
- 不使用 `eslint-plugin-prettier`（把 Prettier 当 ESLint 规则跑），速度慢且报错噪音大。
- 忽略 `dts` 自动生成文件，否则每次 `pnpm dev` 都会产生新警告。

需要局部豁免时，必须带原因：

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- 第三方 SDK 回调参数无类型定义
window.payCallback = (result: any) => { /* ... */ }
```

批量 `/* eslint-disable */` 放文件头视为阻断项。

## 3. Prettier

```json
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "trailingComma": "all",
  "arrowParens": "always",
  "endOfLine": "lf",
  "vueIndentScriptAndStyle": false
}
```

- `printWidth` 100 是前端社区的折中值；不要为了少换行设 160，diff 会变糊。
- `endOfLine: lf` 与仓库 `.gitattributes` 保持一致，避免 Windows 上整文件 diff。
- 忽略生成物：`.prettierignore` 写入 `dist`、`coverage`、`pnpm-lock.yaml`、`src/types/*.d.ts`。

## 4. Stylelint

```js
// .stylelintrc.cjs
module.exports = {
  extends: ['stylelint-config-standard-scss', 'stylelint-config-recommended-vue/scss'],
  rules: {
    'selector-class-pattern': [
      '^[a-z][a-z0-9]*(-[a-z0-9]+)*(__[a-z0-9]+(-[a-z0-9]+)*)?(--[a-z0-9]+(-[a-z0-9]+)*)?$',
      { message: '类名遵循 BEM：block__element--modifier' },
    ],
    'max-nesting-depth': [3, { ignore: ['pseudo-classes'] }],
    'declaration-no-important': true,
    'no-descending-specificity': null,
    'scss/dollar-variable-pattern': '^[a-z][a-z0-9-]*$',
  },
  ignoreFiles: ['dist/**', 'node_modules/**'],
}
```

- 穿透样式只允许 `:deep()`；`::v-deep` 与 `/deep/` 已废弃，由 `stylelint-config-recommended-vue` 直接报错。
- `declaration-no-important` 为 `true`：覆盖组件库统一在 `styles/element.scss` 用更高特异性选择器，而不是逐个 `!important`。
- `max-nesting-depth` 限 3：嵌套过深时改类名，不要靠缩进表达层级。

## 5. 编辑器统一

`.editorconfig`：

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

`.vscode/settings.json`（随仓库提交，保证团队一致）：

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.fixAll.stylelint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "eslint.validate": ["javascript", "typescript", "vue"],
  "stylelint.validate": ["css", "scss", "vue"]
}
```

`.vscode/extensions.json` 推荐 `Vue.volar`、`dbaeumer.vscode-eslint`、`esbenp.prettier-vscode`、`stylelint.vscode-stylelint`。不安装旧版 Vetur，它与 Volar 冲突会导致模板类型失效。

## 6. Git 提交规范

### 6.1 提交信息

采用 Conventional Commits：

```text
<type>(<scope>): <subject>
```

| type | 用途 |
|---|---|
| feat | 新功能 |
| fix | 修 bug |
| refactor | 重构（不改行为） |
| perf | 性能优化 |
| style | 格式（不影响逻辑） |
| docs | 文档 |
| test | 测试 |
| build | 构建与依赖 |
| chore | 杂项 |

```text
feat(order): 订单列表支持按状态批量筛选
fix(request): 修复 401 重复跳转登录页
perf(table): 订单列表改用虚拟滚动
```

- `subject` 用中文写清楚"改了什么"，不超过 50 字，不加句号。
- 一次提交只做一件事；格式化与业务改动分开，否则 review 无法定位。
- 破坏性变更在 body 写明 `BREAKING CHANGE:`。

### 6.2 husky 与 lint-staged

```json
// package.json
{
  "lint-staged": {
    "*.{ts,vue}": ["eslint --fix --max-warnings 0", "prettier --write"],
    "*.{scss,css}": ["stylelint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
pnpm lint-staged

# .husky/commit-msg
pnpm commitlint --edit "$1"
```

```js
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', ['feat', 'fix', 'refactor', 'perf', 'style', 'docs', 'test', 'build', 'chore', 'revert']],
    'subject-max-length': [2, 'always', 72],
    'subject-case': [0],
  },
}
```

- `pre-commit` 只检查**暂存区文件**，全量 lint 放 CI，否则改一个文件要等半分钟。
- 建议增加 `pre-push` 跑 `pnpm typecheck` 与 `pnpm test`，把类型错误拦在推送前。
- 禁止 `git commit --no-verify` 绕过 hook；紧急情况也应在事后补修，而不是留下未检查的提交。

## 7. 类型门禁

**Vite 构建不做类型检查**——它用 esbuild 直接转译，类型错误照样能构建成功。这是最常见的误判来源。因此：

- CI 与 `pre-push` 必须单独执行 `vue-tsc --noEmit`。
- `build` 脚本不要写成 `vue-tsc && vite build`（本地开发反馈慢），把类型检查拆成独立命令，由流水线串联。
- 新增文件的类型错误数不允许增加；历史遗留可先放行，但要在工单里记录基线。

## 8. CI 流水线

```yaml
# .gitlab-ci.yml（GitHub Actions 同理，替换 steps 即可）
stages:
  - install
  - check
  - build

default:
  image: node:20.11-alpine
  cache:
    key:
      files:
        - pnpm-lock.yaml
    paths:
      - .pnpm-store

install:
  stage: install
  script:
    - corepack enable
    - pnpm config set store-dir .pnpm-store
    - pnpm install --frozen-lockfile

lint:
  stage: check
  script:
    - pnpm lint
    - pnpm lint:style
    - pnpm format --check

typecheck:
  stage: check
  script:
    - pnpm typecheck

test:
  stage: check
  script:
    - pnpm test -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'

build:
  stage: build
  script:
    - pnpm build
  artifacts:
    paths:
      - dist
    expire_in: 7 days
```

原则：

- 顺序固定：安装 → 静态检查 → 类型 → 测试 → 构建。任一步失败即中断，不浪费后续资源。
- 缓存 pnpm store，不要缓存 `node_modules`（跨平台易坏）。
- 构建产物作为制品上传，部署阶段只取制品，不在部署机重新安装依赖。

## 9. 禁止事项

1. 提交 `node_modules`、`dist`、`coverage`、`.env.local`、`*.log`。
2. 在仓库里保留多份 lock 文件。
3. 用 `--no-verify` 跳过 hook。
4. 提交 `console.log` 与 `debugger`（ESLint 已拦，但别靠工具兜底）。
5. 用 `eslint-disable` 批量关闭规则而不修问题。
6. 在 CI 里用 `latest` 镜像标签，构建结果不可复现。
7. 把 `dist` 手工上传到服务器，绕过流水线。

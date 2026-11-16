# 样式与界面规范

## 1. 样式文件分层

```text
src/styles
├── variables.scss    Sass 变量：断点、mixin、z-index 层级
├── tokens.scss       CSS 变量：颜色、间距、圆角、字号、阴影
├── reset.scss        浏览器默认样式重置
├── global.scss       全局基础样式（html/body、滚动条、通用工具类）
└── element.scss      组件库样式覆盖
```

职责边界：

- `variables.scss` 只放**编译期**才能用的东西：`@media` 断点、`@mixin`、`$z-index-map`。
- `tokens.scss` 放**运行时**可切换的值：颜色、间距、圆角。定义为 `:root` 下的 CSS 变量，主题切换只需换一组变量。
- `element.scss` 是所有组件库覆盖的唯一入口。散落在业务组件里的 `.el-xxx` 覆盖会在升级时全部失效，且难以排查是谁改的。
- 业务组件不写全局样式，`<style>` 一律加 `scoped`。

`main.ts` 引入顺序（后引入的优先级更高）：

```ts
import '@/styles/reset.scss'
import '@/styles/tokens.scss'
import '@/styles/global.scss'
import '@/styles/element.scss'
```

## 2. scoped 与穿透

```vue
<style scoped lang="scss">
.order-list {
  :deep(.el-table__cell) {
    padding: 8px 0;
  }
}
</style>
```

规则：

- 默认 `scoped`；只有真正的全局样式（`reset`、`tokens`）不加。
- 穿透只用 `:deep()`。`::v-deep`（废弃）、`/deep/`（已移除）、`>>>`（不支持 scss）都禁止。
- `:deep()` 的宿主选择器必须是本组件的类名，不要写 `:deep(.el-button)` 这种全局打击面过大的选择器。
- 需要给子组件根节点加样式时，用 `:deep()` 或让子组件接受 `class`（Vue 3 会自动透传到根节点，`inheritAttrs` 为默认值即可）。
- `:slotted()` 用于插槽内容，`v-bind()` 用于把组件状态绑定到 CSS（`color: v-bind(textColor)`），后者慎用，会增加运行时开销。

## 3. 设计令牌

```scss
/* tokens.scss */
:root {
  --color-primary: #2563eb;
  --color-success: #16a34a;
  --color-warning: #d97706;
  --color-danger: #dc2626;

  --color-text-primary: #1f2329;
  --color-text-regular: #4e5969;
  --color-text-secondary: #86909c;
  --color-border: #e5e6eb;
  --color-bg-page: #f7f8fa;
  --color-bg-container: #ffffff;

  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  --font-size-xs: 12px;
  --font-size-sm: 13px;
  --font-size-base: 14px;
  --font-size-lg: 16px;
  --font-size-xl: 20px;

  --shadow-card: 0 1px 2px rgba(0, 0, 0, 0.04), 0 2px 8px rgba(0, 0, 0, 0.06);
}
```

```scss
/* variables.scss */
$breakpoints: (
  sm: 640px,
  md: 768px,
  lg: 1024px,
  xl: 1280px,
  2xl: 1536px,
);

@mixin respond-to($name) {
  @if map-has-key($breakpoints, $name) {
    @media (min-width: map-get($breakpoints, $name)) {
      @content;
    }
  } @else {
    @error "未知断点: #{$name}";
  }
}

$z-index: (
  dropdown: 1000,
  sticky: 1020,
  modal: 2000,
  message: 3000,
);
```

规则：

- 业务组件禁止写裸色值（`#fff`、`rgba(0,0,0,.5)`），必须用 `var(--color-xxx)`。渐进式统一：新代码强制，旧代码随改动顺手替换。
- 间距只从 `--spacing-*` 取值，不要在组件里散落 `13px`、`17px` 这类"看起来刚好"的数值。
- `z-index` 统一从 Sass map 取，禁止 `z-index: 99999` 的比赛式写法。
- 设计令牌与设计稿的命名保持同源（Figma 变量名 → CSS 变量名），避免"设计说主色、代码说 primary-blue"的翻译损耗。

## 4. 命名与选择器

采用 BEM，块名与组件名对应：

```scss
.order-list {
  &__header { /* element */ }
  &__item { /* element */ }
  &__item--active { /* modifier */ }
  &__item-label { /* 多词 element 用连字符 */ }
}
```

- 不使用"语义模糊的缩写"：`.ol-hd`、`.btn-l` 无法看出含义。
- 不使用 `#id` 选择器做样式。
- 不使用标签选择器限定（`div.order-list`），无必要且降低性能。
- 工具类（`mt-8`、`flex-center`）只允许在 `global.scss` 中定义有限集合，禁止引入完整原子化框架后又混写 BEM（两套体系并存会让覆盖关系不可预测）。

## 5. 嵌套与优先级

- 嵌套不超过 3 层。层级深的样式说明类名没起好，改类名比加缩进有效。
- 禁止用 `!important` 解决覆盖问题。真需要时（覆盖组件库内联样式）集中在 `element.scss` 并注释原因。
- 避免选择器权重竞争：不要用 `div .order-list .item span` 这种长链压过组件库，改用单一类名。
- `:hover` 与 `:focus-visible` 都要写。只写 `:hover` 时键盘用户看不到任何反馈。
- 动画只对 `transform` 与 `opacity` 做过渡；过渡 `width`/`height`/`top` 会触发重排，列表里尤其明显。

## 6. 单位与布局

| 场景 | 单位 | 说明 |
|---|---|---|
| 后台管理系统 | `px` | 与设计稿 1:1，避免换算误差 |
| 面向移动端的 H5 | `rem` 或 `vw` | 配合根字号缩放 |
| 字号 | `px`（后台）/ `rem`（H5） | 不混用两种单位做同一件事 |
| 行高 | 无单位数值（1.5） | 随字号自动缩放 |

布局规则：

- 优先 `flex` 与 `grid`，禁止用 `float` 做常规布局，禁止用绝对定位拼页面结构。
- 居中优先 `display: flex; align-items: center; justify-content: center`，不要 `line-height` 硬撑。
- 全屏高度用 `min-height: 100dvh`，`100vh` 在移动端浏览器地址栏收起时会溢出。
- 内容区滚动用 `overflow: auto` + `min-height: 0`（flex 子项默认 `min-height: auto`，不加会出现"撑破父容器"）。
- 表格列宽固定时用 `table-layout: fixed` + 明确宽度，避免内容多少导致列宽跳动。

## 7. 组件库定制

```scss
/* element.scss */
:root {
  --el-color-primary: var(--color-primary);
  --el-color-success: var(--color-success);
  --el-color-warning: var(--color-warning);
  --el-color-danger: var(--color-danger);
  --el-border-radius-base: var(--radius-sm);
  --el-font-size-base: var(--font-size-base);
}

.el-table {
  --el-table-border-color: var(--color-border);

  th.el-table__cell {
    background-color: var(--color-bg-page);
    font-weight: 500;
  }
}
```

规则：

- 优先覆盖 CSS 变量，其次覆盖类名，**绝不修改 `node_modules`**（升级即丢失，且无法 code review）。
- 覆盖集中在 `element.scss` 一个文件，按组件分块并加注释。
- 组件库升级前先跑一遍页面巡检：`el-table`、`el-form`、`el-dialog` 的 DOM 结构变化最频繁。
- 不用 `el-*` 类名写业务布局，业务布局用自定义类名 + 组件，避免组件库换版本时样式全崩。

## 8. 响应式

- 断点用 `respond-to` mixin，不在组件里写 `@media (max-width: 767px)` 的魔法数字。
- 后台系统以 `lg`（1024px）为最小支持宽度，低于该宽度给出提示或折叠侧栏，而不是硬挤。
- 移动端优先还是桌面优先：面向后台用桌面优先（`max-width` 断点），面向 C 端用移动优先（`min-width` 断点），同一项目二选一。
- 表格在窄屏下的处理顺序：隐藏次要列 → 横向滚动 → 卡片化。不要一上来就卡片化，信息密度损失大。
- 用容器查询（`@container`）替代部分媒体查询的场景：同一个组件在不同容器宽度下布局不同，此时媒体查询不够用。

## 9. 可访问性

基础要求（不做全套无障碍认证，但这几条必须满足）：

- 用语义化标签：`button` 而非 `div @click`，`nav`/`main`/`header` 划分结构，`label` 关联表单控件。
- 图片有 `alt`，装饰性图片 `alt=""`；图标按钮加 `aria-label`。
- 键盘可达：可点击元素必须是 `button`/`a`，或显式加 `tabindex="0"` + `@keydown.enter` / `@keydown.space`。
- 焦点样式不能删。`outline: none` 必须配自定义 `:focus-visible` 样式。
- 颜色不作为唯一信息载体：错误状态不能只靠红色，要配图标或文字。
- 弹窗打开时焦点进入弹窗，关闭后回到触发按钮（Element Plus 的 `el-dialog` 已处理，自定义弹窗需自己实现）。
- 对比度：正文文字与背景对比度 ≥ 4.5:1，大字号 ≥ 3:1。

## 10. 文案与国际化

- 文案不写死在模板里（有国际化需求时），统一放 `src/locales/zh-CN.ts`，模板用 `$t('order.status.waitPay')`。
- 不做字符串拼接：`'共 ' + total + ' 条'` 在英文语序下错乱，用带占位符的模板 `共 {total} 条`。
- 日期、金额、数字用 `Intl` 或 dayjs 的本地化格式，不手写格式串。
- 空态、错误、按钮文案要有主谓，不用"操作成功"这类无信息量的提示替代具体结果。
- 数字千分位、百分比、货币符号按 locale 处理，不硬编码符号。

## 11. 主题与暗色模式

```scss
html.dark {
  --color-text-primary: #e5e6eb;
  --color-bg-page: #17171a;
  --color-bg-container: #232324;
  --color-border: #3a3a3d;
}
```

- 主题切换只改 `html` 上的类名，组件内不写 `if (isDark)` 的分支样式。
- 新增颜色时同时补暗色值，否则暗色模式下会出现白底白字。
- 图片、插画、阴影在暗色下需要单独处理（阴影在深色背景上几乎不可见，改用边框或提亮背景）。
- 主题偏好持久化到 `localStorage`，首屏加载时在 `index.html` 内联脚本中读取并加类名，避免闪白。

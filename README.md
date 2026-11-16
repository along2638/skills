# Skills

AI 编程助手的技能仓库，当前包含两个开发规范技能：

- **`java-backend-standards`**（Java 后端开发规范）
- **`frontend-standards`**（前端开发规范，Vue 3 + TypeScript）

技能把团队的开发约定写成 AI 可执行的规则：新建项目、新增功能、审查代码、回答规范问题时，AI 直接按这里的规定输出，而不是凭通用经验猜测。

## 仓库结构

```text
skills/
├── README.md
├── .gitattributes                          # 统一 LF 换行
├── .gitignore                              # 忽略本地运行时数据与打包产物
├── java-backend-standards/
│   ├── SKILL.md                            # 技能入口：适用范围、硬规则、工作流程
│   └── references/
│       ├── architecture.md                 # 分包分层、模块依赖、对象转换
│       ├── tech-stack.md                   # 版本矩阵、pom、application.yml、配置类
│       ├── database.md                     # 建表、字段、索引、SQL、逻辑删除、迁移
│       ├── java-coding.md                  # 命名、Lombok、注入、Service/Mapper 写法
│       ├── api-contract.md                 # URL、统一响应、错误码、分页、幂等
│       ├── exception-logging.md            # 异常体系、全局处理、日志、脱敏
│       ├── security.md                     # 认证鉴权、密码、注入、上传、CORS
│       ├── testing.md                      # 单元测试、集成测试、覆盖率
│       ├── review-checklist.md             # 提交前自检与代码评审清单
│       ├── example-feature.md              # 订单功能完整垂直切片（可照抄）
│       └── alibaba-manual-map.md           # 与《Java 开发手册》嵩山版对照
└── frontend-standards/
    ├── SKILL.md                            # 技能入口：适用范围、硬规则、工作流程
    └── references/
        ├── architecture.md                 # 目录分层、路由与权限、布局、模块边界
        ├── tech-stack.md                   # 版本矩阵、依赖治理、vite 配置、环境变量
        ├── vue-coding.md                   # 组件写法、props/emits、响应式、生命周期
        ├── typescript.md                   # 类型设计、泛型、类型收窄、禁用 any
        ├── state-and-data.md               # Pinia、请求层、错误处理、三态、表单
        ├── api-contract.md                 # 接口契约承接侧：路径、分页、错误码、幂等
        ├── ui-and-styles.md                # 样式分层、设计令牌、组件库定制、可访问性
        ├── quality-tooling.md              # ESLint/Prettier/Stylelint、husky、CI
        ├── testing.md                      # 单元测试、组件测试、MSW、Playwright
        ├── performance.md                  # 首屏、分包、长列表、渲染优化、监控
        ├── security.md                     # XSS、token 存储、权限边界、依赖安全
        ├── review-checklist.md             # 提交前自检与代码评审清单
        └── example-feature.md              # 订单列表页完整垂直切片（可照抄）
```

两个技能采用同一套分层：`SKILL.md` 只放**必须常驻上下文**的内容（适用范围、硬规则、决策速查、工作流程），细节全部沉到 `references/`，由 AI 按任务类型按需读取，避免每次对话都吞掉几千行。

## 技能一览

### java-backend-standards

| 项 | 规定 |
|---|---|
| JDK | 1.8 语法基线（JDK 11 项目同样禁用 17+ 特性） |
| 框架 | Spring Boot 2.7.18 / Spring Framework 5.3.x，`javax.*` 命名空间 |
| 持久层 | MyBatis-Plus 3.5.7 |
| 数据库 | MySQL 8.0，InnoDB，`utf8mb4` |
| 连接池 | Druid 1.2.28 |
| 构建 | Maven 3.6+，单模块优先 |
| 工具 | Hutool 5.8.48 + Lombok，Knife4j 4.5.0（可选） |

关键约定：统一响应 `Result<T>`（成功 `code = 0`）、`BizException` + `ErrorCode` + 全局处理器、主键 `BIGINT`、逻辑删除 `deleted` + `@TableLogic`、时间 `DATETIME(3)` ↔ `LocalDateTime`、金额 `DECIMAL(18,2)` ↔ `BigDecimal`。

> 若项目是 Spring Boot 3.x / JDK 17+（`jakarta.*`），不要直接照抄版本与 import，只借用分层、数据库、API 设计思想，按其实际版本重写。

### frontend-standards

| 项 | 规定 |
|---|---|
| 运行时 | Node.js 20 LTS，包管理 pnpm 9+（只留 `pnpm-lock.yaml`） |
| 框架 | Vue 3.4.x，Composition API + `<script setup lang="ts">` |
| 语言 | TypeScript 5.4，`strict` 全开 |
| 构建 | Vite 5.2 |
| 路由 / 状态 | Vue Router 4.3 / Pinia 2.1（禁 Vuex） |
| UI | Element Plus 2.6（单一组件库，禁混用） |
| 请求 / 日期 | Axios 1.7 / dayjs |
| 质量 | ESLint 9 flat config + Prettier 3 + Stylelint 16 + husky + commitlint |
| 测试 | Vitest 1.6 + @vue/test-utils 2.4 + Playwright 1.45 |

关键约定：`request<T>` 泛型贯穿接口层、`BizError` 统一错误、`loading/error/empty` 三态必做、路由懒加载、`vue-tsc --noEmit` 作为类型门禁（Vite 构建不检查类型）。

> 若项目是 Vue 2 + Options API + webpack，不要套用组合式 API、Pinia、Vite 配置，只借用分层与工程化思想。

## 前后端契约对齐

两个技能不是独立的两套文档，接口层必须能对上。契约的权威定义在后端 `java-backend-standards/references/api-contract.md`，前端在 `frontend-standards/references/api-contract.md` 承接，两边各有一张同源的**对齐表**。

| 契约项 | 后端定义 | 前端承接 |
|---|---|---|
| 路径 | `/api/v1/{复数资源}`，动作用子路径 | `baseURL=/api/v1`，接口文件只写 `/orders` |
| 响应包 | `code` / `message` / `data` / `traceId` | 拦截器解包 `data`，只判 `code === 0` |
| 分页响应 | `records` / `total` / `pageNum` / `pageSize` / `pages` | 同名取值，**不用 `list`** |
| 分页请求 | `pageNum` / `pageSize` / `sortBy` / `sortOrder` | 同参数透传，`pageSize ≤ 100` |
| ID / 金额 | 均以字符串返回 | 不转数字、不做金额运算 |
| 时间 | `yyyy-MM-dd HH:mm:ss`，东八区 | 显式指定格式解析 |
| 枚举 | code 字符串，可新增值 | 未知值必须有兜底展示 |
| 校验失败 | 400 + `10001` + `[{field,message}]` | 按 `field` 回填表单 |
| 认证失效 | 401 + `20001` | 清 token 跳登录，防重复跳转 |
| 幂等 | 请求头 `Idempotency-Key` | 创建/支付类请求携带并复用 |
| 下载 | 裸流 + `Content-Disposition` | `responseType: 'blob'` |

变更规则：**破坏性变更（改字段名、改类型、删字段）必须升 `/api/v2`**；新增字段与枚举值属兼容变更，前端需有兜底。改契约时同步修改四处：后端契约文件、前端契约文件、OpenAPI 注解、示例数据。

## 安装

技能目录名必须与 `SKILL.md` frontmatter 中的 `name` 一致（分别为 `java-backend-standards`、`frontend-standards`）。

主流 AI 编程助手都通过一个「技能目录」加载本地技能，目录位置因工具而异，约定一致。下文用 `$SKILLS_DIR` 指代该目录，替换为你实际使用的路径即可。

**方式一：克隆后复制（推荐）**

```bash
git clone git@gitee.com:along2638/skills.git
cp -r skills/java-backend-standards "$SKILLS_DIR/"
cp -r skills/frontend-standards "$SKILLS_DIR/"
```

**方式二：克隆到固定目录后软链接**

```bash
git clone git@gitee.com:along2638/skills.git ~/skills-repo
ln -s ~/skills-repo/java-backend-standards "$SKILLS_DIR/java-backend-standards"
ln -s ~/skills-repo/frontend-standards "$SKILLS_DIR/frontend-standards"
```

用软链接的好处是 `git pull` 即更新，不用重复复制。

**方式三：项目级共享**

放到项目内的技能目录（随代码库一起提交），团队成员共享同一份规范。前端项目只放 `frontend-standards`，后端项目只放 `java-backend-standards`，全栈项目可都放。

| 层级 | 放置位置 | 作用范围 |
|---|---|---|
| 用户级 | `$SKILLS_DIR`，通常在家目录下的工具配置目录中 | 本机所有项目可用 |
| 项目级 | 项目根目录下的同名技能目录 | 随项目走，团队共享 |

## 使用

安装后在对话里直接描述任务即可，技能会按场景自动加载。

**后端**

| 场景 | 示例说法 |
|---|---|
| 新建项目 | 「帮我建一个 Spring Boot 后端项目，做订单管理」 |
| 新增功能 | 「给订单加一个退款接口」 |
| 代码审查 | 「按团队规范 review 这段 Service」 |
| 表结构设计 | 「设计订单表，要支持按用户和状态分页查询」 |
| 规范问答 | 「金额为什么不能用 double？」「分页最大多少条？」 |

**前端**

| 场景 | 示例说法 |
|---|---|
| 新建项目 | 「建一个 Vue 3 + TS 后台项目，带登录和侧边栏布局」 |
| 新增页面 | 「做一个订单列表页，支持按状态筛选和分页」 |
| 组件封装 | 「封装一个带防重复提交的确认按钮」 |
| 代码审查 | 「按团队规范 review 这个页面组件」 |
| 规范问答 | 「为什么不能用 index 当 key？」「类型检查为什么不能只看 build？」 |

两个技能内部都有四种工作流程（新建项目 / 新增垂直切片 / 代码审查 / 规范问答），输出时给出完整可编译文件、同步补齐配置与测试，并按各自的 `review-checklist.md` 自检。

## 规约来源

### java-backend-standards

编码规约来自《Java 开发手册》嵩山版（v1.7.0，7 章）的逐条整合，不是另写摘要——同一技能内不存在两套规则。

其中 **10 处有意偏差** 保留本仓库写法，理由写在 `references/alibaba-manual-map.md`：

1. `is_deleted` → `deleted`（手册自身又禁止 POJO 布尔属性带 `is` 前缀，前后矛盾）
2. 金额「分」整型 → `DECIMAL(18,2)`
3. URL 下划线 → kebab-case
4. 版本号放 header → 路径 `/api/v1`
5. 五位 A/B/C 错误码 → 数字区间码（附双向映射表）
6. DO/BO/VO/AO → `Entity` / `Req` / `Resp`
7. 主键索引 `pk_` 前缀 → MySQL 固定 `PRIMARY`（命名了也做不到）
8. 全字段 unsigned → 仅非主键计数列
9. 时间 GMT → `Asia/Shanghai`
10. 核心覆盖率 100% → 目标 100%、准入 80%

不适用条目（Velocity 模板、iBATIS `queryForList` / `<isEqual>` / `resultClass`）已在对照文件第 4 节明确放弃，避免照抄过时 API。

### frontend-standards

以前端官方规范为骨架，按后台系统的实际工程场景取舍：

- **Vue 官方风格指南**：组件命名多单词、props 细化、`v-for` key、`v-if`/`v-for` 不共用等规则直接采纳；不采纳「组件文件必须首字母大写」之外的过时约定。
- **TypeScript 官方手册**：`strict` 全开、禁用 `any`、类型收窄优先于断言。
- **Pinia / Vue Router / Vite 官方文档**：setup store、路由懒加载、构建分包。
- **Element Plus 文档**：组件库定制通过 CSS 变量而非改源码。

在官方规范之外，本仓库额外明确了官方没有覆盖但团队必须统一的部分：请求层封装与错误分层、三态处理、URL 作为筛选条件唯一来源、竞态与请求取消、类型门禁与 CI 顺序、覆盖率口径。

## 维护约定

**改哪一层**

- 新增一条「必须遵守」的规则 → 对应技能的 `SKILL.md`「不可违反的硬规则」，并同步 `review-checklist.md`
- 补充某类实现细节 → 对应技能 `references/*.md`
- 后端规则与阿里手册冲突 → 同时更新 `alibaba-manual-map.md` 的偏差表与理由
- 前端规则与官方风格指南冲突 → 在 `review-checklist.md` 或对应 `references` 文件中写明理由

**提交前**

```bash
git add -A
git commit -m "docs(database): 补充大表分页的稳定排序要求"
git push
```

提交信息用 `docs(scope): 说明`。scope 取值：

- 后端：`skill` / `arch` / `stack` / `db` / `coding` / `api` / `log` / `security` / `test` / `review`
- 前端：`fe` / `vue` / `ts` / `state` / `ui` / `tooling` / `perf` / `security` / `test` / `review`

**不发散**

- 不引入基线外组件（缓存、MQ、分库分表、微前端），除非基线正式升级
- 新增依赖必须先进对应 `references/tech-stack.md` 的版本矩阵
- 规则要可判定：写「方法 ≤ 80 行」「单文件 ≤ 400 行」，而不是「不要太长」

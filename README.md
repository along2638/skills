# Skills

AI 编程助手的技能仓库。当前包含一个技能：**`java-backend-standards`（Java 后端开发规范）**。

技能把团队的后端开发约定写成 AI 可执行的规则：新建项目、新增接口、审查代码、回答规范问题时，AI 直接按这里的规定输出，而不是凭通用经验猜测。

## 仓库结构

```text
skills/
├── README.md
├── .gitattributes                     # 统一 LF 换行
├── .gitignore                         # 忽略本地运行时数据与打包产物
└── java-backend-standards/
    ├── SKILL.md                       # 技能入口：适用范围、硬规则、工作流程
    └── references/
        ├── architecture.md            # 分包分层、模块依赖、对象转换
        ├── tech-stack.md              # 版本矩阵、pom、application.yml、配置类
        ├── database.md                # 建表、字段、索引、SQL、逻辑删除、迁移
        ├── java-coding.md             # 命名、Lombok、注入、Service/Mapper 写法
        ├── api-contract.md            # URL、统一响应、错误码、分页、幂等
        ├── exception-logging.md       # 异常体系、全局处理、日志、脱敏
        ├── security.md                # 认证鉴权、密码、注入、上传、CORS
        ├── testing.md                 # 单元测试、集成测试、覆盖率
        ├── review-checklist.md        # 提交前自检与代码评审清单
        ├── example-feature.md         # 订单功能完整垂直切片（可照抄）
        └── alibaba-manual-map.md      # 与《Java 开发手册》嵩山版对照
```

分层设计：`SKILL.md` 只放**必须常驻上下文**的内容（硬规则、决策速查、工作流程），细节全部沉到 `references/`，由 AI 按任务类型按需读取，避免每次对话都吞掉几千行。

## 技术基线

规范不是通用的，它明确绑定一套基线，换栈时需要先告知 AI 基线不匹配。

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

## 安装

技能目录名必须与 `SKILL.md` frontmatter 中的 `name` 一致（当前为 `java-backend-standards`）。

主流 AI 编程助手都通过一个「技能目录」加载本地技能，目录位置因工具而异，约定一致。下文用 `$SKILLS_DIR` 指代该目录，替换为你实际使用的路径即可。

**方式一：克隆后复制（推荐）**

```bash
git clone git@gitee.com:along2638/skills.git
cp -r skills/java-backend-standards "$SKILLS_DIR/"
```

**方式二：克隆到固定目录后软链接**

```bash
git clone git@gitee.com:along2638/skills.git ~/skills-repo
ln -s ~/skills-repo/java-backend-standards "$SKILLS_DIR/java-backend-standards"
```

用软链接的好处是 `git pull` 即更新，不用重复复制。

**方式三：项目级共享**

放到项目内的技能目录（随代码库一起提交），团队成员共享同一份规范。

| 层级 | 放置位置 | 作用范围 |
|---|---|---|
| 用户级 | `$SKILLS_DIR`，通常在家目录下的工具配置目录中 | 本机所有项目可用 |
| 项目级 | 项目根目录下的同名技能目录 | 随项目走，团队共享 |

## 使用

安装后在对话里直接描述任务即可，技能会按场景自动加载：

| 场景 | 示例说法 |
|---|---|
| 新建项目 | 「帮我建一个 Spring Boot 后端项目，做订单管理」 |
| 新增功能 | 「给订单加一个退款接口」 |
| 代码审查 | 「按团队规范 review 这段 Service」 |
| 表结构设计 | 「设计订单表，要支持按用户和状态分页查询」 |
| 规范问答 | 「金额为什么不能用 double？」「分页最大多少条？」 |

技能内部有四种工作流程（新建项目 / 新增垂直切片 / 代码审查 / 规范问答），输出时会给出完整可编译文件、同步补齐迁移 SQL 与单元测试，并按 `review-checklist.md` 自检。

## 规约来源

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

## 维护约定

**改哪一层**

- 新增一条「必须遵守」的规则 → `SKILL.md` 的「不可违反的硬规则」，并同步 `review-checklist.md`
- 补充某类实现细节 → 对应的 `references/*.md`
- 规则与阿里手册冲突 → 同时更新 `alibaba-manual-map.md` 的偏差表与理由

**提交前**

```bash
git add -A
git commit -m "docs(database): 补充大表分页的稳定排序要求"
git push
```

提交信息用 `docs(scope): 说明`，scope 取 `skill` / `arch` / `stack` / `db` / `coding` / `api` / `log` / `security` / `test` / `review`。

**不发散**

- 不引入缓存、MQ、分库分表等基线外组件，除非基线正式升级
- 新增依赖必须先进 `references/tech-stack.md` 的版本矩阵
- 规则要可判定：写「方法 ≤ 80 行」而不是「方法不要太长」

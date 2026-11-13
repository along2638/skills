---
name: java-backend-standards
description: Java 后端开发规范（Spring Boot 2.7 + JDK 8/11 + MyBatis-Plus + MySQL 8.0）。用于新建或改造 Java 后端项目的包结构与分层、pom 依赖与版本选型、MyBatis-Plus 持久层写法、MySQL 表结构/索引/SQL 设计、REST API 与统一响应、异常与日志、事务、参数校验、安全与测试；也用于按规范审查代码、排查不符合项或回答团队规范问题。
---

# Java 后端开发规范

## 适用范围（先确认）

- 本规范基线为 **Spring Boot 2.7 + JDK 8/11 + MyBatis-Plus + MySQL 8.0**，使用 `javax.*` 命名空间。
- 若项目是 Spring Boot 3.x / JDK 17+（`jakarta.*`），不要直接套用版本与 import；先告知用户基线不匹配，只借用分层、数据库、API 设计思想，并按其实际版本重写。
- 生成代码必须能按 **JDK 8 语法**编译。即使项目使用 JDK 11，也不使用 9+ API，除非用户确认项目已统一升级。
- 编码规约来源为《Java 开发手册》嵩山版（v1.7.0），已按本基线落地；与手册的 10 处有意偏差见 `references/alibaba-manual-map.md`，冲突时以本规范为准。

## 基线（不可协商）

| 项 | 规定 |
|---|---|
| JDK | 1.8 语法基线；JDK 11 项目同样禁用 17+ 特性 |
| 框架 | Spring Boot 2.7.18，Spring Framework 5.3.x |
| 命名空间 | `javax.validation` / `javax.servlet` / `javax.annotation` |
| 持久层 | MyBatis-Plus 3.5.7（`mybatis-plus-boot-starter`） |
| 数据库 | MySQL 8.0，InnoDB，`utf8mb4` |
| 连接池 | Druid 1.2.28（`druid-spring-boot-starter`） |
| 构建 | Maven 3.6+，单模块优先 |
| 转换 | Hutool 5.8.48（`BeanUtil`）+ Lombok |
| 文档 | Knife4j 4.5.0（可选） |
| 统一响应 | `Result<T>`，成功 `code = 0` |
| 分页 | `PageReq` / `PageResp<T>`，`pageSize ≤ 100` |
| 异常 | `BizException` + `ErrorCode` + 全局处理器 |
| 主键 | `BIGINT`，单库 `IdType.AUTO`，分布式 `ASSIGN_ID` |
| 逻辑删除 | `deleted TINYINT` + `@TableLogic` |
| 时间 | `DATETIME(3)` ↔ `LocalDateTime`，Asia/Shanghai |
| 金额 | `DECIMAL(18,2)` ↔ `BigDecimal` |

## 代码结构骨架

```text
com.acme.mall
├── MallApplication.java
├── common
│   ├── api          Result / PageReq / PageResp / ErrorCode
│   ├── config       MybatisPlus / Jackson / WebMvc / Redis / OpenAPI
│   ├── constant     全局常量
│   ├── enums        通用枚举
│   ├── exception    BizException / GlobalExceptionHandler
│   ├── util         无状态工具类
│   └── web          TraceIdFilter / 登录上下文
├── controller       仅参数接收、校验、调用 Service
├── service          XxxService 接口 + impl/XxxServiceImpl 实现（强制）
├── manager          跨域编排、外部系统调用（可选）
├── mapper           BaseMapper + 复杂 SQL 的 XML
├── entity           与表一一对应
├── enums            业务状态枚举
├── dto              入参：XxxCreateReq / XxxUpdateReq / XxxQueryReq
├── vo               出参：XxxResp
└── convert          Hutool BeanUtil 转换器
```

根包使用产品/系统名（如 `com.acme.mall`）。只有一个业务域时，业务包直接放在根包下，不要再套 `modules/{domain}`；出现第二个业务域时，才把各域迁入 `modules/{domain}`。禁止把根包命名为 `order` 这类业务域。

## 工作流程

### 新建项目 / 新增业务域

1. 阅读 `references/architecture.md`、`references/tech-stack.md`、`references/database.md`。
2. 按顺序输出：目录树 → `pom.xml` → `application.yml` → DDL 迁移脚本 → 配置类 → 一个示例接口切片。
3. 所有文件给完整内容，不给省略号、不写 `TODO`；import 必须齐全。
4. 根包名跟随用户项目，使用产品/系统名；用户未指定时使用 `com.example.{project}`，不要用 `com.example.order` 这类业务域名作根包，也不要硬编码 `com.acme`。

### 新增一个功能（垂直切片）

1. 定位业务域与现有命名，阅读 `references/architecture.md`、`references/java-coding.md`、`references/api-contract.md`、`references/exception-logging.md`。
2. 实现顺序：DDL/迁移 SQL → Entity → Mapper（复杂查询加 XML）→ Req/Resp → Hutool 转换器 → Service 接口（`service`）→ 实现（`service/impl`）→ Controller。
3. 同步补齐：参数校验、错误码、事务、日志、单元测试。
4. 输出后逐条对照 `references/review-checklist.md` 自检。

### 代码审查

1. 以 `references/review-checklist.md` 为主，按需读对应专题文件。
2. 输出按严重级别分组：**阻断**（违反硬规则/有缺陷）、**建议**（可读性/优化）。
3. 每条给出 `文件:行号`、违反的规则、最小修复方案；不要无故重写整个文件。

### 规范问答

- 先给结论，再引用本 skill 的具体规则与参考文件。
- 发现项目现状与规范冲突时，明确说明偏差、风险与迁移成本，不默认项目已合规。

## 不可违反的硬规则

**结构**

1. 根包用产品/系统名；单业务域时业务包直接在根包下，出现第二个业务域才引入 `modules/{domain}`；`common` 不得出现业务逻辑。
2. 依赖方向 `controller → service → mapper`；禁止 Controller 调 Mapper、跨域直连 Mapper、`Entity` 作为接口出入参。
3. Controller 保持薄；事务只放在 Service；业务 Service 必须接口 + `service/impl` 实现，Controller 只依赖接口；外部 HTTP/RPC、MQ 发送不得放在事务内。

**技术栈**

4. 不引入基线外依赖、`SNAPSHOT`、动态版本；不混用 `javax`/`jakarta`；禁用 Fastjson 1.x、Springfox、JPA。
5. 构造器注入 + `private final`，不用字段 `@Autowired`；禁止 `System.out`、`printStackTrace()`。

**编码**

6. 金额只用 `BigDecimal`（禁止 `double`/`float`，禁止 `new BigDecimal(double)`），比较用 `compareTo`；包装类型一律 `equals` 比较，`equals` 常量在前。
7. `Collectors.toMap` 必须传 `mergeFunction`；`Arrays.asList`、`Collections.emptyList()` 不可增删；`foreach` 内禁止增删元素。
8. 线程池禁止用 `Executors` 创建；`ThreadLocal` 必须 `finally` 中 `remove()`；`lock()` 必须在 `try` 之外；多资源加锁顺序全局一致。
9. 日期格式年份用小写 `yyyy`；禁用 `java.sql.Date`/`Time`/`Timestamp`；不硬编码 365 天。
10. 类、对外方法（Service 接口、Controller、工具类公有方法）、抽象方法、枚举常量必须有 Javadoc；类注释写职责，并含 `@author` 与 `@date`（作者用实际提交人，不编造）。

**数据库**

11. 表名、字段名 `snake_case`；主键 `BIGINT`；金额 `DECIMAL`；时间 `DATETIME(3)`；布尔 `TINYINT`。
12. 禁用 `FLOAT`/`DOUBLE` 存金额、MySQL `ENUM`/`SET`、物理外键、触发器、存储过程。
13. 每张业务表必须具备审计字段、`deleted`、`version`；逻辑删除统一 `@TableLogic`。
14. 查询禁止 `SELECT *`；SQL 禁止 `${}`；分页 `pageSize ≤ 100`；大表分页必须带稳定排序。
15. 多表 SQL 所有列加表别名；`SUM` 用 `IFNULL` 包裹；`NULL` 判断用 `IS NULL`；`count` 为 0 直接返回。

**API**

16. URL 形如 `/api/v1/{resources}`；统一返回 `Result<T>`，业务失败 `code != 0`，HTTP 状态码语义正确。
17. 入参必须 JSR-303 校验并加 `@Validated`；出参不得直接暴露 Entity；ID 字段按需序列化为字符串防精度丢失。
18. 列表为空返回 `[]`、对象为空返回 `{}`；超过 2^53 的整数用 `String` 传输；URL 参数 ≤ 2048 字节；错误响应含 HTTP 状态码 + 错误码 + `traceId` + 用户可读提示。

**异常与日志**

19. 业务异常只抛 `BizException(ErrorCode)`；由全局处理器兜底；响应中不得出现堆栈、SQL、类名。
20. 日志用 SLF4J 占位符；禁止记录密码、token、密钥、完整手机号/身份证；关键写操作记录业务 ID。
21. 日志至少保留 15 天，敏感操作日志 ≥ 6 个月并多机备份；`additivity="false"`；trace/debug 前判断级别。

**安全与测试**

22. SQL 参数一律 `#{}`；密码 BCrypt；密钥走环境变量；CORS 白名单；上传校验类型、大小、存储路径。
23. 新增/修改 Service 方法必须有单元测试；DAO 变更必须有集成测试；测试不得连接生产库。
24. 敏感数据展示脱敏；短信、邮件、支付等能力必须防重放；所有入参都要校验长度、范围、格式。

## Service 约定（强制）

- 业务 Service 一律接口 + 实现：接口放 `service/XxxService.java`，实现放 `service/impl/XxxServiceImpl.java`；多业务域时前缀 `modules/{domain}/`。
- Controller、其他 Bean 只依赖 `XxxService` 接口，禁止注入或引用 `XxxServiceImpl`。
- `@Service`、`@Transactional` 只标在实现类；接口不标注解，也不声明 Entity。
- 接口方法只表达业务语义，参数/返回值使用 DTO/VO；Entity 只在实现类与 Mapper 之间传递。
- 单元测试直接测 `XxxServiceImpl`；Controller 切片测试用 `@MockBean` mock `XxxService` 接口。
- 本文的 `ServiceImpl` 指业务实现类，不是 MyBatis-Plus 的 `ServiceImpl`；简单 CRUD 用 `BaseMapper`，不要用 MP `IService` 代替业务接口。
- `common` 下的技术组件（如 `JwtTokenService`）不受此条约束，可以是具体类。

## 关键决策速查

| 场景 | 规定 |
|---|---|
| 单库主键 | `IdType.AUTO`，DDL `AUTO_INCREMENT` |
| 分库/多写主键 | `IdType.ASSIGN_ID`（雪花），DDL `BIGINT`，同表不得混用 |
| 对外 ID | `Long` 加 `@JsonSerialize(using = ToStringSerializer.class)` |
| 删除后重建同唯一键 | 用 `deleted_at DATETIME(3) NULL` + `@TableLogic(value="null", delval="now()")`，唯一索引包含 `deleted_at` |
| 枚举/状态入库 | 统一 `VARCHAR(32)` 存 code，Java `enum` 实现 `IEnum<String>` + `@JsonValue`；列注释列全 code |
| 布尔字段 | `TINYINT` 0/1，Java `Boolean` |
| 金额 | `BigDecimal`，`setScale(2, RoundingMode.HALF_UP)`，用 `compareTo` 比较 |
| 时间 | `LocalDateTime` / `LocalDate`，统一 `Asia/Shanghai` |
| 分页插件 | `PaginationInnerInterceptor(DbType.MYSQL)`，`maxLimit = 500` |
| 连接池 | Druid 1.2.28；`spring.datasource.type=com.alibaba.druid.pool.DruidDataSource`，生产关闭监控页 |
| 根包命名 | 产品/系统名（如 `com.acme.mall`）；单域扁平分包，多域才用 `modules/{domain}`；禁止用 `com.acme.order` 作根包 |
| Service 结构 | 每个业务 Service 固定 `XxxService` 接口 + `service/impl/XxxServiceImpl`；Controller 只依赖接口 |
| 对象转换 | Hutool `BeanUtil`；Convert 为 `@Component` 具体类，入参转 Entity 时忽略敏感字段 |
| 对象术语 | PO=`Entity`，DTO=`Req`/内部 DTO，VO=`Resp`；DAO 由 Mapper 承担；不建空 DO/BO |
| 多租户 | 需要时全表 `tenant_id` + 拦截器，禁止部分表遗漏 |
| 数据变更 | 统一版本化 SQL 入库，禁止手工改生产表结构 |
| 相等性比较 | 包装类用 `equals`、`BigDecimal` 用 `compareTo`、浮点用误差范围；禁止 `==` 判等 |
| Map 收集 | `Collectors.toMap(k, v, (a, b) -> b)`；value 为 `null` 先过滤，否则 NPE |
| 线程池 | `ThreadPoolTaskExecutor` + 有界队列 + 线程名前缀 + `CallerRunsPolicy`；禁用 `Executors` |
| 加锁顺序 | 多资源固定加锁顺序；`lock()` 在 `try` 外；`tryLock()` 返回 true 才进业务块 |
| 当前时间 | `System.currentTimeMillis()` / `Instant.now()`；弃用 `new Date().getTime()` |
| 日志保留 | 全量 ≥ 15 天；敏感操作 ≥ 6 个月 + 多机备份；`additivity="false"` |
| 列表空值 | 对前端返回 `[]`/`{}`；Service 内部方法可返回 `null` 但需 Javadoc 说明 |
| 超大整数 | 订单号、流水号等用 `String` 传输，超过 2^53 必丢精度 |
| 错误响应 | HTTP 状态码 + `code` + 用户可读 `message` + `traceId`；技术原因只进日志 |
| 二方库版本 | `主.次.修订`，起始 `1.0.0`；线上禁 `SNAPSHOT`；返回值不用枚举类型 |
| JVM 参数 | `-Xms` 与 `-Xmx` 相同；必开 `-XX:+HeapDumpOnOutOfMemoryError` |

## 输出要求

- 使用中文说明，代码标识符、表名、字段名用英文。
- 生成代码时给出完整文件路径与可编译内容；改动表结构必须同时给迁移 SQL。
- 新增依赖必须说明版本、用途，并确认在 `references/tech-stack.md` 的版本矩阵内。
- 用户项目现状未知时，先读 `pom.xml` 与相邻代码再动手，不凭空假设包名或基类。
- 不要主动引入缓存、消息队列、分库分表等未要求的组件。

## 参考文件

| 文件 | 何时阅读 |
|---|---|
| `references/architecture.md` | 建项目、分包、分层、模块依赖、对象转换 |
| `references/tech-stack.md` | 选型、版本矩阵、pom、application.yml、配置类 |
| `references/database.md` | 建表、字段类型、索引、SQL、逻辑删除、迁移 |
| `references/java-coding.md` | 命名、Lombok、注入、Service/Mapper 写法、JDK 8 API 限制 |
| `references/api-contract.md` | URL、统一响应、错误码、分页、校验、幂等、版本 |
| `references/exception-logging.md` | 异常体系、全局处理、日志级别、traceId、脱敏 |
| `references/security.md` | 认证鉴权、密码、注入、上传、CORS、密钥管理 |
| `references/testing.md` | 单元测试、集成测试、Mock、测试数据与覆盖要求 |
| `references/review-checklist.md` | 提交前自检与代码评审 |
| `references/example-feature.md` | 需要完整可照抄的订单功能垂直切片 |
| `references/alibaba-manual-map.md` | 需要确认规约出处、章节归属，或判断某条与《Java 开发手册》嵩山版是否一致 |

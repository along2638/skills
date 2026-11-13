# 代码评审清单

## 使用方式

1. 先看变更范围与影响面，再按"结构 → API → 数据库 → 安全 → 异常日志 → 测试 → 性能/可读性"顺序检查。
2. 每条问题给出：`文件:行号`、违反的规则、风险、最小修复方案。
3. 按严重级别输出，不把风格偏好写成阻断项。
4. 不确定项目约定时先看相邻代码与 `pom.xml`，不凭规范臆断。
5. 没发现问题时明确写"未发现阻断项"，并列出残余风险。

| 级别 | 含义 |
|---|---|
| 阻断 | 功能错误、数据错误、安全漏洞、违反硬规则、无法编译/测试 |
| 建议 | 可读性、可维护性、性能优化，不修也能上线 |
| 提示 | 风格统一、文档补充、可选优化 |

## A. 阻断项

- [ ] Spring Boot 2.7 项目中混入 `jakarta.*` 或 SB3 API。
- [ ] JDK 8 基线下使用 `var`、`List.of`、`Stream.toList`、文本块等 9+ 特性。
- [ ] Controller 直接调用 Mapper，或跨域直接引用其他域 Entity/Mapper。
- [ ] 业务 Service 未按 `XxxService` 接口 + `service/impl/XxxServiceImpl` 实现拆分，或 Controller 直接注入实现类。
- [ ] Entity 作为接口入参或出参，泄露内部字段。
- [ ] `@Transactional` 出现在 Controller，或同类自调用导致事务失效。
- [ ] `SELECT *`、`${}` 拼接、无 `WHERE` 的 `UPDATE/DELETE`。
- [ ] 业务唯一键没有唯一索引，仅靠代码判重。
- [ ] 逻辑删除表缺少 `deleted`，或 XML 手写 SQL 未过滤已删除数据。
- [ ] 请求未做 JSR-303 校验，或校验异常返回 500。
- [ ] 响应包含堆栈、SQL、类名、内部地址，或 500 返回 HTML。
- [ ] 日志/代码/配置中出现密码、token、密钥、完整身份证/银行卡。
- [ ] 单条资源查询/修改缺少归属或租户校验（水平越权）。
- [ ] 创建/支付/退款类接口没有幂等方案。
- [ ] 新增 Service 逻辑没有单元测试，或 SQL 变更没有集成测试。
- [ ] 依赖版本为 `SNAPSHOT`、动态版本，或引入已知高危 CVE 组件。
- [ ] 用 `catch` 兜 `NullPointerException`/`IndexOutOfBoundsException` 等可预检查异常，或 `finally` 中 `return`。
- [ ] 金额/费率用 `double`/`float`，或 `new BigDecimal(double)`，或用 `equals` 比较 `BigDecimal`。
- [ ] `Collectors.toMap` 未传 `mergeFunction`（重复 key 必抛 `IllegalStateException`）。
- [ ] 响应的手机号、身份证、银行卡、姓名未脱敏。
- [ ] 用 `Executors` 创建线程池或用 `new Thread`，线程池队列无界。
- [ ] 日期格式用了大写 `YYYY`（week-year），跨年周会算错年份。

## B. 结构与分层

- [ ] 根包为产品/系统名；单业务域业务包直接在根包下，多业务域才用 `modules/{domain}`；未用 `com.acme.order` 这类业务域名作根包。
- [ ] 按当前业务域数量分包：单域扁平分层，多域按 `modules/{domain}` 分包；域内再分 Controller/Service/Mapper。
- [ ] 业务 Service 为 `XxxService` 接口 + `service/impl/XxxServiceImpl` 实现，Controller 只依赖接口。
- [ ] `@Service`/`@Transactional` 只在实现类，接口未暴露 Entity。
- [ ] 依赖方向为 `controller → service → mapper`，无反向依赖与循环依赖。
- [ ] `common` 不包含业务逻辑，不依赖业务包。
- [ ] Controller 保持薄：只接收、校验、调用、包装响应。
- [ ] 事务只在 Service，远程调用与 MQ 发送不在事务内。
- [ ] Entity、DTO、Req、Resp 职责清晰，转换使用 Hutool `BeanUtil`，入参转 Entity 已忽略敏感字段。
- [ ] 对象后缀统一为 Entity/Req/Resp/DTO，未混用 PO/VO/DO/BO 别名，未为每张表创建空 BO/DO。
- [ ] 无跨域共享 Entity、无跨库 join。
- [ ] 无无意义 Manager 中转层；跨域编排才使用 Manager。

## C. 技术栈与配置

- [ ] 依赖版本在版本矩阵内，新增依赖有理由且已登记。
- [ ] 构造器注入 + `private final`，无字段 `@Autowired`。
- [ ] 无 `System.out`、`printStackTrace()`、注释掉的代码。
- [ ] 配置按环境拆分，密钥来自环境变量/配置中心。
- [ ] Druid 连接池 `max-active`/`max-wait` 有界，生产关闭或鉴权 `stat-view-servlet`；线程池使用有界队列。
- [ ] 已排除 HikariCP，依赖树中没有两套连接池。
- [ ] MyBatis-Plus 分页插件配置 `maxLimit`，无内存分页。
- [ ] Jackson 时间格式、时区、金额与 ID 序列化符合契约。
- [ ] 生产关闭 Knife4j/OpenAPI（`knife4j.production=true` 且网关拒绝文档路径），Actuator 只暴露必要端点。
- [ ] 依赖声明在 `<dependencies>`、版本仲裁在 `<dependencyManagement>`；同 `groupId`+`artifactId` 无多版本并存。
- [ ] JVM 参数 `-Xms` = `-Xmx`，已开 `-XX:+HeapDumpOnOutOfMemoryError`，显式设置时区与编码。
- [ ] 服务端日志 `maxHistory`/`totalSizeCap` 已配置，磁盘不会被日志打满。

## D. API 契约

- [ ] URL 为 `/api/v1/{resources}`，资源复数、kebab-case，无动词基础路径。
- [ ] 统一返回 `Result`，成功 `code=0`，失败有已登记错误码。
- [ ] HTTP 状态码与错误类型匹配，401/403/404/409/429/500 使用正确。
- [ ] 请求参数校验完整，错误提示为可读中文。
- [ ] 分页参数有默认值与上限，排序字段走白名单。
- [ ] ID 字段以字符串返回，金额以字符串返回，时间格式统一。
- [ ] 出参字段精简，不含敏感信息与内部状态。
- [ ] 破坏性变更已升版本，废弃字段有迁移周期。
- [ ] 创建/支付/退款支持幂等键，重复请求不会产生重复数据。
- [ ] 接口文档与实际实现一致，示例数据脱敏。
- [ ] 列表为空返回 `[]`、对象为空返回 `{}`；`pageNum < 1` 与超过总页数有明确处理。
- [ ] 超过 2^53 的整数（订单号、流水号）以字符串返回，未被 JS 精度改写。
- [ ] URL 参数未超 2048 字节，批量 ID 走 body；请求体大小在网关限制内。
- [ ] 错误响应同时具备 HTTP 状态码、错误码、`traceId` 与用户可读提示。

## E. 数据库

- [ ] 表名、字段名 `snake_case`，无保留字、无拼音。
- [ ] InnoDB + `utf8mb4`，表与字段注释齐全，枚举注释列全 code。
- [ ] 主键 `BIGINT`，自增/雪花策略与 Java 一致。
- [ ] 业务表具备 `create_time/update_time/create_by/update_by/deleted/version`。
- [ ] 金额 `DECIMAL(18,2)`，时间 `DATETIME(3)`，布尔 `TINYINT`，状态/枚举 `VARCHAR(32)`。
- [ ] 无 `FLOAT/DOUBLE` 存金额、无 `TIMESTAMP`、无 MySQL `ENUM/SET`、无物理外键。
- [ ] 索引数量与列顺序合理，核心查询有索引覆盖，无冗余索引。
- [ ] 唯一索引与逻辑删除兼容（必要时使用 `deleted_at`）。
- [ ] 新增 SQL 已 `EXPLAIN`，无全表扫描、无 filesort 大排序。
- [ ] 无深分页 `OFFSET`、无 `LIKE '%x%'`、无索引列函数/隐式转换。
- [ ] 批量写入分批，事务短小，无长事务。
- [ ] 表结构变更通过版本化迁移脚本，未修改已执行脚本。
- [ ] 多表 SQL 的列都加了表别名；`SUM` 用 `IFNULL` 包裹；`NULL` 判断用 `IS NULL` 而非 `= NULL`。
- [ ] `count` 为 0 时直接返回，未继续执行分页查询。
- [ ] `ORDER BY` 字段位于组合索引末尾，`EXPLAIN` 的 `type` 达到 `range` 及以上。
- [ ] 逻辑删除方案下的唯一索引冲突已处理，未依赖代码判重。

## F. 异常与日志

- [ ] 业务异常只使用 `BizException` + 已登记 `ErrorCode`。
- [ ] 全局处理器覆盖参数校验、请求体解析、404/405、唯一键、兜底异常。
- [ ] 异常包装保留 cause，无空 catch、无吞异常。
- [ ] 日志使用 SLF4J 占位符，异常对象作为最后一个参数。
- [ ] 日志级别合理，预期业务拒绝不打 ERROR。
- [ ] 日志包含 traceId 与业务 ID，未重复打印同一异常。
- [ ] 异步线程/MQ 消费传递 traceId 并捕获异常。
- [ ] 日志与响应均无敏感信息，用户输入写日志前已清洗换行。
- [ ] 关键操作有审计日志。
- [ ] trace/debug 日志有 `isXxxEnabled()` 开关；`additivity="false"` 已配置，无重复输出。
- [ ] 日志保留期 ≥ 15 天，敏感操作日志 ≥ 6 个月并多机备份；扩展日志按 `{appName}_{logType}_{logName}.log` 命名。
- [ ] 未用 JSON 工具序列化对象打日志，异常日志带现场信息与堆栈。

## G. 安全

- [ ] 认证方式明确，Token 有效期、刷新、吊销机制齐全。
- [ ] 接口默认拒绝，后台接口独立授权，权限码无硬编码角色。
- [ ] 单条资源访问同时校验 `id + owner/tenant`。
- [ ] 请求中的 `userId/tenantId` 不被信任，由服务端上下文覆盖。
- [ ] SQL 参数化，动态表名/列名/排序走白名单。
- [ ] CSRF 策略与凭证方式匹配，CORS 白名单无通配。
- [ ] 上传校验大小、扩展名、魔数、存储路径，文件不可执行。
- [ ] 外部 URL 请求有白名单与私网拦截，HTTP 客户端有超时与 TLS 校验。
- [ ] 禁用不安全的反序列化与 XML 外部实体。
- [ ] 密钥不入库、不入 Git、不进日志；敏感字段加密/脱敏。
- [ ] 登录、短信、支付等接口有限流。
- [ ] 依赖无已知高危 CVE，生产 HTTPS。

## H. 测试

- [ ] 新增/修改 Service 有单元测试，覆盖异常与边界。
- [ ] Mapper/SQL 变更使用真实 MySQL 集成测试。
- [ ] 测试不依赖生产库、真实第三方、固定 ID、真实时间或执行顺序。
- [ ] 金额用 `isEqualByComparingTo`，时间使用固定 Clock 或容差。
- [ ] 覆盖逻辑删除、唯一索引、乐观锁、幂等、权限等关键语义。
- [ ] 无被禁用的测试；`mvn clean verify` 全绿。
- [ ] 满足 AIR 原则（全自动、用例独立、可重复），用断言判定而非 `System.out`。
- [ ] 用例设计覆盖 BCDE（边界、正确、设计、错误输入）；测试在 `src/test/java`。
- [ ] 数据库数据由程序准备，测试自动回滚或数据带统一前后缀标识。

## I. 性能与可靠性

- [ ] 无 N+1 查询；列表接口不逐条查关联数据。
- [ ] 批量操作使用批量 API，单批不超过 1000 条。
- [ ] 分页有上限，列表默认有稳定排序。
- [ ] 无一次性加载全表/超大集合到内存。
- [ ] 外部调用设置超时、限流、重试上限与降级。
- [ ] 缓存使用有 key 设计、TTL、穿透/雪崩防护（如使用）。
- [ ] 事务内无远程调用、无大文件处理、无用户交互。
- [ ] 无不受控的线程创建与无界队列。

## J. 可读性与可维护性

- [ ] 命名符合规范，无缩写歧义、无拼音。
- [ ] 方法 ≤ 80 行，嵌套 ≤ 3 层，参数 ≤ 5 个。
- [ ] 无魔法值，状态/错误码/常量已枚举或常量化。
- [ ] 无重复代码块，无死代码、无未使用 import/变量。
- [ ] 复杂业务写"为什么"，公共接口有 Javadoc。
- [ ] `TODO` 带负责人与工单号。
- [ ] 注释与代码一致，无过期注释。
- [ ] 类、公有属性、公有方法、抽象方法、枚举常量都有 Javadoc；类注释含 `@author` 与 `@date`。
- [ ] 未留下被注释掉的代码（确有保留必要已用 `///` 说明原因）。
- [ ] 命名无歧视性用语（`allowList`/`blockList`），未混用 `get`/`query`、`delete`/`remove`。

## 评审输出模板

```text
## 结论
存在 2 个阻断项、3 个建议项。

## 阻断
1. src/.../OrderServiceImpl.java:42
   问题：事务方法内调用支付网关，长事务占用连接。
   规则：architecture.md 第 3 节 / SKILL.md 硬规则 3。
   修复：将支付调用移到事务提交后，状态更新与调用分离。

## 建议
1. src/.../OrderController.java:28
   问题：路径使用 /getOrder，不符合资源化命名。
   修复：改为 GET /api/v1/orders/{id}。

## 残余风险
- 未覆盖并发取消场景，建议补充乐观锁集成测试。
```

- 行号必须来自实际文件，禁止猜测。
- 修复建议以最小改动为原则，不借评审做无关重构。
- 若用户要求直接修改，先修阻断项，再按确认范围处理建议项。

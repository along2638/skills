# 架构与代码结构

## 1. 设计原则

- **默认单业务域扁平分包**：只有一个业务域时，业务包直接放在根包下：`controller`、`service`、`service/impl`、`mapper` 等，不再套 `modules/{domain}`。
- **多业务域才引入 `modules`**：出现第二个业务域时，再把各域整体迁入 `modules/{domain}`，形成 `com.acme.mall.modules.order.*`；单域项目不预留 `modules` 空壳。
- **根包与业务解耦**：根包使用产品/系统名（如 `com.acme.mall`）；禁止用 `com.acme.order` 这类业务域名作为根包，否则新增模块会变成 `com.acme.order.modules.user`。
- **依赖单向**：`controller → service 接口 → service/impl → mapper`；跨域只调对方 Service 接口，不依赖对方 `Impl`。
- **对象分层**：Entity 只对数据库，Req/Resp 只对接口，DTO 用于内部传递。禁止一套对象走到底。
- **common 稳定**：只放与业务无关的返回体、异常、配置、工具；不得依赖任何业务包。
- **默认单模块**：没有独立部署、独立团队、独立数据库之前，不拆微服务、不建 Maven 多模块。

## 2. 单模块结构（默认）

```text
mall-service
├── pom.xml
└── src
    ├── main
    │   ├── java/com/acme/mall
    │   │   ├── MallApplication.java
    │   │   ├── common
    │   │   │   ├── api
    │   │   │   │   ├── Result.java
    │   │   │   │   ├── PageReq.java
    │   │   │   │   ├── PageResp.java
    │   │   │   │   ├── ErrorCode.java
    │   │   │   │   └── CommonErrorCode.java
    │   │   │   ├── config
    │   │   │   │   ├── MybatisPlusConfig.java
    │   │   │   │   ├── JacksonConfig.java
    │   │   │   │   ├── WebMvcConfig.java
    │   │   │   │   ├── ThreadPoolConfig.java
    │   │   │   │   └── OpenApiConfig.java
    │   │   │   ├── constant
    │   │   │   ├── enums
    │   │   │   ├── exception
    │   │   │   │   ├── BizException.java
    │   │   │   │   ├── BizAssert.java
    │   │   │   │   └── GlobalExceptionHandler.java
    │   │   │   ├── util
    │   │   │   └── web
    │   │   │       ├── TraceIdFilter.java
    │   │   │       ├── JwtAuthenticationFilter.java
    │   │   │       ├── RestAuthenticationEntryPoint.java
    │   │   │       ├── RestAccessDeniedHandler.java
    │   │   │       └── UserContext.java
    │   │   ├── controller
    │   │   │   └── OrderController.java
    │   │   ├── service
    │   │   │   ├── OrderService.java
    │   │   │   └── impl
    │   │   │       └── OrderServiceImpl.java
    │   │   ├── manager
    │   │   │   └── PaymentManager.java
    │   │   ├── mapper
    │   │   │   └── OrderMapper.java
    │   │   ├── entity
    │   │   │   └── OrderEntity.java
    │   │   ├── enums
    │   │   │   └── OrderStatus.java
    │   │   ├── dto
    │   │   │   ├── OrderCreateReq.java
    │   │   │   └── OrderQueryReq.java
    │   │   ├── vo
    │   │   │   └── OrderResp.java
    │   │   └── convert
    │   │       └── OrderConvert.java
    │   └── resources
    │       ├── application.yml
    │       ├── application-dev.yml
    │       ├── application-prod.yml
    │       ├── logback-spring.xml
    │       ├── mapper/OrderMapper.xml
    │       └── db/migration/V1__init_order.sql
    └── test/java/com/acme/mall
        ├── service/OrderServiceImplTest.java
        └── controller/OrderControllerTest.java
```

根包 `com.acme.mall` 是产品/系统名。只有一个业务域时，业务类直接落在 `controller`、`service`、`mapper` 等包下；出现第二个业务域时，才把各域迁入 `com.acme.mall.modules.{domain}`，例如 `com.acme.mall.modules.user`。不要为单域项目提前创建 `modules` 空壳。

`enums` 有两处，职责不同：根包下的 `enums` 放当前业务域的状态枚举（如 `OrderStatus`），`common/enums` 只放跨域通用枚举（如 `YesNo`）。

Service 强制约定：

- 接口 `service/XxxService.java`，实现 `service/impl/XxxServiceImpl.java`；每个业务 Service 一个接口。
- Controller 只注入 `XxxService`，禁止注入 `XxxServiceImpl`；跨域调用同样只依赖接口。
- `@Service`、`@Transactional` 只在实现类；接口不标注解，接口方法不出现 Entity。
- 单元测试直接测 `XxxServiceImpl`；Controller 切片测试 mock `XxxService` 接口。
- `ServiceImpl` 指业务实现类，不继承 MyBatis-Plus 的 `ServiceImpl`；简单 CRUD 用 `BaseMapper`。

`common` 内容约束：

| 包 | 放什么 | 不放什么 |
|---|---|---|
| `api` | `Result`、`PageReq`、`PageResp`、`ErrorCode` | 具体业务错误码枚举 |
| `config` | 配置类、`@Bean`、拦截器注册 | 业务逻辑、SQL |
| `constant` | 全局常量、请求头名 | 可枚举的状态值（放 enum） |
| `enums` | 跨域通用枚举（如 `YesNo`） | 单个业务域状态 |
| `exception` | 基类异常、全局处理器 | 业务判断 |
| `util` | 无状态纯函数工具 | 依赖数据库/Spring 上下文 |
| `web` | traceId、用户上下文、通用拦截器 | 具体鉴权规则 |

## 3. 分层职责

| 层 | 必须做 | 禁止做 |
|---|---|---|
| Controller | 接收参数、`@Validated`、调 Service、包装 `Result` | 写业务分支、开事务、调 Mapper、返回 Entity |
| Service（接口） | 定义业务语义、入参出参、异常契约 | 写实现、标 `@Service`/`@Transactional`、声明 Entity |
| ServiceImpl（实现） | 业务规则、状态校验、事务边界、对象转换、抛业务异常 | 被 Controller 直接注入、直接操作 `HttpServletRequest`、拼 SQL、调用远程后继续长事务 |
| Manager | 跨域编排、外部 HTTP/RPC、重试与幂等封装 | 作为 CRUD 中转层；无脑包一层 Service |
| Mapper | 单表 CRUD、复杂 SQL、结果映射 | 业务判断、远程调用、事务控制 |
| Convert | Entity / DTO / Resp 字段映射（Hutool `BeanUtil`） | 查库、RPC、业务计算 |
| Entity | 与表结构一一对应 | 业务方法、作为接口出入参、直接序列化给前端 |

规则：

- Controller 方法体通常不超过 15 行，不出现 `if` 业务分支。
- 业务 Service 固定 `XxxService` 接口 + `service/impl/XxxServiceImpl` 实现；Controller、其他 Bean 只依赖接口。
- `@Service` 只标实现类；接口不标。
- 接口方法不得声明 Entity；Entity 只在实现类与 Mapper 之间传递。
- `@Transactional` 只出现在 Service 实现类的 public 方法上。
- 远程调用、MQ 发送放在事务提交之后（`TransactionSynchronization` 或业务编排层）。
- Service 查询返回 Entity 只用于域内；对外必须转 Resp。

分层异常处理（各层职责不同，避免同一异常被打印多次）：

| 层 | 处理方式 |
|---|---|
| Mapper / DAO | 捕获底层异常并转为带 cause 的运行时异常，**不打印日志**（日志由上层统一记录） |
| Service | 捕获后记录 ERROR 日志并带上业务参数（保护案发现场）；业务失败抛 `BizException(ErrorCode)` |
| Manager | 与 Service 同机部署时同 DAO 处理；独立部署时按 Service 处理 |
| Controller / Web | 不再向上抛，由 `@RestControllerAdvice` 统一转 `Result` |
| 开放接口层 | 把异常转成错误码 + 错误信息返回（RPC 场景用 `Result` 包装，不用异常穿透） |

## 4. 业务包与依赖规则

允许：

```text
Controller ---> Service（接口） ---> ServiceImpl ---> Mapper ---> DB
                                      |
                                      +------> Manager ---> 外部系统 / 其他域 Service
```

禁止：

- Controller 直接依赖其他业务域的 Service 或任何 Mapper。
- Controller 直接依赖本域的 `XxxServiceImpl`，或接口方法声明 Entity。
- 域 A 的 Mapper / Entity / ServiceImpl 被域 B 直接引用；多域时表现为 `modules/a` 与 `modules/b` 互相引用。
- `common` 依赖业务包（单域下的 `controller`/`service`/`mapper` 等，或多域下的 `modules`）。
- 业务包之间循环依赖；出现循环时提取共享领域服务或改为事件解耦。
- 在 Controller、Config、Util 中开启事务。

跨域读取优先调用对方 Service 的查询接口（只依赖接口，不依赖 `Impl`）；确需 join 时，由拥有数据的一方提供聚合查询，不在 A 域 SQL 中直接 join B 域的表。

## 5. 对象模型与命名

对象术语与本规范的落地对应：

| 术语 | 含义 | 本规范落地 |
|---|---|---|
| PO（Persistent Object） | 持久化对象，映射数据库表 | `XxxEntity`，放 `entity`；只在持久层与 Service 实现内部使用，不作为接口出入参 |
| DTO（Data Transfer Object） | 跨层、跨服务传输数据 | `dto` 包；入参用 `XxxCreateReq` / `XxxUpdateReq` / `XxxQueryReq`，内部传输用 `XxxDTO` |
| VO（View Object） | 面向调用方/前端展示的数据 | `vo` 包，统一以 `XxxResp` 命名；字段按展示需要裁剪，不直接复用 Entity |
| Query Object | 查询条件 + 分页信息 | `XxxQueryReq extends PageReq`，放 `dto` |
| DAO | 数据访问对象/模式，不是数据载体 | 由 `XxxMapper` + MyBatis-Plus `BaseMapper` 承担 |
| DO / BO | 领域对象 / 业务对象，承载领域行为或聚合 | 默认不建；确有复杂领域行为或聚合时再引入，禁止为每张表套一层空 BO/DO |

- 本规范的 VO 专指 View Object；DDD 的 Value Object 用具体类型或 `XxxValue` 命名，不使用 `XxxVO`，避免混淆。
- 统一后缀为 `Entity`、`Req`、`Resp`、`DTO`，不要同时出现 `OrderPO`、`OrderVO`、`OrderDO` 等别名。
- 对象按边界拆分，不按形式拆分：只有字段差异、展示裁剪或安全边界确实存在时，才新建 DTO/VO，不为每张表机械生成三套相同字段的类。
- 入参 DTO 负责校验与接收，VO 负责展示，Entity 负责持久化；三者不得互相替代。

| 后缀 | 用途 | 包 | 示例 |
|---|---|---|---|
| `Entity` | 数据库映射（PO） | `entity` | `OrderEntity` |
| `Service` | 业务接口 | `service` | `OrderService` |
| `ServiceImpl` | 业务实现（`@Service`/事务） | `service/impl` | `OrderServiceImpl` |
| `CreateReq` | 新建入参 | `dto` | `OrderCreateReq` |
| `UpdateReq` | 更新入参（含 id） | `dto` | `OrderUpdateReq` |
| `QueryReq` | 查询/分页入参 | `dto` | `OrderQueryReq` |
| `DTO` | 内部跨层传递 | `dto` | `OrderDTO` |
| `Resp` | 接口出参（VO） | `vo` | `OrderResp` |
| `Convert` | Hutool `BeanUtil` 转换（`@Component` 具体类） | `convert` | `OrderConvert` |
| `Manager` | 外部/跨域编排 | `manager` | `PaymentManager` |

单域时上表包路径直接位于根包下（如 `com.acme.mall.entity`）；多域时统一加前缀 `modules/{domain}`（如 `com.acme.mall.modules.order.entity`）。

Convert 使用 Hutool `BeanUtil`，类标 `@Component` 并由 Spring 注入；入参转 Entity 必须忽略 `id`、`userId`、`status`、审计与逻辑删除字段，禁止整体复制造成越权赋值。

约束：

- 表的每个字段在 Entity 中有且仅有一个同义字段，使用驼峰映射，不加 `f_` 前缀。
- 对外 ID、外键 ID 加 `@JsonSerialize(using = ToStringSerializer.class)`，避免雪花 ID 在前端丢精度。
- 布尔字段不使用 `is` 前缀（用 `enabled`），避免 Lombok 生成 `isEnabled()` 后 JSON 变成 `enabled`/`isEnabled` 不一致。
- 金额字段类型固定 `BigDecimal`；时间固定 `LocalDateTime`/`LocalDate`；禁止 `Date`、`double`。
- 列表接口：分页返回 `PageResp<XxxResp>`；固定小列表返回 `Result<List<XxxResp>>`。
- 禁止在 Req 中使用 `Map<String, Object>` 承载业务字段。

## 6. Mapper 与 XML

- 简单 CRUD 用 `BaseMapper` + `LambdaQueryWrapper`，不写 XML。
- 多表、聚合、报表、复杂动态条件才写 XML。
- XML 固定放在 `src/main/resources/mapper/{Entity}Mapper.xml`；多域时按域分目录 `mapper/{domain}/{Entity}Mapper.xml`。
- `namespace` 必须是 Mapper 接口全限定名；`resultMap` 显式声明，尤其是枚举与 `JSON` 字段。
- 手写 XML 一律用 `resultMap`，不用 `resultType` 直接映射到 Entity：字段改名、类型处理器调整时只改映射，不动 SQL。单表查询由 MyBatis-Plus 的 `@TableName`/`@TableField` 承担映射，不需要额外写 `resultMap`。
- 禁止用 `HashMap`/`Hashtable` 接收查询结果（`bigint` 在不同驱动版本下会映射为 `Long` 或 `BigInteger`）。
- 公共列抽 `<sql id="Base_Column_List">`，但查询必须按需裁剪，不默认全列。
- 动态 SQL 使用 `<if>`、`<choose>`、`<foreach>`；禁止 `${}` 拼接。
- 批量写入优先 `saveBatch`；超过 1000 条分批，单批 500 条左右。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.acme.mall.mapper.OrderMapper">

    <sql id="Base_Column_List">
        id, order_no, user_id, amount, status, create_time, update_time, deleted, version
    </sql>

    <select id="selectByOrderNo" resultType="com.acme.mall.entity.OrderEntity">
        SELECT <include refid="Base_Column_List"/>
        FROM trade_order
        WHERE order_no = #{orderNo}
          AND deleted = 0
        LIMIT 1
    </select>
</mapper>
```

## 7. 多模块拆分（满足条件才做）

触发条件（满足其一）：独立部署、独立数据库、独立团队发布、需要向其他服务提供契约。

```text
parent/
├── pom.xml               只做 dependencyManagement 与插件版本
├── app/                  启动类、Web 层、配置（可部署）
├── common/               返回体、异常、工具（无业务）
├── api/                  跨服务 DTO + Feign 接口（只含契约）
└── modules/
    ├── order-service/    订单领域实现
    └── user-service/     用户领域实现
```

- `parent` 不放具体依赖，只管理版本。
- 这里的 `modules/` 是 Maven 模块目录，与 Java 包下的 `modules/{domain}` 无关；目录名可以自定义。
- `app` 依赖各 `modules`；`modules` 依赖 `common`；`api` 不依赖实现。
- 不允许 `modules` 依赖 `app`，不允许实现模块之间共享 Entity/Mapper。
- 微服务场景每个服务独占数据库，禁止跨库 join；跨服务调用走 `api` 契约，并设置超时、重试上限与降级。
- `api` 中的 DTO 一旦发布，字段只增不删，语义不变；破坏性变更升版本。

## 8. 配置类职责

| 配置类 | 必配内容 |
|---|---|
| `MybatisPlusConfig` | 分页插件（`DbType.MYSQL`、`maxLimit=500`）、乐观锁插件、防全表更新删除插件、审计字段自动填充 |
| `JacksonConfig` | `JavaTimeModule`、`LocalDateTime` 格式 `yyyy-MM-dd HH:mm:ss`、时区 `Asia/Shanghai`、忽略未知字段 |
| `WebMvcConfig` | 拦截器注册、路径匹配、静态资源（如有） |
| `ThreadPoolConfig` | 有界队列、线程命名、`CallerRunsPolicy`、拒绝日志 |
| `OpenApiConfig` | Knife4j/OpenAPI 3 文档分组与鉴权说明；生产环境设置 `knife4j.production=true` |
| `SecurityConfig` | `SecurityFilterChain`（Spring Security 5.7 写法），见 `security.md` |

禁止：

- 多个 `@MapperScan` 分散在业务类上；统一在配置类声明。
- 在 `@PostConstruct` 中执行外部调用或大批量数据加载。
- 配置类里写业务分支。

## 9. 资源与迁移

- 环境配置：`application.yml` 只放公共项；`dev`、`test`、`prod` 分文件；本地密钥放环境变量或 gitignore 的 `application-local.yml`。
- 表结构：使用版本化 SQL（推荐 Flyway 8.5.13），命名 `V{序号}__{描述}.sql`，例如 `V12__add_order_index.sql`。
- 已执行过的迁移脚本不得修改；修复用新版本脚本。
- 生产变更必须走发布流程，禁止手工改表结构。
- `logback-spring.xml` 分环境输出，生产 JSON；`mapper` XML 不放在 Java 包内。
- 不提交 IDE 配置、本地日志、构建产物、真实密钥。

## 10. 设计规约

设计必须留痕，存储方案与核心数据结构必须先评审再动手。

- 存储方案、表结构、索引方案必须评审通过并沉淀为文档；生产执行后做一次 double check。评审内容：存储介质选型、表结构是否支撑技术方案、容量与性能、字段与索引的合理性、字段命名与类型。
- 表结构变更（含加字段）同样要评审，不允许"顺手加个字段"。

用什么图（达到阈值就必须画，不靠口头描述）：

| 触发条件 | 图 | 必须表达 |
|---|---|---|
| 交互角色 > 1 类且用例 > 5 个 | 用例图 | 角色、用例边界 |
| 业务对象状态 > 3 个 | 状态图 | 全部状态、可达转换、每个转换的触发条件 |
| 调用链路涉及对象 > 3 个 | 时序图 | 调用顺序、每步的输入与输出 |
| 模型类 > 5 个且依赖复杂 | 类图 | 类关系、聚合/组合/依赖方向 |
| 协作对象 > 2 个且流程复杂 | 活动图 | 泳道、并发分支、异常分支 |

架构设计必须明确四件事：系统边界（做什么、不做什么）、模块关系与依赖方向、后续演化遵循的原则、非功能性需求（安全、可用性、可扩展性、性能指标）。

设计原则：

- **单一职责**：一个类只有一个变化原因；系统演进中最容易被忘记的一条。
- **组合优于继承**：优先聚合/组合；必须继承时满足里氏代换（父类出现的地方子类必须能出现）。
- **依赖倒置**：依赖抽象与接口，不依赖具体实现，便于替换与测试。
- **开闭原则**：对扩展开放、对修改关闭；新需求尽量通过新增实现完成，而不是改老代码的分支。
- **DRY**：共性逻辑抽公共方法/类/组件；复制粘贴的代码在维护时会漏改副本。
- **隔离变化点**：可扩展性的本质是识别系统里会变的部分并把它隔离出去（策略、SPI、插件）。

其他要求：

- 需求与设计必须评估异常流程与业务边界，不能只写主流程（如"扣款成功但回调失败"必须有补偿方案）。
- 设计文档的作用是明确需求、理顺逻辑、支撑后期维护，次要目的才是指导编码；避免为设计而设计。
- "代码即文档"是错的：模块依赖、调用纵深、业务场景、非功能约束都需要文档表达。
- 关键设计决策记录为 ADR（背景、备选方案、结论、后果），与代码同仓维护。
- 面向公众的产品需考虑无障碍：可交互控件可 Tab 聚焦且顺序自然、验证码提供图形以外的方式、自定义控件声明 `role`。

## 11. 新建业务域检查项

1. 包结构符合当前域数量：单域时业务包直接在根包下，多域时为 `modules/{domain}`；根包为产品/系统名，不使用业务域作为根包。
2. Entity 字段与表字段一一对应，含审计字段与逻辑删除。
3. Mapper 继承 `BaseMapper`，复杂 SQL 有 XML 且无 `${}`。
4. Req 完成 JSR-303 校验，Resp 不含 Entity 与敏感字段。
5. 业务 Service 为 `XxxService` 接口 + `service/impl/XxxServiceImpl` 实现；Controller 只依赖接口，事务在实现类。
6. Controller 路径符合 `/api/v1/{resources}`，统一返回 `Result`。
7. 错误码已登记，业务异常使用 `BizException`。
8. 单元测试覆盖主流程与关键状态分支。

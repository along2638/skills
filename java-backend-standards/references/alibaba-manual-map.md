# 与《Java 开发手册》嵩山版（v1.7.0）的对照

本规范以《Java 开发手册》嵩山版（2020.08.03，本地文件 `阿里巴巴Java开发手册-嵩山版.pdf`，正文 59 面）为编码规约来源，按本项目基线（Spring Boot 2.7 + JDK 8 + MyBatis-Plus + MySQL 8.0）重写落地。手册是框架中立的通用规约，不含 Spring Boot / MyBatis-Plus 的具体写法，因此本规范只继承其**约束意图**，不照搬表述。

## 1. 规约等级的落地方式

| 手册等级 | 本规范处理 |
|---|---|
| 【强制】 | 转为硬规则；违反即评审**阻断**项 |
| 【推荐】 | 转为评审**建议**项，不阻断上线 |
| 【参考】 | 按场景裁量，写入说明性内容，不作为检查项 |

## 2. 章节对照

| 手册章节 | 落地位 |
|---|---|
| 一、编程规约 (一) 命名风格 | `java-coding.md` §1 |
| (二) 常量定义、(三) 代码格式 | `java-coding.md` §2 |
| (四) OOP 规约 | `java-coding.md` §4、§5 |
| (五) 日期时间 | `java-coding.md` §6 |
| (六) 集合处理 | `java-coding.md` §7 |
| (七) 并发处理 | `java-coding.md` §9 |
| (八) 控制语句 | `java-coding.md` §8 |
| (九) 注释规约 | `java-coding.md` §3 |
| (十) 前后端规约 | `api-contract.md` §2.4、§5、§10 |
| (十一) 其他 | `java-coding.md` §7、§18 |
| 二、异常日志 (一) 错误码 | `api-contract.md` §3、§3.1 |
| (二) 异常处理 | `exception-logging.md` §1 |
| (三) 日志规约 | `exception-logging.md` §3、§4、§7 |
| 三、单元测试 | `testing.md` §1、§6、§8 |
| 四、安全规约 | `security.md` §2、§4、§11 |
| 五、MySQL (一) 建表规约 | `database.md` §1、§2、§3 |
| (二) 索引规约 | `database.md` §4 |
| (三) SQL 语句 | `database.md` §6 |
| (四) ORM 映射 | `architecture.md` §6、`java-coding.md` §14、§15 |
| 六、工程结构 (一) 应用分层 | `architecture.md` §3 |
| (二) 二方库依赖 | `tech-stack.md` §5.1 |
| (三) 服务器 | `tech-stack.md` §6 |
| 七、设计规约 | `architecture.md` §10 |

## 3. 有意的偏差

以下条款本规范**故意**与手册不一致。冲突时以本规范为准；改回手册写法前，先确认下表理由已不成立。

| # | 手册要求 | 本规范 | 理由 |
|---|---|---|---|
| 1 | 布尔字段 `is_xxx` + `unsigned tinyint`，在 `resultMap` 做 `is_xxx → xxx` 映射 | `deleted`/`enabled` + `TINYINT` | 手册自身又禁止 POJO 布尔属性带 `is` 前缀，两处要求互相打架，必然要写额外映射；`deleted` 与 MyBatis-Plus `@TableLogic`、Lombok `getDeleted()` 直接对齐 |
| 2 | 货币金额以最小货币单位（分）整型存储 | `DECIMAL(18,2)` + `BigDecimal` | 与表结构、报表、对账、财务系统一致；整型分位要在每一层换算，漏一处就是百倍差错；确需分位时只在对外边界转换 |
| 3 | URL 多词分隔用下划线 | kebab-case（`/order-items`） | 与 REST 生态、前端路由、网关规则一致；下划线与短横线混用比统一更危险 |
| 4 | 接口路径不带版本号，版本放 HTTP 头 | 路径 `/api/v1` | 网关、访问日志、缓存、灰度都能按路径直接分流与观测；头版本在客户端混用版本时几乎无法排查 |
| 5 | 错误码 5 位字符串 `A/B/C + 4 位数字` | 数字区间码（`1xxxx`/`2xxxx`/`3xxxx`/`4xxxx`/`5xxxx`） | 与 HTTP 状态、模块号、前端 `switch` 判断自然对应；保留 `api-contract.md` §3.1 的双向映射表 |
| 6 | 分层对象命名 DO/BO/DTO/VO/AO | `Entity` / `Req` / `Resp` / `DTO` | 与 MyBatis-Plus、Spring 生态术语一致，少一层概念映射；DO 的职责由 Entity 承担，AO 在本项目无对应场景 |
| 7 | 主键索引名 `pk_字段名` | `PRIMARY`（MySQL 固定命名） | InnoDB 主键索引名不可自定义，写进规范只会造成"知道但做不到" |
| 8 | 非负字段一律 unsigned | 仅非主键计数列可选 unsigned | `BIGINT UNSIGNED` 与 Java `Long` 有符号语义不一致，比较、减法、JSON 序列化都容易出错 |
| 9 | 前后端时间统一 GMT | `Asia/Shanghai` | 前后端与数据库同处东八区，统一东八区更少心智负担；跨时区调用时在契约中显式声明偏移量 |
| 10 | 单表分库分表阈值 500 万行 / 2GB | 同（采纳） | 无偏差，列入 `database.md` 作为评估门槛而非目标 |

## 4. 未采纳或降级的条目

| 手册条目 | 处理 | 原因 |
|---|---|---|
| Velocity 相关（`$!{var}`、视图不写复杂逻辑） | 仅在维护存量 JSP/Velocity 项目时参考 | 本项目前后端分离，无服务端模板渲染 |
| iBATIS `queryForList(String, int, int)`、`<isEqual>`/`<isNotEmpty>`/`<isNotNull>` | 不采纳 | iBATIS 时代 API 与标签；MyBatis/MP 用 `Page` 分页与 `<if>` 动态条件表达同一语义 |
| 用 `resultClass` 直接映射 | 改为强制 `<resultMap>` | MyBatis 已无 `resultClass`；MP 单表映射由 `@TableName`/`@TableField` 承担 |
| 核心模块语句与分支覆盖率 100% | 采纳为目标值，准入底线 80% | 存量项目短期内达不到 100%；用增量覆盖率卡门禁更现实 |
| 服务器 `tcp_fin_timeout`、fd 上限等内核参数 | 采纳为部署清单项 | 云托管/容器环境由平台负责时标 N/A，不强制运维逐个改 |
| 错误码附表的完整 A/B/C 编码表 | 只保留三级宏观码映射 | 具体业务码由各团队在错误码平台登记，抄一份到文档必然过期 |

## 5. 使用约定

- 生成代码按本规范执行；需要解释"为什么这么定"时，引用本文件偏差表，不要引用手册原文。
- 代码评审引用本规范的文件与章节，不引用"手册第几条"——手册版本会演进，条款号不稳定。
- 手册发布新版本时，按第 3、4 节逐条复核偏差是否仍成立，失效的及时改回。
- 本规范与手册的冲突只在第 3 节列出的范围内成立；未列出的条款，手册要求即为本规范要求。

# 数据库规范（MySQL 8.0）

> 本文件整合《Java 开发手册》嵩山版"MySQL 数据库"章节（建表、索引、SQL、ORM 映射），并按本项目基线的逻辑删除方案与命名习惯调整。与手册不一致处见 `alibaba-manual-map.md`。

## 1. 基本约定

| 项 | 规定 |
|---|---|
| 数据库名 | 小写 `snake_case`，`{project}_{env}`，如 `order_dev`、`order_prod` |
| 表名 | 小写 `snake_case`，单数，`{业务域}_{实体}`，如 `trade_order`、`sys_user` |
| 字段名 | 小写 `snake_case`，无类型前缀，如 `user_id`、`create_time` |
| 存储引擎 | 一律 InnoDB |
| 字符集 | `utf8mb4` |
| 排序规则 | MySQL 8 用 `utf8mb4_0900_ai_ci`；需兼容 5.7/MariaDB 时用 `utf8mb4_general_ci` |
| 注释 | 每张表、每个字段必须有 `COMMENT`；枚举字段注释写全量映射 |
| 标识符长度 | 表名/字段名 ≤ 64，索引名 ≤ 64；表名建议 ≤ 30 |
| 大小写 | Linux 下大小写敏感，禁止依赖大小写区分 |

- 全库统一表名前缀策略。采用 `sys_`/`trade_`/`biz_` 这类业务域前缀时，不要再混用 `t_`；存量项目已有 `t_` 前缀则保持全库一致。
- 表名用单数名词，表示实体而非数量：`trade_order` 不是 `trade_orders`，对应 Entity 也是单数。
- 表名推荐"业务名称_表的作用"：`alipay_task`、`trade_config`；库名与应用名尽量一致。
- 表名、字段名只用小写字母、数字、下划线；禁止数字开头，禁止两个下划线中间只出现数字（`level_3_name` 不可用，改写 `level3_name` 或 `level_third_name`）。
- 布尔字段用形容词，不加 `is_` 前缀：`enabled`、`deleted`、`paid`。手册要求 `is_xxx + unsigned tinyint`，本规范改用 `deleted/enabled + TINYINT`（MyBatis-Plus `@TableLogic` 与 Lombok 生成的 `getDeleted()` 更直接），偏差理由见 `alibaba-manual-map.md`。
- 命名后缀约定：时间 `_time`，日期 `_date`，金额 `_amount`，数量 `_count`，状态 `status`，类型 `_type`，编码 `_code`，外键 `_id`。
- 禁止使用 MySQL 保留字。常见：`order`、`group`、`key`、`desc`、`rank`、`system`、`window`、`range`、`rows`、`groups`、`over`、`recursive`、`usage`、`interval`、`match`、`condition`、`lead`、`row_number`。
- 禁止用拼音、无意义缩写（`bz`、`sl`）；英文单词过长可约定缩写，但需全库一致。

## 2. 主键与必备字段

- 每张表必须有单列主键 `id BIGINT`；禁止复合主键、禁止业务字段作主键。
- 单库默认 `id BIGINT NOT NULL AUTO_INCREMENT`，Java 侧 `IdType.AUTO`。
- 分库分表或多写入节点用雪花 ID：`id BIGINT NOT NULL`，Java 侧 `IdType.ASSIGN_ID`；同一张表不得两种策略混用。
- 不使用 `BIGINT UNSIGNED`，避免与 Java `Long` 有符号语义不一致。手册要求"非负字段一律 unsigned"，本规范只在非主键的计数列（如 `view_count`）上可选用 `INT UNSIGNED`，主键与外键保持有符号。
- 业务唯一键必须建唯一索引，不能只靠代码判重。
- 手册的"必备三字段 `id`/`create_time`/`update_time`"是本规范必备字段的最小集；业务表在此基础上追加审计、逻辑删除与乐观锁字段。
- 是否分库分表看数据量：单表超过 500 万行或容量超过 2GB 才评估，预计三年达不到就不要提前分片。

业务表必备字段：

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `id` | `BIGINT` | 自增/雪花 | 主键 |
| `create_time` | `DATETIME(3)` | `CURRENT_TIMESTAMP(3)` | 创建时间 |
| `update_time` | `DATETIME(3)` | `CURRENT_TIMESTAMP(3)` | 更新时间，建议加 `ON UPDATE` |
| `create_by` | `BIGINT` | `0` | 创建人，0 表示系统 |
| `update_by` | `BIGINT` | `0` | 更新人 |
| `deleted` | `TINYINT` | `0` | 0 未删除，1 已删除 |
| `version` | `INT` | `0` | 乐观锁版本 |

- 纯日志、流水、审计表可省略 `deleted`、`version`，但必须有 `create_time`，且数据只增不改。
- 多租户系统每张业务表必须有 `tenant_id BIGINT NOT NULL DEFAULT 0`，并纳入所有唯一索引与查询条件。
- 应用层通过 MyBatis-Plus `MetaObjectHandler` 填充审计字段；数据库默认值只作兜底。
- `create_by`/`update_by` 不得填用户名等字符串；用户标识统一用 ID。

## 3. 类型映射

| Java | MySQL | 规则 |
|---|---|---|
| `Long` | `BIGINT` | ID、外键、大计数 |
| `Integer` | `INT` | 状态、类型、排序；计数小于 21 亿可用 |
| `Boolean` | `TINYINT` | 0/1，`NOT NULL DEFAULT 0` |
| `String` | `VARCHAR(n)` | 按业务定长：编码 32，名称 64，标题 128，邮箱 128，URL 512，备注 500 |
| `BigDecimal` | `DECIMAL(18,2)` | 金额；单价/费率可用 `DECIMAL(20,6)` |
| `LocalDateTime` | `DATETIME(3)` | 毫秒精度 |
| `LocalDate` | `DATE` | 只到日 |
| `LocalTime` | `TIME(3)` | 只到时 |
| 枚举/状态 | `VARCHAR(32)` | 存 code，可读、可扩展；禁止用散落数字码代替枚举 |
| JSON 对象 | `JSON` | MySQL 8 原生类型，需配置 TypeHandler |
| `byte[]` | `VARBINARY`/`BLOB` | 尽量避免，文件放对象存储 |
| UUID | `CHAR(36)` / `BINARY(16)` | 仅在确有需要时使用 |

禁止：

- 用 `FLOAT`/`DOUBLE` 存金额、费率、库存等精确值。
- 用 `TIMESTAMP` 存业务时间（2038 问题 + 时区隐式转换），统一 `DATETIME(3)`。
- 使用 MySQL `ENUM`、`SET` 类型。
- `VARCHAR(255)` 一把梭；长度按业务上限设置。
- `VARCHAR` 长度超过 5000：超过时改 `TEXT` 并独立一张扩展表，用主键关联，避免拖慢其它字段的索引效率。
- 把长文本、JSON、二进制大字段与高频查询字段放同一张宽表；必要时拆扩展表。
- 可空字段滥用：必填字段必须 `NOT NULL` 并给默认值；只有确实"未知/不适用"才允许 `NULL`。
- 用 `NULL` 表示 0、空字符串、未设置等语义。

长度几乎相等的字符串用 `CHAR` 定长（如身份证、统一社会信用代码、`CHAR(2)` 国家码）；`CHAR` 查询无需读长度前缀，短定长场景更快。

整型按实际范围选，不一律 `BIGINT`：

| 场景 | 类型 | 无符号范围 |
|---|---|---|
| 年龄、等级、状态码 | `TINYINT` | 0 ~ 255 |
| 数量、天数 | `SMALLINT` | 0 ~ 65535 |
| 计数、排序号 | `INT` | 0 ~ 约 43 亿 |
| ID、外键、大计数 | `BIGINT` | 0 ~ 约 10^19 |

字段长度建议：

| 场景 | 类型 |
|---|---|
| 手机号（国内） | `VARCHAR(20)`（考虑国家码） |
| 邮箱 | `VARCHAR(128)` |
| 订单号/业务编码 | `VARCHAR(32)` |
| 姓名/昵称 | `VARCHAR(64)` |
| 标题 | `VARCHAR(128)` |
| 地址 | `VARCHAR(255)` |
| URL | `VARCHAR(512)` |
| 备注 | `VARCHAR(500)` |
| 超过 2000 字符的正文 | `TEXT`，且考虑拆表 |
| 金额 | `DECIMAL(18,2)` |

字段冗余（为减少跨服务查询时可适度冗余，但必须能接受一致性延迟）：

- 只冗余不常修改的字段（如商品名称、用户昵称），不冗余频繁变更的字段。
- 不冗余唯一索引字段，否则唯一性语义被破坏。
- 不冗余超长 `VARCHAR`，更不能冗余 `TEXT`。
- 冗余字段的同步策略（写扩散 / 定时校准）必须在设计文档里写明。

JSON 字段写法：

```java
@TableName(value = "trade_order", autoResultMap = true)
public class OrderEntity {

    @TableField(typeHandler = JacksonTypeHandler.class)
    private OrderExt ext;
}
```

## 4. 索引规范

- 主键即聚簇索引；主键要短、递增，避免随机 UUID 造成页分裂。
- 唯一索引命名 `uk_{表名}_{字段简写}`；普通索引 `idx_{表名}_{字段简写}`。
- 业务唯一性必须由唯一索引兜底，代码判重只用于友好提示。
- 复合索引列顺序：**等值条件 → 排序字段 → 范围条件**，遵循最左前缀。
- `ORDER BY` 字段必须是组合索引的最后一段，且位于等值条件之后：`WHERE a = ? AND b = ? ORDER BY c` 建 `idx_a_b_c`。索引列上出现范围查询时无法再利用有序性（`WHERE a > 10 ORDER BY b` 用不了 `idx_a_b`）。
- 区分度最高的列不一定放最左：等值与不等号混合时，等值列必须前置（`WHERE c > ? AND d = ?` 建 `idx_d_c`）。
- 高频查询用覆盖索引避免回表，`EXPLAIN` 的 `Extra` 出现 `Using index` 即命中。
- 深分页用延迟关联或游标，先定位 ID 再关联取列：
  `SELECT t1.* FROM trade_order t1, (SELECT id FROM trade_order WHERE status = ? ORDER BY id LIMIT 100000, 20) t2 WHERE t1.id = t2.id`
- `EXPLAIN` 的 `type` 至少要达到 `range`，争取 `ref`，最好是 `const`；出现 `type = index`（全索引扫描）视为不合格。
- 单表索引数量建议 ≤ 5；写多读少的表 ≤ 3。复合索引列数 ≤ 4。
- 区分度低的字段（状态、性别）不单独建索引；与时间或用户 ID 组成复合索引。
- 索引列禁止套函数、表达式、隐式类型转换；字符串字段与参数类型必须一致。
- 禁止前置模糊 `LIKE '%xx'`；确需全文检索用搜索引擎或全文索引。
- 长字符串可用前缀索引 `KEY idx_name (name(32))`，但唯一性校验不能依赖前缀索引。
- `deleted` 不单独建索引；以 `(业务列, deleted)` 或分区/归档解决。
- 关联字段类型、字符集、排序规则必须一致，且被关联列要有索引。
- 大表排序、分页字段应能被索引覆盖，避免 `Using filesort`、`Using temporary`。
- 新增或修改的 SQL 必须 `EXPLAIN` 后再提交。
- 禁止物理外键；关联关系由应用维护，join 列建索引。
- 大表加索引使用 Online DDL；千万级以上评估 gh-ost / pt-online-schema-change 与变更窗口。

## 5. 逻辑删除

默认方案：

```sql
deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除 1已删除'
```

```java
@TableLogic(value = "0", delval = "1")
private Integer deleted;
```

- 所有业务查询默认排除已删除数据；**Mapper XML 手写 SQL 不会被自动追加条件，必须显式写 `deleted = 0`**。
- 禁止对业务数据执行物理 `DELETE`；清理历史数据走独立归档/清除任务，需审批、备份、限速。
- 删除主表时，关联子表在同一事务内一并逻辑删除；不依赖数据库级联。
- 统计、报表默认排除已删除；若需包含，SQL 中显式注明并走独立接口。

唯一键与逻辑删除冲突：

- 默认 `deleted TINYINT` 方案下，唯一索引不包含 `deleted`，业务编码删除后不可复用（如订单号，正确）。
- 若业务编码删除后必须复用（如用户名、租户内编码），改用：

```sql
deleted_at DATETIME(3) NULL DEFAULT NULL COMMENT '删除时间，NULL 表示未删除'
```

```java
@TableLogic(value = "null", delval = "now()")
private LocalDateTime deletedAt;
```

```sql
UNIQUE KEY uk_sys_user_username (username, deleted_at)
```

MySQL 唯一索引允许多个 `NULL`，因此未删除行仍唯一；已删除行靠删除时间区分。同一毫秒重复删除同一编码的极端情况需应用层加锁或改用 `deleted BIGINT` 存主键值。

## 6. SQL 规范

### 查询

- 禁止 `SELECT *`，必须列出所需字段；大字段（TEXT/JSON/BLOB）非必要不查。
- 分页统一使用 MyBatis-Plus `Page`，`pageSize ≤ 100`；导出等特殊场景单独审批。
- 深分页禁止大 `OFFSET`；改用游标：`WHERE id > #{lastId} ORDER BY id LIMIT #{size}`。
- `IN` 列表 ≤ 1000 个；更大集合分批或改用临时表/关联。
- 多表 join ≤ 3 张；更大规模走宽表、聚合表或搜索引擎。
- 禁止 `ORDER BY RAND()`；随机取数用随机 ID 或预生成策略。
- 时间范围用半开区间：`create_time >= #{start} AND create_time < #{end}`，禁止 `DATE(create_time) = ?`。
- 统计行数用 `COUNT(*)`；不要用 `COUNT(可空列)` 代替。
- `COUNT(DISTINCT col)` 只统计非 `NULL` 的不重复值；`COUNT(DISTINCT c1, c2)` 中任一列全为 `NULL` 时整行返回 0。
- 列值全为 `NULL` 时 `SUM(col)` 返回 `NULL`，Java 侧拆箱即 NPE；统一写 `SELECT IFNULL(SUM(amount), 0)`。
- `NULL` 比较用 `ISNULL(col)` 或 `col IS NULL`，不要写 `col = NULL`（结果恒为 `NULL`，不是 `false`）。
- 分页逻辑先判总数：`count` 为 0 时直接返回空结果，不再执行分页查询。
- 多表查询、更新、删除时，所有列必须加表别名限定（`t1.name`），否则后续加同名列会报 `Column 'name' in field list is ambiguous`。
- 表别名统一加 `as` 并按出现顺序命名 `t1`、`t2`、`t3`。
- `GROUP BY` 必须符合 `ONLY_FULL_GROUP_BY`，不依赖隐藏列。
- 字符长度区分 `LENGTH()`（字节）与 `CHAR_LENGTH()`（字符）；`utf8mb4` 下中文占 3~4 字节。
- 不在开发代码中使用 `TRUNCATE TABLE`（无事务、不触发触发器，误操作不可回滚），历史数据清理走独立归档任务。

### 写入

- `INSERT` 必须显式列名，禁止依赖列顺序。
- 批量插入每批 500～1000 条，JDBC URL 开启 `rewriteBatchedStatements=true`。
- `UPDATE`/`DELETE` 必须带 `WHERE`；生产临时 SQL 必须带 `LIMIT` 并先备份。
- 数据订正（尤其是删除、批量修改）必须先用同样的 `WHERE` 跑 `SELECT` 确认影响范围，确认无误再执行变更。
- 更新只写有改动的字段，不整行覆盖；整行 `UPDATE` 会覆盖并发写入、增加 binlog 体积。
- 批量更新按主键排序，减少死锁；同一事务内更新顺序保持一致。
- 大事务拆小，单事务控制在 1000 行以内、执行时间尽量 < 1s。
- 禁止在事务中等待用户输入、调用远程接口、发送 MQ。
- 需要 `FOR UPDATE` 时，必须命中索引并在事务内使用；只读业务禁止加锁。

### 安全与可维护性

- 参数一律 `#{}`；禁止 `${}`。动态表名/列名只能通过白名单枚举映射。
- 禁止存储过程、触发器、定时事件。
- 禁止 `NOT IN` 配合可空列（结果可能为空），用 `NOT EXISTS` 或确保列非空。
- 禁止对索引列做隐式转换，如 `user_id = '123'`。
- 避免 `OR` 连接不同索引列，改 `UNION ALL` 或拆分查询。
- 生产 SQL 必须带注释说明业务用途；复杂 SQL 在 XML 中保持格式化。
- 慢 SQL 阈值 1s；出现慢查询必须 `EXPLAIN` 并给出优化方案。

## 7. 建表模板

```sql
CREATE TABLE `trade_order` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  `order_no` VARCHAR(32) NOT NULL COMMENT '订单号，全局唯一且不复用',
  `user_id` BIGINT NOT NULL COMMENT '下单用户ID',
  `amount` DECIMAL(18,2) NOT NULL DEFAULT 0.00 COMMENT '订单金额，单位元',
  `status` VARCHAR(32) NOT NULL DEFAULT 'WAIT_PAY'
      COMMENT '状态：WAIT_PAY待支付 PAID已支付 CANCELED已取消 COMPLETED已完成',
  `remark` VARCHAR(500) NOT NULL DEFAULT '' COMMENT '备注',
  `create_time` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `update_time` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
      ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
  `create_by` BIGINT NOT NULL DEFAULT 0 COMMENT '创建人ID，0为系统',
  `update_by` BIGINT NOT NULL DEFAULT 0 COMMENT '更新人ID，0为系统',
  `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除 1已删除',
  `version` INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_trade_order_order_no` (`order_no`),
  KEY `idx_trade_order_user_status_time` (`user_id`, `status`, `create_time`),
  KEY `idx_trade_order_status_time` (`status`, `create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='交易订单';
```

要点：

- `order_no` 全局唯一且永不复用，因此唯一索引不含 `deleted`；若业务允许复用，按第 5 节改用 `deleted_at`。
- `(user_id, status, create_time)` 支持"某用户某状态按时间排序"；`(status, create_time)` 支持后台按状态分页。
- 金额不用 `DOUBLE`；状态注释写全映射；备注非空默认空串。
- 主键索引名由 MySQL 固定为 `PRIMARY`，无法改成 `pk_` 前缀；唯一索引用 `uk_{表}_{字段}`、普通索引用 `idx_{表}_{字段}`。
- `update_time` 的 `ON UPDATE CURRENT_TIMESTAMP(3)` 只作兜底，应用层仍须由 `MetaObjectHandler` 显式填充。

## 8. 迁移与数据治理

- 表结构变更使用版本化脚本（推荐 Flyway），命名 `V{序号}__{描述}.sql`，如 `V13__add_trade_order_index.sql`。
- 已执行的脚本禁止修改；修正通过新脚本完成。
- 生产执行顺序：备份 → 评审 → 灰度/低峰执行 → 校验 → 记录；DDL 与应用发布解耦，先兼容后清理。
- 新增非空列必须给默认值或分三步：加可空列 → 回填 → 改非空。
- 大表加索引、改字段类型需评估锁表时间，必要时使用在线变更工具。
- 种子数据脚本必须幂等（`INSERT ... ON DUPLICATE KEY UPDATE`）。
- 应用连接账号只授予 DML 权限；DDL 由迁移账号执行。
- 手机号、身份证、银行卡等敏感数据加密存储，查询用哈希列或索引列；展示统一脱敏。
- 建立数据字典：状态、类型、错误码、字典表统一维护，禁止散落在代码字符串里。
- 明确数据保留周期与归档策略，历史数据迁至归档表/库，避免主表无限膨胀。

## 9. DDL 评审清单

1. 表名、字段名符合命名规范，无保留字。
2. 字符集 `utf8mb4`、引擎 InnoDB、表与字段注释齐全。
3. 主键 `BIGINT`，自增/雪花策略与 Java 侧一致。
4. 审计字段、`deleted`、`version` 齐全（日志表按例外处理）。
5. 金额 `DECIMAL`、时间 `DATETIME(3)`、布尔 `TINYINT`，无 `FLOAT/DOUBLE`、`TIMESTAMP`、`ENUM`。
6. 必填字段 `NOT NULL` 且有合理默认值；无滥用 `NULL`。
7. 业务唯一键有唯一索引；逻辑删除复用场景已处理唯一键冲突。
8. 索引数量与列顺序合理，能覆盖核心查询；无冗余索引。
9. 无物理外键、触发器、存储过程。
10. 迁移脚本可重复评审、可回滚（或明确前向修复方案），大表变更评估过锁表风险。
11. 多表 SQL 的列都加了表别名；`SUM` 用 `IFNULL` 包裹；`NULL` 判断用 `IS [NOT] NULL` 或 `ISNULL()`。
12. 列表与分页 SQL 有稳定排序，排序字段能被索引覆盖；深分页用游标或延迟关联，无大 `OFFSET`。
13. `EXPLAIN` 的 `type` 达到 `range` 及以上，无 `ALL`、无 `type=index`、无大结果集 `Using filesort`。
14. 数据订正脚本执行前已用相同 `WHERE` 的 `SELECT` 核对影响行数。

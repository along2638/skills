# API 契约规范

> 本文件整合《Java 开发手册》嵩山版"前后端规约"14 条，并按本项目基线的 URL 风格（kebab-case、版本在路径）落地。与手册不一致处见 `alibaba-manual-map.md`。

## 1. URL 设计

- 统一前缀 `/api/v1`，版本写在 URL。
- 资源用**复数名词**、小写、kebab-case：`/orders`、`/order-items`、`/users`。手册要求多词用下划线、版本放 HTTP 头；本规范保留 kebab-case + 路径版本，理由见偏差表，团队内不得两种混用。
- 路径只表示资源，不出现动词；请求方法已表达动作语义。
- 路径不带表示内容类型的后缀（`.json`、`.xml`），用 `Accept` / `Content-Type` 表达。
- 路径参数只放资源标识：`/orders/{id}`。
- 非 CRUD 动作放子路径：`/orders/{id}/cancel`、`/orders/{id}/pay`。
- 禁止动词式基础路径：`/getOrder`、`/order/list`、`/saveOrder`。
- 禁止尾斜杠、文件扩展名、大小写混用。
- 查询条件、排序、分页用 query 参数，不塞进路径。
- 破坏性变更升 `/api/v2`；新增可选字段、增加枚举值不算破坏性变更（前端需容忍未知枚举）。

```text
POST   /api/v1/orders                 创建订单
GET    /api/v1/orders/{id}            查询详情
GET    /api/v1/orders                 分页查询
PUT    /api/v1/orders/{id}            全量更新
PATCH  /api/v1/orders/{id}            部分更新（如需要）
DELETE /api/v1/orders/{id}            逻辑删除
POST   /api/v1/orders/{id}/cancel     取消订单
```

## 2. 统一响应

### 2.1 ErrorCode

```java
package com.acme.mall.common.api;

import org.springframework.http.HttpStatus;

public interface ErrorCode {

    int getCode();

    String getMessage();

    HttpStatus getHttpStatus();
}
```

```java
package com.acme.mall.common.api;

import org.springframework.http.HttpStatus;

public enum CommonErrorCode implements ErrorCode {

    SUCCESS(0, "success", HttpStatus.OK),
    PARAM_INVALID(10001, "参数校验失败", HttpStatus.BAD_REQUEST),
    REQUEST_BODY_INVALID(10002, "请求体格式错误", HttpStatus.BAD_REQUEST),
    METHOD_NOT_ALLOWED(10004, "请求方法不支持", HttpStatus.METHOD_NOT_ALLOWED),
    DATA_CONFLICT(10006, "数据冲突，请勿重复提交", HttpStatus.CONFLICT),
    UNAUTHORIZED(20001, "未登录或登录已过期", HttpStatus.UNAUTHORIZED),
    FORBIDDEN(20002, "无权限访问", HttpStatus.FORBIDDEN),
    RESOURCE_NOT_FOUND(20003, "请求的资源不存在", HttpStatus.NOT_FOUND),
    TOO_MANY_REQUESTS(20005, "请求过于频繁", HttpStatus.TOO_MANY_REQUESTS),
    SYSTEM_ERROR(50000, "系统繁忙，请稍后重试", HttpStatus.INTERNAL_SERVER_ERROR);

    private final int code;
    private final String message;
    private final HttpStatus httpStatus;

    CommonErrorCode(int code, String message, HttpStatus httpStatus) {
        this.code = code;
        this.message = message;
        this.httpStatus = httpStatus;
    }

    @Override
    public int getCode() {
        return code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public HttpStatus getHttpStatus() {
        return httpStatus;
    }
}
```

### 2.2 Result

```java
package com.acme.mall.common.api;

import lombok.Getter;
import org.slf4j.MDC;

@Getter
public class Result<T> {

    private final int code;
    private final String message;
    private final T data;
    private final String traceId;

    private Result(int code, String message, T data, String traceId) {
        this.code = code;
        this.message = message;
        this.data = data;
        this.traceId = traceId;
    }

    public static <T> Result<T> success(T data) {
        return new Result<>(CommonErrorCode.SUCCESS.getCode(),
                CommonErrorCode.SUCCESS.getMessage(), data, MDC.get("traceId"));
    }

    public static Result<Void> success() {
        return success(null);
    }

    public static <T> Result<T> fail(ErrorCode errorCode) {
        return new Result<>(errorCode.getCode(), errorCode.getMessage(), null, MDC.get("traceId"));
    }

    public static <T> Result<T> fail(ErrorCode errorCode, T data) {
        return new Result<>(errorCode.getCode(), errorCode.getMessage(), data, MDC.get("traceId"));
    }

    public static <T> Result<T> fail(ErrorCode errorCode, String message, T data) {
        return new Result<>(errorCode.getCode(), message, data, MDC.get("traceId"));
    }
}
```

- `code = 0` 表示成功；失败时 `data` 一般为 `null`，参数校验失败时可放错误明细列表。
- `traceId` 由服务端生成，与日志中的 traceId 一致，便于排查。
- 禁止在 `message` 中返回异常类名、SQL、堆栈、内部主机信息。
- 禁止用 `Result` 包装文件流、SSE、大文件下载。

### 2.3 成功与失败示例

成功：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": "10001",
    "orderNo": "O1234567890123456789",
    "userId": "20001",
    "amount": "99.00",
    "status": "WAIT_PAY",
    "createTime": "2026-10-25 10:00:00",
    "updateTime": "2026-10-25 10:00:00"
  },
  "traceId": "9f2c1a7b4e6d"
}
```

业务失败（HTTP 409）：

```json
{
  "code": 30102,
  "message": "当前订单状态不允许该操作",
  "data": null,
  "traceId": "9f2c1a7b4e6d"
}
```

参数校验失败（HTTP 400）：

```json
{
  "code": 10001,
  "message": "参数校验失败",
  "data": [
    { "field": "amount", "message": "金额必须大于0" },
    { "field": "userId", "message": "用户ID不能为空" }
  ],
  "traceId": "9f2c1a7b4e6d"
}
```

### 2.4 错误响应的四类受众

服务端出错时，一次响应必须同时满足四类角色，缺一不可：

| 受众 | 需要的信息 | 本规范载体 |
|---|---|---|
| 浏览器 / 网关 | HTTP 状态码 | `ResponseEntity.status(errorCode.getHttpStatus())` |
| 前端开发 | 可判断、可比对的错误码 | `code`（即 errorCode，不要用 `message` 做判断） |
| 排查人员 | 技术原因 + 链路 ID | 日志中的 errorMessage + 响应里的 `traceId` |
| 终端用户 | 简短友好的提示 | `message` |

- `message` 面向用户：简短清晰、可指导下一步（"订单已支付，不能取消"），禁止出现异常类名、SQL、堆栈、主机名、参数名堆砌。
- 技术原因只写日志，不返回前端；前端需要区分场景时判 `code`，不要匹配 `message` 文案。
- `message` 可输出到前端隐藏控件或前端日志，用于线上问题定位。
- 对接要求 `errorCode` / `errorMessage` / `userTip` 三段式的外部契约时，在 Manager 适配层转换，不改变对内 `Result` 结构。

## 3. 业务错误码

编码规则：

| 区间 | 用途 |
|---|---|
| `0` | 成功 |
| `1xxxx` | 通用客户端/参数/请求格式 |
| `2xxxx` | 认证、权限、限流 |
| `3xxxx` | 业务错误：`3` + 两位模块号 + 两位序号 |
| `4xxxx` | 第三方依赖失败（支付、短信、OSS 等） |
| `5xxxx` | 系统内部错误 |

示例：

```java
package com.acme.mall.enums;

import com.acme.mall.common.api.ErrorCode;
import org.springframework.http.HttpStatus;

public enum OrderErrorCode implements ErrorCode {

    ORDER_NOT_FOUND(30101, "订单不存在", HttpStatus.NOT_FOUND),
    ORDER_STATUS_NOT_ALLOWED(30102, "当前订单状态不允许该操作", HttpStatus.CONFLICT),
    ORDER_NO_DUPLICATED(30103, "订单号已存在", HttpStatus.CONFLICT),
    ORDER_CONCURRENT_MODIFIED(30104, "订单已被修改，请刷新后重试", HttpStatus.CONFLICT),
    ORDER_AMOUNT_INVALID(30105, "订单金额不合法", HttpStatus.BAD_REQUEST);

    private final int code;
    private final String message;
    private final HttpStatus httpStatus;

    OrderErrorCode(int code, String message, HttpStatus httpStatus) {
        this.code = code;
        this.message = message;
        this.httpStatus = httpStatus;
    }

    @Override
    public int getCode() {
        return code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public HttpStatus getHttpStatus() {
        return httpStatus;
    }
}
```

规则：

- 错误码全局唯一，发布后不得复用或改变语义。
- 每个业务域固定一个模块号，新增错误码追加序号。
- 错误码集中登记在规范文档/配置中心，禁止在代码中写裸数字。
- 同类错误不要复制十几个码；用参数化 message 补充细节（不得包含敏感信息）。
- 错误码要回答"谁的错、错在哪"：能被快速比对（适合 `equals` / `switch`）、能定位来源、团队对同一错误有统一认知。
- 错误码不体现版本号与错误等级：版本靠追加兼容，等级由日志级别体现。
- 不随意新增错误码：先在错误码表里找语义相同或相近的复用；编号一经分配永久固定，不因业务或组织架构调整而改。
- 错误码不直接作为用户提示；用户看到的是 `message`（提示语），错误码只给前端与排查人员。
- 业务独有的细节由 `message` 承载，不让错误码本身承担过多业务属性。
- 调用第三方服务出错时，向上抛出可转义为本系统错误码（C 类转 B 类），并在 `message` 中带上第三方原始错误码。

### 3.1 与手册五位错误码的对应

手册建议使用 5 位字符串：`来源字母 + 4 位数字`，来源为 `A`（用户端）、`B`（本系统）、`C`（第三方），全正常用 `00000`。本规范使用数字区间码（便于与 HTTP 状态、模块号对应），两者对照如下：

| 手册分类 | 手册示例 | 本规范区间 |
|---|---|---|
| A 用户端错误（参数、登录、权限） | `A0001`、`A0201` | `1xxxx` / `2xxxx` |
| B 本系统错误（业务逻辑出错、程序健壮性差） | `B0001` | `3xxxx`，系统级用 `5xxxx` |
| C 第三方服务错误 | `C0001` | `4xxxx` |
| 全部正常 | `00000` | `0` |

- 三级宏观码可直接复用语义：`A0001` 用户端错误 → `10001`，`B0001` 系统执行出错 → `50000`，`C0001` 调用第三方出错 → `4xxxx`。
- 对接已使用五位码的外部系统时，在适配层做映射，不改内层 `ErrorCode`。
- 错误码后三位与 HTTP 状态码无关，不要用"看起来像状态码"的编号制造误解。

## 4. HTTP 状态码

| HTTP | 场景 | `code` 示例 |
|---|---|---|
| 200 | 成功 | 0 |
| 400 | 参数校验失败、请求体格式错误、业务入参非法 | 10001 / 10002 / 30105 |
| 401 | 未登录、token 过期或无效 | 20001 |
| 403 | 已登录但无权限 | 20002 |
| 404 | 路由不存在、业务资源不存在 | 20003 / 30101 |
| 405 | HTTP 方法不支持 | 10004 |
| 409 | 状态冲突、重复提交、乐观锁失败 | 30102 / 30103 / 30104 |
| 429 | 限流 | 20005 |
| 500 | 未预期系统异常 | 50000 |
| 502/503/504 | 网关或下游不可用 | 4xxxx |

- HTTP 状态码由 `ErrorCode.getHttpStatus()` 决定，不在业务代码中手写。
- 所有错误响应体结构一致，禁止 500 返回 Tomcat HTML 错误页。
- 认证失败用 401，权限不足用 403，不要统一返回 200。
- 业务可预期失败不要伪装成 500。

## 5. 请求与响应约定

### Header

| Header | 方向 | 规则 |
|---|---|---|
| `Authorization` | 请求 | `Bearer <token>` |
| `Content-Type` | 请求 | `application/json;charset=UTF-8` |
| `X-Request-Id` | 请求（可选） | 客户端链路 ID，`[A-Za-z0-9-]{8,64}`，非法则忽略 |
| `X-Trace-Id` | 响应 | 服务端最终 traceId，与日志一致 |
| `Idempotency-Key` | 请求 | 幂等键，`[A-Za-z0-9-]{8,64}`，用于创建/支付等操作 |

### JSON 字段

- 字段名 lowerCamelCase，以小写字母开头，与 Java 字段一致。反例：`ERROR_CODE`、`error_message`、`error-message`、`ErrorMessage`、`msg`。
- ID 字段（主键、外键、用户 ID）序列化为**字符串**，避免雪花 ID 在前端丢精度。
- 超过 2^53 的整数（订单号、交易号、流水号）一律用 `String` 返回，禁止用 `Long`；16 位以上单号在前端几乎必然被改写。
- 金额字段序列化为**字符串**，固定两位小数，如 `"99.00"`。
- 时间：`yyyy-MM-dd HH:mm:ss`；日期：`yyyy-MM-dd`；时区统一 `Asia/Shanghai`（手册建议 GMT，本项目前后端同处东八区，统一东八区并在文档标明）。
- 枚举：返回 code 字符串，如 `"WAIT_PAY"`；前端不得依赖 `desc` 做逻辑。
- 布尔：`true` / `false`，不使用 0/1。
- 列表为空返回 `[]`，对象为空返回 `{}`，不返回 `null`；前端因此不需要到处判空。
- 默认返回所有字段（包括 `null`），保持契约稳定；不要全局开启 `NON_NULL`。
- 请求忽略未知字段，新增字段不导致老客户端失败。
- 不使用 `Map<String, Object>` 作为响应体；无结构约束的 JSON 会让前后端契约失控。

### 分页

请求：

```text
GET /api/v1/orders?pageNum=1&pageSize=10&status=WAIT_PAY&startTime=2026-10-01 00:00:00
```

```java
package com.acme.mall.common.api;

import lombok.Getter;
import lombok.Setter;

import javax.validation.constraints.Max;
import javax.validation.constraints.Min;

@Getter
@Setter
public class PageReq {

    public static final int MAX_PAGE_SIZE = 100;

    @Min(value = 1, message = "页码不能小于1")
    private long pageNum = 1;

    @Min(value = 1, message = "每页条数不能小于1")
    @Max(value = MAX_PAGE_SIZE, message = "每页最多100条")
    private long pageSize = 10;
}
```

响应：

```java
package com.acme.mall.common.api;

import lombok.Getter;

import java.util.List;

@Getter
public class PageResp<T> {

    private final List<T> records;
    private final long total;
    private final long pageNum;
    private final long pageSize;
    private final long pages;

    private PageResp(List<T> records, long total, long pageNum, long pageSize) {
        this.records = records;
        this.total = total;
        this.pageNum = pageNum;
        this.pageSize = pageSize;
        this.pages = pageSize <= 0 ? 0 : (total + pageSize - 1) / pageSize;
    }

    public static <T> PageResp<T> of(List<T> records, long total, long pageNum, long pageSize) {
        return new PageResp<>(records, total, pageNum, pageSize);
    }
}
```

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "records": [],
    "total": 0,
    "pageNum": 1,
    "pageSize": 10,
    "pages": 0
  },
  "traceId": "9f2c1a7b4e6d"
}
```

- 页码从 1 开始，默认 10 条，最大 100 条。
- `pageNum < 1` 时按 1 处理；`pageNum` 超过总页数时返回最后一页（手册约定），不要返回空列表让前端卡在无效页。
- 排序参数固定为 `sortBy`（后端白名单内的字段名）与 `sortOrder`（`asc` / `desc`），两者成对出现；缺省时按业务默认排序（如 `create_time DESC, id DESC`）。禁止接受前端传来的任意字段名，也禁止用 `orderBy=xxx` 这类拼接式参数。
- 分页响应的列表字段名固定为 `records`，与前端约定一致；不得改成 `list`、`rows`、`items`。同时返回 `pages` 总页数，避免前端自行计算。
- 排序字段必须白名单校验，禁止把前端字段直接拼进 SQL。
- 默认排序必须稳定（如 `create_time DESC, id DESC`），避免翻页重复/丢失。
- 深分页导出使用游标接口，不使用大 `OFFSET`。
- URL 传递的参数总量不超过 2048 字节（浏览器最小值）。批量 ID 不塞进 URL，改用 `POST` + body，否则超长会被截断导致漏处理。

## 6. 参数校验

- Body：`@Valid @RequestBody`。
- Controller 入参使用 DTO（`XxxReq`），出参使用 VO（`XxxResp`）；不得直接接收或返回 Entity（PO）。
- Query：`@Valid` 修饰 Query 对象；Controller 类加 `@Validated`。
- Path：`@PathVariable @Positive Long id`。
- 新增/更新使用不同 Req 与校验分组；更新校验 `id` 必填。
- 嵌套对象加 `@Valid`；集合元素使用 `List<@Valid ItemReq>`。
- 校验失败由全局异常处理器统一转为 400 + `10001`，返回字段与中文提示。
- 业务规则（状态是否允许、库存是否足够、权限归属）在 Service 校验，并抛业务错误码。
- 禁止在 Controller 手写 `if (req.getXxx() == null)` 代替注解校验。

## 7. 幂等与并发

- 创建类接口（下单、支付、退款）必须支持幂等键。
- 幂等键存 Redis：key = `idem:{userId}:{api}:{key}`，value 为请求指纹 + 结果，TTL 24 小时。
- 相同 key、不同请求体 → 返回 409；相同 key、相同请求体 → 返回首次结果。
- 数据库唯一索引是最后一道防线；捕获 `DuplicateKeyException` 转为业务错误。
- 乐观锁更新检查影响行数，失败返回 409；禁止静默覆盖。
- 前端按钮置灰、防抖不能作为幂等方案。

## 8. 版本与兼容

- URL 主版本 `/api/v1`；破坏性变更新建 `/api/v2`，旧版本保留至少一个发布周期。
- 兼容变更：新增可选字段、新增枚举值、新增接口、放宽校验。
- 不兼容变更：删除/重命名字段、修改字段类型或语义、收紧校验、改变默认分页。
- 废弃字段先在文档标注 `deprecated`，保留一个周期后再删除。
- 接口文档使用 Knife4j/OpenAPI 3 注解（`@Operation`、`@Schema`），示例值必须是脱敏数据。
- 生产环境设置 `knife4j.production=true`，关闭 `/doc.html` 与 OpenAPI 文档端点。

## 9. 文件上传与下载

- 上传使用 `multipart/form-data`，限制大小（如 10MB）与类型白名单。
- 文件名不参与存储路径；使用服务端生成的对象 key。
- 返回文件 ID/URL，不返回服务器绝对路径。
- 下载接口单独返回流，不包 `Result`；设置 `Content-Disposition` 与正确的 `Content-Type`。
- 禁止根据用户输入直接拼接文件路径，防止路径穿越。

## 10. 传输与部署约束

- 生产环境必须 HTTPS；禁止 HTTP 明文与 HTTP→HTTPS 的静默回退。
- 请求体长度必须受控：Nginx 默认限制 1MB、Tomcat 默认 2MB，超限后端解析失败。确需传大内容时显式调大阈值并评估内存风险；大文件走分片上传或对象存储直传。
- 请求必须显式声明 `Content-Type`：JSON 用 `application/json;charset=UTF-8`，上传用 `multipart/form-data`；不靠服务端猜。
- 服务器内部跳转用 `forward`；外部重定向地址由统一 URL 代理模块生成，禁止硬编码域名，否则 HTTPS 站点会被浏览器提示"不安全"，并造成多环境地址不一致。
- 响应必须显式声明可缓存性：敏感数据 `Cache-Control: no-store`；可缓存数据给出 `s-maxage`/`max-age`，避免中间层缓存错误内容。
- 默认使用 JSON 作为交换格式，不用 XML；`Accept` 只支持约定内的类型。
- 响应体不返回服务器绝对路径、内网 IP、内部端口。

## 11. 前后端契约对齐

前端规范技能 `frontend-standards` 的 `references/api-contract.md` 是本文件的消费侧实现。下表任一契约项变更，前后端必须同批发布，不能只改一侧。

| 契约项 | 本规范定义 | 前端承接方式 | 不一致的后果 |
|---|---|---|---|
| 路径 | `/api/v1/{复数资源}` | `baseURL=/api/v1` + 资源路径 | 路径改名后前端 404 |
| 响应包 | `code` / `message` / `data` / `traceId` | 拦截器解包 `data`，判 `code === 0` | 字段改名后前端全部取到 `undefined` |
| 分页响应 | `records` / `total` / `pageNum` / `pageSize` / `pages` | 同名取值 | 改成 `list` 会导致列表静默空白 |
| 分页请求 | `pageNum` / `pageSize` / `sortBy` / `sortOrder` | 同参数传递 | 参数名不符则后端走默认值 |
| ID | 字符串 | `string`，不转数字 | 返回数字则超 2^53 精度丢失 |
| 金额 | 字符串、两位小数 | 仅格式化展示 | 返回数字则出现浮点误差 |
| 时间 | `yyyy-MM-dd HH:mm:ss`，东八区 | 显式指定格式解析 | 格式或时区不一致导致展示错乱 |
| 枚举 | code 字符串（如 `WAIT_PAY`） | 字面量联合 + 未知值兜底 | 新增枚举值时前端渲染空白 |
| 校验失败 | 400 + `10001` + `data:[{field,message}]` | 按 `field` 回填表单 | 结构变更后无法定位到字段 |
| 认证失败 | 401 + `20001` | 清 token 跳登录（防重入） | 用 200 包错误码则前端不跳转 |
| 幂等 | 请求头 `Idempotency-Key` | 创建类请求携带并复用 | 缺失则重复下单 |
| 下载 | 裸流 + `Content-Disposition` | `responseType: 'blob'` 处理 | 包 `Result` 则文件损坏 |

规则：

- **破坏性变更必须升版本**：改字段名、改类型、删字段、改分页字段名，都走 `/api/v2`，不允许"前端跟着改一下"的隐式破坏。
- **兼容变更要提前打招呼**：新增字段、新增枚举值属于兼容变更，但前端需要有兜底逻辑，后端应在发布说明中列出新增枚举值。
- 联调前双方按上表逐项确认；出现"接口有数据但页面空白"时，优先查分页字段名与响应解包层级。
- 变更契约时同步修改三处：本文件、前端 `references/api-contract.md`、OpenAPI 注解与示例数据。

## 12. 接口评审清单

1. 路径符合 `/api/v1/{resources}`，资源名复数、kebab-case。
2. 请求/响应字段命名、类型、时间格式、金额格式一致。
3. 成功与失败都使用统一 `Result` 结构。
4. 错误码已登记，HTTP 状态与错误类型匹配。
5. 入参校验完整，错误提示可读且不泄露内部信息。
6. 分页参数有上限，排序字段白名单。
7. 创建/支付类接口具备幂等方案。
8. 出参不含 Entity、密码、密钥、完整敏感信息。
9. 接口文档与实际实现一致，示例数据已脱敏。
10. 破坏性变更已升版本并通知调用方。
11. 列表为空返回 `[]`、对象为空返回 `{}`，前端无需到处判 `null`。
12. 超大整数（订单号、流水号）以字符串返回，未被 JS 精度截断。
13. URL 参数未超 2048 字节，批量操作走 body；请求体大小在 Nginx/Tomcat 限制内。
14. 错误响应同时具备 HTTP 状态码、错误码、`traceId` 与用户可读提示。
15. 分页响应字段为 `records` / `total` / `pageNum` / `pageSize` / `pages`，与前端约定一致。
16. 契约变更（字段、类型、分页字段名）已升版本，并同步前端规范与 OpenAPI 文档。

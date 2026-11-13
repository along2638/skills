# 异常与日志规范

## 1. 异常体系

### 1.1 BizException

```java
package com.acme.mall.common.exception;

import com.acme.mall.common.api.ErrorCode;

public class BizException extends RuntimeException {

    private static final long serialVersionUID = 1L;

    private final ErrorCode errorCode;

    public BizException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BizException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public BizException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

### 1.2 使用规则

- 业务失败只抛 `BizException`，不为每个场景定义新异常类。
- 异常必须携带 `ErrorCode`，错误码决定 HTTP 状态与响应 code。
- 需要补充上下文时使用 `new BizException(code, "具体原因")`，但不得包含 SQL、密钥、堆栈、内部地址。
- 包装第三方异常必须保留 cause：`new BizException(code, "支付网关调用失败", e)`。
- 禁止用异常做正常分支控制；可空查找用 `findXxx` 返回 `Optional`，不要用异常判断"没查到"。
- 可预检查的运行时异常不得用 `catch` 兜：`NullPointerException`、`IndexOutOfBoundsException`、`IllegalArgumentException` 一律先用 `if` 判断（`NumberFormatException` 这类无法预检查的除外）。
- 禁止空 catch、`catch (Exception e) {}`、捕获后仅打印日志不处理；不想处理就往上抛。
- 区分稳定代码与非稳定代码：只对可能出错的语句做 `try`，并尽量按异常类型分别处理，不要一大段代码包一个 `catch`。
- 捕获类型与抛出类型必须匹配，或捕获其父类。
- 调用 RPC、二方包、反射/动态代理方法时必须捕获 `Throwable`（可能出现 `NoSuchMethodError`、`NoClassDefFoundError` 等 `Error`，而非 `Exception`）。
- `finally` 中必须关闭资源、流；有异常也要在 `finally` 内部处理。JDK 8 优先 try-with-resources。
- 禁止在 `finally` 中 `return`（会丢弃 `try` 中的返回点，异常也会被吞掉）。
- 事务方法内 `catch` 住异常又不想回滚时，必须显式说明；需要回滚时用 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` 或继续抛出，不要静默吞掉。
- 业务层不声明 checked exception；第三方 checked exception 在 Manager 边界转换为 `BizException` 或系统异常。
- 禁止直接 `throw new RuntimeException()`，更禁止抛 `Exception`/`Throwable`；用有业务含义的 `BizException`。
- 方法可以返回 `null`，但必须在 Javadoc 说明什么情况下返回 `null`；对前端的列表接口返回空集合。
- 异步任务、定时任务、MQ 消费必须捕获异常并记录，禁止让线程静默退出。

### 1.3 BizAssert

```java
package com.acme.mall.common.exception;

import com.acme.mall.common.api.ErrorCode;

public final class BizAssert {

    private BizAssert() {
    }

    public static void isTrue(boolean expression, ErrorCode errorCode) {
        if (!expression) {
            throw new BizException(errorCode);
        }
    }

    public static <T> T notNull(T value, ErrorCode errorCode) {
        if (value == null) {
            throw new BizException(errorCode);
        }
        return value;
    }
}
```

用于 Service 内部的前置条件，避免大量重复 `if (...) throw`。

## 2. 全局异常处理

```java
package com.acme.mall.common.exception;

import com.acme.mall.common.api.CommonErrorCode;
import com.acme.mall.common.api.ErrorCode;
import com.acme.mall.common.api.Result;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.validation.BindException;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
import org.springframework.web.servlet.NoHandlerFoundException;

import javax.servlet.http.HttpServletRequest;
import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public ResponseEntity<Result<Void>> handleBiz(BizException ex, HttpServletRequest request) {
        ErrorCode errorCode = ex.getErrorCode();
        if (errorCode.getHttpStatus().is5xxServerError()) {
            log.error("biz error, code={}, path={}", errorCode.getCode(),
                    request.getRequestURI(), ex);
        } else {
            log.warn("biz reject, code={}, path={}, message={}",
                    errorCode.getCode(), request.getRequestURI(), ex.getMessage());
        }
        return build(errorCode, ex.getMessage(), null);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Result<List<ValidationError>>> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        return buildValidation(ex.getBindingResult(), request);
    }

    @ExceptionHandler(BindException.class)
    public ResponseEntity<Result<List<ValidationError>>> handleBind(
            BindException ex, HttpServletRequest request) {
        return buildValidation(ex.getBindingResult(), request);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<Result<List<ValidationError>>> handleConstraintViolation(
            ConstraintViolationException ex, HttpServletRequest request) {
        List<ValidationError> errors = ex.getConstraintViolations().stream()
                .map(this::toError)
                .collect(Collectors.toList());
        log.warn("param invalid, path={}, fields={}", request.getRequestURI(),
                errors.stream().map(ValidationError::getField).collect(Collectors.toList()));
        return build(CommonErrorCode.PARAM_INVALID, errors);
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<Result<Void>> handleUnreadable(
            HttpMessageNotReadableException ex, HttpServletRequest request) {
        log.warn("request body unreadable, path={}, type={}",
                request.getRequestURI(), ex.getClass().getSimpleName());
        return build(CommonErrorCode.REQUEST_BODY_INVALID, null);
    }

    @ExceptionHandler({MissingServletRequestParameterException.class,
            MethodArgumentTypeMismatchException.class})
    public ResponseEntity<Result<Void>> handleBadParameter(
            Exception ex, HttpServletRequest request) {
        log.warn("bad request parameter, path={}, type={}",
                request.getRequestURI(), ex.getClass().getSimpleName());
        return build(CommonErrorCode.PARAM_INVALID, null);
    }

    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<Result<Void>> handleMethodNotSupported(
            HttpRequestMethodNotSupportedException ex, HttpServletRequest request) {
        log.warn("method not supported, path={}, method={}",
                request.getRequestURI(), ex.getMethod());
        return build(CommonErrorCode.METHOD_NOT_ALLOWED, null);
    }

    @ExceptionHandler(NoHandlerFoundException.class)
    public ResponseEntity<Result<Void>> handleNoHandler(
            NoHandlerFoundException ex, HttpServletRequest request) {
        log.warn("no handler found, path={}", request.getRequestURI());
        return build(CommonErrorCode.RESOURCE_NOT_FOUND, null);
    }

    @ExceptionHandler(DuplicateKeyException.class)
    public ResponseEntity<Result<Void>> handleDuplicateKey(
            DuplicateKeyException ex, HttpServletRequest request) {
        log.warn("duplicate key, path={}, type={}",
                request.getRequestURI(), ex.getClass().getSimpleName());
        return build(CommonErrorCode.DATA_CONFLICT, null);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Result<Void>> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("unexpected error, path={}", request.getRequestURI(), ex);
        return build(CommonErrorCode.SYSTEM_ERROR, null);
    }

    private ResponseEntity<Result<List<ValidationError>>> buildValidation(
            BindingResult bindingResult, HttpServletRequest request) {
        List<ValidationError> errors = bindingResult.getFieldErrors().stream()
                .map(this::toError)
                .collect(Collectors.toList());
        log.warn("param invalid, path={}, fields={}", request.getRequestURI(),
                errors.stream().map(ValidationError::getField).collect(Collectors.toList()));
        return build(CommonErrorCode.PARAM_INVALID, errors);
    }

    private ValidationError toError(FieldError fieldError) {
        return new ValidationError(fieldError.getField(), fieldError.getDefaultMessage());
    }

    private ValidationError toError(ConstraintViolation<?> violation) {
        return new ValidationError(violation.getPropertyPath().toString(), violation.getMessage());
    }

    private <T> ResponseEntity<Result<T>> build(ErrorCode errorCode, T data) {
        return ResponseEntity.status(errorCode.getHttpStatus())
                .body(Result.fail(errorCode, data));
    }

    private <T> ResponseEntity<Result<T>> build(ErrorCode errorCode, String message, T data) {
        return ResponseEntity.status(errorCode.getHttpStatus())
                .body(Result.fail(errorCode, message, data));
    }

    @Getter
    @AllArgsConstructor
    public static class ValidationError {

        private final String field;
        private final String message;
    }
}
```

规则：

- `BizException` 按错误码的 HTTP 状态返回；4xx 用 WARN，5xx 用 ERROR。
- 参数校验统一返回 400 + `10001`，`data` 为字段错误列表。
- 未预期异常返回 500 + `50000`，响应中不得出现异常类名、堆栈、SQL。
- `NoHandlerFoundException` 需要开启：

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
```

开启后需评估静态资源映射；若保留 Knife4j 文档页 `/doc.html`，不要同时关闭 `spring.web.resources.add-mappings`，或仅在非生产环境开启该配置。

- Spring Security 过滤器链中的认证/授权异常不会进入 `@RestControllerAdvice`，必须由 `AuthenticationEntryPoint` 与 `AccessDeniedHandler` 输出统一 `Result`，见 `security.md`。
- 异步线程、MQ 消费者中的异常不会被全局处理器捕获，必须自行捕获并记录。

## 3. 日志级别

| 级别 | 使用场景 | 示例 |
|---|---|---|
| ERROR | 未预期异常、数据不一致、下游故障影响主流程、需要告警 | 数据库连接失败、支付回调处理失败 |
| WARN | 可预期业务拒绝、重试、降级、慢调用、参数错误 | 订单状态不允许取消、第三方超时后重试 |
| INFO | 关键状态变更、启动/关闭、外部调用结果、审计事件 | 订单创建成功、用户登录成功 |
| DEBUG | 开发排查细节、分支条件、耗时 | SQL 参数、缓存命中情况；生产默认关闭 |
| TRACE | 基本不用 | — |

- 生产应用包默认 `INFO`，root 可设为 `WARN`；需要排查时按包临时开到 `DEBUG`，排查后恢复。
- 预期业务错误不要打 ERROR，避免告警风暴。
- 同一异常只记录一次；不要在每层重复打印。

## 4. 日志写法

```java
log.info("order created, orderId={}, orderNo={}, userId={}",
        entity.getId(), entity.getOrderNo(), entity.getUserId());

log.warn("cancel rejected, orderId={}, status={}", id, entity.getStatus());

try {
    paymentClient.pay(request);
} catch (PaymentException e) {
    log.error("payment call failed, orderId={}, channel={}", id, channel, e);
    throw new BizException(PaymentErrorCode.PAY_FAILED, "支付调用失败", e);
}
```

- 只用 SLF4J：`private static final Logger` 或 Lombok `@Slf4j`；禁止直接依赖 Log4j/Logback 的 API。
- 使用 `{}` 占位符；异常对象放最后一个参数，不要 `e.getMessage()` 代替。
- 禁止字符串拼接、`System.out.println`、`printStackTrace()`。
- 参数含方法调用或拼接时，必须加级别开关判断，避免无谓开销：

```java
if (log.isDebugEnabled()) {
    log.debug("current id={}, name={}", id, getName());
}
```

- 打印异常必须同时给现场与堆栈：`log.error("inputParams={}, errorMessage={}", params, e.getMessage(), e);` 只打 message 会丢掉调用链。
- 大量对象或集合只记录 ID、数量、摘要；禁止打印完整请求体、响应体、大 JSON。
- 禁止直接 `JSON.toJSONString(obj)` 打日志：getter 可能抛异常影响主流程，且会带出敏感字段；只打业务字段或 `toString()`。
- 循环内禁止逐条 INFO；改为汇总记录。
- 记录业务标识（订单号、用户 ID、任务 ID）和耗时，便于串联链路。
- 外部调用记录：目标系统、操作、耗时、结果码；不记录密钥与完整报文。
- 日志配置文件必须设置 `additivity="false"`，避免同一日志被父子 appender 重复输出。
- 刚上线观察业务行为时可用 WARN 临时记录，但必须限时限量并在观察期结束后删除。

## 5. 敏感信息与日志安全

禁止记录：

- 密码、密码哈希、验证码、token、refresh token、cookie、session。
- 完整手机号、身份证号、银行卡号、邮箱（按合规要求脱敏）。
- 密钥、AK/SK、证书、数据库连接串。
- 完整支付报文、完整用户资料、完整地址。
- 用户输入未清洗的换行符（防日志伪造/注入）。

脱敏示例：

```java
private String maskPhone(String phone) {
    if (StringUtils.isBlank(phone) || phone.length() < 7) {
        return "***";
    }
    return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
}
```

- 用户输入写日志前去掉 `\r`、`\n`，限制长度。
- 审计日志中的 before/after 数据必须脱敏。
- 日志文件权限最小化，禁止公网可读；日志平台访问需授权。

## 6. traceId

```java
package com.acme.mall.common.web;

import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.UUID;
import java.util.regex.Pattern;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TraceIdFilter extends OncePerRequestFilter {

    public static final String TRACE_ID = "traceId";
    private static final Pattern VALID_TRACE_ID = Pattern.compile("[A-Za-z0-9-]{8,64}");

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String traceId = request.getHeader("X-Request-Id");
        if (traceId == null || !VALID_TRACE_ID.matcher(traceId).matches()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }
        MDC.put(TRACE_ID, traceId);
        response.setHeader("X-Trace-Id", traceId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(TRACE_ID);
        }
    }
}
```

- 客户端传入的 `X-Request-Id` 必须校验格式与长度，防止日志注入。
- 响应头 `X-Trace-Id` 与日志中的 traceId 一致。
- 异步线程必须传递 MDC：

```java
public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            if (context != null) {
                MDC.setContextMap(context);
            } else {
                MDC.clear();
            }
            try {
                runnable.run();
            } finally {
                if (previous != null) {
                    MDC.setContextMap(previous);
                } else {
                    MDC.clear();
                }
            }
        };
    }
}
```

- 线程池注册 `setTaskDecorator(new MdcTaskDecorator())`。
- Feign / RestTemplate / MQ 需要把 traceId 放入请求头或消息属性，消费端恢复 MDC。

## 7. logback-spring.xml 模板

日志文件约定：

- 当天日志：`{应用名}.log`；历史日志：`{应用名}.log.{yyyy-MM-dd}`（滚动后自动加 `.i` 与 `.gz`）。
- 扩展日志（打点、监控、访问日志）：`{appName}_{logType}_{logName}.log`，`logType` 取 `stats`/`monitor`/`access`，如 `mall_monitor_timeZoneConvert.log`。文件名即可看出应用、类型、用途。
- 错误日志与业务日志分开存放；访问日志、监控打点各用独立文件，便于监控采集。
- 保留期：所有日志至少 15 天（部分异常以"周"为频次复现）；涉及网络运行状态、网络安全事件、个人敏感信息操作的日志，留存不少于 6 个月并多机备份（合规要求）。
- 目录：生产放 `/home/admin/{app}/logs/` 或统一日志盘，禁止写在发布目录内，禁止公网可读。
- 滚动策略：按天 + 按大小（如 100MB），必须设置 `maxHistory` 与 `totalSizeCap`，防止打满磁盘。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>

    <property name="LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId:-}] %logger{36} - %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <springProfile name="prod">
        <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/${APP_NAME}-json.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>logs/${APP_NAME}-json.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>100MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>10GB</totalSizeCap>
            </rollingPolicy>
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

    <springProfile name="!prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

- JSON 日志需要 `logstash-logback-encoder` 依赖；不引入时删除 JSON appender。
- 单独声明 `<logger>` 时必须 `additivity="false"`，否则会同时被 root 的 appender 再输出一遍：

```xml
<logger name="com.acme.mall" level="INFO" additivity="false">
    <appender-ref ref="FILE"/>
</logger>
```
- 高吞吐服务给文件 appender 包 `AsyncAppender`，并设置队列容量与丢弃策略。
- 日志目录、保留天数、总量上限按部署环境调整，避免磁盘打满。

## 8. 审计日志

以下操作必须记录审计日志（数据库表或专用日志平台）：

- 登录、登出、密码修改、权限/角色变更。
- 订单状态变更、支付、退款、取消。
- 数据导出、批量导入、删除。
- 配置变更、开关切换、密钥轮换。

审计字段至少包含：操作人 ID、操作类型、资源类型与 ID、变更前后摘要（脱敏）、结果、IP、User-Agent、traceId、操作时间。

- 审计日志只追加，不修改、不删除。
- 保留周期按合规要求；到期归档而非直接丢弃。
- 审计写入失败不得影响主流程，但必须告警。

## 9. 告警建议

- 5xx 比例、ERROR 日志速率、未捕获异常数量。
- 下游调用失败率、超时率、重试次数。
- 慢 SQL（> 1s）、连接池等待、线程池队列积压。
- 业务异常突增（如支付失败、下单失败）。
- 告警信息必须携带 traceId、服务名、环境、时间窗口。

## 10. 检查清单

1. 业务异常是否只使用 `BizException` + 已登记 `ErrorCode`。
2. 全局处理器是否覆盖参数校验、请求体解析、404/405、数据库唯一键、兜底异常。
3. 响应体是否统一 `Result`，是否泄露堆栈或 SQL。
4. 日志是否使用占位符，异常对象是否作为最后参数。
5. 是否记录了业务标识与 traceId，是否避免重复记录同一异常。
6. 是否记录密码、token、完整手机号/身份证等敏感信息。
7. 异步线程、定时任务、MQ 消费是否传递 traceId 并捕获异常。
8. 生产日志是否为 JSON、是否配置滚动与保留上限。
9. 关键操作是否有审计日志。
10. 5xx、慢 SQL、下游失败是否有告警。
11. 日志保留期是否 ≥ 15 天；敏感操作日志是否 ≥ 6 个月并多机备份。
12. 扩展日志是否按 `{appName}_{logType}_{logName}.log` 命名，错误与业务日志是否分文件。
13. 是否存在用 `catch` 处理可预检查异常（NPE、越界）、`finally` 中 `return`、`catch` 后静默吞掉。
14. RPC、二方包、反射调用是否捕获 `Throwable`；异常是否带现场信息与堆栈。

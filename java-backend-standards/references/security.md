# 安全规范

## 1. 认证

基线使用 **Spring Security 5.7 + JWT**，无状态、无 Session。

- Access Token 有效期 2 小时；Refresh Token 有效期 7 天。
- Access Token 只放 `userId`、`jti`、`iat`、`exp`、`iss`，禁止放密码、手机号、权限全集等敏感或易膨胀数据。
- 签名密钥来自环境变量，长度 ≥ 32 字节（HS256），不同环境使用不同密钥，定期轮换。
- Refresh Token 使用随机不透明字符串，数据库/Redis 只存哈希；刷新时轮换，检测到重复使用立即吊销该用户全部令牌。
- 登出将 `jti` 加入 Redis 黑名单，TTL 至 Token 过期；禁止只在前端删除 Token 就算登出。
- 登录失败返回统一提示（"用户名或密码错误"），不得暴露账号是否存在。
- 登录、短信、支付等接口必须限流；连续失败触发验证码或临时锁定。
- 密码使用 `BCryptPasswordEncoder`（强度 10～12）；禁止 MD5/SHA1/明文/可逆加密。
- 密码策略：长度 8～64，禁止弱口令；修改密码后吊销已有 Token。
- 若使用 Cookie/Session，必须 `HttpOnly`、`Secure`、`SameSite=Lax/Strict`，并开启 CSRF 防护。

## 2. 授权

- 权限码格式 `{domain}:{action}`，如 `order:create`、`order:cancel`、`order:view`。
- 使用 `@PreAuthorize("hasAuthority('order:cancel')")` 做接口级授权；角色只是权限集合，不要在 Service 里硬编码角色名。
- 默认拒绝：未明确放行的接口一律需要认证。
- 后台接口使用独立前缀 `/api/v1/admin/**`，与用户端接口分离鉴权策略。
- 水平越权（IDOR）：查询/修改单条资源时，必须同时按 `id + userId/tenantId` 过滤，不能只 `selectById` 后直接返回。
- 请求中的 `userId`、`tenantId`、`role` 一律不可信；当前用户与租户从 Token 上下文获取，服务端覆盖请求值。
- 数据导出、批量删除、权限变更等敏感操作需要更高权限并写审计日志。
- 禁止直接把 Entity 作为请求体绑定，防止批量赋值（mass assignment）篡改非预期字段。

## 3. SQL 注入

- MyBatis 参数一律 `#{}`，禁止 `${}`。
- 动态表名、列名、排序字段只能来自服务端白名单，不能直接使用用户输入。
- MyBatis-Plus Wrapper 的列引用使用 `Entity::getField`，不要把前端字符串传入 `orderBy`。
- 数据库账号最小权限：应用账号只有业务库 DML 权限，无 DDL、无跨库权限；迁移使用独立账号。
- 禁止字符串拼接 SQL；禁止在异常信息中返回 SQL。
- 报表/导出等复杂 SQL 必须经过代码评审与 `EXPLAIN`。

排序白名单示例：

```java
private static final Map<String, SFunction<OrderEntity, ?>> SORT_FIELDS;

static {
    Map<String, SFunction<OrderEntity, ?>> map = new HashMap<>();
    map.put("createTime", OrderEntity::getCreateTime);
    map.put("amount", OrderEntity::getAmount);
    SORT_FIELDS = Collections.unmodifiableMap(map);
}

public SFunction<OrderEntity, ?> resolveSort(String field) {
    SFunction<OrderEntity, ?> column = SORT_FIELDS.get(field);
    if (column == null) {
        throw new BizException(CommonErrorCode.PARAM_INVALID);
    }
    return column;
}
```

## 4. XSS、CSRF、CORS

### XSS

- API 返回 JSON，前端负责转义；响应头 `Content-Type: application/json;charset=UTF-8`。
- 禁止把用户输入直接拼进 HTML、错误页、邮件模板。
- 富文本必须服务端白名单清洗（如 OWASP Java HTML Sanitizer），禁止只靠前端过滤。
- 文件上传禁止 SVG、HTML 等可执行内容；图片重新编码后再存储。
- 设置 `X-Content-Type-Options: nosniff`；Web 页面设置 CSP。

### CSRF

- 纯 Token（Authorization Header）+ 无状态 API：可关闭 CSRF。
- 使用 Cookie 承载会话或 Token：必须开启 CSRF，并使用 `SameSite`。
- 不要因为"是前后端分离"就默认关闭 CSRF，先确认凭证如何传输。

### CORS

- 白名单精确到协议、域名、端口；禁止生产使用 `*`。
- `allowCredentials=true` 时禁止 `allowedOrigins="*"`。
- 只暴露必要响应头（如 `X-Trace-Id`），只允许必要请求头。
- 预检缓存时间合理设置，避免每次请求 OPTIONS。

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(Collections.singletonList("https://admin.example.com"));
    config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(Arrays.asList(
            "Authorization", "Content-Type", "X-Request-Id", "Idempotency-Key"));
    config.setExposedHeaders(Collections.singletonList("X-Trace-Id"));
    config.setAllowCredentials(false);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

## 5. 文件上传与下载

- 限制文件大小（如 10MB）与数量，超出直接拒绝。
- 同时校验扩展名与文件魔数（MIME），不能只看 `Content-Type`。
- 白名单：`jpg/jpeg/png/pdf/xlsx/docx` 等业务必需类型；禁止 `jsp/jspx/php/sh/html/svg`。
- 文件名由服务端生成（UUID/雪花），原始文件名只存元数据；禁止把用户文件名拼入路径。
- 存储目录不在 Web 根目录下，不可执行；对象存储使用私有 Bucket + 临时签名 URL。
- 下载必须鉴权，禁止根据用户输入直接拼路径，防止路径穿越。
- 压缩包解压校验 entry 路径，防止 Zip Slip。
- 图片处理防止 decompression bomb（像素上限）。

## 6. 外部请求与反序列化

### SSRF

- 禁止直接请求用户提供的任意 URL。
- 必须请求外部地址时使用域名白名单，解析后校验 IP，拒绝私网、回环、链路本地、元数据地址。
- 禁止跟随跳转到内网；限制重定向次数。
- HTTP 客户端设置连接/读取超时，TLS 校验证书，禁止 trust-all。

### 反序列化

- 只使用 Jackson，禁止开启 `enableDefaultTyping`。
- 禁止 Fastjson 1.x autotype；如存量项目必须使用，关闭 autotype 并升级到安全版本。
- 禁止 Java 原生序列化跨进程传输。
- XML 解析禁用外部实体（XXE），禁用 DTD。

## 7. 密钥与配置

- 密钥、数据库密码、AK/SK、JWT Secret 只能来自环境变量、密钥管理服务或配置中心。
- 禁止写入代码、`application.yml`、Git、镜像、日志、前端包。
- 不同环境使用不同密钥；上线前检查默认密钥是否被替换。
- 敏感配置变更走审批与审计；密钥定期轮换并支持双密钥过渡。
- 数据库敏感字段（手机号、身份证、银行卡）使用 AES-GCM 加密，密钥放 KMS；需要等值查询时额外存 HMAC-SHA256 哈希列。
- 生产关闭 Knife4j/OpenAPI：设置 `knife4j.production=true`，并在网关/安全层拒绝 `/doc.html`、`/v3/api-docs/**`；Actuator 只暴露 `health`、`info`，管理端口不对外。
- 禁止在错误响应中返回配置、堆栈、SQL、内部主机名。
- 依赖安全：定期执行 OWASP Dependency-Check / `mvn dependency:tree`，禁止已知高危 CVE 的版本上线。
- 全站 HTTPS；反向代理正确传递 `X-Forwarded-Proto`，后端校验可信代理。

## 8. 限流与防刷

- 登录、注册、短信、支付、下单、查询详情等接口必须限流。
- 限流维度：用户 ID、IP、设备、接口；不要只按 IP（NAT/代理会误伤）。
- 使用 Redis 滑动窗口或令牌桶；被限流返回 429 + `20005`。
- 短信/邮件验证码：单手机号频控、单 IP 频控、验证码一次性、有效期 ≤ 5 分钟。
- 防枚举：用户不存在与密码错误返回同一提示；对查询接口限流。
- 写操作配合幂等键，避免重复提交与重放。
- 只信任来自可信代理的 `X-Forwarded-For`，否则可被伪造。

## 9. 数据安全

- 数据分级：公开、内部、敏感（个人信息、支付、凭证）。
- 最小化收集：不需要的字段不采集、不落库、不返回。
- API 返回与日志统一脱敏；导出文件加水印并审计。
- 传输 TLS；存储加密；备份加密；密钥与数据分离。
- 数据保留周期明确，到期删除或匿名化；逻辑删除数据仍需按合规清理。
- 生产数据禁止直接导入测试环境；测试使用脱敏/合成数据。
- 批量导出、跨租户查询需要审批与审计。

## 10. Spring Security 配置骨架

```java
package com.acme.mall.common.config;

import com.acme.mall.common.web.JwtAuthenticationFilter;
import com.acme.mall.common.web.RestAccessDeniedHandler;
import com.acme.mall.common.web.RestAuthenticationEntryPoint;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.Collections;

@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final RestAuthenticationEntryPoint authenticationEntryPoint;
    private final RestAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf().disable()
                .cors().and()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .exceptionHandling()
                .authenticationEntryPoint(authenticationEntryPoint)
                .accessDeniedHandler(accessDeniedHandler).and()
                .authorizeRequests()
                .antMatchers("/actuator/health", "/actuator/info").permitAll()
                .antMatchers("/api/v1/auth/login", "/api/v1/auth/refresh").permitAll()
                .antMatchers("/api/v1/admin/**").hasAuthority("admin:access")
                .antMatchers("/api/v1/**").authenticated()
                .anyRequest().denyAll().and()
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }

    @Bean
    public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(
            JwtAuthenticationFilter filter) {
        FilterRegistrationBean<JwtAuthenticationFilter> registration =
                new FilterRegistrationBean<>(filter);
        registration.setEnabled(false);
        return registration;
    }
}
```

注意：

- `csrf().disable()` 仅适用于"Authorization Header + 无状态"方案；使用 Cookie 时必须开启。
- 自定义 `Filter` 同时是 Spring Bean 时会被 Servlet 容器再次注册；必须像上面一样禁用自动注册，避免重复执行。
- `@EnableGlobalMethodSecurity` 是 Spring Security 5.7 写法；升级到 6 时改为 `@EnableMethodSecurity`。
- 鉴权失败必须返回统一 `Result`，不能返回 Spring 默认 HTML。

认证入口：

```java
@Component
@RequiredArgsConstructor
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        write(response, HttpStatus.UNAUTHORIZED, CommonErrorCode.UNAUTHORIZED);
    }

    private void write(HttpServletResponse response, HttpStatus status,
                       ErrorCode errorCode) throws IOException {
        response.setStatus(status.value());
        response.setCharacterEncoding("UTF-8");
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        objectMapper.writeValue(response.getWriter(), Result.fail(errorCode));
    }
}
```

`RestAccessDeniedHandler` 同理，返回 403 + `FORBIDDEN`。

JWT 过滤器：

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Authentication authentication = jwtTokenService.parseAuthentication(header.substring(7));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (JwtException | IllegalArgumentException e) {
                SecurityContextHolder.clearContext();
                log.debug("invalid jwt, type={}", e.getClass().getSimpleName());
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

JWT 服务：

```java
@Service
public class JwtTokenService {

    private final Key key;

    public JwtTokenService(@Value("${security.jwt.secret}") String secret) {
        byte[] bytes = secret.getBytes(StandardCharsets.UTF_8);
        if (bytes.length < 32) {
            throw new IllegalStateException("security.jwt.secret must be at least 32 bytes");
        }
        this.key = Keys.hmacShaKeyFor(bytes);
    }

    public String createAccessToken(Long userId) {
        Instant now = Instant.now();
        return Jwts.builder()
                .setSubject(String.valueOf(userId))
                .setId(UUID.randomUUID().toString())
                .setIssuedAt(Date.from(now))
                .setExpiration(Date.from(now.plus(2, ChronoUnit.HOURS)))
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }

    public Claims parse(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
}
```

- `parseAuthentication` 应校验签名、过期时间、签发者，并加载用户权限；示例为聚焦安全骨架，未展开权限加载。
- 密钥缺失或过短必须启动失败（fail fast），不能使用默认值。
- 禁止把 Token 放进 URL 参数；统一 `Authorization` 头。

## 11. 输入校验与内容风控

用户请求传入的任何参数都必须做有效性校验，不做校验会带来一连串问题：

- 分页 `pageSize` 过大 → 一次拉全表，内存溢出（本规范 `pageSize ≤ 100`）。
- 排序字段、`order by` 由前端直传 → 慢查询与注入面（必须白名单）。
- 正则校验用户输入 → 恶意构造字符串可触发 ReDoS 死循环；正则要限制输入长度、避免嵌套量词。
- 外部重定向目标地址 → 任意跳转钓鱼（必须白名单，只允许本站或约定域名）。
- 服务端请求用户提供的 URL → SSRF（见第 6 节）。
- 文件路径、命令参数、反序列化数据 → 路径穿越、命令注入、反序列化漏洞。

规则：

- 每个入参都要有明确的类型、长度、取值范围、格式约束；JSR-303 注解 + 业务校验双保险。
- 校验在进入业务逻辑之前完成，失败直接返回 400 + `10001`，不进入事务、不查库。
- 校验为白名单（允许什么），不是黑名单（禁止什么）。
- 向 HTML 页面输出用户数据必须转义或清洗，禁止直接拼接。
- 短信、邮件、下单、支付等平台资源能力必须防重放：数量限制、频次/疲劳度控制、验证码校验、幂等键，缺一不可。
- 发帖、评论、私信、昵称等 UGC 场景必须做文本违禁词过滤、频率限制与账号风控，避免内容风险与资源滥用。
- 敏感数据展示脱敏：手机号 `139****1219`、身份证保留前 6 后 4、银行卡保留后 4、姓名保留首字。脱敏在出参层统一处理（Jackson 序列化器或 VO 转换），不依赖前端。

## 12. 安全评审清单

1. 认证方式明确，Token 有效期、刷新、吊销机制齐全。
2. 密码使用 BCrypt，登录有限流与统一失败提示。
3. 接口默认拒绝，后台接口独立鉴权。
4. 单条资源访问同时校验 `id + owner/tenant`，无水平越权。
5. 请求中的 `userId`/`tenantId` 不被信任。
6. SQL 不含 `${}`，动态排序/列名走白名单。
7. CSRF 策略与凭证传输方式匹配；CORS 白名单无通配。
8. 上传校验类型、大小、路径，存储不可执行。
9. 外部 URL 请求有白名单与私网拦截，HTTP 超时与 TLS 校验开启。
10. 密钥不入库、不入 Git、不进日志；生产关闭文档与敏感 Actuator。
11. 敏感数据加密/脱敏，导出有审批与审计。
12. 依赖无已知高危 CVE，全站 HTTPS。
13. 所有入参都有类型/长度/范围校验，校验在业务逻辑之前完成。
14. 短信、邮件、下单、支付有防重放（数量 + 频次 + 验证码/幂等键）。
15. 展示层已对手机号、身份证、银行卡、姓名脱敏。

# 技术栈与工程配置

## 1. 版本矩阵

除非用户明确要求且记录升级理由，不得偏离下表。

| 组件 | 版本 | 说明 |
|---|---|---|
| JDK | 1.8（基线）/ 11（允许） | 编译目标 1.8；禁用 9+ 语法与 API |
| Maven | 3.6.3+ | 使用 `mvn -U clean verify` |
| Spring Boot | 2.7.18 | 2.7 最后一个开源版本 |
| Spring Framework | 5.3.x | 由 Boot 管理 |
| MyBatis-Plus | 3.5.7 | `mybatis-plus-boot-starter` |
| MySQL | 8.0.x | InnoDB、`utf8mb4` |
| Connector/J | 8.0.33 | `com.mysql:mysql-connector-j` |
| Druid | 1.2.28 | `druid-spring-boot-starter`（Boot 2），替代 HikariCP |
| commons-lang3 | Boot 管理 | `StringUtils` 等字符串/集合工具 |
| Lombok | 1.18.30 | 由 Boot 管理 |
| Hutool | 5.8.48 | `hutool-all`；对象转换使用 `BeanUtil` |
| Knife4j | 4.5.0 | `knife4j-openapi3-spring-boot-starter`（Boot 2），文档地址 `/doc.html` |
| JJWT | 0.11.5 | 无状态鉴权，可选 |
| spring-boot-starter-data-redis | Boot 管理 | 连接池需加 `commons-pool2` |
| Redisson | 3.27.2 | 分布式锁，可选 |
| logstash-logback-encoder | 7.4 | 生产 JSON 日志，可选 |
| Flyway | 8.5.13 | Boot 管理，含 `flyway-mysql` |
| Testcontainers | 1.19.8 | 集成测试，可选 |
| JUnit / Mockito | Boot 管理 | JUnit 5 + Mockito |

禁止：

- 使用 Spring Boot 3.x / `jakarta.*` 混入本项目。
- 使用 Fastjson 1.x（安全漏洞）、Springfox 或 Knife4j 3.x（与 Boot 2.6+ 路径匹配冲突）；文档统一用 Knife4j 4.x 的 OpenAPI 3 starter。
- 使用 `LATEST`、`RELEASE`、`[1.0,)`、`SNAPSHOT` 等动态或快照依赖。
- 同时引入功能重叠的库（如 Knife4j + springfox、Jackson + Fastjson、Druid + HikariCP）。
- 未登记版本就新增依赖；新增前先查版本矩阵与 `mvn dependency:tree`。

## 2. 完整 pom.xml 模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
        <relativePath/>
    </parent>

    <groupId>com.acme</groupId>
    <artifactId>mall-service</artifactId>
    <version>1.0.0</version>
    <name>mall-service</name>
    <description>Mall service</description>

    <properties>
        <java.version>1.8</java.version>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <mybatis-plus.version>3.5.7</mybatis-plus.version>
        <hutool.version>5.8.48</hutool.version>
        <knife4j.version>4.5.0</knife4j.version>
        <jjwt.version>0.11.5</jjwt.version>
        <logstash-encoder.version>7.4</logstash-encoder.version>
        <testcontainers.version>1.19.8</testcontainers.version>
        <druid.version>1.2.28</druid.version>
    </properties>

    <dependencies>
        <!-- Web / 校验 / 监控 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>

        <!-- 持久层 -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>${druid.version}</version>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.0.33</version>
            <scope>runtime</scope>
        </dependency>

        <!-- 对象转换 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>${hutool.version}</version>
        </dependency>

        <!-- 可选：接口文档 -->
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-openapi3-spring-boot-starter</artifactId>
            <version>${knife4j.version}</version>
        </dependency>

        <!-- 可选：Redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>

        <!-- 可选：JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mysql</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

说明：

- `${lombok.version}` 由 `spring-boot-starter-parent` 提供，不要重复写死。
- `annotationProcessorPaths` 只保留 Lombok；Hutool `BeanUtil` 为运行时反射复制，不需要注解处理器。
- 不需要显式声明 `maven-surefire-plugin` 版本，Boot 已管理。
- 多模块时，版本统一放 parent 的 `<properties>` 或 `<dependencyManagement>`，子模块不写版本。
- 引入 Flyway 时增加 `flyway-core` 与 `flyway-mysql`（版本由 Boot 管理），并开启 `spring.flyway.enabled`。

## 3. application.yml 模板

```yaml
server:
  port: 8080
  shutdown: graceful

spring:
  application:
    name: mall-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  lifecycle:
    timeout-per-shutdown-phase: 30s
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://${DB_HOST:127.0.0.1}:${DB_PORT:3306}/${DB_NAME:mall_db}?useUnicode=true&characterEncoding=utf8&useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD:}
    druid:
      name: mall-druid
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 3000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validation-query: SELECT 1
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      pool-prepared-statements: true
      max-pool-prepared-statement-per-connection-size: 20
      filters: stat
      connection-properties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=1000
      stat-view-servlet:
        enabled: false
      web-stat-filter:
        enabled: false
  jackson:
    time-zone: Asia/Shanghai
    deserialization:
      fail-on-unknown-properties: false
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 20MB

mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  type-aliases-package: com.acme.mall.entity
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    banner: false
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
      table-underline: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    root: info
    com.acme.mall: info
  file:
    name: logs/mall-service.log

knife4j:
  enable: true
```

`application-dev.yml`：

```yaml
logging:
  level:
    com.acme.mall: debug
```

`application-prod.yml`：

```yaml
logging:
  level:
    root: warn
    com.acme.mall: info
knife4j:
  enable: true
  production: true
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

约束：

- 数据库密码、JWT 密钥、第三方 AK/SK 只能来自环境变量或配置中心，禁止写入仓库。
- 生产环境 `knife4j.production=true`，并在网关/安全层拒绝 `/doc.html`、`/v3/api-docs/**`、`/swagger-ui/**`；Actuator 只暴露必要端点。
- 时区统一 `Asia/Shanghai`，JDBC URL 必须带 `serverTimezone=Asia/Shanghai`。
- 生产建议 `useSSL=true`；`allowPublicKeyRetrieval=true` 仅在内网确认无中间人风险时使用。
- `id-type: auto` 与单库自增对应；若采用雪花 ID，改为 `assign_id` 并在 DDL 使用 `BIGINT`（非 `AUTO_INCREMENT`），全库统一。
- `log-impl` 仅用于开发排查 SQL；生产若 SQL 日志量大，调整为不输出或只输出慢 SQL。
- `type-aliases-package` 默认指向 `com.acme.mall.entity`；迁入多业务域后补充各域 `entity` 包，或 XML 一律使用全限定类型。
- `@MapperScan` 扫描根包并按 `@Mapper` 注解过滤，单域、多域都无需改扫描范围。
- 连接池固定 Druid：`spring.datasource.type=com.alibaba.druid.pool.DruidDataSource`，不要同时保留 HikariCP；启动后用 `mvn dependency:tree` 确认没有 Hikari。
- `max-active` 不得超过 MySQL `max_connections` 按实例分摊后的额度；`max-wait` 必须有界，禁止无限等待。
- `filters` 默认只开 `stat`；`wall` 需先验证现有 SQL 不被拦截。
- 生产环境 `stat-view-servlet`、`web-stat-filter` 必须关闭；如要开启监控页，必须配置登录账号、限制内网访问，禁止匿名暴露。
- Druid 监控页 `/druid/**` 不纳入业务 API 鉴权体系，不能直接暴露在公网。

## 4. 核心配置类

### 4.1 MybatisPlusConfig

```java
package com.acme.mall.common.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.BlockAttackInnerInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.OptimisticLockerInnerInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.reflection.MetaObject;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.LocalDateTime;

@Configuration
@MapperScan(basePackages = "com.acme.mall", annotationClass = Mapper.class)
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.MYSQL);
        pagination.setMaxLimit(500L);
        pagination.setOverflow(false);
        interceptor.addInnerInterceptor(pagination);
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return interceptor;
    }

    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new MetaObjectHandler() {
            @Override
            public void insertFill(MetaObject metaObject) {
                LocalDateTime now = LocalDateTime.now();
                strictInsertFill(metaObject, "createTime", LocalDateTime.class, now);
                strictInsertFill(metaObject, "updateTime", LocalDateTime.class, now);
                strictInsertFill(metaObject, "version", Integer.class, 0);
            }

            @Override
            public void updateFill(MetaObject metaObject) {
                strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
            }
        };
    }
}
```

`@MapperScan` 必须带 `annotationClass = Mapper.class`，避免把 Service 接口误注册为 Mapper；每个 Mapper 接口仍要标注 `@Mapper`。`maxLimit = 500` 是安全上限，业务分页仍限制 `pageSize ≤ 100`。

### 4.2 JacksonConfig

```java
package com.acme.mall.common.config;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.TimeZone;

@Configuration
public class JacksonConfig {

    private static final DateTimeFormatter DATE_TIME =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
        return builder -> {
            builder.serializers(new LocalDateTimeSerializer(DATE_TIME));
            builder.deserializers(new LocalDateTimeDeserializer(DATE_TIME));
            builder.serializers(new LocalDateSerializer(DateTimeFormatter.ISO_LOCAL_DATE));
            builder.deserializers(new LocalDateDeserializer(DateTimeFormatter.ISO_LOCAL_DATE));
            builder.serializerByType(BigDecimal.class, ToStringSerializer.instance);
            builder.timeZone(TimeZone.getTimeZone("Asia/Shanghai"));
            builder.featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        };
    }
}
```

`spring.jackson.date-format` 只影响 `java.util.Date`，`LocalDateTime` 必须通过上面的序列化器统一格式。`BigDecimal` 统一序列化为字符串，避免前端浮点精度问题；请求端仍同时接受字符串和数字。

### 4.3 线程池

- 必须使用 `ThreadPoolTaskExecutor`，禁止 `Executors.newFixedThreadPool` / `newCachedThreadPool` 默认无界队列。
- 必须设置有界队列、线程名前缀、`CallerRunsPolicy`、优雅停机。
- 异步任务中需要 traceId 时手动传递 MDC，禁止依赖线程继承。

```java
@Bean("bizExecutor")
public ThreadPoolTaskExecutor bizExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("biz-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(30);
    executor.initialize();
    return executor;
}
```

## 5. 依赖治理

1. 新增依赖前确认：是否已有同类库、是否支持 JDK 8、是否仍在维护、是否有已知 CVE。
2. 新增后运行 `mvn dependency:tree -Dincludes=groupId:artifactId`，排除冲突的传递依赖。
3. 统一排除 `commons-logging`（Spring 用 `spring-jcl`）与旧版 `fastjson`。
4. 升级 Spring Boot 补丁版本时，同步执行全量测试与 `mvn dependency:analyze`。
5. 禁止在业务模块自行覆盖 Boot 管理的版本，除非版本矩阵同步更新。

### 5.1 坐标与版本语义

- `groupId`：`com.{公司/BU}.{业务线}[.{子业务线}]`，最多 4 级，如 `com.acme.mall`。
- `artifactId`：`{产品线}-{模块名}`，语义不重复不遗漏，如 `mall-api`、`mall-common`。
- `version`：`主版本.次版本.修订号`，起始 `1.0.0`（不是 `0.0.1`，也不是 `1.0.0.0`）。

| 位 | 何时递增 |
|---|---|
| 主版本 | 产品方向改变、大规模 API 不兼容、架构不兼容升级 |
| 次版本 | 向下兼容地增加主要功能、影响范围极小的不兼容调整 |
| 修订号 | 完全兼容地修 BUG、加次要特性 |

- 线上应用禁止依赖 `SNAPSHOT`（安全补丁包除外）；正式版本先到仓库查证，版本号只增不改、不覆盖发布。
- 同一个 GroupId 下多个 jar 必须共用一个版本变量（`${spring.version}`），不要逐处写死。
- 同一子项目内禁止出现相同 `groupId` + `artifactId` 但不同 `version`（合并成一个包时只会保留一个，线下正常线上故障）。
- 依赖声明放 `<dependencies>`，版本仲裁放 `<dependencyManagement>`；子模块不写版本。
- 升级二方库时除功能点外，其它 jar 的仲裁结果必须保持不变；有变化要 `dependency:tree` 比对并 `<exclude>` 掉差异。
- 二方库自身要精简可控：只暴露 API、必要的领域模型、工具类、常量与枚举，依赖尽量 `provided`；不绑定日志实现，只依赖门面。
- 二方库可以用枚举作入参，但**接口返回值不要用枚举**（调用方升级才能认识新值，等于不兼容）。
- 二方库尽量不带配置项；新增配置项等于扩大兼容面。
- 不使用不稳定的工具包（编译期正常、运行期抛异常的那种）。

## 6. 运行参数（JVM 与操作系统）

JVM：

```bash
java -server \
  -Xms2g -Xmx2g \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/home/admin/mall/logs/heapdump \
  -XX:+UseG1GC \
  -Duser.timezone=Asia/Shanghai \
  -Dfile.encoding=UTF-8 \
  -jar mall-service.jar
```

- `-Xms` 与 `-Xmx` 设为相同值，避免运行期反复扩缩堆带来的停顿。
- 必须开启 `-XX:+HeapDumpOnOutOfMemoryError`：OOM 可能几个月才出现一次，现场的堆信息是唯一线索。
- 容器环境用 `-XX:MaxRAMPercentage=75` 代替固定 `-Xmx`，避免超出 cgroup 限制被 OOMKilled。
- 显式设置时区与文件编码，不依赖宿主机默认值。

操作系统（高并发服务）：

- 调小 `TIME_WAIT` 超时：`/etc/sysctl.conf` 中 `net.ipv4.tcp_fin_timeout = 30`（默认 240 秒会让大量连接卡在 `TIME_WAIT`，新连接建不上）。
- 调大文件句柄数：默认 1024 在并发上千时会出现 `open too many files`；按内存容量调整 `nofile`（如 65535），同时确认 `fs.file-max`。
- 确认 `net.core.somaxconn` 与 `tcp_max_syn_backlog` 与连接队列匹配，避免瞬时高峰丢连接。
- 应用端口只对内网或负载均衡开放；日志盘单独挂载并设配额；`/actuator`、`/druid` 不进公网。

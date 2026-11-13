# 测试规范

> 本文件整合《Java 开发手册》嵩山版"单元测试"章节（AIR / BCDE 原则、目录与覆盖率要求），并按本项目技术栈（JUnit 5 + Mockito + Testcontainers）落地。

## 1. 测试分层

| 层级 | 工具 | 范围 | 速度 |
|---|---|---|---|
| 单元测试 | JUnit 5 + Mockito + AssertJ | Service 业务规则、状态流转、金额计算 | 毫秒级，无 Spring |
| Web 切片 | `@WebMvcTest` + MockMvc | Controller 路由、校验、统一响应、异常映射 | 秒级 |
| 集成测试 | `@SpringBootTest` + Testcontainers MySQL | Mapper、SQL、事务、逻辑删除、唯一索引 | 十秒级 |
| 端到端 | 独立测试环境 | 跨服务主流程 | 分钟级，不纳入常规 CI |

原则：

- **AIR 原则**：Automatic（全自动、非交互）、Independent（用例之间互不调用、不依赖执行顺序）、Repeatable（可重复执行，不受网络/中间件/时间等外界环境影响）。
- 测试结果必须靠断言判定，禁止用 `System.out` 打日志人肉核对。
- 依赖通过构造器注入，测试时替换为内存实现或 Mock；不要在测试里连真实第三方。
- 业务规则优先用单元测试覆盖，不为每个方法启动 Spring。
- 涉及真实 SQL 语义（分页、索引、唯一键、逻辑删除）必须用 MySQL 集成测试，不用 H2 替代。
- 新增/修改 Service 方法必须有单元测试；新增/修改 Mapper SQL 必须有集成测试。
- 测试粒度到方法级，最多到类级；跨类、跨系统的交互归集成测试，不塞进单元测试。
- 测试不得连接生产库、不得调用真实第三方、不得依赖执行顺序。
- 测试代码必须写在 `src/test/java`，禁止混在业务代码目录。
- 不可测的代码应重构使其可测，不要为了覆盖率写不规范测试。

## 2. 命名与结构

- 测试类与被测类同包、同名 + `Test`；集成测试以 `IT` 结尾。
- 方法名：`方法名_when场景_预期结果`，如 `cancel_whenStatusNotWaitPay_throwsBizException`。
- 使用 `@DisplayName` 写中文业务语义，便于非开发人员阅读。
- 每个测试只验证一个行为；一个测试内可以有多个相关断言。
- 结构建议 given / when / then 三段，空行分隔。
- 测试方法不写业务分支、不写循环断言、不依赖其他测试产生的数据。

## 3. 单元测试

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceImplTest {

    @Mock
    private OrderMapper orderMapper;

    @Mock
    private OrderConvert orderConvert;

    @InjectMocks
    private OrderServiceImpl orderService;

    @Test
    @DisplayName("非待支付订单取消时抛出状态不允许异常")
    void cancel_whenStatusNotWaitPay_throwsBizException() {
        OrderEntity entity = new OrderEntity();
        entity.setId(1L);
        entity.setStatus(OrderStatus.PAID);
        when(orderMapper.selectById(1L)).thenReturn(entity);

        BizException ex = assertThrows(BizException.class, () -> orderService.cancel(1L));

        assertThat(ex.getErrorCode()).isEqualTo(OrderErrorCode.ORDER_STATUS_NOT_ALLOWED);
        verify(orderMapper, never()).updateById(any(OrderEntity.class));
    }

    @Test
    @DisplayName("订单不存在时查询抛出未找到异常")
    void get_whenOrderMissing_throwsBizException() {
        when(orderMapper.selectById(1L)).thenReturn(null);

        BizException ex = assertThrows(BizException.class, () -> orderService.get(1L));

        assertThat(ex.getErrorCode()).isEqualTo(OrderErrorCode.ORDER_NOT_FOUND);
    }
}
```

规则：

- 使用 `@ExtendWith(MockitoExtension.class)`，不启动 Spring。
- 业务 Service 单测直接实例化 `XxxServiceImpl`（`@InjectMocks`）；Controller 切片测试 mock `XxxService` 接口。
- `@Mock` 依赖，`@InjectMocks` 被测对象；不要 mock 被测类本身。
- 只 stub 当前测试实际用到的调用，避免宽泛 `any()` 掩盖参数错误。
- 返回值断言用 AssertJ；异常用 JUnit `assertThrows` 后断言错误码。
- `verify` 只验证关键交互（如状态非法时不得写库），不要逐行验证内部调用。
- 不测试 private 方法；通过 public 方法覆盖。
- 不使用 `Thread.sleep`、不依赖当前真实时间；时间敏感逻辑注入 `Clock` 或使用固定时间。

## 4. Controller 切片测试

```java
@WebMvcTest(OrderController.class)
@AutoConfigureMockMvc(addFilters = false)
@Import(GlobalExceptionHandler.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    @DisplayName("查询订单成功返回统一结构")
    void get_whenOrderExists_returnsSuccess() throws Exception {
        OrderResp resp = new OrderResp();
        resp.setId(1L);
        resp.setOrderNo("O1");
        resp.setStatus(OrderStatus.WAIT_PAY);
        given(orderService.get(1L)).willReturn(resp);

        mockMvc.perform(get("/api/v1/orders/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(0))
                .andExpect(jsonPath("$.data.id").value("1"))
                .andExpect(jsonPath("$.data.status").value("WAIT_PAY"));
    }

    @Test
    @DisplayName("创建订单缺少金额返回400")
    void create_whenAmountMissing_returnsBadRequest() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code").value(10001));
    }
}
```

规则：

- `@WebMvcTest` 只加载 Web 层；Service 用 `@MockBean`。
- 需要验证全局异常映射时 `@Import(GlobalExceptionHandler.class)`。
- 聚焦 Controller 时用 `@AutoConfigureMockMvc(addFilters = false)` 关闭安全过滤器；需要验证鉴权时不要关闭，改用 `@WithMockUser` 或真实 Token。
- 断言 JSON 结构与错误码，不断言完整字符串。
- 覆盖：成功、参数校验失败、业务异常、未找到、无权限（如适用）。

## 5. MySQL 集成测试

```java
@SpringBootTest
@Testcontainers
class OrderMapperIT {

    @Container
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0.33")
            .withDatabaseName("mall_db")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void mysqlProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
    }

    @Autowired
    private OrderMapper orderMapper;

    @Test
    @DisplayName("逻辑删除后按ID查询返回空")
    void selectById_whenDeleted_returnsNull() {
        OrderEntity entity = new OrderEntity();
        entity.setOrderNo("T-1001");
        entity.setUserId(1L);
        entity.setAmount(new BigDecimal("99.00"));
        entity.setStatus(OrderStatus.WAIT_PAY);
        entity.setRemark("");
        orderMapper.insert(entity);

        orderMapper.deleteById(entity.getId());

        assertThat(orderMapper.selectById(entity.getId())).isNull();
    }
}
```

规则：

- 使用 Testcontainers 启动真实 MySQL 8.0，镜像版本与生产一致。
- 表结构由 Flyway 迁移创建，测试不手写建表 SQL（除非专门测试迁移）；未引入 Flyway 时执行 `db/migration` 下的同一份脚本，不要维护两套表结构。
- 覆盖：唯一索引冲突、逻辑删除、乐观锁、分页与排序、复杂 XML SQL。
- 测试数据用构造器/Builder 生成，不依赖生产数据与固定自增 ID。
- 默认每个测试独立回滚或清理；测试事务行为时不要套测试级 `@Transactional`。
- 无 Docker 的 CI 使用独立测试 MySQL 实例与独立 schema，禁止降级为 H2 验证 SQL。
- 集成测试类命名 `XxxIT`，由 Failsafe 执行。

Failsafe 配置：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

命令：

```bash
mvn clean test      # 单元 + Web 切片
mvn clean verify    # 单元 + 集成 + 打包
```

## 6. 测试数据与断言

用例设计遵循 **BCDE 原则**：

- **B**order：边界值——循环边界、`0`/`-1`/最大值、特殊时间点（月末、闰年 2/29、跨年）、数据顺序。
- **C**orrect：正确输入得到预期结果。
- **D**esign：与设计文档、需求用例对齐，不只按实现写测试。
- **E**rror：强制错误输入（非法数据、异常流程、超出业务允许范围），并断言错误码与提示。

数据准备：

- 使用 `OrderEntityFixtures.waitPayOrder()` 一类的固定夹具，避免每个测试重复构造。
- 不发真实手机号、身份证、姓名；使用明显虚构数据。
- 数据库相关测试必须用程序插入或导入脚本准备数据，禁止先手工往库里插一行再测删除——手工数据往往不符合业务插入规则，测出来的结论是假的。
- 集成测试默认自动回滚，或给测试数据统一加前后缀标识（如 `UT_`），不给环境留下脏数据。
- 断言具体值，避免 `assertNotNull` 走天下。
- 浮点/金额断言用 `isEqualByComparingTo`，不要用 `isEqualTo`。
- 时间断言使用固定 `Clock` 或允许误差，不依赖机器时区。
- 异步逻辑使用 Awaitility 等待条件，禁止 `Thread.sleep`。
- 测试失败信息要能定位业务原因，不使用无意义断言消息。

## 7. Mock 规则

- 只 mock 边界依赖：Mapper、外部客户端、消息发送器、缓存。
- 不 mock 值对象、集合、JDK 类、被测类。
- 不过度 mock 导致测试与实现耦合；SQL 行为交给集成测试。
- `@MockBean` 会破坏 Spring 上下文缓存，切片测试中控制数量。
- 外部系统在 Manager 层 mock；不要在 Service 内部深层 mock 第三方 SDK。

## 8. 覆盖率与质量门槛

- 新增/修改代码语句覆盖率 ≥ 70%；核心领域（金额、状态机、权限）语句与分支覆盖率目标 100%，准入底线 80%，未达标需在评审记录中说明原因。
- 覆盖率按增量代码统计，不用历史代码摊平指标。
- 核心业务、核心应用、核心模块的增量代码必须带测试并通过。
- 不追求 100% 覆盖率；优先覆盖分支与异常路径。
- 必须覆盖：主流程、参数非法、资源不存在、状态冲突、重复提交、乐观锁失败、逻辑删除、权限不足。
- 禁止提交被 `@Disabled` / `@Ignore` 的测试，除非附工单号与恢复计划。
- 测试必须稳定：不依赖网络、不依赖执行顺序、不依赖当前时间、不产生共享副作用。
- CI 中单元测试失败立即阻断；集成测试可单独阶段执行，但发布前必须通过。
- 单测范围在技术方案评审时与测试同学一起确定，覆盖主要用例（UC）。
- 单元测试在提测前完成，不接受发布后补。

## 9. 测试评审清单

1. 新增 Service 方法是否有对应单元测试。
2. 是否覆盖异常与边界，而不只是 happy path。
3. Mapper/SQL 变更是否有 MySQL 集成测试。
4. 是否 mock 了不该 mock 的对象，或断言了实现细节。
5. 测试是否依赖固定 ID、真实时间、执行顺序或外部网络。
6. 集成测试是否使用真实 MySQL 与迁移脚本。
7. 金额、时间、枚举断言是否正确。
8. 是否验证了逻辑删除、唯一索引、乐观锁等关键语义。
9. 测试命名是否表达场景与预期。
10. `mvn clean verify` 是否全绿，覆盖率是否达标。

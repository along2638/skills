# Java 编码规范（JDK 8 基线）

> 本文件整合《Java 开发手册》嵩山版（v1.7.0）"编程规约"章节，并按本项目基线（Spring Boot 2.7 + JDK 8 + MyBatis-Plus）落地。手册与本规范不一致处，按 `alibaba-manual-map.md` 的偏差表执行，不要两套并用。

## 1. 命名

| 元素 | 规则 | 示例 |
|---|---|---|
| 包 | 全小写；根包用产品名；单域业务包直接在根包下，多域用 `modules/{domain}` | `com.acme.mall`、`com.acme.mall.modules.order` |
| 类/接口 | UpperCamel，无 `I` 前缀 | `OrderService`、`OrderController` |
| 抽象类 | `Abstract` 或 `Base` 开头 | `AbstractOrderHandler`、`BaseEntity` |
| 实现类 | 接口名 + `Impl` | `OrderServiceImpl` |
| 异常类 | `Exception` 结尾 | `BizException` |
| 方法/变量 | lowerCamel | `cancelOrder`、`orderNo` |
| 常量 | UPPER_SNAKE | `MAX_PAGE_SIZE` |
| 枚举常量 | UPPER_SNAKE，code 与常量名一致 | `WAIT_PAY` |
| 测试类 | 被测类名 + `Test` / `IT` | `OrderServiceImplTest` |
| 配置类 | `XxxConfig` | `MybatisPlusConfig` |

- 禁止下划线、`$` 开头或结尾：`_name`、`name_`、`$name` 全部不允许。
- 禁止拼音、拼音与英文混写、中文标识符：`DaZhePromotion`、`getPingfenByName()` 不可用；`ali`、`taobao` 这类国际通用词可视同英文。
- 禁止不规范缩写：`AbsClass`、`condi`、`Fu` 一律展开为 `AbstractClass`、`condition`、`Function`；通用缩写只保留 `id`、`url`、`dto`、`vo`、`api`、`config`，不要大小写混写（`userId` 不是 `userID`）。
- 表示类型的名词放词尾：`startTime`、`workQueue`、`nameList`，不用 `startedAt`、`QueueOfWork`。
- 包名全小写，点分隔符之间只有一个自然语义的英文单词，且用单数；类名有复数含义时可以用复数。
- 数组类型与中括号紧贴：`int[] arrayDemo`，不写 `String args[]`。
- 枚举类名统一为 `XxxStatus` / `XxxType` / `XxxEnum` 之一，同一项目不混用；常量名 UPPER_SNAKE。
- 接口中的方法和常量不加 `public`/`abstract` 等修饰符；接口不定义可变状态。
- 禁止父子类成员变量同名，禁止同一方法内不同代码块的同名局部变量；非 setter 的参数名不得与成员变量同名。
- 布尔字段/变量不加 `is` 前缀：`enabled`、`deleted`；方法用 `isEnabled()`。
- 避免歧视性用语：用 `blockList`/`allowList`/`primary`/`secondary`，不用 `blackList`/`whiteList`/`slave`。
- 同一语义只用一个词：`delete` / `remove` 不混用，`get` / `query` / `fetch` 不混用。
- 对象后缀统一为 `Entity`（PO）、`Req`（入参 DTO）、`Resp`（VO）、`DTO`（内部传输）；不要混用 `XxxPO`、`XxxVO`、`XxxDO` 等别名。
- 类名全局唯一，避免不同包同名 Bean 在 Spring 中冲突（Boot 2.6+ 默认禁止覆盖 Bean）。
- 命名体现设计模式：`OrderFactory`、`LoginProxy`、`ResourceObserver`。

Service / Mapper 方法命名固定前缀：

| 语义 | 前缀 | 示例 |
|---|---|---|
| 取单个对象 | `get` | `getOrder(Long id)` |
| 取多个对象 | `list`（复数结尾） | `listOrders(OrderQueryReq req)` |
| 取统计值 | `count` | `countByStatus(OrderStatus status)` |
| 新增 | `save` / `insert` | `saveOrder(...)` |
| 删除 | `remove` / `delete` | `removeOrder(Long id)` |
| 修改 | `update` | `updateOrder(...)` |
| 可空查询 | `find`（返回 `Optional` 或 `null`） | `findByOrderNo(String orderNo)` |

## 2. 格式与复杂度

- 4 空格缩进，禁止 Tab；单行 ≤ 120 字符；UTF-8；LF 换行（不用 CRLF）。
- 空代码块写成 `{}`，中间不留换行和空格；非空代码块：左大括号前不换行、左大括号后换行、右大括号前换行，`else` 前的右大括号不换行。
- 保留字 `if/for/while/switch/do` 与括号之间有且仅有一个空格；括号内侧不加空格。
- 二目、三目运算符两侧各一个空格；`//` 与注释内容之间有且仅有一个空格。
- 强制转换右括号与值之间不加空格：`int second = (int) first + 2;`。
- 超 120 字符换行：第二行缩进 4 空格、后续行不再叠加缩进；运算符与点号随下文换行；多参数在逗号后换行；禁止在括号前换行。
- 方法参数定义与调用时，逗号后必须加空格。
- 不用空格把 `=` 对齐；不同语义的代码块之间用一个空行分隔，不插入多个空行。
- `if/for/while` 必须带大括号；一行只写一条语句。
- import 不用 `*`，不保留未使用 import；顺序：`java` → `javax` → 第三方 → 项目。
- 方法 ≤ 80 行（含签名、大括号、空行），类 ≤ 500 行；超过时按职责拆分。
- 嵌套 ≤ 3 层，优先早返回（guard clause）。
- 方法参数 ≤ 5 个；更多时封装为 Req 对象。
- 禁止布尔旗标参数（`save(boolean force)`），改用枚举或拆分方法。
- 禁止魔法值：数字、状态、错误码、请求头名必须常量化或枚举化。
- `long`/`Long` 字面量用大写 `L`，禁止小写 `l`（`2l` 与 `21` 难以区分）。
- 常量按功能归类维护，不建大而全常量类：`CacheConsts`、`SystemConfigConsts`；固定范围取值优先用 `enum`。

## 3. 注释与 Javadoc

- 类、类属性、类方法统一 Javadoc（`/** */`），禁止用 `//` 代替。必写范围：类、Service 接口方法、Controller 方法、工具类公有方法、抽象方法、枚举常量；Entity/DTO/Resp 的 getter/setter 由类注释覆盖，不必逐个写。
- 所有抽象方法（含接口方法）必须有 Javadoc：做什么、参数、返回值、异常，以及实现约束与调用注意事项。
- 类注释必须写职责，并带创建者与创建日期（作者用实际提交人，未知时留待提交者补全，不编造）：

```java
/**
 * 订单应用服务。
 *
 * @author zhangsan
 * @date 2026/10/25
 */
```

- 方法内部单行注释用 `//` 并另起一行；多行注释用 `/* */` 且与代码对齐。
- 枚举的每个常量都必须注释说明用途与业务含义。
- 代码与注释同步修改；参数、返回值、异常、核心逻辑变更必须同步更新注释。
- 删除未使用的字段、方法、内部类、参数与局部变量；不再使用的代码直接删，需要保留历史就查代码仓库。
- 被临时注释掉的代码，在注释上方用 `///` 写明原因与恢复条件；无恢复计划的直接删除。
- 注释写"为什么"和业务含义，不复述代码；`put(elephant, fridge)` 这类自解释代码不需要注释。
- 中文注释优于半吊子英文；专有名词与关键字保留英文原文。
- 特殊标记必须带标记人与时间：`// TODO(zhangsan, 2026/10/25): 补充退款回调幂等`、`// FIXME(lisi, 2026/10/25): 并发下可能重复扣减`。

## 4. OOP 与相等性

- 所有覆写方法必须加 `@Override`；接口过时用 `@Deprecated` 并说明替代接口。
- 不使用已过时的类或方法（如 `URLDecoder.decode(String)`），改用当前推荐重载。
- 外部已调用或二方库依赖的接口不得修改方法签名；升级只做新增。
- `equals` 用常量或确定有值的对象调用，避免 NPE：

```java
// 正确
if ("PAID".equals(status)) { ... }
if (Objects.equals(a.getUserId(), b.getUserId())) { ... }
```

- 所有整型包装类比较一律 `equals` 或先拆箱比较；`==` 只在 -128~127 的 `Integer` 缓存区间"看起来正确"，不可依赖。
- 浮点数等值判断：基本类型禁用 `==`，包装类型禁用 `equals`；用误差范围或 `BigDecimal`：

```java
if (Math.abs(a - b) < 1e-6F) { ... }

BigDecimal x = new BigDecimal("1.0").subtract(new BigDecimal("0.9"));
BigDecimal y = new BigDecimal("0.9").subtract(new BigDecimal("0.8"));
if (x.compareTo(y) == 0) { ... }
```

- `BigDecimal` 等值比较用 `compareTo()`，不用 `equals()`（`1.0` 与 `1.00` 的 `equals` 为 false）。
- 禁止 `new BigDecimal(double)`；用 `BigDecimal.valueOf(double)` 或字符串构造。
- 覆写 `equals` 必须同时覆写 `hashCode`；自定义对象作为 `Set` 元素或 `Map` key 时必须两者都覆写。
- POJO 属性、RPC 出入参一律用包装类型；局部变量用基本类型。属性无初值才能表达 `null`（未设置/远程失败），避免自动拆箱 NPE。
- POJO 不设任何属性默认值（如 `createTime = LocalDateTime.now()`），否则属性在未被赋值时仍带"当前时间"，更新其它字段时会被顺手写回。
- POJO 必须实现 `toString()`；继承场景先拼 `super.toString()`。
- 禁止 POJO 同时存在 `isXxx()` 与 `getXxx()`；布尔属性不加 `is` 前缀。
- 构造方法只做赋值，禁止业务逻辑与远程调用；初始化逻辑放 `init` 方法。
- `final` 用在这些场合：不允许被继承的类、不允许改引用的域对象、不允许覆写的方法（如 POJO setter）、运行期不再重赋值的局部变量。
- 序列化类新增字段不修改 `serialVersionUID`；完全不兼容升级时才改。
- 属性类型必须与数据库字段类型一致：`bigint` ↔ `Long`，`int` ↔ `Integer`，`decimal` ↔ `BigDecimal`；用 `Integer` 接 `bigint` 迟早溢出。
- 静态成员直接用类名访问，不通过实例引用。
- 可变参数只用于同类型、同业务含义的参数，且放参数列表最后；尽量不用。
- 慎用 `clone()`：默认浅拷贝，深拷贝必须逐层处理。
- 访问控制从严：工具类私有构造器；仅本类使用用 `private`；只对子类开放用 `protected`。
- 多个构造器或同名方法集中放置；类内顺序为公有方法 > 保护方法 > 私有方法 > getter/setter。
- getter/setter 不写业务逻辑。
- `split` 结果按索引取值前先判断末尾是否有内容（`"a,b,c,,".split(",")` 长度为 3）。

## 5. 数值与金额

- 只用 `BigDecimal`，不用 `double`/`float` 表达金额、费率、库存等精确值。
- 用 `BigDecimal.valueOf(...)` 或字符串构造，禁止 `new BigDecimal(0.1)`。
- 比较用 `compareTo`，禁止 `equals`。
- 除法必须指定 scale 与 `RoundingMode`，否则除不尽抛 `ArithmeticException`。
- 入库前统一 `setScale(2, RoundingMode.HALF_UP)`；展示需要时再 `stripTrailingZeros`。

```java
BigDecimal total = price.multiply(BigDecimal.valueOf(quantity))
        .setScale(2, RoundingMode.HALF_UP);
if (total.compareTo(BigDecimal.ZERO) <= 0) {
    throw new BizException(OrderErrorCode.ORDER_AMOUNT_INVALID);
}
```

- 手册要求"货币金额以最小货币单位整型存储"，本规范用 `DECIMAL(18,2)` + `BigDecimal`；对接以"分"为单位的外部系统时，在 Manager 边界显式换算并统一舍入，不在业务代码里混用两种单位。
- 整型运算注意溢出：`int` 相乘先提升为 `long`（`(long) a * b`）；计数超过 21 亿用 `long`。
- 金额入参用 `@DecimalMin`、`@Digits(integer = 16, fraction = 2)` 校验，禁止只靠前端限制。

## 6. 日期时间

- 统一 `LocalDateTime` / `LocalDate` / `LocalTime`；禁止 `java.util.Date`、`Calendar`。
- 禁止 `java.sql.Date`、`java.sql.Time`、`java.sql.Timestamp`（精度语义不一致，跨类型比较容易踩 JDK 缺陷）。
- 日期格式年份用小写 `y`：`yyyy-MM-dd HH:mm:ss`。大写 `YYYY` 是 week-year，跨年周会返回下一年。
- 分清 `M`（月）与 `m`（分）、`H`（24 小时制）与 `h`（12 小时制）。
- `DateTimeFormatter` 声明为 `static final`（线程安全）；禁止共享 `SimpleDateFormat`。
- 当前毫秒数用 `System.currentTimeMillis()`，不用 `new Date().getTime()`；纳秒级耗时用 `System.nanoTime()`；JDK 8 计时场景可用 `Instant`。
- 业务时间统一 `Asia/Shanghai`，跨时区转换必须显式指定 `ZoneId`。
- 不写死一年 365 天：用 `LocalDate.now().lengthOfYear()`、`LocalDate.of(2011, 1, 1).lengthOfYear()`。
- 闰年 2 月 29 日加一年不存在该日期，会员到期、账单周期等场景需单独处理。
- 月份用枚举或 `Month` 表达；`Calendar` 的 month 是 0-11，极易错一月。
- 时间范围查询用左闭右开，不允许对字段做 `DATE()` 等函数转换。

## 7. 字符串、集合与 Stream

字符串与判空：

```java
if (StringUtils.isBlank(req.getRemark())) { ... }
if (!Objects.equals(a.getUserId(), b.getUserId())) { ... }
```

- 字符串比较用 `Objects.equals` 或常量在前，禁止 `==`。
- 判空用 `StringUtils.isBlank`（commons-lang3）/ `CollectionUtils.isEmpty`。
- 循环拼接用 `StringBuilder`，禁止 `+=`（每次循环都会新建对象）。
- 禁止 `new String(...)`、`toString()` 后再判空等反模式。
- 正则 `Pattern` 必须预编译为 `static final`，禁止在方法体内 `Pattern.compile`。
- 禁止用 Apache `BeanUtils` 做属性拷贝（性能差），用 Hutool `BeanUtil`。
- 禁止用 JSON 工具把对象转字符串去打日志（getter 可能抛异常，且泄露敏感字段）。

集合：

- 判断集合是否为空用 `isEmpty()`，不用 `size() == 0`。
- 初始化时指定容量：`new HashMap<>(16)`；已知规模时 `initialCapacity = size / 0.75 + 1`。
- `Collectors.toMap` 必须传 `mergeFunction`，否则重复 key 抛 `IllegalStateException`；`value` 为 `null` 时仍会抛 NPE，取值前先过滤或改用 `forEach` 组装。
- `subList` 返回视图而非独立列表：不能强转 `ArrayList`，父集合增删会导致 `ConcurrentModificationException`。
- `keySet()`/`values()`/`entrySet()` 返回的集合不可新增元素；`Collections.emptyList()`/`singletonList()` 不可变。
- `foreach` 中禁止 `add`/`remove`，删除用 `Iterator.remove()`；并发场景对 Iterator 加锁。
- 集合转数组用 `toArray(new String[0])`，不用无参 `toArray()`（只能得到 `Object[]`）。
- `addAll` 前必须判空入参（内部直接 `c.toArray()`）。
- `Arrays.asList()` 返回定长列表，`add/remove/clear` 抛 `UnsupportedOperationException`，且与原数组互相影响。
- 遍历 Map 用 `entrySet` 或 `Map.forEach`，不用 `keySet` 再 `get`（遍历两次）。
- 泛型通配符遵循 PECS：频繁读取用 `<? extends T>`，频繁写入用 `<? super T>`；`<? extends T>` 不能 `add`。
- 无泛型集合赋值给泛型集合后，取元素前必须 `instanceof` 判断。
- `Comparator` 必须满足自反/对称/传递三条，相等时返回 `0`；只返回 `1`/`-1` 会导致 `IllegalArgumentException: Comparison method violates its general contract`。
- Map 对 null 的支持差异：

| 集合 | Key | Value | 线程安全 |
|---|---|---|---|
| `Hashtable` | 不允许 `null` | 不允许 `null` | 是 |
| `ConcurrentHashMap` | 不允许 `null` | 不允许 `null` | 是 |
| `TreeMap` | 不允许 `null` | 允许 `null` | 否 |
| `HashMap` | 允许 `null` | 允许 `null` | 否 |

- 去重用 `Set`，不要用 `List.contains()` 遍历判断。
- 返回集合：Service 内部方法可返回 `null`，但必须在 Javadoc 说明；对前端的列表接口一律返回空集合/空数组，不返回 `null`。

Stream：

- 已知容量时 `new ArrayList<>(list.size())`。
- Stream 只做无副作用转换；禁止在 `map` 中写库、发 MQ。
- 禁止用 `parallelStream` 处理 IO/数据库/远程调用。
- 收集统一 `Collectors.toList()`（JDK 8 无 `Stream.toList()`）。

## 8. 控制语句与参数校验

- `switch` 每个 `case` 必须以 `break`/`return`/`continue` 终止，或注释说明继续执行到哪个 `case`；必须包含 `default` 且放最后。
- `switch` 的变量是 `String` 且来自外部参数时，必须先判 `null`（`switch(null)` 直接 NPE）。
- `if/else/for/while/do` 一律加大括号，即使只有一行。
- 三目运算符注意自动拆箱 NPE：任一分支为原始类型或两分支类型不一致时会拆箱，`flag ? a * b : c`（`c` 为 `null` 的 `Integer`）会抛 NPE。
- 高并发场景禁止用"等于"作为中断/退出条件（数量可能被并发写成负数导致永不退出），用大于/小于区间判断。
- `if-else` 不超过 3 层；超过时用卫语句、策略模式、状态模式改写。
- 复杂条件判断抽成有语义的布尔变量，不在 `if` 里堆多重与或非。
- 不在条件表达式或算术表达式中插入赋值语句。
- 避免取反逻辑：写 `if (x < 628)`，不写 `if (!(x >= 628))`。
- 循环体内不做对象创建、连接获取、可外提的 `try-catch`。
- 方法超过 10 行时，`return`/`throw` 所在的右大括号后加一个空行。
- 公开接口必须做入参保护：批量接口限制数量上限（如一次 ≤ 500 个 ID），防止调用方传超大数组打爆内存。
- 需要参数校验的场景：对外 HTTP/RPC 接口、敏感权限入口、低频率重方法、要求高可用的方法。
- 可省略校验的场景：DAO 层（同进程内已被 Service 校验）、仅本类调用的 `private` 方法。

## 9. 并发与锁

- 单例、工具类、资源驱动类必须线程安全。
- `ThreadLocal` 必须 `static` 修饰，并在 `finally` 中 `remove()`（线程池复用会导致脏数据与内存泄漏）。
- `ThreadLocal` 只解决线程内共享，不解决共享对象更新问题。
- 创建线程必须指定有意义的线程名（含业务分组前缀），便于 `jstack` 回溯。
- 线程资源必须走线程池，禁止 `new Thread(...)`、`Executors.newFixedThreadPool`/`newCachedThreadPool`；用 `ThreadPoolExecutor`/`ThreadPoolTaskExecutor` 显式设置有界队列与拒绝策略。
- `SimpleDateFormat` 线程不安全，禁止作为 `static` 共享；用 `DateTimeFormatter`。
- 多资源同时加锁必须保持全局一致的加锁顺序，否则死锁。
- 阻塞式加锁：`lock()` 必须在 `try` 之外，且 `lock()` 与 `try` 之间不能有可抛异常的方法调用。
- 尝试式加锁：`tryLock()` 返回 `true` 才进入业务块，并在 `finally` 中 `unlock()`。
- 锁粒度：能用无锁结构就不用锁；能锁代码块就不锁方法；能用对象锁就不用类锁；锁内不调用 RPC、不发 MQ、不做大计算。
- 双重检查锁的实例字段必须 `volatile`。
- 并发修改同一记录必须加锁：应用层锁、缓存锁或数据库乐观锁（`version`）；冲突率 < 20% 用乐观锁，否则用悲观锁；乐观锁重试不少于 3 次。
- 资金、库存扣减等金融敏感场景用悲观锁，遵循"一锁、二判、三更新、四释放"。
- `volatile` 只保证可见性与有序性，`count++` 仍要用 `AtomicInteger`/`AtomicLong`，JDK 8 高并发计数用 `LongAdder`。
- 异步转同步用 `CountDownLatch`，子线程必须在 `finally` 中 `countDown()`，否则主线程只能等到超时。
- 共享 `Random` 实例会因 seed 竞争降性能，用 `ThreadLocalRandom`。
- 定时任务不用 `Timer`（一个任务抛异常会导致全部终止），用 `ScheduledExecutorService`。
- `HashMap` 高并发 `resize` 可能成环导致 CPU 飙升，并发场景用 `ConcurrentHashMap`。
- 禁止用 `synchronized` 锁 `String` 常量、包装类、公开可变的类对象。

## 10. Lombok 与对象转换

允许：

- DTO/Resp：`@Data`（或 `@Getter @Setter` + 显式构造器）。
- Entity：`@Getter @Setter @ToString`，不要 `@Data`，避免生成不合适的 `equals/hashCode`。
- Service 实现：`@Slf4j`、`@RequiredArgsConstructor`。
- 继承场景：`@EqualsAndHashCode(callSuper = true)`。

约束：

- Req 必须保留无参构造器供 Jackson 反序列化。若用 `@Builder`，同时加 `@NoArgsConstructor`、`@AllArgsConstructor`。
- 禁止 `@SneakyThrows`、`@Cleanup`、字段 `@Autowired`。
- 禁止用 Lombok 的 `@Value` 替代业务对象；不可变对象按需手写。
- 对象转换统一使用 Hutool `BeanUtil`，Convert 类标 `@Component`，由 Spring 注入；不要引入 MapStruct 或另写反射工具。
- 入参转 Entity 必须用 `CopyOptions` 忽略 `id`、`userId`、`status`、审计与逻辑删除字段，防止请求体越权赋值。
- 字段名不一致、枚举/时间/金额需要格式化时显式转换，不依赖自动复制的隐式行为。
- Entity 的注解顺序固定：`@TableName` → `@TableId` → 字段注解。

```java
package com.acme.mall.convert;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.bean.copier.CopyOptions;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.entity.OrderEntity;
import com.acme.mall.vo.OrderResp;
import org.springframework.stereotype.Component;

@Component
public class OrderConvert {

    private static final CopyOptions CREATE_REQ_TO_ENTITY = CopyOptions.create()
            .setIgnoreProperties("id", "orderNo", "userId", "status",
                    "createTime", "updateTime", "createBy", "updateBy", "deleted", "version");

    public OrderEntity toEntity(OrderCreateReq req) {
        return BeanUtil.toBean(req, OrderEntity.class, CREATE_REQ_TO_ENTITY);
    }

    public OrderResp toResp(OrderEntity entity) {
        return entity == null ? null : BeanUtil.toBean(entity, OrderResp.class);
    }
}
```

`hutool-all` 已包含 `BeanUtil`；不需要 MapStruct processor 或 `lombok-mapstruct-binding`。

## 11. 依赖注入与 Bean

- 只使用构造器注入，字段 `private final`；单构造器不需要 `@Autowired`。
- `@Service` 标在实现类，接口不加。
- `@RestController`、`@Configuration`、`@Mapper` 各司其职，不混用 `@Component`。
- 禁止循环依赖；不要开启 `spring.main.allow-circular-references`。
- 禁止在 Controller、Util、Config 中注入 Mapper。
- 工具类使用 `private` 构造器 + `static` 方法，禁止被 Spring 实例化。
- `@Transactional` 基于代理，类内自调用不生效；需要事务的调用必须经过代理（拆分 Bean 或上移调用）。

## 12. Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public Result<Long> create(@Valid @RequestBody OrderCreateReq req) {
        return Result.success(orderService.create(req));
    }

    @GetMapping("/{id}")
    public Result<OrderResp> get(@PathVariable @Positive Long id) {
        return Result.success(orderService.get(id));
    }

    @GetMapping
    public Result<PageResp<OrderResp>> page(@Valid OrderQueryReq req) {
        return Result.success(orderService.page(req));
    }

    @PostMapping("/{id}/cancel")
    public Result<Void> cancel(@PathVariable @Positive Long id) {
        orderService.cancel(id);
        return Result.success();
    }
}
```

- Controller 只注入 `XxxService` 接口，禁止注入 `XxxServiceImpl`。
- Controller 不写业务分支、不 try/catch、不开启事务、不返回 Entity。
- 路径参数用 `@Positive` 校验；类上必须 `@Validated` 才生效。
- Body 用 `@RequestBody` + `@Valid`；Query 对象用 `@Valid`，日期字段加 `@DateTimeFormat`。
- 统一返回 `Result<T>`；特殊下载/流式接口除外。
- 一个 HTTP 方法对应一个业务动作；复杂动作放子路径（`/{id}/cancel`），不要在基础路径堆动词。

## 13. Service

业务 Service 固定接口 + 实现：接口放 `service`，实现放 `service/impl`；Controller 只依赖接口。

```java
package com.acme.mall.service;

import com.acme.mall.common.api.PageResp;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.dto.OrderQueryReq;
import com.acme.mall.vo.OrderResp;

public interface OrderService {

    Long create(OrderCreateReq req);

    OrderResp get(Long id);

    PageResp<OrderResp> page(OrderQueryReq req);

    void cancel(Long id);
}
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderMapper orderMapper;
    private final OrderConvert orderConvert;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long create(OrderCreateReq req) {
        OrderEntity entity = orderConvert.toEntity(req);
        entity.setOrderNo("O" + IdWorker.getIdStr());
        entity.setStatus(OrderStatus.WAIT_PAY);
        try {
            orderMapper.insert(entity);
        } catch (DuplicateKeyException e) {
            throw new BizException(OrderErrorCode.ORDER_NO_DUPLICATED);
        }
        log.info("order created, orderId={}, orderNo={}", entity.getId(), entity.getOrderNo());
        return entity.getId();
    }

    @Override
    public OrderResp get(Long id) {
        OrderEntity entity = orderMapper.selectById(id);
        if (entity == null) {
            throw new BizException(OrderErrorCode.ORDER_NOT_FOUND);
        }
        return orderConvert.toResp(entity);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void cancel(Long id) {
        OrderEntity entity = orderMapper.selectById(id);
        if (entity == null) {
            throw new BizException(OrderErrorCode.ORDER_NOT_FOUND);
        }
        if (entity.getStatus() != OrderStatus.WAIT_PAY) {
            throw new BizException(OrderErrorCode.ORDER_STATUS_NOT_ALLOWED);
        }
        OrderEntity update = new OrderEntity();
        update.setId(id);
        update.setStatus(OrderStatus.CANCELED);
        update.setVersion(entity.getVersion());
        int rows = orderMapper.updateById(update);
        if (rows == 0) {
            throw new BizException(OrderErrorCode.ORDER_CONCURRENT_MODIFIED);
        }
        log.info("order canceled, orderId={}", id);
    }
}
```

规则：

- 业务 Service 强制 `XxxService` 接口 + `service/impl/XxxServiceImpl` 实现；Controller、其他 Bean 只依赖接口。
- `@Service`、`@Transactional` 只在实现类；接口不标注解，接口方法不出现 Entity。
- 单元测试直接测 `XxxServiceImpl`；Controller 切片测试用 `@MockBean` mock `XxxService` 接口。
- 本文的 `ServiceImpl` 是业务实现类，不是 MyBatis-Plus 的 `ServiceImpl`；简单 CRUD 用 `BaseMapper`，不要用 MP `IService` 代替业务接口。
- 写方法统一 `@Transactional(rollbackFor = Exception.class)`；多语句只读查询可用 `readOnly = true`，单条查询不开事务。
- 业务状态流转必须显式校验，枚举比较用 `==`，`Long`/`String` 比较用 `equals`。
- 不存在用 `getXxx` 抛 `ORDER_NOT_FOUND`；可空查找用 `findXxx` 并明确返回 `Optional` 或 `null`。
- 捕获可预期异常（如 `DuplicateKeyException`）并翻译为业务异常；禁止空 catch。
- 乐观锁更新必须检查影响行数，`0` 行抛并发冲突。
- 幂等：先查唯一键 + 唯一索引兜底；外部重复请求使用幂等键，不能只靠"先查后插"。
- 查询条件用 `LambdaQueryWrapper`，只允许出现在 Service/Mapper 层；宽表查询用 `.select(...)` 裁剪字段。
- 分页统一转换为 `PageResp`，不把 MyBatis-Plus 的 `Page` 直接返回给前端。

```java
@Override
public PageResp<OrderResp> page(OrderQueryReq req) {
    LambdaQueryWrapper<OrderEntity> wrapper = Wrappers.<OrderEntity>lambdaQuery()
            .eq(OrderEntity::getUserId, UserContext.requireUserId())
            .eq(req.getStatus() != null, OrderEntity::getStatus, req.getStatus())
            .ge(req.getStartTime() != null, OrderEntity::getCreateTime, req.getStartTime())
            .lt(req.getEndTime() != null, OrderEntity::getCreateTime, req.getEndTime())
            .orderByDesc(OrderEntity::getCreateTime)
            .orderByDesc(OrderEntity::getId);

    Page<OrderEntity> page = orderMapper.selectPage(
            new Page<>(req.getPageNum(), req.getPageSize()), wrapper);

    List<OrderResp> records = page.getRecords().stream()
            .map(orderConvert::toResp)
            .collect(Collectors.toList());
    return PageResp.of(records, page.getTotal(), page.getCurrent(), page.getSize());
}
```

普通用户接口的 `userId` 条件来自登录上下文；后台查询另建接口并单独授权，不能直接复用该条件。

## 14. Mapper

```java
@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {

    OrderEntity selectByOrderNo(@Param("orderNo") String orderNo);
}
```

- 简单 CRUD 用 `BaseMapper`；多表、聚合、动态复杂条件才写 XML。
- 多参数必须 `@Param`；SQL 中引用参数名，不依赖 `arg0/param1`。
- `selectOne` 必须确保条件唯一，否则用 `selectList` 或加 `LIMIT 1`。
- `updateById` 默认忽略 `null` 字段；需要把字段更新为 `NULL` 时用 `LambdaUpdateWrapper.set`。
- 批量操作使用 `saveBatch` / `updateBatchById`，单批 500 条左右，禁止在循环中逐条写库。
- Mapper 接口不写业务判断，复杂状态条件在 Service 组装。
- XML 手写 SQL 必须显式处理逻辑删除：`AND deleted = 0`。
- 禁止用 `HashMap`/`Hashtable` 接收查询结果；`bigint` 在不同驱动版本下可能映射为 `Long` 或 `BigInteger`。
- 数据订正类 SQL 先 `SELECT` 确认范围，再执行 `UPDATE`/`DELETE`。

## 15. Entity

```java
@Getter
@Setter
@ToString
@TableName("trade_order")
public class OrderEntity {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String orderNo;
    private Long userId;
    private BigDecimal amount;
    private OrderStatus status;
    private String remark;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)
    private Long createBy;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updateBy;

    @TableLogic
    private Integer deleted;

    @Version
    private Integer version;
}
```

- Entity 与表字段一一对应，不做业务计算、不写业务方法、不作为接口出入参。
- 业务枚举字段直接声明为枚举类型（实现 `IEnum<String>`），不要用 `Integer`/`String` 裸字段。
- 大字段（正文、JSON、附件）非必要不放进主 Entity；确需同表时用 `@TableField(select = false)` 避免默认查询。
- 审计字段通过 `MetaObjectHandler` 填充；启用登录上下文时同时填充 `createBy`/`updateBy`，否则由数据库默认 0 兜底。
- 不使用 `@Data`、不实现 `Serializable`（除非框架要求）。
- 禁止在 Entity 上直接加 `@JsonFormat`、`@JsonSerialize` 等接口层注解。
- 更新操作必须同时刷新 `update_time`（由 `MetaObjectHandler` 保证，不要绕过）。

业务枚举写法：

```java
public enum OrderStatus implements IEnum<String> {

    WAIT_PAY("WAIT_PAY", "待支付"),
    PAID("PAID", "已支付"),
    CANCELED("CANCELED", "已取消"),
    COMPLETED("COMPLETED", "已完成");

    private final String code;
    private final String desc;

    OrderStatus(String code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    @Override
    @JsonValue
    public String getValue() {
        return code;
    }

    public String getDesc() {
        return desc;
    }
}
```

- code 与枚举名保持一致，Jackson 可直接按名称反序列化。
- 若 code 与枚举名不同，必须补 `@JsonCreator` 静态方法，否则请求体反序列化失败。
- 数据库列必须注释全部 code；禁止在 Service 中散落魔法字符串。
- 枚举常量逐个写注释；二方库接口的返回值不使用枚举类型（避免调用方升级即不兼容）。

## 16. DTO 与 Resp

```java
@Data
public class OrderCreateReq {

    @NotNull(message = "金额不能为空")
    @DecimalMin(value = "0.01", message = "金额必须大于0")
    @Digits(integer = 16, fraction = 2, message = "金额最多两位小数")
    private BigDecimal amount;

    @Size(max = 500, message = "备注最多500字")
    private String remark;
}
```

用户/租户 ID 从登录上下文获取，不放在请求体；后台代下单接口另建 Req 并单独鉴权。

- 格式、长度、范围、非空用 JSR-303 注解；业务规则（状态是否允许、余额是否足够）放 Service。
- 校验提示必须是可读中文，不能把异常类名返回给前端。
- 嵌套对象加 `@Valid`；集合元素校验用 `List<@Valid ItemReq>`。
- 新增与更新使用不同 Req；更新必须携带 `id`，并发更新携带 `version`。
- Resp 只包含前端需要的字段，禁止返回 Entity、密码、密钥、内部状态字段。
- 禁止用 `Map<String, Object>` 作为请求或响应体，也不作为跨层传输的查询条件。
- Query Req 继承统一 `PageReq`，日期参数加 `@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")`。

## 17. JDK 8 兼容红线

以下 API/语法在 JDK 8 基线下禁止使用：

| 禁止 | 替代 |
|---|---|
| `var` | 显式类型 |
| `List.of` / `Map.of` / `Set.of` | `Arrays.asList` / `Collections.unmodifiableList` / `new HashMap<>()` |
| `Optional.isEmpty()` | `!optional.isPresent()` |
| `String.isBlank()` / `strip()` / `repeat()` | `StringUtils.isBlank` / `trim()` / 手写循环 |
| `Stream.toList()` | `collect(Collectors.toList())` |
| `Collectors.toUnmodifiableList()` | `Collections.unmodifiableList(collect(...))` |
| `Files.readString` / `writeString` | `Files.readAllBytes` + `new String(bytes, UTF_8)` |
| `InputStream.readAllBytes()` | `ByteArrayOutputStream` 循环读取 |
| `record` / `sealed` / 文本块 | 普通类 / final 类 / 字符串拼接 |
| switch 表达式、箭头 case | 传统 `switch` |
| `instanceof` 模式匹配 | 显式强转 |
| `java.net.http.HttpClient` | `RestTemplate` / OkHttp |
| 私有接口方法、匿名类钻石语法 | JDK 8 兼容写法 |

项目即便运行在 JDK 11，也默认遵守本表；确需使用 11 API 时必须全项目统一并在 pom 中声明。

## 18. 异常与日志

- 业务失败只抛 `BizException`，不要自定义大量异常类。
- 禁止 `e.printStackTrace()`、禁止空 catch、禁止捕获 `Throwable` 后不处理。
- 不用异常控制流程：可预检查的 NPE、越界用 `if` 判断，不用 `catch` 兜。
- 日志使用 SLF4J 占位符；异常对象作为最后一个参数传入。
- 详细规则见 `exception-logging.md`。

## 19. 静态检查

- IDE 安装 Alibaba Java Coding Guidelines（p3c）与 SonarLint，提交前无阻断级告警。
- 禁止提交注释掉的代码、调试输出、临时开关。
- 构建必须通过 `mvn clean verify`；格式与 import 由 IDE 统一配置，不手工混排。

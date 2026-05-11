# Java 单元测试与质量保障 · 面试题汇总

> 本篇覆盖 JUnit / Mockito / Spring Boot 测试 / 覆盖率 / TDD 等 20 个高频考点，每题包含题目、核心答案、追问方向、避坑提示。

---

## 1. JUnit 4 vs JUnit 5 区别

### 题目
JUnit 4 和 JUnit 5 在注解层面有哪些核心区别？`@Before`、`@After`、`@BeforeClass`、`@AfterClass`、`@Test` 在 JUnit 5 中分别被什么替代？

### 核心答案

| JUnit 4 注解 | JUnit 5 对应 | 说明 |
|---|---|---|
| `@Test` | `org.junit.jupiter.api.Test` | JUnit 5 的 `@Test` 不再接受 `expected` / `timeout` 参数，移至独立注解 |
| `@Before` | `@BeforeEach` | 每个测试方法**前**执行 |
| `@After` | `@AfterEach` | 每个测试方法**后**执行 |
| `@BeforeClass` | `@BeforeAll` | 必须配合 `static` 方法（或 Kotlin object / JUnit 5 的 `@TestInstance(PER_CLASS)`） |
| `@AfterClass` | `@AfterAll` | 同上 |
| `@Ignore` | `@Disabled` | 禁用某个测试 |
| `@Category` | `@Tag` | 测试分组 |
| `@Rule` / `@ClassRule` | `@ExtendWith` + `Callback` | 扩展机制完全重写 |

**关键变化：**
- JUnit 5 由 **Platform** + **Jupiter** + **Vintage** 三层构成，支持同时运行 JUnit 4 的遗留测试。
- `@Test(timeout = 5000)` 在 JUnit 5 中改为 `Assertions.assertTimeout(Duration.ofSeconds(5), …)`。
- `@Test(expected = Exception.class)` 改为 `Assertions.assertThrows(Exception.class, …)`。

### 追问方向
- JUnit 5 的 `@TestInstance(Lifecycle.PER_CLASS)` 解决了什么问题？什么时候用 `PER_METHOD` vs `PER_CLASS`？
- 如何在 Maven/Gradle 中配置同时兼容 JUnit 4 和 JUnit 5（`junit-vintage-engine`）？
- `@RegisterExtension` 与 `@ExtendWith` 的区别是什么？

### 避坑提示
- 切勿在 JUnit 5 中混用 JUnit 4 的 `@Rule`（除非通过 `VintageEngine`），扩展机制不兼容。
- `@BeforeAll` / `@AfterAll` 默认要求 `static`，忘记加 `static` 会报 runtime 异常，错误信息容易误导。
- 导入包要区分 `org.junit.Test`（Junit 4）和 `org.junit.jupiter.api.Test`（Junit 5），编译器不会报错但运行不起来。

---

## 2. JUnit 5 新特性

### 题目
除了注解更名，JUnit 5 带来了哪些 Jupiter 独有特性？`@DisplayName`、`@Nested`、`@ParameterizedTest`、`@RepeatedTest` 分别用于什么场景？

### 核心答案

**`@DisplayName`**
```java
@DisplayName("计算器相加测试")
```
为测试类或方法指定中文/任意字符串展示名，测试报告更可读，不影响测试逻辑。

**`@Nested`**
```java
class CalculatorTest {
    @Nested
    @DisplayName("相加操作")
    class AddTests {
        @Test @DisplayName("正数相加")
        void addPositive() { ... }
        @Test @DisplayName("负数相加")
        void addNegative() { ... }
    }
}
```
支持**嵌套测试类**，logical grouping，报告结构更清晰；嵌套类内部可以使用 `@BeforeEach` 等。

**`@ParameterizedTest`**
```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
@DisplayName("参数化测试：输入 {0} 乘 2 等于 {1}")
void multiplyTest(int input) {
    assertEquals(input * 2, multiplyBy2(input));
}
```
用不同入参**多次执行**同一测试，数据来源支持 `@ValueSource`、`@CsvSource`、`@EnumSource`、`@MethodSource` 等。

**`@RepeatedTest`**
```java
@RepeatedTest(value = 10, name = "{displayName} 第 {currentRepetition} 次")
void repeatTest() { ... }
```
将测试重复执行固定次数，常用于验证随机性场景或压力测试局部逻辑。

**其他重要新特性：**
- **动态测试** `DynamicTest.dynamicTest(...)`：运行时根据数据动态生成测试。
- **标签组** `@Tag`：替代 Category，支持在构建阶段过滤。
- **条件执行** `@EnabledOnOs`、`@EnabledOnJre`、`@DisabledIf`：按环境跳过测试。
- **内置扩展** `Assertions.assertTimeout`：更精确的超时断言。

### 追问方向
- `@ParameterizedTest` 的 `@CsvSource` 如何映射到方法参数？CSV 格式如何处理引号和逗号？
- `@Nested` 嵌套深度有限制吗？嵌套类能访问外层类的 `@BeforeEach` 吗？
- JUnit 5 的测试实例生命周期 `PER_CLASS` 模式下，`@BeforeAll` 不需要 static，如何实现？

### 避坑提示
- `@ParameterizedTest` 需要额外依赖 `junit-jupiter-params`，在 Maven pom 中别忘了加。
- `@RepeatedTest` 的 `{displayName}` 占位符在自定义名称时要注意写法，官方推荐格式。
- `@Nested` 类默认不继承父类的 `@Tag`，需要单独标注。

---

## 3. 断言库对比

### 题目
AssertJ 的流式断言、Hamcrest 匹配器、TestNG 断言三者各有什么特点？如何选择？

### 核心答案

**AssertJ（流式断言）**
```java
assertThat(user.getName()).isNotNull()
                           .isEqualTo("Alice")
                           .startsWith("Ali")
                           .hasSize(5);
```
- **链式调用**，接近自然语言，可读性最强。
- 自动生成详细失败信息（包含实际值），调试友好。
- 支持 `softAssertions()` 批量收集错误不立即失败。
- 社区活跃，版本迭代快。

**Hamcrest（匹配器）**
```java
assertThat(user.getName(), equalTo("Alice"));
assertThat(list, hasItem(3));
assertThat(map, hasKey("key"));
```
- 使用 **Matcher** 对象组合复杂断言，`assertThat(actual, matcher)` 模式。
- 可扩展性强，自定义 `TypeSafeMatcher` 即可。
- JUnit 4 内置支持（`assertThat` 来自 Hamcrest-core）。
- 语法啰嗦，IDE 自动补全支持弱。

**TestNG 断言**
```java
Assert.assertEquals(actual, expected);
Assert.assertTrue(condition, "message");
Assert.fail("unreachable");
```
- 静态方法风格，简单直接。
- 支持依赖测试 `dependsOnMethods`。
- 内置 DataProvider（类似 JUnit 5 `@ParameterizedTest`）。

**对比总结**

| 特性 | AssertJ | Hamcrest | TestNG |
|---|---|---|---|
| 链式/可读性 | ★★★★★ | ★★★ | ★★ |
| 失败信息详细度 | ★★★★★ | ★★★★ | ★★★ |
| 自定义成本 | 低 | 中 | 高 |
| JUnit 5 集成 | 原生 | 需额外依赖 | 不兼容（用 JUnit Jupiter） |
| 软断言 | `SoftAssertions` | 无（需第三方） | `SoftAssert` |

### 追问方向
- 如何在 Spring Boot 项目中统一使用 AssertJ？它的 `AbstractMixin` 和 `BDDAssertions` 类区别是什么？
- Hamcrest 的 `MatcherAssert.assertThat` 和 JUnit 的 `Assertions.assertThat` 有什么区别？
- `SoftAssertions` 的 `assertAll()` 为什么要显式调用？不调用会怎样？

### 避坑提示
- AssertJ 3.x 之后包名从 `org.assertj.core` 变为标准包，迁移时注意 import。
- Hamcrest 2.x 后把 core 和 contrib 分开了，有些 matcher 需要额外引包。
- 不要在一个项目里混用多种断言风格（保持团队代码风格统一）。

---

## 4. Mockito 框架核心

### 题目
Mockito 中 `@Mock`、`@Spy`、`@InjectMocks` 注解的作用是什么？`when().thenReturn()` 的完整调用链是怎样的？

### 核心答案

**`@Mock`**
```java
@Mock
private UserService userService;
```
创建一个**完全的模拟对象**，所有方法默认返回默认值（null / 0 / false / 空集合），方法体不执行。

**`@Spy`**
```java
@Spy
private List<String> list = new ArrayList<>();
```
创建一个**部分模拟对象**，默认调用**真实方法**，可选择性 stub 特定方法。

**`@InjectMocks`**
```java
@InjectMocks
private OrderService orderService;
```
自动将 `@Mock` / `@Spy` 标注的依赖注入到被标注字段。Mockito 会按类型或名字匹配，支持构造函数注入、setter 注入、字段注入。

**`when().thenReturn()` 完整链**
```java
// 普通返回值
when(mockService.findById(1L)).thenReturn(user);

// 连续打桩（第三次调用返回最后一个值）
when(mockService.findById(1L))
    .thenReturn(user1)
    .thenReturn(user2)
    .thenThrow(new RuntimeException("not found"));

// doReturn-when（避免 spy 时调用真方法）
doReturn(user).when(spyList).get(0);

// 泛型兼容 thenAnswer
when(mockService.findById(anyLong()))
    .thenAnswer(inv -> (Long) inv.getArgument(0) + 1);
```

**必须调用 `MockitoAnnotations.openMocks(this)` 或使用 `@ExtendWith(MockitoExtension.class)` 才能激活注解。**

### 追问方向
- `@InjectMocks` 的注入优先级是怎样的？当有多个同类型 `@Mock` 时如何区分？
- `when(mock.method()).thenReturn(x)` vs `doReturn(x).when(mock).method()` 有什么区别？什么时候必须用后者？
- `@Mock` 的 `answer` 参数（比如 `Answer.CALLS_REAL_METHODS`）和 `@Spy` 有什么区别？

### 避坑提示
- `@Mock` 对象的方法默认返回**默认值**，不会抛异常。忘记 stub 会得到 null 或 0，容易引入隐蔽 bug。
- `@InjectMocks` 如果构造函数有多个参数，Mockito 可能注入失败，优先用带默认构造函数的类。
- `when(...).thenThrow()` 中如果实际调用没有匹配到参数，抛的是 ` org.mockito.exceptions.verification.WantedButNotInvoked`，不是业务异常。

---

## 5. Mock vs Spy

### 题目
Mockito 中 Mock 和 Spy 的本质区别是什么？各自适用于什么场景？

### 核心答案

| 维度 | `@Mock` | `@Spy` |
|---|---|---|
| 方法执行 | **不执行**，全部 stub | **默认执行真实方法** |
| 状态 | 无状态 | 有真实状态，可变 |
| 返回值 | 默认返回值（null/0/false） | 默认返回真实结果 |
| 典型场景 | 隔离被测类，专注被测逻辑 | 验证部分行为，保留大部分真实逻辑 |
| 性能 | 快（无反射调用） | 稍慢（代理真实对象） |

**使用原则：**
- **用 Mock**：当你只想测试 A 类，而 A 依赖 B 服务。此时 mock 掉 B，完全控制 B 的行为，A 的每个分支都可以被独立验证。
- **用 Spy**：当你需要**验证**某个对象方法的调用次数/顺序，或想 stub 少数几个方法而其他方法保持真实。最典型：监控一个集合对象是否按预期被操作。

```java
// Mock：只关心 login 是否被调用，不关心内部实现
@Mock
private AuthService authService;
@Test
void shouldCallAuthService() {
    service.login("alice", "pass");
    verify(authService).login("alice", "pass");
}

// Spy：想保留 List 的真实行为，只验证某个操作
@Spy
private List<String> list = new ArrayList<>();
@Test
void shouldAddItem() {
    list.add("item");
    assertThat(list).hasSize(1);  // 真实 add 生效
    verify(list).add("item");
}
```

### 追问方向
- `@Spy` 包装的是真实对象还是代理？它的实现原理是什么（ByteBuddy/ASM/CGlib）？
- Spy 对象调用构造函数时 `new SpyBean()` 和 `@Spy Bean bean` 有什么区别？
- `lenient()` stub 在 Mock 和 Spy 上分别什么时候用到？

### 避坑提示
- Spy 永远不要 stub 大量方法，否则就是在写"伪装的 Mock"，应该直接用 `@Mock`。
- 在 JUnit 5 + MockitoExtension 下，`@Spy` 必须提供默认初始化值（`= new ArrayList<>()`），否则报 `org.mockito.exceptions.base.MockitoException: Unable to create spy instance`。
- 不要对 `final` 类或 `final` 方法使用 Spy，Mockito 默认不支持（需要 `mock-maker-inline` 扩展）。

---

## 6. Spring Boot 测试注解

### 题目
`@SpringBootTest`、`@WebMvcTest`、`@DataJpaTest`、`@JsonTest` 四个注解各自的职责和适用范围是什么？

### 核心答案

**`@SpringBootTest`**
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
```
启动**完整 Spring 上下文**，所有 Bean 都加载，适合**全链路集成测试**（Service + Controller + Repository）。
- `webEnvironment` 可选 `MOCK`（默认）、`RANDOM_PORT`、`DEFINED_PORT`、`NONE`。
- 测试启动慢，测试间可能有状态污染。

**`@WebMvcTest(YourController.class)`**
```java
@WebMvcTest(UserController.class)
```
只启动 **Spring MVC 基础设施**（DispatcherServlet、Controller、ControllerAdvice、Converters），不加载 Service / Repository 层。
- 配合 `@MockBean` 注入被依赖的 Service。
- 启动快，适合**Controller 层单元测试**。
- 注意：默认只加载标注了 `@Controller` / `@ControllerAdvice` 的类。

**`@DataJpaTest`**
```java
@DataJpaTest
```
只加载 **JPA 相关组件**（EntityManager、DataSource、Repository），用内存数据库（H2 默认）替代真实 DB。
- 配合 `@AutoConfigureTestDatabase(replace = Replace.NONE)` 可连接真实数据库。
- 自动回滚每个测试方法的事务（默认行为）。
- 对 `Repository` 的 `findBy` 方法进行 CRUD 验证非常方便。

**`@JsonTest`**
```java
@JsonTest
```
只加载 **Jackson / JSON 处理相关组件**，测试 `ObjectMapper` 序列化/反序列化。
- 支持 `@JsonTest MyJacksonTest` 自定义配置。
- 可测试自定义序列化器、反序列化器。

**选择决策树：**
```
需要真实 Spring Context + 全链路？  → @SpringBootTest
只测 Controller（REST API）？       → @WebMvcTest + @MockBean
只测 JPA Repository？               → @DataJpaTest
只测 JSON 序列化？                  → @JsonTest
```

### 追问方向
- `@WebMvcTest` 如何 mock 掉 Security 配置？`@MockBean SecurityFilterChain` 够用吗？
- `@DataJpaTest` 用的 H2 内存数据库和真实 MySQL/PostgreSQL 行为有哪些差异（方言、DDL、函数）？
- `@SpringBootTest` 如何禁用 `@Configuration` 类中的某些 Bean？

### 避坑提示
- `@WebMvcTest` 如果测试的是带 `@RequestMapping` 的类而非 Controller，会报 404。
- `@DataJpaTest` 默认回滚事务，所以 `Repository.findAll().size()` 永远是 0（查不到数据）。需要显式 `@Transactional(propagation = Propagation.NOT_SUPPORTED)` 或不用 `@DataJpaTest`。
- 多个 `@SpringBootTest` 并行执行时，如果测试类成员变量用了非 static 的 `@Autowired`，注意线程安全。

---

## 7. @MockBean vs @Mock

### 题目
`@MockBean` 和 `@Mock` 都用于创建 Mock 对象，它们有什么区别？`@MockBean` 在 Spring 测试上下文中的行为是什么？

### 核心答案

| | `@Mock` | `@MockBean` |
|---|---|---|
| 所属框架 | Mockito（JUnit 测试框架无关） | Spring Boot Test（Spring Context） |
| 是否进入 Spring 上下文 | **否** | **是**（作为 Bean 注册进上下文） |
| 自动注入 | 需配合 `@ExtendWith(MockitoExtension)` | 自动替换上下文中同类型 Bean |
| 依赖 | 只需 `mockito-junit-jupiter` | 需要 `spring-boot-test` + `mockito-core` |
| 使用场景 | 纯单元测试，不需要 Spring | 集成测试，需要 Spring 上下文 |

**`@MockBean` 本质**：在 Spring `ApplicationContext` 中注册一个 **Mockito 生成的 Bean**，替换原本同类型的真实 Bean。对 Spring 上下文完全透明，所有 `@Autowired` 都会拿到这个 Mock 对象。

```java
@SpringBootTest
class OrderServiceIntegrationTest {
    @MockBean   // 替换 Spring 上下文中真实的 UserService Bean
    private UserService userService;

    @Autowired
    private OrderService orderService;  // 内部引用的 userService 是 Mock 的

    @Test
    void testOrder() {
        when(userService.findById(1L)).thenReturn(new User("alice"));
        Order order = orderService.createOrder(1L);
        assertThat(order.getUserName()).isEqualTo("alice");
    }
}
```

### 追问方向
- `@MockBean` 会不会影响同一个测试类中的其他 `@Test` 方法（状态是否隔离）？
- 当有多个 `@MockBean` 时，`@InjectMocks` 还能用吗？Spring 是如何处理多个 Mock Bean 注入的？
- `@MockBean` 的 `name` 属性有什么用？什么场景下需要指定？

### 避坑提示
- `@MockBean` 每个测试方法都会**重置**（Mockito 默认 `lenient` 模式下单测内 stub 不自动清理，但建议每个 test 用完显式 `reset()` 或使用 `@MockitoSettings(strictness = Strictness.LENIENT)`）。
- `@MockBean` 依赖于 Spring Boot Test 的自动配置，不要在普通 JUnit 5 测试（非 `@SpringBootTest`）上使用。
- `@MockBean` 替换 Bean 时，如果原 Bean 有 `@Primary`，Mockito Mock 没有，所以 `List<MyInterface>` 注入时可能拿到原 Bean 而不是 Mock。

---

## 8. 测试切片：@WebMvcTest

### 题目
为什么 `@WebMvcTest` 能大幅减少测试启动时间？它的"切片"机制是怎样的？

### 核心答案

**问题根源**：`@SpringBootTest` 加载完整上下文，100+ Bean 全部初始化，一个复杂项目启动可能需要 **30秒~2分钟**，极大拖慢 TDD 节奏。

**`@WebMvcTest` 切片原理**：
Spring Boot Test 提供了一系列 **测试切片注解**（`@WebMvcTest`、`@DataJpaTest`、`@JsonTest` 等），每个切片只激活**特定层**所需的自动配置类（`AutoConfiguration`），其余全部跳过。

```java
// @WebMvcTest 实际做的事（伪代码）
@AutoConfigureWebMvc  // 只加载 Web MVC 相关配置
@ImportAutoConfiguration  // 仅这些：ErrorMvcAutoConfiguration,
//  JacksonHttpMessageConverters, WebMvcAutoConfiguration ...
// 排除了：DataSourceAutoConfiguration, JpaAutoConfiguration,
//  SecurityAutoConfiguration ...
```

**组件加载对比**：

| 组件 | @SpringBootTest | @WebMvcTest |
|---|---|---|
| Controller | ✅ | ✅ |
| Service | ✅ | ❌（用 `@MockBean` 替代） |
| Repository | ✅ | ❌ |
| DataSource | ✅ | ❌ |
| Security | ✅ | ❌（需手动 @MockBean） |
| 启动时间 | 30s~2min | 3~8s |

**`@WebMvcTest` 的限制**：
- 只能测试**带 `@Controller` / `@RestController` 注解**的类。
- 方法体中如果调用了 `@Service` Bean，必须用 `@MockBean` 注入，否则 `NullPointerException`。
- 不加载 `Interceptor`、`Filter`（除非在测试中手动注册）。

### 追问方向
- `@WebMvcTest` 配合 `@MockBean` 后，如何验证 `Mock` 对象没被调用（检查 stub 数量）？
- `@WebMvcTest` 切片是否支持测试 `@ControllerAdvice` 全局异常处理？
- Spring Boot 2.x 和 3.x 的 `@WebMvcTest` 在 Security 测试上有什么差异（Spring Security 5 vs 6）？

### 避坑提示
- `@WebMvcTest` 默认不会加载 `application.yml` 中的配置，建议配合 `@TestConfiguration` 手动注入测试所需的 `ObjectMapper` 等。
- 如果 Controller 依赖了自定义注解（比如 `@Valid`），确保相关 converter/validator 已被加载。
- 切片测试写多了容易出现"只测了 Controller，Service 层没人管"的情况，建议配合 `@SpringBootTest` 做少量关键路径的集成测试。

---

## 9. 集成测试 vs 单元测试

### 题目
集成测试和单元测试的核心区别是什么？什么时候该选单元测试，什么时候该选集成测试？

### 核心答案

**单元测试（Unit Test）**
- **被测对象**：最小的独立代码单元（通常是类的一个方法）。
- **依赖关系**：通过 Mock 隔离外部依赖（Service mock 掉 Repository）。
- **特点**：运行快（毫秒级）、可重复、无副作用、定位问题精准。
- **典型工具**：JUnit 5 + Mockito。

**集成测试（Integration Test）**
- **被测对象**：多个组件协作的完整功能（Controller → Service → Repository → DB）。
- **依赖关系**：使用真实组件或 TestContainers 等轻量替代。
- **特点**：能发现 Mock 隔离不了的 bug（SQL 错误、事务边界、序列化问题、HTTP 层协议问题）。
- **典型工具**：`@SpringBootTest`、`REST Assured`、`TestContainers`。

**何时选单元测试**
- 验证纯算法/业务逻辑（计算税率、字符串处理）。
- 依赖外部系统但需要稳定快速反馈。
- TDD 开发阶段，先写单元测试快速验证设计。

**何时选集成测试**
- REST API 端到端（Controller → Service → JSON 序列化）。
- 数据库读写、事务传播、多数据源。
- 与第三方系统集成（消息队列、外部 API）。
- 发现单元测试覆盖率很好但线上仍出 bug 的"盲区"。

**经典比例（业界经验）**：单元测试 : 集成测试 ≈ **70% : 30%**（按数量），覆盖重在单元，质量重在集成。

### 追问方向
- "金字塔策略"（Test Pyramid）是什么？为什么底层单元测试要多做？
- 集成测试中使用 H2 内存数据库替代 MySQL，有什么陷阱？
- 集成测试中如何处理事务回滚，避免测试数据污染？

### 避坑提示
- 用 Mock 过度隔离会导致"Mock 测试"——只验证了调用顺序，不验证真实行为，本末倒置。
- 集成测试启动慢，尽量不要在每个小改动时跑全量 `@SpringBootTest`，用 Maven profile 区分 `unit` 和 `integration`。
- 集成测试中的 `Thread.sleep` 是反模式，应该用 `@Transactional` 回滚或手动清理数据。

---

## 10. TDD 开发流程

### 题目
什么是 TDD（测试驱动开发）？红绿重构循环的具体步骤是什么？实际项目中如何落地？

### 核心答案

**红绿重构循环（Red-Green-Refactor）**

```
┌─────────────────────────────────────────────────────┐
│  1. 写一个 FAILING 测试（Red）                        │
│     → 编译失败或断言失败，明确"要做什么"               │
│                                                      │
│  2. 写最少量代码让测试通过（Green）                    │
│     → 不管实现多丑，先让测试变绿                       │
│                                                      │
│  3. 重构（Refactor）                                 │
│     → 消除重复、提升可读性，测试仍然通过               │
│                                                      │
│  重复循环，持续演进                                   │
└─────────────────────────────────────────────────────┘
```

**TDD 核心原则**
- **小步快走**：每个循环只做一件事，测试粒度控制在分钟级。
- **意图导向**：测试名即文档，`should_return_404_when_user_not_found` 比 `testGetUser` 好十倍。
- **测试先行**：不是"先写测试再写代码"，而是"用测试定义行为，再实现行为"。

**实际落地建议**
```bash
# 典型的 TDD 节奏（每个循环 2~5 分钟）
1. 写 testFindUserById_notFound_returns404  → RED ❌
2. 写 if (user == null) throw ...           → GREEN ✅
3. 重构：异常类型是否合适？日志是否需要？    → REFACTOR 🔄
4. 下一个循环...
```

- **工具支撑**：IDEA 自带 TDD 快捷键，测试类和方法并行创建。
- **覆盖率门禁**：覆盖率 < 80% 时不允许合并，保证 TDD 习惯不倒退。
- **团队约束**：Code Review 必须看测试是否"先行"、是否"有意义"。

### 追问方向
- TDD 和 BDD（行为驱动开发）的区别是什么？Cucumber/JBehave 在 Java 中的角色？
- 遗留代码（没有测试的老项目）如何做 TDD 改造？有哪些策略（Sprout Method / Sprout Class）？
- TDD 在团队中推行遇到阻力（"测试拖慢速度"），如何说服团队/管理层？

### 避坑提示
- 红阶段不要写"永远会过"的测试，必须明确失败的 assertion 再进入绿阶段。
- 绿阶段不要"过度实现"——TDD 的绿是"刚好通过"，不是"完美实现"。
- 重构时测试不能改，只能改生产代码。改了测试不算重构，是重写。
- 遇到"测试三角"（Test Triangulation）时：只有当存在多个等价输入时，才需要多个测试验证同一逻辑。

---

## 11. 覆盖率：JaCoCo 与合格标准

### 题目
JaCoCo 如何统计覆盖率？行覆盖率和分支覆盖率有什么区别？覆盖率多少算合格？

### 核心答案

**JaCoCo 工作原理**
JaCoCo 通过 **Java Agent** 运行时插桩，记录每个类每个分支的的执行情况，生成 `.exec` 文件，最终输出 HTML/JSON/XML 报告。

```xml
<!-- Maven 配置 -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <goals><goal>report</goal></goal>
        </execution>
        <execution>
            <goals><goal>check</goal></goal>
            <configuration>
                <rules>
                    <rule>
                        <element>CLASS</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**行覆盖率（Line Coverage）**
```
被执行的代码行数 / 总代码行数
```
指标直观，但有局限：**一行多语句**（如 `if (a && b)`）时，行覆盖无法体现分支是否全覆盖。

**分支覆盖率（Branch Coverage）**
```
被执行的分支数 / 总分支数
```
衡量每个条件分支（如 `if`、`while`、`?: `）的 true/false 是否都被测试覆盖。
- **条件覆盖**（Condition Coverage）更细：每个布尔子表达式都独立变化。
- JaCoCo 中 `BRANCH` 类型对应 **分支覆盖**，`METHOD` 对应方法覆盖。

**合格标准（业界经验值）**

| 类型 | 建议最低值 | 说明 |
|---|---|---|
| 行覆盖率 | **80%** | 核心业务类建议 90%+ |
| 分支覆盖率 | **70%** | 关键 if/else 路径 |
| 方法覆盖率 | **80%** | 每个 public 方法都要测 |
| 类覆盖率 | **90%** | 不允许有无测试的重要类 |

**覆盖率不是质量唯一标准**：100% 覆盖不代表没有 bug（可能是错误的断言），覆盖率是**下限保障**，不是充分条件。

### 追问方向
- JaCoCo 的 `exec` 文件格式是什么？CI 中如何解析并生成趋势图？
- JaCoCo 与 Cobertura、Emma 相比有什么优势？什么时候用 `agent` 模式 vs `offline` 模式？
- 什么代码是"不可测"的（lambda 捕获、private 方法）？JaCoCo 如何处理？

### 避坑提示
- 不要盲目追求 100% 覆盖率，可能导致大量无意义的"傻测试"。覆盖率只是辅助指标。
- `BRANCH` 覆盖率的 70% 是平均值，**关键业务类**（订单、支付、风控）必须单独要求更高。
- JaCoCo 无法统计**运行时动态生成的字节码**（如某些 ORM、Mock 框架），需要在配置中排除这些包。

---

## 12. 单测写到什么程度

### 题目
单元测试应该覆盖哪些内容？边界条件、异常情况、等价类划分分别指什么？测试写到"什么程度"算合格？

### 核心答案

**三个核心维度**

**① 边界条件（Boundary / Edge Cases）**
```java
// 边界测试：数组索引、长度的临界点
@Test
void shouldThrow_whenIndexNegative() {
    list.get(-1);  // 边界：-1
}
@Test
void shouldThrow_whenIndexEqualsSize() {
    list.get(list.size());  // 边界：size()
}
// 数值边界：0, 1, -1, MAX_VALUE, MIN_VALUE, 空值
```

**② 异常情况（Exception Cases）**
```java
@Test
void shouldThrowNullPointerException_whenInputNull() {
    assertThrows(NullPointerException.class,
        () -> service.process(null));
}
// 要测：IllegalArgumentException、BusinessException、第三方异常
```

**③ 等价类划分（Equivalence Partitioning）**
把输入域划分为若干"效果相同"的等价类，每个类只需测一个代表值：
```
输入：1~100 的整数
等价类1：有效 [1, 100] → 测 x=50
等价类2：无效 < 1      → 测 x=0
等价类3：无效 > 100    → 测 x=101
等价类4：无效 null     → 测 x=null
```
每个等价类取**一个代表值**即可，减少冗余测试。

**合格标准自检清单**
```
✅ 所有 public 方法都有至少一个测试
✅ 正常路径覆盖（happy path）
✅ 边界条件：0, -1, MAX, null, ""
✅ 异常路径：所有 throws 声明的异常
✅ 等价类：有效类 × 1，无效类 × 每个至少 1
✅ 测试是幂等的（可重复执行，结果一致）
✅ 测试间无依赖（不依赖执行顺序）
```

### 追问方向
- 等价类划分和边界值分析是什么关系？两者如何结合？
- 如何发现"测试盲区"——写了测试但漏掉的重要场景？有什么系统性方法（mutation testing）？
- 对于 setter/getter 这种"透明"的类，有必要写单测吗？

### 避坑提示
- 不要测"实现细节"（private 方法），要测"行为"（public API）。改 private 实现不应该破坏测试。
- 断言不足的测试是"假测试"：`assertTrue(true)` 是零价值测试。
- 测试数据要避免magic number，统一放在 `@BeforeEach` 或测试数据 Builder 中。

---

## 13. 参数化测试

### 题目
JUnit 5 的 `@ParameterizedTest` 是什么？支持哪些数据来源？如何在测试中消费不同参数？

### 核心答案

**基础用法**
```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void shouldMultiplyBy2(int input) {
    assertEquals(input * 2, multiplyBy2(input));
}
```

**常用数据来源注解**

| 注解 | 用途 | 示例 |
|---|---|---|
| `@ValueSource` | 基本类型数组、String、Class | `ints = {1,2,3}`、`strings = {"a","b"}` |
| `@CsvSource` | CSV 格式，多参数映射到方法参数 | `@CsvSource({"1,2,3", "4,5,6"})` |
| `@CsvFileSource` | 外部 CSV 文件 | `@CsvFileSource(resources = "/test-data.csv")` |
| `@EnumSource` | 枚举所有值 | `@EnumSource(Month.class)` |
| `@MethodSource` | 调用工厂方法返回 `Stream<Arguments>` | `@MethodSource("provideArguments")` |
| `@ArgumentSource` | 自定义 `ArgumentsProvider` | 用于复杂数据源 |

**`@CsvSource` 多参数映射**
```java
@ParameterizedTest
@CsvSource({
    "1,  2,  3",          // a=1, b=2, expected=3
    "10, 20, 30",         // a=10, b=20, expected=30
    "-1, 1,  0"           // a=-1, b=1, expected=0
})
void shouldAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

**`@MethodSource` 外部工厂**
```java
static Stream<Arguments> provideArguments() {
    return Stream.of(
        Arguments.of("alice@email.com", "Alice", 25),
        Arguments.of("bob@email.com",   "Bob",   30)
    );
}

@ParameterizedTest
@MethodSource("provideArguments")
void shouldCreateUser(String email, String name, int age) { ... }
```

### 追问方向
- `@CsvSource` 如何处理带引号的 CSV 字符串（如 `"hello, world"`）？
- 如何给 `@ParameterizedTest` 添加自定义显示名（包含参数值）？
- `@MethodSource` 的工厂方法一定要 static 吗？`@TestInstance(PER_CLASS)` 模式下呢？

### 避坑提示
- `@ParameterizedTest` 方法必须是 `void` 返回类型，JUnit 5 规范不接受其他返回类型。
- CSV 中空值用 `''`（两个单引号）表示，null 值用空白（不写）。
- `@ValueSource(strings = {"a", "b"})` 传入 `null` 需要额外加 `@NullSource`，不会自动传。

---

## 14. 异步测试

### 题目
在 Java 中如何测试 `CompletableFuture` 等异步代码？`CountDownLatch` 在测试中的作用是什么？

### 核心答案

**测试 `CompletableFuture`**
```java
@Test
void shouldReturnFutureResult() throws Exception {
    CompletableFuture<String> future = service.fetchData();

    // 方式1：阻塞等待（简单但有超时风险）
    String result = future.get(5, TimeUnit.SECONDS);
    assertEquals("data", result);

    // 方式2：thenAccept 回调（推荐）
    future.thenAccept(result -> assertEquals("data", result))
          .get(5, TimeUnit.SECONDS);
}
```

**`@Async` 方法测试**
```java
@Test
void shouldTestAsyncMethod() throws Exception {
    CompletableFuture<String> future = new CompletableFuture<>();
    doAnswer(inv -> {
        future.complete("async result");
        return null;
    }).when(mockService).asyncCall();

    // 触发异步调用
    service.triggerAsync();

    // 等待并断言
    assertEquals("async result", future.get(3, TimeUnit.SECONDS));
}
```

**CountDownLatch 用法**
```java
@Test
void shouldTestListenerPattern() throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(1);

    listener.onEvent("test", () -> latch.countDown());

    // 最多等3秒，超时则失败
    boolean completed = latch.await(3, TimeUnit.SECONDS);
    assertTrue(completed, "事件未在超时前触发");
}
```
`CountDownLatch` 核心作用：**主测试线程等待一个或多个并发任务完成**。
- `countDown()` 被调用次数达到构造时传入的值，`await()` 才放行。
- 适用于：事件监听、回调、多线程协作场景。

**AssertJ 异步测试辅助**
```java
@Test
void shouldTestAsync() {
    await().atMost(5, TimeUnit.SECONDS)
           .untilAsserted(() ->
        assertThat(service.getData()).isEqualTo("expected")
    );
}
// 需要 assertj-async 依赖
```

### 追问方向
- `CompletableFuture.get()` 和 `join()` 有什么区别？测试中用哪个更好？
- `CountDownLatch` vs `Phaser` vs `CyclicBarrier` 的选择场景是什么？
- `@Async` 方法在 `@SpringBootTest` 中如何测试？需要开启什么配置？

### 避坑提示
- `CompletableFuture.get()` 是**阻塞调用**，不要在已配置好的 `@SpringBootTest` 并行执行环境中滥用。
- `CountDownLatch` 是一次性的，用完不能重置；`CyclicBarrier` 才能重置。
- 测试异步代码时**不要用 `Thread.sleep` 轮询**，要用更可靠的 `CompletableFuture.get(timeout)` 或 `CountDownLatch`。

---

## 15. 数据库测试

### 题目
Spring Boot 如何做数据库层测试？`@DataJpaTest`、`@AutoConfigureTestDatabase`、`TestContainers` 各自的作用是什么？

### 核心答案

**`@DataJpaTest`**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    @Autowired
    private UserRepository repository;

    @Test
    void shouldSaveAndFindById() {
        User user = repository.save(new User("alice"));
        assertThat(repository.findById(user.getId()).get().getName())
            .isEqualTo("alice");
    }
}
```
- 只加载 JPA 相关组件，使用 H2 内存数据库（默认）。
- 每个测试方法默认在**事务中**执行，测试完成后**自动回滚**，不污染 DB。
- 配合 `@DirtiesContext` 可以控制事务策略。

**`@AutoConfigureTestDatabase`**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)  // 不替换，用真实数据库
```
控制是否用测试数据库替换真实数据库：
- `Replace.NONE`：使用 `application.yml` 中配置的数据库。
- `Replace.ANY`（默认）：用 H2 内存库替换。

**TestContainers（真实数据库测试）**
```java
@Testcontainers
@SpringBootTest
class UserRepositoryIT {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("root")
        .withPassword("secret");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
}
```
- 在 Docker 容器中启动**真实 MySQL/PostgreSQL/MongoDB**，而非 H2 模拟。
- 完全兼容生产数据库方言（DDL、SQL 函数、字符集）。
- 测试间隔离：每个测试类启动独立容器，测试结束销毁。

**选择策略**
```
测试 Repository 基础 CRUD     → @DataJpaTest + H2
测试 SQL 方言/性能/索引        → TestContainers + 真实 DB
CI 流程（必须稳定）            → TestContainers（避免 H2 差异）
```

### 追问方向
- H2 模拟 MySQL 模式（`MODE=MySQL`）有哪些已知差异（函数名、DDL、字符集）？
- `@DataJpaTest` 中 `@Query` 的 nativeQuery 和 JPQL 在 H2 下行为一致吗？
- TestContainers 在 CI（GitHub Actions / Jenkins）中如何配置 Docker 权限？

### 避坑提示
- H2 兼容模式和真实 MySQL 行为有差异（如 `LIMIT` 语法、`ON DUPLICATE KEY UPDATE`），CI 测过了可能生产仍有问题。
- TestContainers 启动有延迟（约10~20秒），单测运行速度变慢，用 `@Container` + `static` 让容器在所有测试间复用。
- `@DataJpaTest` 的自动回滚对 `INSERT ... RETURNING` 序列测试无效，需要手动清理。

---

## 16. 契约测试

### 题目
什么是契约测试？Spring Cloud Contract 的 Provider 端和 Consumer 端分别要做什么？契约测试解决什么问题？

### 核心答案

**解决的问题**
微服务间通过 HTTP / 消息队列通信，Consumer 依赖 Provider 的 API。一次 Provider 的 API 变更（如字段改名）可能导致 Consumer 端隐性破坏，传统的集成测试无法覆盖。**契约测试在 Consumer 和 Provider 双方各自验证对方遵守约定的 API 格式，提前发现不兼容。**

**Spring Cloud Contract 两种模式**

**Provider 端（服务提供方）**
```groovy
// contracts/hello/HelloService.groovy
Contract.make {
    description("should return greeting")
    request {
        method GET()
        url '/greet'
        queryParameters {
            parameter name: 'name', value: equalTo('Alice')
        }
    }
    response {
        status OK()
        body([
            message: "Hello, Alice"
        ])
        headers {
            contentTypeapplicationJson()
        }
    }
}
```
Provider 端生成 **WireMock 存根**，本地启动后自动验证是否满足契约。

**Consumer 端（服务消费方）**
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@AutoConfigureMockMvc
@AutoConfigureStubRunner(ids = "com.example:hello-service:+:8080")
class GreetingClientTest {
    @Test
    void shouldConsumeGreeting() {
        // 调用本地 WireMock stub，验证 consumer 处理逻辑
        assertThat(client.greet("Bob").getMessage()).isEqualTo("Hello, Bob");
    }
}
```

**Consumer-Driven Contract（消费者驱动的契约）**
Consumer 主动定义期望，Provider 收到契约后实施。是更推荐的做法。

**契约测试 vs 传统测试**

| | 单元测试 | 集成测试 | 契约测试 |
|---|---|---|---|
| 范围 | 单类 | 服务间 | 接口兼容性 |
| 依赖 | Mock | 真实服务 | WireMock stub |
| 速度 | 最快 | 慢 | 中 |

### 追问方向
- Spring Cloud Contract 的 `stubsPerConsumer` 功能解决了什么问题？
- 契约测试和 API 文档（OpenAPI/Swagger）之间的关系是什么？两者能互相替代吗？
- 消息队列（Kafka / RabbitMQ）场景下如何做契约测试？

### 避坑提示
- 契约定义文件（Groovy/YAML）不要放在 `src/test/resources`，要用 `src/test/resources/contracts` 或独立模块，防止被默认加载。
- Provider 端跑契约测试会**启动 HTTP 服务**（默认 random port），确保端口不冲突。
- 契约变更后，双方需要**同时更新**并重新跑 CI，只改一方会导致测试失败。

---

## 17. 冒烟测试、回归测试、压力测试

### 题目
冒烟测试、回归测试、压力测试分别是什么？各自在什么阶段执行？和单元测试/集成测试的关系是什么？

### 核心答案

**冒烟测试（Smoke Testing）**
- **目的**：快速验证系统"能不能跑起来"，在大量测试运行前过滤致命问题。
- **触发时机**：每次 CI 构建后、部署前。
- **覆盖范围**：系统最核心的 1~5 个路径（如登录 + 下单）。
- **性质**：通常是**自动化**的，但比单测粗糙。
- **类比**：接新电器，先插电看有没有火花和焦味。

**回归测试（Regression Testing）**
- **目的**：确保新改动没有破坏已有功能。
- **触发时机**：每次代码变更合并到主分支后。
- **覆盖范围**：所有已有功能的测试用例（单测 + 集成测）。
- **性质**：全量或按影响范围选择测试套件。
- **工具**：Selenium（UI）、REST Assured（API）、JUnit + Mockito。

**压力测试（Stress / Load Testing）**
- **目的**：验证系统在极端负载下的表现（吞吐量、响应时间、稳定性）。
- **触发时机**：上线前、大促前、架构变更后。
- **工具**：JMeter、Gatling、k6、Locust。
- **关注指标**：TPS、RPS、P99 延迟、错误率、资源饱和度。
- **性质**：**非功能需求测试**，不属于单元/集成测试范畴。

**四者关系**

```
开发阶段：单元测试（单测覆盖每一行代码）
            ↓
CI 阶段：   冒烟测试（快速验证构建可部署）
            ↓
部署前：    回归测试（全量功能验证）
            ↓
上线前：    压力测试（非功能验证）
```

### 追问方向
- 冒烟测试和"健全性测试"（Sanity Test）有什么区别？业界有混淆，如何区分？
- 回归测试套件如何在"执行时间"和"覆盖率"之间做平衡？测试优先级如何排序？
- 压力测试中 `TPS` 和 `RPS` 的区别是什么？它们的关系是什么？

### 避坑提示
- 冒烟测试不要写成"所有单测都跑一遍"——冒烟的目的是快，5分钟还没跑完的冒烟测试失去意义。
- 回归测试要按功能模块分组（`mvn test -Dgroups=order,user`），避免每次全量跑。
- 压力测试数据要脱敏，不要在生产环境直接压测。

---

## 18. SonarQube 代码质量门禁

### 题目
SonarQube 能检测哪些质量问题？"代码异味"、"漏洞"、"覆盖率门禁"分别对应什么？如何配置质量门禁？

### 核心答案

**SonarQube 三大检测维度**

| 维度 | 含义 | 示例 |
|---|---|---|
| **Bug** | 运行时必然出现的错误 | ` NullPointerException`、资源泄漏 |
| ** Vulnerability** | 安全漏洞 | SQL 注入、XSS、硬编码密码 |
| **Code Smell（代码异味）** | 可维护性差、风格问题 | 重复代码、长方法、magic number |
| **Coverage** | 覆盖率不足 | 行覆盖率 < 80% |

**质量门禁（Quality Gate）**
SonarQube 通过定义 Quality Gate 设置项目的准入条件：
```
✅ 新代码 Bug = 0（阻断）
✅ 新代码 Vulnerability = 0（阻断）
✅ 覆盖率 ≥ 80%
✅ 重复行数 ≤ 3%
✅ 技术债务 ≤ 总代码量的 5%
```
**不满足质量门禁，CI 构建失败，无法合并代码。**

**Maven 集成**
```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>sonar</goal></goals>
            <phase>verify</phase>
        </execution>
    </executions>
</plugin>
```
```bash
mvn clean verify sonar:sonar \
    -Dsonar.projectKey=my-project \
    -Dsonar.host.url=http://sonarqube:9000
```

**IDE 集成**：IDEA 安装 SonarLint 插件，在编码时实时扫描本地代码问题。

### 追问方向
- SonarQube 的 `SonarQube Scanner` 和 `SonarCloud`（云版）的区别是什么？
- 如何在 GitHub PR 中集成 SonarQube 门禁（SonarQube GitHub App）？
- 代码异味中"Cognitive Complexity"（认知复杂度）是什么？它和圈复杂度有什么区别？

### 避坑提示
- SonarQube 只扫描**提交到 SCM 的代码**，本地 `mvn test` 不触发分析，确保 CI 配置正确。
- 覆盖率需要配合 JaCoCo / Cobertura 插件上传 `.exec` 文件到 SonarQube。
- SonarQube 的 `new_code`（新代码）和 `overall`（整体）有不同的质量标准，质量门禁配置要区分这两个维度。

---

## 19. 单元测试原则：AIR 原则

### 题目
单元测试的 AIR 原则是什么？每个字母代表什么含义？为什么这三条原则是单元测试的基础？

### 核心答案

**A — Automatic（自动化）**
- 测试从执行到结果判定**完全自动**，无需人工介入。
- 不需要手动设置环境、启动服务、对比结果。
- 任何人都能在 CI 中一键运行，结果可复现。

**I — Independent（独立性）**
- 每个测试方法**独立运行**，不依赖其他测试的执行顺序或结果。
- 测试间**不共享可变状态**（静态变量、文件、数据库），避免"测试顺序依赖"导致的间歇性失败。
- 单个测试失败时不影响其他测试。

**R — Repeatable（可重复性）**
- 同一测试在**任何环境、任何时间、执行任意次**，结果必须一致。
- 不能依赖本地时间（`new Date()`）、随机数（未设 seed）、网络（外部 API）、文件系统等不稳定因素。
- 涉及外部资源时，使用 Test Doubles（Mock / Stub）隔离。

**为什么 AIR 是基础**

```
没有 A（自动化）  →  无法进入 CI，测试不落地
没有 I（独立性）  →  顺序依赖，CI 并行跑失败，调试困难
没有 R（可重复性）→  随机失败，CI 红绿不稳定，信任崩塌
```

**反模式示例**
```java
// ❌ 违反 A：需要手动启动 Redis 才能跑
@Test
void testCache() {
    // 依赖本地 Redis 启动状态
}

// ❌ 违反 I：测试 B 依赖测试 A 的副作用
static List<String> shared = new ArrayList<>();
@Test
void testB() {
    assertThat(shared).contains("fromA"); // 依赖执行顺序
}

// ❌ 违反 R：依赖当前时间
@Test
void testTokenExpiry() {
    Token t = new Token();
    assertTrue(t.isExpired()); // 时间变化，结果不确定
}
```

### 追问方向
- "测试金字塔"（Test Pyramid）和 AIR 原则的关系是什么？A/I/R 不达标会导致金字塔变形吗？
- JUnit 5 的 `@TestMethodOrder` 配合 `@Order` 注解会破坏 I 原则吗？什么场景下才需要？
- 数据库测试中，事务回滚是保证 I 还是 R 的手段？

### 避坑提示
- `@BeforeEach` 中初始化可变成员变量是常见的 I 原则违反点，静态变量不要存测试数据。
- `@SpringBootTest` 的多个测试方法如果共享 `@MockBean`，记得每个 `@Test` 后 `reset`，否则 stub 状态泄漏。
- 用 `@RepeatedTest` 时注意 R 原则：每次重复的结果必须一致，随机性测试要设好 seed。

---

## 20. 测试注解执行顺序

### 题目
在 JUnit 4 和 JUnit 5 中，`@BeforeSuite`、`@BeforeClass`、`@Before`、`@Test` 等注解的执行顺序分别是什么？哪些是套件级别，哪些是方法级别？

### 核心答案

**JUnit 4 执行顺序（按生命周期层级）**

```
@BeforeSuite        → 套件启动前（仅 TestNG，JUnit 4 无此注解）
    ↓
@BeforeClass         → 测试类加载前（一次性，static）
    ↓
@Before              → 每个测试方法前
    ↓
@Test                → 测试方法本身
    ↓
@After               → 每个测试方法后
    ↓
@AfterClass          → 测试类结束（一次性，static）
```

**JUnit 5（Jupiter）执行顺序**

```
@BeforeAll           → 测试类生命周期开始前（PER_CLASS 模式下非 static，PER_METHOD 必须 static）
    ↓
@BeforeEach          → 每个测试方法前
    ↓
@Test                → 测试方法
    ↓
@AfterEach           → 每个测试方法后
    ↓
@AfterAll            → 测试类生命周期结束时
```

**TestNG 执行顺序（参考对比）**
```
@BeforeSuite         → 整个测试套件前
    ↓
@BeforeTest          → 套件中某个 <test> tag 前
    ↓
@BeforeClass         → 测试类前
    ↓
@BeforeMethod        → 每个测试方法前
    ↓
@Test                → 测试方法
    ↓
@AfterMethod         → 每个测试方法后
    ↓
@AfterClass          → 测试类后
    ↓
@AfterTest           → 套件中某个 <test> tag 后
    ↓
@AfterSuite          → 整个测试套件后
```

**JUnit 4 中没有 `@BeforeSuite`**（这是 TestNG 的概念），JUnit 5 中也无套件级别注解，需要用 `@Suite` 配合 `@SelectPackages` 实现类似效果。

**嵌套类执行顺序（JUnit 5）**
```
外层类 BeforeAll
    外层类 BeforeEach
        外层 @Test
    外层类 AfterEach
嵌套类 BeforeAll
    嵌套类 BeforeEach
        嵌套 @Test
    嵌套类 AfterEach
嵌套类 AfterAll
外层类 AfterAll
```

### 追问方向
- `@BeforeClass` 的 static 方法中抛出异常会导致什么？JUnit 5 的 `@BeforeAll` 在 PER_CLASS 模式下呢？
- 嵌套测试的 `@BeforeAll` / `@AfterAll` 默认要求 static，如何用 `@TestInstance(Lifecycle.PER_CLASS)` 改为非 static？
- `@Suite` 和 `@RunWith(Suite.class)` 在 JUnit 4 中如何配置多测试类组合？

### 避坑提示
- `@BeforeClass` / `@AfterClass` 中的代码要谨慎——所有测试方法共享，容易引起状态污染。
- 在 `@BeforeEach` 中 new 对象 vs 在 `@BeforeClass` 中 static 初始化——前者保证测试间隔离，后者共享。
- JUnit 5 中 `@Tag` 过滤在 `@BeforeAll` 之前生效，不会阻止 `@BeforeAll` 的执行，但会跳过被过滤的测试方法。

---

## 附录：面试准备清单

```markdown
□ JUnit 4 → JUnit 5 迁移清单（注解对照表）
□ Mockito 常用 API 速查（when / doReturn / verify / ArgumentCaptor）
□ Spring Boot 测试切片选择决策树
□ TDD 红绿重构循环流程图
□ JaCoCo + SonarQube CI 集成配置
□ TestContainers 快速启动模板（MySQL / PostgreSQL）
□ @ParameterizedTest CSV 数据文件格式示例
□ Spring Cloud Contract Provider / Consumer 脚手架
□ 常用测试注解执行顺序速记表
```

---

*以上内容覆盖 Java 单元测试与质量保障方向 20 个高频考点，建议配合代码示例动手实验，不要只背理论。面试中能讲清楚"踩过什么坑"比"知道什么概念"更有说服力。*

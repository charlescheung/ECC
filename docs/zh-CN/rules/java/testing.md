---
paths:
  - "**/*.java"
---

# Java 测试

> 本文档扩展了 [common/testing.md](../common/testing.md) 中与 Java 相关的内容。

## 测试框架

* **JUnit 5** (`@Test`, `@ParameterizedTest`, `@Nested`, `@DisplayName`)
* **AssertJ** 用于流式断言 (`assertThat(result).isEqualTo(expected)`)
* **Mockito** 用于模拟依赖
* **Testcontainers** 用于需要数据库或服务的集成测试

## 测试组织

```
src/test/java/com/example/app/
  service/           # 服务层单元测试
  controller/        # Web 层/API 测试
  repository/        # 数据访问测试
  integration/       # 跨层集成测试
```

在 `src/test/java` 中镜像 `src/main/java` 的包结构。

## 单元测试模式

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(orderRepository);
    }

    @Test
    @DisplayName("findById returns order when exists")
    void findById_existingOrder_returnsOrder() {
        var order = new Order(1L, "Alice", BigDecimal.TEN);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

        var result = orderService.findById(1L);

        assertThat(result.customerName()).isEqualTo("Alice");
        verify(orderRepository).findById(1L);
    }

    @Test
    @DisplayName("findById throws when order not found")
    void findById_missingOrder_throws() {
        when(orderRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> orderService.findById(99L))
            .isInstanceOf(OrderNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

## 参数化测试

```java
@ParameterizedTest
@CsvSource({
    "100.00, 10, 90.00",
    "50.00, 0, 50.00",
    "200.00, 25, 150.00"
})
@DisplayName("discount applied correctly")
void applyDiscount(BigDecimal price, int pct, BigDecimal expected) {
    assertThat(PricingUtils.discount(price, pct)).isEqualByComparingTo(expected);
}
```

## 集成测试

使用 Testcontainers 进行真实的数据库集成：

```java
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    private OrderRepository repository;

    @BeforeEach
    void setUp() {
        var dataSource = new PGSimpleDataSource();
        dataSource.setUrl(postgres.getJdbcUrl());
        dataSource.setUser(postgres.getUsername());
        dataSource.setPassword(postgres.getPassword());
        repository = new JdbcOrderRepository(dataSource);
    }

    @Test
    void save_and_findById() {
        var saved = repository.save(new Order(null, "Bob", BigDecimal.ONE));
        var found = repository.findById(saved.getId());
        assertThat(found).isPresent();
    }
}
```

关于 Spring Boot 集成测试，请参阅技能：`springboot-tdd`。
关于 Quarkus 集成测试，请参阅技能：`quarkus-tdd`。

## 测试命名

使用带有 `@DisplayName` 的描述性名称：

* `methodName_scenario_expectedBehavior()` 用于方法名
* `@DisplayName("human-readable description")` 用于报告

## 覆盖率

* 目标为 80%+ 的行覆盖率
* 使用 JaCoCo 生成覆盖率报告
* 重点关注服务和领域逻辑 — 跳过简单的 getter/配置类

## 阿里巴巴规范 —— 单元测试要求

- 单元测试必须覆盖 **BCDE** 四类场景：Border（边界值）、Correct（正确输入）、Design（设计文档对应场景）、Error（错误输入）
- 每个测试方法只测试**一个**逻辑场景，测试体内禁止出现分支判断
- 测试必须完全**独立**：测试方法间无共享可变状态，无执行顺序依赖
- 单元测试中必须 Mock 所有外部依赖（DB、HTTP、缓存、文件系统），Testcontainers 仅用于集成测试
- 测试方法命名：`methodName_scenario_expectedResult()`，不读方法体也能理解测试意图
- 测试中禁止使用 `System.out.println`，使用 AssertJ 断言或 logger
- 单元测试每个用例执行时间必须**在 1 秒以内**；慢测试使用 `@Tag("slow")` 标记
- 核心领域/服务逻辑（`src/core/**`、`src/service/**`）必须达到 **100% 分支覆盖**
- 每个测试必须包含至少一个**正常路径**和一个**错误路径**用例

```java
// 正确 —— 命名规范、测试隔离、BCDE 覆盖
@Test
@DisplayName("折扣超过 100% 时抛出 IllegalArgumentException")
void applyDiscount_discountOver100_throwsIllegalArgument() {
    // Arrange
    var price = new BigDecimal("100.00");

    // Act & Assert
    assertThatThrownBy(() -> PricingUtils.applyDiscount(price, 101))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("discount");
}

@Test
@DisplayName("折扣为 100% 时返回零")
void applyDiscount_discount100_returnsZero() {
    assertThat(PricingUtils.applyDiscount(new BigDecimal("100.00"), 100))
        .isEqualByComparingTo(BigDecimal.ZERO);
}
```

## 参考

关于使用 MockMvc 和 Testcontainers 的 Spring Boot TDD 模式，请参阅技能：`springboot-tdd`。
关于使用 REST Assured 和 Camel 测试的 Quarkus TDD 模式，请参阅技能：`quarkus-tdd`。
关于测试期望，请参阅技能：`java-coding-standards`。

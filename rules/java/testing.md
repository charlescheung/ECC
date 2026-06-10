---
paths:
  - "**/*.java"
---
# Java Testing

> This file extends [common/testing.md](../common/testing.md) with Java-specific content.

## Test Framework

- **JUnit 5** (`@Test`, `@ParameterizedTest`, `@Nested`, `@DisplayName`)
- **AssertJ** for fluent assertions (`assertThat(result).isEqualTo(expected)`)
- **Mockito** for mocking dependencies
- **Testcontainers** for integration tests requiring databases or services

## Test Organization

```
src/test/java/com/example/app/
  service/           # Unit tests for service layer
  controller/        # Web layer / API tests
  repository/        # Data access tests
  integration/       # Cross-layer integration tests
```

Mirror the `src/main/java` package structure in `src/test/java`.

## Unit Test Pattern

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

## Parameterized Tests

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

## Integration Tests

Use Testcontainers for real database integration:

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

For Spring Boot integration tests, see skill: `springboot-tdd`.
For Quarkus integration tests, see skill: `quarkus-tdd`.

## Test Naming

Use descriptive names with `@DisplayName`:
- `methodName_scenario_expectedBehavior()` for method names
- `@DisplayName("human-readable description")` for reports

## Coverage

- Target 80%+ line coverage
- Use JaCoCo for coverage reporting
- Focus on service and domain logic — skip trivial getters/config classes

## Alibaba Unit Test Rules (阿里规范 — 单元测试)

- Unit tests must be **BCDE**: Border (边界值), Correct (正确输入), Design (设计文档对应场景), Error (错误输入)
- Each test method tests **one** logical scenario — no branching inside a test
- Tests must be fully **independent**: no shared mutable state between test methods, no execution-order dependency
- Mock all external dependencies (DB, HTTP, cache, filesystem) in unit tests — use Testcontainers only in integration tests
- Test method naming: `methodName_scenario_expectedResult()` — must be self-documenting without reading the body
- Do not use `System.out.println` in tests — use AssertJ assertions or logger
- Tests must run in **under 1 second** each for unit tests; flag slow tests with `@Tag("slow")`
- Achieve **100% branch coverage** for core domain/service logic (`src/core/**`, `src/service/**`)
- Every test must include at least one **happy path** and one **error path** case

```java
// GOOD — naming, isolation, and BCDE coverage
@Test
@DisplayName("applyDiscount throws when discount exceeds 100%")
void applyDiscount_discountOver100_throwsIllegalArgument() {
    // Arrange
    var price = new BigDecimal("100.00");

    // Act & Assert
    assertThatThrownBy(() -> PricingUtils.applyDiscount(price, 101))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("discount");
}

@Test
@DisplayName("applyDiscount returns zero when discount is 100%")
void applyDiscount_discount100_returnsZero() {
    assertThat(PricingUtils.applyDiscount(new BigDecimal("100.00"), 100))
        .isEqualByComparingTo(BigDecimal.ZERO);
}
```

## References

See skill: `springboot-tdd` for Spring Boot TDD patterns with MockMvc and Testcontainers.
See skill: `quarkus-tdd` for Quarkus TDD patterns with REST Assured and Dev Services.
See skill: `java-coding-standards` for testing expectations.

---
paths:
  - "**/*.java"
---
# Java Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Java-specific content.

## Repository Pattern

Encapsulate data access behind an interface:

```java
public interface OrderRepository {
    Optional<Order> findById(Long id);
    List<Order> findAll();
    Order save(Order order);
    void deleteById(Long id);
}
```

Concrete implementations handle storage details (JPA, JDBC, in-memory for tests).

## Service Layer

Business logic in service classes; keep controllers and repositories thin:

```java
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public OrderService(OrderRepository orderRepository, PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }

    public OrderSummary placeOrder(CreateOrderRequest request) {
        var order = Order.from(request);
        paymentGateway.charge(order.total());
        var saved = orderRepository.save(order);
        return OrderSummary.from(saved);
    }
}
```

## Constructor Injection

Always use constructor injection — never field injection:

```java
// GOOD — constructor injection (testable, immutable)
public class NotificationService {
    private final EmailSender emailSender;

    public NotificationService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}

// BAD — field injection (untestable without reflection, requires framework magic)
public class NotificationService {
    @Inject // or @Autowired
    private EmailSender emailSender;
}
```

## DTO Mapping

Use records for DTOs. Map at service/controller boundaries:

```java
public record OrderResponse(Long id, String customer, BigDecimal total) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(order.getId(), order.getCustomerName(), order.getTotal());
    }
}
```

## Builder Pattern

Use for objects with many optional parameters:

```java
public class SearchCriteria {
    private final String query;
    private final int page;
    private final int size;
    private final String sortBy;

    private SearchCriteria(Builder builder) {
        this.query = builder.query;
        this.page = builder.page;
        this.size = builder.size;
        this.sortBy = builder.sortBy;
    }

    public static class Builder {
        private String query = "";
        private int page = 0;
        private int size = 20;
        private String sortBy = "id";

        public Builder query(String query) { this.query = query; return this; }
        public Builder page(int page) { this.page = page; return this; }
        public Builder size(int size) { this.size = size; return this; }
        public Builder sortBy(String sortBy) { this.sortBy = sortBy; return this; }
        public SearchCriteria build() { return new SearchCriteria(this); }
    }
}
```

## Sealed Types for Domain Models

```java
public sealed interface PaymentResult permits PaymentSuccess, PaymentFailure {
    record PaymentSuccess(String transactionId, BigDecimal amount) implements PaymentResult {}
    record PaymentFailure(String errorCode, String message) implements PaymentResult {}
}

// Exhaustive handling (Java 21+)
String message = switch (result) {
    case PaymentSuccess s -> "Paid: " + s.transactionId();
    case PaymentFailure f -> "Failed: " + f.errorCode();
};
```

## API Response Envelope

Consistent API responses:

```java
public record ApiResponse<T>(boolean success, T data, String error) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null);
    }
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
}
```

## Alibaba Design Pattern Rules (阿里规范 — 设计模式约束)

### Singleton

- Prefer enum-based singleton or initialization-on-demand holder — never double-checked locking with a non-`volatile` field
- Stateless service beans managed by a DI container (Spring, Quarkus CDI) are singletons by default — do not implement the pattern manually in that context

```java
// GOOD — holder pattern (lazy, thread-safe, no sync overhead)
public class AppConfig {
    private AppConfig() {}
    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }
    public static AppConfig getInstance() { return Holder.INSTANCE; }
}
```

### Strategy Pattern

- Extract varying algorithms behind an interface; inject the strategy via constructor
- Prefer a `Map<String, Strategy>` lookup table over long `if-else` or `switch` chains

```java
// BAD — long if-else, closed to extension
if ("alipay".equals(type)) { ... }
else if ("wechat".equals(type)) { ... }

// GOOD — strategy map, open for extension
Map<String, PaymentStrategy> strategies = Map.of(
    "alipay", new AlipayStrategy(),
    "wechat", new WechatStrategy()
);
strategies.get(type).pay(amount);
```

### Template Method

- Use for algorithms with a fixed skeleton but variable steps
- Prefer default interface methods (Java 8+) over abstract classes when state is not needed

### Layer Separation (阿里分层约束)

- **Controller** — input validation and response mapping only; no business logic
- **Service** — business logic; no direct HTTP or persistence concern
- **Repository/DAO** — data access only; no business logic
- Cross-layer calls must go downward only: Controller → Service → Repository
- DTOs cross layer boundaries; domain objects must not leak to the Controller layer

## GoF 23 Design Patterns — When and How to Use in Java

Prefer these patterns over ad-hoc solutions when the scenario matches. YAGNI still applies — do not introduce a pattern speculatively.

---

### Creational Patterns (创建型)

#### 1. Singleton
**When**: One shared instance per JVM is required (config holder, registry, connection pool wrapper).
**How**: Holder idiom or enum. Never DCL without `volatile`. In a DI container, use the container's scope instead.
```java
public class AppConfig {
    private AppConfig() {}
    private static class Holder { static final AppConfig INSTANCE = new AppConfig(); }
    public static AppConfig getInstance() { return Holder.INSTANCE; }
}
```

#### 2. Factory Method
**When**: A base class defines the creation contract but subclasses decide the concrete type.
**How**: Abstract creator with a `create()` method; each subclass returns its own product.
```java
public abstract class NotificationFactory {
    public abstract Notification create(String recipient);
}
public class EmailNotificationFactory extends NotificationFactory {
    public Notification create(String recipient) { return new EmailNotification(recipient); }
}
```

#### 3. Abstract Factory
**When**: Families of related objects must be created together and swapped as a unit (e.g., UI themes, database dialects).
**How**: Interface with multiple `createXxx()` methods; each concrete factory produces a consistent family.
```java
public interface UiFactory {
    Button createButton();
    Dialog createDialog();
}
public class DarkUiFactory implements UiFactory { ... }
public class LightUiFactory implements UiFactory { ... }
```

#### 4. Builder
**When**: Object construction needs 4+ parameters, many of which are optional.
**How**: Static inner `Builder` class with a `build()` terminal method. Prefer Lombok `@Builder` in Spring projects.
```java
SearchCriteria criteria = new SearchCriteria.Builder()
    .query("shoes")
    .page(0)
    .size(20)
    .build();
```

#### 5. Prototype
**When**: Creating an object is expensive and a copy of an existing instance is cheaper.
**How**: Implement a `copy()` factory method (preferred over `Cloneable` — avoids `CloneNotSupportedException`).
```java
public class ReportTemplate {
    public ReportTemplate copy() {
        return new ReportTemplate(this.title, new ArrayList<>(this.sections));
    }
}
```

---

### Structural Patterns (结构型)

#### 6. Adapter
**When**: An existing class has an incompatible interface that cannot be changed (third-party library, legacy code).
**How**: Wrap the adaptee; implement the target interface; delegate calls.
```java
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPayClient client;
    public LegacyPaymentAdapter(LegacyPayClient client) { this.client = client; }
    public void charge(BigDecimal amount) { client.doCharge(amount.doubleValue()); }
}
```

#### 7. Bridge
**When**: Abstraction and implementation must vary independently (e.g., shapes × rendering engines).
**How**: Abstraction holds a reference to the implementor interface; extend each dimension independently.

#### 8. Composite
**When**: Part-whole tree structures where clients treat leaf and composite uniformly (menus, file trees, expression trees).
**How**: Common `Component` interface; `Leaf` implements it directly; `Composite` holds a `List<Component>` and delegates.
```java
public interface MenuItem { void render(); }
public class MenuGroup implements MenuItem {
    private final List<MenuItem> children = new ArrayList<>();
    public void add(MenuItem item) { children.add(item); }
    public void render() { children.forEach(MenuItem::render); }
}
```

#### 9. Decorator
**When**: Adding responsibilities to an object dynamically without subclassing (e.g., caching wrappers, logging, I/O streams).
**How**: Implement the same interface; wrap the delegate; add behavior before/after delegation.
```java
public class CachingOrderRepository implements OrderRepository {
    private final OrderRepository delegate;
    private final Cache<Long, Order> cache;
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(cache.get(id, k -> delegate.findById(k).orElse(null)));
    }
}
```

#### 10. Facade
**When**: Simplifying a complex subsystem behind a single entry point (e.g., `OrderFacade` coordinating inventory, payment, and shipping).
**How**: One class with high-level methods; internally orchestrates multiple subsystem classes.
```java
public class OrderFacade {
    public OrderSummary placeOrder(CreateOrderRequest req) {
        inventory.reserve(req.items());
        payment.charge(req.total());
        shipping.schedule(req.address());
        return orderRepository.save(Order.from(req));
    }
}
```

#### 11. Flyweight
**When**: Large numbers of fine-grained objects share most of their state (e.g., character glyphs, map tiles, icon instances).
**How**: Factory caches and returns shared instances; extrinsic state passed at runtime.
```java
public class IconFactory {
    private static final Map<String, Icon> cache = new HashMap<>();
    public static Icon get(String type) {
        return cache.computeIfAbsent(type, Icon::load);
    }
}
```

#### 12. Proxy
**When**: Controlling access to an object — lazy init, access control, remote call, logging, or caching.
**How**: Same interface as the real subject; delegate after applying cross-cutting logic. Spring AOP generates proxies automatically for `@Transactional`, `@Cacheable`, etc.
```java
public class SecureOrderRepository implements OrderRepository {
    private final OrderRepository real;
    public Optional<Order> findById(Long id) {
        authorizationService.checkRead(currentUser(), id);
        return real.findById(id);
    }
}
```

---

### Behavioral Patterns (行为型)

#### 13. Chain of Responsibility
**When**: A request must pass through a sequence of handlers; each handler decides to process or pass on (filter chains, validation pipelines, middleware).
**How**: Each handler holds a reference to the next; call `next.handle(request)` to pass along.
```java
public abstract class ValidationHandler {
    private ValidationHandler next;
    public ValidationHandler then(ValidationHandler next) { this.next = next; return next; }
    public abstract void validate(OrderRequest req);
    protected void passToNext(OrderRequest req) { if (next != null) next.validate(req); }
}
```

#### 14. Command
**When**: Encapsulating a request as an object to support undo/redo, queuing, or audit logging (task queues, transactional scripts, editor actions).
**How**: `Command` interface with `execute()` (and optionally `undo()`); invoker holds and calls commands.
```java
public interface Command { void execute(); }
public class PlaceOrderCommand implements Command {
    public void execute() { orderService.place(request); }
}
```

#### 15. Iterator
**When**: Providing sequential access to elements without exposing the underlying structure.
**How**: Implement `Iterable<T>` / `Iterator<T>`. In most cases JDK collections already provide this — prefer them over hand-rolled iterators.

#### 16. Mediator
**When**: Many objects communicate in complex ways; centralizing coordination reduces coupling (event buses, UI form controllers, chat rooms).
**How**: Objects send messages to the mediator; the mediator routes to the appropriate receiver.
```java
public class OrderEventMediator {
    public void on(OrderPlacedEvent event) {
        inventoryService.reserve(event.items());
        notificationService.notifyCustomer(event.customerId());
    }
}
```

#### 17. Memento
**When**: Capturing and restoring an object's state without breaking encapsulation (undo history, snapshots, drafts).
**How**: Originator creates `Memento` value objects (use `record`); caretaker stores and restores them.
```java
public record DraftMemento(String content, Instant savedAt) {}
public class DocumentEditor {
    public DraftMemento save() { return new DraftMemento(content, Instant.now()); }
    public void restore(DraftMemento m) { this.content = m.content(); }
}
```

#### 18. Observer
**When**: One-to-many dependency — when one object changes state, dependents are notified (domain events, UI data binding, pub/sub).
**How**: In Spring, use `ApplicationEventPublisher` / `@EventListener`. In plain Java, maintain a `List<Observer>` and call `notify()`.
```java
// Spring style — preferred
@Component
public class InventoryEventListener {
    @EventListener
    public void on(OrderPlacedEvent event) { inventory.reserve(event.items()); }
}
```

#### 19. State
**When**: An object's behavior depends on its current state and must change at runtime (order lifecycle: PENDING → PAID → SHIPPED → DELIVERED).
**How**: Extract each state into a class implementing a common `State` interface; the context delegates to the current state.
```java
public interface OrderState {
    void pay(OrderContext ctx);
    void ship(OrderContext ctx);
}
public class PendingState implements OrderState {
    public void pay(OrderContext ctx) { ctx.setState(new PaidState()); }
    public void ship(OrderContext ctx) { throw new IllegalStateException("Pay first"); }
}
```

#### 20. Strategy
**When**: A family of interchangeable algorithms (sorting, pricing rules, payment methods, export formats).
**How**: Interface per algorithm family; inject via constructor; use `Map<String, Strategy>` to select by key.
```java
public interface DiscountStrategy { BigDecimal apply(BigDecimal price); }
Map<String, DiscountStrategy> strategies = Map.of(
    "vip",      price -> price.multiply(new BigDecimal("0.8")),
    "seasonal", price -> price.multiply(new BigDecimal("0.9"))
);
```

#### 21. Template Method
**When**: An algorithm has a fixed skeleton with steps that vary by subclass (report generation, data import pipelines).
**How**: Abstract class defines the skeleton in a `final` method; subclasses override hook methods. Prefer interface `default` methods when no shared state is needed.
```java
public abstract class ReportGenerator {
    public final String generate() {
        return renderHeader() + formatBody(fetchData()) + renderFooter();
    }
    protected abstract String fetchData();
    protected abstract String formatBody(String data);
    protected String renderHeader() { return ""; }
    protected String renderFooter() { return ""; }
}
```

#### 22. Visitor
**When**: Adding operations to a class hierarchy without modifying it (AST traversal, export to different formats, metrics collection).
**How**: `Visitor` interface with a `visit(ConcreteElement)` overload per element type; each element implements `accept(Visitor)`.
```java
public interface PricingVisitor {
    void visit(PhysicalItem item);
    void visit(DigitalItem item);
}
public class TaxCalculator implements PricingVisitor {
    public void visit(PhysicalItem item) { /* apply shipping tax */ }
    public void visit(DigitalItem item) { /* apply digital VAT */ }
}
```

#### 23. Interpreter
**When**: A grammar or expression language needs to be evaluated repeatedly (rule engines, query DSLs, expression parsers). Use sparingly — complex grammars belong in a parser library.
**How**: Each grammar rule becomes a class implementing `Expression`; composite expressions hold sub-expressions.
```java
public interface Expression { boolean interpret(Map<String, Boolean> context); }
public class AndExpression implements Expression {
    private final Expression left, right;
    public boolean interpret(Map<String, Boolean> ctx) {
        return left.interpret(ctx) && right.interpret(ctx);
    }
}
```

---

### Pattern Selection Guide

| Scenario | Preferred Pattern(s) |
|----------|----------------------|
| Object has many optional params | Builder |
| Swap algorithm at runtime | Strategy |
| Fixed algorithm, variable steps | Template Method |
| Add behavior without subclassing | Decorator |
| One shared instance | Singleton (Holder/enum) |
| Notify dependents on state change | Observer |
| Object behavior varies by state | State |
| Complex subsystem, simple API | Facade |
| Request pipeline / middleware | Chain of Responsibility |
| Undo / command queue | Command |
| Part-whole tree | Composite |
| Expensive object reuse | Flyweight / Prototype |
| Incompatible interface | Adapter |
| Access control / lazy init | Proxy |
| Many objects communicate | Mediator |
| Save / restore state | Memento |
| Operations on a type hierarchy | Visitor |
| Evaluate grammar / rules | Interpreter |

---

## References

See skill: `springboot-patterns` for Spring Boot architecture patterns.
See skill: `quarkus-patterns` for Quarkus architecture patterns with REST, Panache, and messaging.
See skill: `jpa-patterns` for entity design and query optimization.

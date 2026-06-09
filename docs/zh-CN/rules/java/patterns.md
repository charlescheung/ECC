---
paths:
  - "**/*.java"
---

# Java 模式

> 本文档扩展了 [common/patterns.md](../common/patterns.md) 中的内容，增加了 Java 特有的部分。

## 仓储模式

将数据访问封装在接口之后：

```java
public interface OrderRepository {
    Optional<Order> findById(Long id);
    List<Order> findAll();
    Order save(Order order);
    void deleteById(Long id);
}
```

具体的实现类处理存储细节（JPA、JDBC、用于测试的内存存储等）。

## 服务层

业务逻辑放在服务类中；保持控制器和仓储层的精简：

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

## 构造函数注入

始终使用构造函数注入 —— 绝不使用字段注入：

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

## DTO 映射

使用记录（record）作为 DTO。在服务层/控制器边界进行映射：

```java
public record OrderResponse(Long id, String customer, BigDecimal total) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(order.getId(), order.getCustomerName(), order.getTotal());
    }
}
```

## 建造者模式

用于具有多个可选参数的对象：

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

## 使用密封类型构建领域模型

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

## API 响应封装

统一的 API 响应格式：

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

## GoF 23 种设计模式 —— Java 中的适用场景与写法

在场景匹配时优先使用这些模式，而不是临时方案。YAGNI 原则依然适用——不要为了未来可能存在的需求提前引入模式。

---

### 创建型模式

#### 1. 单例模式（Singleton）
**适用场景**：整个 JVM 中需要唯一共享实例（配置持有者、注册表、连接池包装）。
**写法**：Holder 惯用法或枚举。非 `volatile` 字段的双重检查锁定（DCL）禁止使用。在 DI 容器中直接使用容器的作用域，无需手动实现。
```java
public class AppConfig {
    private AppConfig() {}
    private static class Holder { static final AppConfig INSTANCE = new AppConfig(); }
    public static AppConfig getInstance() { return Holder.INSTANCE; }
}
```

#### 2. 工厂方法模式（Factory Method）
**适用场景**：基类定义创建契约，子类决定具体类型。
**写法**：抽象创建者包含 `create()` 方法；每个子类返回自己的产品。
```java
public abstract class NotificationFactory {
    public abstract Notification create(String recipient);
}
public class EmailNotificationFactory extends NotificationFactory {
    public Notification create(String recipient) { return new EmailNotification(recipient); }
}
```

#### 3. 抽象工厂模式（Abstract Factory）
**适用场景**：一组相关对象必须一起创建并作为整体切换（如 UI 主题、数据库方言）。
**写法**：包含多个 `createXxx()` 方法的接口；每个具体工厂生产一族一致的对象。
```java
public interface UiFactory {
    Button createButton();
    Dialog createDialog();
}
public class DarkUiFactory implements UiFactory { ... }
public class LightUiFactory implements UiFactory { ... }
```

#### 4. 建造者模式（Builder）
**适用场景**：对象构造需要 4 个以上参数，且其中许多是可选的。
**写法**：静态内部 `Builder` 类，以 `build()` 方法收尾。Spring 项目优先使用 Lombok `@Builder`。
```java
SearchCriteria criteria = new SearchCriteria.Builder()
    .query("shoes")
    .page(0)
    .size(20)
    .build();
```

#### 5. 原型模式（Prototype）
**适用场景**：创建对象代价高昂，复制现有实例更高效。
**写法**：实现 `copy()` 工厂方法（优于 `Cloneable`，避免 `CloneNotSupportedException`）。
```java
public class ReportTemplate {
    public ReportTemplate copy() {
        return new ReportTemplate(this.title, new ArrayList<>(this.sections));
    }
}
```

---

### 结构型模式

#### 6. 适配器模式（Adapter）
**适用场景**：已有类接口不兼容且无法修改（第三方库、遗留代码）。
**写法**：包装被适配者；实现目标接口；委托调用。
```java
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPayClient client;
    public LegacyPaymentAdapter(LegacyPayClient client) { this.client = client; }
    public void charge(BigDecimal amount) { client.doCharge(amount.doubleValue()); }
}
```

#### 7. 桥接模式（Bridge）
**适用场景**：抽象与实现需要独立变化（如形状 × 渲染引擎）。
**写法**：抽象层持有实现接口的引用；两个维度独立扩展。

#### 8. 组合模式（Composite）
**适用场景**：部分-整体树形结构，客户端对叶子和组合节点统一处理（菜单、文件树、表达式树）。
**写法**：公共 `Component` 接口；`Leaf` 直接实现；`Composite` 持有 `List<Component>` 并委托。
```java
public interface MenuItem { void render(); }
public class MenuGroup implements MenuItem {
    private final List<MenuItem> children = new ArrayList<>();
    public void add(MenuItem item) { children.add(item); }
    public void render() { children.forEach(MenuItem::render); }
}
```

#### 9. 装饰器模式（Decorator）
**适用场景**：不通过继承动态添加职责（缓存包装、日志、I/O 流）。
**写法**：实现相同接口；包装委托对象；在委托调用前后添加行为。
```java
public class CachingOrderRepository implements OrderRepository {
    private final OrderRepository delegate;
    private final Cache<Long, Order> cache;
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(cache.get(id, k -> delegate.findById(k).orElse(null)));
    }
}
```

#### 10. 外观模式（Facade）
**适用场景**：将复杂子系统隐藏在单一入口后面（如 `OrderFacade` 协调库存、支付、物流）。
**写法**：一个类提供高层方法；内部编排多个子系统类。
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

#### 11. 享元模式（Flyweight）
**适用场景**：大量细粒度对象共享大部分状态（字符字形、地图图块、图标实例）。
**写法**：工厂缓存并返回共享实例；外部状态在运行时传入。
```java
public class IconFactory {
    private static final Map<String, Icon> cache = new HashMap<>();
    public static Icon get(String type) {
        return cache.computeIfAbsent(type, Icon::load);
    }
}
```

#### 12. 代理模式（Proxy）
**适用场景**：控制对对象的访问——延迟初始化、权限控制、远程调用、日志、缓存。
**写法**：与真实对象实现相同接口；在委托前后施加横切逻辑。Spring AOP 会为 `@Transactional`、`@Cacheable` 等自动生成代理。
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

### 行为型模式

#### 13. 责任链模式（Chain of Responsibility）
**适用场景**：请求需要经过一系列处理器，每个处理器决定处理或继续传递（过滤器链、校验管道、中间件）。
**写法**：每个处理器持有下一个处理器的引用；调用 `next.handle(request)` 传递。
```java
public abstract class ValidationHandler {
    private ValidationHandler next;
    public ValidationHandler then(ValidationHandler next) { this.next = next; return next; }
    public abstract void validate(OrderRequest req);
    protected void passToNext(OrderRequest req) { if (next != null) next.validate(req); }
}
```

#### 14. 命令模式（Command）
**适用场景**：将请求封装为对象以支持撤销/重做、队列或审计日志（任务队列、事务脚本、编辑器操作）。
**写法**：`Command` 接口包含 `execute()`（可选 `undo()`）；调用者持有并调用命令。
```java
public interface Command { void execute(); }
public class PlaceOrderCommand implements Command {
    public void execute() { orderService.place(request); }
}
```

#### 15. 迭代器模式（Iterator）
**适用场景**：不暴露底层结构的情况下顺序访问元素。
**写法**：实现 `Iterable<T>` / `Iterator<T>`。大多数情况下 JDK 集合已经提供——优先使用，避免手写迭代器。

#### 16. 中介者模式（Mediator）
**适用场景**：多对象之间通信复杂；集中协调可降低耦合（事件总线、UI 表单控制器、聊天室）。
**写法**：对象向中介者发送消息；中介者路由到合适的接收方。
```java
public class OrderEventMediator {
    public void on(OrderPlacedEvent event) {
        inventoryService.reserve(event.items());
        notificationService.notifyCustomer(event.customerId());
    }
}
```

#### 17. 备忘录模式（Memento）
**适用场景**：在不破坏封装的情况下捕获并恢复对象状态（撤销历史、快照、草稿）。
**写法**：发起者创建 `Memento` 值对象（使用 `record`）；管理者负责存储和恢复。
```java
public record DraftMemento(String content, Instant savedAt) {}
public class DocumentEditor {
    public DraftMemento save() { return new DraftMemento(content, Instant.now()); }
    public void restore(DraftMemento m) { this.content = m.content(); }
}
```

#### 18. 观察者模式（Observer）
**适用场景**：一对多依赖——一个对象状态变化时自动通知所有依赖方（领域事件、UI 数据绑定、发布/订阅）。
**写法**：Spring 项目使用 `ApplicationEventPublisher` / `@EventListener`。纯 Java 维护 `List<Observer>` 并调用 `notify()`。
```java
// Spring 风格 —— 优先使用
@Component
public class InventoryEventListener {
    @EventListener
    public void on(OrderPlacedEvent event) { inventory.reserve(event.items()); }
}
```

#### 19. 状态模式（State）
**适用场景**：对象行为依赖当前状态且需在运行时切换（订单生命周期：待支付 → 已支付 → 已发货 → 已完成）。
**写法**：每个状态提取为实现公共 `State` 接口的类；上下文委托给当前状态。
```java
public interface OrderState {
    void pay(OrderContext ctx);
    void ship(OrderContext ctx);
}
public class PendingState implements OrderState {
    public void pay(OrderContext ctx) { ctx.setState(new PaidState()); }
    public void ship(OrderContext ctx) { throw new IllegalStateException("请先完成支付"); }
}
```

#### 20. 策略模式（Strategy）
**适用场景**：一族可互换算法（排序、定价规则、支付方式、导出格式）。
**写法**：每族算法一个接口；通过构造函数注入；使用 `Map<String, Strategy>` 按键选择。
```java
public interface DiscountStrategy { BigDecimal apply(BigDecimal price); }
Map<String, DiscountStrategy> strategies = Map.of(
    "vip",      price -> price.multiply(new BigDecimal("0.8")),
    "seasonal", price -> price.multiply(new BigDecimal("0.9"))
);
```

#### 21. 模板方法模式（Template Method）
**适用场景**：算法骨架固定但步骤可变（报表生成、数据导入管道）。
**写法**：抽象类用 `final` 方法定义骨架；子类重写钩子方法。不需要共享状态时优先使用接口 `default` 方法。
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

#### 22. 访问者模式（Visitor）
**适用场景**：在不修改类层次结构的情况下添加操作（AST 遍历、多格式导出、指标采集）。
**写法**：`Visitor` 接口为每种元素类型提供 `visit(ConcreteElement)` 重载；每个元素实现 `accept(Visitor)`。
```java
public interface PricingVisitor {
    void visit(PhysicalItem item);
    void visit(DigitalItem item);
}
public class TaxCalculator implements PricingVisitor {
    public void visit(PhysicalItem item) { /* 计算物流税 */ }
    public void visit(DigitalItem item) { /* 计算数字增值税 */ }
}
```

#### 23. 解释器模式（Interpreter）
**适用场景**：需要反复求值的语法或表达式语言（规则引擎、查询 DSL、表达式解析）。复杂语法优先使用解析器库。
**写法**：每条语法规则对应一个实现 `Expression` 的类；复合表达式持有子表达式。
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

### 模式选型速查表

| 场景 | 推荐模式 |
|------|----------|
| 对象有大量可选参数 | 建造者（Builder） |
| 运行时切换算法 | 策略（Strategy） |
| 骨架固定，步骤可变 | 模板方法（Template Method） |
| 不通过继承添加行为 | 装饰器（Decorator） |
| 全局唯一实例 | 单例（Singleton，Holder/枚举） |
| 状态变化通知依赖方 | 观察者（Observer） |
| 行为随状态变化 | 状态（State） |
| 复杂子系统简单入口 | 外观（Facade） |
| 请求管道/中间件 | 责任链（Chain of Responsibility） |
| 撤销/命令队列 | 命令（Command） |
| 部分-整体树形结构 | 组合（Composite） |
| 昂贵对象复用 | 享元（Flyweight）/ 原型（Prototype） |
| 接口不兼容 | 适配器（Adapter） |
| 访问控制/延迟初始化 | 代理（Proxy） |
| 多对象复杂通信 | 中介者（Mediator） |
| 保存/恢复状态 | 备忘录（Memento） |
| 对类层次结构添加操作 | 访问者（Visitor） |
| 求值语法/规则 | 解释器（Interpreter） |

---

## 参考

有关 Spring Boot 架构模式，请参见技能：`springboot-patterns`。
有关使用 Camel 和 Panache 的 Quarkus 架构模式，请参见技能：`quarkus-patterns`。
有关实体设计和查询优化，请参见技能：`jpa-patterns`。

### 单例模式

- 优先使用枚举单例或初始化持有者（Holder）模式，禁止使用非 `volatile` 字段的双重检查锁定
- DI 容器（Spring、Quarkus CDI）管理的无状态 Service Bean 默认为单例，不要手动实现该模式

```java
// 正确 —— Holder 模式（懒加载、线程安全、无同步开销）
public class AppConfig {
    private AppConfig() {}
    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }
    public static AppConfig getInstance() { return Holder.INSTANCE; }
}
```

### 策略模式

- 将变化的算法抽取到接口后面，通过构造函数注入策略
- 优先使用 `Map<String, Strategy>` 查找表代替 3 个分支以上的 `if-else` 或 `switch` 链

```java
// 错误 —— 长 if-else，对扩展封闭
if ("alipay".equals(type)) { ... }
else if ("wechat".equals(type)) { ... }

// 正确 —— 策略 Map，对扩展开放
Map<String, PaymentStrategy> strategies = Map.of(
    "alipay", new AlipayStrategy(),
    "wechat", new WechatStrategy()
);
strategies.get(type).pay(amount);
```

### 模板方法模式

- 用于骨架固定但步骤可变的算法
- 不需要状态时，优先使用接口 default 方法（Java 8+）代替抽象类

### 分层约束

- **Controller** —— 仅做入参校验和响应映射，禁止包含业务逻辑
- **Service** —— 业务逻辑；不直接处理 HTTP 或持久化细节
- **Repository/DAO** —— 仅做数据访问，禁止包含业务逻辑
- 跨层调用必须单向向下：Controller → Service → Repository，禁止跨层调用
- DTO 用于跨层传递数据；领域对象禁止泄漏到 Controller 层

## 参考

有关 Spring Boot 架构模式，请参见技能：`springboot-patterns`。
有关使用 Camel 和 Panache 的 Quarkus 架构模式，请参见技能：`quarkus-patterns`。
有关实体设计和查询优化，请参见技能：`jpa-patterns`。

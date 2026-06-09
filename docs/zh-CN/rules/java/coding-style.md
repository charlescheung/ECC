---
paths:
  - "**/*.java"
---

# Java 编码风格

> 本文档基于 [common/coding-style.md](../common/coding-style.md)，补充了 Java 特有的内容。

## 格式

* 使用 **google-java-format** 或 **Checkstyle**（Google 或 Sun 风格）进行强制规范
* 每个文件只包含一个顶层的公共类型
* 保持一致的缩进：2 或 4 个空格（遵循项目标准）
* 成员顺序：常量、字段、构造函数、公共方法、受保护方法、私有方法

## 不可变性

* 对于值类型，优先使用 `record`（Java 16+）
* 默认将字段标记为 `final` —— 仅在需要时才使用可变状态
* 从公共 API 返回防御性副本：`List.copyOf()`、`Map.copyOf()`、`Set.copyOf()`
* 写时复制：返回新实例，而不是修改现有实例

```java
// GOOD — immutable value type
public record OrderSummary(Long id, String customerName, BigDecimal total) {}

// GOOD — final fields, no setters
public class Order {
    private final Long id;
    private final List<LineItem> items;

    public List<LineItem> getItems() {
        return List.copyOf(items);
    }
}
```

## 命名

遵循标准的 Java 命名约定：

* `PascalCase` 用于类、接口、记录、枚举
* `camelCase` 用于方法、字段、参数、局部变量
* `SCREAMING_SNAKE_CASE` 用于 `static final` 常量
* 包名：全小写，使用反向域名（`com.example.app.service`）

## 现代 Java 特性

在能提高代码清晰度的地方使用现代语言特性：

* **记录** 用于 DTO 和值类型（Java 16+）
* **密封类** 用于封闭的类型层次结构（Java 17+）
* 使用 `instanceof` 进行**模式匹配** —— 避免显式类型转换（Java 16+）
* **文本块** 用于多行字符串 —— SQL、JSON 模板（Java 15+）
* 使用箭头语法的**Switch 表达式**（Java 14+）
* **Switch 中的模式匹配** —— 用于处理密封类型的穷举情况（Java 21+）

```java
// Pattern matching instanceof
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}

// Sealed type hierarchy
public sealed interface PaymentMethod permits CreditCard, BankTransfer, Wallet {}

// Switch expression
String label = switch (status) {
    case ACTIVE -> "Active";
    case SUSPENDED -> "Suspended";
    case CLOSED -> "Closed";
};
```

## Optional 的使用

* 从可能没有结果的查找方法中返回 `Optional<T>`
* 使用 `map()`、`flatMap()`、`orElseThrow()` —— 绝不直接调用 `get()` 而不先检查 `isPresent()`
* 绝不将 `Optional` 用作字段类型或方法参数

```java
// GOOD
return repository.findById(id)
    .map(ResponseDto::from)
    .orElseThrow(() -> new OrderNotFoundException(id));

// BAD — Optional as parameter
public void process(Optional<String> name) {}
```

## 错误处理

* 对于领域错误，优先使用非受检异常
* 创建扩展自 `RuntimeException` 的领域特定异常
* 避免宽泛的 `catch (Exception e)`，除非在最顶层的处理器中
* 在异常消息中包含上下文信息

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: id=" + id);
    }
}
```

## 流

* 使用流进行转换；保持流水线简短（最多 3-4 个操作）
* 在可读性好的情况下，优先使用方法引用：`.map(Order::getTotal)`
* 避免在流操作中产生副作用
* 对于复杂逻辑，优先使用循环而不是难以理解的流流水线

## 阿里巴巴 Java 开发规范

以下规则源自**阿里巴巴 Java 开发手册**，适用于本项目所有 Java 代码。

### 命名规范

- 类名必须使用 `PascalCase`；抽象类必须以 `Abstract` 开头；异常类必须以 `Exception` 结尾；测试类必须以 `Test` 结尾
- 方法名、参数、局部变量必须使用 `camelCase`；布尔字段**禁止**以 `is` 开头（与 POJO getter 命名规则冲突）
- 常量必须使用 `UPPER_SNAKE_CASE` 并声明为 `static final`
- 包名全部小写，优先使用单个英文单词，不使用复数形式
- 禁止使用已有 JDK 类名作为标识符（如 `String`、`Thread`）
- 数组声明风格：`String[] args`，**禁止** `String args[]`

```java
// 错误 —— 布尔字段使用 "is" 前缀会导致 getter 命名冲突
private Boolean isDeleted;   // 某些工具会生成 isIsDeleted()

// 正确
private Boolean deleted;
```

### 注释规范

- 所有公共类及公共/受保护方法**必须**有 Javadoc
- Javadoc 必须包含 `@param` 和 `@return`；受检异常必须有 `@throws`
- 单行注释写在代码行上方，不写在行尾；缩进与所注释语句对齐
- 合并前必须删除 TODO/FIXME 注释，转换为可追踪的 Issue
- 方法体内部不使用 `/* */` 多行注释，使用 `//` 单行注释

```java
/**
 * 为指定客户下订单。
 *
 * @param request 订单创建请求，不能为 null
 * @return 已保存的订单摘要
 * @throws InsufficientStockException 当任意商品库存不足时抛出
 */
public OrderSummary placeOrder(CreateOrderRequest request) { ... }
```

### OOP 规约

- 重写 `equals()` 时必须同时重写 `hashCode()`，两者要么同时重写，要么都不重写
- 禁止用 `==` 比较 `String` 或包装类型；使用 `equals()` 或 `Objects.equals()`
- 循环内禁止使用 `+` 拼接字符串；使用 `String.format()`、文本块或 `StringBuilder`
- 领域对象必须重写 `toString()`，禁止使用默认的地址形式
- 避免深层继承（超过 3 层）；优先使用组合代替继承
- 所有重写方法必须加 `@Override`
- 静态成员必须通过类名访问，禁止通过实例访问

```java
// 错误 —— 循环内字符串拼接，O(n²) 内存分配
String result = "";
for (String s : list) {
    result = result + s;
}

// 正确
String result = String.join("", list);
```

### 集合处理

- 创建 `HashMap`/`ArrayList` 时必须指定初始容量：`new HashMap<>(expectedSize * 4 / 3 + 1)`
- 同时需要 key 和 value 时使用 `entrySet()` 迭代，禁止 `keySet()` + `get()` 方式
- `subList()` 返回的是视图，禁止序列化或跨层传递；需先用 `new ArrayList<>()` 包装
- 禁止在 for-each 迭代集合时修改集合；使用 `Iterator.remove()` 或 stream `filter()`
- 对外暴露的只读集合使用 `Collections.unmodifiableList/Set/Map` 或 `List.of()` / `Map.of()`
- 优先使用 `Map.getOrDefault()`、`computeIfAbsent()`、`putIfAbsent()` 代替 null 检查模式

```java
// 错误 —— keySet 迭代，存在重复查找
for (String key : map.keySet()) {
    process(key, map.get(key));
}

// 正确
for (Map.Entry<String, Value> entry : map.entrySet()) {
    process(entry.getKey(), entry.getValue());
}
```

### 并发规约

- 线程和线程池必须有有意义的名称，使用 `ThreadFactory` 设置
- 必须直接使用 `ThreadPoolExecutor` 创建线程池，禁止使用 `Executors` 工厂方法（存在隐藏的无界队列风险）
- 共享可变状态必须受保护，使用 `volatile`、`AtomicXxx`、`synchronized` 或 `java.util.concurrent`
- `SimpleDateFormat` 非线程安全，使用不可变的 `DateTimeFormatter` 替代
- 锁的获取与释放必须在同一方法中，并在 `try/finally` 中完成

```java
// 错误 —— Executors 隐藏了无界队列
ExecutorService pool = Executors.newFixedThreadPool(10);

// 正确 —— 显式指定边界和拒绝策略
ExecutorService pool = new ThreadPoolExecutor(
    4, 10, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(200),
    new ThreadFactoryBuilder().setNameFormat("order-worker-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 异常处理

- 禁止静默吞掉异常，至少要记录完整的堆栈日志
- 禁止用异常控制流程，异常开销大且会掩盖意图
- 捕获最具体的异常类型；除顶层处理器外避免 `catch (Exception e)`
- 重新抛出时附带上下文：`throw new ServiceException("上下文信息", cause)`
- `finally` 块中禁止使用 `return` 语句，会导致异常被吞掉

### 日志规范

- 使用 `{}` 占位符，禁止在日志语句中使用字符串拼接
- 对于开销大的日志参数，使用 `log.isDebugEnabled()` 保护
- 禁止记录敏感数据（密码、token、卡号、个人信息）
- 日志级别：ERROR 用于需要立即处理的故障；WARN 用于可恢复问题；INFO 用于业务事件；DEBUG 用于诊断详情

```java
// 错误 —— 字符串拼接，始终执行
log.debug("处理订单: " + order.toDetailString());

// 正确 —— 仅在 DEBUG 开启时才求值
log.debug("处理订单: {}", order.getId());
```

## 参考

完整编码标准及示例，请参阅技能：`java-coding-standards`。
JPA/Hibernate 实体设计模式，请参阅技能：`jpa-patterns`。

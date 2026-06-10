---
paths:
  - "**/*.java"
---
# Java Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Java-specific content.

## Formatting

- **google-java-format** or **Checkstyle** (Google or Sun style) for enforcement
- One public top-level type per file
- Consistent indent: 2 or 4 spaces (match project standard)
- Member order: constants, fields, constructors, public methods, protected, private

## Immutability

- Prefer `record` for value types (Java 16+)
- Mark fields `final` by default — use mutable state only when required
- Return defensive copies from public APIs: `List.copyOf()`, `Map.copyOf()`, `Set.copyOf()`
- Copy-on-write: return new instances rather than mutating existing ones

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

## Naming

Follow standard Java conventions:
- `PascalCase` for classes, interfaces, records, enums
- `camelCase` for methods, fields, parameters, local variables
- `SCREAMING_SNAKE_CASE` for `static final` constants
- Packages: all lowercase, reverse domain (`com.example.app.service`)

## Modern Java Features

Use modern language features where they improve clarity:
- **Records** for DTOs and value types (Java 16+)
- **Sealed classes** for closed type hierarchies (Java 17+)
- **Pattern matching** with `instanceof` — no explicit cast (Java 16+)
- **Text blocks** for multi-line strings — SQL, JSON templates (Java 15+)
- **Switch expressions** with arrow syntax (Java 14+)
- **Pattern matching in switch** — exhaustive sealed type handling (Java 21+)

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

## Optional Usage

- Return `Optional<T>` from finder methods that may have no result
- Use `map()`, `flatMap()`, `orElseThrow()` — never call `get()` without `isPresent()`
- Never use `Optional` as a field type or method parameter

```java
// GOOD
return repository.findById(id)
    .map(ResponseDto::from)
    .orElseThrow(() -> new OrderNotFoundException(id));

// BAD — Optional as parameter
public void process(Optional<String> name) {}
```

## Error Handling

- Prefer unchecked exceptions for domain errors
- Create domain-specific exceptions extending `RuntimeException`
- Avoid broad `catch (Exception e)` unless at top-level handlers
- Include context in exception messages

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: id=" + id);
    }
}
```

## Streams

- Use streams for transformations; keep pipelines short (3-4 operations max)
- Prefer method references when readable: `.map(Order::getTotal)`
- Avoid side effects in stream operations
- For complex logic, prefer a loop over a convoluted stream pipeline

## Alibaba Java Development Specification

The following rules are derived from the **Alibaba Java Coding Guidelines** and apply to all Java code in this project.

### Naming (命名规范)

- Class names must be `PascalCase`; abstract classes must be prefixed with `Abstract`; exception classes must be suffixed with `Exception`; test classes must be suffixed with `Test`
- Method names, parameters, and local variables must be `camelCase`; boolean fields must **not** be prefixed with `is` (POJO getter convention conflict)
- Constants must be `UPPER_SNAKE_CASE` and declared `static final`
- Packages must be all lowercase; single English words preferred — no plural forms
- Avoid using existing JDK class names (e.g. `String`, `Thread`) as identifiers
- Array declaration style: `String[] args`, **not** `String args[]`

```java
// BAD — boolean field with "is" prefix causes getter name collision
private Boolean isDeleted;   // generates isIsDeleted() in some tools

// GOOD
private Boolean deleted;
```

### Comment Standards (注释规范)

- All public classes and public/protected methods **must** have Javadoc
- Javadoc `@param` and `@return` are required; `@throws` required for checked exceptions
- Single-line comments above the code line, not at the end; indent to match the statement
- Remove TODO/FIXME comments before merging — convert to tracked issues
- Do not use `/* */` multi-line comments inside method bodies; use `//` line comments

```java
/**
 * Places an order for the given customer.
 *
 * @param request the order creation request; must not be null
 * @return the saved order summary
 * @throws InsufficientStockException if any item is out of stock
 */
public OrderSummary placeOrder(CreateOrderRequest request) { ... }
```

### OOP Rules (OOP规约)

- Override `equals()` only alongside `hashCode()` — always both or neither
- Never use `==` to compare `String` or boxed primitives; use `equals()` or `Objects.equals()`
- Use `String.format()` or text blocks for string concatenation inside loops — never `+` in a loop
- `toString()` must be overridden in domain objects; do not let it default to the address form
- Avoid deep inheritance chains (>3 levels); prefer composition over inheritance
- All overriding methods must carry `@Override`
- Static members must be accessed via the class name, never through an instance

```java
// BAD — string concat in loop
String result = "";
for (String s : list) {
    result = result + s;   // O(n²) allocations
}

// GOOD
String result = String.join("", list);
// or StringBuilder for conditional logic
```

### Collection Rules (集合处理)

- Always specify initial capacity when creating `HashMap`/`ArrayList`: `new HashMap<>(expectedSize * 4 / 3 + 1)`
- Use `entrySet()` for iteration when both key and value are needed; never `keySet()` + `get()`
- `subList()` returns a view — do not serialize or pass it across layer boundaries; wrap with `new ArrayList<>()` first
- Do not modify a collection while iterating it with a for-each; use `Iterator.remove()` or stream `filter()`
- Use `Collections.unmodifiableList/Set/Map` or `List.of()` / `Map.of()` for read-only collections passed out of a class
- `Map.getOrDefault()`, `computeIfAbsent()`, `putIfAbsent()` are preferred over null-check patterns

```java
// BAD — keySet iteration with redundant lookup
for (String key : map.keySet()) {
    process(key, map.get(key));
}

// GOOD
for (Map.Entry<String, Value> entry : map.entrySet()) {
    process(entry.getKey(), entry.getValue());
}
```

### Concurrency (并发规约)

- Thread and thread-pool names must be meaningful — use `ThreadFactory` to set names
- Always use `ThreadPoolExecutor` directly rather than `Executors` factory shortcuts (avoids hidden unbounded queues)
- Shared mutable state must be protected — use `volatile`, `AtomicXxx`, `synchronized`, or `java.util.concurrent` classes
- `SimpleDateFormat` is not thread-safe — use `DateTimeFormatter` (immutable) instead
- Lock acquisition and release must occur in the same method and in `try/finally`

```java
// BAD — Executors hides an unbounded queue
ExecutorService pool = Executors.newFixedThreadPool(10);

// GOOD — explicit bounds and rejection policy
ExecutorService pool = new ThreadPoolExecutor(
    4, 10, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(200),
    new ThreadFactoryBuilder().setNameFormat("order-worker-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### Exception Handling (异常处理)

- Never swallow exceptions silently — at minimum log with full stack trace
- Do not use exceptions for flow control; they are expensive and obscure intent
- Catch the most specific exception type available; avoid `catch (Exception e)` except at top-level handlers
- When rethrowing, wrap with context: `throw new ServiceException("context message", cause)`
- `finally` blocks must not contain `return` statements — they suppress exceptions

### Logging (日志规范)

- Use `{}` placeholders — never string concatenation in log statements
- Guard expensive log arguments with `log.isDebugEnabled()` checks
- Do not log sensitive data (passwords, tokens, card numbers, PII)
- Log level guidance: ERROR for failures requiring immediate action; WARN for recoverable issues; INFO for business events; DEBUG for diagnostic detail

```java
// BAD — string concat always evaluated
log.debug("Processing order: " + order.toDetailString());

// GOOD — evaluated only when DEBUG is enabled
log.debug("Processing order: {}", order.getId());
```

## References

See skill: `java-coding-standards` for full coding standards with examples.
See skill: `jpa-patterns` for JPA/Hibernate entity design patterns.

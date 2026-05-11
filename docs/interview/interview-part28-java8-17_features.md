# Java 8~17 新特性面试题汇总

> 本篇覆盖 Java 8 至 Java 17 最核心的新特性，涵盖 Stream API、Lambda、Optional、CompletableFuture、模块系统、Record、Sealed Classes、GC 演进等 20 个高频面试主题。每题包含题目、核心答案、追问方向、避坑提示。

---

## 1. Java 8 Stream API：vs Collection、串行/并行流

### 题目

Java 8 Stream API 与传统 Collection 集合的区别是什么？串行流与并行流 `parallelStream()` 的底层差异是什么？

### 核心答案

| 对比维度 | Collection | Stream |
|---|---|---|
| **数据来源** | 自身存储元素 | 描述计算逻辑，不存储数据 |
| **惰性求值** | 即时求值（迭代器立即遍历） | 惰性求值（Terminal Op 才触发计算） |
| **迭代方式** | 外部迭代（用户自己 for/while） | 内部迭代（库内部控制迭代） |
| **只能遍历一次** | 否（可重复迭代） | 是（流是一次性的） |
| **副作用** | 可变accumulator | 推荐无状态/非干扰 |

```java
// 串行流
List<String> list = Arrays.asList("a", "b", "c");
list.stream().filter(s -> s.startsWith("a")).collect(Collectors.toList());

// 并行流 — 底层 ForkJoinPool.commonPool()
list.parallelStream()
    .filter(s -> s.startsWith("a"))
    .collect(Collectors.toList());
```

**串行流**：单线程顺序执行，遵循 `源 → 中间操作 → 终端操作` 管道。
**并行流**：底层使用 `ForkJoinPool.commonPool()`，默认线程数为 `Runtime.getRuntime().availableProcessors() - 1`（至少为 1），数据切分成多个 chunk 分发到不同线程并行处理，最后合并结果。

### 追问方向

- 并行流使用的默认线程池可以替换吗？→ 可以通过 `System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "4")` 调整，但这是** JVM 全局设置**，影响所有并行流。
- 流的惰性求值如何实现？→ 依赖 `Spliterator` 的 tryAdvance / trySplit，split 产生子块供并行使用。
- 什么场景下并行流反而更慢？→ 数据量小、计算简单、线程池创建开销大于计算收益时。

### 避坑提示

- 并行流并非万能加速药，数据量不够大或操作本身很快时，线程切换开销反而拖累性能。
- 并行流操作如果涉及**共享变量**，必须有线程同步措施，否则结果不确定。
- 顺序流调用 `parallel()` 可中途切换为并行，调用 `sequential()` 可切回串行，但连续调用以**最后一次**为准。

---

## 2. Lambda 表达式：函数式接口、捕获规则、方法引用

### 题目

Lambda 表达式的核心概念是什么？函数式接口有哪些？Lambda 的变量捕获规则是什么？方法引用 `::` 有哪几种形式？

### 核心答案

**函数式接口**：有且仅有一个抽象方法的接口（Object 的 public 方法不计入）。常见例子：

| 接口 | 抽象方法 | 签名 |
|---|---|---|
| `java.util.function.Predicate<T>` | `test(T t) → boolean` | T→boolean |
| `java.util.function.Function<T,R>` | `apply(T t) → R` | T→R |
| `java.util.function.Consumer<T>` | `accept(T t) → void` | T→void |
| `java.util.function.Supplier<T>` | `get() → T` | ()→T |
| `java.util.function.UnaryOperator<T>` | `apply(T t) → T` | T→T |
| `java.util.function.BinaryOperator<T>` | `apply(T t, T u) → T` | (T,T)→T |
| `Runnable` | `run() → void` | ()→void |
| `Callable<V>` | `call() → V` | ()→V |

**变量捕获规则**：

1. **局部变量**：必须是 effectively final（即未显式修改）或 final 的才能捕获。Java 8 之前匿名内部类也有此限制。
2. **实例字段**：捕获 `this` 引用，无需 effectively final。
3. **静态变量**：无限制。
4. **注意**：Lambda 本身不是独立作用域，`this` 指向**包围它的那个类实例**（不同于匿名内部类）。

```java
int x = 10;           // effectively final
Predicate<Integer> p = n -> n > x;  // OK，捕获 x

// x = 20;            // 取消注释则编译错误——x 不再是 effectively final
```

**方法引用四种形式**：

```java
// 1. 类名 :: 静态方法
Function<String, Integer> f = Integer::parseInt;

// 2. 类名 :: 实例方法（instance method of object, formal param is receiver）
BiPredicate<String, String> bp = String::equals;
// 等价于 (s1, s2) -> s1.equals(s2)，s1 是接收者，s2 是参数

// 3. 实例 :: 实例方法
String s = "hello";
Supplier<Integer> slen = s::length;
// 等价于 () -> s.length()

// 4. 类名 :: new（构造器引用）
Function<Integer, String[]> arrFactory = String[]::new;
// 等价于 n -> new String[n]
```

### 追问方向

- `@FunctionalInterface` 注解的作用？→ 编译期检查接口是否符合函数式接口定义，若有多于一个抽象方法则编译错误。
- Lambda 表达式在字节码层面是什么？→ 编译成 `invokedynamic` 指令，首次调用时由 `LambdaMetafactory` 生成一个内部类（lambda 对象的实际类）。
- 为什么局部变量捕获要求 effectively final？→ 底层是值拷贝（虽然语法上像引用），防止多线程下的数据竞争和内存可见性问题。

### 避坑提示

- Lambda 捕获局部变量时，**不要在 Lambda 表达式内部修改该局部变量**，这在编译期就会报错（Java 8+）。
- 警惕在 Lambda 中捕获**非线程安全对象**（如 ArrayList）并随后在多线程中使用——Lambda 本身是无状态的，但若引用的对象本身共享且可变，就会出问题。
- 方法引用 `类名::实例方法` 的签名容易写错，确认 receiver 和参数对应关系。

---

## 3. Optional 类：of/empty/ofNullable、map/flatMap/filter、链式调用

### 题目

`java.util.Optional` 的创建方式有哪些？`map/flatMap/filter` 的区别是什么？如何正确使用 Optional 避免空指针？

### 核心答案

**创建方式**：

```java
Optional<String> empty   = Optional.empty();
Optional<String> nonNull = Optional.of("value");       // null 传入则 NPE
Optional<String> nullable = Optional.ofNullable(null); // 允许 null，转化为 empty
```

**三剑客方法**：

| 方法 | 入参 | 返回类型 | 语义 |
|---|---|---|---|
| `map` | `Function<T, U>` | `Optional<U>` | 若值存在则应用函数，否则返回 empty |
| `flatMap` | `Function<T, Optional<U>>` | `Optional<U>` | 若值存在则应用函数并**展平**，否则返回 empty |
| `filter` | `Predicate<T>` | `Optional<T>` | 若值存在且满足断言则保留，否则返回 empty |

```java
// map vs flatMap 区别
Optional<String> name = Optional.of("hello");

Optional<Optional<String>> wrapped = name.map(s -> Optional.of(s.toUpperCase()));
Optional<String> flattened = name.flatMap(s -> Optional.of(s.toUpperCase()));

// flatMap 避免嵌套 Optional——正确用法
Optional<String> result = Optional.ofNullable(user)
    .flatMap(User::getAddress)   // User::getAddress 返回 Optional<Address>
    .flatMap(Address::getCity);  // Address::getCity 返回 Optional<City>
```

**链式调用典型模式**：

```java
String city = Optional.ofNullable(user)
    .map(User::getName)              // T -> Optional<U>... 哦不，map
    .flatMap(u -> Optional.ofNullable(u.getAddress()))
    .map(a -> a.getCity())
    .orElse("未知");                  // 提供默认值
```

### 追问方向

- `orElse` vs `orElseGet` vs `orElseThrow`？→ `orElse` 无论是否为空都会执行（参数求值），`orElseGet(Supplier)` 仅在空时才执行（惰性），`orElseThrow` 空时抛指定异常。
- Optional 能否被序列化？→ **不能**直接序列化（Java 9 之前），Java 9+ 有 `Optional.empty()` 等特定实现但仍不建议序列化。替代方案：用 `Optional` 作为字段类型，但在序列化前手动转成实际值。
- Optional 在集合中的使用？→ `List<Optional<T>>` 可以表示"可能缺值"的列表，但更推荐用 `Map<K, Optional<V>>` 表示可选值映射。

### 避坑提示

- **永远不要**用 `Optional` 作为方法参数类型——这会让 API 变得笨重且不兼容。Optional 设计初衷是**返回值**类型。
- 不要在 `map` 里返回 `Optional`，应该用 `flatMap`，否则会得到 `Optional<Optional<T>>`。
- 避免滥用 `isPresent() + get()` 组合——这等同于 `if (obj != null)`，失去了 Optional 的语义优势。应该用 `orElse / orElseGet / map`。
- Optional 在 Java 8 中没有 `stream()` 方法（Java 9 才有），Java 8 如果需要把 Optional 转 Stream 要手动处理。

---

## 4. 接口默认方法：default 方法、静态方法、多重继承冲突解决

### 题目

Java 8 为什么允许接口有 default 方法？静态方法的作用是什么？当一个类同时继承两个接口，且两个接口都有同签名默认方法时，编译器如何解决冲突？

### 核心答案

**default 方法引入原因**：在不破坏向后兼容性的前提下，为已有接口添加新方法。集合框架大量使用（如 `Collection.stream()`、`List.sort()`）。

```java
public interface Animal {
    default void speak() {
        System.out.println("...");
    }
}
```

**静态方法**：提供接口级别的工具方法，不需要实现类实例即可调用。经典例子 `List.of()`、`Comparator.comparing()`。

```java
public interface Color {
    static Color rgb(int r, int g, int b) {
        return () -> (r << 16) | (g << 8) | b;
    }
}
```

**多重继承冲突解决规则（Java 8+）**：

1. **类的方法优先于接口的默认方法**。
2. 如果没有类的方法胜出，则**子类必须显式选择**要使用的默认方法（通过 `InterfaceName.super.methodName()`）。
3. 如果两个接口**都没有默认实现**（或都是抽象的），不存在冲突，无需解决。

```java
public interface A { default void hello() { System.out.println("A"); } }
public interface B { default void hello() { System.out.println("B"); } }

public class C implements A, B {
    @Override
    public void hello() {
        // 必须显式选择，否则编译错误
        A.super.hello();  // 选择 A 的默认实现
        // 或 B.super.hello();
        // 或完全重写
    }
}
```

### 追问方向

- 接口可以有私有方法吗（Java 9+）？→ 可以。`private default` 和 `private static` 方法用于在接口内部代码复用，但不能被实现类继承。
- default 方法能否被实现类标记为 `@Override`？→ 可以，实现类可以重写（override）default 方法。
- 多个接口的 default 方法形成钻石继承时，子类没有重写，调用哪个？→ 编译错误，子类必须显式选择。

### 避坑提示

- **不要用 default 方法来设计 API 的主要抽象**——它只是兼容性补丁。核心抽象仍然应该用 abstract class。
- default 方法**不是真正的多继承**，因为实现类只能继承一个类的状态（字段），接口没有实例字段。
- 当接口的 default 方法依赖的抽象方法被移除/修改时，可能导致运行时行为变化，要注意版本兼容性。

---

## 5. Java 8 日期时间 API：LocalDateTime/ZoneId/ZonedDateTime、格式化

### 题目

`LocalDate`、`LocalTime`、`LocalDateTime`、`ZonedDateTime` 的区别是什么？如何处理不同时区的时间转换？日期格式化有哪些坑？

### 核心答案

| 类型 | 包含信息 | 用途 |
|---|---|---|
| `LocalDate` | 年-月-日 | 本地日期，无时区 |
| `LocalTime` | 时:分:秒[.纳秒] | 本地时间，无时区 |
| `LocalDateTime` | 年-月-日 时:分:秒 | 本地日期时间，无时区 |
| `ZonedDateTime` | 年-月-日 时:分:秒 +时区 | 带时区的完整时间点 |
| `OffsetDateTime` | 年-月-日 时:分:秒 +偏移量 | 带 UTC 偏移的时间点 |

```java
LocalDate date = LocalDate.now();
LocalTime time = LocalTime.now();
LocalDateTime ldt = LocalDateTime.now();
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
ZonedDateTime utc = ZonedDateTime.now(ZoneId.of("UTC"));

// 不同时区转换
ZonedDateTime shanghaiTime = ZonedDateTime.of(ldt, ZoneId.of("Asia/Shanghai"));
Instant instant = zdt.toInstant();  // ZonedDateTime -> Instant
ZonedDateTime fromInstant = instant.atZone(ZoneId.of("UTC"));
```

**格式化**：

```java
// DateTimeFormatter（线程安全，推荐）
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String str = ldt.format(formatter);
LocalDateTime parsed = LocalDateTime.parse(str, formatter);

// 预定义格式
LocalDate.parse("2024-01-01", DateTimeFormatter.ISO_LOCAL_DATE);
```

### 追问方向

- `Instant` 是什么？→ 时间线上的一点（自 1970-01-01T00:00:00Z 的秒/纳秒），等价于 `ZonedDateTime.ofInstant(..., UTC)`，适合日志/持久化。
- 旧版 `java.util.Date` vs 新 API？→ `Date` 有时区陷阱（内部用 UTC，toString 用本地时区），新 API 一律时区显式处理。
- 如何处理"夏令时"？→ `ZonedDateTime` 自动处理，跨夏令时边界计算duration要用 `Duration.between()`，跨时区用 `ZonedDateTime.withZoneSameInstant()`。

### 避坑提示

- **不要混用 `java.util.Date` 和新的 `java.time` API**——除非显式转换。新代码一律用 `java.time`。
- `DateTimeFormatter` 是**线程安全**的（模板模式），但 `SimpleDateFormat` 不是——这是新 API 的重大改进之一。
- `LocalDateTime` 没有时区，`ZonedDateTime` 才是完整的时间点。时间点才能正确比较先后和计算 duration。
- 格式化pattern中的 `HH` 是24小时制，`hh` 是12小时制，`yyyy`（小写 y）是周年（week-based year），`YYYY`（大写）是 ISO 周年份——容易混淆。

---

## 6. CompletableFuture：异步编程、thenApply/combine/accept、异常处理

### 题目

`CompletableFuture` 相比传统 `Future` 的优势是什么？`thenApply`、`thenCompose`、`thenCombine`、`thenAccept` 的区别是什么？如何处理异步异常？

### 核心答案

`Future` 的局限：只能通过 `get()` 阻塞获取结果，无法链式组合，无法手动完成。

**`CompletableFuture` 核心优势**：
- 手动完成（`complete()` / `completeExceptionally()`）
- 链式组合（then* 系列）
- 多个 Future 组合（`thenCombine` / `allOf` / `anyOf`）
- 回调式（非阻塞）

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    // 异步执行的任务
    return fetchData();
}).thenApply(data -> {
    // 上一个结果变换（同步）
    return process(data);
}).exceptionally(ex -> {
    // 异常处理，返回默认值
    return "default";
});
```

**方法对比**：

| 方法 | 入参函数签名 | 返回类型 | 语义 |
|---|---|---|---|
| `thenApply(Function)` | `T → U` | `CompletableFuture<U>` | 转换结果 |
| `thenCompose(Function)` | `T → CompletableFuture<U>` | `CompletableFuture<U>` | 扁平化链式异步（如 flatMap） |
| `thenCombine(CompletionStage, BiFunction)` | `(T, U) → V` | `CompletableFuture<V>` | 两个 CF 都完成后合并 |
| `thenAccept(Consumer)` | `T → void` | `CompletableFuture<Void>` | 消费结果，不返回新值 |
| `thenRun(Runnable)` | `() → void` | `CompletableFuture<Void>` | 不依赖结果，只在完成后执行 |

```java
// thenCompose：异步链式调用（避免嵌套 CF）
CompletableFuture<User> userCF = CompletableFuture
    .supplyAsync(() -> getUserId())
    .thenCompose(userId -> getUserAsync(userId)); // 返回 CompletableFuture<User>

// thenCombine：并行任务合并
CompletableFuture<Integer> cf1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> cf2 = CompletableFuture.supplyAsync(() -> 20);
cf1.thenCombine(cf2, Integer::sum).thenAccept(System.out::println); // 输出 30
```

**异常处理**：

```java
// 方式1: exceptionally（只处理异常，不影响正常流程）
cf.exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return "fallback";
});

// 方式2: handle（无论成功失败都处理）
cf.handle((result, ex) -> {
    if (ex != null) {
        return "error: " + ex.getMessage();
    }
    return result;
});

// 方式3: whenComplete（不修改结果，仅观察）
cf.whenComplete((result, ex) -> {
    if (ex != null) System.err.println("Failed: " + ex);
    else System.out.println("Success: " + result);
});
```

### 追问方向

- `join()` vs `get()`？→ 两者都阻塞等待结果，`get()` 抛受检 `ExecutionException`，`join()` 抛 `CompletionException`（非受检），更方便。
- 默认线程池？→ `supplyAsync` / `runAsync` 默认使用 `ForkJoinPool.commonPool()`（并行流共享），也可以传入自定义 `Executor`。
- 如何取消一个 CompletableFuture？→ `cancel()` 内部调用 `completeExceptionally(new CancellationException())`，但不会中断正在执行的任务。

### 避坑提示

- **不要在异步任务中抛出受检异常**——`CompletionStage` 的方法签名不支持throws，异常会被包裹在 `CompletionException` 中。
- `thenApply` 是同步的（如果前一步已完成），真正异步用 `thenApplyAsync`。确认你需要同步还是异步执行。
- `allOf()` 所有 CF 都完成才继续（即使部分异常），`anyOf()` 任一完成就继续——注意选择。
- 默认 `ForkJoinPool` 的并行度是 `n-1`（n=CPU核数），不适合 IO 密集型任务（应该用更大的线程池）。

---

## 7. Stream API 高级操作：collect/groupingBy/partitioningBy/reducing

### 题目

`Collectors` 的 `groupingBy`、`partitioningBy`、`reducing` 的区别是什么？如何在收集过程中实现多级分组？

### 核心答案

```java
// partitioningBy：只分两组（true/false），基于 Predicate
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 5000));

// groupingBy：基于任意分类函数，支持多级分组
Map<Department, Map<City, List<Employee>>> multiLevel = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment,
                 Collectors.groupingBy(Employee::getCity)));

// reducing：归约操作
// 等价于 reduce，但作为 collector 可与其他收集器组合
Double avgSalary = employees.stream()
    .collect(Collectors.reducing(
        0.0,
        Employee::getSalary,
        Double::sum
    )) / employees.size();

// 或者用 averagingDouble 更简洁：
Double avg = employees.stream()
    .collect(Collectors.averagingDouble(Employee::getSalary));
```

**常见 Collectors 工厂方法**：

| 收集器 | 返回类型 | 作用 |
|---|---|---|
| `toList()` / `toSet()` / `toCollection(Supplier)` | `List<T>` / `Set<T>` / `Collection<T>` | 收集到集合 |
| `toMap(k, v)` / `toConcurrentMap(k, v)` | `Map<K,V>` | 收集到 Map |
| `groupingBy(k)` / `groupingBy(k, downstream)` | `Map<K, List<T>>` | 分组 |
| `partitioningBy(pred)` / `partitioningBy(pred, downstream)` | `Map<Boolean, List<T>>` | 二分分组 |
| `joining(delimiter)` | `String` | 字符串拼接 |
| `counting()` | `Long` | 计数 |
| `summingInt/Long/Double(fn)` | `Integer/Long/Double` | 求和 |
| `averagingInt/Long/Double(fn)` | `Double` | 平均值 |
| `maxBy(comparator)` / `minBy(comparator)` | `Optional<T>` | 最值 |
| `reducing(identity, mapper, combiner)` | `Optional<T>` | 自定义归约 |
| `collectingAndThen(collector, finisher)` | - | 收集后再处理 |

**多级分组 + 聚合**：

```java
Map<Department, Map<City, DoubleSummaryStatistics>> stats = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(
            Employee::getCity,
            Collectors.summarizingDouble(Employee::getSalary)
        )
    ));
```

### 追问方向

- `groupingBy` 的第三个参数 `Collector` 是什么？→ 下游收集器，控制每组的结果类型，如 `groupingBy(k, toSet())` 或 `groupingBy(k, counting())`。
- `Collectors.toMap` 的 key 冲突怎么处理？→ 需要 `mergeFunction` 参数：`(v1, v2) -> v1` 取前者、`(v1, v2) -> v2` 取后者、或其他逻辑。
- `partitioningBy` 可以用 `groupingBy` 实现吗？→ 可以，但 `partitioningBy` 性能更好（直接用 boolean key）。

### 避坑提示

- `groupingBy` 必须指定下游收集器，否则默认 `Collectors.toList()`，但显式写出更清晰。
- `toMap` / `toConcurrentMap` 如果 key 重复且未提供 merge 函数，会抛 `IllegalStateException`。
- `joining()` 只适用于 `Stream<String>`，`intStream` 要先 `mapToObj` 再 `joining`。

---

## 8. 方法引用：四种类名/实例引用形式

### 题目

方法引用 `类名::静态方法`、`实例::实例方法`、`类名::实例方法`、`类名::new` 的使用场景和函数签名是什么？

### 核心答案

```java
// 形式1：类名 :: 静态方法
// 签名：T -> R，参数类型 T，返回 R
Function<String, Integer> parser = Integer::parseInt;        // parseInt(String) -> Integer
IntBinaryOperator maxOf = Math::max;                          // max(int, int) -> int

// 形式2：实例 :: 实例方法
// 签名：() -> R 或 (T) -> R，取决于方法是否有参数
String str = "hello";
Supplier<Integer> len = str::length;                          // length() -> int，无参数
Consumer<String> printer = System.out::println;               // println(String) -> void
List<String> list = new ArrayList<>();
Predicate<String> containsA = list::contains;                 // contains(String) -> boolean

// 形式3：类名 :: 实例方法
// 签名：(T, T) -> R 或 (T, *) -> R，第一个参数是 receiver（方法作用的对象）
BiPredicate<String, String> eq = String::equals;              // (s1, s2) -> s1.equals(s2)
Function<String, String> upper = String::toUpperCase;         // (s) -> s.toUpperCase()
BinaryOperator<Integer> sum = Integer::sum;                    // (a, b) -> a + b
ToIntBiFunction<String, String> indexOf = String::indexOf;   // (s, sub) -> s.indexOf(sub)

// 形式4：类名 :: new（构造器引用）
// 签名：() -> T 或 (T) -> T 或 (T, U) -> T，取决于构造函数参数
Supplier<ArrayList<String>> listFactory = ArrayList::new;     // ArrayList()
Function<Integer, String[]> arrFactory = String[]::new;       // String[int]
BiFunction<String, Integer, BigDecimal> bdFactory = BigDecimal::new; // BigDecimal(String, int)
```

### 追问方向

- `String::toUpperCase` 是类名::实例方法，但它没有显式接收者，如何解释？→ `Function<String, String> f = String::toUpperCase` 相当于 `x -> x.toUpperCase()`，Java 根据目标类型自动将第一个参数作为 receiver。
- `Comparator.comparing` 与方法引用结合？→ `Comparator.comparing(Person::getName).thenComparing(Person::getAge)` 经典链式用法。
- 数组构造器引用 `String[]::new` 的参数是什么？→ 参数是 `int`（数组长度），所以是 `IntFunction<String[]>`。

### 避坑提示

- `实例::实例方法` 如果实例为 null，调用时才会 NPE，引用本身不检查。
- 混淆 `类型::实例方法` 和 `类型::静态方法`：编译器根据**目标类型（@FunctionalInterface）的抽象方法签名**来推断receiver是谁。
- 构造器引用 `类名::new` 匹配哪个构造函数由**目标类型**决定。例如 `Supplier<ArrayList>` 匹配无参构造，`Function<int, ArrayList>` 匹配 `ArrayList(int initialCapacity)`。

---

## 9. Stream 短路操作：anyMatch/allMatch/noneMatch/findFirst/findAny

### 题目

Stream 的短路操作（short-circuiting）是什么？`anyMatch`、`allMatch`、`noneMatch`、`findFirst`、`findAny` 的区别和使用场景是什么？

### 核心答案

短路操作：在遇到满足条件的元素后**立即停止**遍历，不需要处理所有元素（类似 && / || 的惰性求值）。只有**终端操作**才触发计算。

```java
// anyMatch：任一元素满足即返回 true（短路）
boolean hasAdult = people.stream().anyMatch(p -> p.getAge() >= 18);

// allMatch：全部满足才返回 true，遇到不满足即短路
boolean allAdult = people.stream().allMatch(p -> p.getAge() >= 18);

// noneMatch：全部不满足才返回 true，遇到满足即短路
boolean noneSmoker = people.stream().noneMatch(p -> p.isSmoker());

// findFirst：返回第一个元素（Optional），有序流返回确定第一个，无序流不确定
Optional<Person> first = people.stream().filter(p -> p.getAge() > 20).findFirst();

// findAny：返回任意一个满足条件的元素（Optional），并行下性能可能更好
Optional<Person> any = people.stream().filter(p -> p.getAge() > 20).findAny();
```

**区别**：
- `findFirst` 总是返回流中**第一个**匹配元素（顺序 stream 的物理顺序第一个）；并行时可能有额外开销（需确认顺序）。
- `findAny` 在**并行流中更高效**（不保证顺序，可能返回任何匹配），串行流中等同于 `findFirst`。
- `allMatch/noneMatch/anyMatch` 在**无匹配可确定时立即返回**，不需要全部遍历。

### 追问方向

- 为什么 `findAny` 在并行时比 `findFirst` 更优？→ `findFirst` 需要保持顺序，保证找到的是"第一个"，并行时需要额外同步/合并逻辑；`findAny` 可以直接返回任意一个完成的线程结果。
- `findFirst` / `findAny` 返回空 `Optional` 是什么时候？→ 流中没有匹配元素时。
- 短路操作和 `peek()` 的区别？→ `peek()` 是中间操作（惰性，不执行），用于调试；短路操作是终端操作。

### 避坑提示

- 对于**有序流**（如 `List.stream()`），使用 `findFirst` 是确定的；对于**并行计算**且不关心顺序时，用 `findAny` 可能更快。
- `allMatch` / `noneMatch` 在流为空时行为：`allMatch` 永远返回 `true`（vacuous truth），`noneMatch` 永远返回 `true`（vacuous truth）——记住这个特殊情况。
- 不要在 `anyMatch/allMatch/noneMatch` 的 Predicate 中引入副作用（修改共享变量），因为流是惰性的，执行时机不确定。

---

## 10. 并行流：ForkJoinPool、线程数设置、线程安全问题

### 题目

并行流 `parallelStream()` 的底层线程池是什么？如何设置线程数？在并行流操作中哪些写法会导致线程安全问题？

### 核心答案

**ForkJoinPool**：

- `ForkJoinPool.commonPool()` 是所有并行流共享的全局线程池，使用**工作窃取（work-stealing）**算法。
- 默认并行度 = `Runtime.getRuntime().availableProcessors() - 1`，最少为 1。
- 线程数量**不是**固定的，是按需创建的。

```java
// 查询默认并行度
System.out.println(ForkJoinPool.commonPool().getParallelism()); // e.g., 7 (8核-1)

// 全局修改并行度（JVM 参数或代码）
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "16");
ForkJoinPool pool = new ForkJoinPool(8); // 自定义池，给特定并行流用
pool.submit(() -> myStream.parallel().collect(toList())).join();
```

**线程安全问题**：

并行流操作必须是**无状态、不相互干扰、不修改共享状态**的。

```java
// ❌ 错误：并发修改共享变量
List<Integer> result = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(result::add); // 不安全，结果不确定

// ✅ 正确：使用线程安全的收集器
List<Integer> safeResult = IntStream.range(0, 1000).parallel()
    .boxed()
    .collect(Collectors.toList());

// ❌ 错误：依赖于顺序的副作用
Set<Integer> seen = new HashSet<>();
list.parallelStream().forEach(x -> {
    if (!seen.add(x)) throw new RuntimeException("Duplicate: " + x);
}); // HashSet.add 不是线程安全的，抛出异常

// ✅ 正确：使用 ConcurrentHashMap 或 groupingBy
Map<Boolean, Long> count = list.parallelStream()
    .collect(Collectors.groupingBy(k -> k, Collectors.counting()));
```

### 追问方向

- `forEachOrdered` vs `forEach`？→ `forEachOrdered` 保证顺序但牺牲并行性能；`forEach` 并行但不保证顺序。
- 自定义线程池给并行流用？→ `parallelStream()` 不接受 Executor 参数，但可以包装：`.parallel().collect(Collectors.toList())` 通过 ForkJoinPool 的子类型可以传。
- 哪些收集器天生线程安全？→ `toList()`、`toSet()`、`toCollection(ConcurrentHashMap::newKeySet)` 等，**不**是所有都线程安全。

### 避坑提示

- `ArrayList` 不是线程安全的，但在并行收集时用 `.collect(Collectors.toList())` 是安全的（内部用 `ArrayList`，正确同步）。
- **避免在并行流中使用 `forEachOrdered`**，这会强制串行化，失去并行优势。
- 并行流和 `ThreadLocal` 一起用时要小心——`ThreadLocal` 本身是线程隔离的，但如果 `ForkJoinPool` 的线程被复用，`ThreadLocal` 残留值可能造成问题。
- 永远不要在并行流中使用 `System.out.println`——输出交错，且 `PrintStream` 同步开销大。

---

## 11. Java 9 新特性：模块系统（JPMS）、接口私有方法、钻石操作符

### 题目

Java 9 的模块系统（JPMS / Project Jigsaw）是什么？模块的 `requires`、`exports`、`opens` 区别是什么？Java 9 对接口和钻石操作符做了哪些改进？

### 核心答案

**模块系统（JPMS）**：

```java
// module-info.java 放在源码根目录
module com.myapp {
    // 依赖其他模块
    requires com.example.utils;        // 强制依赖（编译+运行）
    requires static com.example.logger; // 编译时依赖，运行可选

    // 导出包（使对其他模块可见）
    exports com.myapp.api;
    exports com.myapp.service;

    // 开放包给反射访问（Java 9+ 强封装）
    opens com.myapp.internal;          // 允许 deep reflection
    opens com.myapp.internal to com.example.testing; // 仅限指定模块
}
```

**关键概念**：
- `requires` — 声明模块依赖。
- `exports` — 导出包（仅编译时可见）。
- `opens` / `open` — 开放包供运行时反射（`open` 是整个模块开放，`opens` 是特定包）。
- `uses` — 声明服务使用（如 `requires` 服务接口）。
- `provides ... with` — 提供服务实现（如 `provides Service with Impl`）。

**接口私有方法（Java 9）**：

```java
public interface MyInterface {
    default void defaultMethod() {
        // 复用逻辑，私有方法仅在接口内部调用
        helper();
        System.out.println("default method");
    }

    private void helper() {
        // Java 9 允许的私有方法，用于代码复用
        System.out.println("helper logic");
    }

    static void staticMethod() {
        staticHelper();
    }

    private static void staticHelper() {
        // 静态私有方法，供同一接口的静态方法调用
    }
}
```

**钻石操作符改进**：

```java
// Java 7/8：钻石操作符可推断泛型，但不能用于匿名类
List<String> list7 = new ArrayList<>(); // OK
// MyClass<T> obj = new MyClass<>() { ... }; // ❌ 编译错误

// Java 9：钻石操作符可用于匿名类
MyClass<T> obj = new MyClass<>() {  // Java 9+ OK
    @Override
    public void doSomething(T t) { }
};
```

### 追问方向

- 为什么 `exports` 和 `opens` 不同？→ `exports` 是编译时可见性，`opens` 是运行时反射可见性（反射可以访问 private 成员）。
- `requires transitive` 做什么？→ 传递依赖，引入模块同时也给下游模块提供了依赖。
- Java 9 对 `sun.*` 包的影响？→ 大部分 `sun.*` 不再可访问，模块系统强封装。

### 避坑提示

- 模块系统是**可选的**，传统 JAR（非模块化）作为"自动模块"仍可使用。
- `opens` 用于测试框架（JUnit、Mockito）和 ORM（Hibernate）——它们需要反射访问私有字段。
- Java 9 的 JPMS 本身不是面试重点，但理解 `exports` vs `opens` 的区别能体现对模块化设计的深度理解。
- 钻石操作符在 Java 9 的改进相对次要，不要花太多时间。

---

## 12. Java 10 新特性：局部变量类型推断 var、CopyOnWriteList 改进

### 题目

Java 10 引入的 `var` 关键字是什么？有哪些使用限制？`CopyOnWriteArrayList` 有哪些改进？

### 核心答案

**`var` 类型推断**：

```java
// 编译器根据右侧表达式推断变量类型
var message = "Hello";                  // String
var numbers = new ArrayList<Integer>();  // ArrayList<Integer>
var stream = list.stream();              // Stream<T>
var map = Map.of("a", 1, "b", 2);        // Map<String, Integer>

// ❌ 错误用法
var x;                                   // 必须初始化
var lambda = (a, b) -> a + b;            // ❌ 匿名函数需要明确类型
var[] array = new int[3];                // ❌ 数组初始化器不能推断
var m = null;                            // ❌ null 无法推断
```

**`var` 使用限制**：
1. 只能用于**局部变量**（方法内、代码块内、for 循环内）。
2. **必须初始化**。
3. **不能用于字段**（类成员变量）。
4. **不能用于方法参数**和**返回类型**。
5. **不能用于 lambda 参数**（需要显式类型以推导目标类型）。
6. `var` 不是关键字，是**保留类型名**（类似 `int`），但可以用作变量名。

**`CopyOnWriteArrayList` 改进（Java 10）**：

Java 10 将 `CopyOnWriteArrayList` 的内部实现从 `ReentrantLock` 改为**`synchronized` 块**实现，减少了锁开销（`synchronized` 在 JVM 层有更多优化）。

```java
// Java 9+: CopyOnWriteArraySet 底层也是 CopyOnWriteArrayList
Set<String> set = new CopyOnWriteArraySet<>();
// 底层使用 COWSubSet（COWIterator），读不加锁，写复制
```

### 追问方向

- `var` 和 `Object` 的区别？→ `var` 是**编译期推断**，字节码中类型信息不变；`Object` 是**运行时多态`，丧失具体类型信息。
- `var` 用于 lambda 表达式会怎样？→ 编译错误，因为 lambda 需要目标类型（Target Type）才能推断，var 不能提供。
- `CopyOnWriteArrayList` 适用场景？→ 读多写少（遍历操作远多于修改操作）的并发场景，如监听器列表。

### 避坑提示

- `var` 不要滥用——在类型信息重要时（如返回值的类型），显式声明更清晰。
- `var` 用于很长的泛型链时会让代码更难读，这时显式写更好：`Map<String, List<Map<K, V>>>`  vs `var`。
- `CopyOnWriteArrayList` 的写操作（add/set/remove）开销巨大（复制整个底层数组），**不要用于写多的场景**。

---

## 13. Java 11 新特性：ZGC、HTTP Client API、字符串增强

### 题目

Java 11 的 ZGC 的特点是什么？新的 HTTP Client API 是什么？字符串有哪些增强方法？

### 核心答案

**ZGC（Z Garbage Collector）**：

- **目标**：停顿时间不超过 10ms，且停顿时间不随堆大小增加而增加（可扩展到 TB 级堆）。
- **核心原理**：染色指针（Colored Pointers）+ 读屏障（Load Barrier），并发标记和压缩。
- **启用**：`-XX:+UseZGC`（Java 15 前需要 `-XX:+UnlockExperimentalVMOptions`）。
- Java 11 为 experimental，Java 15 正式，Java 21 成为默认。

```bash
java -XX:+UseZGC -Xmx16g -jar myapp.jar
```

**HTTP Client API（Java 11 正式版）**：

```java
// Java 9 Incubator → Java 11 正式
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("Accept", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
    .build();

// 同步
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// 异步
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

**字符串增强（Java 11）**：

```java
// isBlank() — 是否空白（空格/空白字符）
"  ".isBlank(); // true（Java 11）

// lines() — 按行分割，返回 Stream<String>
"hello\nworld".lines().collect(Collectors.toList()); // ["hello", "world"]

// repeat(n) — 重复字符串 n 次
"ab".repeat(3); // "ababab"

// strip() / stripLeading() / stripTrailing()
// strip() 去除前后 Unicode 空白（比 trim() 更全面）
// trim() 只去除 ASCII 32 以下字符
"  hello  ".strip();    // "hello"（支持 Unicode）
"  hello  ".trim();     // "hello"（同 strip 对纯 ASCII 等效）
"\u2000hello\u2000".strip(); // "hello"（trim 不处理这类 Unicode 空白）
```

### 追问方向

- ZGC vs G1 的区别？→ G1 停顿时间随堆增大而增大，ZGC 声称 <10ms 且不随堆增大；G1 是分代回收，ZGC 是单代（non-generational，Java 21 后支持 generational ZGC）。
- `HttpClient` 默认使用 HTTP/2，自动协商 HTTP/1.1。
- `strip()` vs `trim()` 实际区别？→ `trim` 只去除 ASCII 控制字符（码点 ≤ 32），`strip` 去除 Unicode "White_Space" 类别中的所有字符（包括 non-breaking space `\u00A0` 等）。

### 避坑提示

- Java 11 的 ZGC 在 Java 15 前是 experimental，需要 unlock。
- `HttpClient` 在 Java 11 是正式 API，但默认 HTTP/2，注意服务端兼容性。
- `String.lines()` 不保留空行，`String.split("\n")` 行为略有不同（split 会忽略尾部空字符串）。

---

## 14. Java 14/15/16 新特性：Record、Sealed Classes、Pattern Matching for instanceof

### 题目

Java 14 引入的 Record 是什么？Sealed Classes 在哪个版本成为正式版？Pattern Matching for instanceof 的演进过程是什么？

### 核心答案

**Record（Java 14-preview, 16-final）**：

```java
// Java 16+ 正式版
public record Point(int x, int y) {
    // 自动生成：
    // - private final 字段 x, y
    // - 构造函数 Point(int x, int y)
    // - getter: x(), y()
    // - equals() / hashCode() / toString()

    // 可以添加自定义构造函数（必须是完整构造函数或链到主构造函数）
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException();
    }

    // 可以添加额外方法
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }

    // 不能添加实例字段（会破坏 equals 契约）
}

// 使用
Point p = new Point(3, 4);
p.x(); // 3
p.y(); // 4
```

**Sealed Classes（Java 15-preview, 17-final）**：

```java
// Java 17 正式版
public sealed class Shape permits Circle, Rectangle, Square { }

// 每个子类必须声明如何继承：final / sealed / non-sealed
public final class Circle extends Shape { }
public sealed class Rectangle extends Shape permits TransparentRect { }
public non-sealed class Square extends Shape { }

// permits 列表必须在同一模块或同一个包内
// 编译器可以穷尽检查（exhaustive switch）
```

**Pattern Matching for instanceof（Java 14-preview, 16-final）**：

```java
// Java 16 正式版
if (obj instanceof String s) {
    // s 在此作用域内已强制转型为 String
    System.out.println(s.length()); // 直接用 s
} else {
    // s 不在此作用域
}

// 配合 switch（Java 21 正式）
switch (obj) {
    case String s when s.length() > 5 -> System.out.println("Long: " + s);
    case String s -> System.out.println("Short: " + s);
    case Integer i -> System.out.println("Int: " + i);
    default -> System.out.println("Unknown");
}
```

### 追问方向

- Record 能否实现接口？→ 可以，`implements Serializable`。
- Record 可以是泛型吗？→ 可以，`record Pair<K, V>(K key, V value) { }`。
- Sealed Classes 的子类必须全部列出吗？→ 必须，且在同一模块/包内。
- Pattern Matching 能用在 `switch` 表达式中吗？→ Java 21 `switch` 的 pattern matching 是正式版，Java 17 只支持 `instanceof`。

### 避坑提示

- Record 是**浅层不可变**的（字段 `final`），但如果字段是引用类型，内部可变对象（如 `List`）仍可变——需要自定义构造函数防御性拷贝。
- Record 不能继承类（隐式继承 `java.lang.Record`），也不能被继承（默认 `final`）。
- Sealed Classes 的 permits 列表必须在**直接子类**层面完整列举，间接子类不受限制（除非其父类也是 sealed）。
- Pattern Matching for instanceof 在 Java 16 之前是 preview，语义可能微调。

---

## 15. Java 17 新特性：Sealed Classes 正式版、Switch 表达式增强

### 题目

Java 17 将哪些特性定为正式版？Sealed Classes 为什么重要？Switch 表达式有哪些改进？

### 核心答案

**Java 17 LTS 新特性（正式版）**：

| 特性 | 引入版本 | Java 17 状态 |
|---|---|---|
| Sealed Classes | 15 (preview) | **17 正式版** |
| Pattern Matching for instanceof | 16 (preview) | 21 正式版（17 仍是 preview） |
| Records | 14 (preview) | **16 正式版** |
| Text Blocks | 13 (preview) | **15 正式版** |
| Switch Expressions | 12 (preview) | **14 正式版** |
| Foreign Function & Memory API | 14 (Incubator) | 21 正式版 |
| `jpackage` | 14 | **17 正式版** |
| 伪随机数生成器改进 | 17 | **17 正式版** |

**Sealed Classes 为什么重要**：

1. **穷尽性检查**：编译器确保 `switch/instanceof` 覆盖所有可能类型，添加新子类不修改现有 switch 代码会收到编译错误。
2. **设计意图表达**：显式声明"此类只能被这些类扩展"，替代 `final` 的"禁止扩展"和 `non-sealed` 的"随意扩展"的二元对立。
3. **模块化 API 设计**：API 作者可以精确控制哪些类可以扩展。

```java
sealed interface Expr permits AddExpr, MulExpr, LiteralExpr { }

sealed class AddExpr implements Expr { /* ... */ }
sealed class MulExpr implements Expr { /* ... */ }
non-sealed class LiteralExpr implements Expr { /* 任意扩展 */ }

// 编译器穷尽检查
String eval(Expr e) {
    return switch (e) {
        case AddExpr a -> eval(a.left()) + " + " + eval(a.right());
        case MulExpr m -> eval(m.left()) + " * " + eval(m.right());
        case LiteralExpr l -> l.value().toString();
    }; // 编译器保证覆盖所有 permits
}
```

**Switch 表达式改进（Java 14 正式）**：

```java
// Java 12-13: 箭头语法 + yield
int result = switch (day) {
    case MONDAY, FRIDAY -> 6;
    case TUESDAY -> 7;
    default -> {
        int len = day.length();
        yield len; // Java 14: 用 yield 返回值
    }
};

// Java 14+: switch 是表达式，可以有返回值
DayType type = switch (day) {
    case SATURDAY, SUNDAY -> DayType.WEEKEND;
    default -> DayType.WEEKDAY;
};
```

### 追问方向

- Java 17 是 LTS 吗？→ 是，Oracle JDK 提供 8 年支持到 2029。
- Record vs Sealed Classes 可以组合吗？→ 可以，`sealed record SPoint(int x, int y) permits ... {}`。
- `jpackage` 做什么？→ 将 Java 应用打包成平台原生安装包（MSI、EXE、Dmg、Deb、RPM）。

### 避坑提示

- Java 17 的 Switch Pattern Matching 仍是 **preview**（`--enable-preview`），不要在生产环境不带此 flag 使用。
- Sealed Classes 的 permits 列表**不能有空**——至少要有一个子类。
- Switch 表达式中的 `yield` 和 `return` 不会混淆吗？→ `yield` 是**专用于 switch 表达式**的返回值语句，`return` 是方法返回值，两者不会混淆。

---

## 16. 字符串底层变化：String 实现从 char[] 到 byte[]，Compact String

### 题目

Java 9 为什么将 String 的内部实现从 `char[]` 改为 `byte[]`？Compact String 是什么？它带来了哪些性能和内存收益？

### 核心答案

**变化背景**：

- Java 8：`private final char[] value;` — 每个字符占 2 字节（UTF-16）。
- Java 9+：`private final byte[] value;` + `private final byte coder;`（coder = 0 表示 LATIN1，coder = 1 表示 ISO-8859-1/UTF-16）。

**Compact String（JEP 254，Java 9）**：

- **动机**：大多数字符串在 Java 应用中只包含 Latin-1 字符（ASCII/欧洲语言），只需要 1 字节/字符，但 UTF-16 总是用 2 字节，造成 50% 的空间浪费。
- **解决方案**：当字符串只包含 Latin-1 字符时，使用 `byte[]`（1 字节/字符）存储；否则使用 UTF-16 `byte[]`（2 字节/字符，隐含在coder标记中）。

```java
// 内部表示（Java 9+）
public final class String {
    private final byte[] value;      // 存储字符数据
    private final byte coder;        // 0=LATIN1, 1=UTF16
    private final int hash;          // 缓存哈希码

    // StringLatin1 (coder=0): 单字节字符
    // StringUTF16 (coder=1): 双字节字符（UTF-16 BE）
}
```

**内存收益**：Latin-1 字符串内存占用减少 50%。对大量字符串存储的应用（如数据库、缓存）效果显著。

**性能影响**：
- 改进的空间局部性（cache line 友好）。
- `charAt()`、`substring()` 等方法需要检查 coder。
- 大多数场景性能提升或持平，部分场景轻微下降（JNI 调用成本）。

### 追问方向

- `String` 的 `charAt` 在 Java 9+ 怎么做？→ 有 `StringLatin1::charAt` 和 `StringUTF16::charAt` 两个内部类方法，根据 coder 分发。
- 字符串拼接 `"a" + "b"` 会产生新的 byte[] 吗？→ 是，编译期优化（StringConcatFactory）和运行时优化（MH+CallSite）会尽量避免中间字符串，但逻辑上仍是新字符串。
- String 的 hashCode 缓存？→ `private int hash`（默认 0），首次 `hashCode()` 调用时计算并存缓存，**线程安全**吗？→ 不是，但 String 是不可变的，且 hash 是 volatile（JDK 8u40+加了volatile），多线程安全。

### 避坑提示

- **不要依赖字符串内部结构**（如反射 `value` 字段）——Java 9+ 是 `byte[]` 而不是 `char[]`，旧代码反射可能出问题。
- String 的 `getBytes(StandardCharsets.UTF_8)` 和 `String(byte[], charset)` 转换是外部操作，与内部 `coder` 无关。
- Compact String 对 `String.intern()` 行为无影响。

---

## 17. GC 变化：G1 成为默认、ZGC/Shenandoah 低延迟 GC、Epsilon

### 题目

Java 9 为什么将默认 GC 改为 G1？ZGC 和 Shenandoah 的设计目标是什么？Epsilon GC 是什么，使用场景？

### 核心答案

**G1（Garbage-First）成为默认（Java 9）**：

- Java 9 之前默认 GC 是 **Parallel GC**（throughput优先，停顿时间长）。
- Java 9 将默认 GC 改为 **G1**（停顿时间可控，适合大堆）。
- G1 是**分代收集器**（young + old），但逻辑上是**区域化（Region）**的，将堆分成多个等大小的 Region，优先回收垃圾最多的区域（Garbage-First）。

```bash
# 查看当前 GC
java -XX:+PrintCommandLineFlags -version
# 输出包含 -XX:+UseG1GC
```

**ZGC（JDK 11 experimental, 15 正式）**：

- **目标**：停顿时间 < 10ms，停顿时间不随堆大小增加而增加，支持 TB 级堆。
- **核心**：染色指针（Colored Pointers，GC 标记在指针上）+ 读屏障（Load Barrier）+ 并发操作。
- **触发条件**：不是基于分代，是基于**吞吐量和停顿时间**的自适应。
- **不适合**：ZGC 不执行压缩（compact），长时间运行后可能内存碎片化（但有 Scoped Memory API 缓解）。

```bash
# 启用
java -XX:+UseZGC -Xmx16g myapp.jar
```

**Shenandoah（JDK 12 experimental, 15 正式）**：

- 与 ZGC 类似，**停顿时间与堆大小无关**，但采用不同的实现（ Brooks Pointer / 转发指针）。
- 支持**并发压缩**。
- Red Hat 主导，适合 OpenJDK 构建。

```bash
java -XX:+UseShenandoahGC -Xmx16g myapp.jar
```

**Epsilon GC（Java 11）**：

- **"无操作" GC**：分配内存，但不回收任何垃圾（`-XX:+UseEpsilonGC`）。
- **用途**：
  - 短生命周期程序（进程退出前内存就用完，不需要回收）。
  - 性能测试基准（消除 GC 干扰）。
  - 内存压力测试（暴露内存泄漏）。
- **特点**：停顿时间极短（几乎为0），但内存用尽会OOM。

### 追问方向

- G1 的 `-XX:MaxGCPauseMillis` 参数是什么？→ G1 的目标停顿时间（默认 200ms），G1 会调整 region 大小和收集策略来尝试满足，但不保证。
- ZGC vs Shenandoah 的区别？→ ZGC 用染色指针（在上个世纪末是实验性的，如今成熟），Shenandoah 用 Brooks 转发指针；ZGC 只在 Solaris/Linux 支持（且要求 OS 支持多映射），Shenandoah 不依赖 OS；两者都支持并发压缩。
- CMS 收集器？→ 已废弃（Java 9+），被 G1 替代。

### 避坑提示

- ZGC 在 Java 11-14 需要 `--enable-preview` 或 experimental flag，Java 15+ 是正式功能。
- **不要在生产环境使用 Epsilon GC**，除非明确知道程序会在 OOM 前退出。
- G1 的 Full GC 仍可能 STW（Stop-The-World），大量对象分配时可能出现，此时 G1 会退化为串行收集。
- 选择 GC 不是越新越好——根据应用特点（吞吐量 vs 延迟 vs 堆大小）选择。

---

## 18. 枚举类优化：内部类访问枚举值、枚举单例 vs 懒加载

### 题目

枚举类在 Java 8-17 中有哪些内部访问优化？枚举单例模式相比懒加载（双重检查锁）和饿汉式有什么优势？

### 核心答案

**枚举类内部访问枚举值（JVM 层面）**：

```java
public enum Status {
    RUNNING,    // JVM: 编译器生成 public static final Status RUNNING = new Status()
    WAITING,
    TERMINATED;

    // 枚举类的构造函数（隐式 private）
    Status() { }

    // 枚举方法
    public boolean isActive() {
        return this == RUNNING;
    }
}
```

JVM 访问枚举值时，**不需要运行时查找**，编译器将枚举值编译为**静态字段初始化**，在类加载阶段即完成初始化。访问 `Status.RUNNING` 等价于访问 `public static final` 字段，**无锁且线程安全**（JLS 保证 Class 初始化线程安全）。

**枚举单例 vs 其他单例模式**：

```java
// 枚举单例（推荐，最简洁）
public enum Singleton {
    INSTANCE;

    public void doSomething() { }
}

// 懒加载 + 双重检查锁（线程安全）
public class SingletonLazy {
    private static volatile SingletonLazy instance;
    private SingletonLazy() { }
    public static SingletonLazy getInstance() {
        if (instance == null) {                 // 第一次检查
            synchronized (SingletonLazy.class) {
                if (instance == null) {          // 第二次检查
                    instance = new SingletonLazy();
                }
            }
        }
        return instance;
    }
}

// 饿汉式（类加载时即创建，可能浪费）
public class SingletonHungry {
    private static final SingletonHungry INSTANCE = new SingletonHungry();
    private SingletonHungry() { }
    public static SingletonHungry getInstance() { return INSTANCE; }
}
```

**枚举单例优势**：

| 特性 | 枚举单例 | 双重检查锁 | 饿汉式 |
|---|---|---|---|
| 线程安全 | JLS 保障，绝对 | volatile + synchronized | 类加载保障 |
| 延迟加载 | 否（类加载即创建） | 是 | 否 |
| 防反射攻击 | 构造函数抛异常 | 可防 | 可防 |
| 防反序列化 | 枚举天然防（JVM保证） | 需自定义 `readResolve` | 需自定义 `readResolve` |
| 代码简洁度 | ⭐⭐⭐ 最佳 | ⭐⭐ | ⭐⭐ |
| 能否实现枚举接口 | 是 | 否 | 否 |

```java
// 反射攻击防御（枚举单例）
// Constructor.newInstance() 在创建枚举时抛异常
// Hotspot 在 newInstance() 检测到 enum 类型则抛出 IllegalArgumentException
```

### 追问方向

- 枚举类可以有抽象方法吗？→ 可以，每个枚举值必须实现抽象方法（类似于匿名内部类实例）。
- 枚举的 `values()` 方法哪来的？→ 编译器在编译时自动生成 `public static T[] values()` 方法。
- 枚举可以实现接口吗？→ 可以，`enum Status implements Runnable { RUNNING { public void run() { } } }`。

### 避坑提示

- 枚举单例**不支持延迟初始化**，如果单例创建代价高且不一定被使用，用双重检查锁更合适。
- **枚举值数量固定**，运行时不能动态增减（这是枚举的本质限制）。
- 枚举的 `valueOf(String)` 方法是大小写敏感的，如果需要大小写不敏感的查找，需要自定义方法。
- 枚举实现接口时，每个枚举值都需要实现方法体，不能共享默认实现（除非用 `default` 方法在接口中提供）。

---

## 19. 重复注解与类型注解：@Repeatable、TYPE_PARAMETER 注解位置

### 题目

Java 8 引入的 `@Repeatable` 注解解决了什么问题？如何正确声明？类型注解（TYPE_USE）是什么，它可以放在哪些位置？

### 核心答案

**@Repeatable 重复注解**：

```java
// 1. 定义容器注解（必须有一个 value() 方法，类型是被容纳注解的数组）
@Retention(RetentionPolicy.RUNTIME)
@interface Schedules {
    Schedule[] value();
}

// 2. 定义可重复注解（加上 @Repeatable）
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Schedules.class)  // 指向容器注解
public @interface Schedule {
    String cron();
}

// 3. 使用：直接多次写，编译器自动放入容器
@Schedule(cron = "0 0 9 * * ?")
@Schedule(cron = "0 0 13 * * ?")
public class MyService { }

// 4. 获取所有注解（两种方式等价）
Schedules container = MyService.class.getAnnotation(Schedules.class);
Schedule[] schedules = MyService.class.getAnnotationsByType(Schedule.class);
```

**类型注解（TYPE_USE，Java 8）**：

```java
import java.lang.annotation.*;

// TYPE_USE 可标注任何类型出现的位置
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE_USE)
@interface NonNull { }

// 使用示例
@NonNull String name;                    // 字段
List<@NonNull String> list;              // 泛型参数
String @NonNull [] array;               // 数组类型
void process(@NonNull String input) { }  // 参数
String returnType() @NonNull {          // 返回类型（Java 8 不支持，放在前面）
    return "data";
}
Map.@NonNull Entry<String, Integer> e;  // 泛型类型参数的位置

// TYPE_PARAMETER：标注类型参数声明（在声明处，非使用处）
@Target(ElementType.TYPE_PARAMETER)
@interface Generic { }

class Container<@Generic T> { }         // 泛型参数声明处
```

**TYPE_USE vs TYPE_PARAMETER**：

| 注解位置 | ElementType | 示例 |
|---|---|---|
| 泛型参数**声明**处 | `TYPE_PARAMETER` | `<@NonNull T>` |
| 泛型参数**使用**处 | `TYPE_USE` | `List<@NonNull String>` |
| 字段类型 | `TYPE_USE` | `@NonNull String s;` |
| 返回值类型 | `TYPE_USE` | `@NonNull String get()` |
| 参数类型 | `TYPE_USE` | `void m(@NonNull String)` |

### 追问方向

- `@Repeatable` 的容器注解为什么必须叫 `value()`？→ JLS 规定，容器的注解属性必须叫 `value` 且类型是被容纳注解的数组。
- 注解继承吗？→ 默认**不继承**（除非注解声明 `@Inherited` 且在类上）。
- Java 8 的类型注解只是**声明式**，需要配合 Checker Framework 等工具才能真正做类型检查。

### 避坑提示

- `@Repeatable` 的**反射获取方式**：`getAnnotation(Schedules.class)` 直接拿到容器，`getAnnotationsByType(Schedule.class)` 拿到展开后的注解数组。
- **TYPE_USE 和 TYPE_PARAMETER 容易混淆**——TYPE_PARAMETER 在泛型**声明**处，TYPE_USE 在泛型**使用**处和所有其他类型位置。
- Java 8 注解机制支持了 Checker Framework、Error Prone 等工具，但这些工具是**编译时增强**，运行时注解本身不做检查。

---

## 20. Java 8~17 整体演进路线：每个版本的 Breaking Changes

### 题目

从 Java 8 到 Java 17，每个 LTS 版本（9, 11, 17）带来了哪些 Breaking Changes？升级时需要关注什么？

### 核心答案

**Java 8 → Java 9 Breaking Changes**：

| 变化 | 影响 |
|---|---|
| **模块系统（JPMS）** | `java.base` 不再默认导出所有内部 `sun.*` 包，反射调用会抛 `IllegalAccessError` |
| 移除 `endorsed` 和 `ext` 目录 | 旧的扩展类加载机制移除 |
| `java.net.URLEncoder` / `URLDecoder` 默认编码 | 从 `定型` 改为 `UTF-8`（长期应该是改进，但与旧行为不同） |
| `java.se` 平台模块不自动打开 | 需要 `--add-opens` 启动参数 |
| 移除 `Thread.stop()`、`SecurityManager` 废弃 | 旧代码中使用的 Thread.stop() 直接报错 |

**Java 9 内部 Breaking Changes**：
- String 从 `char[]` 改为 `byte[]`（内部变化，API 兼容，但反射访问 `value` 字段会出问题）。

**Java 11 Breaking Changes**：

| 变化 | 影响 |
|---|---|
| **移除 Java EE 模块**（`java.xml.ws`, `java.corba`, `java.transaction`, `java.activation`, `java.xml.bind` 全部移除） | JAXB 不再可用，需要 Maven 依赖 |
| **移除 JavaFX**（从 JDK 中移除，成为独立模块） | 需要单独安装 `openjfx` |
| **移除 `java --version` 输出版本中的 `-ea` 后缀**（小变化） | |
| **Nashorn JavaScript 引擎移除**（Java 15 彻底移除） | Java 11 标记废弃，15 移除 |
| **HTTP Client API** 在 Java 11 前是 Incubator | 使用 `java.net.http` 包 |

**Java 11 之前支持**: 长期支持的客户端应使用 Java 11。

**Java 17 Breaking Changes（最严格）**：

| 变化 | 影响 |
|---|---|
| **移除 Applet API** | Web 浏览器插件彻底废弃 |
| **移除 AOT 编译（jaotc）** | 实验性 AOT 移除 |
| **移除 `java.security.acl` 包** | 已废弃多年，终于移除 |
| **严格浮点运算（`-strictfp`）默认开启** | 极少数代码受影响 |
| **移除 `InputService` 和 `AbstractInputService`** | 极少使用 |
| **Shenandoah 从 experimental 升级为正式** | 但仍需确认 GC 参数 |
| **启用 Strong Encapsulation** | `--add-opens` 成为唯一标准访问方式 |
| **ZGC 默认化**（Java 15+） | 并发 GC 默认行为变化 |
| **Nashorn 彻底移除**（Java 15 彻底） | 如果依赖 JS 评估引擎需要替换 |

**各版本关键非 LTS 新特性时间线**：

```
Java 8  (2014)  ← LTS  — Lambda, Stream, Optional, LocalDateTime, CompletableFuture, 接口 default/static, 重复注解, 类型注解, Nashorn
Java 9  (2017)           — 模块系统(JPMS), 接口私有方法, 集合工厂方法(List.of/Set.of/Map.of), Stream 新方法(takeWhile/dropWhile/ofNullable), Optional.stream()
Java 10 (2018)           — var 类型推断, CopyOnWriteArrayList 优化, G1 成为默认
Java 11 (2018)  ← LTS  — ZGC(experimental→15正式), HTTP Client(正式), 字符串增强(isBlank/lines/repeat/strip), 移除 Java EE 模块, 移除 JavaFX
Java 12 (2019)           — Switch 表达式预览, String 新方法(indent/transform)
Java 13 (2019)           — Text Blocks 预览(->15正式), Switch 表达式(->14正式)
Java 14 (2020)           — Pattern Matching for instanceof 预览, Records 预览, ZGC improvements
Java 15 (2020)           — Records 预览, Sealed Classes 预览, Text Blocks 正式, ZGC/Shenandoah 正式
Java 16 (2021)           — Records 正式版, Pattern Matching for instanceof 正式版, Stream.toList()
Java 17 (2021)  ← LTS  — Sealed Classes 正式版, Switch 表达式增强, 移除 Applet, ZGC 默认, RandomGenerator
```

### 追问方向

- `Thread.stop()` 为什么不安全？→ 会释放在任何代码位置持有的锁，导致对象状态不一致（数据损坏），其他线程可能看到损坏的对象。
- `SecurityManager` 什么时候彻底移除？→ Java 17 废弃，Java 21 移除。
- `java --version` 输出格式变化？→ Java 9 开始，`java -version` 输出变为多行格式，带 `(build 1.8.0_xxx)` 的格式变为 `version "11.0.1"` 格式。

### 避坑提示

- 升级 JDK 最重要的第一步：**在 CI 中跑所有测试**，确认所有单元测试通过是 Baseline。
- 每次升级优先关注**直接依赖的第三方库**是否支持目标 Java 版本（如 JAXB 在 Java 11 移除）。
- **不要同时升级 JDK 版本和应用代码**——分两步：先升级 JDK 跑旧代码，再逐步迁移新特性。
- Java 17 的强封装（encapsulation）最严格，很多通过反射访问内部 API 的库（如部分 ORM、测试框架旧版本）会报错，需要 `--add-opens`。

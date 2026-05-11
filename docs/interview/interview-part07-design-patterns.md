# Java 设计模式面试题（上）

> 本篇涵盖 20 道高频设计模式面试题，适合 1-3 年经验工程师准备面试。全程口语化追问风格，带你模拟真实面试场景。

---

## 第 1 题：七大设计原则

### 题目

> Java 有七大设计原则，你能简单说一下吗？能结合实际例子讲讲开闭原则和里氏替换原则吗？

### 核心答案

七大设计原则简称"SOLID+DACCP"，逐个简述：

| 原则 | 全称 | 一句话概括 |
|------|------|-----------|
| **开闭原则（OCP）** | Open-Closed Principle | **对扩展开放，对修改关闭**。新功能通过扩展实现，不改动已有代码 |
| **里氏替换原则（LSP）** | Liskov Substitution Principle | **子类可以替换父类，且程序行为不变**。父类出现的地方换成子类，结果一致 |
| **依赖倒置原则（DIP）** | Dependency Inversion Principle | **高层模块不依赖低层模块，两者都依赖抽象**。面向接口编程 |
| **接口隔离原则（ISP）** | Interface Segregation Principle | **接口要小而专，不要大而全**。一个类不应该强迫实现它不需要的方法 |
| **单一职责原则（SRP）** | Single Responsibility Principle | **一个类只做一件事**，只为一个变化原因负责 |
| **迪米特法则（LoD）** | Law of Demeter | **只与直接朋友通信**，不要和陌生人说话（不认识的人） |
| **合成复用原则（CRP）** | Composite Reuse Principle | **优先使用组合/聚合，而不是继承**，来复用代码 |

**开闭原则举例：**

```java
// 违背 OCP：每加一种形状就要改 shape() 方法
void printShape(String type) {
    if ("circle".equals(type)) { /* 画圆 */ }
    if ("square".equals(type)) { /* 画方 */ }
    // 新增 rectangle → 修改这里
}

// 符合 OCP：新增形状只需扩展，不用改已有代码
interface Shape { void draw(); }
class Circle implements Shape { @Override public void draw() { /* 画圆 */ } }
class Square implements Shape { @Override public void draw() { /* 画方 */ } }
// 新增 Rectangle 只需写新类，不动现有代码
```

**里氏替换原则举例：**

```java
// 违背 LSP：子类 sqrt() 的前置条件比父类更严格
class Parent {
    void process(int value) { /* value 只要 > 0 就行 */ }
}
class Child extends Parent {
    @Override
    void process(int value) {
        if (value > 10) throw new IllegalArgumentException(); // 前置条件更严格！
        // 调用方用 Parent 引用传入 5 是合法的，但实际会报错
    }
}

// 符合 LSP：正方形是矩形子类，但"长宽相等"导致面积计算逻辑变化
// 正确的建模：正方形不应该继承矩形，或重新审视继承关系
```

### 追问方向

- "合成复用和里氏替换都是关于复用的，它们有什么区别？" → 组合复用是"has-a"，里氏替换是"is-a"
- "迪米特法则和外观模式有什么关系？" → 外观模式就是迪米特法则的典型实现
- "接口隔离和单一职责有什么区别？" → 单一职责针对类，接口隔离针对接口；接口隔离是单一职责在接口层面的延伸
- "你项目中哪个模块体现了依赖倒置？" → Spring DI 容器就是 DIP 的最佳实践

### 避坑提示

- 不能只背名字，要能举出**项目中的实际例子**；纯背书容易被追问戳穿
- 里氏替换容易被误解为"子类比父类功能更多"，实际上是**替换后行为一致**，不能更严格
- 开闭原则不是说绝对不能改，而是**对扩展开放，对修改封闭**——核心是找到变化的抽象并封装

---

## 第 2 题：单例模式

### 题目

> 单例模式有哪几种写法？线程安全吗？能不能防止反射和反序列化攻击？

### 核心答案

**五种写法：**

```java
// 1. 饿汉式（类加载时实例化）
class HungrySingleton {
    private static final HungrySingleton INSTANCE = new HungrySingleton();
    private HungrySingleton() {}
    public static HungrySingleton getInstance() { return INSTANCE; }
}

// 2. 懒汉式（第一次使用时才创建）
class LazySingleton {
    private static LazySingleton instance;
    private LazySingleton() {}
    public static synchronized LazySingleton getInstance() { // 同步方法，线程安全但开销大
        if (instance == null) instance = new LazySingleton();
        return instance;
    }
}

// 3. 双重检查锁（DCL）
class DCLSingleton {
    private static volatile DCLSingleton instance; // volatile 必须有！
    private DCLSingleton() {}
    public static DCLSingleton getInstance() {
        if (instance == null) {                    // 第一次检查
            synchronized (DCLSingleton.class) {
                if (instance == null)              // 第二次检查
                    instance = new DCLSingleton(); // 指令重排问题需要 volatile
            }
        }
        return instance;
    }
}

// 4. 静态内部类
class StaticInnerSingleton {
    private StaticInnerSingleton() {}
    private static class Holder {
        private static final StaticInnerSingleton INSTANCE = new StaticInnerSingleton();
    }
    public static StaticInnerSingleton getInstance() {
        return Holder.INSTANCE; // 加载 InnerHolder 时才创建，延迟加载 + 线程安全（JLS 保证）
    }
}

// 5. 枚举（最安全，Java 层面天然防反射/反序列化）
enum EnumSingleton {
    INSTANCE;
    public void doSomething() {}
}
```

**线程安全分析：**

| 写法 | 线程安全 | 懒加载 | 性能 |
|------|---------|--------|------|
| 饿汉式 | ✅ 安全（类加载时由 JVM 保证） | ❌ 否 | 最优 |
| 懒汉式（同步方法） | ✅ 安全 | ✅ 是 | 最差（每次 getInstance 都加锁） |
| DCL | ✅ 安全（volatile 禁止指令重排） | ✅ 是 | 优 |
| 静态内部类 | ✅ 安全（JLS 保证） | ✅ 是 | 最优 |
| 枚举 | ✅ 安全（Java 语言层面保证） | ✅ 是 | 最优 |

### 追问方向

- "为什么要双重检查，单层不行吗？" → 单层 synchronized 每次调用都要抢锁，DCL 大部分时候不加锁
- "volatile 在 DCL 里解决了什么问题？" → 防止 `instance = new DCLSingleton()` 的指令重排（分配内存→设置引用→执行构造），导致其他线程看到不完整的对象
- "枚举为什么能防反射？" → `newInstance()` 在枚举类上被 JDK 强制抛出 `IllegalAccessException`，无法通过反射构造
- "序列化怎么破坏单例？" → 反序列化会通过反射调用 `readResolve()` 生成新对象。枚举序列化由 JVM 保证，天然防护

### 避坑提示

- **DCL 必须加 volatile**，面试时忘写或解释不清是致命伤
- 枚举是《Effective Java》作者 Josh Bloch 推荐的**最优写法**，如果面试没提到，面试官可能暗示你知识面不够新
- 如果被追问"你项目中用的哪种"，大部分场景推荐**静态内部类**（延迟加载+无锁开销）或**枚举**（最简洁+绝对安全）

---

## 第 3 题：工厂模式

### 题目

> 工厂模式有哪三种？各自适用什么场景？和 Spring 的 `BeanFactory` 有什么关系？

### 核心答案

**三种工厂模式：**

```java
// 简单工厂：产品固定，按参数创建（不扩展，符合 OCP 程度低）
class SimpleFactory {
    public static Product create(String type) {
        return switch (type) {
            case "A" -> new ProductA();
            case "B" -> new ProductB();
            default -> throw new IllegalArgumentException();
        };
    }
}

// 工厂方法：每个产品对应一个工厂接口，扩展产品时不用改已有工厂
interface Factory { Product create(); }
class FactoryA implements Factory { @Override public Product create() { return new ProductA(); } }
class FactoryB implements Factory { @Override public Product create() { return new ProductB(); } }

// 抽象工厂：产品族概念，一个工厂生产一组相关产品（产品族级别扩展）
interface AbstractFactory {
    ProductX createX();
    ProductY createY();
}
```

| 模式 | 产品数量 | 扩展方式 | 违背 OCP 程度 |
|------|---------|---------|-------------|
| 简单工厂 | 单一产品线 | 新增产品**必须**改工厂代码 | 高 |
| 工厂方法 | 单一产品族 | 新增产品只需新增工厂类 | 低 |
| 抽象工厂 | 多产品族 | 新增产品族容易，新增产品困难 | 中 |

**Spring 的 `BeanFactory` 就是工厂模式的体现**：通过 `getBean()` 根据 name/id 获取对象实例。`ApplicationContext` 是其子接口，支持更丰富的初始化策略。

### 追问方向

- "简单工厂违背开闭原则，为什么还用？" → 产品种类少且稳定时，简单工厂代码最简洁，不必要过度设计
- "工厂方法和策略模式有什么区别？" → 工厂模式专注于**创建对象**，策略模式专注于**选择行为**；工厂返回的是不同实例，策略传入的是同一接口的不同实现
- "抽象工厂的优缺点？" → 优点：保证产品族内对象一致性；缺点：新增产品族中任意产品都要改抽象工厂接口，扩展产品族困难

### 避坑提示

- 工厂方法模式和抽象工厂容易混淆，记住：**工厂方法产一种，抽象工厂产一族**
- Spring 的 `FactoryBean` 接口是另一种工厂模式的应用（`Mybatis` 用它创建 `SqlSessionFactory`）

---

## 第 4 题：建造者模式

### 题目

> 建造者模式和构造器相比有什么优势？Lombok 的 `@Builder` 怎么实现的？你在项目中用过吗？

### 核心答案

**为什么需要建造者模式：**

```java
// 传统构造器：参数一多就爆炸，且难以阅读
new User("张三", "13800000000", "北京", " male ", 25, true, "vip");
// 参数顺序不能错，可选参数只能传 null 或重载多个构造器

// 建造者模式：可读性强，可选参数清晰
User user = User.builder()
    .name("张三")
    .phone("13800000000")
    .city("北京")
    .gender("male")
    .age(25)
    .vip(true)
    .level("vip")
    .build();
```

**手写建造者（标准写法）：**

```java
class User {
    private String name;
    private String phone;
    private String city;
    // 私有构造器
    private User(Builder b) {
        this.name = b.name;
        this.phone = b.phone;
        this.city = b.city;
    }
    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String name;
        private String phone;
        private String city;
        public Builder name(String n) { this.name = n; return this; }
        public Builder phone(String p) { this.phone = p; return this; }
        public Builder city(String c) { this.city = c; return this; }
        public User build() { return new User(this); }
    }
}
```

**Lombok `@Builder` 原理：** Lombok 在编译期通过注解处理器（Annotation Processor）自动生成 `Builder` 内部类、`builder()` 静态方法、以及用 `builder()` 创建对象的私有构造器。整个过程对源代码无侵入，字节码里才有 Builder 类。

**流式 API（Fluid API）：** 核心是 `return this`，让每个 setter 返回自身，实现链式调用。Lombok `@Builder` 和很多第三方库（Guava `ImmutableList`、OkHttpClient）都采用这种方式。

### 追问方向

- "建造者和工厂方法都能创建对象，区别在哪？" → 工厂方法一次性创建完整对象；建造者通过**一步一步设置参数**创建，**适合参数多或可选参数复杂**的场景
- "@Builder 默认生成的 Builder 能不能加校验？" → 可以用 `@Builder(toBuilder = true)` 和 `@BeforeBuilder` / 自定义方法做校验；也可用 `@Setter(onMethod_ = @Autowired)` 让 Lombok 生成的 setter 带有额外注解
- "如果对象字段间有依赖关系（比如字段 A > 0 时字段 B 才能设置），Builder 怎么保证？" → `build()` 方法里做校验，抛 `IllegalStateException`

### 避坑提示

- 建造者模式不是银弹：**参数少于 4-5 个时用构造器就够了**，不要过度工程化
- Lombok `@Builder` 生成的代码不是源码而是字节码，调试时要看 class 文件或反编译
- `@Builder.Default` 用于指定字段默认值，否则 builder 调用时没设置的字段会是零值

---

## 第 5 题：代理模式

### 题目

> 静态代理和动态代理有什么区别？JDK 动态代理和 CGLIB 底层实现是什么？Spring AOP 用的是哪种？

### 核心答案

**静态代理：编译时生成代理类，代码长这样：**

```java
// 接口
interface UserService { void save(); }
// 真实对象
class UserServiceImpl implements UserService {
    @Override public void save() { System.out.println("保存用户"); }
}
// 静态代理（手动写死）
class UserServiceProxy implements UserService {
    private UserService target;
    public UserServiceProxy(UserService target) { this.target = target; }
    @Override public void save() {
        System.out.println("before..."); // 前置增强
        target.save();
        System.out.println("after...");  // 后置增强
    }
}
```

**动态代理：运行时生成代理类，不需要手写代理类。**

```java
// JDK 动态代理（基于接口）
class JdkProxyFactory implements InvocationHandler {
    private Object target;
    public Object bind(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),  // 必须有接口
            this
        );
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before...");
        Object result = method.invoke(target, args);
        System.out.println("after...");
        return result;
    }
}

// CGLIB 动态代理（基于继承）
class CglibProxyFactory implements MethodInterceptor {
    private Object target;
    public Object getProxy(Class<?> cls) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(cls);        // 设置父类
        enhancer.setCallback(this);          // 回调
        return enhancer.create();
    }
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("before...");
        Object result = proxy.invokeSuper(obj, args); // 调用父类方法
        System.out.println("after...");
        return result;
    }
}
```

**Spring AOP 的选择策略：**

| 条件 | 代理方式 |
|------|---------|
| 目标类有接口 | **JDK 动态代理**（优先，性能更好） |
| 目标类无接口 | **CGLIB 代理**（继承方式） |
| Spring 5.x+ 强制要求 `spring.aop.proxy-target-class=true` 才用 CGLIB | |

Spring AOP 的完整流程：**切面定义 → 切点匹配 → 代理创建 → 方法调用时拦截执行通知**。

### 追问方向

- "JDK 动态代理为什么必须有接口？" → 它通过 `Proxy.newProxyInstance()` 生成一个**实现了一组接口**的代理类，`$Proxy0 implements UserService, OtherService`，所以接口是必需的
- "CGLIB 生成的代理类是什么样的？" → 生成一个继承自目标类的子类，覆盖父类方法，在方法前后加增强逻辑。**无法代理 `final`/`private` 方法**
- "Spring Boot 2.0 默认用 CGLIB，为什么？" → 为了统一行为，避免有接口却想用 CGLIB 的场景
- "MyBatis 的 `Mapper` 代理是怎么实现的？" → `MapperProxy` 是一个 `InvocationHandler`，用 JDK 动态代理生成 Mapper 代理对象，执行 `sqlSession.selectList()` 等方法时触发 invoke 调用 SQL

### 避坑提示

- 很多人误以为"Spring AOP 永远用 CGLIB"，其实是有接口优先用 JDK；这是高频误区
- CGLIB 通过继承实现，**被代理类不能是 `final`**，代理方法也不能是 `final`/`private`
- 如果被问到 AOP 原理，不能只说"动态代理"，要能讲出 **Advice（通知）、Pointcut（切点）、Advisor（通知器）、Proxy（代理）** 四个核心概念

---

## 第 6 题：装饰器模式

### 题目

> 装饰器模式和继承相比有什么优势？Java I/O 流里是怎么用装饰器模式的？

### 核心答案

**装饰器模式核心：动态地给对象附加额外职责，比继承更灵活。**

```java
// 基础饮料接口
interface Coffee { String getDesc(); int getPrice(); }

// 基础实现
class SimpleCoffee implements Coffee {
    @Override public String getDesc() { return "美式咖啡"; }
    @Override public int getPrice() { return 25; }
}

// 装饰器抽象类（关键：持有被装饰对象）
abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee; // 组合而非继承
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

// 具体装饰器：加牛奶
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    @Override public String getDesc() { return coffee.getDesc() + " + 牛奶"; }
    @Override public int getPrice() { return coffee.getPrice() + 8; }
}

// 具体装饰器：加糖
class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }
    @Override public String getDesc() { return coffee.getDesc() + " + 糖"; }
    @Override public int getPrice() { return coffee.getPrice() + 3; }
}

// 使用：牛奶+糖+美式（多层装饰）
Coffee coffee = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
```

**Java I/O 装饰器模式：**

```
InputStream (抽象组件)
  ├── FileInputStream (具体组件)
  ├── ByteArrayInputStream (具体组件)
  └── FilterInputStream (装饰器基类)
        ├── BufferedInputStream (给 FileInputStream 加缓冲)
        └── DataInputStream (加数据类型解析)
```

`BufferedInputStream` 包装 `FileInputStream`，`DataInputStream` 再包装 `BufferedInputStream`，形成**多层装饰链**，每层只专注一件事。

**装饰器 vs 继承：**

| 维度 | 继承（继承基类） | 装饰器模式 |
|------|---------------|-----------|
| 扩展方式 | 编译时静态 | 运行时动态 |
| 职责数量 | 每种组合都要写新类 | 自由组合，按需装饰 |
| 类爆炸 | O(n²) | O(n) |
| 运行时行为 | 固定 | 可叠加、可替换 |

### 追问方向

- "@Decorator 注解（Java EE / Jakarta EE）和装饰器模式有什么关系？" → 这是 CDI（Contexts and Dependency Injection）提供的**注解驱动的装饰器**，用 `@Decorator` 注解标记一个类，在 beans.xml 中注册，实现自动装饰器链装配
- "装饰器模式和代理模式的区别？" → 代理模式侧重**控制访问**（增强是附加的），装饰器模式侧重**动态增加职责**（职责是核心）；代理通常一对一，装饰器可以多层叠加
- "装饰器模式有什么缺点？" → 装饰器链长时调试困难（调用栈深），不如单继承类直观；增加了很多小类

### 避坑提示

- **装饰器模式和继承的选择**：如果扩展种类少且固定，用继承即可；种类多且动态组合，用装饰器
- Java I/O 里的 FilterInputStream 就是装饰器基类，这是 JDK 源码中装饰器模式的经典实现，面试能说出来是加分项

---

## 第 7 题：策略模式

### 题目

> 策略模式怎么用？和 if-else 相比好在哪？能用 Lambda 代替策略接口吗？

### 核心答案

**策略模式核心：定义算法族（策略），让它们可以互换。**

```java
// 策略接口
interface PaymentStrategy {
    void pay(double amount);
}

// 具体策略
class AlipayStrategy implements PaymentStrategy {
    @Override public void pay(double amount) { System.out.println("支付宝付 " + amount); }
}
class WechatPayStrategy implements PaymentStrategy {
    @Override public void pay(double amount) { System.out.println("微信付 " + amount); }
}

// Context（使用策略的类）
class Order {
    private PaymentStrategy strategy; // 可运行时替换
    public void setStrategy(PaymentStrategy strategy) { this.strategy = strategy; }
    public void pay(double amount) { strategy.pay(amount); }
}

// 使用
Order order = new Order();
order.setStrategy(new AlipayStrategy());
order.pay(100.0);
```

**策略模式 vs if-else：**

| 场景 | if-else | 策略模式 |
|------|---------|---------|
| 新增策略 | 修改原代码，违背 OCP | 新增策略类，不改已有代码 |
| 测试 | 难以单独测试每个分支 | 可以单独测试每个策略 |
| 可维护性 | 条件多时代码臃肿 | 清晰分离，职责明确 |
| 运行时切换 | ❌ 写死 | ✅ 可以运行时动态切换 |

**Lambda 代替策略接口（Java 8+）：**

```java
// 不用定义策略接口，直接用函数式接口
interface PaymentStrategy {
    void pay(double amount);
}

// 用 Lambda（策略接口只有一个方法时自动匹配）
Order order = new Order();
order.setStrategy(amount -> System.out.println("支付宝付 " + amount));
order.pay(100.0);

// 更进一步：用 enum 实现策略（内置策略集合）
enum PaymentType {
    ALIPAY(amount -> System.out.println("支付宝付 " + amount)),
    WECHAT(amount -> System.out.println("微信付 " + amount));

    private final PaymentStrategy strategy;
    PaymentType(PaymentStrategy strategy) { this.strategy = strategy; }
    public void pay(double amount) { strategy.pay(amount); }
}
```

### 追问方向

- "策略模式和工厂模式都能消除 if-else，有什么区别？" → 工厂模式决定**创建哪个对象**（创建者视角），策略模式决定**使用哪个行为**（使用者视角）
- "Spring 源码里哪些地方用到了策略模式？" → `Resource` 资源的 `getInputStream()` 实现、 Bean 的实例化策略（`instantiateClass()`）、 `@Transactional` 的传播行为
- "什么情况下不推荐用策略模式？" → 策略只有 1-2 种且非常稳定；策略算法极简单（如只有 `a + b`）；不需要运行时切换

### 避坑提示

- Lambda 替代策略接口是**Java 8+ 的标准实践**，如果还在手写匿名类，会被认为知识陈旧
- 策略模式如果策略数量持续增长（超过 10+），考虑用**注册机制**（Map<String, PaymentStrategy>）动态注册，避免 if-else 或 switch

---

## 第 8 题：模板方法模式

### 题目

> 模板方法模式是什么？钩子方法有什么用？Spring JdbcTemplate 怎么体现这个模式？

### 核心答案

**模板方法：在父类里定义算法骨架，把某些步骤延迟到子类。**

```java
abstract class AbstractRedisTemplate {
    // 模板方法（final，不允许子类覆盖）
    public final Object execute() {
        Object result = null;
        try {
            openConnection();
            result = doInConnection();   // 抽象方法，子类实现
            return result;
        } finally {
            closeConnection();
        }
    }

    // 通用方法（可被子类覆盖）
    protected void openConnection() { /* 默认实现 */ }

    // 抽象方法（必须子类实现）
    protected abstract Object doInConnection();

    // 钩子方法（默认空实现，子类可选覆盖）
    protected void onError(Exception e) { /* 默认不处理 */ }
}
```

**钩子方法（Hook Method）：** 用 `default` 空实现或可覆盖的 `protected` 方法，让子类**选择性**地干预模板流程，而不是被迫全部实现。例子：`afterPropertiesSet()`、`destroy()` 方法。

**Spring JdbcTemplate 源码体现：**

```java
// JdbcTemplate 核心方法
public <T> T execute(ConnectionCallback<T> action) {
    // 模板骨架：获取连接→执行 SQL→异常处理→关闭连接
    Connection con = getConnection();
    try {
        return action.doInConnection(con); // 回调函数 = 子类逻辑
    } catch (SQLException e) {
        throw translateException(e);  // 异常处理模板
    } finally {
        releaseConnection(con);
    }
}
```

`execute()` 是模板方法，`ConnectionCallback` 是回调接口（类似策略），`getConnection()` 和 `releaseConnection()` 是通用步骤，`action.doInConnection()` 是需要子类/调用方提供的"个性化步骤"。

### 追问方向

- "模板方法和回调（Callback）有什么区别？" → 模板方法是**继承**机制，子类通过覆盖方法定制；回调是**委托**机制，把逻辑作为参数传入。回调更灵活，不受继承层次限制
- "Spring 的 `RestTemplate` 也是模板方法吗？" → `RestTemplate` 的 `execute`/`doExecute` 方法是典型的模板方法模式：定义 HTTP 请求骨架，开放 `RequestCallback` 和 `ResponseExtractor` 两个回调让调用方定制
- "钩子方法和抽象方法怎么选？" → 父类有默认实现用钩子，没有默认行为就用抽象方法强迫子类实现

### 避坑提示

- 模板方法模式在 Spring 源码中**出现频率极高**（JdbcTemplate、RestTemplate、TransactionTemplate、JmsTemplate 等），面试必问
- 如果被要求手写模板方法，**记得用 `final` 修饰模板方法**，防止子类破坏骨架流程

---

## 第 9 题：观察者模式

### 题目

> 观察者模式有什么用？JDK 内置的 Observer/Observable 为什么不推荐用了？Spring 事件机制是怎么实现的？

### 核心答案

**观察者模式：定义对象间一对多依赖，当一个对象状态变化，所有依赖它的对象自动收到通知。**

```java
// JDK 内置（已废弃）
class NewsOffice extends Observable {
    void publish(String news) {
        setChanged();         // 标记状态已变
        notifyObservers(news); // 通知所有观察者
    }
}
class Subscriber implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("收到新闻：" + arg);
    }
}
```

**JDK 内置 Observer 被废弃的原因：**
1. `Observable` 是一个类而非接口，**单继承**限制了 Observer 的复用
2. 线程安全依赖 `setChanged()` 的手动调用，容易遗漏导致不通知
3. 顺序通知没有保障
4. Java 9 正式废弃，官方推荐使用 `Flow` API 或自定义事件机制

**Spring 事件机制（现代方案）：**

```java
// 1. 定义事件（继承 ApplicationEvent）
class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
    public Order getOrder() { return order; }
}

// 2. 定义监听器（@EventListener 注解方式）
@Component
class OrderListener {
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("收到订单事件：" + event.getOrder().getId());
    }

    // 条件监听：只有订单金额 > 100 才触发
    @EventListener(condition = "#event.order.amount > 100")
    public void handleHighValueOrder(OrderCreatedEvent event) {
        System.out.println("高价值订单：" + event.getOrder().getId());
    }
}

// 3. 发布事件
@Autowired
private ApplicationEventPublisher publisher;
public void createOrder(Order order) {
    orderRepo.save(order);
    publisher.publishEvent(new OrderCreatedEvent(this, order));
}
```

Spring 事件机制底层：基于**观察者模式** + **策略模式** + **单例模式**。事件发布后，通过 `ApplicationEventMulticaster` 广播给所有匹配的监听器，支持 `@Async` 异步执行、 `@Order` 排序、`@SpEL` 条件表达式。

### 追问方向

- "Spring 的 `SmartInstantiationAwareBeanPostProcessor` 用的是什么模式？" → 观察者模式，在 Bean 初始化生命周期各节点广播给所有后置处理器
- "Guava 的 `EventBus` 和 Spring 事件机制有什么区别？" → EventBus 是进程内同步/异步事件总线，不需要 Spring 容器；Spring 事件依赖 ApplicationContext 容器
- "观察者模式和 MQ 消息队列的区别？" → 观察者模式是**同步**的，发布者和订阅者在同一线程；MQ 是**异步**的，可以跨进程、跨机器。观察者模式适用于 JVM 内一对多解耦，MQ 适用于分布式解耦

### 避坑提示

- JDK 内置 `Observer/Observable` 已被废弃，**不要在代码里使用**。面试中如果主动提到"我知道它被废弃了"会非常加分
- Spring 事件的 `@TransactionalEventListener` 可以让事件在事务提交后才发布，适用于事务相关的监听逻辑

---

## 第 10 题：适配器模式

### 题目

> 适配器模式有哪两种实现方式？Spring MVC 的 `HandlerAdapter` 是怎么用适配器的？

### 核心答案

**适配器模式：将一个类的接口转换成客户期望的另一个接口，让不兼容的类能一起工作。**

```java
// 类适配器：通过继承实现
interface Target { void request(); }
class Adaptee { public void specificRequest() { System.out.println("特殊请求"); } }

// 类适配器（继承 Adaptee，实现 Target）
class ClassAdapter extends Adaptee implements Target {
    @Override
    public void request() { specificRequest(); } // 转换调用
}
```

```java
// 对象适配器：通过组合实现（更常用，符合合成复用原则）
class ObjectAdapter implements Target {
    private Adaptee adaptee;
    public ObjectAdapter(Adaptee adaptee) { this.adaptee = adaptee; }
    @Override
    public void request() { adaptee.specificRequest(); }
}
```

| 对比 | 类适配器 | 对象适配器 |
|------|---------|-----------|
| 机制 | 继承（单继承限制） | 组合（更灵活） |
| 覆盖 | 可以覆盖 adaptee 的方法 | 无法覆盖，只能委托 |
| 依赖 | 强依赖 | 弱依赖 |
| 推荐场景 | 需要覆盖部分 adaptee 行为 | 大部分场景 |

**Spring MVC HandlerAdapter：**

```
HandlerMapping（映射） → HandlerAdapter（适配） → 业务处理
     /user              ↓
  Controller ------→ HandlerAdapter
                          ↓
                    Controller（实现 Controller 接口）
                    HttpRequestHandler（实现 HttpRequestHandler 接口）
                    Servlet（直接继承 HttpServlet）
```

Spring MVC 需要支持多种 `Handler`（Controller 接口、`HttpRequestHandler`、注解 `@RequestMapping`），每种 handler 的处理方法签名不同。`HandlerAdapter` 通过适配器模式，**统一对外暴露相同的 `handle()` 接口**，屏蔽底层 handler 的差异：

```java
public interface HandlerAdapter {
    boolean supports(Object handler);          // 判断是否支持该 handler
    ModelAndView handle(HttpServletRequest request,
                        HttpServletResponse response,
                        Object handler) throws Exception; // 统一入口
}
```

### 追问方向

- "适配器模式和装饰器模式有什么区别？" → 适配器**改变接口**使其兼容（做转换），装饰器**不改变接口**只增强功能
- "Spring Security 的 `UserDetailsService` 是适配器吗？" → 是的，它把自定义用户对象适配成 Spring Security 需要的 `UserDetails` 接口
- "适配器模式在系统集成中怎么用？" → 集成第三方 SDK、遗留系统对接、接口版本兼容，用适配器包装旧接口而不是修改调用方代码

### 避坑提示

- 适配器模式经常被误认为装饰器模式，记住：**适配器是改变接口的"翻译官"，装饰器是不改变接口的"增强器"**
- Spring MVC 的 `DispatcherServlet` 正是通过 HandlerAdapter 实现了对 Controller、Servlet、HttpRequestHandler 的统一调度，体现了适配器模式的价值

---

## 第 11 题：桥接模式

### 题目

> 桥接模式的核心是什么？JDBC 驱动是怎么体现桥接模式的？

### 核心答案

**桥接模式核心：抽象部分和实现部分分离，使两者可以独立变化。**

```
Abstraction（抽象部分） →  RefinedAbstraction（扩充抽象）
         ↕（桥接）                  ↕
Implementation（实现部分）→ ConcreteImplementation（具体实现）
```

**JDBC 桥接模式：**

```
Java 程序（抽象）
    ↓ DriverManager.getConnection() （桥接）
    ↓
JDBC API（java.sql.* 接口：Connection, Statement, ResultSet）
    ↓ （每种数据库厂商各自实现）
MySQL Driver / Oracle Driver / PostgreSQL Driver（实现部分）
```

关键：Java 程序只依赖 `java.sql` 接口（抽象），不关心底层是 MySQL 还是 Oracle。切换数据库只需换驱动包，不改业务代码。

**浏览器兼容示例：**

```java
// 实现部分接口（不同浏览器内核）
interface RenderingEngine { void render(String shape); }
class OpenGLRenderer implements RenderingEngine { ... }
class VulkanRenderer implements RenderingEngine { ... }
class DirectXRenderer implements RenderingEngine { ... }

// 抽象部分（图形 API）
abstract class GraphicLibrary {
    protected RenderingEngine engine; // 桥接引用
    protected GraphicLibrary(RenderingEngine engine) { this.engine = engine; }
    abstract void drawCircle();
}

class Direct2D extends GraphicLibrary { // 抽象部分变化独立
    Direct2D(RenderingEngine engine) { super(engine); }
    @Override void drawCircle() { engine.render("circle"); }
}
```

可以独立变化的两条维度：
- **抽象**：图形库（Direct2D、OpenGL-ES、Vulkan）
- **实现**：渲染引擎（OpenGL、Vulkan、DirectX）

### 追问方向

- "桥接模式和装饰器/代理有什么区别？" → 桥接是**两个维度**独立扩展，装饰器是**同一维度**层层叠加，代理是**不改变结构**的控制访问
- "JDBC 为什么是桥接模式而不是适配器模式？" → JDBC 定义了一套**抽象接口**，各数据库实现这套接口，是"抽象与实现分离"；适配器是**转换已有接口**，两者核心动机不同
- "JDBC-ODBC 桥是怎么回事？" → ODBC 是 C 语言的数据库接口，JDBC-ODBC Bridge 是把 JDBC API 适配成 ODBC，是**适配器模式**的经典应用

### 避坑提示

- 桥接模式最容易被忽略，很多候选人在类图里见过但项目里从没用过。能说出 JDBC 驱动这个例子就超过了大部分人
- 桥接模式用**组合**替代继承，避免了类爆炸（如果用继承，`m` 种抽象 × `n` 种实现 = `m×n` 个类）

---

## 第 12 题：组合模式

### 题目

> 组合模式用在什么场景？Spring 的 Component 层级是怎么体现的？文件系统目录结构能用组合模式吗？

### 核心答案

**组合模式核心：让单个对象和组合对象有一致的接口，树形结构中客户代码可以统一处理叶子节点和容器节点。**

```java
abstract class FileSystemNode {
    protected String path;
    public FileSystemNode(String path) { this.path = path; }
    public abstract int countFiles();       // 统一的抽象操作
    public abstract long getSize();         // 叶子/容器都实现
}

class File extends FileSystemNode {         // 叶子节点
    public File(String path) { super(path); }
    @Override public int countFiles() { return 1; }
    @Override public long getSize() { return Files.size(Path.of(path)); }
}

class Directory extends FileSystemNode {    // 容器节点（组合）
    private List<FileSystemNode> children = new ArrayList<>();
    public Directory(String path) { super(path); }
    public void add(FileSystemNode node) { children.add(node); }

    @Override
    public int countFiles() {
        return children.stream().mapToInt(FileSystemNode::countFiles).sum();
        // 容器递归调用，每个子节点都实现了统一接口
    }
    @Override
    public long getSize() {
        return children.stream().mapToLong(FileSystemNode::getSize).sum();
    }
}
```

**Spring Component 层级（组合模式的体现）：**

```
Component（根接口）
  ├── Controller（@Controller）
  │     └── 每个 @RequestMapping 方法
  ├── Service
  │     └── ServiceImpl
  └── Repository
        └── JPA Repository / MyBatis Mapper
```

`GenericApplicationContext` 在刷新容器时，会把 `BeanDefinition` 按 `component-class` 类型注册，整个容器对 `BeanFactory` 暴露统一的 `getBean` 接口，但内部是多种 `BeanDefinition` 的组合。Spring MVC 的 `HandlerMapping` 也用组合模式管理多个 `HandlerMapping` 实现。

### 追问方向

- "组合模式怎么解决透明性和安全性的矛盾？" → 透明性：组件接口包含子节点操作（叶子也有空实现），调用方不用判断；安全性：在组件接口中去掉子节点操作，容器提供单独方法。**Spring 用透明性**（Composite 模式）
- "组合模式和责任链模式都涉及递归，区别在哪？" → 组合模式是**树形结构**，所有节点走同一条逻辑链；责任链是**链式结构**，每个节点选择处理或传递
- "组合模式适合什么数据结构？" → **树形结构**：文件系统、GUI 组件树、组织架构、XML/HTML DOM

### 避坑提示

- 组合模式的关键是**让叶子节点和容器节点实现同一接口**，客户代码可以递归地处理整个树
- Spring 源码中组合模式主要用在容器内的 BeanDefinition 注册和管理，不一定每个候选人都知道这个角度

---

## 第 13 题：外观模式

### 题目

> 外观模式是什么？Tomcat 里是怎么用的？为什么说它体现了迪米特法则？

### 核心答案

**外观模式：为子系统中一组接口提供统一的高层入口，简化客户代码和子系统的交互。**

```java
// 子系统：多个复杂类
class CPU { public void start() { /* ... */ } }
class Memory { public void load() { /* ... */ } }
class Disk { public void read() { /* ... */ } }

// 外观类：提供简单接口
class ComputerFacade {
    private CPU cpu = new CPU();
    private Memory memory = new Memory();
    private Disk disk = new Disk();

    public void start() {           // 一键启动，屏蔽了底层复杂性
        cpu.start();
        memory.load();
        disk.read();
    }
}

// 客户代码：不需要知道 CPU/Memory/Disk 怎么协作
new ComputerFacade().start();
```

**Tomcat 门面模式（StandardWrapper valve）：**

Tomcat 的 Valve（阀门）链是责任链，每个 Valve 处理 HTTP 请求的不同方面。`StandardWrapperValve` 是其中一个 Valve，它作为门面，封装了 `filterChain.doFilter()` 和 `servlet.service()` 的调用链，让 Valve 层不需要知道 Servlet 和 Filter 的细节。

```java
// Tomcat 门面模式的典型应用：RequestFacade / ResponseFacade
// HttpServletRequest 有一个实现类是 RequestFacade
// 它包装了复杂的 Tomcat 内部请求处理细节，对外暴露简化的 HttpServletRequest 接口
public class RequestFacade extends Request implements HttpServletRequest {
    // 对外只暴露接口方法，隐藏了 Tomcat 内部实现
}
```

**迪米特法则体现：** 客户代码只需要认识 `ComputerFacade` 这一个"朋友"，不需要知道 `CPU`、`Memory`、`Disk` 等子系统内部细节。外观模式就是迪米特法则最直接的实现手段。

### 追问方向

- "外观模式和中介者模式都减少耦合，区别在哪？" → 外观模式是**单向**的（客户→子系统），子系统不知道外观的存在；中介者是**双向**的（对象之间互相知道，通过中介者通信）
- "外观模式违背开闭原则吗？" → 如果为子系统新增操作要改外观类，但这是外观类本身的职责；子系统的变化不需要改外观，符合 OCP
- "Spring 里有外观模式吗？" → `ApplicationContext` 就是最大的门面，它统一暴露 `getBean()`、`getEnvironment()` 等各种功能，让客户不需要知道背后有多少子系统

### 避坑提示

- 外观模式是最简单的模式之一，很多框架都在用，但要能**从源码里认出它**才能证明你真正理解
- Tomcat 源码中 `RequestFacade`、`ResponseFacade` 是面试加分项，说出来比背定义有力得多

---

## 第 14 题：享元模式

### 题目

> 享元模式解决什么问题？String 常量池和 Integer 缓存是享元模式吗？线程池为什么是享元模式的应用？

### 核心答案

**享元模式核心：复用大量细粒度对象，减少内存占用。** 关键是把对象分为**内部状态**（可共享）和**外部状态**（不可共享，由调用方管理）。

```java
// 享元工厂：管理享元对象池
class ChessPieceFactory {
    private static final Map<String, ChessPiece> pool = new HashMap<>();

    public static ChessPiece getChessPiece(String color) {
        // 内部状态是 color（只有黑白），外部状态是位置 (x,y)
        return pool.computeIfAbsent(color, k -> new ChessPiece(color));
    }
}

class ChessPiece { // 享元对象：只存内部状态
    private final String color; // 可共享
    public void draw(int x, int y) { /* x,y 是外部状态，不存在对象里 */ }
}
```

**String 常量池：**

```java
String s1 = "hello";        // 字面量，直接查常量池
String s2 = "hello";        // 返回常量池中已有对象，s1 == s2
String s3 = new String("hello"); // 堆中新建对象
s1 == s2        // true（同一引用）
s1 == s3        // false（不同对象）
"hello".intern() // 手动入池，返回池中引用
```

String 在 JVM 中是典型的享元模式——字符串字面量放在**方法区字符串常量池**中，相同字面量不会创建多个对象。`"hello" + "world"` 在编译期会被优化成 `"helloworld"`，共享同一个常量池对象。

**Integer 缓存：**

```java
Integer a = 127;   // Integer.valueOf(127)，命中缓存
Integer b = 127;   // 返回同一个 Integer 对象
a == b             // true（-128 到 127 的 ByteIntegerCache）

Integer c = 128;   // 超出缓存范围
Integer d = 128;   // 新建对象
c == d             // false
```

`Integer.valueOf(int)` 内部有一个 `IntegerCache`（默认缓存 -128~127），这个缓存就是享元模式的体现。

**线程池（享元模式的经典应用）：**

```java
// 线程池：复用线程对象，而不是每次都创建/销毁
ExecutorService pool = Executors.newFixedThreadPool(4);
// Runnable/Callable 是外部状态（任务），线程是内部状态（可复用）
```

线程池复用工作线程，任务（Runnable）作为外部状态传入，这是享元模式的核心——**内部状态（线程对象）可复用，外部状态（任务）由调用方管理**。

### 追问方向

- "享元模式和单例模式都能减少对象创建，区别是什么？" → 单例是**全局唯一**，享元是**相同内部状态的对象共享**，可以有多个享元对象
- "Spring 里有享元模式吗？" → `StringHttpMessageConverter` 用到了 `Charset` 常量池；` PropertySource` 也有享元特征
- "享元模式怎么避免线程安全问题？" → 享元对象必须是**不可变**的（内部状态设成 final），或者线程安全的，否则共享会出问题

### 避坑提示

- Integer 缓存范围默认是 -128~127，但可以通过 `-Djava.lang.Integer.IntegerCache.high=512` 调整，这是一道经典的"你以为知道其实不知道"题
- 享元模式的本质是**分离内部状态和外部状态**，如果对象没有明显的内部/外部状态划分，硬套享元只会增加复杂度

---

## 第 15 题：责任链模式

### 题目

> 责任链模式怎么实现？MyBatis 的 Plugin 机制是怎么基于拦截器链的？

### 核心答案

**责任链模式核心：让多个处理器对象形成一条链，请求沿链传递，直到被某个处理器处理（或链结束）。**

```java
abstract class Handler {
    private Handler next; // 下一个处理器
    public Handler setNext(Handler next) { this.next = next; return next; } // 链式装配
    public final void handle(Request request) {
        if (doHandle(request)) return; // 处理了就不继续
        if (next != null) next.handle(request); // 没处理就传给下一个
    }
    protected abstract boolean doHandle(Request request); // 子类实现处理逻辑
}

class AuthHandler extends Handler {
    @Override
    protected boolean doHandle(Request request) {
        if (request.needAuth() && !request.isAuth()) {
            request.reject("未登录");
            return true; // 处理了
        }
        return false; // 放行
    }
}

class ValidateHandler extends Handler { /* ... */ }
// 使用
new AuthHandler()
    .setNext(new ValidateHandler())
    .setNext(new BusinessHandler())
    .handle(request);
```

**MyBatis Plugin 机制（拦截器链）：**

MyBatis 的四大核心对象（Executor、StatementHandler、ParameterHandler、ResultSetHandler）都支持插件拦截。插件通过实现 `Interceptor` 接口，在方法调用时做环绕增强：

```java
@Intercepts({
    @Signature(type = StatementHandler.class,
              method = "prepare",
              args = {Connection.class, Integer.class})
})
public class MyPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 目标方法执行前的拦截逻辑
        Object proceed = invocation.proceed(); // 执行目标方法
        // 目标方法执行后的增强逻辑
        return proceed;
    }
}
```

底层基于 **JDBC 动态代理**（`Plugin.wrap()` 用 `Proxy.newProxyInstance` 生成代理对象），`InvocationHandler` 是 `Plugin` 本身。每个拦截器包装下一个对象，形成**层层代理的责任链**。

### 追问方向

- "Servlet Filter 和 MyBatis Plugin 有什么相同和不同？" → 相同：都是责任链模式；不同：Filter 是 Web 容器的标准扩展机制，MyBatis Plugin 是基于 JDK 动态代理的拦截机制
- "Spring Security 的过滤器链是怎么组成的？" → `FilterChainProxy` 包含多个 Security Filter（`SecurityContextPersistenceFilter`、`UsernamePasswordAuthenticationFilter` 等），请求进入后按顺序逐一通过，每个 Filter 决定是否继续传递
- "责任链模式和装饰器模式都能增强对象，区别在哪？" → 装饰器是**同层叠加**（每个装饰器都包装同一个对象），责任链是**链式传递**（每个处理器决定处理或放行）

### 避坑提示

- MyBatis 插件的 `@Signature` 注解里 `args` 的顺序必须和目标方法签名完全一致，这是实际踩坑点
- 责任链模式如果链太长会增加延迟，线上日志排查时调用栈会非常深，容易找不到原始调用点

---

## 第 16 题：迭代器模式

### 题目

> 迭代器模式解决了什么问题？fail-fast 和 fail-safe 有什么区别？ArrayList 和 CopyOnWriteArrayList 分别用了哪种？

### 核心答案

**迭代器模式核心：提供一种方法顺序访问集合元素，不暴露集合底层结构（List、Set、Map 都不需要知道遍历方式）。**

```java
interface Iterator<T> {
    boolean hasNext();
    T next();
    void remove(); // 可选
}

class ArrayListIterator<T> implements Iterator<T> {
    private final List<T> list;
    private int index = 0;
    public ArrayListIterator(List<T> list) { this.list = list; }
    @Override public boolean hasNext() { return index < list.size(); }
    @Override public T next() { return list.get(index++); }
}
```

**fail-fast vs fail-safe：**

| 特性 | fail-fast（快速失败） | fail-safe（安全失败） |
|------|---------------------|---------------------|
| 机制 | 遍历同时检测结构性修改，抛 `ConcurrentModificationException` | 复制原集合，在副本上遍历 |
| 代表实现 | `ArrayList`、`HashMap` | `CopyOnWriteArrayList`、`ConcurrentHashMap` 的视图 |
| 内存开销 | 无额外开销 | 需要复制集合（开销大） |
| 数据一致性 | 遍历结果可能不完整 | 遍历的是快照，可能不一致 |

```java
// fail-fast 示例（ArrayList）
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {           // foreach 底层是 Iterator
    if ("b".equals(s)) {
        list.remove(s);           // ❌ 结构性修改，触发 ConcurrentModificationException
    }
}
// 正确做法：用 Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) {
        it.remove();              // ✅ Iterator.remove() 不触发 fail-fast
    }
}
```

`CopyOnWriteArrayList` 在每次修改（`add/set/remove`）时复制整个底层数组，遍历时读取的是快照，**不会抛异常但数据可能过时**。

### 追问方向

- "HashMap 的 fail-fast 是怎么实现的？" → 每个 HashMap 有一个 `modCount` 计数器，Iterator 遍历前记录 `expectedModCount`，遍历中每次 `next()` 前比较，不一致就抛异常
- "为什么不把 fail-safe 设成默认行为？" → 复制集合开销大，fail-fast 可以在开发阶段快速暴露并发问题（Bug），fail-safe 用于真正需要并发安全的场景
- "Java 的 `Iterator` 和 `ListIterator` 有什么区别？" → `Iterator` 只能前向遍历、`ListIterator` 可以双向遍历并支持添加/修改元素

### 避坑提示

- 很多候选人知道 fail-fast 但不知道 `Iterator.remove()` 是安全做法，这是现场手写代码的高频坑
- `CopyOnWriteArrayList` 适合**读多写少**的场景，写多了每次复制数组开销极大，不适合写密集型

---

## 第 17 题：命令模式

### 题目

> 命令模式是什么？它和策略模式的区别在哪？撤销功能怎么实现？

### 核心答案

**命令模式核心：将请求封装成对象，从而可以用不同的请求参数化客户，对请求排队、记录日志、支持撤销。**

```java
// 命令接口
interface Command {
    void execute();
    void undo(); // 支持撤销
}

// 具体命令
class LightOnCommand implements Command {
    private Light light;
    public LightOnCommand(Light light) { this.light = light; }
    @Override
    public void execute() { light.on(); }
    @Override
    public void undo() { light.off(); }
}

class LightOffCommand implements Command { /* ... */ }

// 调用者（Invoker）：持有命令对象
class RemoteControl {
    private Command command;
    public void setCommand(Command command) { this.command = command; }
    public void pressButton() { command.execute(); }
    public void pressUndo() { command.undo(); }

    // 批量撤销（命令历史）
    private Stack<Command> history = new Stack<>();
    public void execute(Command command) {
        command.execute();
        history.push(command);
    }
    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}
```

**命令模式的关键价值：**
- **请求封装**：把请求（参数+接收者）打包成对象
- **队列化**：把命令对象放进队列，按顺序执行
- **日志记录**：`Command.execute()` 中写日志，重启后重放
- **撤销支持**：每个命令记得自己的反向操作

### 追问方向

- "命令模式和策略模式有什么区别？" → 策略模式是**选择算法**，命令模式是**封装请求**；命令有 `execute()` 和可能的 `undo()`，策略只有 `execute()`；命令可以有 receiver（接收者），策略没有
- "数据库的 Undo Log 是不是命令模式？" → 从广义上说，Undo Log 保存了"反向操作"（撤销 SQL），和命令模式的 undo 思路一致，但实现机制不同
- "Spring 里有命令模式吗？" → `JdbcTemplate` 的 `execute(ConnectionCallback<T>)` 回调、Spring MVC 的 `HandlerInterceptor` 的 `postHandle` 都有命令模式的影子

### 避坑提示

- 命令模式和策略模式容易混淆，抓住核心：**策略是选择"怎么做"，命令是封装"做什么"**
- 如果被追问撤销的实现，必须说清楚每个 Command 要实现 `undo()`，调用方维护命令历史栈（Stack）

---

## 第 18 题：备忘录模式

### 题目

> 备忘录模式用来干什么？游戏存档和数据库事务回滚是怎么体现的？

### 核心答案

**备忘录模式核心：不破坏封装性地保存和恢复对象内部状态。**

```java
// 发起人：需要保存状态的对象
class GameRole {
    private int hp;
    private int mp;
    public Memento save() { return new Memento(hp, mp); } // 保存快照
    public void restore(Memento m) { this.hp = m.getHp(); this.mp = m.getMp(); }
    public void fight() { this.hp -= 50; this.mp -= 30; }
}

// 备忘录：只存储 GameRole 的内部状态（不引用 GameRole，防止泄露封装）
class Memento {
    private final int hp;
    private final int mp;
    public Memento(int hp, int mp) { this.hp = hp; this.mp = mp; }
    public int getHp() { return hp; }
    public int getMp() { return mp; }
}

// 管理者：保存/管理多个备忘录
class Caretaker {
    private final Map<String, Memento> saves = new HashMap<>();
    public void save(String slot, Memento m) { saves.put(slot, m); }
    public Memento load(String slot) { return saves.get(slot); }
}
```

**数据库事务回滚（undo log 机制）：**

```
开始事务
  ↓
记录 undo log（每条修改前的数据快照）
  ↓
执行 SQL（修改数据）
  ↓
  成功 → COMMIT，删除 undo log
  失败 → ROLLBACK，按 undo log 逆向恢复
```

事务的 undo log 就是备忘录模式的应用——**保存数据修改前的状态**，回滚时恢复。每个事务的 undo log 是按链表逆序执行的（先回滚最后修改的记录）。

### 追问方向

- "备忘录模式怎么避免破坏封装性？" → 备忘录类只保存状态，不引用发起人；且备忘录由发起人主动创建，管理者只能持有、不能修改
- "和原型模式（clone）保存状态有什么区别？" → 原型模式复制整个对象，备忘录只保存**需要的部分状态**；原型模式是创建新对象，备忘录是精确恢复
- "备忘录模式在 Android 开发中怎么用？" → `onSaveInstanceState()` / `onRestoreInstanceState()` Bundle 机制就是备忘录模式

### 避坑提示

- 备忘录模式有三个角色：**Originator（发起人）、Memento（备忘录）、Caretaker（管理者）**，必须说清楚三者职责才能让面试官满意
- 状态信息大时备忘录会占用大量内存，可以用**增量快照**或**命令行模式**（只记录操作而不是完整状态）来优化

---

## 第 19 题：状态模式

### 题目

> 状态模式和策略模式长得很像，怎么区分？订单状态流转是怎么用状态模式实现的？

### 核心答案

**状态模式核心：对象内部状态改变时，其行为也跟着改变——把状态转换逻辑封装到状态对象里。**

```java
// 订单上下文（Context）
class Order {
    private OrderState state; // 组合状态对象
    public void setState(OrderState state) { this.state = state; }
    public void pay() { state.pay(this); }
    public void deliver() { state.deliver(this); }
    public void receive() { state.receive(this); }
}

// 订单状态接口
interface OrderState {
    void pay(Order order);      // 支付
    void deliver(Order order);  // 发货
    void receive(Order order);  // 收货
    void cancel(Order order);   // 取消
}

// 具体状态：待支付
class PendingState implements OrderState {
    @Override
    public void pay(Order order) {
        // 状态转换：待支付 → 已支付
        order.setState(new PaidState());
        System.out.println("支付成功");
    }
    @Override public void deliver(Order order) { throw new IllegalStateException("未支付不能发货"); }
    @Override public void receive(Order order) { throw new IllegalStateException("未支付不能收货"); }
    @Override public void cancel(Order order) { order.setState(new CancelledState()); }
}

// 具体状态：已支付
class PaidState implements OrderState {
    @Override public void pay(Order order) { throw new IllegalStateException("已支付"); }
    @Override
    public void deliver(Order order) {
        order.setState(new DeliveredState());
        System.out.println("发货成功");
    }
    @Override public void receive(Order order) { throw new IllegalStateException("未发货不能收货"); }
    @Override public void cancel(Order order) { /* 可能需要退款逻辑 */ order.setState(new CancelledState()); }
}
```

**状态模式 vs 策略模式（核心区别）：**

| 维度 | 状态模式 | 策略模式 |
|------|---------|---------|
| 关注点 | **状态转换**（状态决定行为） | **算法选择**（用户选择行为） |
| 调用方 | 客户端只调用 Context，状态对象自己决定转换 | 客户端**主动**选择策略对象 |
| 状态对象 | **彼此知道**，可以互相转换 | **互不相知**，独立存在 |
| 运行时 | Context 状态在运行中**自动变化** | 策略可以**随时切换**，但需要外部触发 |
| 典型场景 | 订单流程、TCP 连接、灯开关 | 支付方式、排序算法、压缩算法 |

### 追问方向

- "状态机（State Machine）和状态模式是什么关系？" → 状态机是**理论模型**，状态模式是**实现状态机的一种编码方式**。Spring StateMachine 是状态机的框架实现
- "状态模式怎么避免状态类爆炸？" → 状态多时用**状态转换表**（Map<State, Map<Event, State>>）或**分层状态机**
- "为什么不直接用 if-else 判断状态？" → 订单状态 4 种还好，但如果状态转换规则复杂（状态+事件→新状态），if-else 会变成一锅粥，且每种状态的行为分散在各处，状态模式把它们**集中到状态类里**

### 避坑提示

- 区分状态模式和策略模式的经典问题："你自己开车（策略）" vs "车根据油量自动切换驾驶模式（状态）"——一个是用户决定，一个是系统根据内部状态决定
- 状态模式中状态转换规则可以放到**状态工厂**里管理，而不是散落在各个状态类中

---

## 第 20 题：设计模式综合

### 题目

> Spring 源码里用到了哪些设计模式？说出来至少 10 种。你怎么理解"避免过度设计"？

### 核心答案

**Spring 源码设计模式（至少 10 种）：**

| # | 设计模式 | Spring 中的体现 |
|---|---------|---------------|
| 1 | **工厂模式** | `BeanFactory`、`FactoryBean` |
| 2 | **单例模式** | Spring Bean 默认单例（`scope="singleton"`） |
| 3 | **代理模式** | Spring AOP（JDK/CGLIB 动态代理） |
| 4 | **装饰器模式** | `BufferedReader` 包装 `FileReader`（Spring 用装饰器处理 I/O） |
| 5 | **策略模式** | `Resource` 的 `getInputStream()` 实现、事务传播行为 |
| 6 | **模板方法模式** | `JdbcTemplate`、`RestTemplate`、`TransactionTemplate` |
| 7 | **观察者模式** | Spring 事件机制 `ApplicationEvent`、`@EventListener` |
| 8 | **适配器模式** | `HandlerAdapter` |
| 9 | **桥接模式** | JDBC 驱动管理（`DriverManager` 与具体驱动解耦） |
| 10 | **组合模式** | Spring Component 层级、BeanDefinition 树形结构 |
| 11 | **外观模式** | `ApplicationContext` 统一入口，`WebMvcConfigurer` |
| 12 | **享元模式** | `StringHttpMessageConverter` 的 Charset 常量、`PropertySource` |
| 13 | **责任链模式** | `HandlerInterceptor` 链、MyBatis Plugin 链 |
| 14 | **迭代器模式** | `BeanDefinitionIterator`、`Environment` 属性迭代 |
| 15 | **命令模式** | `JdbcTemplate` 的 ConnectionCallback、事务管理的回调 |

**如何避免过度设计：**

```
过度设计的信号：
❌ 只有 2-3 个类，上来就分了 5 层
❌ 预计永远不会有扩展的地方，提前做了可扩展设计
❌ 简单查询也用了策略模式 + 工厂模式 + 装饰器模式
❌ 设计图很美，但代码量和复杂度远超实际需求
```

**实用原则：**

1. **YAGNI 原则**（You Aren't Gonna Need It）：不要做你认为将来会需要的抽象，只做当前需要的
2. **先简单后复杂**：先用 if-else 或简单工厂，等复杂度真的上升了再重构
3. **成本收益评估**：引入设计模式带来的灵活性 > 额外复杂度成本，才值得
4. **代码可读性优先**：团队其他人能不能看懂？如果为了设计模式牺牲可读性，就是过度设计
5. **重构成模式**：先用简单方式实现，等模式出现得自然而然再提取——而不是一开始就套模式

**真实案例：** 项目中很多"为了以后扩展"的设计最终都被删掉了。好的设计是在**解决当前问题**和**保持扩展性**之间找到平衡，而不是提前打满所有补丁。

### 追问方向

- "Spring Boot 的自动配置是怎么判断要用哪个 Bean 的？" → 条件注解 `@ConditionalOnMissingBean`、`@ConditionalOnClass`，基于条件注册 Bean
- "MyBatis 里用了哪些设计模式？" → 装饰器（LogProxy）、责任链（Plugin）、工厂（SqlSessionFactory）、模板方法（JdbcExecutor）
- "你在项目中有过过度设计的教训吗？" → 诚实回答，展示对工程化的理解

### 避坑提示

- 说出 10+ 种设计模式不难，但一定要能**结合 Spring 源码具体类名**说出来，空对空背书说服力差
- 过度设计这个问题很容易变成"你觉得自己设计能力强不强"的试探，回答时不要过度谦虚也不可过度自信，用真实项目案例最有说服力

---

## 附录：设计模式速查表

| 模式 | 分类 | 核心意图 |
|------|------|---------|
| 工厂方法 | 创建型 | 产品由子类工厂创建 |
| 单例 | 创建型 | 全局唯一实例 |
| 建造者 | 创建型 | 分步构建复杂对象 |
| 原型 | 创建型 | clone 复制对象 |
| 适配器 | 结构型 | 接口转换 |
| 装饰器 | 结构型 | 动态增加职责 |
| 代理 | 结构型 | 控制访问 |
| 桥接 | 结构型 | 两个维度独立变化 |
| 组合 | 结构型 | 树形结构统一处理 |
| 外观 | 结构型 | 统一入口 |
| 享元 | 结构型 | 对象池复用 |
| 观察者 | 行为型 | 一对多通知 |
| 策略 | 行为型 | 算法可互换 |
| 模板方法 | 行为型 | 算法骨架 |
| 责任链 | 行为型 | 链式处理请求 |
| 迭代器 | 行为型 | 统一遍历接口 |
| 命令 | 行为型 | 请求封装 |
| 备忘录 | 行为型 | 快照恢复 |
| 状态 | 行为型 | 状态驱动的行为变化 |

> 本题库覆盖 Java 设计模式核心面试知识点，建议配合源码阅读（Spring Framework、MyBatis）加深理解。祝你面试顺利！

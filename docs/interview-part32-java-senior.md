# Java 高级进阶面试题（第32期）

> 适用级别：P7+ / 高级研发 / 架构师  
> 涵盖范围：JVM 底层、并发编程、字节码机制、模块系统、问题诊断  
> 每题结构：题目 → 核心答案 → 追问方向 → 避坑提示

---

## 第 1 题：JVM 内存结构与 OOM 场景

### 题目
请描述 JVM 运行时数据区的构成，并分别说明各区域可能出现的 OOM（OutOfMemoryError）场景。

### 核心答案

JVM 运行时数据区按线程是否共享分为两类：

| 区域 | 线程是否共享 | 可能 OOM |
|------|------------|---------|
| **堆（Heap）** | ✅ 共享 | `java.lang.OutOfMemoryError: Java heap space` |
| **方法区（Method Area）** | ✅ 共享 | `java.lang.OutOfMemoryError: Metaspace`（JDK 8+）或 `PermGen space`（JDK 7-） |
| **虚拟机栈（VM Stack）** | ❌ 私有 | `java.lang.StackOverflowError`（栈帧过深）或 OOM（线程申请过多栈内存） |
| **本地方法栈（Native Method Stack）** | ❌ 私有 | 同上，在 JNI 场景下触发 |
| **程序计数器（PC Register）** | ❌ 私有 | 无 OOM（唯一不抛 OOM 的区域） |

**堆内存结构（JDK 8+）**：
- Young Generation：`Eden` + `Survivor S0` + `Survivor S1`
- Old Generation
- Metaspace（非堆，从本地内存分配）

**OOM 典型场景**：
```java
// 堆溢出：大量对象创建
List<byte[]> list = new ArrayList<>();
while(true) list.add(new byte[1024*1024]);

// 栈溢出：递归调用没有退出条件
public int foo() { return foo() + 1; }

// Metaspace 溢出：动态类生成（CGLIB、ASM大量字节码加载）
```

### 追问方向
- 对象在堆中的分配过程（TLAB、Eden 区分配）
- 为什么 JDK 8 用 Metaspace 替换 PermGen？
- `StackOverflowError` 和 OOM 在虚拟机栈上的区别

### 避坑提示
- 别说"方法区就是永久代"——JDK 8 已经移除永久代，方法区在 Metaspace
- OOM 发生在堆不一定只是内存泄漏，也可能是业务确实需要大内存（此时应调大堆）
- 程序计数器是唯一**不会**发生 OOM 的区域，面试时主动提会加分

---

## 第 2 题：垃圾回收算法与收集器

### 题目
列举主要的垃圾回收算法，并对比三代收集器（年轻代/老年代/不分代）的核心区别。

### 核心答案

**三大基础算法**：

| 算法 | 思路 | 优点 | 缺点 |
|------|------|------|------|
| **标记-清除（Mark-Sweep）** | 先标记存活对象，再清除未标记对象 | 简单 | 产生内存碎片 |
| **复制（Copying）** | 将存活对象复制到另一块空白区，原区全部清除 | 无碎片，效率高 | 浪费一半空间 |
| **标记-整理（Mark-Compact）** | 标记后移动存活对象紧邻排列，再清边界外内存 | 无碎片 | 移动成本高 |

**分代收集理论**：
- 多数对象朝生夕死 → 年轻代用复制算法（Minor GC）
- 长寿对象进入老年代 → 老年代用标记-整理/标记-清除（Major/Full GC）

**主要收集器**：

| 收集器 | 工作区域 | 算法 | 特性 | 并行/并发 |
|--------|---------|------|------|----------|
| **Serial** | 年轻代 | 复制 | 单线程，STW | 串行 |
| **ParNew** | 年轻代 | 复制 | Serial 的多线程版本 | 并行 |
| **Parallel Scavenge** | 年轻代 | 复制 | 吞吐量优先（`-XX:MaxGCPauseMillis` / `-XX:GCTimeRatio`） | 并行 |
| **Serial Old** | 老年代 | 标记-整理 | Serial 老年代版本 | 串行 |
| **Parallel Old** | 老年代 | 标记-整理 | 吞吐量优先 | 并行 |
| **CMS** | 老年代 | 标记-清除 | 低延迟目标（`-XX:CMSInitiatingOccupancyFraction`） | 并发 |
| **G1** | 不分代（整堆） | 标记-整理+复制 | 可预测停顿（`-XX:MaxGCPauseMillis`），化整为零 | 并发 |
| **ZGC** | 不分代（整堆） | 标记-整理 | 停顿时间 < 1ms（`-XX:+UseZGC`） | 并发 |
| **Shenandoah** | 不分代（整堆） | 标记-整理 | 停顿时间极短，不依赖 JVM 版本 | 并发 |

**G1 vs CMS 对比**：
- CMS：老年代收集器，采用"标记-清除"算法，会产生内存碎片；并发阶段影响用户线程
- G1：将堆划分为多个大小相等的 Region（1MB~32MB），优先回收价值最大的 Region，兼顾吞吐量与延迟

### 追问方向
- G1 的 Region 划分和 Humongous 区域（大对象）
- ZGC 的 Color Pointers 机制与读屏障
- 为什么 CMS 要使用"初始标记→并发标记→重新标记→并发清除"而非全程并发

### 避坑提示
- 别说"G1 是新生代收集器"——G1 是整堆收集器，可同时收集年轻代和老年代
- 分代收集器不是"只用一种算法"，而是不同代用不同算法
- CMS 已经 deprecated（JDK 14+），ZGC/Shenandoah 是未来方向，但 CMS 仍是面试高频

---

## 第 3 题：类加载机制与双亲委派

### 题目
什么是双亲委派模型？有哪些场景需要打破双亲委派？JDK SPI 如何实现？

### 核心答案

**类加载器的层次结构**：

```
Bootstrap ClassLoader（C++实现，加载 JAVA_HOME/lib）
    ↑
Extension ClassLoader（加载 JAVA_HOME/lib/ext）
    ↑
Application ClassLoader（加载 classpath -Dcp 指定路径）
    ↑
自定义 ClassLoader
```

**双亲委派流程**：类加载请求向上传递直到 Bootstrap ClassLoader，只有父加载器无法完成时，才由子加载器自己加载。

```java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);       // 1. 检查是否已加载
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);  // 2. 父加载器加载
                } else {
                    c = findBootstrapClassOrNull(name); // 3. Bootstrap 尝试
                }
            } catch (ClassNotFoundException e) {}
            if (c == null) {
                c = findClass(name);  // 4. 自身加载
            }
        }
        return c;
    }
}
```

**打破双亲委派的典型场景**：

| 场景 | 原因 |
|------|------|
| **Tomcat** | 每个 Webapp 有独立的 `WebappClassLoader`，优先加载自身 `/WEB-INF/lib` 和 `/classes`，隔离不同应用的类 |
| **JDBC 驱动加载** | `DriverManager` 在 Bootstrap ClassLoader 加载，但其实现类（如 `com.mysql.jdbc.Driver`）在 Application ClassLoader，需要逆向委派 |
| **OSGi** | 每个 bundle 有独立类加载器，形成网状结构 |
| **JNDI** | JDK 1.3 引入了 `Thread.currentThread().getContextClassLoader()` |

**JDK SPI（Service Provider Interface）打破方式**：
```java
// java.sql.DriverManager 源码片段
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
// 使用 Thread.currentThread().getContextClassLoader() 加载 SPI 实现类
ClassLoader cl = Thread.currentThread().getContextClassLoader();
```
通过**线程上下文类加载器**（Context ClassLoader）实现逆向查找：父类加载器去使用子类加载器加载的类。

### 追问方向
- 双亲委派的优势（类的唯一性、安全性、防止核心 API 被篡改）
- Tomcat 的 `StandardWrapper.loadServlet()` 源码流程
- `Thread.currentThread().setContextClassLoader()` 在框架中的应用（Spring、JDBC）

### 避坑提示
- 别说"所有框架都遵守双亲委派"——Tomcat、OSGi 等都打破了它
- SPI 和双亲委派的关系要讲清楚：SPI 本质上是"主动使用上下文类加载器"来规避双亲委派的限制
- `Class.forName()` 默认使用调用者的 ClassLoader，与线程上下文类加载器的区别

---

## 第 4 题：JVM 调优实战

### 题目
列举常用 JVM 调优参数，并说明如何通过 GC 日志和 Arthas 进行问题诊断。

### 核心答案

**核心调优参数**：

| 参数 | 含义 | 示例 |
|------|------|------|
| `-Xms` | 堆初始大小 | `-Xms2g` |
| `-Xmx` | 堆最大大小 | `-Xmx2g` |
| `-Xmn` | 年轻代大小 | `-Xmn1g` |
| `-Xss` | 线程栈大小 | `-Xss1m`（JDK 8 默认 1MB） |
| `-XX:MetaspaceSize` | Metaspace 初始大小 | `-XX:MetaspaceSize=256m` |
| `-XX:MaxMetaspaceSize` | Metaspace 最大大小 | `-XX:MaxMetaspaceSize=512m` |
| `-XX:NewRatio` | 老年代/年轻代比例 | `-XX:NewRatio=2`（老年代:年轻代=2:1） |
| `-XX:SurvivorRatio` | Eden/Survivor 比例 | `-XX:SurvivorRatio=8`（Eden:S0:S1=8:1:1） |
| `-XX:+UseG1GC` | 使用 G1 收集器 | |
| `-XX:MaxGCPauseMillis` | 最大 GC 停顿时间目标 | `-XX:MaxGCPauseMillis=200` |

**GC 日志参数**：
```bash
# 经典 GC 日志配置（JDK 8）
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/myapp-gc.log
-XX:+UseGCLogFileRotation
-XX:GCLogFileSize=10M
-XX:NumberOfGCLogFiles=5

# JDK 9+ 日志改革（统一日志框架）
-Xlog:gc*:/var/log/myapp-gc.log:time,uptime,level,tags
```

**GC 日志核心指标**：
```
2025-05-11T10:15:30.123+0800: [GC (Allocation Failure) 
  [PSYoungGen: 524288K->43520K(761856K)] 1.2G->650M(2G), 0.035s]
```
- `Allocation Failure`：年轻代分配失败（Minor GC 触发原因）
- `PSYoungGen`：Parallel Scavenge 年轻代
- `1.2G->650M(2G)`：GC 前堆使用→GC后堆使用（堆总大小）

**Arthas 常用命令**：
```bash
# 启动
java -jar arthas-boot.jar

# 核心诊断命令
dashboard              # 查看进程整体信息（CPU、内存、线程、GC）
thread -n 3           # 查看 CPU 占比 top 3 的线程
jvm                    # 查看 JVM 参数、内存区域、GC 信息
heapdump /tmp/dump.hprof  # 导出堆快照（等价于 jmap -dump）
ognl '@java.lang.System@getProperty("java.version")'  # OGNL 表达式
mc /root/Test.java     # 内存编辑器（编译）
redefine /root/Test.class  # 热更新类
```

**JVM 参数校验命令**：
```bash
jps -l                          # 查看 Java 进程
jinfo -flags <pid>             # 查看所有 VM 参数
jinfo -flag <flagname> <pid>   # 查看单个参数值
```

### 追问方向
- 如何通过 GCEasy（gceasy.io）分析 GC 日志
- MAT（Memory Analyzer Tool）分析堆 dump 的常用功能（Dominator Tree、Leak Suspects）
- Young GC 和 Full GC 的频率控制与业务延迟的关系

### 避坑提示
- `-Xms` 和 `-Xmx` 通常设成一样，避免堆动态伸缩带来的性能抖动
- GC 日志文件要配置滚动，否则容易写满磁盘
- Arthas 热更新不是真正的类重定义（`redefine` 加载后类结构不能改），`retransform` 才是

---

## 第 5 题：JIT 编译优化

### 题目
JIT（即时编译器）如何工作？逃逸分析带来了哪些优化（锁消除、锁粗化）？

### 核心答案

**JIT 编译器的工作原理**：

JVM 解释执行字节码效率低，JIT 将"热点代码"（Hot Spot Code）编译为本地机器码执行。

```
字节码 → 解释执行（首次）→ 方法调用计数器和回边计数器达到阈值
      → 触发 JIT 编译 → 编译为本地机器码 → 直接执行
```

**分层编译（JIT Tiered Compilation，JDK 8+ 默认开启）**：

| 层级 | 编译级别 | 说明 |
|------|---------|------|
| 0 | 解释执行 | C1 编译（未优化）|
| 1 | 简单 C1 编译 | 无 profiling 的机器码 |
| 2 | 受限 C1 编译 | 有基本 profiling |
| 3 | 完全 C1 编译 | 完整 profiling |
| 4 | C2 编译 | 激进优化（仅非递归热方法）|

**逃逸分析（Escape Analysis）**：
分析对象的动态作用域，判断是否"逃逸"出线程。

```java
public void foo() {
    // 分析：sb 不会逃逸出 foo() 方法
    StringBuilder sb = new StringBuilder();
    sb.append("hello");
    System.out.println(sb.toString());
    // JIT 可能优化：直接用 String concat 或消除锁
}
```

**逃逸分析带来的优化**：

| 优化 | 说明 | 示例 |
|------|------|------|
| **锁消除（Lock Elision）** | 对象不逃逸时，synchronized 无意义，JIT 消除锁 | 上例中 `StringBuilder` 上的锁会被消除 |
| **锁粗化（Lock Coarsening）** | 相邻 synchronized 块合并，减少加锁解锁次数 | 多个连续的 `append` 合并为一次加锁 |
| **标量替换（Scalar Replacement）** | 对象不逃逸时，将其成员变量拆解为分散的局部变量 | 避免对象分配 |
| **栈上分配（Stack Allocation）** | JDK 11+ 实验性支持，对象在栈上分配（替代堆分配） | |

### 追问方向
- `-XX:+PrintCompilation` 输出解读
- C2 编译器的激进优化与去优化（Reoptimization）机制
- GraalVM 和 JVMCI（JVM Compiler Interface）对 JIT 的影响

### 避坑提示
- 逃逸分析默认开启（`+XX:+DoEscapeAnalysis`），别在简历上写"关闭逃逸分析提升性能"
- 锁消除发生在 JIT 编译时，运行时无法通过 `synchronized` 关键字本身看出来
- 对象分配在堆还是栈由 JIT 决定，不是程序员显式控制的

---

## 第 6 题：Java 内存模型（JMM）

### 题目
描述 Java 内存模型（JMM）的结构，以及 volatile、CAS、synchronized 的实现原理。

### 核心答案

**JMM 的核心抽象**：

```
线程A                    线程B
┌─────────┐            ┌─────────┐
│ 工作内存 │←──────────→│ 工作内存 │
│ (CPU缓存)│   刷新      │ (CPU缓存)│
└────┬────┘   读取/写入 └────┬────┘
     │     主内存（RAM）      │
     └────────────────────────┘
```

- **主内存（Main Memory）**：所有共享变量存储的物理内存
- **工作内存（Working Memory）**：每个线程独有的 CPU 缓存/寄存器副本

**8 种内存屏障**（Load 和 Store 的组合）：

| 屏障类型 | 指令 | 作用 |
|---------|------|------|
| LoadLoad | `Load1; LoadLoad; Load2` | 确保 Load1 在 Load2 之前读取 |
| StoreStore | `Store1; StoreStore; Store2` | 确保 Store1 对其他线程可见先于 Store2 |
| LoadStore | `Load1; LoadStore; Store2` | 确保 Load1 先于 Store2 |
| StoreLoad | `Store1; StoreLoad; Load2` | 最强屏障，同时刷新写缓冲和失效队列 |

**volatile 的实现原理**：
- 写操作后插入 `StoreStore` + `StoreLoad` 屏障
- 读操作前插入 `LoadLoad` + `LoadStore` 屏障
- 保证**可见性**（线程对 volatile 变量的写入立即刷新到主内存，后续线程读取强制从主内存加载）和**有序性**（禁止指令重排序）
- **不保证原子性**（如 `i++` 这类复合操作）

**CAS（Compare-And-Swap）原理**：
```java
// AtomicInteger.incrementAndGet() 底层
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
// unsafe.getAndAddInt 内部：do...while(!compareAndSwapInt(obj, offset, expect, update))
```
- CAS 是 CPU 原子指令（`cmpxchg`），在多核 CPU 上需要 `lock cmpxchg` 配合缓存一致性协议（MESI）
- ABA 问题：加版本号（`AtomicStampedReference`）或时间戳（`AtomicMarkableReference`）解决

**synchronized 原理**：
- JDK 6 之前：操作系统互斥锁（重量级），用户态→内核态切换开销大
- JDK 6+：引入**偏向锁**（Biased Locking，单线程重复获取）和**轻量级锁**（自旋 CAS + 对象头 Mark Word 升级）
- `synchronized` 字节码：`monitorenter` / `monexit`
- 底层依赖对象头的 **Mark Word**（存储锁状态、哈希码、GC 信息）和 **ObjectMonitor**（JDK 6+）

### 追问方向
- synchronized 的四种状态转换路径（无锁→偏向锁→轻量级锁→重量级锁）
- `ReentrantLock` 与 `synchronized` 的区别（可中断、可超时、公平锁）
- happens-before 原则的 8 条规则

### 避坑提示
- volatile 不保证 `i++` 原子性——这是高频踩坑点
- synchronized 不是"轻量级锁就一定比重量级锁快"，在严重竞争下自旋会消耗 CPU
- JMM 的目的是解决可见性和有序性问题，不仅仅是"内存分配在哪里"

---

## 第 7 题：并发编程进阶工具

### 题目
对比 Phaser、CyclicBarrier、CountDownLatch 和 ForkJoinPool 的使用场景与核心区别。

### 核心答案

| 工具 | 线程间同步方式 | 能否重用 | 典型场景 |
|------|--------------|---------|---------|
| **CountDownLatch** | 倒计数门栓，计数到 0 放行 | ❌ 不可重用（计数不可重置）| 主线程等待 N 个子任务完成 |
| **CyclicBarrier** |  parties 个线程互相等待，全部到达后一起放行 | ✅ 可重用（调用 `reset()`）| 多线程计算后合并结果 |
| **Phaser** | 动态注册 parties，支持多阶段屏障 | ✅ 可重用，支持动态增减 parties | 多阶段任务（如分词→清洗→统计） |
| **ForkJoinPool** | 工作窃取（Work-Stealing），分治任务 | ✅ 可重用（提交多个任务）| 递归任务并行化（归并排序、遍历树）|

**CountDownLatch 示例**：
```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) new Thread(() -> { /* work */ latch.countDown(); }).start();
latch.await(); // 主线程等待
```

**CyclicBarrier 示例**：
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("汇总结果"));
for (int i = 0; i < 3; i++) new Thread(() -> {
    // 并行计算
    barrier.await();  // 所有人等齐
    // 一起执行下一阶段或汇总
}).start();
```

**Phaser 示例**：
```java
Phaser phaser = new Phaser(3); // 3个parties
phaser.arriveAndAwaitAdvance(); // 阶段1屏障
phaser.arriveAndAwaitAdvance(); // 阶段2屏障
// 支持动态注册：phaser.register() / phaser.arriveAndDeregister()
```

**ForkJoinPool 工作窃取原理**：
```
线程1: [task A] → [task B] → steal ← [task E(从线程2偷)]
线程2: [task C] → [task D] → steal ← [task F(从线程1偷)]
```
- 每个线程有自己的双端队列（deque）
- 空闲线程从其他线程队列尾部偷任务
- 适合计算密集型、递归拆分任务
- JDK 8 `ParallelStream` 底层使用 `ForkJoinPool.commonPool()`

### 追问方向
- Phaser 的 `arrive()` vs `arriveAndAwaitAdvance()` vs `arriveAndDeregister()`
- ForkJoinPool 的 `invoke()` vs `submit()` vs `execute()` 区别
- 为什么 ForkJoinPool 适合递归而不适合 IO 密集型任务

### 避坑提示
- CountDownLatch 一旦计数到 0 就不可再用，想复用选 CyclicBarrier 或 Phaser
- `CyclicBarrier.await()` 超时会抛 `BrokenBarrierException`，需要处理
- ForkJoinPool 的任务不能有相互依赖，否则会死锁（因为工作窃取是后进先出）

---

## 第 8 题：AQS 源码实现

### 题目
从 AQS（AbstractQueuedSynchronizer）源码角度，分析 ReentrantLock、CountDownLatch、Semaphore 的实现原理。

### 核心答案

**AQS 核心结构**：
```java
public abstract class AbstractQueuedSynchronizer {
    // 双向 FIFO 队列（CHL 队列）
    private volatile int state;         // 同步状态
    private Node head;                 // 等待队列头
    private Node tail;                 // 等待队列尾
    
    static final class Node {
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        Node nextWaiter;              // SHARED 或 EXCLUSIVE 模式
    }
}
```

**AQS 两种模式**：
- **独占模式（Exclusive）**：`ReentrantLock`——同一时刻只有一个线程持有
- **共享模式（Shared）**：`CountDownLatch`、`Semaphore`、`CyclicBarrier`——多个线程同时获取

**ReentrantLock（独占，可重入）实现**：
```java
// NonfairSync（非公平锁）acquire 流程：
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// tryAcquire 由子类实现（ReentrantLock 自己的 Sync 类）
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {                      // 无锁状态
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current); // 标记持有线程
            return true;
        }
    } else if (getExclusiveOwnerThread() == current) { // 可重入
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```
- `state` 表示锁的持有次数（重入计数）
- 释放时 `state - 1`，减到 0 才完全释放

**CountDownLatch（共享）实现**：
```java
// 内部类 Sync 继承 AQS
protected int tryAcquireShared(int acquires) {
    return getState() == 0 ? 1 : -1; // state=0 返回1（成功），否则-1（失败）
}

protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0) return false; // 已经为0
        int nextc = c - 1;
        if (compareAndSetState(c, nextc)) // CAS 递减
            return nextc == 0;            // 减到0才返回true，唤醒等待线程
    }
}
```
- 每次 `countDown()` 调用 `tryReleaseShared`，递减 `state`
- `state` 减到 0 时，`doReleaseShared()` 唤醒队列中的 `Node.SHARED` 节点

**Semaphore（共享，限流）实现**：
```java
// NonfairSync
protected int tryAcquireShared(int reduces) {
    for (;;) {
        int available = getState();
        int remaining = available - reduces;
        if (remaining < 0 || compareAndSetState(available, remaining))
            return remaining;
    }
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int p = getState();
        if (compareAndSetState(p, p + releases)) return true;
    }
}
```
- `state` 表示可用许可证数量
- `acquire()` 消费许可证，`release()` 归还许可证

### 追问方向
- ReentrantLock 公平锁和非公平锁的实现差异（`FairSync` 多了 `hasQueuedPredecessor()` 判断）
- Condition 的 `await()` / `signal()` 原理（AQS 内部的等待队列，与 CHL 队列并列）
- AQS 为什么用双向链表而不是单向链表

### 避坑提示
- AQS 本身只是框架，`state` 的含义由子类定义（CountDownLatch=计数，Semaphore=许可数，ReentrantLock=重入次数）
- `tryAcquire` 等方法由子类实现，AQS 只负责排队和阻塞
- 不要把 AQS 和 `Lock` 接口混淆——AQS 是实现类，`Lock` 是接口

---

## 第 9 题：ThreadLocal 源码与内存泄漏

### 题目
分析 ThreadLocal 的源码实现，ThreadLocalMap 的弱引用机制，以及内存泄漏的原因。

### 核心答案

**ThreadLocal 数据结构**：
```
Thread.currentThread() 
    ↓
ThreadLocalMap（ThreadLocalMap 是 Thread 的成员变量）
    ↓
Entry[] table
    ↓
Entry { key: ThreadLocal<?> (弱引用), value: Object }
```

**ThreadLocalMap 核心源码**：
```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    private Entry[] table;
    private int size;
    
    // get()
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (len-1);
        Entry e = table[i];
        if (e != null && e.get() == key) return e;
        return getEntryAfterMiss(key, i, e); // 线性探测
    }
    
    // set()
    private void set(ThreadLocal<?> key, Object value) {
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = table[i]; e != null; e = table[i = nextIndex(i, len)])
            if (e.get() == key) { e.value = value; return; }
        table[i] = new Entry(key, value);
        if (++size >= threshold) rehash();
    }
    
    // 弱引用清理（expungeStaleEntry）
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        tab[staleSlot].clear();  // 清除弱引用，value仍存在
        // 重新扫描，清理过期Entry（value和key都为null的Entry）
    }
}
```

**为什么使用弱引用（WeakReference）？**：

```
如果 Entry 的 key 使用强引用：
  ThreadLocalRef（强引用）→ ThreadLocal 对象
  当代码中不再有 ThreadLocal 强引用时，ThreadLocal 对象仍被 Map key 引用 → 内存泄漏

如果 Entry 的 key 使用弱引用：
  ThreadLocalRef（强引用）→ ThreadLocal 对象
  WeakReference → ThreadLocal 对象（仅 Map 持有）
  当不再有外部强引用时，GC 可回收 ThreadLocal 对象
  → key=null（俗称"空洞"），下次 get/set 时惰性清理 value
```

**内存泄漏的完整原因链**：

```
线程池（长生命周期 Thread）
    ↓ 持有 ThreadLocalMap
    ↓ Map 持有大量 Entry
    ↓ Entry.value（强引用）→ 大对象（如数据库连接、缓存数据）
    
    ① ThreadLocal 对象被回收 → key=null
    ② value 仍在，被 Entry 强引用（如果用强引用的话）→ 内存泄漏
    ③ 即使 key 是弱引用，value 仍是强引用 → 弱引用清理了 key，但 value 未清理
    ④ ThreadLocalMap 的 Entry[] 不收缩 → 内存持续占用
```

**正确使用 ThreadLocal 的方式**：
```java
// 每次使用完后手动清理（最佳实践）
ThreadLocal<Object> tl = new ThreadLocal<>();
try {
    tl.set(obj);
    // use obj
} finally {
    tl.remove(); // 显式 remove，防止内存泄漏
}

// 或者使用 JDK 11+ 的 InheritableThreadLocal
```

### 追问方向
- `ThreadLocal.withInitial(() -> defaultValue)` JDK 8+ 新API
- `InheritableThreadLocal` 的继承机制与问题（父子线程传递，但线程池复用场景下有问题）
- ThreadLocalMap 的扩容机制（`rehash()` / `resize()`，负载因子 2/3）

### 避坑提示
- 不要在 ThreadPool 中直接使用 ThreadLocal 而不清理——线程复用时旧数据串到新任务
- 弱引用只保护 key 不保护 value，value 的清理依赖 `get()`/`set()` 时的惰性删除或显式 `remove()`
- `ThreadLocalMap.size()` 不准确（只计有效 entry），`expungeStaleEntries()` 后才能准确

---

## 第 10 题：并发容器详解

### 题目
对比 ConcurrentHashMap、ConcurrentLinkedQueue 和 ConcurrentSkipListMap 的实现原理与适用场景。

### 核心答案

**ConcurrentHashMap（JDK 8+）**：

| 维度 | JDK 7（Segment 分段锁） | JDK 8+（CAS + synchronized） |
|------|----------------------|--------------------------|
| 结构 | Segment[] + HashEntry[] + 链表 | Node[] + 链表/红黑树 |
| 锁粒度 | 整个 Segment（16个） | 每个桶（bucket） |
| 并发度 | 固定 16 | 数组长度（动态）|
| 实现 | ReentrantLock | CAS + `synchronized`（头节点锁）|

```java
// JDK 8+ 核心 put 逻辑
putVal(hash, key, value, false, false);

// 1. 无冲突：直接 CAS 插入
// 2. 冲突：synchronized (node) 同步，遍历链表/树
// 3. 链表转红黑树：binCount >= TREEIFY_THRESHOLD(8)
// 4. 扩容：transfer() 支持多线程并发迁移
```

**ConcurrentLinkedQueue（无界 FIFO 队列）**：
- 基于**单向链表** + **CAS** 实现
- `offer()` / `poll()` 无阻塞，不保证 FIFO 顺序的绝对一致性（ABA 问题需注意）
- 内部结构：`Node { item: volatile, next: volatile }`
- 使用 `U.compareAndSwapObject()` 更新 `next` 指针
- 适合高并发队列场景，但**不适合需要阻塞的场景**（如 `BlockingQueue`）

**ConcurrentSkipListMap（跳表实现）**：
- 基于**跳表（Skip List）**实现，有序映射
- 时间复杂度：平均 `O(log n)`，最坏 `O(n)`（但实际几乎不可能发生）
- 线程安全的有序 Map（替代 `Collections.synchronizedMap(new TreeMap<>())`）
- 与 `ConcurrentHashMap` 的区别：后者无序，前者有序

**对比总结**：

| 容器 | 底层结构 | 有序 | 线程安全机制 | 适用场景 |
|------|---------|------|------------|---------|
| `ConcurrentHashMap` | 数组+链表+红黑树 | ❌ 无序 | CAS+synchronized | 高并发 K-V 读写（最常用）|
| `ConcurrentLinkedQueue` | 单向链表 | ✅ FIFO | CAS | 高并发无阻塞队列 |
| `ConcurrentSkipListMap` | 跳表 | ✅ 有序 | CAS | 高并发有序 K-V |

### 追问方向
- ConcurrentHashMap 的 `size()` 方法为什么不加锁（JDK 8：累加 modCount，不准确但快速）
- 为什么 ConcurrentHashMap 不支持 `null` key/value（防止二义性）
- `CopyOnWriteArrayList` 的适用场景（读多写少的一致性要求不高的场景）

### 避坑提示
- `ConcurrentHashMap` 的并发度不是固定 16——JDK 8+ 取决于数组长度
- `ConcurrentLinkedQueue` 无限增长，没有 `capacity()` 方法
- 不要再写 JDK 7 旧版本的 ConcurrentHashMap 分段锁实现，那是过时的知识

---

## 第 11 题：字节码增强技术

### 题目
ASM、Javassist、ByteBuddy 分别如何实现字节码增强？AOP 的字节码实现原理是什么？

### 核心答案

**三大字节码操作库对比**：

| 库 | 层级 | 编程模型 | 性能 | 典型框架 |
|----|------|---------|------|---------|
| **ASM** | 底层字节码指令 |  Visitor 模式（手动操作字节数组） | 最高 | JVM 自身、cglib 底层 |
| **Javassist** | 源代码级别 | 修改方法体字符串 / CtClass | 中等 | Hibernate（早期）、Dubbo RPC |
| **ByteBuddy** | 高层 API | 注解 + 链式 API（`AgentBuilder`）| 较快 | Mockito、Mockito、Arthas 字节码注入 |

**ASM 使用示例（Visitor 模式）**：
```java
ClassReader cr = new ClassReader(bytes);
ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_FRAMES);
cr.accept(new ClassVisitor(ASM9, cw) {
    @Override
    public MethodVisitor visitMethod(int access, String name, ...) {
        MethodVisitor mv = super.visitMethod(access, name, desc, sig, exceptions);
        if ("targetMethod".equals(name)) {
            // 在方法入口前插入
            mv.visitCode();
            mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            mv.visitLdcInsn("Method called: targetMethod");
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V");
        }
        return mv;
    }
}, 0);
```

**ByteBuddy 使用示例**：
```java
new AgentBuilder.Default()
    .type(ElementMatchers.named("MyService"))
    .transform((builder, typeDescription, classLoader, module) ->
        builder.method(named("doSomething"))
            .intercept(MethodDelegation.to(MyInterceptor.class))
    ).installOn(inst); // inst = Instrumentation
```

**AOP 字节码增强原理（cglib/ByteBuddy 动态代理）**：

```
被代理类 BaseService
    ↓
ByteBuddy 在运行时生成子类 BaseService$$ByteBuddy$$xxx
    ↓
覆盖目标方法，插入增强逻辑
    ↓
调用 super.targetMethod() 保留原始行为
```

```java
// cglib 实现原理
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(BaseService.class);
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method m, Object[] args, MethodProxy mp) {
        before();
        Object result = mp.invokeSuper(obj, args); // 调用原始方法
        after();
        return result;
    }
});
BaseService proxy = (BaseService) enhancer.create();
```

**不同增强方式的选择**：

| 场景 | 推荐方案 |
|------|---------|
| 编译期织入（AspectJ） | 编译后直接修改 .class |
| 类加载时织入（Java Agent） | `premain` / `agentmain` |
| 运行时代理（Spring AOP） | JDK 动态代理 / CGLIB |
| 运行时字节码注入（Arthas） | Java Agent + 重新 transform |

### 追问方向
- Java Agent（`java.lang.instrument`）的 `premain` 和 `agentmain` 区别
- ASM `COMPUTE_FRAMES` vs `SKIP_FRAMES` 的取舍
- 为什么 ByteBuddy 比 Javassist 更快（Javassist 需要解析/重新生成字节码，ByteBuddy 直接生成）

### 避坑提示
- ASM 虽然性能最好但开发效率低，不建议在业务代码中使用
- `MethodProxy.invokeSuper()` vs `MethodProxy.invoke()` 的区别（前者更快但调用链路不同）
- Java 17+ 的 `JEP 411` 禁用了 `sun.reflect`，字节码增强更依赖 Agent 机制

---

## 第 12 题：类对象与 Class 对象

### 题目
Class 对象如何获取？反射获取类信息有哪些核心 API？Class 对象的加载时机是什么？

### 核心答案

**Class 对象的本质**：
- Class 类的实例，JVM 为每个类创建的**唯一**对象
- 存储了类的元信息（属性、方法、构造函数、修饰符、父类、注解等）
- 类的 `.class` 文件被类加载器加载后，在堆中生成 Class 对象

**获取 Class 对象的 5 种方式**：

```java
// 1. 字面量（编译期常量，不触发类初始化）
Class<?> c1 = String.class;

// 2. Object.getClass()
Class<?> c2 = "hello".getClass();

// 3. Class.forName()（触发类初始化）
Class<?> c3 = Class.forName("java.lang.String");

// 4. ClassLoader.loadClass()（不触发类初始化）
Class<?> c4 = classLoader.loadClass("java.lang.String");

// 5. primitive types（基本类型）
Class<?> c5 = int.class;
Class<?> c6 = Integer.TYPE; // 等价于 int.class
```

**Class.forName vs ClassLoader.loadClass**：

| 特性 | `Class.forName(String)` | `ClassLoader.loadClass(String)` |
|------|------------------------|-------------------------------|
| 是否初始化类 | ✅ 初始化（static 块执行）| ❌ 不初始化 |
| 使用哪个类加载器 | 调用者的类加载器 | 指定类加载器 |
| 典型场景 | JDBC 驱动加载（触发 static 块注册驱动）| OSGi、插件框架（延迟加载）|

**反射核心 API**：

```java
// 获取类信息
Class<?> cls = Class.forName("com.example.User");

// 获取属性
Field[] fields = cls.getDeclaredFields();    // 所有属性（含私有）
Field[] fields2 = cls.getFields();           // 公开属性

// 获取方法
Method[] methods = cls.getDeclaredMethods(); // 所有方法（含私有）
Method m = cls.getMethod("toString");         // 公开方法

// 获取构造函数
Constructor<?>[] cons = cls.getDeclaredConstructors();

// 获取父类/接口
Class<?> supercls = cls.getSuperclass();
Class<?>[] interfaces = cls.getInterfaces();

// 创建对象
Object obj = cls.getDeclaredConstructor().newInstance();

// 调用方法
Method setName = cls.getMethod("setName", String.class);
setName.invoke(obj, "Alice");

// 访问私有属性/方法
f.setAccessible(true); // 禁用安全检查
f.set(obj, "value");
```

### 追问方向
- `getDeclaredFields()` vs `getFields()` 的区别（Declared 包含私有，Fields 仅 public）
- `Class` 对象的加载时机（第一次主动使用时，由类加载器的 `loadClass` → `defineClass` 触发）
- 数组类型、注解类型的 Class 对象特殊处理（`Array.newInstance()`，`Class.forName("int[]")` 特殊语法）

### 避坑提示
- `Class` 对象在**堆**中，但很多人误以为在方法区——这是错误认知
- `getMethods()` 返回 public 方法（含继承的 Object 方法），`getDeclaredMethods()` 返回本类声明的所有方法
- `Class.forName()` 默认**初始化**类，如果只想加载不用，优先用 `ClassLoader.loadClass()`

---

## 第 13 题：动态代理

### 题目
对比 JDK 动态代理、CGLIB 和 Javassist 的实现机制，Proxy.newProxyInstance 如何工作？

### 核心答案

**三者的核心对比**：

| 特性 | JDK 动态代理 | CGLIB | Javassist |
|------|------------|-------|----------|
| **原理** | 反射 + `InvocationHandler` | 继承（子类继承被代理类）| 字节码生成 |
| **代理类生成** | 运行时生成 `$Proxy0` 类 | 运行时生成子类字节码 | 编译时/运行时生成 |
| **被代理类要求** | 必须实现接口 | 可以不实现接口 | 可以不实现接口 |
| **性能** | 反射调用，慢 | 继承调用，FastClass（比反射快）| 字节码直接调用 |
| **限制** | 接口方法不能重复 | 不能代理 final/private 方法 | 能力介于两者之间 |
| **JDK 版本** | 所有版本 | 所有版本（需依赖 cglib 包）| 所有版本 |
| **典型框架** | Spring AOP（默认）、RPC 客户端 | Spring AOP（类代理）、MyBatis 延迟加载 | Dubbo RPC、字节码工具 |

**JDK 动态代理原理**：

```java
// 1. Proxy.newProxyInstance 源码
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) {
    // 1.1 生成代理类字节码（$Proxy0 extends Proxy implements Foo, Bar）
    Class<?> cl = getProxyClass0(loader, interfaces);
    
    // 1.2 获取构造函数（构造函数参数是 InvocationHandler）
    Constructor<?> cons = cl.getConstructor(InvocationHandler.class);
    
    // 1.3 调用构造函数创建实例
    return cons.newInstance(new Object[] { h });
}

// 2. 生成的 $Proxy0 字节码结构
public final class $Proxy0 extends Proxy implements Foo {
    // 每个接口方法：
    @Override
    public void doSomething() {
        h.invoke(this, m3, null); // h = InvocationHandler, m3 = doSomething 的 Method 对象
    }
}
```

**Spring AOP 中的选择策略**：

```java
// Spring 4.x + CGLIB
@EnableAspectJAutoProxy(proxyTargetClass = true)  // 强制使用 CGLIB
// Spring 5.x 默认 proxyTargetClass = true

// Spring 根据条件选择：
// 目标类实现了接口 → 默认使用 JDK 动态代理
// 目标类没实现接口 → 使用 CGLIB
```

**CGLIB 原理**：
```
被代理类 Target
    ↓
Enhancer.generateClass() 生成 Target$$EnhancerByCGLIB$$xxx
    ↓
覆盖所有非 final 方法
    ↓
FastClass（生成 TargetFastClass 和 Target$$FastClassByCGLIB$$xxx）
    ↓
方法调用直接通过 FastClass 索引，不走反射
```

### 追问方向
- JDK 动态代理为什么只能代理接口（因为 `extends Proxy implements Foo` 单一继承限制）
- `MethodHandler`（JDK 9+）替代 `InvocationHandler` 的改进
- Spring Boot 2.0 后为什么默认使用 CGLIB（解决 JDK 动态代理的 proxy 对象类型转换问题）

### 避坑提示
- 不要说"JDK 动态代理比 CGLIB 快"——JDK 动态代理每次调用要走反射，CGLIB 用 FastClass 更快
- CGLIB 不能代理 `final` 方法（无法被覆盖），也不能代理 `private` 方法
- Javassist 生成代码比手动写字节码快，因为不需要解析十六进制指令

---

## 第 14 题：泛型擦除

### 题目
什么是泛型类型擦除？桥接方法如何生成？泛型与反射混用时会有什么特殊情况？

### 核心答案

**类型擦除的过程**：

泛型在编译后被擦除为**上限类型**或 **Object**：

```java
// 源代码
class Container<T extends Number> {
    T value;
    void set(T value) { this.value = value; }
    T get() { return value; }
}

// 编译后（类型擦除后）
class Container {
    Number value;                    // T → Number（上限）
    void set(Number value) { this.value = value; }
    Number get() { return value; }   // 返回类型变成 Number
}
```

| 泛型声明 | 擦除后的类型 |
|---------|------------|
| `<T>` | `Object` |
| `<T extends A>` | `A` |
| `<T super B>` | `B`（JDK 10+ 也有了 `super` 通配符）|
| `<Integer>` | `Integer`（具体类型参数保留为 Integer）|

**桥接方法（Bridge Method）的生成**：

```java
// 源代码
class StringContainer extends Container<Integer> {
    @Override
    Integer get() { return 42; }
}

// 编译后（编译器自动生成桥接方法）
class StringContainer extends Container {
    // 原始方法
    Integer get() { return 42; }
    
    // 编译器生成的桥接方法
    Object get() {          // 方法签名变为 Object，与父类方法签名一致
        return this.get();   // 内部调用泛型版本的 get()
    }
}
```
- **为什么需要桥接方法**？确保子类override父类方法时，方法签名兼容
- `StringContainer` 的 `Integer get()` override 了父类的 `Object get()`？不对——这是**重写**，但编译器生成了 `Object get()` 作为对父类的实现

**泛型与反射混用**：

```java
// 通过反射获取泛型信息（利用 ParameterizedType）
class Response<T> {
    T data;
}
class UserResponse extends Response<User> {}

// 反射获取泛型父类的真实类型
Type genericSuperclass = UserResponse.class.getGenericSuperclass();
ParameterizedType pt = (ParameterizedType) genericSuperclass;
Type actualType = pt.getActualTypeArguments()[0]; // → User.class
```

```java
// 泛型数组的限制
// 编译错误：new T[10]，因为 T 被擦除为 Object
// 正确做法：
T[] array = (T[]) java.lang.reflect.Array.newInstance(componentType, size);

// 获取方法返回值的泛型类型
Method m = MyClass.class.getMethod("getData");
Type returnType = m.getGenericReturnType(); // 如果返回 List<String>，得到 ParameterizedType
```

### 追问方向
- `Class` 对象本身没有泛型，但 `Class<T>` 的泛型有什么用（`Class<T>` 的 `cast` 方法做类型检查）
- `getTypeParameters()` 获取泛型类型参数（`TypeVariable[]`）
- Gson/Jackson 的 TypeToken 如何突破泛型擦除（匿名内部类继承 `ParameterizedType` 记录泛型信息）

### 避坑提示
- `List<String>` 和 `List<Integer>` 在运行时是同一个 Class（`List.class`）——这是泛型"伪类型"的本质
- 泛型数组（如 `new List<String>[3]`）在 Java 中**永远不能**创建
- 反射可以绕过泛型检查（`Field.set(obj, "string")` 在 `Integer` 字段上），但编译期不报错

---

## 第 15 题：注解处理器

### 题目
编译时注解（Annotation Processor）和运行时注解有什么区别？AnnotationProcessor 如何工作？

### 核心答案

**两类注解处理器对比**：

| 维度 | 编译时注解（APT）| 运行时注解（反射处理）|
|------|----------------|-------------------|
| **处理时机** | 编译期间（`.java` → `.class`）| 运行期间（类加载后）|
| **处理工具** | `AnnotationProcessor`（JSR 269）| `反射 + AnnotatedElement` API |
| **性能影响** | 零运行时开销 | 增加类加载和反射开销 |
| **能做什么** | 生成新 `.java` / `.class` 文件、警告 | 运行时读取注解值，改变行为 |
| **典型框架** | Lombok（`@Getter/@Setter`）、Dagger、MapStruct | Spring（`@Controller/@Autowired`）、JUnit（`@Test`）|
| **能否修改原类** | 不能修改已存在的类字节码，但可以生成新类 | 不能改变类行为 |

**编译时注解处理流程**：

```
.java 源码
    ↓ javac 编译
    ↓ 解析 → 填充符号表
    ↓ 注解处理器（Round 1）
         ├── 发现注解 → 处理 → 生成 .java / .class
         └── Round 2（处理第一轮生成的 .java）
    ↓ ... 直到没有新文件生成
.class 输出
```

**自定义注解处理器示例**：

```java
// 1. 定义注解
@Retention(RetentionPolicy.SOURCE)  // SOURCE = 仅编译期
@Target(ElementType.METHOD)
public @interface BuilderProperty {}

// 2. 实现 AbstractProcessor
@SupportedAnnotationTypes("com.example.BuilderProperty")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {
    
    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (Element element : roundEnv.getElementsAnnotatedWith(BuilderProperty.class)) {
            // 获取被注解的元素信息
            ExecutableElement method = (ExecutableElement) element;
            TypeElement enclosing = (TypeElement) method.getEnclosingElement();
            
            // 生成新的 .java 文件（Filer）
            Filer filer = processingEnv.getFiler();
            // ... 创建源文件并写入
        }
        return true;
    }
}
```

**Lombok 的原理**：
```
Lombok.jar → 在 javac 编译时作为注解处理器加载
    ↓
解析 @Getter/@Setter 等注解
    ↓
修改 AST（抽象语法树），插入字段/方法节点
    ↓
继续编译，生成 .class（用户看不到 .java 的变化）
```
- **不是修改字节码**，而是修改 AST——所以 `.class` 中已经有 getter/setter

**运行时注解示例**：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyAnnotation {
    String value() default "";
}

// 使用反射读取
Method m = MyClass.class.getMethod("myMethod");
MyAnnotation ann = m.getAnnotation(MyAnnotation.class);
if (ann != null) {
    String val = ann.value();
}
```

### 追问方向
- `RoundEnvironment` 的 `getElementsAnnotatedWith()` 与 `getRootElements()` 的区别
- JDK 9 Module System 下注册注解处理器的方式（`META-INF/services/javax.annotation.processing.Processor`）
- 为什么 Lombok 在 IDE 中需要插件支持（IDE 使用自己的编译器，不是 javac）

### 避坑提示
- `RetentionPolicy.SOURCE` 注解仅在源码存在，编译后消失，运行时反射无法读取
- `@Inherited` 注解仅对**类**的继承有效，对接口/方法注解无效
- 不要混淆"编译时注解处理器"和"字节码增强"——前者不修改原类的字节码（Lombok 改的是 AST，不是字节码）

---

## 第 16 题：Java 9 模块化系统

### 题目
Java 9 的模块化系统（JPMS）是什么？module-info.java 的核心配置如何使用？

### 核心答案

**为什么需要模块化？**

- 之前 JAR hell（类路径传递依赖、类冲突）问题
- JDK 本身也变成模块化（JDK 9+ 将 rt.jar 拆分为 90+ 模块）
- 应用可以声明对外部模块的依赖，实现真正的封装

**module-info.java 语法**：

```java
// module-info.java（放在 src root 下）
module com.example.myapp {
    
    // 1. 依赖其他模块（_requires）
    requires com.example.utils;        // 强制依赖
    requires static com.example.extra; // 编译时依赖（运行时可选）
    requires transitive com.example.base; // 传递依赖
    
    // 2. 导出包（exports）—— 允许其他模块访问
    exports com.example.api;          // 导出接口
    exports com.example.dto;
    
    // 3. 仅内部使用（不导出）
    // com.example.internal 不会被外部模块访问
    
    // 4. 开放模块（opens）—— 运行时深度反射
    opens com.example.config;        // 允许运行时反射（模块级别）
    opens com.example.config to com.example.tools; // 仅允许特定模块反射
    
    // 5. 使用服务（uses）—— 声明使用服务接口
    uses com.example.spi.MyService;
    
    // 6. 提供服务实现（provides）—— 提供服务实现
    provides com.example.spi.MyService 
        with com.example.impl.MyServiceImpl;
}
```

**模块的四大特性**：

| 特性 | 说明 |
|------|------|
| **强封装** | 未导出的包对其他模块不可见 |
| **明确依赖** | `requires` 声明显式依赖 |
| **服务解耦** | `uses` + `provides` 解耦服务接口与实现 |
| **可配置可访问域** | `opens` 控制运行时反射可访问性 |

**模块路径 vs 类路径**：
```
JDK 9 之前：
  classpath = 所有 JAR 扁平放在一起，互相可见

JDK 9+：
  modulepath = 模块的层级结构，遵循 exports/requires 规则
  classpath = 仍可用，但放入"未命名模块"（unnamed module）
```

**常见兼容性问题**：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `java.lang.reflect.InaccessibleObjectException` | JDK 9+ 加强了模块边界 | `--add-opens java.base/java.lang=ALL-UNNAMED` |
| `NoClassDefFoundError` | 间接依赖被自动关闭 | 添加缺失的 `requires` |
| Split Package（分 package）| 同一包分布在多个 JAR | 合并 JAR |

**相关 JVM 参数**：
```bash
--module-path mods       # 指定模块路径
--add-modules java.sql   # 添加模块
--add-opens java.base/java.lang=ALL-UNNAMED  # 开放反射
--illegal-access=permit|deny   # JDK 16 前默认为 permit
```

### 追问方向
- `requires static`（可选依赖）和普通 `requires` 的区别
- `uses` vs `provides` 在 ServiceLoader 中的使用
- Maven/Gradle 对 Java 9 模块化的支持情况

### 避坑提示
- Java 9 模块化在实践中用得不多（大多数项目仍在 Java 8 或用类路径运行），但面试中考察的是对 Java 模块化思想的理解
- `opens` 是运行时授权，编译时仍然受 `exports` 控制
- 不要混淆"模块"和"Maven/Gradle 模块"——前者是语言层面的（JPMS），后者是构建工具的概念

---

## 第 17 题：引用类型与 ReferenceQueue

### 题目
Java 的四种引用类型（强/软/弱/虚）分别用于什么场景？WeakHashMap 的原理是什么？

### 核心答案

**四种引用类型对比**：

| 类型 | 回收时机 | 典型应用 |
|------|---------|---------|
| **强引用（Strong Reference）** | 从不回收（除非引用置 null）| `Object obj = new Object()` |
| **软引用（Soft Reference）** | **OOM 前** GC 时回收 | 缓存（内存敏感）|
| **弱引用（Weak Reference）** | **下一次 GC** 时回收 | `ThreadLocalMap` Entry key、`WeakHashMap` 键 |
| **虚引用（Phantom Reference）** | **任何时候**都可回收（无法 get 对象）| NIO 直接内存（`DirectByteBuffer`）回收通知 |

**ReferenceQueue 的作用**：

```java
// ReferenceQueue 用于跟踪已被 GC 回收的引用对象
ReferenceQueue<Object> queue = new ReferenceQueue<>();
SoftReference<byte[]> ref = new SoftReference<>(new byte[1024*1024], queue);

// 当 byte[] 被回收后，ref 被加入 queue
Reference<? extends byte[]> polled = queue.poll();
if (polled != null) {
    // 对象已被回收，可以在这里做清理逻辑
}
```

**WeakHashMap 原理**：

```java
// WeakHashMap 的 Entry 继承 WeakReference
private static class Entry<K,V> extends WeakReference<Object> 
    implements Map.Entry<K,V> {
    // key 作为弱引用，value 作为强引用
    Entry(Object key, V value, ReferenceQueue<Object> queue) {
        super(key, queue);
        this.value = value;
    }
}

// get/put 时会自动清理 key 已回收的 Entry
private expungeStaleEntries() {
    for (Object k; (k = queue.poll()) != null; ) {
        // 移除 key 已回收的 Entry
    }
}
```
- `WeakHashMap` 的 key 被 GC 回收后，Entry 的 value 仍然存活（强引用）
- 解决：`WeakHashMap` 会在每次操作时调用 `expungeStaleEntries()` 清理 value

**四种引用的 GC 行为对比**：

```java
// 强引用：永不回收
Object s = new Object(); // GC 不会回收 s 指向的对象（除非 s=null）

// 软引用：内存不足时回收
SoftReference<byte[]> sr = new SoftReference<>(new byte[10*1024*1024]);

// 弱引用：下次 GC 必回收
WeakReference<Thread> wr = new WeakReference<>(thread);

// 虚引用：get() 永远返回 null，用于追踪对象死亡
PhantomReference<Object> pr = new PhantomReference<>(obj, referenceQueue);
```

### 追问方向
- `Cleaner`（JDK 9+）替代 `PhantomReference` 用于 NIO 直接内存管理
- `WeakHashMap` 不适合做缓存（因为 key 回收后 value 不一定立即清理）
- `SoftReference` 的回收时机与 `-Xmx` 设置的关系

### 避坑提示
- 软引用不等于缓存——只有内存不足时才回收，所以不适合做严格意义上的缓存
- 虚引用本身无法获取对象，只能通过 ReferenceQueue 知道对象被回收了
- WeakHashMap 的 value 如果持有 key 的强引用，会阻止 key 被回收（反向内存泄漏）

---

## 第 18 题：JVM 问题排查工具

### 题目
如何使用 jstack、jmap、jhat、GCEasy 和 MAT 进行 JVM 问题排查？请说明各工具的使用场景。

### 核心答案

**jstack（查看线程堆栈）**：

```bash
# 导出线程堆栈
jstack -l <pid> > thread_dump.log

# 常用选项
jstack -l <pid>     # 列出所有锁信息（显示 java.util.concurrent 锁）
jstack -F <pid>     # Force 模式（进程卡死时用）
jstack -m <pid>     # 混合模式（包含 native 栈帧，仅支持 Solaris）
```

线程状态分析：
```
"http-nio-8080-exec-1" #XX daemon prio=5 os_prio=0 tid=0x... 
    java.lang.Thread.State: WAITING (parking)
    - parking to wait for <0x...> (a java.util.concurrent.locks.AbstractQueuedSynchronizer)
    java.lang.Thread.State: BLOCKED (on object monitor)  ← 阻塞状态要关注
```

**jmap（查看内存/生成堆 dump）**：

```bash
# 查看堆内存概况
jmap -heap <pid>

# 查看对象统计（按对象大小排序）
jmap -histo:live <pid> | head -20

# 生成堆快照（等同于 Arthas 的 heapdump）
jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# JDK 11+ 首选
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

**jhat（Heap Analysis Tool，已 deprecated）**：

```bash
# JDK 8 及之前可用，JDK 9+ 已移除
jhat /tmp/heap.hprof
# 启动 HTTP 服务 http://localhost:7000
# 在浏览器查看对象引用关系
```

**GCEasy（GC 日志分析）**：

```bash
# 将 GC 日志上传到 gceasy.io 或使用命令行工具
# 或者直接访问 https://gceasy.io
# 上传 GC 日志文件，自动分析：
#   - GC 吞吐量
#   - 停顿时间分布
#   - 对象分配率
#   - GC 算法推荐
#   - 内存泄漏建议
```

**MAT（Memory Analyzer Tool）**：

```bash
# 下载 eclipse.org/mat
# 打开 heap.hprof 文件
# 核心功能：
# 1. Histogram：查看对象数量和内存占用
# 2. Dominator Tree：支配树，找出占用内存最大的对象路径
# 3. Leak Suspects：自动分析可能的内存泄漏
# 4. Top Consumers：按类/包查看内存消耗
# 5. OQL（Object Query Language）：SQL 风格查询对象
```

```java
// OQL 示例：在 MAT 中查询占用大量内存的 String 对象
SELECT * FROM java.lang.String s WHERE s.count > 100
```

**Arthas 高级用法**：

```bash
# 火焰图（profiler）
profiler start --event cpu
profiler stop --format html > flame.html

# 查看方法调用耗时
trace com.example.Service doTask '#cost > 100'

# 动态修改日志级别
logger -c com.example --name ROOT -e DEBUG
```

### 追问方向
- `jcmd <pid> Thread.print` 与 `jstack <pid>` 的区别（jcmd 可向运行中 JVM 发送命令，不需要 attach）
- OOM 时如何自动 dump（`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp`）
- MAT 如何分析 `ConcurrentHashMap` 的桶分布（查看 `ConcurrentHashMap.Node[]` 数组）

### 避坑提示
- `jmap -dump:live` 会触发一次 Full GC，生成 dump 时注意对线上服务的影响
- `jhat` 在 JDK 9 后被移除，分析堆 dump 改用 MAT 或 `jcmd <pid> GC.heap_info`
- `jstack` 看 `BLOCKED` 线程数过多，说明锁竞争严重；`WAITING` 线程数过多，需要检查线程池配置

---

## 第 19 题：JIT 即时编译器

### 题目
JIT 编译器如何探测热点代码？C1 和 C2 编译器的区别是什么？分层编译如何工作？

### 核心答案

**热点代码探测机制**：

JVM 通过**计数器**判断代码是否为热点：

| 计数器 | 触发阈值 | 位置 |
|--------|---------|------|
| **方法调用计数器** | `-XX:CompileThreshold=10000`（默认）| 每个方法入口 |
| **回边计数器** | 相同阈值 | 循环体末尾（判断循环是否为热点）|

```java
// 计数逻辑（概念层面）
if (++invocationCount >= CompileThreshold) {
    // 触发 OSR（On-Stack Replacement）或不触发
    // 编译请求进入编译队列
}
```

**OSR（栈上替换）**：当一个方法已经在解释执行，JIT 编译完成需要替换当前栈帧（不常见于方法级别，常见于循环热点的替换）

**C1 vs C2 编译器**：

| 维度 | C1（Client Compiler）| C2（Server Compiler）|
|------|--------------------|--------------------|
| **目标** | 快速启动、快速编译 | 深度优化、峰值性能 |
| **优化级别** | 轻量级（方法内联、锁消除）| 激进（逃逸分析、向量优化）|
| **适用场景** | 客户端程序（启动要快）| 服务器程序（长期运行）|
| **JVM 选项** | `-client`（JDK 32位默认）| `-server`（JDK 64位默认）|
| **编译质量** | 较快但质量较低 | 编译慢但质量高 |

**C2 编译器的高级优化**：

- 激进方法内联（`MaxInlineSize` 默认 35字节）
- 逃逸分析（见第5题）
- 循环展开（Loop Unrolling）
- 去除高频分支（Branch Prediction）
- 标量替换

**分层编译（JIT Tiered，JDK 8+ 默认）**：

```
首次调用 → 解释执行（Tier 0）
    ↓ invocationCount 达到 0
Tier 1（C1，简单编译，无 profiling）
    ↓ 继续达到阈值
Tier 2（C1，有限 profiling）
    ↓
Tier 3（C1，完整 profiling）
    ↓ 方法足够热
Tier 4（C2，激进优化编译）

← 反优化（Deoptimization）→ 可能退回解释执行
```

```bash
# 分层编译参数
-XX:+TieredCompilation        # 开启（JDK 8+ 默认开启）
-XX:TieredStopAtLevel=1      # 仅用 C1 编译，用于快速启动测试
-XX:CompileThreshold=10000   # 方法调用阈值（影响 Tier 升级速度）
```

### 追问方向
- JIT 编译队列的管理（`CompilationPolicy` 接口，可自定义）
- GraalVM 对 JIT 的替代（C2 被 Graal 替代是趋势）
- `-XX:+PrintCompilation` 日志中各符号的含义（`%`=OSR，`s`=同步，`b`=阻塞编译）

### 避坑提示
- 分层编译不等于"C2 一定比 C1 好"——对于运行时间不长的代码，Tier 1-2 已经足够
- C2 编译器的激进优化可能导致"去优化"（Deoptimization），反而降低性能
- JIT 编译的热点探测基于调用次数，不是基于执行时间——高频短方法优先被编译

---

## 第 20 题：字符串拼接

### 题目
StringBuilder、StringBuffer 和 String.concat 的区别是什么？字符串常量池是如何工作的？

### 核心答案

**三者对比**：

| 特性 | StringBuilder | StringBuffer | String.concat |
|------|--------------|--------------|--------------|
| **线程安全** | ❌ 不安全 | ✅ 安全（synchronized）| ❌ 不安全 |
| **性能** | 最高 | 较低（锁开销）| 较高（创建新对象）|
| **底层实现** | 可变 char[]（JDK 9+ 是 byte[]）| 同左 | 创建新 String 对象 |
| **空指针安全** | N/A | N/A | NPE 风险（null 参数）|

**StringBuilder 核心实现**：

```java
// JDK 8
char[] value;  // 底层 char 数组

public StringBuilder append(String str) {
    super.append(str);  // 调用 AbstractStringBuilder
    return this;
}

// AbstractStringBuilder.append
public AbstractStringBuilder append(String str) {
    if (str == null) appendNull();  // 处理 null
    int len = str.length();
    ensureCapacityInternal(count + len);  // 扩容检查
    str.getChars(0, len, value, count);   // 复制内容
    count += len;
    return this;
}
```

**JDK 9+ String 实现变更**：
- 之前：`char[] value`（每个字符 2 字节，Latin-1 浪费空间）
- 之后：`byte[] value + coder`（Latin-1 用 1 字节，UTF-16 用 2 字节）
- `StringBuilder`/`StringBuffer` 同步变更

**扩容机制**：

```java
private void newCapacity(int minCapacity) {
    int newCap = (value.length << 1) + 2;  // 原容量 * 2 + 2
    if (newCap - minCapacity < 0) newCap = minCapacity;
    value = Arrays.copyOf(value, newCap); // 数组复制，代价高
}
```
- 扩容时需要 `Arrays.copyOf()`，所以尽量 `new StringBuilder(capacity)` 预分配

**String.concat 原理**：

```java
public String concat(String str) {
    if (str == null) throw new NullPointerException();
    if (str.isEmpty()) return this;  // 空字符串直接返回自己
    char[] buf = new char[this.len + str.len];  // 每次创建新 char[]
    this.getChars(0, this.len, buf, 0);
    str.getChars(0, str.len, buf, this.len);
    return new String(buf, true);  // 每次创建新 String 对象
}
```

**字符串常量池（String Constant Pool）**：

| 区域 | 说明 |
|------|------|
| **JDK 7 之前** | 位于 PermGen（方法区），大小固定（默认 32MB），容易 OOM |
| **JDK 7** | 移到堆中（`-XX:StringTableSize` 可配置）|
| **JDK 8+** | 仍在堆中，通过 `-XX:StringTableSize=N` 配置桶数量 |

```java
// String.intern() 的作用
String s1 = new String("hello");          // 堆中新建 String，SCP 中无 "hello"
String s2 = "hello";                       // 从 SCP 返回现有对象
System.out.println(s1 == s2); // false

String s3 = new String("hello").intern();  // 主动加入 SCP
System.out.println(s3 == s2); // true
```

**编译期字符串拼接优化**：

```java
String a = "hello" + "world";  // 编译期优化 → "helloworld"（常量折叠）
String s = "hello";
String b = s + "world";       // 编译期生成 → new StringBuilder(s).append("world").toString()
                               // JDK 5+ 自动优化为 StringBuilder
```

### 追问方向
- `-XX:StringTableSize` 参数对常量池性能的影响（减少哈希冲突）
- String 的 `hashCode()` 为什么用 31 作为乘数（31 是奇数且是质数，散布效果好；JVM 对 31 的乘法有优化）
- `StringJoiner`（JDK 8+）与 `StringBuilder` 的场景区分

### 避坑提示
- 循环内不要用 `+=` 拼接字符串（JDK 5+ 每次循环生成新的 StringBuilder），要用 StringBuilder
- `StringBuilder` 不是线程安全的，高并发下如果需要线程安全且频繁拼接，用 `StringBuffer`（但现在更推荐 `ConcurrentLinkedQueue` + 流处理）
- `String.intern()` 不是万能的——大量调用会污染常量池，反而造成 OOM（JDK 7+ PermGen 移到堆后好一些，但仍有风险）

---

## 附录：面试速查表

| 主题 | 高频追问 | 必记住的关键数字 |
|------|---------|---------------|
| JVM 堆 | `-Xms=-Xmx`，TLAB 默认开启 | Eden:S0:S1 = 8:1:1 |
| G1 | Region 大小 1~32MB | MaxGCPauseMillis 默认 200ms |
| CMS | 4 阶段：初始→并发→重新→并发清除 | Old Gen 使用 68% 时触发 |
| ZGC | Color Pointers | 停顿 < 1ms |
| synchronized | 4 种锁状态升级路径 | 偏向锁延迟 4s |
| volatile | StoreLoad 屏障 | 不保证原子性 |
| AQS | state 的含义由子类定义 | CLH 双向 FIFO 队列 |
| ThreadLocal | 弱引用保护 key | 每次使用完要 remove() |
| ConcurrentHashMap | JDK 8 红黑树阈值 | TREEIFY_THRESHOLD = 8 |
| JIT | 分层编译 Tier 0~4 | CompileThreshold = 10000 |
| String | JDK 9+ byte[] 实现 | 扩容 newCap = old*2+2 |

---

> 本文档共 20 道高频面试题，涵盖 JVM 底层、并发编程、字节码机制、模块系统全栈知识。建议配合实际源码阅读（JDK 8/11/17）和 GC 日志实操练习。

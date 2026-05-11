# Java 并发编程进阶面试题

> 本文档涵盖 Java 并发编程高频面试题 20 道，结合 WMS 仓库管理系统实际场景，助你备战 senior/架构师级别面试。

---

## 1. Java 内存模型 JMM

### 题目
请描述 Java 内存模型（JMM）的结构，以及主内存与工作内存的交互机制。volatile 是如何保证可见性和有序性的？8 种内存屏障分别是什么？

### 核心答案

**JMM 结构：**
- **主内存（Main Memory）**：所有共享变量存储于主内存，物理上对应 JVM 堆中的对象实例部分。
- **工作内存（Working Memory）**：每个线程拥有独立的工作内存，存储该线程读写的共享变量副本（对应 CPU 寄存器 + L1/L2 缓存）。
- 线程对共享变量的所有操作必须在工作内存中进行，不能直接读写主内存；线程间通过主内存传递共享变量。

**8 种内存屏障（JSR-133 规范定义）：**

| 屏障类型 | 指令 | 作用 |
|---------|------|------|
| LoadLoad | `load1; LoadLoad; load2` | 确保 load1 在 load2 及后续所有 load 之前执行 |
| StoreStore | `store1; StoreStore; store2` | 确保 store1 在 store2 及后续所有 store 之前刷回主内存 |
| LoadStore | `load1; LoadStore; store2` | 确保 load1 在 store2 及后续所有 store 之前完成 |
| StoreLoad | `store1; StoreLoad; load2` | 确保 store1 刷回主内存后，load2 才能读取（最强力屏障，开销最大） |

**volatile 实现原理：**
- 写操作（`volatile write`）后插入 **StoreStore** + **StoreLoad** 屏障，保证写指令刷回主内存。
- 读操作（`volatile read`）前插入 **LoadLoad** + **LoadStore** 屏障，保证读指令强制从主内存读取最新值。
- 底层依赖 **Lock** 前缀指令（`0xF0`），在多核处理器上触发缓存一致性协议（MESI/MOESI），将其他 CPU 缓存行失效，强制从主存读取。

**可见性保证：**
- 每次读取 volatile 变量，强制从主内存读取最新值。
- 每次写入 volatile 变量，强制刷回主内存并通知其他 CPU 缓存行失效。

**有序性保证：**
- volatile 变量与程序中前后操作的指令排序受 JMM 约束（happen-before 规则），禁止指令重排序越过 volatile 读/写边界。

### 追问方向
- happen-before 规则有哪些？volatile 与 happen-before 是什么关系？
- 什么场景下 volatile 不能保证原子性（如 `i++`）？
- JMM 为什么要定义 happens-before，而不直接定义为"顺序一致性"？

### 避坑提示
- ❌ 不能说 "volatile 修饰的变量存在主内存，线程直接读写主内存"——这是误解，实际仍通过工作内存缓存。
- ❌ 误以为 volatile 完全没有性能开销——StoreLoad 屏障是现代处理器上最贵的屏障之一，高频写场景慎用。
- ✅ 正确理解：volatile 是轻量级同步机制，仅保证可见性和有序性，不保证原子性。

---

## 2. synchronized 关键字

### 题目
请描述 synchronized 的锁机制：对象头 Mark Word 的结构、偏向锁/轻量级锁/重量级锁三种状态，以及锁升级（膨胀）过程。

### 核心答案

**对象头 Mark Word（64位 JVM）：**

| 锁状态 | Mark Word（64位）存储内容 |
|-------|--------------------------|
| 无锁 | 对象 hashCode(31位) + age(4位) + biased_lock(1位) + lock(2位=01) |
| 偏向锁 | thread ptr(54位) + epoch(2位) + age(4位) + biased_lock(1位) + lock(2位=01) |
| 轻量级锁 | 指向栈中锁记录的指针(62位) + lock(2位=00) |
| 重量级锁 | 指向互斥量（monitor）指针(62位) + lock(2位=10) |
| GC 标记 | 空(62位) + lock(2位=11) |

**三种锁状态：**

1. **偏向锁（Biased Lock）**：
   - 适用场景：同一线程反复进入同一对象的同步块。
   - 原理：Mark Word 记录线程 ID，后续该线程进入同步块只需比对 thread ID，无需 CAS 操作。
   - 优点：零成本加锁。

2. **轻量级锁（Lightweight Lock）**：
   - 适用场景：多线程交替进入同步块，无实际竞争。
   - 原理：线程在栈帧中创建 Lock Record，通过 CAS 将 Mark Word 指向自己的 Lock Record。若 CAS 成功则获得锁，失败则自旋等待。
   - 优点：避免线程阻塞/唤醒的 OS 内核态开销。

3. **重量级锁（Heavyweight Lock）**：
   - 适用场景：多线程竞争同一把锁，自旋失败。
   - 原理：依赖操作系统的 Mutex 实现，线程阻塞于内核等待队列，涉及用户态到内核态的上下文切换。
   - 缺点：开销大（1000+ CPU 时钟周期）。

**锁升级（膨胀）过程：**

```
无锁 → (首次线程进入) → 偏向锁
                      ↓ (其他线程尝试获取)
                 偏向锁撤销 → 轻量级锁
                              ↓ (自旋超过阈值 / 锁竞争激烈)
                         重量级锁
```

- 偏向锁在 **compilation 期**（JIT 编译）或 **safepoint** 时批量撤销。
- 轻量级锁在 CAS 自旋失败（自旋次数超限或 CPU 核数≤2）时膨胀为重量级锁。
- **锁只升不降**：一旦膨胀为重量级锁，不会降回轻量级锁或偏向锁。

### 追问方向
- 锁对象头中的 hashCode 在有锁状态下存在哪里？（有锁时 hashCode 移至线程栈的 Lock Record）
- 为什么 Java 6 开始默认启用偏向锁延迟（biased locking delay）？
- 批量重偏向（bulk rebias）和批量撤销（bulk revoke）是什么机制？

### 避坑提示
- ❌ 误认为偏向锁在每次获取时都做 CAS——偏向锁是"记录线程 ID"，同一线程重入只需比对 ID，无需 CAS。
- ❌ 混淆"轻量级锁使用自旋"和"自适应自旋"——轻量级锁用固定次数自旋，自旋超过阈值才膨胀；自适应自旋是 Java 6+ 的优化策略，由 JVM 根据竞争程度动态调整。
- ✅ 重点理解：synchronized 的锁优化是 JVM 层面行为，与用户代码无关，底层都依赖 Monitor（重量级锁的互斥锁）。

---

## 3. synchronized vs Lock

### 题目
synchronized 和 Lock 接口有什么区别？AQS 的工作原理是什么？ReentrantLock 的公平锁和非公平锁如何实现？

### 核心答案

**synchronized vs Lock 对比：**

| 特性 | synchronized | Lock |
|------|-------------|------|
| 锁获取方式 | 隐式获取，代码块结束自动释放 | 显式 lock()/unlock() |
| 能否尝试非阻塞获取 | ❌ 不能 | ✅ `tryLock()` |
| 能否设置超时 | ❌ 不能 | ✅ `tryLock(long time, TimeUnit)` |
| 能否响应中断 | ❌ 不能 | ✅ `lockInterruptibly()` |
| 公平/非公平 | JVM 实现（不可控） | ✅ 可通过构造函数选择 |
| 多条件变量 | ❌ 只有一个隐式条件队列 | ✅ `newCondition()` 多个条件队列 |
| 底层实现 | 对象头 Monitor | AQS（AbstractQueuedSynchronizer） |

**AQS 原理（AbstractQueuedSynchronizer）：**

- **核心数据结构**：CLH 变体队列（FIFO 双向队列），存放等待获取锁的线程节点。
- **状态管理**：`volatile int state` 表示资源状态，子类通过 `compareAndSetState()` 原子修改。
- **两种模式**：
  - **独占模式（Exclusive）**：如 ReentrantLock，同一时刻只有一个线程持有锁（如 `state=1`，持有线程重入则 `state++`）。
  - **共享模式（Shared）**：如 Semaphore/CountDownLatch，多个线程可同时获取（如 `state` 表示剩余许可数）。
- **模板方法**：子类需实现 `tryAcquire()`/`tryRelease()`（独占）和 `tryAcquireShared()`/`tryReleaseShared()`（共享），AQS 负责排队和阻塞。

**ReentrantLock 公平 vs 非公平实现：**

```java
// 公平锁：hasQueuedPredecessors() 检查队列中是否有更早等待的线程
protected final boolean tryAcquire(int acquires) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false; // 公平锁直接失败，不排队
}

// 非公平锁：直接 CAS 抢锁，不管队列顺序
// 优势：减少线程切换开销，刚释放锁的线程更可能立刻重入
```

- **非公平锁**：新线程可能"插队"抢锁，适合高并发场景（减少线程阻塞唤醒开销）。
- **公平锁**：严格按等待顺序获取锁，避免饥饿（Starvation），但吞吐率低。

### 追问方向
- ReentrantLock 的可重入如何实现？（`state` 计数 + 持有线程校验）
- Condition 是如何实现的？（AQS 内部的条件队列，独立于 CLH 队列）
- 如果 Lock 不调用 unlock() 会发生什么？（线程永久阻塞——死锁风险）

### 避坑提示
- ❌ 误以为 `synchronized` 性能一定比 `Lock` 差——Java 6-11 对 synchronized 做了大量优化（偏向锁、轻量级锁、锁粗化等），低竞争场景下两者差距已不大。
- ❌ `Lock` 必须放在 `try-finally` 中确保释放——忘记 unlock() 导致死锁。
- ✅ 在 WMS 系统中，库存扣减等高并发场景建议用 `LongAdder` 或 `ConcurrentHashMap` 分段锁而非全局 synchronized/Lock。

---

## 4. CAS（Compare-And-Swap）

### 题目
什么是 CAS？它是如何实现原子操作的？ABA 问题是什么？AtomicInteger 和 AtomicReference 在使用时有什么注意事项？

### 核心答案

**CAS 原理：**
- CAS 是 CPU 提供的 **乐观锁** 指令：`Compare-And-Swap(addr, expected, newValue)`。
- 语义：若内存地址 `addr` 的值等于预期值 `expected`，则将其更新为 `newValue`，返回 `true`；否则什么都不做，返回 `false`。
- Java 中由 `sun.misc.Unsafe` 提供底层支持（Java 9+ 改为 `VarHandle`），`AtomicInteger` 等类封装为对外 API。
- 底层实现依赖 **LL/SC**（Load-Linked/Store-Conditional）或 **cmpxchg** 指令，保证读-改-写原子性。

**ABA 问题：**
- 线程 T1 读取值 A，线程 T2 将值 A 改为 B，线程 T3 又将值 B 改回 A。此时 T1 做 CAS，发现值仍是 A，误以为没有被修改过。
- ABA 问题在某些场景是致命的（如栈的 top 指针操作、链表节点删除等）。
- **解决方案**：
  - **版本号/时间戳**：`AtomicStampedReference`（维护 pair：`{value, stamp}`），每次修改 `stamp++`。
  - **AtomicMarkableReference**：维护 `{value, mark}`，适合"删除标记"等场景。

**AtomicInteger 常用方法：**
```java
incrementAndGet()  // 等价 ++i，CAS 自旋
getAndIncrement()  // 返回旧值再自增
addAndGet(delta)    // 原子加法
compareAndSet(expect, update) // CAS
```

**AtomicReference：**
```java
AtomicReference<Node> ref = new AtomicReference<>(new Node());
ref.compareAndSet(oldNode, newNode); // 替换引用，ABA 敏感
```

### 追问方向
- CAS 的开销在哪里？（CPU 总线带宽、缓存一致性协议开销）
- 为什么 AtomicInteger 的 `incrementAndGet()` 在高并发下比 synchronized 效率高？（无锁，自旋重试，但高竞争时 CPU 浪费严重）
- LongAdder vs AtomicLong 在高并发下的区别？（LongAdder 分段计数，减少热点竞争）

### 避坑提示
- ❌ 在循环中反复 CAS 但没有 sleep 或 pause——会导致 CPU 100% 空转。
- ❌ 误以为 CAS 能解决一切并发问题——高竞争场景下 CAS 失败率极高，性能反而不如锁（如 LongAdder 分段设计）。
- ✅ Java 9+ 推荐使用 `VarHandle` 替代直接 Unsafe 操作，API 更安全。

---

## 5. AQS（AbstractQueuedSynchronizer）

### 题目
请描述 AQS 的工作原理，包括独占模式和共享模式的区别，CLH 队列是如何工作的？

### 核心答案

**AQS 核心组件：**
- `volatile int state`：同步状态，由子类定义含义（如 ReentrantLock 表示重入次数，Semaphore 表示剩余许可数）。
- `Node`：等待队列节点，包含 `waitStatus`、`prev`、`next`、`thread`。
- `Node.SHARED`：共享模式标记。
- `Node.EXCLUSIVE`：独占模式标记。

**CLH 队列（Craig, Landin, Hagersten 队列）：**
- AQS 使用 CLH 变体——**双向 FIFO 队列**。
- 入队：tail 节点总是指向最后加入的 Node，新线程追加到 tail 后自旋等待前驱节点的 status。
- 出队：head 节点是已获取锁的节点，释放锁时唤醒后继节点。
- **自旋等待策略**：节点不是直接 park 阻塞，而是在前驱节点状态变化时自旋检查，减少唤醒延迟。

**独占模式（Exclusive）：**
- 同一时刻只有一个线程持有锁，如 ReentrantLock。
- `tryAcquire()`：子类实现，CAS 修改 state 成功则获取锁。
- `tryRelease()`：state 减至 0 时释放锁，唤醒后继节点。
- 典型实现：`ReentrantLock`、`ReentrantReadWriteLock.WriteLock`。

**共享模式（Shared）：**
- 多个线程可同时持有锁，如 Semaphore、CountDownLatch。
- `tryAcquireShared()`：返回负数表示获取失败，0 表示成功（不唤醒后继），正数表示成功后还有资源可唤醒后继。
- `tryReleaseShared()`：原子增加 state，唤醒后继节点。
- 典型实现：`Semaphore`、`CountDownLatch`、`CyclicBarrier`。

### 追问方向
- AQS 的 `acquireQueued()` 方法中，如果自旋等待一直不成功会怎样？（最终 park 阻塞，等待前驱节点 unpark）
- 为什么 AQS 队列是双向而不是单向的？（需要快速删除节点，尤其是取消排队的节点）
- 条件队列（Condition Queue）与 CLH 队列的关系？（Condition 独立于 CLH，通过 `await()` / `signal()` 关联）

### 避坑提示
- ❌ 自定义同步器时，`tryAcquire`/`tryRelease` 必须保证原子性和线程安全，使用 `compareAndSetState()` 而非直接赋值。
- ❌ 混淆 `releaseShared()` 和 `release()` 的使用场景——前者用于共享模式，后者用于独占模式。
- ✅ 大多数场景无需自定义 AQS，直接使用 `ReentrantLock`、`Semaphore`、`CountDownLatch` 等成熟实现。

---

## 6. ThreadLocal

### 题目
ThreadLocal 是如何实现线程隔离的？为什么会导致内存泄漏？InheritableThreadLocal 有什么特点？

### 核心答案

**ThreadLocal 原理：**
- 每个 Thread 对象内部有一个 `ThreadLocalMap`（类似 HashMap，但 Entry 继承 WeakReference）。
- `ThreadLocalMap` 的 key 是 ThreadLocal 对象（WeakReference），value 是实际存储的值。
- 线程通过 `Thread.currentThread()` 访问自己的 `ThreadLocalMap`，实现线程隔离。

**内存泄漏原因：**
```
Thread → ThreadLocalMap → Entry(key=WeakRef<ThreadLocal>, value=Object)
                              ↓
                    key 被 GC 回收后，Entry.key = null
                    但 value 仍强引用 → value 永远无法回收
                    如果 Thread 生命周期很长（线程池）→ 内存泄漏
```

- **根因**：ThreadLocalMap 的 Entry 继承 `WeakReference<ThreadLocal>`，key 被 GC 后，value 仍然被 Entry 持有。
- **正常清理机制**：`get()`/`set()` 时会调用 `expungeStaleEntry()` 清理过期的 Entry（key=null 的条目）。
- **内存泄漏场景**：ThreadLocal 用完未调用 `remove()`，且线程长期存活（线程池中的 Worker 线程）。

**InheritableThreadLocal：**
- 子线程继承父线程的 ThreadLocal 值。
- 原理：在 `Thread.init()` 时，如果父线程有 `inheritableThreadLocals`，则复制到子线程的 `inheritableThreadLocals`。
- **特点**：仅在线程创建时复制一次，后续父线程修改不会同步给子线程。
- **使用场景**：传递上下文（如 TraceId、用户信息）到子线程。
- **缺陷**：线程池场景下，子线程复用后不会重新复制，InheritableThreadLocal 意义有限。

### 追问方向
- 为什么 ThreadLocalMap 的 Entry 要用 WeakReference？（防止 ThreadLocal 对象无法被 GC——若强引用，ThreadLocal 对象即使外部无引用也因 Map 持有而无法回收）
- 既然 key 是弱引用，为什么还会内存泄漏？（value 是强引用，且非主动清理）
- 线程池中如何安全传递上下文？（Alibaba Cloud 的 `TransmittableThreadLocal`，或手动传递上下文对象）

### 避坑提示
- ❌ 在线程池中使用 InheritableThreadLocal 传递请求上下文——Worker 线程复用，上下文不会重置，导致数据串读。
- ❌ 使用 ThreadLocal 存储数据库连接——连接池复用线程时，旧连接的残留会导致严重的业务逻辑错误。
- ✅ 用完必调 `threadLocal.remove()`，尤其在线程池场景。

---

## 7. 线程池

### 题目
线程池的七大参数是什么？拒绝策略有哪些？线程池有哪些状态？实际项目中如何合理配置线程池参数？

### 核心答案

**七大参数：**
```java
public ThreadPoolExecutor(
    int corePoolSize,          // 1. 核心线程数
    int maximumPoolSize,       // 2. 最大线程数
    long keepAliveTime,        // 3. 空闲线程存活时间
    TimeUnit unit,             // 4. keepAliveTime 单位
    BlockingQueue<Runnable> workQueue,      // 5. 任务队列
    ThreadFactory threadFactory,             // 6. 线程工厂
    RejectedExecutionHandler handler        // 7. 拒绝策略
)
```

**工作流程：**
```
任务提交
    ↓
线程数 < corePoolSize → 创建核心线程执行
    ↓ (已满)
队列未满 → 进入阻塞队列等待
    ↓ (队列满)
线程数 < maximumPoolSize → 创建临时线程执行
    ↓ (已满)
执行拒绝策略
```

**拒绝策略：**

| 策略 | 行为 |
|------|------|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` |
| `CallerRunsPolicy` | 由提交任务的线程执行（调用者自行消化） |
| `DiscardPolicy` | 直接丢弃，不抛异常 |
| `DiscardOldestPolicy` | 丢弃队列最旧的任务，再尝试提交 |

**线程池状态：**

| 状态 | 值 | 说明 |
|------|---|------|
| RUNNING | `-1 << 29` | 接受新任务，执行队列中任务 |
| SHUTDOWN | `0 << 29` | 不接受新任务，但执行完队列任务 |
| STOP | `1 << 29` | 不接受新任务，不执行队列任务，中断正在执行的任务 |
| TIDYING | `2 << 29` | 所有任务终止，workerCount=0，线程转为 TIDYING 状态 |
| TERMINATED | `3 << 29` | `terminated()` 执行完毕 |

**合理参数配置：**

| 场景 | 线程数公式 | 推荐策略 |
|------|-----------|---------|
| CPU 密集型 | `CPU核心数 + 1` | 核心线程数=核数，避免任务切换 |
| IO 密集型 | `CPU核心数 × CPU利用率 × (1 + 平均等待时间/平均工作时间)` | 常用 2× 核数，或根据压测调整 |
| 混合型 | 参考 IO 密集型 + 分治 | CPU 密集任务与 IO 密集任务分离 |

- **队列选择**：高并发场景避免用 `LinkedBlockingQueue`（无界，易 OOM），推荐 `SynchronousQueue`（直接提交）或 `ArrayBlockingQueue`（有界）。
- **线程工厂**：自定义 ThreadFactory 设置有意义的线程名（便于排查问题），设置 `daemon=true`（JVM 退出时不阻塞）。

### 追问方向
- 为什么阿里巴巴 Java 规范禁止使用 Executors 创建线程池？（无界队列导致 OOM）
- `prestartAllCoreThreads()` 预启动核心线程的适用场景？
- 线程池的线程如何做到复用的？（Worker 线程从队列取任务，任务执行完后继续取，形成循环）

### 避坑提示
- ❌ 使用 `Executors.newFixedThreadPool()` 和 `Executors.newCachedThreadPool()`——前者用无界 LinkedBlockingQueue，后者线程数无上限，高并发下极易 OOM。
- ❌ 线程池参数不做压测，拍脑袋设定——常见 workers=10 导致 CPU 利用率低或 workers=100 导致线程争抢。
- ✅ WMS 系统中，入库任务、出库任务、盘点任务等应分别创建独立的线程池，避免相互影响。

---

## 8. CompletableFuture

### 题目
CompletableFuture 是什么？它如何实现异步编程？`thenApply`、`thenCompose`、`thenCombine` 的区别是什么？如何处理异常？

### 核心答案

**CompletableFuture 特性：**
- JDK 8 引入，实现了 `Future` + `CompletionStage` 接口。
- 支持 **非阻塞** 异步编程，弥补了传统 `Future` 无法链式调用、无法组合多个 Future、无法手动完成等缺陷。

**常用方法对比：**

| 方法 | 作用 | 返回类型 |
|------|------|---------|
| `thenApply(Function)` | 接收上一个结果，变换后返回新值 | `CompletionStage<R>` |
| `thenCompose(Function)` | 接收上一个结果，**返回一个新的 CompletableFuture**（扁平化） | `CompletionStage<R>` |
| `thenCombine(CompletionStage, BiFunction)` | 两个 Future 都完成后，用两边结果计算新值 | `CompletionStage<R>` |
| `thenAccept(Consumer)` | 消费结果，无返回值 | `CompletionStage<Void>` |
| `thenRun(Runnable)` | 不依赖结果，仅做收尾动作 | `CompletionStage<Void>` |

**thenApply vs thenCompose：**
```java
// thenApply: 返回值，适合同步转换
CompletableFuture<String> f1 = CompletableFuture.completedFuture(1)
    .thenApply(i -> "number:" + i); // String

// thenCompose: 返回CompletableFuture，适合异步链式调用（flatMap语义）
CompletableFuture<CompletableFuture<R>> f2 = CompletableFuture.completedFuture(1)
    .thenApply(i -> queryDb(i)); // CompletableFuture<CompletableFuture<R>> ❌
CompletableFuture<R> f3 = CompletableFuture.completedFuture(1)
    .thenCompose(i -> queryDb(i)); // CompletableFuture<R> ✅
```

**异常处理：**

```java
// 方式1：exceptionally — 捕获异常，返回默认值
future.exceptionally(ex -> {
    log.error("error", ex);
    return "default";
});

// 方式2：handle — 不管成功失败都处理
future.handle((result, ex) -> {
    if (ex != null) {
        return "error:" + ex.getMessage();
    }
    return result;
});

// 方式3：whenComplete — 不修改结果，仅做副作用
future.whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("async task failed", ex);
    }
});
```

**WMS 场景示例：**
```java
// 入库流程：校验商品 → 扣减库存 → 创建入库单 → 发送消息
CompletableFuture
    .supplyAsync(() -> inventoryService.validate(itemId))
    .thenCompose(v -> inventoryService.reserveStock(itemId, qty))
    .thenCompose(v -> inboundOrderService.create(v))
    .thenAccept(order -> mqService.send(order))
    .exceptionally(ex -> {
        log.error("入库流程异常", ex);
        return null;
    });
```

### 追问方向
- `thenApply` 是同步还是异步？（默认是 **同步**，除非用 `thenApplyAsync`）
- `allOf` vs `anyOf` 的区别？（全部完成 vs 任意一个完成）
- 线程池如何指定？（`thenApplyAsync(func, executor)` 可指定自定义线程池）

### 避坑提示
- ❌ `CompletableFuture.allOf()` 不返回合并结果——返回 `CompletableFuture<Void>`，需手动收集各结果。
- ❌ 不设置线程池，所有异步任务默认使用 `ForkJoinPool.commonPool()`——在高并发/混合业务场景下，容易与其他任务相互干扰。
- ✅ 生产环境务必传入自定义线程池，避免和业务计算线程池混用。

---

## 9. 生产者消费者模式

### 题目
生产者消费者模式有哪些实现方式？请分别用 BlockingQueue、信号量和 wait/notifyAll 实现。

### 核心答案

**方式一：BlockingQueue（最推荐）**
```java
public class ProducerConsumerBQ {
    private final BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(100);

    class Producer implements Runnable {
        public void run() {
            try {
                queue.put(produce()); // 队列满则阻塞
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    class Consumer implements Runnable {
        public void run() {
            try {
                Integer item = queue.take(); // 队列空则阻塞
                consume(item);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

**方式二：信号量（Semaphore）**
```java
public class ProducerConsumerSem {
    private final Semaphore items = new Semaphore(0);    // 已生产的资源
    private final Semaphore slots = new Semaphore(100); // 槽位数量

    void produce() throws InterruptedException {
        slots.acquire();    // 占一个槽位
        // 生产资源...
        items.release();    // 提供一个资源
    }

    void consume() throws InterruptedException {
        items.acquire();    // 消耗一个资源
        // 消费资源...
        slots.release();    // 释放一个槽位
    }
}
```

**方式三：wait/notifyAll（最原始）**
```java
public class ProducerConsumerWN {
    private final Object lock = new Object();
    private final Queue<Integer> queue = new LinkedList<>();
    private final int CAPACITY = 100;

    void produce(Integer item) throws InterruptedException {
        synchronized (lock) {
            while (queue.size() >= CAPACITY) {
                lock.wait(); // 队列满，等待消费
            }
            queue.offer(item);
            lock.notifyAll(); // 唤醒消费者
        }
    }

    void consume() throws InterruptedException {
        synchronized (lock) {
            while (queue.isEmpty()) {
                lock.wait(); // 队列空，等待生产
            }
            Integer item = queue.poll();
            lock.notifyAll(); // 唤醒生产者
            return item;
        }
    }
}
```

**WMS 场景：**
- 入库消息（生产者：扫码枪 / EDI）→ `BlockingQueue<InboundTask>` → 多个入库处理线程（消费者）。
- 出库指令（生产者：WCS 系统）→ `BlockingQueue<OutboundTask>` → 分拣线程池（消费者）。

### 追问方向
- 为什么 `while` 循环判断条件优于 `if`？（防止过早唤醒/伪唤醒——spurious wakeup）
- BlockingQueue 内部是如何实现阻塞的？（使用 ReentrantLock + Condition 的 await/signal）
- 三种方式各自适用场景？（生产/消费速率差异大 → BlockingQueue；资源总数固定 → Semaphore；需要极端自定义 → wait/notifyAll）

### 避坑提示
- ❌ `if` 判断条件下使用 `wait()`——伪唤醒会导致越界（下溢/超限）。
- ❌ 生产者通知时用 `notify()` 而非 `notifyAll()`——若多个消费者，可能只唤醒同类，导致死锁。
- ✅ 推荐使用 BlockingQueue，代码简洁，线程安全由 JDK 保证。

---

## 10. CountDownLatch vs CyclicBarrier

### 题目
CountDownLatch 和 CyclicBarrier 的区别是什么？各自的使用场景是什么？它们能否联合使用？

### 核心答案

| 特性 | CountDownLatch | CyclicBarrier |
|------|---------------|---------------|
| 概念 | 倒计时门栓，倒计数到 0 后门开 | 循环栅栏，所有线程都到达后一起放行 |
| 计数方向 | 只能 **递减**，不可重置** | 只能 **递增**（到达后重置为初始值） |
| 能否重复使用 | ❌ 不能（一次性） | ✅ 可以（Cyclic，可循环） |
| 等待线程行为 | `await()` 后阻塞，直到 count=0 | `await()` 后阻塞，直到 party=N，然后一起放行，重置 |
| 典型用途 | 主线程等待 N 个子任务完成 | N 个线程互相等待，共同推进 |

**使用场景：**

```java
// CountDownLatch：主线程等待所有子任务完成
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        processTask(i);
        latch.countDown(); // 计数-1
    }).start();
}
latch.await(); // 主线程等待
System.out.println("所有任务完成");

// CyclicBarrier：所有线程到达后同时开始下一步（模拟"多人同时起跑"）
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("都准备好了，开跑！"));
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        prepare();
        barrier.await(); // 等待其他人准备好
        run();
    }).start();
}
```

**联合使用场景（WMS 盘点流程）：**

```java
// 第一阶段：各仓库分区同时盘点（CountDownLatch 让主线程等待分区完成）
CountDownLatch phase1Latch = new CountDownLatch(N_WAREHOUSES);
for (Warehouse w : warehouses) {
    executor.submit(() -> {
        inventoryService.scanInventory(w);
        phase1Latch.countDown();
    });
}
phase1Latch.await();

// 第二阶段：汇总数据一致性校验（所有分区扫描完成后，需要所有子任务再次同步）
CyclicBarrier barrier = new CyclicBarrier(N_WAREHOUSES, () -> {
    // 所有分区扫描线程都到达后，执行汇总校验
    reportService.generateSummary();
});
```

### 追问方向
- CyclicBarrier 的 `parties` 线程中，有一个线程中断了会怎样？（CyclicBarrier broken，所有 await 抛出 BrokenBarrierException）
- CountDownLatch 能否替代 CyclicBarrier？（不能，CountDownLatch 不可重置，不能实现"所有人到达后一起放行"的语义）
- `await(long timeout, TimeUnit)` 超时后如何处理？（返回 false 或抛出 TimeoutException，需自行处理后续逻辑）

### 避坑提示
- ❌ 混淆两者的语义——"等N个任务完成"用 CountDownLatch；"等N个线程互相等待后同时开始"用 CyclicBarrier。
- ❌ CyclicBarrier 设置 parties=线程数时，确保所有线程都会调用 await()——若中途有线程不调用，会导致永久等待。
- ✅ 在 WMS 多仓库盘点场景中，CountDownLatch 适合"汇总任务等待各分区完成"，CyclicBarrier 适合"所有分区准备好后统一开始下一阶段"。

---

## 11. Semaphore

### 题目
Semaphore 的原理是什么？它如何实现限流？acquire 和 release 的配对有什么要求？

### 核心答案

**原理：**
- Semaphore（信号量）维护一个 `permit` 数量（对应 `state`）。
- `acquire()`：原子地获取一个或多个 permit，若无可用则阻塞（支持公平/非公平模式）。
- `release()`：释放一个或多个 permit，唤醒等待队列中的线程。
- 底层依赖 AQS 的共享模式实现。

**限流实现：**
```java
// 场景：数据库连接池最多 20 个连接
Semaphore dbSemaphore = new Semaphore(20);

// 获取连接
public Connection getConnection() throws InterruptedException {
    dbSemaphore.acquire();
    return connectionPool.getConnection();
}

// 归还连接
public void returnConnection(Connection conn) {
    connectionPool.returnConnection(conn);
    dbSemaphore.release();
}
```

```java
// 场景：控制接口 QPS 不超过 100
Semaphore rateLimiter = new Semaphore(100);
public void handleRequest(Request req) throws InterruptedException {
    rateLimiter.acquire();
    try {
        process(req);
    } finally {
        rateLimiter.release();
    }
}
```

**acquire/release 配对要求：**
- **必须配对**：每 `acquire()` 必须对应 `release()`，否则 permit 泄漏，最终所有线程阻塞。
- **必须在 finally 中 release**：`acquire()` 可能抛 `InterruptedException`，必须确保 release 执行。
- **可一次获取/释放多个**：`acquire(int permits)` / `release(int permits)`。

**公平 vs 非公平：**
- `Semaphore(boolean fair)`：公平模式按等待顺序分配，非公平模式允许插队。
- 高并发限流场景通常用非公平（减少线程切换开销）。

### 追问方向
- Semaphore 和阻塞队列实现限流的区别？（Semaphore 控制并发数，不存储任务；BlockingQueue 可缓存任务）
- 如何实现"令牌桶"限流？（Guava RateLimiter，或 Semaphore + 定时任务补充令牌）
- `tryAcquire()` 有什么用？（非阻塞尝试获取，获取失败立即返回 false，不阻塞）

### 避坑提示
- ❌ `acquire()` 和 `release()` 不配对——若 release 在异常分支未执行，permit 泄漏，最终死锁。
- ❌ 线程池中使用 Semaphore 限流时，子线程 release 但主线程 acquire——若 release 前主线程已超时放弃，不会重置已消耗的 permit。
- ✅ 推荐使用 `acquireUninterruptibly()` + `finally { release(); }` 模式，避免中断导致 permit 泄漏。

---

## 12. ReadWriteLock

### 题目
ReadWriteLock 是什么？读写锁分离解决了什么问题？StampedLock 相比 ReadWriteLock 有什么升级？

### 核心答案

**ReadWriteLock 接口：**
```java
public interface ReadWriteLock {
    Lock readLock();  // 读锁（共享锁）
    Lock writeLock(); // 写锁（独占锁）
}
```

**读写锁分离解决的问题：**
- 读操作之间不互斥，多线程可同时读取，提升读多写少场景的并发度。
- 写操作独占，写锁获取时阻塞所有读锁和写锁。

**ReentrantReadWriteLock 实现：**
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// 读操作（可并发）
readLock.lock();
try {
    return inventory.get(itemId); // 多线程可同时读
} finally {
    readLock.unlock();
}

// 写操作（独占）
writeLock.lock();
try {
    inventory.update(itemId, qty); // 写时阻塞所有读写
} finally {
    writeLock.unlock();
}
```

**StampedLock（Java 8+）升级点：**

| 特性 | ReadWriteLock | StampedLock |
|------|--------------|-------------|
| 读锁 | 悲观读（阻塞写） | **乐观读**（不阻塞，写入时验证） |
| 性能 | 读多写少时性能好 | 读多写少时性能更好 |
| API | readLock/writeLock | `readLock()`（悲观）/ `tryOptimisticRead()`（乐观）/ `writeLock()` |
| 可重入 | ✅ ReentrantReadWriteLock 支持 | ❌ StampedLock 不支持重入 |
| 条件变量 | ✅ 支持 `newCondition()` | ❌ 不支持 `newCondition()` |

**StampedLock 乐观读示例：**
```java
StampedLock sl = new StampedLock();
long stamp = sl.tryOptimisticRead(); // 获取乐观读戳
int value = inventory.get(itemId);   // 读取
if (!sl.validate(stamp)) {            // 检查戳是否有效（写锁是否插队）
    stamp = sl.readLock();            // 升级为悲观读
    try {
        value = inventory.get(itemId);
    } finally {
        sl.unlockRead(stamp);
    }
}
```

### 追问方向
- 读写锁的饥饿问题如何解决？（ReentrantReadWriteLock 支持"公平模式"或"非公平模式+写锁优先"）
- StampedLock 的 `writeLock()` 是否会饿死读线程？（StampedLock 支持读者和写者公平队列，或写者优先）
- 为什么 StampedLock 不支持重入？（性能考量，重入会增加复杂度）

### 避坑提示
- ❌ 读写锁中，读锁获取后，写锁等待——如果读操作耗时，会导致写线程长期饥饿（写饥饿）。
- ❌ StampedLock 的乐观读不是"免费"的——读取后必须 `validate()` 检查，若有写锁冲突需要升级为悲观读。
- ✅ WMS 库存查询（读多）配合库存更新（写少）场景，适合用 StampedLock 乐观读优化吞吐量。

---

## 13. 死锁

### 题目
死锁的必要条件是什么？如何通过 jstack 排查死锁？活锁和饥饿与死锁有什么区别？

### 核心答案

**死锁必要条件（同时满足）：**
1. **互斥条件**：资源一次只能被一个线程持有。
2. **持有并等待**：线程持有资源的同时，请求其他资源。
3. **不可抢占条件**：已持有的资源不能被强制释放，只能主动释放。
4. **循环等待条件**：存在循环链 T1→T2→...→Tn→T1。

**排查命令 jstack：**
```bash
# 1. 找到 Java 进程 PID
jps -l | grep ApplicationName
# 或
ps -ef | grep java

# 2. 导出线程快照
jstack <pid> > thread_dump.log

# 3. 在快照中搜索死锁
jstack -l <pid>
# 输出中包含 "Found 1 deadlock." 或 "Found N deadlocks."
```

**jstack 输出关键信息：**
```
"Thread-1" #16 prio=5 os_prio=0 tid=0x... nid=... waiting for monitor entry
    java.lang.Thread.State: BLOCKED
    waiting for ownable synchronizer: <0x...> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
    ...
```

**活锁 vs 死锁 vs 饥饿：**

| 类型 | 表现 | 线程状态 | 例子 |
|------|------|---------|------|
| 死锁（Deadlock） | 线程永久阻塞，互相等待对方持有的锁 | BLOCKED | T1持A等B，T2持B等A |
| 活锁（Livelock） | 线程都在运行，但因某种机制不断重复（始终无法推进） | RUNNABLE（反复重试） | 两人在走廊相遇互相让路，永远过不去 |
| 饥饿（Starvation） | 线程能运行，但永远得不到资源（被其他线程长期抢占） | RUNNABLE 或 TIMED_WAITING | 低优先级线程永远得不到 CPU 时间片 |

**解决活锁：**引入随机退让（如 Transaction 冲突时随机 sleep）；或检测重复冲突后主动让出资源。

### 追问方向
- 哲学家就餐问题中，如何避免死锁？（资源有序分配、一次性获取所有资源、检测到死锁后一个线程退让）
- 如何在代码层面避免死锁？（按固定顺序加锁；使用 `tryLock(timeout)` 替代永久等待；减少锁粒度）
- jstack 看不到死锁就一定没有吗？（某一时刻的快照可能恰好不在死锁状态，需多次 dump 观察）

### 避坑提示
- ❌ 死锁排查只依赖 jstack——有时需要结合 `jConsole` / `jmc` 的实时监控。
- ❌ 误认为"线程 BLOCKED"就是死锁——BLOCKED 状态也可能是一个线程等待另一个线程释放锁，但尚未形成循环等待。
- ✅ 最佳实践：编码规范中强制"按固定顺序获取多把锁"，从根本上消除循环等待条件。

---

## 14. 线程间通信

### 题目
Java 中线程间通信的方式有哪些？请分别说明 volatile、wait/notify、CountDownLatch、CyclicBarrier、Semaphore 在线程通信中的应用。

### 核心答案

**线程通信方式汇总：**

| 方式 | 通信原理 | 典型场景 |
|------|---------|---------|
| **volatile** | 共享变量可见性（线程间无协调，只是通知） | 标志位、开关 |
| **wait/notify** | Object 方法，synchronized 内配合使用，阻塞+唤醒 | 生产者消费者、顺序控制 |
| **await/signal** | Condition（AQS），比 wait/notify 更灵活（多条件队列） | 复杂同步场景 |
| **CountDownLatch** | 线程等待倒计时完成 | 主线程等待N个子任务 |
| **CyclicBarrier** | N个线程互相等待，到齐后同时放行 | 多阶段任务同步 |
| **Semaphore** | 控制资源数量，同时起到限流和协调作用 | 限流、资源池 |
| **PipedInputStream/OutputStream** | 管道（很少使用） | 进程内线程通信 |

**volatile 通信示例：**
```java
volatile boolean ready = false;
// 生产者
ready = true; // 写入后其他线程立即可见
// 消费者
while (!ready) { Thread.sleep(100); }
process();
```

**wait/notifyAll 示例：**
```java
synchronized (obj) {
    while (条件不满足) {
        obj.wait(); // 等待并释放锁
    }
    // 条件满足，处理
}
synchronized (obj) {
    obj.notifyAll(); // 唤醒所有等待线程，重新竞争锁
}
```

**WMS 多线程协调示例：**
```java
// 入库流程：扫码(线程1) → 质检(线程2) → 上架(线程3)，顺序执行
CountDownLatch latch1 = new CountDownLatch(1);
CountDownLatch latch2 = new CountDownLatch(1);

Thread t1 = new Thread(() -> {
    scan();
    latch1.countDown(); // 通知线程2开始
});
Thread t2 = new Thread(() -> {
    latch1.await();     // 等待扫码完成
    qualityCheck();
    latch2.countDown(); // 通知线程3开始
});
Thread t3 = new Thread(() -> {
    latch2.await();     // 等待质检完成
    shelf();
});
```

### 追问方向
- `notify()` 和 `notifyAll()` 的区别？（notify 只唤醒一个等待线程，由 JVM 决定哪个；notifyAll 唤醒所有等待线程，全部重新竞争锁）
- `wait()` 为何必须在 synchronized 内调用？（wait 释放锁，需要先持有对象 monitor）
- 多线程的 join() 算不算线程通信？（join 让父线程等待子线程执行完，本质上是 CountDownLatch 的简化版）

### 避坑提示
- ❌ `wait()` 在循环中用 `if` 判断——防止伪唤醒（spurious wakeup），必须用 while。
- ❌ volatile 只能实现"通知"，不能实现"协调"——无法让线程等待某个条件满足（busy-wait 除外）。
- ✅ 在 WMS 系统中，复杂的多阶段流程推荐用 `CountDownLatch`；需要循环同步的场景用 `CyclicBarrier`；简单信号用 `volatile boolean` 或 `AtomicBoolean`。

---

## 15. 并发容器

### 题目
Java 并发包提供了哪些并发容器？ConcurrentHashMap、ConcurrentLinkedQueue、CopyOnWriteArrayList 的实现原理和适用场景是什么？

### 核心答案

**主要并发容器分类：**

| 分类 | 容器 | 特点 |
|------|------|------|
| Map | ConcurrentHashMap | 分段锁（1.7）/ CAS+synchronized（1.8+） |
| Map | ConcurrentSkipListMap | 跳表实现，无锁，支持有序 |
| List | CopyOnWriteArrayList | 写时复制，读不加锁（最终一致） |
| List | ConcurrentLinkedQueue | 无界 FIFO 队列，基于 CAS + Node.next |
| Deque | ConcurrentLinkedDeque | 无界双端队列 |
| Queue | BlockingQueue 系列 | 有界队列，支持阻塞（生产者消费者） |
| Set | ConcurrentHashSet | 内部用 ConcurrentHashMap |
| Set | CopyOnWriteArraySet | 内部用 CopyOnWriteArrayList |

**ConcurrentHashMap 原理（Java 8+）：**

- 移除分段锁，使用 **CAS + synchronized**。
- 每个桶（bucket）独立的锁，首次 CAS 失败后升级为 synchronized 锁该桶。
- 链表长度超过 **TREEIFY_THRESHOLD=8** 时转为红黑树（查询 O(log n)）。
- 扩容时支持 **并发扩容**（多个线程一起迁移数据）。

```java
// 关键操作
putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table; ...) {
        if (tabAt(tab, i) == null) {
            // CAS 占位，无锁插入
            if (casTabAt(tab, i, null, new Node(...)))
                break;
        } else {
            // synchronized 锁住当前桶节点
            synchronized (f) {
                // 处理链表或红黑树
            }
        }
    }
}
```

**ConcurrentLinkedQueue 原理：**

- 无界 FIFO 队列，基于 **CAS + Node.next**（单向链表）。
- 入队：`tail` 节点.next CAS 设置为新节点，若失败则移动 tail 重试。
- 出队：头节点.next CAS 设置为 null，若队列空则返回 null（不阻塞）。
- 适合高并发读写，无界但注意 OOM 风险。

**CopyOnWriteArrayList 原理：**

- 写操作时对整个数组加锁（`ReentrantLock`），复制一份副本，在副本上修改，然后原子替换引用。
- 读操作完全不加锁，直接读取数组引用（读取可能读到旧数据，最终一致）。
- 适用场景：**读多写少且数据量不大**（如配置信息、黑白名单）。
- 缺点：每次写都要复制整个数组，写操作成本高。

### 追问方向
- ConcurrentHashMap 的 `size()` 准确吗？（不准确，1.8 中 size() 需要遍历每个桶，不加锁，结果是估算值）
- 为什么 CopyOnWriteArrayList 迭代器是快照而非弱一致？（迭代器持有数组引用的快照，不影响后续写操作）
- `BlockingQueue` 与 `ConcurrentLinkedQueue` 的区别？（BlockingQueue 支持阻塞的 take/put，适合生产者消费者；CLQ 不阻塞，适合异步队列）

### 避坑提示
- ❌ 在 ConcurrentHashMap 上做复合操作（如 `if (!map.containsKey(k)) map.put(k, v)`）——需要外部加锁，不能依赖原子性。
- ❌ CopyOnWriteArrayList 用于写多的场景——每次写都复制数组，高频写会导致 CPU 和内存开销极大。
- ✅ WMS 系统中，库存缓存用 ConcurrentHashMap，消息队列用 ConcurrentLinkedQueue，黑白名单用 CopyOnWriteArrayList。

---

## 16. 伪共享（False Sharing）

### 题目
什么是伪共享？为什么 CPU 缓存行会导致性能问题？@sun.misc.Contended 注解如何使用？Disruptor 是如何解决伪共享的？

### 核心答案

**什么是伪共享：**
- CPU 缓存行（Cache Line）是 CPU 读写内存的最小单位，通常为 **64 字节**。
- 同一缓存行中的多个变量，被不同 CPU 核心修改时，虽无逻辑关联，但会导致缓存行失效——一个核心写入后，其他核心的缓存行被标记为无效（Invalidate），必须重新从内存读取。
- 这就是"伪共享"——物理上不相关的数据在同一缓存行，却相互影响性能。

**伪共享示例：**
```java
// 两个不相关的计数器，在同一缓存行
class Counter {
    volatile long c1; // 线程1 修改
    volatile long c2; // 线程2 修改
}
// c1 和 c2 在同一 64 字节缓存行
// 线程1 修改 c1 → c2 对线程2 的缓存行也失效
```

**@Contended 注解（Java 8+）：**
```java
// 注解在类或字段上，让 JVM 自动添加填充（padding）
@sun.misc.Contended
class SafeCounter {
    volatile long value;
}

// 使用：添加后 value 会被填充到独立的缓存行
// 需加 JVM 参数：-XX:-RestrictContended
```

**Disruptor 解决伪共享的方案：**

Disruptor 是一个高性能队列（用于 LMAX 交易所系统），通过**缓存行填充**避免伪共享：

```java
// RingBuffer 中的 Sequence（类似指针），每个 Sequence 占 8 字节
// 为每个 Sequence 填充 7 个空 Long（56字节），确保各自独占缓存行
class Sequence {
    volatile long value = INITIAL_VALUE;
    // 填充 7 个 Long
    long p1, p2, p3, p4, p5, p6, p7; // 7 × 8 = 56 字节
    // 加上 value 的 8 字节 = 64 字节 = 1 个缓存行
}
```

```java
// Disruptor 的 RingBuffer.pad() 方法批量填充
public abstract class RingBufferPad {
    protected long p1, p2, p3, p4, p5, p6, p7; // 7 × 8 = 56 字节填充
}

// 消费者使用 Sequence 数组，每个 Sequence 独占缓存行
// 生产者使用单独的 Sequence（volatile long）
```

**效果：**高并发下，Disruptor 的吞吐量是 ArrayBlockingQueue 的 10-100 倍，核心原因之一就是消除伪共享。

### 追问方向
- 缓存行填充的代价是什么？（额外内存开销；缓存命中率降低）
- Java 9+ 的 `@Contended` 和手动填充哪个更好？（`@Contended` 是 JVM 标准，由 JVM 负责维护；手动填充不依赖 JVM 但代码侵入性大）
- 如何查看 JVM 的缓存行大小？（`java -XX:+PrintFlagsFinal -version | grep CacheLineSize`，通常是 64）

### 避坑提示
- ❌ 过度使用 `@Contended` 填充——每个字段独占缓存行，内存开销极大，仅用于高频竞争的核心字段（如 Disruptor 的 Sequence）。
- ❌ 认为只有 Long 类型才会有伪共享——任何在高频写入变量附近的变量都可能受影响（如数组元素）。
- ✅ 在 WMS 高频交易场景（订单撮合、库存扣减）中，LongAdder 的分段计数设计本身就是利用缓存行隔离减少伪共享。

---

## 17. 偏向锁撤销

### 题目
偏向锁的批量偏向/批量重偏向/撤销偏向是什么机制？为什么 safepoint 是偏向锁撤销的关键节点？

### 核心答案

**偏向锁生命周期：**
- 对象创建时，若偏向锁启用（`BiasedLockingStartupDelay=4s`），第一个线程访问同步块时，对象头记录该线程 ID。
- 偏向锁的目标：同一线程反复进入同一锁的同步块，完全消除同步开销。

**偏向锁撤销（Revoke）场景：**

1. **有线程竞争**（最常见）：
   - 线程 B 尝试获取偏向锁，发现对象头偏向线程 A → 触发 **偏向锁撤销**（BiasedLocking）。
   - 撤销过程需要 **safepoint**（所有线程到达安全点），才能修改对象头。

2. **调用 hashCode()**：
   - 对象已有 hashCode（不可偏向），偏向锁必须被撤销。
   - 无锁状态的对象头存 hashCode；偏向锁状态下 hashCode 被移至线程栈的 Lock Record。

**批量偏向/批量重偏向（Bulk Rebias）：**
- JVM 维护一个 `epoch`（纪元）字段，每个类一个偏向 epoch。
- 当一个类的对象批量撤销偏向次数过多（超过 `BiasedLockingRebulkThreshold=20`），JVM 批量重置该类的 `epoch++`，使所有对象偏向失效——新线程重新获取时变成"批量重偏向"。
- 目的：避免频繁单个撤销的性能损耗，批量处理。

**批量撤销（Bulk Revoke）：**
- 当撤销次数继续增加（超过 `BiasedLockingBulkRevokeThreshold=40`），JVM 永久禁止该类对象启用偏向锁。
- 后续所有该类新建对象直接无锁状态，偏向锁机制对该类完全失效。

**Safepoint（安全点）：**
- JVM 需要停顿所有线程（Stop-The-World）才能安全修改对象头、进行 GC 等操作。
- Safepoint 是代码中线程都能到达的一个"安全点"，通常在方法调用、循环回跳、异常抛出处。
- 线程在 Safepoint 检查自己是否需要被挂起，被挂起后 JVM 才能执行偏向锁撤销。
- **问题**：Safepoint 同步点可能在业务代码执行路径上，大量偏向锁撤销导致业务线程停顿（GC 停顿之外的人为停顿）。

### 追问方向
- 为什么 Java 15+ 默认禁用了偏向锁？（偏向锁在现代多核 CPU 上维护成本高，实际收益有限；JEP 374 废弃了偏向锁）
- 偏向锁撤销是否会导致业务线程长时间停顿？（理论上不会，Safepoint 的停顿通常很短，但高并发场景批量撤销可能导致毛刺）
- JOL（Java Object Layout）如何查看对象头？（`jol-core` 工具可打印对象内存布局）

### 避坑提示
- ❌ 在 Java 15+ 仍讨论偏向锁调优——已废弃，JVM 默认关闭偏向锁（`-XX:+UseBiasedLocking` 不再生效）。
- ❌ 混淆"撤销"和"重偏向"——撤销是退回无锁；重偏向是换一个新线程偏向。
- ✅ 现代 JVM（15+）推荐使用轻量级锁或 JUC 的 Lock，而非依赖偏向锁优化。

---

## 18. JVM 锁优化

### 题目
JVM 有哪些锁优化策略？锁粗化、锁消除、自旋锁、自适应自旋分别是什么？

### 核心答案

**锁粗化（Lock Coarsening / Lock Merging）：**
- JVM 检测到同一线程反复对同一个对象加锁（如循环内加锁），将多次锁操作合并为一次更粗粒度的锁。
- **例子**：
  ```java
  // JVM 优化前
  for (int i = 0; i < 1000; i++) {
      synchronized (lock) {
          list.add(i); // 每次循环加锁+解锁 1000 次
      }
  }
  // JVM 优化后（锁粗化）
  synchronized (lock) {
      for (int i = 0; i < 1000; i++) {
          list.add(i); // 整个循环只加锁一次
      }
  }
  ```
- 由 JIT 编译器在编译期（server 模式）自动优化。

**锁消除（Lock Elision）：**
- JIT 编译时，如果检测到锁对象只能被当前线程访问（逃逸分析证明不会逃逸），直接消除 synchronized 关键字。
- **逃逸分析**（Escape Analysis）：分析对象的动态作用域，判断是否逃逸出当前线程。
  ```java
  // JIT 优化：mergeSort 是局部方法，arr 不会逃逸
  void sort(int[] arr) {
      int[] tmp = new int[arr.length];
      synchronized (tmp) { // JIT 分析发现 tmp 不逃逸 → 消除锁
          System.arraycopy(arr, 0, tmp, 0, arr.length);
      }
  }
  ```
- `-XX:+DoEscapeAnalysis` 启用逃逸分析（JDK 6u23+ 默认开启）。

**自旋锁（Spin Lock）：**
- 轻量级锁获取失败后，不立即阻塞线程，而是让线程"自旋"（空转 busy-wait）一段时间。
- 假设锁很快会被释放，自旋避免了线程阻塞/唤醒的内核态切换开销。
- 固定自旋次数（通常 10 次），次数用完后膨胀为重量级锁。

**自适应自旋（Adaptive Spinning，Java 6+）：**
- 由 JVM 根据运行时统计信息**动态调整**自旋次数。
- 如果同一锁的上一次自旋等待成功了，说明该锁被持有的时间短，增加自旋次数。
- 如果自旋等待失败，说明锁被持有时间长，减少甚至取消自旋。
- 实现：`HotSpot` 虚拟机维护一个 `Spinning` 状态统计，JIT 编译器根据历史数据调整。

### 追问方向
- 锁粗化是否总是好的？（不一定，粗化后锁范围变大，并发度降低；若临界区包含耗时操作，可能导致其他线程长时间等待）
- 逃逸分析的开销是否值得？（逃逸分析本身有开销，若分析结果发现不逃逸，锁消除的收益才值得）
- `-XX:+UseSpinning` 参数在 Java 9+ 的变化？（Java 9+ 自旋锁行为有调整，部分场景默认启用）

### 避坑提示
- ❌ 过度依赖 JVM 锁优化而忽视代码设计——优化效果不可预期，应优先设计好锁的粒度。
- ❌ 误以为 synchronized 加了锁 JIT 一定会优化——逃逸分析、锁粗化都是基于热点代码的统计，不保证每次都生效。
- ✅ 最佳实践：代码层面控制锁粒度（缩小临界区），不要依赖 JVM 消除不合理的锁设计。

---

## 19. 分代GC与并发

### 题目
G1 和 ZGC 是如何实现并发标记的？卡表（Card Table）在并发标记中起到什么作用？分代GC和并发有什么关系？

### 核心答案

**卡表（Card Table）原理：**
- JVM 将堆内存划分为固定大小的卡片（Card），每张 Card 大小为 **512 字节**。
- Card Table 是一个字节数组（`byte[] card_of_heaps`），每个 Entry 对应堆中的一张 Card。
- 当老年代对象引用了新生代对象时，需要标记该 Card 为 **dirty**（脏卡），避免全堆扫描。
- 写屏障（Write Barrier）在每次引用赋值时执行：
  ```java
  // 赋值前：o.field = newObj
  // 写屏障：卡表[cardIndex(o)] = dirty
  ```
- **Minor GC**：只需扫描 dirty cards（少量），而非整个老年代。

**G1 的并发标记（三色标记法）：**

G1（Garbage First）使用 **SATB（Snapshot-At-The-Beginning）** 算法实现并发标记：

| 颜色 | 含义 | 说明 |
|------|------|------|
| 白（White） | 不可达对象 | 最终被回收 |
| 灰（Gray） | 已发现但未扫描其引用 | GC roots 直接引用的对象 |
| 黑（Black） | 已完成扫描 | 所有字段已处理，不会指向白对象 |

**并发标记阶段（Concurrent Marking）：**
1. **初始标记（Initial Mark）**：STW，标记 GC Roots（老年代直接引用的新生代对象），时间短。
2. **并发标记（Concurrent Mark）**：与用户线程并发执行，从灰对象向下遍历，标记所有可达对象。
3. **最终标记（Remark）**：STW，处理并发阶段漏标的对象（SATB 保证漏标不漏）。
4. **筛选回收（Cleanup）**：STW，计算各区域的存活对象比例，决定回收哪些区域。

**ZGC 的并发标记（着色指针）：**

ZGC（Z Garbage Collector）使用 **着色指针（Colored Pointers）** 和 **读屏障（Load Barrier）**，实现**全并发**（几乎无 STW）：

- 64位指针中，低 4 位（或 4+1 位）作为 **mark bit**：Finalizable、Marked0、Marked1、Remapped。
- 对象在 GC 过程中地址不变（**relocating**，而非复制），通过指针颜色判断是否需要活度。
- **GC 线程与用户线程并发**，用户线程访问对象时，读屏障检查指针颜色，若需要修正则"自愈"（对象在 GC 期间被移动后，原位置存 forwarding pointer）。
- 停顿时间控制在 **10ms 以内**（与堆大小无关）。

**分代GC与并发：**

- **分代假设**（Generational Hypothesis）：大部分对象是短命的（朝生夕死），分代GC利用这一特点减少 GC 次数和时间。
- **Minor GC**：只收集年轻代，频率高但停顿短（因为年轻代小）。
- **Major/Full GC**：老年代收集，频率低但停顿长。
- **并发收集的目标**：让 GC 的停顿（STW）不随堆大小增长而增长（如 G1、ZGC、Shenandoah）。

### 追问方向
- 三色标记法中"漏标"是如何产生的？（灰对象被回收后，其引用的白对象被其他黑对象引用，但黑对象在并发阶段不会重新扫描该白对象）
- SATB 如何保证不漏标？（快照开始时的可达对象都会被标记，即使后续变成不可达也不会漏——通过 pre-write barrier 记录旧引用）
- G1 的 Humongous 区域是什么？（存储超过 Region 50% 的大对象，单独管理，可能导致频繁 GC）

### 避坑提示
- ❌ 认为 G1 一定比 ParNew + CMS 好——G1 在小堆（< 6GB）场景下优势不明显，且 G1 的 GC 停顿是可控但不确定的。
- ❌ ZGC 不支持类卸载（JDK 15 前）——Java 15+ 才支持 ZGC 类卸载，需注意。
- ✅ WMS 系统堆内存配置建议：若 GC 停顿敏感，使用 G1（`XX:+UseG1GC`）并设置 `MaxGCPauseMillis`；若追求最低停顿使用 ZGC（`XX:+UseZGC`）。

---

## 20. 并发设计模式

### 题目
请介绍不变模式（Immutable Pattern）、生产者消费者模式（Producer-Consumer）和双重检查锁定（Double-Checked Locking）三种并发设计模式。

### 核心答案

**1. 不变模式（Immutable Pattern）：**
- 核心思想：**对象一旦创建后，状态不可修改**（所有字段 final，类 final 或所有 setter 删除）。
- 天然线程安全：无需同步，所有线程只读，无竞争。
- 典型实现：JDK 中的 `String`、`Integer`、`BigDecimal`、Guava 的 `ImmutableList`、`ImmutableMap`。

```java
// 不变对象示例
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    // 无 setter，所有字段 final
    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }
}
```

**适用场景：**配置对象、共享只读数据、多线程间的数据交换（消息对象）。

**2. 生产者消费者模式（Producer-Consumer）：**
- 核心思想：**生产者将任务放入队列，消费者从队列取任务处理**，解耦生产与消费。
- 优点：平衡生产/消费速率差异；支持批量处理；便于扩展（增减消费者）。
- 实现：`BlockingQueue`（推荐），或 `wait/notifyAll`，或 `Semaphore`。

```java
// WMS 入库场景：扫码枪（生产者）→ 队列 → 入库处理线程池（消费者）
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(500);

// 生产者（扫码线程）
public void onScan(String barcode) {
    queue.put(new Task(barcode)); // 队列满则阻塞
}

// 消费者（线程池）
@PostConstruct
public void init() {
    Executors.newFixedThreadPool(4).submit(() -> {
        while (true) {
            Task task = queue.take(); // 队列空则阻塞
            process(task);
        }
    });
}
```

**3. 双重检查锁定（Double-Checked Locking，DCL）：**
- 目的：实现**延迟初始化**（Lazy Initialization）+ **线程安全**，同时避免每次访问都加锁的性能损耗。
- 问题起源：单例模式中，`if (instance == null)` + `synchronized` 组合有性能问题（每次 `getInstance()` 都竞争锁）。

```java
public class Singleton {
    // 1. volatile 禁止指令重排序
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        // 2. 第一次检查（无锁快速路径）
        if (instance == null) {
            synchronized (Singleton.class) {
                // 3. 第二次检查（加锁后确认）
                if (instance == null) {
                    instance = new Singleton(); // 4. 问题在这里！
                }
            }
        }
        return instance;
    }
}
```

**为什么要 volatile？（Java 5+）**
- `instance = new Singleton()` 不是原子操作：
  - Step 1: 分配内存
  - Step 2: 调用构造函数
  - Step 3: 将引用赋值给 instance
  - **问题（Java 5 前）**：步骤 2 和 3 可能被 JIT 重排序，导致其他线程在 instance!=null 时拿到了未构造完成的对象。
- `volatile` 的 **StoreLoad 屏障** 禁止步骤 3 重排序到步骤 2 之前。

**DCL 适用场景：**单例对象初始化、重量级资源的延迟初始化（数据库连接池等）。

### 追问方向
- 不变模式中，如果包含集合字段如何处理？（深拷贝或返回不可变视图：`Collections.unmodifiableList(list)`）
- DCL 在 Java 5 前有哪些替代方案？（ `synchronized` 全方法、饿汉式单例、`holder` 模式——内部类持有单例）
- 生产者消费者模式中，队列满了怎么办？（`BlockingQueue.put()` 阻塞，或 `offer()` 返回 false 由生产者自行决策）

### 避坑提示
- ❌ 不变模式的对象中有可变引用（如 List、Map 字段）——必须深拷贝或返回不可变视图，否则绕过了不变性的保护。
- ❌ DCL 中忘记 `volatile`（Java 5 之前版本，或使用了旧 JMM）——可能导致线程拿到未初始化完成的对象。
- ❌ 生产者消费者模式中队列设置无界（`LinkedBlockingQueue()` 默认 Integer.MAX_VALUE）——生产者速度远快于消费者时，内存暴涨 OOM。
- ✅ 现代 Java 推荐使用枚举或 `holder` 模式替代 DCL 做单例——更简洁且绝对线程安全。

---

## 附录：WMS 仓库管理系统并发场景速查

| 业务场景 | 推荐并发工具 |
|---------|-------------|
| 库存扣减（多仓库并行入库） | `LongAdder` 或 `ConcurrentHashMap` 分段 + CAS |
| 订单消息异步处理 | `CompletableFuture` + 自定义线程池 |
| 多仓库盘点同步完成 | `CountDownLatch` |
| 多阶段入库流程 | `CyclicBarrier` |
| 数据库连接池限流 | `Semaphore` |
| 库存读写分离（读多写少） | `StampedLock` 乐观读 |
| 黑白名单配置热加载 | `CopyOnWriteArrayList` |
| 高性能日志队列 | `Disruptor`（无锁队列） |
| 多线程库存模型计算 | `Immutable` 对象 + `parallelStream()` |
| 线程池拒绝策略 | `CallerRunsPolicy`（让调用方消化，背压机制） |

---

*本文档由 Hermes Agent 自动生成，覆盖 Java 并发编程 20 大核心知识点。结合 WMS 仓库管理系统的实际场景设计面试题，帮助候选人深入理解原理，而非死记硬背。祝面试顺利！*

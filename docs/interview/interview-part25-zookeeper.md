# ZooKeeper 分布式协调面试题

## 第1题：ZooKeeper 架构

### 题目
ZooKeeper 的架构是怎样的？Leader/Follower/Observer 分别承担什么角色？ZAB 协议在其中起什么作用？

### 核心答案

**架构角色：**

| 角色 | 职责 | 数量要求 |
|------|------|----------|
| **Leader** | 写请求的唯一入口，负责事务提案（proposal）的发起和广播，维护事务更新顺序 | 最多1个 |
| **Follower** | 参与 Leader 选举，参与事务提案的投票（ACK），转发写请求给 Leader，提供读服务 | N个（>=1） |
| **Observer** | 不参与投票，只转发写请求给 Leader，提供读服务（3.3.0+引入） | N个（可选） |

**ZAB 协议作用：**
ZAB（ZooKeeper Atomic Broadcast）是 ZooKeeper 的核心一致性协议，定义了：
- 如何选举 Leader（Leader Election）
- 如何同步数据（Discovery & Synchronization）
- 如何广播事务（Broadcast）

ZAB 保证：**事务在所有 Follower 之间原子广播，且 Leader 崩溃后能通过崩溃恢复模式保证数据一致性。**

```
Client → Leader → Propose（事务提案）→ 广播到所有 Follower
                                    → Follower ACK
                                    → Leader Commit
                                    → 广播到所有 Follower Commit
```

### 追问方向
- Observer 和 Follower 的本质区别是什么？（答：Observer 不参与投票，不参与 Leader 选举，不参与事务 ACKs）
- 为什么 ZooKeeper 适合做分布式协调而非通用存储？（答：读多写少场景，CP 系统，写请求串行化）
- ZAB 和 Paxos 的核心区别是什么？（答：ZAB 有明确的 Leader 角色，专为 ZooKeeper 设计，依赖 epoch 编号）

### 避坑提示
- 不能说 Observer 也参与投票，Observer 是只读节点
- 不能混淆 ZAB 和 Raft，ZAB 的 zabEpoch 相当于 Raft 的 term，但编号方向不同
- ZooKeeper 集群节点数通常为奇数（3/5/7），偶数不推荐（脑裂风险相同但容错能力相同却不必要）

---

## 第2题：ZAB 协议详解

### 题目
详细描述 ZAB 协议的两种模式：崩溃恢复模式和消息广播模式。

### 核心答案

**消息广播模式（Normal Operation）—— 用于正常运行状态：**

```
Leader 收到写请求 (z_txn)
    ↓
生成 zxid = (epoch << 32) | counter
    ↓
发送 PROPOSAL(proposal, zxid) 给所有 Follower（含 Observer）
    ↓
Follower 收到 PROPOSAL，写事务日志，返回 ACK
    ↓
Leader 收到过半 ACK，发送 COMMIT(zxid) 给所有 Follower
    ↓
Follower 收到 COMMIT，应用事务到内存数据库
    ↓
返回客户端响应
```

类似 2PC，但优化为**过半确认即提交**，效率高于传统 2PC。

**崩溃恢复模式（Recovery）—— 用于 Leader 宕机后：**

触发条件：Leader 失去过半Follower支持、心跳超时、或自身宕机

恢复步骤：
1. **Leader Election**：所有节点进入 LOOKING 状态，各自投票，遵循 max(zxid) + max(myid) 规则选出新 Leader
2. **Discovery**：新 Leader 收集各 Follower 的最新 epoch，取最大 epoch + 1 作为新 epoch
3. **Synchronization**：新 Leader 将自身状态同步给所有 Follower（TRUNC/COPY/SNAP 机制）
4. **Broadcast**：恢复正常消息广播

**关键保证：**
- 新 Leader 的 zxid 一定是所有节点中最大的
- 被跳过的事务会被新 Leader 丢弃（TRUNC）

### 追问方向
- 如果一个 Follower 宕机后重启，发现自己的 epoch 比新 Leader 小，会发生什么？（答：TRUNC 丢弃未提交事务，同步新 Leader 数据）
- 消息广播模式为什么不采用同步复制（全部确认）？（答：ZooKeeper 追求可用性，同步复制会因单点拖慢整体可用性）
- ZAB 的崩溃恢复如何保证不丢事务？（答：依赖事务日志 + 过半确认机制，已过半确认的事务最终都会被提交）

### 避坑提示
- 崩溃恢复不是"回滚"，而是"同步"，未提交的事务可能被丢弃
- 不能说 Leader 是通过"多数派投票"选出的——实际上是自推荐的票 + 逻辑时钟比较
- zxid 是 64 位，不是简单递增整数，高 32 位是 epoch

---

## 第3题：ZooKeeper 数据模型

### 题目
ZooKeeper 的数据模型是什么？ZNode 有哪几种类型？版本号的作用是什么？

### 核心答案

**数据模型：**
ZooKeeper 将数据组织为**树形结构**（类似文件系统），每个节点叫 ZNode。

```
/app1
  /config        (持久节点)
  /workers       (持久节点)
  /workers/worker-1  (临时节点，Session 断开后自动删除)
  /tasks/task-0000000001  (顺序节点)
```

**ZNode 四种类型：**

| 类型 | 创建命令 | 生命周期 | 特点 |
|------|----------|----------|------|
| **持久节点（PERSISTENT）** | `create /path data` | 显式删除前一直存在 | 最常用 |
| **临时节点（EPHEMERAL）** | `create -e /path data` | Session 结束自动删除 | 不能有子节点，用于心跳/分布式锁 |
| **持久顺序节点（PERSISTENT_SEQUENTIAL）** | `create -s /path data` | 同持久节点 | 节点名后缀递增序号（0000000001...） |
| **临时顺序节点（EPHEMERAL_SEQUENTIAL）** | `create -e -s /path data` | 同临时节点 | 节点名后缀递增序号 |

**版本号（Version）：**
每个 ZNode 都有 `stat` 结构，记录三个版本：
- `version`（dataVersion）：数据被修改的次数
- `cversion`（cversion）：子节点被修改的次数
- `aversion`（aclVersion）：ACL 被修改的次数

版本用于**乐观锁（Compare-And-Set）**：
```
set /path data 3  # 只有 version==3 时才设置成功
```
若版本不匹配，返回 `BadVersionException`。

### 追问方向
- 临时节点为什么不能有子节点？（答：设计约束，防止 Session 过期后清理复杂度爆炸）
- 顺序节点的值域溢出怎么办？（答：最大 10 位数字，到 2147483647 后溢出，理论上不会达到）
- 临时节点的 Session 是如何被追踪的？（答：Leader 维护 SessionTracker，每个 Follower 也维护本地 session）

### 避坑提示
- 临时节点不是永久删除，是 Session 断开后由 Leader 异步删除
- 版本号从 0 开始，每修改一次 +1，不是时间戳
- ZooKeeper 不适合存储大文件（建议 < 1MB），适合存储配置和协调元数据

---

## 第4题：Watch 机制

### 题目
ZooKeeper 的 Watch 机制是什么？为什么是一次性的？有哪些事件类型？

### 核心答案

**Watch 定义：**
客户端可以在 ZNode 上设置 Watch，当该 ZNode 发生变化（创建/删除/修改/子节点变化）时，ZooKeeper 会主动通知设置了 Watch 的客户端。

**核心特性：一次性（One-time Trigger）：**
```
Watch 触发流程：
1. 客户端 getData('/config', watch=true)
2. ZNode 被修改
3. 服务器推送 WatchedEvent 给客户端
4. 该 Watch 自动失效（仅触发一次）
5. 若要继续监听，客户端需重新设置 Watch
```

**为什么要一次性？**
- 服务器不维护客户端状态，减少服务器端资源消耗
- 避免通知风暴和状态同步复杂性
- 客户端主动重新注册体现"拉模型"，更可控

**事件类型（EventType）：**

| EventType | 触发条件 |
|-----------|----------|
| `NodeCreated` | ZNode 被创建 |
| `NodeDeleted` | ZNode 被删除 |
| `NodeDataChanged` | ZNode 数据被修改 |
| `NodeChildrenChanged` | ZNode 子节点列表被修改（新增/删除，不包含修改） |
| `None` | 连接状态变化（Disconnected/Connected/Expired） |

**Watch 注册和触发位置：**

| 操作 | 能否注册 Watch | 触发的事件类型 |
|------|----------------|----------------|
| `getData()` | ✅ | `NodeDataChanged` / `NodeDeleted` |
| `getChildren()` | ✅ | `NodeChildrenChanged` / `NodeDeleted` |
| `exists()` | ✅ | 所有事件类型 |
| `create()` | ❌（无法直接） | 通过父节点 `getChildren(watch=true)` 触发 |

**回调（Watcher）：**
客户端需实现 Watcher 接口：
```java
public void process(WatchedEvent event) {
    Event.KeeperState state = event.getState();  // SyncConnected/Disconnected/etc
    EventType type = event.getType();             // 事件类型
    String path = event.getPath();                // 触发路径
}
```

### 追问方向
- Watch 事件是否会丢失？（答：客户端断连期间的事件，服务器不保留，客户端重连后通过 SyncConnected 事件感知）
- 如果先收到 NodeDeleted 再收到 NodeDataChanged 事件，会发生什么？（答：事件可能乱序，客户端需要自行判断当前状态）
- Curator 如何解决一次性 Watch 的问题？（答：Curator 的 Cache 机制会在 Watch 触发后自动重新注册，实现持续监听）

### 避坑提示
- Watch 是异步推送的，但服务器保证按事务顺序推送
- Watch 不能跨 Session 共享，每个客户端连接独立
- 设置 Watch 的操作本身也是一次网络往返（getData + watch=true 是一次请求）

---

## 第5题：分布式锁实现

### 题目
如何用 ZooKeeper 实现分布式锁？什么是羊群效应？如何用 Curator 避免？

### 核心答案

**基于临时顺序节点的分布式锁原理：**

```
锁竞争路径：/lock

1. 客户端A: create -e -s /lock/lock-  → 创建 /lock/lock-0000000001
2. 客户端B: create -e -s /lock/lock-  → 创建 /lock/lock-0000000002
3. 客户端C: create -e -s /lock/lock-  → 创建 /lock/lock-0000000003

获取锁逻辑（A拥有锁）：
- A 的序号是 1，查看 /lock/ 所有子节点
- 发现没有比 1 更小的序号 → 获取锁成功
- B 的序号是 2，发现存在 1 → 监听 1 的删除事件
- C 的序号是 3，发现存在 1 和 2 → 监听 2 的删除事件

释放锁：
- A 主动删除 /lock/lock-0000000001
- B 收到通知，发现自己是最小序号 → 获取锁
```

**核心思想：让最小序号的节点持有锁，其他节点监听比自己小的那个节点。**

**羊群效应（Herd Effect）问题：**

当大量客户端竞争同一把锁时：
```
错误实现：所有等待者都监听锁节点本身
- 锁释放 → 触发所有等待者的 Watch
- 所有等待者同时尝试创建节点
- 只有一人成功，其余全部失败重新等待
- 造成惊群效应，大量无效通知和重试
```

**正确实现（顺序节点解决羊群）：**
```
每个客户端只监听比自己序号小的那个节点：
- N1 持有锁，N2-N1000 等待
- 锁释放只通知 N2（最小序号等待者）
- 只有 N2 尝试创建，失败才通知 N3
- 每次只有 1 个客户端尝试创建
```

**Curator 分布式锁（InterProcessMutex）：**
```java
InterProcessMutex lock = new InterProcessMutex(client, "/lock");
lock.acquire(10, TimeUnit.SECONDS);  // 获取锁（可重入）
// ... 临界区操作 ...
lock.release();  // 释放锁
```

Curator recipes 还提供：
- `InterProcessReadWriteLock`：读写锁
- `InterProcessSemaphoreMutex`：不可重入互斥锁
- `MultiSharedLock`：锁组合

### 追问方向
- ZooKeeper 分布式锁是公平锁还是非公平锁？（答：公平锁，按创建顺序获取）
- 客户端在持有锁期间宕机怎么办？（答：临时节点自动删除，其他客户端感知后获取锁，锁自动续期可避免）
- 如何防止分布式锁的客户端异常退出？（答：Curator 的 `lock.acquire()` 可传入超时时间，内部实现锁续期）

### 避坑提示
- 不能用临时节点直接做锁（无法实现等待队列）
- 不能用持久顺序节点做锁（删除需要主动操作，客户端宕机无法自动释放）
- Curator 锁默认是可重入的，跨 JVM 需要使用 `InterProcessSemaphoreMutex`

---

## 第6题：Leader 选举

### 题目
ZooKeeper 的 Leader 选举算法是什么？myid、zxid、epoch 分别的作用是什么？

### 核心答案

**FastLeaderElection 算法（默认）：**

```
选举过程（假设 5 节点，myid 1-5）：

Phase 1: 发起选举
- 所有进入 LOOKING 状态的节点，发起投票 (zxid, myid, epoch)
- 广播给所有其他节点

Phase 2: 收集投票
- 节点收到投票后，按规则比较：
  1. epoch 更大 → 胜出（表示对方经历过更多 Leader 周期）
  2. epoch 相等，zxid 更大 → 胜出（表示对方数据更完整）
  3. epoch 和 zxid 相等，myid 更大 → 胜出（人为打破平局）
- 胜出的票成为新的投票，广播出去

Phase 3: 统计票数
- 节点获得过半相同投票 → 选举完成
- 若过半票投给自己 → 成为 Leader
- 否则 → 成为 Follower
```

**三个核心参数：**

| 参数 | 含义 | 作用 |
|------|------|------|
| **myid** | 节点在 `myid` 文件中配置的唯一 ID（整数） | 最终打破平局的依据（越大越优先） |
| **zxid** | 最后一次事务的 ID，(epoch << 32) \| counter | 衡量节点数据完整性的最重要指标 |
| **epoch**（或 ZabEpoch） | 当前 Leader 的任期编号 | 区分不同 Leader 周期，防止脑裂 |

**选举示例：**
```
节点状态：
- A(myid=1): zxid=100, epoch=1
- B(myid=2): zxid=99, epoch=1
- C(myid=3): zxid=100, epoch=1
- D(myid=4): zxid=98, epoch=1
- E(myid=5): zxid=100, epoch=1

选举过程：
- A、C、E 互相投票，zxid=100 相同
- A(myid=1) vs C(myid=3) → C 胜出（myid 更大）
- A 改投 C
- C 获得 4 票（A,B,D,E）→ 成为 Leader
```

**为什么 ZooKeeper 推荐 2N+1 节点部署？**
- 3 节点：容忍 1 节点故障，需要过半（2）节点同意
- 5 节点：容忍 2 节点故障，需要过半（3）节点同意
- 偶数节点（如 4）容忍 1 节点故障，需要过半（3）同意，但 4 和 5 容错能力相同，4 多浪费资源

### 追问方向
- 选举过程中 epoch 不同会发生什么？（答：epoch 小的节点会被告知更新 epoch，进入同步阶段）
- Observer 参与选举吗？（答：不参与，只参与消息广播）
- Leader 崩溃后，客户端请求会怎样？（答：客户端会收到 Disconnected 事件，自动重连到新 Leader）

### 避坑提示
- zxid 是 64 位，高 32 位是 epoch，不能和普通递增 ID 混淆
- 选举算法在网络分区时可能无法选出 Leader（没有过半节点可达），此时集群不可用
- myid 只是打破平局的最后手段，不是决定性因素

---

## 第7题：ZooKeeper 应用场景

### 题目
ZooKeeper 的典型应用场景有哪些？请举例说明配置管理、服务注册、分布式锁、命名服务的实现方式。

### 核心答案

**1. 配置管理（Configuration Management）**

```java
// 配置存储
create /config/application '{"db_url":"jdbc:mysql://...","timeout":3000}'
create /config/application/db_password 'encrypted_password'

// 配置变更监听
getData('/config/application', watch=true)

// 变更时自动通知所有订阅客户端
// 适合：开关配置、数据库连接池参数、feature toggle
```

**典型场景：**
- 配置热更新（修改 ZooKeeper 配置，客户端自动感知，无需重启）
- 动态路由（MQ 分区数、线程池大小调整）
- 黑白名单更新

**2. 服务注册与发现（Service Registry）**

```java
// 服务启动时注册
create -e /services/users/192.168.1.10:8080 '{\"weight\":1,"region":"us-east"}'
create -e /services/users/192.168.1.11:8080 '{\"weight\":1,"region":"us-east"}'

// 服务发现
getChildren('/services/users', watch=true)

// 健康检查（临时节点心跳）
// 服务宕机 → Session 断开 → 节点自动删除 → 通知消费者
```

**典型场景：**
- 微服务注册中心（替代 Eureka）
- 作业调度器的任务节点发现
- 数据库主从切换通知

**3. 分布式锁（Distributed Lock）**

见第5题详解。核心是临时顺序节点实现公平锁。

**4. 命名服务（Naming Service）**

```java
// 生成唯一 ID
create -s /idgen/taskid ''   // 返回 /idgen/taskid0000000001
create -s /idgen/taskid ''   // 返回 /idgen/taskid0000000002

// 全局唯一名称分配
// 类似 Snowflake，但 ZooKeeper 提供持久化和一致性保证
```

**扩展场景：**

**5. 分布式队列（Distributed Queue）**
```java
// 生产者
create -s /queue/task- ''   // 顺序入队

// 消费者
getChildren('/queue', watch=true)
delete(最小序号的节点)  // 出队
```

**6. 分布式屏障（Barrier）**
```java
// 设置屏障：/barrier/limit = N
// 所有节点就绪后（加入 /barrier/nodes），主节点 getChildren 达到 N 开始执行
```

### 追问方向
- ZooKeeper 做配置中心有什么优缺点？（答：优点：强一致性、Watch 通知；缺点：不适合高频读取（每次都从服务器）、存储量有限）
- 服务注册中心选 ZooKeeper 还是 Eureka？（答：Eureka 强调 AP，高可用优先；ZooKeeper 强调 CP，一致性优先；生产环境 Eureka 更常见）
- 如何用 ZooKeeper 实现分布式 Barrier 的超时控制？（答：可在节点数据中记录时间戳，定期检查超时）

### 避坑提示
- ZooKeeper 不是消息队列，不适合做数据流处理
- Watch 通知不保证送达（客户端断连期间的事件不会重发），需要客户端自行处理
- 不建议存储超过 1MB 的数据，会影响性能

---

## 第8题：一致性保证与 CAP 定理

### 题目
ZooKeeper 的一致性保证是什么？它属于 CAP 定理中的 CP 还是 AP？为什么？

### 核心答案

**CAP 定理：**
```
分布式系统最多只能同时满足以下两个特性：
- Consistency（一致性）：所有节点在同一时刻看到相同的数据
- Availability（可用性）：每个请求都能收到响应
- Partition Tolerance（分区容错）：网络分区时系统仍能运行

由于网络分区不可避免，分布式系统必须在 C 和 A 之间选择。
```

**ZooKeeper 是 CP 系统：**

```
ZooKeeper 的设计选择：
- 一致性优先：写请求必须通过 Leader 广播到过半节点才提交
- Leader 选举期间：集群不可用（无法处理写请求）
- 不保证可用性：网络分区时，可能无法达成过半同意，放弃可用性
```

**具体一致性保证：**

1. **线性一致性写（Linearizable Writes）**
   - 所有写请求必须通过 Leader 序列化
   - 不存在两个并发写被不同节点处理

2. **顺序一致性（Sequential Consistency）**
   - 来自同一个客户端的写请求，按发送顺序执行
   - 客户端看到的读，一定是 Leader 最新已提交的数据

3. **前缀一致性（Prefix Committed Reads）**
   - 读请求不会读到更旧版本的数据

4. **非强一致性读取**
   - Follower/Observer 读可能是脏读（读到上一个 Leader 的数据）
   - 可通过 `sync()` 命令强制从 Leader 读

**为什么不是 AP？**

```
场景：网络分区发生
- 分区两侧各有 2 节点（共 4 节点）
- 没有过半节点（需要 3），无法选出新 Leader
- 写请求无法完成，集群放弃可用性
- 旧 Leader 所在分区也可能无法完成写
→ 选择 CP
```

**与 etcd/Raft 的对比：**
- etcd（Rafr）：CP，Raft 同样是强一致Leader模型
- Eureka：AP，采用最终一致性，允许读到过期数据

### 追问方向
- ZooKeeper 的读为什么不是强一致的？（答：Follower 异步同步 Leader 数据，可能读到稍旧数据；可通过 `sync()` + `getData()` 实现强一致读）
- 如何判断 ZooKeeper 集群是否满足 CAP？（答：在网络分区时测试：是否停止服务、是否丢失数据）
- 为什么 ZooKeeper 不选择 AP？（答：ZooKeeper 定位是分布式协调服务，数据不一致比错误协调更严重）

### 避坑提示
- ZooKeeper 读不是强一致，混用 Follower 读时需要注意
- `sync()` 命令可以让 Follower 强制从 Leader 同步后再读
- ZooKeeper 的"一致性"不等于"强一致"，是最终一致 + 顺序保证

---

## 第9题：Session 机制

### 题目
ZooKeeper 的 Session 是如何工作的？SessionTracker 是什么？临时节点和 Session 的关系是什么？

### 核心答案

**Session 生命周期：**

```
创建连接 → Client 发送 connect 请求 → Server 分配 sessionId + timeout
    ↓
客户端心跳保持 session 活跃（发送 ping）
    ↓
Session 超时未续期 → Server 删除该 Session 创建的所有临时节点
    ↓
通知所有监听这些节点的客户端
    ↓
Session 结束
```

**Session 属性：**
- `sessionId`：全局唯一 64 位整数
- `timeout`：客户端请求的超时时间，由 Server 根据负载动态调整（默认 tickTime * 2）
- `isClosing`：Session 是否已关闭（收到 CLOSE 请求或过期）

**SessionTracker（服务端组件）：**

```
SessionTracker 维护结构：
- sessionsById: Map<Long, SessionImpl>    // sessionId → Session
- sessionsByTime: TreeMap<Long, SessionImpl>  // 按过期时间排序
- sessionSets: Map<Long, SessionImpl[]>   // 同一过期时间的 session 集合

核心功能：
1. 创建 Session（generateSessionId）
2. 心跳续期（touchSession）
3. 超时检查（ExpierThread 后台线程）
4. 清理 Session 关联的临时节点
```

**临时节点与 Session 绑定：**

```
Session 断开（主动/超时）：
    ↓
SessionTracker 标记 session 为 closed
    ↓
删除该 Session 创建的所有临时节点（EphemeralType: TERMINAL）
    ↓
触发 NodeDeleted 事件给监听这些节点的客户端
```

**重要特性：**
- 临时节点不能有子节点（Ephemeral 노드는 자식 노드를 가질 수 없다）
- Session 迁移：客户端断线重连后，原 Session 关闭，新 Session 创建
- Session 超时时间由客户端在 connect 时请求，由 Server 最终决定

### 追问方向
- 客户端在 Session 超时前重连，会复用原 Session 吗？（答：取决于重连速度，如果原 Session 还未被标记为过期，可以复用）
- Session 超时时间设置太长或太短有什么问题？（答：太长 = 故障检测慢；太短 = 网络抖动导致误判，临时节点频繁删除重建）
- Follower 上的 Session 是如何保持的？（答：Leader 维护全局 Session，Follower 只是透传心跳，最终由 Leader 判断 Session 存活）

### 避坑提示
- 临时节点不一定是立即删除，可能有延迟（异步删除）
- Session 超时不是"精确的"，是"至少这么多时间"
- ZooKeeper 不提供跨集群的 Session 共享

---

## 第10题：ZooKeeper 命令

### 题目
ZooKeeper 有哪些常用命令？zkServer.sh 和 zkCli.sh 的作用是什么？四个 letter 命令是什么？

### 核心答案

**zkServer.sh（服务端命令）：**

```bash
# 启动 ZooKeeper
./zkServer.sh start

# 停止 ZooKeeper
./zkServer.sh stop

# 查看状态（显示角色和模式）
./zkServer.sh status

# 启动前台模式（调试用）
./zkServer.sh start-foreground

# 查看帮助
./zkServer.sh
```

**zkCli.sh（客户端命令）：**

```bash
# 连接 ZooKeeper
./zkCli.sh -server 127.0.0.1:2181

# 连接指定节点
./zkCli.sh -server host1:2181,host2:2181

# 执行指定命令后退出
./zkCli.sh -server 127.0.0.1:2181 get /path

# 递归删除（含子节点）
./zkCli.sh rmr /path

# 列出所有 znode
./zkCli.sh ls /

# 客户端内命令
[zk: 127.0.0.1:2181(CONNECTED) 1] help
```

**客户端常用命令：**

```bash
# 增删改查
create /path data                    # 创建节点
create -e /path data                  # 创建临时节点
create -s /path data                  # 创建顺序节点
get /path                             # 获取数据
set /path data                        # 设置数据
delete /path                          # 删除节点（无子节点）
delete /path version                  # CAS 删除（version 匹配才删除）

# 监听
get /path watch                       # 设置 watch
ls /path watch                        # 监听子节点变化

# 状态
stat /path                            # 查看节点 stat
ls2 /path                             # ls + stat
getAcl /path                          # 获取 ACL
setAcl /path acl                      # 设置 ACL

# 四字命令（通过 telnet/nc）
```

**四个 letter 命令（Four-Letter Commands）：**

通过 `echo "command" | nc localhost 2181` 或 `telnet localhost 2181` 执行：

| 命令 | 作用 | 返回内容 |
|------|------|----------|
| **ruok** | 检查 Server 是否正在运行 | `imok`（正在运行）或无响应 |
| **stat** | Server 状态信息 | 角色、版本、连接数、延迟、队列 |
| **srvr** | Server 详细信息 | 版本、启动时间、内存、事务 |
| **srvr** | Server 详细信息 | 版本、启动时间、内存、事务 |
| **conf** | Server 配置 | tickTime/initLimit/syncLimit 等 |
| **envi** | 环境变量 | Java 环境、ZooKeeper 版本 |
| **cons** | 所有连接详情 | IP、SessionID、队列状态 |
| **crst** | 重置连接统计 | `connection stats reset` |
| **dump** | Session 和临时节点 | 存活的 sessions 和 ephemerals |
| **mntr** | 监控指标 | 吞吐量、延迟、ZooKeeper 内部统计（可喂给监控系统） |
| **wchc** | 所有 watch（按路径分组） | 路径 → 监听该路径的连接 |
| **wchp** | 所有 watch（按 session 分组） | session → 该 session 监听的路径 |
| **wchc** | 所有 watch（按连接分组） | 连接 → 监听的路径 |

### 追问方向
- `ruok` 返回 `imok` 能完全证明服务正常吗？（答：不能，只能证明 JVM 进程在运行，锁死、JVM GC 问题可能仍然存在）
- `mntr` 输出如何接入 Prometheus？（答：ZooKeeper 4-letter-word output 格式可直接解析，或使用 Jolokia JMX Exporter）
- 为什么生产环境不直接暴露四字命令？（答：安全问题，需要通过防火墙限制 2181 端口的访问）

### 避坑提示
- 四字命令通过 `zkEnv.sh` 中的配置控制是否开启（`zookeeper.admin.enableServer`）
- 3.5+ 版本引入新的 AdminServer，默认端口 8080，可通过 HTTP JSON API 替代四字命令
- `stat` 和 `srvr` 输出内容相似但不完全相同，`stat` 包含连接信息

---

## 第11题：ACL 权限控制

### 题目
ZooKeeper 的 ACL 权限控制机制是什么？scheme/id/permission 三要素是什么？有哪些常用方案？

### 核心答案

**ACL 三要素：**

```
ACL = scheme:id:permissions

1. scheme（授权方案）
2. id（身份标识）
3. permissions（权限位）
```

**权限位（Permissions）- CRUD 四种 + Admin：**

| 权限 | 位 | 说明 |
|------|-----|------|
| `CREATE` | c | 创建子节点 |
| `READ` | r | 读取节点数据和列表 |
| `WRITE` | w | 设置节点数据 |
| `DELETE` | d | 删除子节点 |
| `ADMIN` | a | 设置 ACL |

简写：`crwda` 或 `cdrwa`

**常用 Scheme：**

| Scheme | ID | 说明 | 场景 |
|--------|-----|------|------|
| **world** | `anyone` | 所有人（默认） | 公开只读数据 |
| **auth** | `user`（可不指定） | 已认证用户 | 调试/开发环境 |
| **digest** | `username:password`（Base64 编码） | 用户名密码认证 | 生产环境（常用） |
| **ip** | `192.168.1.0/24` | IP 地址或网段 | 限制特定服务器访问 |
| **x509** | `DN` | SSL 证书 DN | mTLS 环境 |

**digest 认证流程：**

```bash
# 设置 digest 权限（需要先对密码做 SHA256）
# ZooKeeper 使用 MD5(password):Digest 来标识 digest 用户

# 使用 zkCli.sh 添加 digest 用户
addauth digest user:password
setAcl /path digest:user:password:crwda

# 客户端连接后需要认证
zkCli.sh -server host:port
addauth digest user:password
get /path  # 才能成功
```

**常用配置示例：**

```bash
# 世界可读，管理员可写
setAcl /config world:anyone:r auth:admin:password:crwda

# IP 白名单
setAcl /internal ip:10.0.0.1:rwda,ip:10.0.0.2:r

# 多重 ACL
setAcl /multi digest:user1:pass1:crwd,digest:user2:pass2:r
```

### 追问方向
- ACL 能否继承？（答：每个节点独立设置 ACL，不继承父节点）
- 临时节点能否设置 ACL？（答：可以，临时节点同样支持 ACL）
- super 用户如何实现？（答：启动时通过 `-Dzookeeper.DigestAuthenticationProvider.superDigest=user:password` 配置，可绕过 ACL 检查）

### 避坑提示
- ACL 权限是叠加的，`auth:user:pass:r` + `ip:10.0.0.1:w` 意味着 user 读 + 10.0.0.1 写
- `delete` 权限只能删除子节点，不能删除节点本身（需要 `WRITE`）
- `ADMIN` 允许设置 ACL，这很危险，生产环境慎重

---

## 第12题：Observer 配置与原理

### 题目
Observer 和 Follower 的区别是什么？为什么需要 Observer？如何配置 Observer？

### 核心答案

**Follower vs Observer 核心区别：**

| 特性 | Follower | Observer |
|------|----------|----------|
| 参与 Leader 选举 | ✅ | ❌ |
| 参与事务投票（ACK） | ✅ | ❌ |
| 转发写请求给 Leader | ✅ | ✅ |
| 提供读服务 | ✅ | ✅ |
| 收到 PROPOSAL | ✅ | ❌（不参与投票） |
| 收到 COMMIT | ✅ | ✅（只接收提交） |

**为什么需要 Observer？**

```
问题：ZooKeeper 写性能受限于投票节点数量

场景：跨数据中心部署
- DC1: 3 Follower
- DC2: 2 Follower（只读请求多）
- DC3: 2 Follower（只读请求多）

如果 DC2/DC3 都是 Follower：
- 每个写请求需要 7 节点中的过半数（4）ACK
- 跨数据中心延迟高，写性能差

改为 Observer：
- DC1: 3 Follower（投票）
- DC2: 2 Observer（只同步，不投票）
- DC3: 2 Observer（只同步，不投票）
- 写请求只需要 3 个 Follower 确认，延迟降低
- Observer 收到写请求后转发给 Leader，返回同步数据
```

**配置 Observer：**

```bash
# zoo.cfg
# 方式1：在集群配置中指定
server.1=192.168.1.1:2888:3888
server.2=192.168.1.2:2888:3888
server.3=192.168.1.3:2888:3888:observer   # 标记为 observer

# 方式2：单独配置
peerType=observer

# observer 的节点配置
server.4=192.168.1.4:2888:3888:observer
```

**Observer 的数据同步：**
- Observer 通过 `OBSERVERINFO` 消息告知 Leader 自己身份
- Leader 在广播事务时不发给 Observer（跳过投票阶段）
- 但会发送 INFORM 消息给 Observer，告知事务已提交
- Observer 收到 INFORM 后应用事务

**Observer 的适用场景：**
- 跨机房部署（读多写少）
- 需要扩展读吞吐量但不想增加写延迟
- 不参与投票，不影响 Leader 选举

### 追问方向
- Observer 会收到哪些消息？（答：PROPOSAL 跳过，收到 INFORM 消息用于数据同步）
- Observer 能设置 Watch 吗？（答：可以，Observer 同样支持 Watch 机制）
- Observer 宕机会影响集群吗？（答：不影响，Observer 不参与投票）

### 避坑提示
- Observer 数量不影响系统可用性（不参与过半判断）
- Observer 仍需要完整数据副本，存储需求和 Follower 相同
- Observer 可能在 Leader 广播期间读到稍旧数据（类似 Follower）

---

## 第13题：ZooKeeper 数据目录

### 题目
ZooKeeper 的数据目录是什么？dataDir 和 dataLogDir 的区别是什么？snapshot 文件的作用是什么？

### 核心答案

**目录结构：**

```bash
# zoo.cfg 配置
dataDir=/var/lib/zookeeper
dataLogDir=/var/lib/zookeeper/logs

# dataDir 目录内容
/var/lib/zookeeper/
├── version-2/                    # 存储 snapshot（内存数据的持久化快照）
│   ├── snapshot.0000001234       # snapshot 文件（zk 数据库内容）
│   └── snapshot.0000001240
├── myid                          # 节点 ID（1/2/3...）
├── zookeeper_server.pid         # 进程 PID
├── logs/                         # dataLogDir 目录（如果分开配置）
│   └── log.0000001234            # 事务日志文件
```

**dataDir vs dataLogDir：**

| 目录 | 内容 | 作用 | 建议 |
|------|------|------|------|
| **dataDir** | snapshot 快照文件、myid | 存储内存数据库的持久化快照 | 建议使用 SSD |
| **dataLogDir** | 事务日志（write-ahead log） | 记录所有事务操作，用于崩溃恢复 | **必须使用 SSD**，性能关键 |

**事务日志（Transaction Log）：**

```
日志文件名格式：log.zxid
每个日志文件包含连续的事务记录

日志内容结构：
[offset, txnheader, txn, auth, txn, auth ...]
(offset: 文件内偏移量, txnheader: txn类型/czxid/time/version)

作用：
- 崩溃后恢复数据（重放日志）
- 正常运行时，先写日志再应用事务（write-ahead）
```

**Snapshot（快照）：**

```
快照文件名格式：snapshot.zxid

生成时机：
- Leader 生成快照（内存 dump）
- Follower 定期生成（snapCount 配置）
- 重启后加载最新快照 + 快照之后的日志

为什么需要快照？
- 日志无限增长，压缩为快照
- 加快启动时的数据恢复
```

**数据恢复流程：**

```
1. 加载最新的 snapshot（最接近但不超过 last zxid）
2. 重放 snapshot 之后的日志（追平数据）
3. 删除旧的 snapshot 和日志
```

### 追问方向
- snapshot 是阻塞生成还是异步？（答：Follower 异步生成，Leader 可以同步生成）
- 事务日志文件可以删除吗？（答：不能，崩溃恢复依赖日志；可配置自动清理策略）
- 如何监控数据目录的磁盘使用？（答：`mntr` 命令的 `zk_bytes_top_thresholds` 指标，4-letter-word `stat` 显示磁盘使用）

### 避坑提示
- 事务日志是顺序写入，必须用 SSD，HDD 会导致 fsync 延迟
- snapshot 是全量数据，不是增量
- ZooKeeper 不支持日志压缩，必须依赖 snapshot 释放空间

---

## 第14题：ZooKeeper 日志

### 题目
ZooKeeper 的事务日志机制是什么？appendFile 是什么？为什么 fsync 性能至关重要？

### 核心答案

**事务日志写入流程：**

```
写请求到达 Leader
    ↓
生成事务（txn）并序列化
    ↓
追加写入事务日志（log write）
    ↓
返回 Proposal 给 Follower
    ↓
收到过半 ACK
    ↓
提交事务（commit）
    ↓
日志追加 fsync 确认
    ↓
应用事务到内存数据库
    ↓
返回客户端
```

**关键特性：Write-Ahead Log（预写日志）**

```java
// 简化逻辑
log.write(txn);          // 先写日志
followerAck();           // 等待过半确认
log.commit();            // 标记事务已提交（fsync）
apply(txn);              // 应用到内存数据库
```

**事务日志文件结构：**

```
每个事务条目结构：
- txn header: (zxid, time, type, size)
- transaction data: (nodePath, data, acls, version)
- auth info: (auth scheme, auth data)

文件格式：Checksummed Binary（使用 Checksum 算法验证完整性）
```

**appendFile 问题（高并发写入时）：**

```
appendFile 机制：
- ZooKeeper 使用 FileChannel.append() 追加写入
- 每次 append 后调用 force（fsync）刷新到磁盘
- 高并发时，频繁 fsync 成为瓶颈

问题场景：
- fsync 耗时 10ms
- 每秒 1000 个事务
- 10ms * 1000 = 10s 延迟，无法处理

优化方向：
- 批量提交（group commit）：多个事务合并一次 fsync
- 事务日志分片（log4j RollingFileAppender）：分多个文件写
- 使用更低延迟的存储（NVMe SSD）
```

**fsync 性能影响：**

| fsync 延迟 | 对写吞吐量的影响 |
|-----------|-----------------|
| 1ms | ~1000 txn/s（单线程） |
| 10ms | ~100 txn/s（单线程） |
| 100ms | ~10 txn/s（不可接受） |

**JVM 参数调优：**

```bash
# zoo.cfg
# 事务日志刷新策略（高吞吐量场景）
# 每 snapCount 个事务或每 syncLimit ms 刷盘，默认 10000
snapCount=1000

# 预分配的日志文件大小（默认 64MB）
preAllocSize=67108864

# 使用自定义日志系统时禁用 force
forceSync=no  # 不推荐，仅用于极高性能场景（数据丢失风险）
```

### 追问方向
- ZooKeeper 如何做日志清理？（答：基于快照和 count/size/age 策略自动清理）
- forceSync=no 有什么风险？（答：机器断电可能丢失最近一批未刷盘事务）
- group commit 是如何工作的？（答：多个事务请求堆积后，一次 fsync 刷新多个日志条目）

### 避坑提示
- 事务日志的 fsync 是 ZooKeeper 写性能瓶颈的首要原因
- 不能用 NFS（网络文件系统）存储事务日志，延迟太高
- 磁盘 IO 饱和会导致 Leader 处理请求超时，触发重新选举

---

## 第15题：客户端连接

### 题目
ZooKeeper 有哪些常用客户端？ZKClient 和 Curator 有什么区别？如何监听连接状态变化？

### 核心答案

**原生客户端（ZooKeeper）：**

```java
// 原生 API（基于事件监听）
ZooKeeper zk = new ZooKeeper("host:2181", 3000, watchEvent -> {
    if (watchEvent.getState() == KeeperState.SyncConnected) {
        // 连接成功
    }
});
zk.getData("/path", true, new ByteArrayCallback(), null);
```

缺点：Watch 是一次性的，API 复杂，需要手动重注册。

**ZKClient：**

```java
// ZKClient 封装（简化 Watch，重试机制）
ZKClient zkClient = new ZKClient("host:2181", 30000);
zkClient.subscribeDataChanges("/path", new DataListener() {
    @Override
    public void handleDataChange(String path, Object data) {}
    @Override
    public void handleDataDeleted(String path) {}
});
zkClient.writeData("/path", "newData");
```

**Curator（推荐）：**

```java
// Curator 完整封装（Recipes + 重试 + 连接管理）
CuratorFramework client = CuratorFrameworkFactory.newClient(
    "host:2181",
    RetryPolicy              // 重试策略
);
client.start();

// 监听连接状态
client.getConnectionStateListenable().addListener((c, state) -> {
    switch (state) {
        case CONNECTED:      // 首次连接
        case RECONNECTED:    // 重新连接
        case SUSPENDED:      // 连接丢失
        case LOST:           // Session 失效
        case READ_ONLY:      // 只读模式
    }
});

client.close();
```

**连接状态（ConnectionState）：**

| 状态 | 含义 | 客户端行为 |
|------|------|-----------|
| `CONNECTED` | 首次连接成功 | 正常操作 |
| `SUSPENDED` | 连接暂时丢失（网络抖动） | 暂停操作，等待重连 |
| `RECONNECTED` | 重连成功 | 恢复操作，但需注意临时节点状态 |
| `LOST` | Session 过期或确认不可恢复 | 重新创建 client 或业务回滚 |
| `READ_ONLY` | 进入只读模式（配置了 readOnly） | 可继续读操作 |

**Curator 重试策略：**

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
// 初始间隔 1s，最大重试 3 次，间隔指数增长

RetryPolicy retryPolicy = new RetryNTimes(3, 1000);
// 重试 3 次，间隔 1s

RetryPolicy retryPolicy = new RetryUntilElapsed(5000, 1000);
// 5s 内重试，间隔 1s
```

### 追问方向
- 客户端断连重连后，临时节点还在吗？（答：Session 过期前重连成功则保留；Session 过期则临时节点被删除）
- Curator 的 `curator-blockUntilConnected` 有什么用？（答：阻塞直到连接建立，用于启动时的初始化等待）
- 如何避免 Curator 连接丢失导致业务问题？（答：监听 LOST 状态，业务层做降级处理）

### 避坑提示
- 不要在 Watch 回调中执行阻塞操作，会影响 ZooKeeper 客户端线程池
- 连接状态变更可能乱序（如先收到 SUSPENDED 再收到 RECONNECTED）
- Session 过期后，客户端的 Watch 需要重新注册

---

## 第16题：ZooKeeper 参数调优

### 题目
ZooKeeper 有哪些关键参数？tickTime、initLimit、syncLimit 如何调优？

### 核心答案

**核心配置参数：**

```bash
# zoo.cfg
tickTime=2000                    # ZooKeeper 基本时间单元（ms）
initLimit=10                     # Follower 启动时连接 Leader 的超时次数（tickTime * initLimit = 20s）
syncLimit=5                      # Follower 与 Leader 同步的超时次数（tickTime * syncLimit = 10s）
dataDir=/var/lib/zookeeper      # 数据目录
dataLogDir=/var/lib/zookeeper   # 日志目录（建议独立 SSD）
snapCount=100000                 # 每多少事务生成一次快照
preAllocSize=65536               # 事务日志预分配大小（KB）
```

**tickTime（基本时间单元）：**

| tickTime | 适用场景 | 说明 |
|----------|----------|------|
| 2000ms（默认） | 通用 | 平衡心跳和超时检测 |
| 3000-5000ms | 网络稳定环境 | 降低心跳频率 |
| 1000ms | 低延迟要求 | 更快的故障检测，但增加网络负载 |

**initLimit（初始化超时）：**

```
作用：Follower 启动时，连接到 Leader 并同步数据的最大等待时间

计算：initLimit * tickTime = 最大等待时间
默认值：10 * 2000 = 20000ms = 20秒

调整场景：
- 数据量大 / 快照大 → 增加 initLimit
- Follower 机器性能差 → 增加 initLimit
- 跨机房部署延迟高 → 大幅增加 initLimit
```

**syncLimit（同步超时）：**

```
作用：Follower 与 Leader 通信的最大超时时间

计算：syncLimit * tickTime = 最大等待时间
默认值：5 * 2000 = 10000ms = 10秒

调整场景：
- Leader 和 Follower 网络延迟高 → 增加 syncLimit
- syncLimit 设置过小会导致假阳性（正常同步被误判为失败）
- 过大会延迟故障检测
```

**其他关键参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `snapCount` | 100000 | 每 10 万事务生成一次快照 |
| `preAllocSize` | 65536 | 日志文件预分配 64MB |
| `maxClientCnxns` | 60 | 单 IP 最大连接数（防 DoS） |
| `autopurge.snapRetainCount` | 3 | 保留的快照数量 |
| `autopurge.purgeInterval` | 0 | 自动清理间隔（0=禁用） |
| `quorumListenOnAllIPs` | false | 是否监听所有 IP（多网卡环境） |

**生产环境推荐配置：**

```bash
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper
dataLogDir=/data/zookeeper/txlog
snapCount=50000
preAllocSize=131072        # 128MB 预分配
autopurge.snapRetainCount=10
autopurge.purgeInterval=1   # 每小时清理一次
maxClientCnxns=100
quorumListenOnAllIPs=true   # 多网卡容器环境
```

### 追问方向
- initLimit 和 syncLimit 设置过大有什么副作用？（答：延长故障检测时间，可能拖慢快速故障转移）
- 如何判断 ZooKeeper 性能瓶颈？（答：`mntr` 命令的 `zk_avg_latency` 和 `zk_outstanding_requests`）
- 容器环境下 ZooKeeper 配置有什么特殊点？（答：需要设置 `quorumListenOnAllIPs=true`，绑定所有网卡）

### 避坑提示
- tickTime 必须是 2 的倍数（内部用于计算）
- 跨机房部署时，initLimit/syncLimit 需要大幅增加
- autopurge 清理的是 snapshot 和对应日志，不清理独立的事务日志

---

## 第17题：ZooKeeper 故障恢复

### 题目
当 ZooKeeper Leader 宕机时，集群如何进行故障恢复？新 Leader 是如何选举的？未提交的事务如何处理？

### 核心答案

**故障恢复流程（以 5 节点为例）：**

```
时刻 T：Leader（myid=3）宕机

阶段 1：Leader 失效检测
- Follower 心跳超时 → 切换为 LOOKING 状态
- 其他 Follower 收到 Leader 无响应通知
- 集群进入"不可用写"状态

阶段 2：Leader 选举
- 所有节点发起投票
- 投票内容：(zxid, myid)
- 节点收集投票后，按规则比较
- 选出新 Leader（假设 myid=4 的节点）

阶段 3：数据同步（Discovery & Sync）
- 新 Leader 广播最新 epoch
- 各 Follower 汇报自己的 zxid 和 epoch
- 新 Leader 同步数据给落后节点

阶段 4：恢复正常服务
- 新 Leader 开始处理写请求
- 集群恢复正常服务
```

**Leader 选举详细算法（FastLeaderElection）：**

```java
// 投票比较逻辑（伪代码）
if (收到的 epoch > 我的 epoch) {
    更新我的 epoch
    我的票改为投给他
}
else if (收到的 epoch == 我的 epoch) {
    if (收到的 zxid > 我的 zxid) {
        我的票改为投给他
    }
    else if (收到的 zxid == 我的 zxid) {
        if (收到的 myid > 我的 myid) {
            我的票改为投给他
        }
    }
}
```

**事务恢复机制：**

```
场景：Leader 宕机前发送了 PROPOSAL，但未收到过半 ACK

情况 A：Follower 已收到 PROPOSAL 但未 COMMIT
- 这些事务被丢弃（新 Leader 的 zxid 更大）
- 新 Leader 会将自己的状态同步给这些 Follower

情况 B：Leader 已收到过半 ACK，但未发送 COMMIT
- 新 Leader 发现自己的 zxid 比某 Follower 高
- 执行 TRUNC 命令，丢弃该 Follower 的多余事务

情况 C：Leader 和 Follower 都有部分日志
- 新 Leader 取所有节点的最大 zxid 作为基准
- 落后的节点通过同步追上

核心原则：新 Leader 的数据是最完整的，任何不一致都由新 Leader 决定保留哪个版本
```

**故障恢复期间的行为：**

```
不可用窗口：
- 选举期间：集群不可写入
- 通常在 ms~s 级别（取决于网络和配置）

读服务：
- 多数情况下 Follower 仍可提供读服务（取决于是否与 Leader 失联）
- Observer 始终提供读服务

客户端感知：
- 客户端收到 Disconnected 事件
- 客户端自动尝试重连
- 重连后收到 NewLeader/Connected 事件
- 临时节点可能已失效（需要重新创建）
```

### 追问方向
- 如果 5 节点集群的 Leader 和另一个 Follower 同时宕机会怎样？（答：只剩 3 节点，无法达成过半，无法选出新 Leader，集群不可用）
- 故障恢复期间，客户端请求会怎样？（答：收到 `KeeperException.ConnectionLossException`，需要业务层重试）
- 如何缩短故障恢复时间？（答：优化 tickTime/syncLimit、使用 SSD、减少 snapshot 大小）

### 避坑提示
- ZooKeeper 的故障恢复不保证数据零丢失（CAP 的 C），可能有少量未提交的事务被丢弃
- 选举期间写请求会被拒绝，客户端需要重试
- Observer 宕机不影响集群可用性（但不建议大量 Observer 宕机）

---

## 第18题：脑裂问题

### 题目
什么是脑裂问题？ZooKeeper 如何避免脑裂？法定人数配置与隔离双写是什么？

### 核心答案

**脑裂（Split-Brain）定义：**

```
脑裂：网络分区导致集群被分割成多个独立部分
每个部分都认为自己是主（Leader）
各自独立处理请求，导致数据不一致

举例：4 节点集群被分割为 [2, 2]
- 分区 A：2 节点，选出 Leader X
- 分区 B：2 节点，选出 Leader Y
- X 和 Y 同时服务，产生不一致
```

**ZooKeeper 如何避免脑裂：**

**1. 法定人数（Quorum）机制**

```
过半同意原则：
- ZooKeeper 需要过半节点同意才能选出 Leader
- 只有过半节点可达才能处理写请求
- 4 节点被分割为 [2, 2]：没有过半，无法选 Leader，不会产生双主

正确场景：5 节点被分割为 [3, 2]
- 分区 A（3 节点）：过半，选出 Leader
- 分区 B（2 节点）：不过半，无法选出 Leader，不处理写请求
→ 只有一个主，不会脑裂
```

**2. ZAB 协议的 epoch 机制**

```
epoch（任期编号）：
- 每个新 Leader 有唯一的 epoch
- epoch 递增，旧 Leader 的提案无效
- 节点拒绝来自旧 epoch 的提案

即使出现双主：
- 主 A（epoch=5）
- 主 B（epoch=5，被旧 Leader 提升）
- epoch 相同，zxid 大的胜出
- epoch 小的节点被拒绝
```

**3. 隔离双写（Isolation）**

```
ZooKeeper 不支持同时双写：
- 写请求必须经过 Leader
- Leader 是唯一的（ZAB 保证）
- 不存在两个活跃 Leader 同时处理写请求

Follower 收到写请求会转发给 Leader：
- 不允许 Follower 直接处理写请求
- Leader 确认不可用时，Follower 停止服务
```

**为什么偶数节点不是最佳选择：**

```
4 节点场景：
- 分割为 [2, 2]
- 没有过半，无法选 Leader
- 但 [2, 2] 各自都无法服务，浪费

5 节点场景：
- 分割为 [3, 2]
- 3 过半，选出 Leader，正常服务
- 2 不过半，不选 Leader，不服务
- 相同容错能力，5 节点更优
```

### 追问方向
- ZooKeeper 脑裂和数据库脑裂有什么不同？（答：数据库脑裂通常指主从切换后出现双主；ZooKeeper 通过 ZAB epoch 机制严格避免双主）
- ZooKeeper 的网络分区策略是什么？（答：没有明确的网络分区策略，完全依赖法定人数）
- 如何监控 ZooKeeper 脑裂风险？（答：监控节点存活数、Leader 变更频率、选主延迟）

### 避坑提示
- ZooKeeper 不是完全避免脑裂，而是通过一致性协议让脑裂后只有一个主存活
- 如果配置不当（如 2 节点集群），确实可能产生问题
- 生产环境强烈建议使用 3、5、7 这样的奇数节点

---

## 第19题：ZooKeeper 替代方案

### 题目
ZooKeeper 有哪些替代方案？etcd、Consul、Redis 分布式锁各有何优缺点？

### 核心答案

**ZooKeeper vs etcd vs Consul：**

| 特性 | ZooKeeper | etcd | Consul |
|------|-----------|------|--------|
| **一致性协议** | ZAB（自定义） | Raft | Raft |
| **CAP 取向** | CP | CP | CP（可配置为 AP） |
| **存储模型** | ZNode 树 | flat KV | KV + Service Discovery |
| **语言实现** | Java | Go | Go |
| **多数据中心** | 需 Observer | 原生支持 | 原生支持 |
| **健康检查** | 无（依赖 Session） | Lease + TTL | 内置多种健康检查 |
| **认证** | ACL（复杂） | RBAC + TLS | RBAC + TLS + ACL |
| **运维复杂度** | 中等 | 低 | 低 |
| **社区活跃度** | 中等（Apache） | 高（CNCF） | 高（HashiCorp） |

**etcd 特点：**
```
优点：
- Raft 协议更通用，文档丰富
- Kubernetes 原生使用 etcd
- gRPC API，支持范围查询
- 更好的多数据中心支持

缺点：
- 不支持 Watch 通配符
- 不支持临时节点（用 Lease 实现类似功能）
- Java 客户端不如 Curator 成熟
```

**Consul 特点：**
```
优点：
- 服务发现 + 健康检查 + KV 一体化
- 内置 DNS 接口
- 友好 Web UI
- Raft 一致性保证

缺点：
- KV 功能不如 ZooKeeper 强大
- 不适合做分布式锁（虽然有，但不如 Curator/ZooKeeper 成熟）
- 分布式锁有 session 概念，但实现不如 ZooKeeper 完善
```

**Redis 分布式锁：**

```java
// Redis 分布式锁实现（Redisson）
RLock lock = redisson.getLock("myLock");
lock.lock(10, TimeUnit.SECONDS);  // 获取锁
try {
    // 临界区
} finally {
    lock.unlock();
}
```

**Redis vs ZooKeeper 分布式锁对比：**

| 维度 | ZooKeeper | Redis |
|------|-----------|-------|
| **可靠性** | 高（CAP 优先一致性） | 中等（CP 配置下牺牲可用性） |
| **实现复杂度** | 简单（临时顺序节点） | 中等（需要考虑 GC/时钟/超时） |
| **性能** | 中等（网络往返多） | 高（纯内存操作） |
| **锁丢失风险** | Session 断开自动释放 | EXPIRE 可能失败（需 Lua 脚本保证） |
| **公平锁** | 原生支持（顺序节点） | 需要额外逻辑 |
| **可重入** | 原生支持 | Redisson 支持 |
| **Redlock** | 不支持 | 官方提供（多 Redis 实例） |

**Redlock 争议：**
```
Redlock = 5 个独立 Redis 实例，获取过半锁才算成功
争议点：
- Martin Fowler：分布式锁需要强一致性，不推荐 Redlock
- Redis 作者 antirez：Redlock 是合理的工程权衡
- 本质：Redlock 是 AP 方案，不是 CP
```

**选型建议：**

```
选 ZooKeeper：
- 已有 ZooKeeper 基础设施
- 需要强一致性的分布式协调
- 微服务注册中心（CP 优先）

选 etcd：
- Kubernetes 环境
- 需要高性能 KV + 强一致性
- 已有 gRPC 客户端

选 Consul：
- 需要服务发现 + 健康检查 + KV
- 喜欢 Consul UI
- 微服务架构中做配置中心

选 Redis：
- 追求极致性能
- 锁粒度粗（秒级容忍）
- 已有 Redis 基础设施
```

### 追问方向
- 为什么 Kubernetes 选择 etcd 而不是 ZooKeeper？（答：etcd 使用 Raft 更简单，Go 实现与 Kubernetes 统一，snapshot 机制更高效）
- Redis 分布式锁的常见陷阱是什么？（答：SETNX + EXPIRE 不是原子操作；锁续期问题；主从切换丢锁问题）
- ZooKeeper 在云原生时代还有优势吗？（答：在需要强一致性和成熟分布式锁的场景仍有优势，但 etcd/Consul 在云原生生态中更流行）

### 避坑提示
- Redis 锁不等于 ZooKeeper 锁，Redis 锁在极端情况下可能不安全
- 不要混用多个分布式锁实现（容易产生不可预期行为）
- Consul 的 KV 功能适合配置中心，但不适合做分布式锁（功能较弱）

---

## 第20题：Curator 框架

### 题目
Curator 框架的模块结构是什么？Framework、Client、Recipes、Utilities 分别是做什么的？

### 核心答案

**Curator 架构四层模块：**

```
┌─────────────────────────────────────────┐
│           Curator Recipes               │  ← 高级封装（分布式锁、Leader 选举等）
├─────────────────────────────────────────┤
│           Curator Framework             │  ← 核心客户端封装（连接、重试、Watch）
├─────────────────────────────────────────┤
│           Curator Client                 │  ← 低级 ZooKeeper 客户端封装
├─────────────────────────────────────────┤
│      ZooKeeper (原生)                    │  ← Apache ZooKeeper
└─────────────────────────────────────────┘
```

**1. Curator Client（底层封装）：**

```java
// CuratorClient 封装了原生 ZooKeeper 客户端
// 主要是 ConnectionStateManager 和 Session 管理
// 不推荐直接使用，用 CuratorFramework 代替
```

**2. Curator Framework（核心）：**

```java
CuratorFrameworkFactory.newClient(
    connectionString,    // ZooKeeper 地址
    retryPolicy,         // 重试策略
    sessionTimeout,      // Session 超时
    connectionTimeout,   // 连接超时
    new DefaultACLProvider()
);

CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("localhost:2181")
    .sessionTimeoutMs(60000)
    .connectionTimeoutMs(15000)
    .retryPolicy(new ExponentialBackoffRetry(1000, 3))
    .namespace("myapp")  // 命名空间（所有路径自动加上前缀）
    .build();

client.start();
// 使用 builder 模式构建，更灵活
```

**Framework 核心特性：**
- 自动重连（重试 + Session 恢复）
- 命名空间支持（隔离不同业务）
- 连接状态管理（ConnectionStateManager）
- 线程安全操作

**3. Curator Recipes（分布式协调recipes）：**

```java
// 常用 Recipes 列表

// 分布式锁
InterProcessMutex          // 可重入互斥锁
InterProcessSemaphoreMutex // 不可重入互斥锁
InterProcessReadWriteLock  // 读写锁

// Leader 选举
LeaderLatch                // 单次 Leader 选举（退出后放弃）
LeaderSelector             // 可重复参与选举

// 分布式队列
DistributedQueue           // FIFO 队列
DistributedPriorityQueue   // 优先级队列
DistributedDelayQueue      // 延迟队列
BoundedDistributedQueue    // 有界队列

// 分布式计数器
SharedCount                // 共享计数器
DistributedAtomicLong      // 分布式原子长整型

// 分布式 Barrier
DistributedDoubleBarrier   // 双栅栏

// Cache（监听封装）
PathChildrenCache          // 监听子节点变化
NodeCache                   // 监听节点本身
TreeCache                   // 监听整个子树
```

**Curator Cache（解决一次性 Watch 问题）：**

```java
// PathChildrenCache 示例
PathChildrenCache cache = new PathChildrenCache(client, "/config", true);
cache.start();

cache.getListenable().addListener((c, event) -> {
    switch (event.getType()) {
        case CHILD_ADDED:      // 子节点添加
        case CHILD_UPDATED:    // 子节点更新
        case CHILD_REMOVED:    // 子节点删除
    }
});

// 自动重新注册 Watch，持续监听
```

**4. Curator Utilities（工具类）：**

```java
// ZKPaths
ZKPaths.create(client, "/parent/child");      // 路径拼接
ZKPaths.getNodeAndParents(path, true);        // 获取父节点路径

// PathUtils
PathUtils.validatePath("/valid/path");       // 路径合法性校验

// ConcurrentQueueUtils
// 队列工具

// EnsurePath
EnsurePath ensurePath = client.newEnsurePath("/must/exist");
ensurePath.ensure(client.getZookeeperClient());  // 确保路径存在
```

**版本兼容：**

```
Curator 5.x  → ZooKeeper 3.6+
Curator 4.x  → ZooKeeper 3.5+
Curator 3.x  → ZooKeeper 3.4+

注意：ZooKeeper 3.5+ 引入了静态成员（Observer）和 AdminServer
Curator 版本需要匹配才能使用新特性
```

### 追问方向
- Curator 的 recipes 是线程安全的吗？（答：Recipes 本身是线程安全的，但同一个 path 使用多个 recipes 可能冲突）
- Curator 的 namespace 和 ZooKeeper 的 chroot 有什么区别？（答：Curator namespace 是客户端视角的前缀，chroot 是服务端路径前缀，作用相同）
- Curator 如何处理 Session 过期？（答：Curator 会自动重建 session 和重新创建 ephemeral 节点，但需要业务层处理 LOST 状态）

### 避坑提示
- Curator recipes 内部实现依赖 Watch + 重试，可能有竞态条件
- `LeaderSelector` 释放后会自动重新参与选举，需要显式调用 `close()`
- `PathChildrenCache.start()` 需要在所有操作前调用，否则可能丢失事件

---

## 参考资料

- ZooKeeper 官方文档：https://zookeeper.apache.org/doc/current/
- Curator 官方文档：https://curator.apache.org/
- 《ZooKeeper: Distributed Process Coordination》
- ZAB 协议论文：Zab: A Simple Totally Ordered Broadcast Protocol

---

*本面试题整理自 ZooKeeper 核心技术知识，覆盖架构、协议、数据模型、应用场景、运维调优等核心领域。*

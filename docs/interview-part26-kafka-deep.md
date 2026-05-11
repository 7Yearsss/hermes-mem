# Kafka 深入原理面试题（20道高频）

---

## 1. Kafka 高性能原因：顺序写、页缓存、零拷贝

### 题目
Kafka 为什么能做到这么高的吞吐量？其底层 I/O 优化机制是什么？

### 核心答案

**1. 顺序写磁盘**
- Kafka 的消息写入采用**顺序追加**（Sequential Append）模式，写入磁盘时磁头无需寻道，近似于内存随机读写的速度（实测可达 600MB/s+）。
- 每个 Partition 对应一个物理文件，消息追加到文件末尾，避免随机写入。

**2. 页缓存（Page Cache）**
- Kafka 依赖操作系统页缓存机制，写入时数据先进入 Page Cache（由 OS 管理的一块内存区域），由 OS 异步刷盘。
- 读取时，若数据仍在 Page Cache 中（Cache Hit），直接从内存返回，绕过磁盘。
- Kafka 的 `sendfile` 系统调用（零拷贝）直接从 Page Cache 发送数据到 Socket，**不经过用户态**，减少 2 次 CPU 拷贝和 2 次上下文切换。

**3. 零拷贝（sendfile）**
传统数据发送路径（4次拷贝、4次上下文切换）：
```
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket缓冲区 → 网卡
```
`sendfile` 路径（2次拷贝、2次上下文切换）：
```
磁盘 → 内核缓冲区（Page Cache）→ 网卡
```
数据直接由 DMA 引擎从 Page Cache 拷贝到网卡，绕过了用户态。

### 追问方向
- Page Cache 与 Kafka 自身 Buffer 有何重叠/取舍？
- `log.flush.interval.messages` 与 OS 刷盘策略的关系是什么？
- 机械硬盘 vs SSD 场景下，顺序写优势是否依然成立？

### 避坑提示
- 很多人误以为 Kafka 自己实现了缓存，实际完全依赖 OS Page Cache。**机器内存大小直接影响 Kafka 缓存效果**，内存过小会导致频繁 Cache Miss。
- `sendfile` 只在消费者读取时生效，生产者写入**不是零拷贝**。

---

## 2. Kafka 分区分配策略：Range / RoundRobin / StickyAssignor

### 题目
Kafka 有哪几种分区分配策略？各自有什么优缺点？

### 核心答案

**1. RangeAssignor（默认）**
- 按 Topic维度分配：将每个 Topic 的 partitions 按范围分配给消费者。
- 例如 3 个 partition、2 个消费者：C0 分 P0,P1，C1 分 P2。
- **问题**：多个 Topic 时，消费者可能分配到的分区数不均匀。

**2. RoundRobinAssignor**
- 将所有 Topic 的 partition 混合排序后轮询分配。
- 例如 Consumer C0、C1，partition 按 hash 排序后轮询。
- 比 Range 更均匀，但若消费者订阅不同 Topic 组，可能出现分配不均。

**3. StickyAssignor（0.11+）**
- 核心目标：**分配均匀 + 分配过程尽量保持粘性**（已有绑定关系不变则不动）。
- Rebalance 时尽量少移动 partition，减少 Rebalance 带来的消息乱序和重复消费风险。
- 生产者也受益：同一分区的消息尽量发送到同一 Broker，提升批处理效率。

### 追问方向
- 自定义 Assignor 如何实现？`ConsumerPartitionAssignor` 接口核心方法？
- StickyAssignor 在 Rebalance 时具体如何减少 partition 移动？
- 多消费者组场景下，分区分配是否相互独立？

### 避坑提示
- 面试时容易混淆：RoundRobin 的均匀是跨 Topic 维度的，而 Range 是单 Topic 维度。**如果只有一个 Topic，二者分配结果相同**。
- StickyAssignor 的"粘性"主要体现在 **Rebalance 时尽量不移动**，而不是"固定分配不变"。

---

## 3. Kafka 消息压缩：算法对比与压缩时机

### 题目
Kafka 支持哪些压缩算法？各自特点是什么？消息在什么时候被压缩？

### 核心答案

| 算法   | 压缩比    | 速度   | CPU 开销 | 典型场景               |
|--------|-----------|--------|----------|------------------------|
| gzip   | 最高      | 最慢   | 中       | 离线、冷数据           |
| snappy | 中等      | 最快   | 低       | 实时消息、低延迟优先   |
| lz4    | 中等偏上  | 很快   | 低       | 高速网络、低 CPU 优先   |
| zstd   | 很高      | 快     | 中高     | 跨数据中心、带宽敏感   |

**压缩时机（Producer 端）**
- Kafka 压缩发生在 **Producer 端批次（batch）发送时**，而非单条消息。
- 多条消息组成 batch 后统一压缩，压缩比更高（消息体越小，压缩收益越明显）。
- Broker 收到后**不解压**直接存储，原样存入日志文件，只有消费者拉取时才会解压。

**关键参数**
- `compression.type`：配置压缩算法（`producer` 表示复用 Producer 配置）。
- `linger.ms`：batch 等待时间，越长 batch 越大，压缩收益越高。

### 追问方向
- Broker 端是否会二次压缩？
- 消息体很大（如 JSON 超过 10KB）时，还需要压缩吗？
- 压缩后索引文件（index）是否也压缩？

### 避坑提示
- 压缩是 **Producer 行为**，Broker 只存储，不解压也不重新压缩。** Broker CPU 不是压缩瓶颈**。
- 消息 key 不参与压缩，只压缩 value。因此 key 重复率高时压缩比会下降。

---

## 4. Kafka Exactly Once 语义：幂等生产者 + 事务消息

### 题目
Kafka 如何实现 Exactly Once 语义？有哪几种配置方式？

### 核心答案

**1. 幂等生产者（Idempotent Producer）**
- `enable.idempotence=true` 开启。
- 核心原理：**每条消息带一个 ProducerID + SequenceNumber**，Broker 端去重。
- 同一个 Producer 发送的相同消息，Broker 只会写入一次。
- 只能保证**单 Producer 单 Partition** 的幂等，无法跨 Producer 或跨 Partition 去重。

**2. 事务消息（Transactional Producer）**
- `transactional.id` 配置开启。
- 支持**多 Partition 跨 Topic 的原子性写入**，要么全部成功，要么全部失败。
- 依赖日志头部标记（Transaction Marker）实现，消费者可见性受 `isolation.level` 控制：
  - `read_uncommitted`：未提交事务也能读取
  - `read_committed`：只读取已提交事务的消息

**3. Exactly Once 配置组合**
```properties
enable.idempotence=true
acks=all
retries=Integer.MAX_VALUE
transactional.id=xxx   # 需保证唯一性
```
配合消费者 **手动 offset 提交**（`enable.auto.commit=false`），实现端到端 Exactly Once。

### 追问方向
- 幂等生产者的 ProducerID 是如何生成的？和 transaction.id 是什么关系？
- 事务消息对消费者可见性有何影响？Pending 的消息会阻塞消费吗？
- EOS 对 Throughput 的影响（事务开销）？

### 避坑提示
- **幂等生产者 ≠ Exactly Once**。幂等只能防止单 Producer 重复写入，无法防止消费者重复消费。
- `transactional.id` 和 `enable.idempotence` 是独立的，可以单独开启幂等但不开事务。
- 消费者必须配合**手动提交 offset** 才能实现真正的端到端 Exactly Once。

---

## 5. Kafka 消费组重平衡（Rebalance）：触发条件与 STABLE 策略

### 题目
什么情况下会触发 Rebalance？STABLE 策略是什么？消费者加入/离开时的行为？

### 核心答案

**触发条件**
1. 消费者加入（`consumer.join`）或离开（`consumer.leave`，主动 close 或超时）
2. 分区数量变化（Admin 操作或 Broker 宕机导致 Preferred Leader 切换）
3. 订阅 Topic 数量变化
4. 消费者心跳超时（`session.timeout.ms` 内未收到心跳）

**Rebalance 流程（GroupCoordinator 主导）**
1. JoinGroup：所有消费者向 Coordinator 发送 JoinGroup 请求
2. Leader 选举：第一个加入的消费者成为 Group Leader（负责分配方案）
3. SyncGroup：Leader 制定分配方案后通过 Coordinator 同步给所有消费者
4. Heartbeat：正常运行时通过心跳维持 group 成员关系

**STABLE 策略（0.11+）**
- 核心：减少 Rebalance 频率，保护消费者。
- 若消费者在 `rebalance.timeout.ms` 内重新加入，Coordinator 会尝试复用之前的 partition 分配（不重新分配）。
- 避免因瞬时网络抖动或 GC 停顿导致的短暂失联就触发完整 Rebalance。

### 追问方向
- Rebalance 过程中消费者能消费消息吗？
- Consumer Leader 和 Kafka Controller 是同一个概念吗？
- JoinGroup 和 SyncGroup 的超时参数是什么？设置不当会怎样？

### 避坑提示
- **Rebalance 期间整个消费者组暂停消费**，这是 Kafka 的设计缺陷之一（已通过 Static Membership 优化）。
- GC 停顿导致心跳超时是最常见的 Rebalance 原因之一，`session.timeout.ms` 不要设置过短。
- 消费者数量过多时，JoinGroup 阶段的并发压力会集中在 Coordinator 上，可能引发"惊群效应"。

---

## 6. Kafka 控制器（Controller）：选举与 Partition Leader 选举

### 题目
Kafka Controller 是什么？如何选举？Partition Leader 选举和 Controller 有什么关系？

### 核心答案

**Controller 职责**
- 集群中**唯一**的 Controller，负责：
  1. Partition Leader 选举（当 Leader 宕机时）
  2. 分区状态变更管理（创建、删除、复制因子变更）
  3. Broker 元数据管理（Broker 加入/离开）
  4. Topic 配置管理

**Controller 选举（ZooKeeper / KRaft）**
- 依赖 ZooKeeper 的 **临时节点（Ephemeral Node）** 机制：第一个成功创建 `/controller` 节点的 Broker 成为 Controller。
- 节点宕机或会话超时，临时节点消失，其他 Broker 竞争创建。
- **ZooKeeper Watch 机制**：所有 Broker 监听 Controller 变更事件，快速感知并参与新的选举。

**Partition Leader 选举流程**
1. Controller 收到 `LeaderAndIsrRequest`（Leader 宕机通知）
2. 从 **ISR（In-Sync Replicas）** 列表中选出新的 Leader
3. 选举策略：`leader.election.algorithm`（默认：` unclean.leader.election.enable=false` 时只从 ISR 选举）
4. 写入 ZooKeeper，更新 `/brokers/topics/[topic]/partitions/[partition]/state`
5. 向所有 Broker 发送 `LeaderAndIsrResponse`

### 追问方向
- 什么是 Unclean Leader Election？为什么默认关闭？
- KRaft 模式下 Controller 的选举逻辑和 ZooKeeper 模式有何不同？
- Controller 单点故障的风险如何控制？

### 避坑提示
- Kafka 0.11 后 Controller 选举**不依赖 ZooKeeper 的 Leader**，而是利用其临时节点机制，**每个 Broker 都可以成为 Controller**。
- Controller 选举时写入 ZooKeeper 的操作是**顺序写**，压力不大，但 Watch 通知风暴是常见隐患。

---

## 7. Kafka 日志存储：Segment / Index / Offset索引 / 时间戳索引

### 题目
Kafka 的日志文件结构是什么？Segment、Index、Offset 索引、时间戳索引各自作用？

### 核心答案

**Segment 结构（每个 Partition 目录下的物理文件）**
```
00000000000000000000.log      # 消息数据（主要存储）
00000000000000000000.index    # Offset 稀疏索引
00000000000000000000.timeindex # 时间戳稀疏索引
00000000000000000000.txnindex  # 事务索引（仅事务消息有）
```

**日志段配置**
- `log.segment.bytes`：单个 segment 文件大小（默认 1GB）
- `log.roll.ms/hours`：时间滚动策略

**Offset 索引（`.index`）**
- 稀疏索引（Sparse Index），每 **4KB** 索引数据建立一个映射条目。
- 存储格式：`(relativeOffset → physicalPosition)` — relativeOffset 是相对起始 offset。
- 通过二分查找快速定位消息物理位置。

**时间戳索引（`.timeindex`）**
- 存储格式：`(timestamp → relativeOffset)`。
- 支持按时间查找消息，例如 `FetchRequest` 带 timestamp 条件。

**消息格式（Log Append Time / Create Time）**
- `message.timestamp.type` 配置：CreateTime（默认）或 LogAppendTime。
- 消息物理结构包含：Offset、Timestamp、Headers、Key、Value、压缩信息。

### 追问方向
- Segment 大小对查询性能有何影响？
- Index 文件损坏或稀疏索引 miss 时如何处理？
- 日志压缩（Log Compaction）和日志删除的区别是什么？

### 避坑提示
- `.index` 是**稀疏索引**，不是每条消息都有对应索引项。查找时可能需要顺序扫描一小段。
- Segment 滚动**同时基于大小和时间**，满足任一条件即滚动新 Segment。
- 删除过期数据默认按时间策略（`retention.ms`），而非按 offset。

---

## 8. Kafka 副本机制：ISR / OSR / AR / HW / LEO

### 题目
Kafka 副本的核心概念：ISR、OSR、AR、HW、LEO 是什么？副本同步流程？

### 核心答案

**核心概念**
- **AR（Assigned Replicas）**：分区的所有副本集合，包含所有 Broker 上的副本。
- **ISR（In-Sync Replicas）**：与 Leader 保持**同步**的副本集合（滞后 <= `replica.lag.max.messages` 且心跳正常）。
- **OSR（Out-of-Sync Replicas）**：与 Leader 失去同步的副本集合。
- **HW（High Watermark）**：已提交消息的最高水位，消费者只能读到 HW 之前的数据。
- **LEO（Log End Offset）**：副本日志末位 offset，表示下一条待写入消息的位置。

**同步流程**
1. Producer 发送消息到 Leader，Leader 写入本地 Log。
2. Leader 异步复制到 ISR 中的 Follower（FetchRequest/FetchResponse 轮询）。
3. Follower 收到消息后写入本地 Log，并返回 ACK。
4. 当 ISR 中所有副本都写入后，Leader 更新 HW。
5. 消费者只能消费到 HW 位置。

**HW 更新机制**
- `HW = min(各副本 LEO)`，确保消费者不会读到未完成复制的消息。
- Follower 宕机恢复后，从 HW 截断（Truncate）超出部分，再重新同步。

### 追问方向
- ISR 收缩的条件是什么？副本 lag 超过阈值后何时出 ISR？
- Follower 复制是同步还是异步？可配置吗？
- OSR 中的副本如何重新加入 ISR？

### 避坑提示
- ISR 是**动态**的，Follower 慢或 Broker 宕机会导致 ISR 收缩。
- **HW 是消费者视角的提交水位**，不是存储水位——Leader 的 LEO 通常 > HW。
- OSR 不参与 Leader 选举，但数据仍保留（直到 Log Segment 过期）。

---

## 9. Kafka 分区扩容：数据重分配与分区迁移

### 题目
Kafka 分区扩容时，现有数据如何重分配？分区迁移的流程是什么？

### 核心答案

**分区扩容（Admin 操作）**
```bash
kafka-topics.sh --alter --topic my-topic --partitions 6
```
- 新增分区**立即生效**，旧数据**不会自动迁移**到新分区。
- 已有数据只存在于旧分区，新消息会写入新分区。
- **负载不均衡**：新增分区所在 Broker 负载偏低。

**分区重分配（Data Migration）**
- 使用 `kafka-reassign-partitions.sh` 工具，分为三个阶段：
  1. **生成方案**（`--generate`）：指定新 Broker 列表，生成迁移计划 JSON。
  2. **执行迁移**（`--execute`）：启动分区副本重分配。
  3. **验证完成**（`--verify`）：确认所有副本已完成同步。

**迁移原理**
- Controller 在 ZooKeeper 中写入 `Reassignment` 节点。
- 新 Broker 开始从 Leader 复制数据（副本间同步）。
- 迁移完成后，旧 Broker 上的多余副本被删除。
- 期间分区仍可正常读写，**不阻塞服务**。

**负载均衡**
- `kafka-reassign-partitions.sh` 配合 `bootstrap.servers`，可指定分区到不同 Broker 以均衡负载。
- 副本因子变更（`--generation`）也走同一套重分配流程。

### 追问方向
- 分区数量可以减少吗？减少后数据如何处理？
- 迁移过程中消息顺序是否受影响？
- 如何控制迁移速度以避免对线上服务造成压力？

### 避坑提示
- 分区数量**只能增不能减**（Kafka 不支持减少分区数）。
- 大量分区迁移期间，**网络和磁盘 I/O 压力会显著上升**，建议通过 `throttle` 参数限速。
- 迁移完成后务必 `--verify`，避免部分分区停留在迁移中间状态。

---

## 10. Kafka 消费者 Offset 管理：自动提交 vs 手动提交

### 题目
消费者 offset 提交有哪几种策略？各自适用场景是什么？

### 核心答案

**自动提交（默认）**
```properties
enable.auto.commit=true
auto.commit.interval.ms=5000  # 每5秒提交一次
```
- 后台线程周期性提交上一次 `poll()` 拉取的最大 offset。
- **可能重复消费**：提交周期内崩溃，已处理消息未被提交。

**手动提交（需要）**
```java
// 同步手动提交
consumer.commitSync();

// 异步手动提交（不阻塞消费）
consumer.commitAsync();

// 指定 offset 提交
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("topic", 0), 
            new OffsetAndMetadata(100, "metadata"));
consumer.commitSync(offsets);
```
- 手动提交完全由业务控制时机，**常配合业务事务使用**。
- 同步提交会阻塞消费线程；异步提交吞吐更高但存在提交失败重试问题。

**提交语义**
- 提交 offset N 表示：**已成功消费到 offset N-1**，消费将从 N 继续。
- **At Least Once**：`enable.auto.commit=false` + 业务处理完再手动提交。
- **At Most Once**：`enable.auto.commit=true` + 处理前提交（高风险，不推荐）。

### 追问方向
- `commitSync` 和 `commitAsync` 混合使用会有什么问题？
- 消费者多线程并发处理消息时，手动 offset 提交的线程安全如何保证？
- Rebalance 前后 offset 提交的行为是什么？

### 避坑提示
- 自动提交 **无法做到 Exactly Once**，只能实现 At Least Once。
- 手动提交时，`commitSync` 在 Rebalance 之前调用可能失败，需在 `ConsumerRebalanceListener` 中处理。
- 手动提交**不能提交不连续的 offset**——必须按 offset 大小顺序递增提交。

---

## 11. Kafka 消息追回：消费者重启后从哪继续消费？seek 方法

### 题目
Kafka 消费者重启后从哪个位置继续消费？如何手动控制消费位置？

### 核心答案

**消费位置确定规则**
1. **新消费组**（无 committed offset）：从 `auto.offset.reset` 配置决定：
   - `earliest`：从 earliest offset 开始消费
   - `latest`（默认）：从最新消息开始消费
2. **已有消费组**（有 committed offset）：从 **上次提交的 offset 继续消费**。
3. **offset 过期**：若提交的 offset 已被 Broker 删除（超出 retention），行为等同于新消费组。

**seek 方法（手动定位）**
```java
// 定位到指定 offset
consumer.seek(topicPartition, offset);

// 定位到最早 offset
consumer.seekToBeginning(TopicPartition...);

// 定位到最新 offset（最新消息之后）
consumer.seekToEnd(TopicPartition...);

// 结合时间戳定位（需使用 offsetsForTimes）
Map<TopicPartition, Long> queryTimestamps = new HashMap<>();
queryTimestamps.put(tp, timestamp);
Map<TopicPartition, OffsetAndTimestamp> results = 
    consumer.offsetsForTimes(queryTimestamps);
consumer.seek(tp, results.get(tp).offset());
```

**典型应用场景**
- 消费失败后回退 N 条重试：`currentPosition - N`
- 消息时间戳断点续传：故障恢复时从特定时间点继续
- 数据回溯分析：从头消费历史数据

### 追问方向
- `seekToBeginning` 是绝对的最早 offset 吗？Segment 删除后呢？
- 消费者没有 poll 之前可以调用 seek 吗？
- `offsetsForTimes` 的时间精度是什么？

### 避坑提示
- `seekToBeginning` 和 `seekToEnd` 是**异步**的，返回后 offset 可能尚未更新，需配合 `poll()` 确认。
- offset 提交和 seek 是两个独立机制——**seek 后不会自动提交**，下次 poll 才生效。
- 如果 seek 的 offset 超出 retention 范围，会抛出 `OffsetOutOfRangeException`。

---

## 12. Kafka Streams：DSL API / 状态存储 / 窗口计算

### 题目
Kafka Streams 的核心 API 是什么？状态存储和窗口计算如何实现？

### 核心答案

**DSL API（Domain Specific Language）**
- 基于**函数式编程**风格，核心抽象：
  - `KStream`：每条记录独立处理的无状态流
  - `KTable`/`GlobalKTable`：带状态的 changelog 流，支持 upsert
  - `KGroupedStream`：分组后的流

**常用 DSL 操作**
```java
// 流处理
KStream<String, Long> wordCounts = builder.stream("input-topic")
    .flatMapValues(text -> Arrays.asList(text.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word)
    .count(Materialized.as("word-counts-store"))
    .toStream();

// 表处理
KTable<String, String> table = builder.table("user-topic");
```

**状态存储（State Store）**
- **本地状态**（Kafka Streams 自动管理）：
  - `Materialized.as("store-name")` 创建本地 RocksDB 或内存状态存储
  - 故障恢复：状态变更写入 changelog topic（`@SuppressWarnings`）
- **WindowedKTable**：时间窗口状态，如 `SessionWindowedKTable`
- 支持**近实时查询**：通过 `QueryableStoreInterface` 从外部查询状态

**窗口计算**
- **滚动窗口（Tumbling Window）**：固定大小、不重叠
- **滑动窗口（Hopping Window）**：固定大小、可重叠
- **会话窗口（Session Window）**：基于活跃事件的动态窗口
```java
// 5分钟滚动窗口统计
TimeWindowedKStream<String, Long> windowed = 
    stream.groupByKey().windowedBy(TimeWindows.of(Duration.ofMinutes(5)));
```

### 追问方向
- Kafka Streams 的状态存储如何保证容错？changelog 的开销？
- Kafka Streams 的 standby 副本机制是什么？
- Processor API 和 DSL API 的关系和适用场景？

### 避坑提示
- Kafka Streams 是**客户端库**，每个实例自己管理状态——没有独立服务器进程。
- 状态存储默认是 **RocksDB**（磁盘），不是内存，生产环境需配置 RocksDB 参数。
- 窗口计算的 `Grace Period` 必须设置，否则 late event 被丢弃会报错。

---

## 13. Kafka Connect：Source/Sink Connector / Connect API / 转换器

### 题目
Kafka Connect 的作用是什么？Source/Sink Connector 的区别？转换器如何工作？

### 核心答案

**架构角色**
- Kafka Connect 是 **Kafka 提供的框架**，用于在 Kafka 和外部系统之间批量复制数据。
- 核心组件：
  - **Connect Worker**：运行 Connector 的进程（Standalone / Distributed 模式）
  - **Connector**：管理 Task 的生命周期，决定数据分区
  - **Task**：实际执行数据读写
  - **Converter**：序列化/反序列化（Kafka 内部用 bytes）
  - **Transform**：数据转换（可选，字段过滤、路由等）

**Source Connector**
- 从外部系统（数据库、文件系统）读取数据，发送到 Kafka Topic。
- 典型：`DebeziumSourceConnector`（CDC）、`JdbcSourceConnector`

**Sink Connector**
- 从 Kafka Topic 消费数据，写入外部系统（数据仓库、HDFS）。
- 典型：`JdbcSinkConnector`、`HdfsSinkConnector`、`ElasticsearchSinkConnector`

**Connect API 核心接口**
```java
public interface Connector {
    void start(Map<String, String> props);      // 初始化
    void stop();                                 // 清理
    List<Task> spinupTask();                    // 启动 Task
    void onPartitionsAssigned(Collection<TopicPartition> partitions); // 分区分配
}
```

**转换器（Converter）**
- 将外部数据格式 ↔ Kafka 内部 bytes 格式。
- 内置：`JsonConverter`、`AvroConverter`、`StringConverter`、`ProtobufConverter`。
- 配合 Schema Registry 使用（Confluent Schema Registry）保证 Schema 版本兼容。

### 追问方向
- Connector 和 Task 的关系？一个 Connector 可以启动多少 Task？
- Connect 的 Exactly Once 是如何实现的？
- Offset 管理在 Connect 中是如何工作的？

### 避坑提示
- Kafka Connect **不是应用程序**，是**独立服务**，需要单独部署和配置。
- Transform 是在 Converter 之后执行的，**不能改变消息的 key/value 类型**。
- Distributed 模式下，Connector 配置存储在 Kafka Topic 中（`__consumer_offsets` 类似），重启后自动恢复。

---

## 14. Kafka 安全：SSL/SASL 认证 + ACL 权限控制

### 题目
Kafka 的安全机制有哪些？SSL、SASL、ACL 分别解决什么问题？

### 核心答案

**1. SSL（传输加密）**
```properties
listeners=SSL://host:9093
ssl.keystore.location=/path/keystore.jks
ssl.keystore.password=xxx
ssl.key.password=xxx
ssl.truststore.location=/path/truststore.jks
ssl.client.auth=required  # 双向认证
```
- 加密 Broker ↔ 客户端、Broker ↔ Broker 的通信。
- `client.auth=required` 启用双向认证（mTLS）。

**2. SASL（身份认证）**
常见机制：
- **SASL/PLAIN**：用户名密码（静态），适用于简单场景
- **SASL/SCRAM-SHA-256/512**：用户名密码（动态盐值），生产推荐
- **SASL/OAUTHBEARER**：基于 Token，适合 Kerberos 替代方案
- **SASL/GSSAPI**：对接 Kerberos

配置示例（SCRAM）：
```properties
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
```

**3. ACL（权限控制）**
```bash
# 授权
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Read --topic test-topic

# 查看
kafka-acls.sh --list ...
```
- 细粒度控制：Topic/Group/TransactionalId 的 Read/Write/Create/Delete 等操作。
- 默认 **Deny All**，显式 Allow 才可访问。
- `super.users=User:admin` 可绕过 ACL。

### 追问方向
- SSL 和 SASL 可以同时启用吗？
- Broker 间通信（Internal）需要单独配置安全吗？
- ACL 的动态更新如何在不重启 Broker 的情况下生效？

### 避坑提示
- **SSL 加密会带来性能损耗**（CPU 加解密），高吞吐场景建议用 SASL（认证开销低于 SSL）。
- ACL 权限检查在每次请求时发生，**Kafka 不缓存 ACL 结果**（通过 Watch 感知变更）。
- ZooKeeper 模式下 ACL 信息存在 ZooKeeper；KRaft 模式下存在 Kafka 自身日志。

---

## 15. Kafka 监控：JMX 指标 / Prometheus Exporter / 消费者 LAG 监控

### 题目
Kafka 有哪些核心监控指标？如何接入 Prometheus？消费者 LAG 如何监控？

### 核心答案

**JMX 核心指标**
- **Broker 级别**：
  - `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec`（消息速率）
  - `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec`（入带宽）
  - `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent`（网络繁忙度）
  - `kafka.log:type=LogFlushManager,name=LogFlushRateAndTimeMs`（刷盘延迟）

- **Controller 级别**：
  - `kafka.controller:type=KafkaController,name=OfflinePartitionsCount`（离线分区数）
  - `kafka.controller:name=ActiveControllerCount`（当前 Controller 数量，>1 则脑裂）

- **消费者 LAG**：
  - `consumer_lag`：`kafka.consumer:type=consumer-fetch-manager-metrics,client-id=xxx,name=records-lag-max`

**Prometheus Exporter**
```yaml
# jmx_exporter 配置（采集 Broker JMX）
jmx-url: "service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi"
```

**Kafka Exporter（消费者 LAG 专用）**
```bash
kafka_exporter --kafka.server=localhost:9092 --web.listen-address=":9308"
```
- 拉取 `consumergroup` 的 lag 并暴露为 Prometheus 指标：
  - `kafka_consumer_group_lag`
  - `kafka_consumer_group_current_offset`
  - `kafka_consumer_group_log_end_offset`

**Grafana Dashboard 关键面板**
- Producer/Consumer 吞吐量
- 消费者 LAG（按 Topic、Partition、Group 维度）
- Controller 状态
- ISR 变更事件
- 副本滞后（Fetch 延迟）

### 追问方向
- LAG 突增的常见原因是什么？
- JMX Exporter 和 Kafka Exporter 的区别？
- `records-lag-max` 和 `records-lag` 的区别是什么？

### 避坑提示
- **LAG 为 0 不一定正常**——如果消费者完全追平，可能说明消费能力过剩或生产停滞。
- JMX Exporter 采集的指标非常**大量**（数千个），存储和查询成本高，建议按需过滤。
- Kafka 0.9 之前用 `SimpleConsumer` 时没有原生 LAG 指标，升级消费者客户端后才有。

---

## 16. Kafka 参数优化：batch.size / linger.ms / buffer.memory / acks

### 题目
Kafka Producer 调优的核心参数有哪些？各自的作用域和推荐值？

### 核心答案

**核心 Producer 参数**

| 参数                | 含义                               | 默认值 | 推荐场景                        |
|---------------------|------------------------------------|--------|---------------------------------|
| `batch.size`        | 单批次最大字节数                   | 16KB   | 吞吐优先：32~128KB；低延迟：1KB |
| `linger.ms`         | 批次等待时间（ms）                 | 0      | 吞吐优先：5~20ms；低延迟：0     |
| `buffer.memory`     | Producer 缓冲总内存                | 32MB   | 按吞吐量调整                    |
| `acks`              | Leader 确认写入的副本数            | 1      | 可靠：all；性能：1；折中：-1    |
| `compression.type`  | 压缩算法                           | none   | 吞吐优先：lz4；带宽优先：zstd   |
| `retries`           | 重试次数                           | 0      | 生产环境：Integer.MAX_VALUE     |
| `max.in.flight...`  | 飞行中请求数（幂等开启时 ≤ 5）      | 5      | 幂等生产者必须 ≤ 5              |
| `request.timeout.ms`| 请求超时                           | 30s    | —                               |

**关键调优思路**

**吞吐优先**：
```properties
batch.size=131072        # 128KB
linger.ms=20
compression.type=lz4
acks=1
```
大 batch + 较长 linger + 快速压缩 = 高吞吐。

**可靠优先（不丢消息）**：
```properties
acks=all
retries=Integer.MAX_VALUE
enable.idempotence=true
max.in.flight.requests.per.connection=5
```
配合 `transactional.id` 可实现 Exactly Once。

**低延迟优先**：
```properties
batch.size=16384
linger.ms=0
compression.type=none
```
单条消息快速发出，但吞吐下降。

### 追问方向
- `buffer.memory` 和 `batch.size` 的关系？OOM 是如何产生的？
- `linger.ms=0` 时 batch 一定为空吗？
- `acks=all` 时，如果 ISR 只有 1 个（其他都宕机）还能写入吗？

### 避坑提示
- `acks=all` 不等于 `replication.factor=3`，必须同时配置 `min.insync.replicas` 才能保证真正写入多个副本。
- `buffer.memory` 不足时 Producer 会**阻塞**（`Block Exceptions`），而不是丢消息，需配合监控。
- `batch.size` 越大，内存占用越高，且单条大消息可能导致 batch 永久填不满（linger 等待）。

---

## 17. Kafka 脑裂问题：Controller 选举冲突与分区状态不一致

### 题目
Kafka 脑裂是什么？会产生什么问题？如何避免？

### 核心答案

**什么是脑裂（Split-Brain）**
- Kafka 集群中出现**多个 Broker 同时认为自己是 Controller** 的状态。
- 正常情况下 ZooKeeper 的临时节点机制保证**同一时刻只有 1 个 Controller**。
- 但在 ZooKeeper 会话超时（Session Expire）瞬间，可能出现短暂的脑裂窗口。

**脑裂产生的场景**
1. **ZooKeeper 网络分区**：Broker A 与 ZooKeeper 断开，ZooKeeper 删除其临时节点；Broker B 趁此机会创建 `/controller` 节点成为新 Controller。
2. **GC 停顿**：Broker 的 GC 停顿超过 `zookeeper.session.timeout.ms`，ZooKeeper 判定会话超时，删除 Controller 节点。
3. **ZooKeeper 自身脑裂**：ZooKeeper Leader 和 Follower 之间网络分区。

**脑裂的症状**
- `kafka.controller:type=KafkaController,name=ActiveControllerCount` > 1
- 两个 Controller 同时下发 `LeaderAndIsrRequest`，导致分区状态不一致
- 分区出现**两个不同的 Leader**，消费者可能读到脏数据或数据丢失

**解决方案**
1. **合理配置 ZooKeeper 超时**：`zookeeper.session.timeout.ms` 不要设置过短（建议 18~30s）。
2. **GC 调优**：避免长时间 GC停顿，使用 G1/CMS 并监控 `maxGCPauseMillis`。
3. **网络隔离**：确保 Broker 与 ZooKeeper 之间的网络稳定。
4. **Kafka 3.0+ KRaft 模式**：去除 ZooKeeper 依赖，从根本上避免 ZooKeeper 脑裂。

### 追问方向
- 如何检测 Kafka 脑裂已经发生？
- ZooKeeper 脑裂和 Kafka 脑裂的关系？
- KRaft 模式下 Controller 选举如何避免脑裂？

### 避坑提示
- **脑裂期间写数据是危险的**——两个 Controller 可能产生两个 Leader，消息写入错误的副本导致数据不一致。
- 监控 `ActiveControllerCount` 是发现脑裂最直接的方式（正常 = 1）。

---

## 18. Kafka 多集群：MirrorMaker2 跨集群复制 / 集群联邦

### 题目
Kafka 如何实现跨集群数据复制？MirrorMaker2 的工作原理是什么？

### 核心答案

**MirrorMaker2（MM2）架构**
- Kafka 自带的跨集群复制工具，基于 **Kafka Connect 框架**。
- 核心组件：Source Connector（消费源集群）+ Sink Connector（写入目标集群）。

**复制拓扑**
```
Cluster A (Source)  →  MirrorMaker2  →  Cluster B (Target)
                     (consumer group: mm2-xxx)
```
- MM2 在目标集群创建**前缀重命名的 Topic**（如 `TopicA` → `TopicA` 可配置过滤）。
- 保留原 Topic 的分区数、配置、ACL（可选同步）。

**核心配置**
```properties
clusters = source, target
source.bootstrap.servers = hostA:9092
target.bootstrap.servers = hostB:9092

source→target.enabled = true
source→target.topics = .*  # 正则匹配

# 复制策略
sync.topic.configs.enabled = true    # 同步 Topic 配置变更
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy
# 或：DefaultReplicationPolicy（含前缀）
```

**跨数据中心复制策略**
1. **主动-被动（主备）**：一个集群为主，写入；另一个仅复制，用于灾备。
2. **主动-主动（双活）**：两个集群均可写入，需处理跨集群消息重复和顺序问题（MM2 不保证全局 Exactly Once）。
3. **联邦（Federation）**：统一入口，分发到不同集群（需业务层路由）。

**注意事项**
- MM2 **不保证全局事务顺序**（跨集群网络延迟不确定）。
- offset 不对齐，目标集群消费位点需要 `offset-sync` 机制对齐。
- 监控 `consumer_lag` 和 `source.cluster.producer` 错误率。

### 追问方向
- MirrorMaker1 和 MirrorMaker2 的区别？
- 如何避免跨集群复制导致的消息乱序？
- 集群联邦的实现方式（Kafka 本身不支持，需要 Kafka Streams 或其他方案）？

### 避坑提示
- MM2 是**异步复制**，不适用于强一致性要求的场景。
- MM2 默认**不复制 ACL 和 Config**，需要单独开启。

---

## 19. Kafka 常见问题：消息丢失 / 消息重复 / 消费顺序异常

### 题目
Kafka 消息丢失、重复消费、顺序异常的原因是什么？如何解决？

### 核心答案

**1. 消息丢失**

| 环节 | 原因 | 解决方案 |
|------|------|----------|
| Producer 发送时 | 网络抖动、`retries=0`、broker 已回复但网络丢失 | `acks=all` + `retries=MAX` + 幂等生产者 |
| Broker 刷盘前 | OS 宕机、Page Cache 未刷盘 | `unclean.leader.election.enable=false` + 副本因子 ≥ 3 |
| Consumer 消费时 | 自动提交 offset 早于业务处理完成 | 手动提交 offset（业务处理完成后）|

**2. 消息重复消费**

| 原因 | 解决方案 |
|------|----------|
| Producer 重试导致重复发送 | 幂等生产者（Idempotent Producer） |
| Consumer Rebalance 后重复消费 | 手动 offset 提交 + 幂等处理（业务去重） |
| 手动 commit 失败但已消费 | 幂等消费（业务去重表/唯一键） |

**3. 消费顺序异常**

| 原因 | 解决方案 |
|------|----------|
| 多消费者并发处理同一分区的消息 | 同一分区内单线程处理 |
| 多分区场景下 Topic 级别乱序 | 按消息 Key 路由到同一分区 |
| Broker 副本同步导致乱序（LEO 不一致） | `acks=all` + `min.insync.replicas=2` 保证有序性 |
| 消费者 seek 操作导致跳消费 | 避免在处理中调用 seek |

**防丢失 + 防重复的最佳实践**
```java
// 业务层幂等处理
@KafkaListener(topics = "order-topic")
public void consume(OrderMessage msg) {
    if (idempotentChecker.exists(msg.getOrderId())) {
        return; // 已处理，跳过
    }
    processOrder(msg);
    kafkaTemplate.send("order-out", msg.getOrderId(), result);
    idempotentChecker.save(msg.getOrderId()); // 标记已处理
    consumer.commitSync(); // 手动提交 offset
}
```

### 追问方向
- 幂等生产者和业务幂等处理能否二选一？
- 多线程并发消费同一分区消息的场景是什么？如何安全实现？
- `enable.auto.commit=false` 场景下，Rebalance 发生时的 offset 提交如何处理？

### 避坑提示
- **幂等生产者防止重复，但不防止消息丢失**；业务幂等处理才是最后防线。
- **按 key 路由到同一分区是保证消息顺序的标准做法**，而非依赖单消费者线程。
- 消息重复和消息乱序**不是同一个问题**，解决思路完全不同，不要混淆。

---

## 20. Kafka 与 RocketMQ 对比：架构差异 / 顺序消息 / 事务消息

### 题目
Kafka 和 RocketMQ 在架构设计上有什么区别？各自如何实现顺序消息和事务消息？

### 核心答案

**架构差异**

| 维度        | Kafka                               | RocketMQ                          |
|-------------|-------------------------------------|-----------------------------------|
| 架构         | 分区副本模型，ZooKeeper/KRaft 管理元数据 | NameServer（注册中心）+ Broker 集群  |
| 元数据存储    | ZooKeeper（早期）/ KRaft（3.0+）     | NameServer（轻量级，无 Leader 选主）|
| 消息模型     | 分区 + 消费者组                      | 普通队列 / 顺序队列 / 事务消息       |
| 消费模型     | 拉取（Pull）模式                     | 拉取 + 推送（Push）混合             |
| 延时消息     | 不支持原生，需第三方                 | 原生支持延时消息（固定延时级别）       |
| 消息回溯     | 按 offset 或时间戳                   | 按时间戳（只支持顺序队列）            |

**顺序消息实现**

**Kafka 顺序消息**：
- 单 Partition 内有序（**分区有序**）。
- 通过 `partitioner.class` 将相同 Key 的消息路由到同一 Partition。
- 消费端单线程消费对应 Partition。

**RocketMQ 顺序消息**：
- **MessageQueueSelector**：全局有序（多分区按序写入同一队列）。
- 按 Queue（队列）维度有序，消费时需锁住 Queue。
- RocketMQ 的有序是**全局有序**（代价是牺牲吞吐量）。

**事务消息实现**

**Kafka 事务消息**：
- `transactional.id` 配置，依赖 **2PC**（预提交 + Commit/Rollback Marker）。
- 消费者端受 `isolation.level` 控制（`read_committed` 跳过未提交消息）。
- **缺陷**：消费端 Pending 事务过多会阻塞后续消息（需配置 `transaction.timeout.ms`）。

**RocketMQ 事务消息**：
- **半消息（Half Message）** 机制：发送时先发 Half 消息，本地事务执行成功后提交。
- RocketMQ Server 主动回查（Check）未决的 Half 消息状态。
- 不依赖消费者侧过滤未提交消息，**对消费端更友好**。

**选型建议**
- **高吞吐、日志采集、事件流**：选 Kafka（生态成熟、社区活跃）。
- **电商订单、金融交易、需要事务 + 延时消息**：选 RocketMQ（原生支持）。
- **强顺序要求 + 高可用**：Kafka 多副本 + 幂等生产者 + 手动提交（成本高）。

### 追问方向
- RocketMQ NameServer 和 Kafka ZooKeeper 的角色定位有何本质不同？
- Kafka 3.0 KRaft 模式后，元数据管理有什么变化？
- 两者在 Exactly Once 实现路径上有何本质区别？

### 避坑提示
- Kafka 的顺序消息是** Partition 级别的**，不是全局的。RocketMQ 声称的全局有序在实践中吞吐量很低。
- RocketMQ 的事务消息实现**比 Kafka 更早成熟**（Kafka 0.11 才引入），但 Kafka 的优势在于端到端生态打通。
- 两者**都不是完美的分布式事务解决方案**，高并发场景下的分布式事务仍需 Saga/TCC 等补偿机制。

---

*文档版本：2025 | 面试题整理自 Kafka 官方文档 + 生产实践*

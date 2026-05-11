# 消息队列（MQ）深度面试题

> 适用岗位：Java后端 / 分布式系统 / WMS仓储系统开发
> 核心考察维度：异步通信原理、Kafka/RabbitMQ/RocketMQ核心机制、可靠性与一致性保障、实际项目落地能力

---

## 第1题：消息队列的核心作用是什么？异步、解耦、削峰具体怎么理解？

### 核心答案

**异步**：生产者和消费者无需同步等待。典型场景：WMS中货架扫码完成 → 发MQ → 后续的库位更新、报表生成、通知TMS等动作全部异步化，响应时间从串联的500ms降到100ms。

**解耦**：上游系统和下游系统通过MQ解耦。上游只管发消息，下游各自订阅，互不影响。WMS对接ERP/WMS/TMS三方时，任何一方改造都不影响另外两方。

**削峰填谷**：大促或突发流量先写入MQ，下游消费者按自身能力拉取消费。Kafka可处理百万级TPS，RabbitMQ适合万级场景，避免把下游打爆。

---

### 追问方向

- 「你们的WMS系统里，具体哪个流程用了MQ？用了多久？」
- 「削峰场景下MQ满了怎么办？有没有做过压测？」
- 「既然MQ这么好，为什么不把所有流程都改成异步？」

---

### 避坑提示

❌ 别说「MQ就是用来异步的」，面试官会追问具体哪个环节、延迟多少、吞吐量多少。
❌ 别把MQ当成数据库用，持久化不是它 primary concern。
✅ 最好能结合WMS项目举例：出库单创建 → MQ通知 → 库存预扣 → TMS调度，三个系统解耦。

---

## 第2题：Kafka的核心概念有哪些？Topic、Partition、Broker、Leader、Follower、ISR分别代表什么？

### 核心答案

- **Topic**：消息主题逻辑容器，发布/订阅的单位。WMS里的「出库单事件」「库存变动事件」各是一个Topic。
- **Partition**：Topic的物理分片，每个Partition内部消息有序。Partition是Kafka并行消费的单位。
- **Broker**：Kafka集群中的一个服务节点，一个Broker管理多个Partition。
- **Leader/Follower**：每个Partition有多个副本，Leader处理读写请求，Follower被动同步。
- **ISR（In-Sync Replicas）**：与Leader保持同步的Follower集合。只有ISR里的副本才有资格被选为新Leader。

```
WMS出库Topic → 3个Partition → 分布在3个Broker上
每个Partition有2个副本 → 1 Leader + 1 Follower → 都在ISR中
```

---

### 追问方向

- 「Partition数和Broker数的关系是什么？Partition太多会有什么问题？」
- 「Follower宕机了，ISR怎么变化？恢复后何时重新加入？」
- 「Controller是怎么产生的？在Kafka集群里负责什么？」

---

### 避坑提示

❌ 不要把ISR和Replication Factor混淆，ISR是动态的，跟上同步进度的副本集合。
❌ 别说「所有副本都是ISR」，ISR是小于等于RF的真子集。
✅ 如果参与过Kafka集群配置，提及`replica.lag.max.messages`等参数。

---

## 第3题：Kafka的分区策略有哪几种？Partition数和Consumer数有什么关系？

### 核心答案

**三大分区策略**：
1. **轮询（RoundRobin）**：消息逐个发给下一个Partition，负载最均匀，默认策略。
2. **按Key哈希（Key Hashing）**：`hash(key) % num.partitions`，相同Key的消息一定发到同一Partition，保证有序性。WMS中同一出库单的事件必须有序，用出库单号作Key。
3. **自定义分区**：实现`Partitioner`接口，根据业务字段（如仓库ID、货主ID）路由。

**Partition数 vs Consumer数**：
- Partition是并行消费的上限：**Consumer数量 ≤ Partition数量**（多余Consumer会空闲）。
- Partition数宜稍大于Consumer数，留有扩展余量。
- Partition数不宜过多（zookeeper压力、文件句柄开销），一般设置为Broker数的2-4倍。

---

### 追问方向

- 「你们WMS项目里分区数设了多少？怎么评估的？」
- 「如果Consumer挂了，重新分配后消息会不会乱序？」
- 「Key为空时消息会怎么分发？」

---

### 避坑提示

❌ 不要说「分区数越多越好」，Partition过多会导致元数据压力大、文件句柄耗尽。
❌ 不要忽略`max.poll.records`对消费并行度的影响。
✅ 结合WMS场景：同一出库单的事件必须有序 → 用出库单号作Key。

---

## 第4题：Kafka是如何实现持久化的？顺序写、页缓存、零拷贝分别是什么原理？

### 核心答案

**顺序写磁盘**：Kafka追加写入只写到文件末尾（Append-only Log），机械硬盘顺序写速度可达500MB/s，接近内存速度，远超随机写。

**页缓存（Page Cache）**：OS将空闲内存作为磁盘缓存。Kafka写入时先到Page Cache，由OS异步刷盘。读取时热点数据也在Page Cache中命中，无需真正从磁盘读。

**零拷贝（Zero Copy）**：传统路径：`磁盘→内核缓冲区→用户空间→Socket缓冲区→网卡`，共4次拷贝。Zero Copy通过`sendfile()`系统调用：`磁盘→内核缓冲区→网卡`，只需2次拷贝，数据完全不经过用户态。Kafka消费者读取消息时走的正是这条路径。

```
// Java中使用Zero Copy
fileChannel.transferTo(position, size, socketChannel);
```

---

### 追问方向

- 「页缓存会不会丢数据？OS什么时候真正刷盘？」
- 「Kafka的`log.flush.interval.messages`参数是控制什么的？」
- 「零拷贝在哪些场景下不适用？」

---

### 避坑提示

❌ 不要说「Kafka数据存在磁盘所以慢」，这是对Kafka最常见的误解。
❌ 不要混淆Page Cache刷盘和Kafka自身的`log.flush`配置，Kafka推荐由OS管理刷盘。
✅ 能讲出`log.retention.hours`和`log.segment.bytes`的配置思路。

---

## 第5题：Kafka的可靠性是怎么保证的？acks参数、HW、LEO、故障恢复流程说清楚。

### 核心答案

**acks参数**：
- `acks=0`：生产者发完即认为成功，丢了不负责，极高吞吐。
- `acks=1`：Leader写入成功即返回，Follower未同步就认为成功，Leader宕机会丢。
- `acks=all`（或`-1`）：ISR中所有副本都写入后才返回，最强可靠性。

**HW（High Watermark）**：已提交消息的分界线，消费者只能读到HW之前的数据，保证不读脏数据。

**LEO（Log End Offset）**：各副本最新写入消息的偏移量。

**故障恢复**：
1. Broker宕机 → Controller检测到Leader失联 → 在ISR中选举新Leader。
2. 旧Leader恢复后，读取本地日志，发现HW，将多余消息截断（Truncate）保持一致。
3. 对外服务恢复。

---

### 追问方向

- 「`min.insync.replicas`参数设多少合理？和acks=all什么关系？」
- 「HW机制保证了什么？HW和LEO的区别是什么？」
- 「故障恢复时消息会不会重复？」

---

### 避坑提示

❌ 不要把`acks=all`等同于「一条不丢」，必须配合`min.insync.replicas >= 2`。
❌ 不要忽略`replication.factor >= 3`的配置要求。
✅ 能在白板上画出Leader选举和故障恢复的时序图。

---

## 第6题：Kafka的Exactly-Once是怎么实现的？幂等生产者和事务消息有什么区别？

### 核心答案

**幂等生产者（Idempotent Producer）**：
开启后`enable.idempotence=true`，每个Producer分配唯一PID，消息带递增序列号。Broker端去重，实现**单会话单分区**的幂等。开销低，适合大多数场景。

**事务消息（Transactional Messages）**：
完整ACID事务能力，支持**跨分区多Partition**原子写。开启`transactional.id`：
```java
producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.commitTransaction(); // 全部成功或全部回滚
```
常用于：WMS中「库存预扣」和「出库单状态变更」必须原子完成。

**Exactly-Once语义**：Kafka 0.11+支持在consume-transform-produce场景下实现端到端Exactly-Once，即消费者处理结果再输出到另一个Topic时不会重复输出。

---

### 追问方向

- 「幂等生产者能跨分区幂等吗？不能的话怎么解决跨分区幂等？」
- 「事务消息的性能损耗大概多少？你们用过吗？」
- 「Exactly-Once能保证业务层的幂等吗？还是只保证Kafka层不重复？」

---

### 避坑提示

❌ 不要把「Kafka的Exactly-Once」和「业务幂等」混为一谈。Kafka只保证不丢、不重复，业务处理仍需自己实现幂等（如数据库唯一键）。
❌ 事务消息有额外开销，小场景不必开。
✅ 结合WMS场景：库存扣减+出库单状态变更 → 必须用事务消息。

---

## 第7题：Kafka的消费模型是怎样的？Pull vs Push、消费者组、Rebalance流程说清楚。

### 核心答案

**Pull vs Push**：
Kafka采用**Consumer Pull**模型——消费者主动从Leader拉取数据。相比RabbitMQ的Push，Pull优势是消费者可控制速率，避免被压垮；批处理更高效。

**消费者组（Consumer Group）**：同一Group内多个Consumer分担不同Partition，实现并行消费。不同Group各自独立消费全量数据（发布-订阅语义）。

```
CG: wms-outbound-group
  Consumer-A → Partition-0, Partition-1
  Consumer-B → Partition-2, Partition-3
```

**Rebalance流程**：
触发条件：Consumer加入/离开/心跳超时/订阅Topic的Partition数变化。
流程：Coordinator通知所有成员 → 全体停止消费 → 重新分配Partition → 各成员提交新offset后恢复消费。

**STICKY_ASSIGNOR**：Rebalance后尽量保持原有Partition分配，减少大规模数据迁移。

---

### 追问方向

- 「Rebalance期间消息会不会丢？」
- 「消费者心跳超时设多少？为什么不能设太短？」
- 「STICKY_ASSIGNOR和RANGE分配策略有什么区别？」

---

### 避坑提示

❌ 不要说「Rebalance很快」，大规模消费者变更时STW明显，是Kafka的痛点之一。
❌ 不要忽略max.poll.interval.ms参数，它是Rebalance延迟的根源之一。
✅ 如果公司用过Kafka 3.x的KRaft模式，知道其去ZooKeeper依赖的优势。

---

## 第8题：Kafka消息的offset是怎么管理的？消费者重启后从哪里继续消费？

### 核心答案

**Offset管理**：
- **提交方式**：Consumer定期向`__consumer_offsets`Topic提交offset（默认`enable.auto.commit=true`，5秒一次）。
- **手动提交**：`commitSync()`/`commitAsync()`精确控制时机，常用于业务处理成功后提交。

**消费者重启后继续点**：
- **自动提交**：上次提交的offset之后的消息。5秒内的已消费未提交消息会**重复消费**。
- **手动提交**：精确到「业务处理完成后」再提交，restart后从上次成功位置继续，**最多重复，不漏消息**。

**earliest/latest策略**：
- `auto.offset.reset=earliest`：无有效offset时从最早消息开始（从头消费）。
- `auto.offset.reset=latest`：无有效offset时从最新消息开始（新消息才被消费）。

---

### 追问方向

- 「如果先手动提交offset=100，但业务处理到99时Consumer挂了，会重复还是漏消息？」
- 「__consumer_offsets这个Topic的Partition数是多少？怎么计算consumer的Coordinator？」
- 「旧版Consumer和新版Consumer的offset存储有什么区别？」

---

### 避坑提示

❌ 不要说「从上次消费的位置继续」，这是模糊表述。必须说清是「已提交的offset之后」，而不是「最后拉取的消息之后」。
❌ 自动提交模式下宕机会重复消费，业务不允许重复的场景必须手动提交。
✅ 能说明`__consumer_offsets`是Kafka内部的Topic，50个Partition，消息格式是`<group_id, topic, partition> → offset`。

---

## 第9题：RabbitMQ的核心概念是什么？Exchange、Queue、Binding、VirtualHost分别负责什么？

### 核心答案

- **Exchange**：消息路由器，按规则把消息路由到一个或多个Queue。**不存储消息**，只做路由决策。
- **Queue**：消息真正存放的FIFO队列，消费者从Queue拉取消息。
- **Binding**：Exchange和Queue之间的绑定关系，包含routing key规则。
- **VirtualHost（Vhost）**：RabbitMQ的逻辑隔离单元，每个Vhost有独立的Exchange/Queue/Binding，用户权限也是按Vhost划分。不同项目/环境隔离。

```
// WMS场景
Vhost: /wms
Exchange: wms.outbound (Topic类型)
Queue: outbound.notify.tms.queue
Binding: wms.outbound → outbound.notify.tms.queue (routing key: outbound.*)
```

---

### 追问方向

- 「Exchange和Queue能一对一，也能一对多，什么场景需要一对多？」
- 「Vhost之间能不能直接跨Vhost路由消息？」
- 「如果Queue不存在，发消息到Exchange会怎样？」

---

### 避坑提示

❌ 不要把Exchange理解为「存储消息的地方」，Queue才是。
❌ 不要忽略Vhost的权限隔离作用，生产环境必须为不同项目分配独立Vhost。
✅ 能用AMQP协议的手工理解Exchange/Queue/Binding三者的关系。

---

## 第10题：RabbitMQ有哪几种路由模式？Direct、Fanout、Topic、Headers分别用在什么场景？

### 核心答案

| 路由模式 | 匹配规则 | 典型场景 |
|---------|---------|---------|
| **Direct** | 完全匹配routing key | WMS库存变动 → 只通知特定系统（如TMS只关心「出库完成」） |
| **Fanout** | 忽略routing key，广播到所有绑定Queue | 系统公告、全量日志收集 |
| **Topic** | wildcard匹配（`*`单级，`#`多级） | WMS事件路由：`outbound.*`匹配所有出库相关事件 |
| **Headers** | 按消息头属性匹配，性能差 | 极少用，某些特殊多条件路由场景 |

```
// Topic示例
wms.events (Exchange, Topic类型)
  binding: outbound.created      → queue_wms_create
  binding: outbound.completed   → queue_wms_complete
  binding: outbound.*           → queue_wms_all_events
  binding: inventory.#          → queue_inventory_all  (#匹配多级)
```

---

### 追问方向

- 「Fanout模式下Consumer挂了，消息会丢失吗？」
- 「Topic的`#`和`*`能混用吗？`*`具体匹配几个词？」
- 「Headers模式为什么性能差？」

---

### 避坑提示

❌ 不要说「routing key就是消息内容」，routing key是路由规则，和消息body无关。
❌ 不要在生产环境使用Headers交换机，性能差且不被某些客户端完整支持。
✅ WMS中推荐Topic交换机，灵活性最高，适合多系统订阅不同事件子集。

---

## 第11题：RabbitMQ的消息确认机制是怎样的？生产者确认、消费者ACK、手动vs自动确认有什么区别？

### 核心答案

**生产者确认（Publisher Confirms）**：
- 开启`publisher-confirms=true`
- Broker收到消息后返回ACK，失败返回NACK，生产者可以重发。
- 同步等待：`addConfirmListener()`；异步回调。

**消费者ACK**：
- **自动ACK（默认）**：Consumer收到消息即认为成功，RabbitMQ立即删除消息。**危险**，Consumer处理中途挂了会丢消息。
- **手动ACK**：Consumer处理完成后显式调用`basicAck()`。支持：
  - `basicAck(deliveryTag, false)`：确认单条
  - `basicAck(deliveryTag, true)`：确认批量
  - `basicNack()/basicReject()`：拒绝消息，可选是否重入队列

**幂等性**：AMQP协议不保证幂等，同一条消息可以多次ACK。业务层必须自己实现幂等（如扣库存用数据库唯一约束）。

---

### 追问方向

- 「手动ACK时，如果处理成功但ACK失败会怎样？」
- 「basicNack的requeue参数为true和false有什么区别？」
- 「消费者处理超时了MQ会重发吗？超时时间怎么配？」

---

### 避坑提示

❌ 不要用自动ACK做业务处理成功率高的假设，生产环境99%场景用手動ACK。
❌ 不要忘记ACK是按deliveryTag操作的，不能跨消息ACK。
✅ 如果能说出prefetch count（`basic.qos`）的作用，控制消费者预取数量实现背压。

---

## 第12题：RabbitMQ的死信队列是什么？DLX、DLQ、消息过期、拒收、超限分别怎么触发？

### 核心答案

**死信队列（DLX/DLQ）**：
- **DLX（Dead Letter Exchange）**：专门处理死信消息的交换机。
- **DLQ（Dead Letter Queue）**：死信最终存放的队列，供人工处理和分析。

**三种触发死信的情况**：
1. **消息过期（TTL）**：队列设置TTL，消息超时未被消费 → 进入DLQ
2. **消息拒收（Reject/Nack）**：消费者显式拒绝且`requeue=false` → 进入DLQ
3. **队列超限（Max Length）**：队列满了，新消息进来时最旧的消息被丢弃成为死信

```java
// 配置死信队列
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx.exchange");
args.put("x-dead-letter-routing-key", "dlq.routing.key");
channel.queueDeclare("normal.queue", false, false, false, args);
```

**WMS应用**：下游系统处理失败的消息进DLQ，定时任务扫描重试或人工处理，避免阻塞正常流程。

---

### 追问方向

- 「DLX本身也可以绑定多个DLQ吗？」
- 「消息TTL过期和队列TTL过期有什么区别？」
- 「DLQ里的消息一般怎么处理？」

---

### 避坑提示

❌ 不要把「拒收但requeue=true」当成死信，这是重试，不是死信。
❌ 不要在生产环境对DLQ置之不理，要有监控和告警。
✅ 能说出在WMS中如何用DLQ实现「下游TMS接口不可用时的延迟重试」。

---

## 第13题：RocketMQ的事务消息是怎么工作的？半消息、本地事务、事务状态回查分别是什么？

### 核心答案

**两阶段提交流程**：

**第一阶段（Half Message）**：
```java
// 生产者发送半消息，此时消费者看不到
producer.sendMessageInTransaction("topic", msg, localTransaction);
```
Broker保存消息但标记为「不可投递给消费者」状态（HALF_MESSAGE）。

**第二阶段（本地事务 + Commit/Rollback）**：
- 生产者执行本地业务（如库存扣减）
- 成功后通知Broker：`commitTransaction()` → 消息变为正常，可被消费
- 失败后通知Broker：`rollbackTransaction()` → 消息被删除

**事务状态回查（Transaction Recovery）**：
如果Broker没收到提交/回滚结果（网络问题、生产者崩溃），会主动回查：
```java
// 实现TransactionListener
@Override
public LocalTransactionState checkLocalTransaction(MessageExt msg) {
    // 查询本地事务状态：已提交/回滚/未知
    return LocalTransactionState.COMMIT_MESSAGE;
}
```
RocketMQ默认回查15次，间隔递增（1s→10s→…）。

---

### 追问方向

- 「Half Message占用Broker存储吗？如果事务长时间不提交怎么办？」
- 「RocketMQ的事务消息和Kafka事务消息相比，区别是什么？」
- 「本地事务执行成功了，但commit网络失败，会发生什么？」

---

### 避坑提示

❌ 不要把RocketMQ事务消息和普通消息混淆，普通消息发送是同步的，事务消息是异步两阶段的。
❌ 不要忽略事务回查的幂等性，本地事务状态表必须记录消息ID防止重复处理。
✅ 结合WMS：出库单创建（Half Message）→ 本地库存预扣 → Commit → 若TCC接口超时则回查本地事务状态。

---

## 第14题：Kafka、RabbitMQ、RocketMQ各自适用什么场景？选型时主要考虑哪些因素？

### 核心答案

| 特性 | Kafka | RabbitMQ | RocketMQ |
|-----|-------|----------|----------|
| **吞吐量** | 百万级TPS | 万级TPS | 十万级TPS |
| **延迟** | 毫秒级 | 微妙～毫秒级 | 毫秒级 |
| **消息可靠性** | 可配置 | 可配置（事务消息） | 支持事务消息 |
| **顺序消息** | 单Partition内有序 | 支持 | 支持 |
| **广播/订阅** | ConsumerGroup模型 | Fanout/Topic | Topic/Queue |
| **死信队列** | 无原生DLQ | 原生支持DLX/DLQ | 支持死信Topic |
| **延迟消息** | 不支持（需插件） | 支持（TTL+DLX） | 原生支持延迟消息 |
| **** | **日志采集、大数据、实时流** | **业务消息、可靠投递、小吞吐** | **电商交易链路、事务消息** |

**选型决策树**：
- **高吞吐、日志采集、流处理** → Kafka
- **复杂路由、可靠投递、延迟队列** → RabbitMQ
- **交易链路、事务消息、需要延迟** → RocketMQ
- **WMS全场景** → Kafka（事件驱动）+ RabbitMQ（业务可靠性消息）

---

### 追问方向

- 「你们WMS用的什么MQ？为什么这么选？」
- 「如果Kafka丢了消息怎么办？怎么保证不丢？」
- 「RabbitMQ的集群模式和镜像队列了解吗？」

---

### 避坑提示

❌ 不要说「Kafka最厉害所以全部用Kafka」，Kafka不适合需要复杂路由和延迟消息的场景。
❌ 不要忽略运维成本，Kafka需要专业运维，RabbitMQ更轻量。
✅ 能说清楚自己项目中不同MQ混用的原因和收益。

---

## 第15题：Kafka如何保证消息顺序？全局有序和分区有序有什么区别？

### 核心答案

**分区有序（Partition-level Order）**：
Kafka单Partition内消息有序。发送时用业务ID作Key（如出库单号），相同Key的消息必然路由到同一Partition，消费时保持有序。这是**最常用的有序保证方式**。

```java
// 保证同一出库单的事件有序
producer.send(new ProducerRecord<>("wms-outbound", outboundOrderId, message));
```

**全局有序（Global Order）**：
要求所有消息按某字段全局有序，有两种方案：
1. **单Partition方案**：Topic只设置1个Partition，生产者/消费者各1个 → 并行度降为1，**严重牺牲性能**。
2. **业务层排序**：发送端按时间戳排序，消费端按时间戳窗口排序（如1分钟内）。推荐方案。

**WMS场景**：
- 「同一出库单的事件必须有序」→ 用出库单号作Key，分区内有序。
- 「同一货架的上下架事件必须有序」→ 用货架ID作Key。

---

### 追问方向

- 「Rebalance之后，同一Key的消息会乱序吗？」
- 「如果一个Partition的Consumer处理慢了怎么办？」
- 「如何保证消费端的消息顺序和发送端一致？」

---

### 避坑提示

❌ 不要用单Partition做全局有序，这是面试官挖的坑，正确的答案是业务层排序。
❌ 不要忽略Consumer端处理失败导致的消息乱序（处理失败→重试→后续消息先被处理）。
✅ 能说明如何用「单调递增序列号」检测乱序并告警。

---

## 第16题：消息积压了怎么办？消费者挂了如何处理？如何快速消费堆积？

### 核心答案

**消费者挂了**：
1. Kafka：消费者掉线 → Group Coordinator触发Rebalance → 其他Consumer接管partition → 自动继续消费。
2. RabbitMQ：Consumer断开 → 消息仍留在Queue → 重新上线Consumer继续消费（若使用手动ACK则未ACK消息会被重发）。

**快速消费堆积**：

| 策略 | 操作 | 风险 |
|-----|------|-----|
| **扩容Consumer** | 增加Consumer实例，Partition数决定上限 | Partition数不足时需先增加Partition |
| **提高消费并行度** | 调大`max.poll.records`/`prefetch` | 数据库压力增大 |
| **临时写透** | 消费者跳过复杂处理，先落库再异步处理 | 可能丢失数据 |
| **消费者降级** | 关闭非核心消费者，资源让给核心 | 非核心功能不可用 |
| **增加Partition** | `bin/kafka-topics.sh --alter` | 已有消息的Key路由会变化 |

**预防措施**：
- 监控消息积压数量，设置告警阈值（如积压>1万条）
- 使用报警 + 自动扩容机制

---

### 追问方向

- 「Kafka增加Partition后，之前的数据会被重新分配吗？」
- 「RabbitMQ中队列积压满了会怎样？新消息被丢弃吗？」
- 「如何设计一个消息积压的监控大盘？」

---

### 避坑提示

❌ 不要在不清楚Partition数的情况下盲目扩容Consumer，扩容后没有Partition的Consumer会空闲。
❌ 不要临时关闭消费者提交offset的逻辑，这会导致重启后大量重复。
✅ 能结合WMS场景：「出库单通知TMS」积压 → 紧急扩容Consumer + 降级非核心处理。

---

## 第17题：消息丢失的场景有哪些？生产者丢失、Broker丢失、消费者丢失分别怎么解决？

### 核心答案

**生产者丢失**：
- **原因**：发送失败但未重试，或重试后仍失败。
- **方案**：
  1. `acks=all` + `retries=MAX` + `enable.idempotence=true`
  2. 发送结果回调检查：失败写入本地数据库/日志，定时重发
  3. 事务消息（性能损耗大，只用于关键链路）

**Broker丢失**：
- **原因**：Leader收到消息后宕机，Follower未同步完成。
- **方案**：
  1. `replication.factor >= 3`
  2. `min.insync.replicas >= 2`
  3. `acks=all`

**消费者丢失**：
- **原因**：自动ACK模式下消费了但处理失败，消息已被删除。
- **方案**：
  1. **手动ACK**：业务处理完成后才提交offset
  2. **先处理再ACK**：如果处理失败则不ACK，MQ会重发
  3. **业务幂等**：数据库唯一键/Redis去重，防止重复处理

```
WMS库存场景防丢失完整链路：
生产者(try-catch+重试+幂等ID) 
  → Broker(副本数3+minISR=2+acks=all) 
  → 消费者(手动ACK+业务幂等)
```

---

### 追问方向

- 「RocketMQ的事务消息能100%防止丢失吗？还有什么边界情况？」
- 「Kafka的`unclean.leader.election`参数设为true会怎样？」
- 「消费者先ACK再处理和先处理再ACK各有什么风险？」

---

### 避坑提示

❌ 不要说「用了MQ就高枕无忧」，MQ只是工具，可靠性靠配置和业务设计。
❌ 不要忽略事务消息的副作用：Kafka事务消息会降吞吐，RocketMQ事务消息有回查机制。
✅ 能画出「消息从生产到消费的完整可靠性保障架构图」。

---

## 第18题：消息重复消费怎么解决？如何设计消费者的幂等性？

### 核心答案

**MQ的At-Least-Once vs 业务的Exactly-Once**：
MQ的「At-Least-Once投递 + 消费者幂等」= 业务层的「Exactly-Once」。

**消费者幂等设计**：

| 方案 | 实现方式 | 适用场景 |
|-----|---------|---------|
| **数据库唯一键** | 消息带唯一ID，插入数据库时用`INSERT IGNORE` | 通用，推荐 |
| **Redis去重** | 消息ID写入Redis SETNX，过期时间 | 高并发、短期去重 |
| **状态机幂等** | 消息携带状态字段，数据库`UPDATE WHERE status=expected` | 状态机流转 |
| **乐观锁版本号** | 消息带版本号，更新时`WHERE version=?` | 并发更新同一记录 |
| **去重表** | 独立去重表，消息ID作主键，插入即幂等 | 需要长期追溯的场景 |

```java
// WMS出库完成幂等示例
@Transactional
public void handleOutboundCompleted(OutboundMessage msg) {
    int rows = outboundOrderMapper.updateStatusIfCurrent(
        msg.getOrderId(), 
        "COMPLETED",      // 目标状态
        "PROCESSING"      // 预期状态（防止重复处理）
    );
    if (rows == 0) {
        log.info("重复消息，已忽略: orderId={}", msg.getOrderId());
    }
}
```

---

### 追问方向

- 「Redis去重和数据库唯一键哪个更好？各自的问题是什么？」
- 「如果消息重复是因为消费者超时重发，如何区分「真的重复」和「恰好相同业务数据」？」
- 「Kafka的幂等生产者能解决消费者幂等的问题吗？」

---

### 避坑提示

❌ 不要说「MQ有Exactly-Once所以不用管重复」，Kafka的Exactly-Once只保证不丢不重复，不代表业务处理幂等。
❌ 不要用消息内容作唯一键（可能重复），必须用MQ提供的唯一消息ID。
✅ 能说出：Kafka消息有`offset+partition`唯一标识，RabbitMQ消息有`deliveryTag`。

---

## 第19题：Kafka为什么这么快？顺序写、零拷贝、页缓存、批量处理、压缩分别如何提升性能？

### 核心答案

**顺序写磁盘**：
机械硬盘顺序写速度500MB/s，随机写0.1MB/s。Kafka只追加写，不修改，顺序IO掩盖了磁盘慢的问题。

**零拷贝（Zero Copy）**：
`FileChannel.transferTo()` → 内核态直接传输，无用户态参与。传输1GB文件：
- 传统方式：4次CPU拷贝 + 4次上下文切换
- Zero Copy：2次CPU拷贝 + 2次上下文切换

**页缓存（Page Cache）**：
OS用空闲内存做磁盘缓存。Kafka写：先入Page Cache（极快），OS异步刷盘。读：热点数据在Page Cache中命中。

**批量处理（Batch）**：
- **批量发送**：`batch.size` + `linger.ms`，攒一批再发，减少网络RTT。
- **批量拉取**：`fetch.min.bytes` + `fetch.max.wait.ms`，一次拉取多MessageSet。
- **批量压缩**：`compression.type=lz4/zstd/snappy`，批量压缩率更高。

**性能全景**：
```
顺序写 → 磁盘IO不再是瓶颈
Zero Copy → CPU不再是瓶颈
Page Cache → 热数据在内存
批量 + 压缩 → 网络IO效率最大化
综合效果：单机百万级TPS
```

---

### 追问方向

- 「Page Cache在Kafka重启后会怎样？冷启动如何处理？」
- 「LZ4、ZStd、Snappy三种压缩算法各有什么优劣？」
- 「Kafka的MessageSet是怎么组织消息的？Record Batch是什么？」

---

### 避坑提示

❌ 不要只说「Kafka快是因为顺序写」，这是不完整的答案，零拷贝和批量处理同样关键。
❌ 不要忽略页缓存的副作用：重启后冷数据需要从磁盘加载，有预热期。
✅ 能从IO模型角度讲：Kafka是**顺序写 + 内存读 + 网络批量发**，每个环节都没有阻塞点。

---

## 第20题：MQ在WMS系统中是怎么用的？库存异步通知、出库完成回调、TMS对接具体怎么实现？

### 核心答案

**1. 库存异步通知（库存变更 → 下游系统）**

```
货架扫码完成 
  → MQ: wms.inventory.changed (Topic, key=sku_id)
  → TMS订阅: 扣减可用库存 → 更新派单计划
  → 报表系统订阅: 实时库存变动报表
  → 预警系统订阅: 库存低于阈值触发补货提醒
```
好处：WMS不关心下游有多少系统，新增系统只需订阅MQ，上游零改动。

**2. 出库完成回调（出库单完成 → TMS调度）**

```java
// 出库单完成时
@PostConstruct
public void init() {
    kafkaTemplate.send("wms.outbound.completed", outboundOrderId, event);
}

// TMS端消费（手动ACK保证不丢）
@KafkaListener(topics = "wms.outbound.completed", groupId = "tms-group")
public void onOutboundCompleted(ConsumerRecord<String, OutboundEvent> record) {
    try {
        tmsService.scheduleDispatch(record.value()); // 回调TMS调度
        kafkaConsumer.commitSync(); // 成功后再ACK
    } catch (Exception e) {
        // 失败不ACK，Kafka会重发；日志记录DLQ
        throw e; 
    }
}
```

**3. TMS对接（双向MQ通信）**

```
WMS → TMS：
  Topic: wms.outbound.created  → TMS订阅 → 预分配车辆
  Topic: wms.outbound.completed → TMS订阅 → 更新运单状态

TMS → WMS：
  Topic: tms.dispatch.assigned  → WMS订阅 → 关联司机信息
  Topic: tms.dispatch.cancelled  → WMS订阅 → 取消出库单
```

**关键设计**：
- 使用出库单号/运单号作MQ消息Key，保证同一业务单的事件有序
- TMS接口不可用 → 消息进入RabbitMQ DLQ → 延迟重试
- 幂等性：所有消费端用业务单号作唯一键，防止重复处理

---

### 追问方向

- 「WMS和TMS之间的消息丢了怎么办？有没有补偿机制？」
- 「如果TMS处理速度比WMS慢很多，MQ积压了怎么处理？」
- 「如何保证WMS和TMS之间的消息顺序一致？」

---

### 避坑提示

❌ 不要只讲「发MQ就行了」，要讲出具体的Topic设计、Key策略、消费幂等、DLQ处理全链路。
❌ 不要忽略MQ消息的版本管理，业务升级后旧消息格式兼容问题。
✅ 能说出自己负责过MQ的哪个环节，配置过什么参数，为什么这么配置。

---

## 附录：面试高频追问方向汇总

| 类别 | 高频追问 |
|-----|---------|
| **Kafka原理** | ISR动态变化、CK/SCK区别、Controller选举 |
| **Kafka可靠性** | unclean.leader.election、min.insync.replicas配置 |
| **Kafka消费** | Rebalance过程、STICKY_ASSIGNOR、offset存储位置 |
| **RabbitMQ** | 镜像队列、Lazy Queue、与Kafka的取舍 |
| **RocketMQ** | 事务消息回查机制、延迟消息实现 |
| **项目实战** | 具体配置参数、监控指标、故障处理经验 |
| **可靠性设计** | 幂等、防重、补偿事务的全链路设计 |

---

> 📌 **面试提示**：MQ是Java后端面试高频模块，建议结合自己项目中的具体使用场景回答。面试官更关注：你遇到过什么问题？怎么解决的？配置了什么参数？单纯背概念很难拿到高薪Offer。

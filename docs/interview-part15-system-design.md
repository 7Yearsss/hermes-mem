# 系统设计面试题——第15部分

> 本套题目面向 Java 后端开发者，结合 WMS 仓库管理系统实战经验（wms-backend、inventory-service-backend、item-master-backend、xms-starter），重点考察候选人在高并发、分布式、库存管理领域的设计能力。

---

## 第1题：设计一个库存扣减系统

### 题目

假设你负责 WMS 系统中 **inventory-service-backend** 的库存扣减模块。业务场景：仓库每小时处理上万次出库指令，要求库存数据强一致，且接口响应时间 P99 ≤ 50ms。请设计一套完整的库存扣减方案，说明 Redis 预减库存、数据库扣减（乐观锁/悲观锁）如何配合，并处理超卖问题。

### 核心答案

**整体架构：三层扣减**

```
客户端请求
    │
    ▼
┌──────────────────────────────────────┐
│  第一层：Redis 预减库存（高性能路径）  │
│  DECR inventory:{skuId}              │
│  返回 < 0 则直接 reject（快速失败）    │
└────────────────┬─────────────────────┘
                 │ 预扣成功
                 ▼
┌──────────────────────────────────────┐
│  第二层：异步消息队列（削峰）          │
│  RocketMQ / Kafka                    │
│  消息体：skuId, quantity, orderId     │
└────────────────┬─────────────────────┘
                 │ 消费
                 ▼
┌──────────────────────────────────────┐
│  第三层：数据库扣减（最终一致性）       │
│  UPDATE inventory SET stock = stock - ? │
│  WHERE sku_id = ? AND stock >= ?     │
│  （乐观锁：version 条件 或 悲观锁：FOR UPDATE）│
└──────────────────────────────────────┘
```

**超卖防护三道关：**

1. **Redis 原子扣减**：`DECR` 是原子操作，多线程不会超扣。扣减后判断 < 0 则加回并拒绝。
2. **数据库乐观锁**：`UPDATE inventory SET stock = stock - qty, version = version + 1 WHERE sku_id = ? AND stock >= qty AND version = ?`。影响行数 = 0 时说明库存不足，触发回滚。
3. **库存回滚补偿**：消费失败时，通过 Redis `INCR` 归还预扣数量；若数据库扣减失败，监听死信队列或定时扫描未完成的订单，人工补偿。

**关键代码片段（inventory-service-backend 风格）：**

```java
// 1. Redis 预减
Long remain = redisTemplate.opsForValue().decrement("inventory:stock:" + skuId, qty);
if (remain < 0) {
    redisTemplate.opsForValue().increment("inventory:stock:" + skuId, qty);
    throw new StockInsufficientException(skuId);
}

// 2. 数据库乐观锁扣减（重试3次）
@Transactional
public boolean deductStock(Long skuId, Integer qty) {
    int rows = jdbcTemplate.update(
        "UPDATE wms_inventory SET stock = stock - ?, " +
        "version = version + 1, update_time = NOW() " +
        "WHERE sku_id = ? AND stock >= ? AND version = ?",
        qty, skuId, qty, currentVersion
    );
    if (rows == 0) {
        // 库存不足或版本冲突，触发补偿
        rollbackRedisStock(skuId, qty);
        return false;
    }
    return true;
}
```

### 追问方向

1. **Redis 库存和数据库库存的一致性如何保证？** —— 答：定时对账（每小时对比 Redis 与 DB 库存差值，触发调平脚本）、双写校验。
2. **Redis 扣减和数据库扣减之间如果机器宕机，怎么处理？** —— 答：消息队列兜底 + 定时任务扫描 + 库存流水表 `inventory_transaction_log` 记录每次扣减，用于事后对账和恢复。
3. **如果秒杀峰值 QPS 达到 10 万，怎么进一步优化？** —— 答：Redis 集群 + Pipeline 批量扣减 + 库存分段（将一个 SKU 的库存分散到多个 Redis Key）。

### 避坑提示

- ❌ 不能只靠数据库扣减，高并发下数据库必被打爆。
- ❌ 不能省略 Redis 预扣失败的回滚逻辑，否则 Redis 库存会逐渐偏离真实值。
- ❌ 不能在 RPC 调用链中做跨服务的数据库分布式事务（延迟太高），应该通过消息最终一致。

---

## 第2题：设计一个WMS库位分配系统

### 题目

在 **wms-backend** 的库位分配模块中，收到一个出库指令（含 SKU 和数量），需要从上千个库位中自动推荐最优库位。请设计库位推荐算法，说明如何处理容量约束、库位类型匹配、区域限制等约束条件，并给出算法的工程实现思路。

### 核心答案

**库位推荐模型：多维度打分制**

```
候选库位筛选 → 约束过滤 → 评分排序 → 推荐TOP-N
```

**约束条件建模：**

| 约束类型 | 描述 | 实现方式 |
|---------|------|---------|
| 容量约束 | 库位剩余容量 ≥ 商品体积 | `WHERE remain_capacity >= productVolume` |
| 类型约束 | 食品/医药需专用库位 | `WHERE location_type = 'COLD' OR 'NORMAL'` |
| 区域约束 | 优先同区域拣选，减少行走距离 | `WHERE zone_id = targetZone` |
| 批次约束 | 先进先出（FIFO） | `ORDER BY production_date ASC` |

**评分公式（加权打分）：**

```
Score = w1×距离分 + w2×容量利用率 + w3×FIFO合规度 + w4×同类库位集中度
```

**工程实现：**

```java
// 库位推荐服务（wms-backend LocationAssignmentService）
public List<Location> recommendLocations(Sku sku, Integer qty) {
    // Step1: 查询所有可用库位（含约束过滤）
    List<Location> candidates = locationMapper.findAvailableLocations(
        sku.getSkuId(), qty, sku.getZoneId(), sku.getLocationType()
    );

    // Step2: 批量计算评分
    return candidates.stream()
        .map(loc -> calculateScore(loc, sku))
        .sorted(Comparator.comparing(LocationScore::getScore).reversed())
        .limit(5)
        .map(LocationScore::getLocation)
        .collect(Collectors.toList());
}

// 评分计算（考虑距离、容量利用率、FIFO、同类聚集）
private double calculateScore(Location loc, Sku sku) {
    double distanceScore = 100 - loc.getDistanceFromDock() * 2; // 距离越近分越高
    double utilizationScore = (double) loc.getUsedCapacity() / loc.getMaxCapacity();
    double fifoScore = isFifoCompliant(loc, sku) ? 100 : 0;
    double clusteringScore = calculateClustering(loc, sku); // 同SKU聚集加分

    return 0.4 * distanceScore + 0.2 * (1 - utilizationScore) * 100
         + 0.25 * fifoScore + 0.15 * clusteringScore;
}
```

**动态规划/贪心选择：** 当订单涉及多 SKU 时，先对 SKU 按体积降序排列，依次为每个 SKU 分配最优库位（贪心），避免大商品占位后小商品无法放置。

### 追问方向

1. **如果某个 SKU 在所有库位都库存不足，怎么处理？** —— 答：触发缺货告警，同时查询是否有同规格替代品库位；支持部分分配 + 缺货等待队列。
2. **多仓库场景下，跨仓库调拨和本仓库分配如何选择？** —— 答：优先本仓库，跨仓库作为 fallback；同时考虑调拨时效（次日达 vs. 当小时出库）。
3. **库位推荐如何支持人工干预和机器学习优化？** —— 答：维护库位推荐规则表（人工配置权重）+ 记录实际选择与推荐的偏差，用于 RL 反馈优化。

### 避坑提示

- ❌ 不能暴力遍历所有库位（上千个），必须在 SQL 层过滤掉明显不满足约束的候选集。
- ❌ FIFO 约束如果只靠 `ORDER BY production_date`，在高并发新入库时可能存在幻读，需要使用 `SELECT ... FOR UPDATE` 锁定库位记录。
- ❌ 评分权重不能写死，应支持配置化调整，以适应不同仓库的业务策略。

---

## 第3题：设计一个分布式ID生成器

### 题目

**item-master-backend** 中的商品主数据需要全局唯一 ID，订单系统也需要订单号。设计一个分布式 ID 生成器，要求：全局唯一、趋势递增（MySQL B+Tree 友好）、高吞吐（单节点 QPS ≥ 10万）、支持水平扩展。请详细说明雪花算法（Snowflake）的结构，并重点讨论时钟回拨问题及处理方案。

### 核心答案

**Snowflake 结构（64bit）：**

```
┌────────────┬──────────────────────────────────────────┐
│  符号位(1) │  时间戳(41bit) │ 机器ID(10bit) │ 序列号(12bit) │
│     0      │  Delta seconds │   WorkerID   │   Sequence    │
└────────────┴──────────────────────────────────────────┘
```

- **时间戳**：相对时间（可使用系统启动epoch或2020-01-01），41bit 约支持 69 年。
- **机器ID**：10bit → 最多 1024 个节点，每个节点唯一。
- **序列号**：12bit → 每毫秒每节点最多 4096 个 ID。

**核心实现（Zookeeper/Etcd 注册 WorkerID）：**

```java
public class SnowflakeIdGenerator {
    private final long workerId;
    private long lastTimestamp = -1L;
    private long sequence = 0L;

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();

        if (timestamp < lastTimestamp) {
            // 时钟回拨：等待回拨差恢复，或直接抛异常
            throw new ClockBackwardsException(lastTimestamp - timestamp);
        }

        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & 4095; // 12bit mask
            if (sequence == 0) {
                timestamp = waitNextMillis(timestamp);
            }
        } else {
            sequence = 0L;
        }

        lastTimestamp = timestamp;
        return (timestamp << 22) | (workerId << 12) | sequence;
    }
}
```

**时钟回拨处理策略（三种方案）：**

| 方案 | 原理 | 优缺点 |
|------|------|--------|
| 方案A：等待自愈 | 检测到回拨后 `while (timestamp <= lastTimestamp)` 循环等待 | 简单，但回拨过大时阻塞时间长 |
| 方案B：WorkerID 漂移 | 将回拨差值叠加到 WorkerID 上，生成"虚拟节点"ID | 牺牲部分 ID 空间，但不停顿 |
| 方案C：落盘保存时间戳 | 将 lastTimestamp 持久化到 Redis/DB，重启后恢复 | 解决重启后的回拨问题，但不能解决运行中 NTP 回拨 |

**推荐实践：方案A + 方案C 组合**，即运行时遇到回拨最多等待 1 秒，超过则告警并拒绝发号；进程重启时从 Redis 加载上次时间戳。

### 追问方向

1. **如果机器 ID 由 Zookeeper 分配，节点挂掉后 WorkerID 怎么回收？** —— 答：临时节点 + 心跳检测，超时删除并释放 WorkerID；新节点启动时争抢临时节点。
2. **Snowflake 的趋势递增在 MySQL 索引上有什么优势？** —— 答：趋势递增使 B+Tree 叶子节点顺序写入，减少页分裂和索引维护开销。
3. **百度 UidGenerator 或 滴滴 TDSQL ID 分布式方案对比 Snowflake 有何优劣？** —— 答：TDSQL 基于批量号段（一次从 DB 拿 1000 个 ID），性能更高但依赖 DB；Snowflake 纯本地不依赖外部服务，但时钟是潜在风险。

### 避坑提示

- ❌ 不能忽略时钟回拨，线上 NTP 服务异常时会导致 ID 重复或服务卡死。
- ❌ 不能将 WorkerID 硬编码在配置文件中，水平扩展时会冲突。
- ❌ 雪花算法生成的 ID 是 long 类型（64bit），前端 JS 会出现精度丢失（`Number.MAX_SAFE_INTEGER` ≈ 9000万亿），需要转成 String 或使用补贴方案（前三位随机+后12位序列）。

---

## 第4题：设计一个延迟任务调度系统

### 题目

在 WMS 系统中，存在大量延迟任务场景：订单超时未支付自动取消、库存占用超时自动释放、库位锁定期满自动解锁。请设计一个延迟任务调度系统，说明时间轮算法（TimingWheel）、Redis ZSet、MQ 延迟消息三种方案的原理和选型，并讨论如何保证任务不丢、重试可靠。

### 核心答案

**三种方案对比：**

| 方案 | 精度 | 容量 | 复杂度 | 适用场景 |
|------|------|------|--------|---------|
| 时间轮（TimingWheel） | 毫秒 | 受内存限制 | 高 | 单机低延迟任务 |
| Redis ZSet | 毫秒 | 受 Redis 内存限制 | 中 | 分布式、广度高 |
| MQ延迟消息 | 秒级（RocketMQ支持秒级） | 海量 | 低 | 跨服务、可靠性要求高 |

**时间轮算法原理：**

```
            ┌─────────────────────────────────────┐
            │        TimingWheel（时间轮）         │
            │  ┌──┬──┬──┬──┬──┬──┬──┬──┐          │
 Tick指针 ──→│  │0 │1 │2 │3 │4 │5 │6 │7 │ ... 63  │
            │  └──┴──┴──┴──┴──┴──┴──┴──┘          │
            │  每格 = 100ms, 总刻度 = 6.4s         │
            │  每个格子是一个 TaskBucket（链表）    │
            └─────────────────────────────────────┘
```

- **推进机制**：定时器每 100ms 推进一格，到达刻度的任务被执行。
- **降级轮**：当任务延迟 > 当前轮总刻度时，放入下一层轮子（类似时钟的秒/分/时）。
- **Netty HashedWheelTimer** 是常见实现。

**Redis ZSet 方案：**

```java
// 添加延迟任务：score = 执行时间戳
redisTemplate.opsForZSet().add("delay:tasks",
    taskId + ":" + payload,
    System.currentTimeMillis() + delayMs);

// 定时轮询（每100ms）：获取已到期的任务
Set<String> expiredTasks = redisTemplate.opsForZSet()
    .rangeByScore("delay:tasks", 0, System.currentTimeMillis());

// 消费并删除
for (String task : expiredTasks) {
    // 原子操作：先删除再执行，防止重复消费
    if (redisTemplate.opsForZSet().remove("delay:tasks", task) > 0) {
        executeTask(task);
    }
}
```

**RocketMQ 延迟消息方案：**

```java
// 发送延迟消息（延迟等级 1=1s, 2=5s, 3=10s ... 18=2h）
message.setDelayTimeLevel(3); // 10秒后投递
producer.send(message);

// 消费端监听
@RocketMQListener(topic = "order-timeout")
public void handleTimeoutOrder(Message msg) {
    // 幂等检查：查询订单状态是否已支付
    if (orderService.isPaid(orderId)) return;
    orderService.cancelOrder(orderId);
}
```

### 追问方向

1. **如果任务执行成功但 Redis ZSet 删除失败，怎么处理？** —— 答：使用 Lua 脚本保证删除的原子性；或记录执行日志，消费端根据幂等 ID 去重。
2. **时间轮在单点故障时任务会丢失，怎么做高可用？** —— 答：多级时间轮 + 任务持久化到 DB；或使用 Redis ZSet 作为时间轮的持久化层。
3. **延迟任务如果需要精确到毫秒级，MQ 延迟消息精度能满足吗？** —— 答：RocketMQ 延迟精度是秒级（基于延迟等级）；需要高精度时用 Redis ZSet + 多线程池。

### 避坑提示

- ❌ 不能依赖单级时间轮处理跨天甚至更长的延迟任务，必须有多级轮子或借助外部存储。
- ❌ Redis ZSet 的 `ZRANGEBYSCORE` 查询要加游标分页，避免一次返回过多数据阻塞 Redis。
- ❌ MQ 延迟消息不能用于事务场景（如库存扣减），MQ 消费失败后的重试机制可能导致库存重复释放，需要额外幂等保护。

---

## 第5题：设计一个接口幂等性方案

### 题目

**wms-backend** 开放了一批 HTTP 接口供上下游调用，网络超时、客户端重试都会导致同一个请求被重复执行。请设计一套完整的接口幂等性方案，覆盖：Token+Redis 机制、唯一消息 ID 去重、业务状态机三种核心手段，并说明各自适用场景和落地实践。

### 核心答案

**幂等性三道防线：**

```
┌─────────────────────────────────────────────────────┐
│  第一道：Token+Redis（适用于 HTTP 接口层）            │
│  ┌────────┐    ┌─────────┐    ┌──────────────┐      │
│  │ Client │───→│ 获取Token │───→│ Redis SETNX │      │
│  └────────┘    └─────────┘    └──────┬───────┘      │
│        │                            │              │
│        │  携带Token请求              │ 存在则拒绝   │
│        └───────────────────────────→│              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  第二道：唯一消息ID去重（适用于 MQ 消息消费）         │
│  ┌──────────────┐    ┌──────────────────┐          │
│  │ 消息ID(msgId)│───→│ Redis SETNX(msgId)│          │
│  └──────────────┘    └────────┬─────────┘           │
│                              │ 不存在则处理         │
│                              ▼                      │
│                     ┌──────────────────┐            │
│                     │ 执行业务逻辑      │            │
│                     │ 并记录流水        │            │
│                     └──────────────────┘            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  第三道：业务状态机（适用于订单等状态流转场景）       │
│                                                     │
│   ┌─────┐  create  ┌─────────┐  pay  ┌───────┐     │
│   │INIT │─────────→│ PENDING │──────→│ PAID  │     │
│   └─────┘          └─────────┘       └───────┘     │
│      △                  │                            │
│      │ cancel            │ timeout/cancel           │
│      └──────────────────┘                          │
│                                                     │
│  状态机保证了：只有合法状态转换才能执行               │
└─────────────────────────────────────────────────────┘
```

**Token+Redis 落地：**

```java
// 生成Token
public String generateToken(String orderId) {
    String token = UUID.randomUUID().toString();
    redisTemplate.opsForValue().set("idemtoken:" + token, orderId, 30, TimeUnit.MINUTES);
    return token;
}

// 幂等校验拦截器
public void checkIdempotent(String token, String orderId) {
    String cached = redisTemplate.opsForValue().get("idemtoken:" + token);
    if (cached == null) {
        throw new IdempotentTokenExpiredException();
    }
    if (!cached.equals(orderId)) {
        throw new IdempotentTokenMismatchException();
    }
    // 校验通过后删除Token（保证一次有效）
    redisTemplate.delete("idemtoken:" + token);
}
```

### 追问方向

1. **Token 校验通过后删除和先执行后删除，哪个更好？** —— 答：先删后执——高并发下可能漏掉重复请求；先执后删——如果执行成功但删除失败，重复请求会被拒绝但已执行成功。实际推荐：执行前检查 + 执行后删除用 Lua 原子操作。
2. **消息队列场景下，消费者超时重试会导致幂等失效怎么处理？** —— 答：消费前记录处理中的状态（PROCESSING），消费完成后更新为 SUCCESS；配合消息 ID 的 Redis 去重。
3. **订单状态机如何防止并发状态转换（如重复支付）？** —— 答：数据库行锁 `SELECT ... FOR UPDATE` 锁定订单记录；或乐观锁版本号 `WHERE status = 'PENDING' AND version = ?`。

### 避坑提示

- ❌ 不能把 Token 存在 Session 中，多实例部署时无法共享。
- ❌ 唯一消息 ID 去重不能只靠 DB 主键约束，高并发下插入会热点竞争，应该用 Redis SETNX 兜底。
- ❌ 业务状态机不是万能的，状态字段必须加索引，且状态变更要用 `UPDATE ... WHERE status = ?` 的方式而不是先查后改。

---

## 第6题：设计一个秒杀系统

### 题目

WMS 系统中偶尔有促销场景（如库存大批量出库活动），需要在短时间内承接极高并发请求。请设计一个秒杀系统，覆盖：库存预热、请求削峰、服务限流、库存扣减、库存回滚的完整链路，并说明如何在 10 万 QPS 下保证不超卖、不错减。

### 核心答案

**秒杀系统架构图（文字版）：**

```
用户请求
    │
    ▼
┌────────────────────────────────────────────┐
│  接入层：CDN + LVS/Nginx                     │
│  静态资源走CDN，动态请求到秒杀网关            │
└────────────────┬───────────────────────────┘
                 │
    ┌────────────▼────────────┐
    │  秒杀网关：限流 + 防刷    │
    │  Guava RateLimiter      │
    │  限制：每用户 N 次/秒    │
    └────────────┬────────────┘
                 │ 限流通过
    ┌────────────▼────────────┐
    │  库存预热：Redis Hash    │
    │  HMSET seckill:sku:{id} │
    │   stock 10000           │
    │   start_time 14:00      │
    │   end_time 14:05        │
    └────────────┬────────────┘
                 │ 预扣
    ┌────────────▼────────────┐
    │  Redis Lua 原子扣减       │
    │  if redis.call('decrby',  │
    │    'sk:stock', n) < 0    │
    │    then return 0         │
    │  end                     │
    │  return 1                │
    └────────────┬────────────┘
                 │ 扣减成功
    ┌────────────▼────────────┐
    │  下发秒杀资格令牌         │
    │  异步写入订单MQ           │
    └────────────┬────────────┘
                 │ 消费
    ┌────────────▼────────────┐
    │  订单服务：创建订单       │
    │  数据库乐观锁扣减         │
    └─────────────────────────┘
```

**Redis Lua 原子扣减脚本：**

```lua
-- 秒杀扣减脚本（原子操作）
local stock = redis.call('GET', KEYS[1])
if not stock then
    return -1  -- 库存未初始化
end
if tonumber(stock) < tonumber(ARGV[1]) then
    return 0   -- 库存不足
end
redis.call('DECRBY', KEYS[1], ARGV[1])
return 1       -- 扣减成功
```

**库存回滚机制：**

1. **超时回滚**：用户获得秒杀资格后 N 分钟内未完成支付，MQ 延迟消息触发回滚。
2. **失败回滚**：订单创建失败时，同步回滚 Redis 库存 + 补偿数据库库存。
3. **定时对账**：每小时对比 Redis 库存和 DB 库存，差值触发告警和调平。

### 追问方向

1. **限流方案如何选择：单机限流 vs. 分布式限流？** —— 答：单机用 Guava RateLimiter，分布式用 Redis + Lua 令牌桶或滑动窗口算法（Redis ZSet 实现）。
2. **如果 Redis 挂了，秒杀系统怎么应急？** —— 答：降级到数据库扣减（悲观锁），但需限制 QPS；或者熔断，直接返回"系统繁忙"。
3. **如何防止黄牛党：同一用户多次请求？** —— 答：用户 ID + IP 维度的限流；请求必须携带秒杀令牌（服务端下发的临时凭证）；风控系统识别集合出价。

### 避坑提示

- ❌ 不能把库存全量放入 Redis，若 Redis 宕机则全部库存信息丢失，必须配合数据库最终兜底。
- ❌ 不能省略库存回滚机制，否则超卖库存无法恢复，导致少卖。
- ❌ 不能用 `DECR` 后判断返回值再做 `INCR` 回滚（两个操作非原子），必须用 Lua 脚本或 Redis 事务。

---

## 第7题：设计一个分布式锁

### 题目

在 **inventory-service-backend** 中，跨多个实例的库存扣减需要互斥；库位分配时同一库位不能同时被两个出库指令占用。请设计一个分布式锁，说明 Redis SETNX + Lua 方案、Redisson 方案、Zookeeper 方案的原理和选型，并讨论：可重入性、锁超时、 watchdog 续期、公平锁等核心问题。

### 核心答案

**三种方案对比：**

| 维度 | Redis SETNX+Lua | Redisson | Zookeeper |
|------|----------------|----------|-----------|
| 实现复杂度 | 低 | 中 | 高 |
| 性能 | 最高 | 高 | 中 |
| 可重入 | 需自行实现 | 原生支持 | 原生支持（临时有序节点） |
| 公平锁 | 不支持（需额外设计） | 支持（看门狗机制） | 支持（临时顺序节点） |
| 可靠性 | 主从异步可能丢锁 | 单机/集群 | 强一致 |
| 适用场景 | 绝大多数并发控制 | 需可重入/续期 | 高可靠事务 |

**Redis SETNX + Lua 实现：**

```lua
-- 加锁脚本（SETNX + 设置过期时间，原子的）
if redis.call('SETNX', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) == 1 then
    return 1  -- 加锁成功
else
    -- 避免重复加锁（可重入）
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        redis.call('PEXPIRE', KEYS[1], ARGV[2])
        return 1
    end
    return 0  -- 加锁失败
end

-- 解锁脚本（必须只能解锁自己的锁）
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

**Redisson 核心原理（Watchdog 自动续期）：**

```java
RLock lock = redissonClient.getLock("inventory:lock:" + skuId);
boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
// Watchdog：每 10 秒检查锁是否仍被持有，
// 若仍持有则自动延长 30 秒（直到 unlock 或宕机）
try {
    // 业务逻辑
    inventoryService.deductStock(skuId, qty);
} finally {
    if (isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 追问方向

1. **Redis 主从模式下，主节点加锁成功但未同步从节点就宕机了，怎么处理？** —— 答：RedLock 算法（向 N 个独立 Redis 节点加锁，超过半数成功才算成功）；但实际工程中推荐使用 Redisson 的单节点锁或 Redlock 由业务容忍度决定。
2. **如果锁持有者宕机，锁超时释放，但业务还在执行，怎么处理？** —— 答：业务执行时间必须 < 锁超时时间；使用 watchdog 续期机制（Redisson 默认 30 秒超时，每 10 秒续期）；或业务层记录锁持有者唯一标识，消费时做幂等检查。
3. **分布式锁 vs. 分布式事务，怎么选？** —— 答：分布式锁用于并发互斥控制（如库存扣减）；分布式事务用于跨服务的原子性操作（如转账）；两者可组合使用。

### 避坑提示

- ❌ 不能在 finally 中 unlock 前不检查锁是否属于当前线程，否则会解锁别人的锁。
- ❌ 不能把锁的过期时间设置得太短，业务还没执行完锁就释放了；也要防止设置太长导致故障节点锁不释放。
- ❌ 不能在 Redis 集群中混用不同的 Key 模式（如有的带前缀有的不带），导致锁粒度混乱。

---

## 第8题：设计一个消息队列

### 题目

**wms-backend** 中订单模块（`wms-backend`）、库存模块（`inventory-service-backend`）、物流模块需要通过消息队列解耦和异步通信。请设计消息队列选型方案，对比 Kafka 和 RocketMQ，说明顺序消息、事务消息的实现机制，并讨论：消息丢失、消息重复消费、消息积压的应对策略。

### 核心答案

**Kafka vs. RocketMQ 选型：**

| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 吞吐量 | 极高（百万级 QPS） | 高（十万级 QPS） |
| 消息延迟 | 低 | 低（但略高于Kafka） |
| 事务消息 | 支持（但较弱） | 原生支持（半消息+确认机制） |
| 顺序消息 | 支持（单Partition内有序） | 支持（MessageQueue维度） |
| 消息回溯 | 支持（offset 回溯） | 支持（时间戳回溯） |
| 延迟/定时消息 | 不支持 | 支持（延迟等级） |
| 适用场景 | 日志、大数据分析 | 交易、订单、金融级可靠 |

**顺序消息实现（Kafka）：**

```java
// 同一个 SKU 的库存扣减消息必须路由到同一 Partition
// Producer端：自定义分区器，根据 skuId 哈希到特定 Partition
producer.send(new ProducerRecord<>(
    "inventory-deduct-topic",
    skuId.toString(),  // key 相同则路由到同一 Partition
    inventoryMessage
));

// Consumer端：单线程消费单个 Partition（避免乱序）
kafkaConsumer.subscribe(Collections.singletonList("inventory-deduct-topic"));
while (true) {
    ConsumerRecords<String, InventoryMessage> records =
        kafkaConsumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, InventoryMessage> record : records) {
        // 按顺序处理扣减
        inventoryService.processDeduct(record.value());
    }
}
```

**RocketMQ 事务消息实现：**

```
┌────────┐   发送半消息    ┌─────────┐   本地事务执行   ┌────────────┐
│ Producer│─────────────→│  Broker  │←───────────────→│ Local DB    │
│         │   成功        │ (Half Q) │                 │ (事务表)    │
└────────┘               └────┬────┘                 └────────────┘
     ↑                         │
     │  COMMIT / ROLLBACK      │
     └─────────────────────────┘
            ┌─────────────────────────┐
            │  若超时未收到确认：       │
            │  Broker回查Producer      │
            │  查询本地事务状态         │
            └─────────────────────────┘
```

```java
// RocketMQ 事务消息
@Transactional
public void sendOrderDeductMessage(Order order) {
    Message message = new Message("order-deduct-topic", order.toJson());
    Transaction transaction = producer.beginTransaction();
    try {
        producer.sendMessageInTransaction(message, order, transaction);
        // 本地事务执行（创建订单、扣减库存）
        orderService.createOrder(order);
        transaction.commit();
    } catch (Exception e) {
        transaction.rollback();
        throw e;
    }
}
```

**消息丢失/重复/积压应对：**

| 问题 | 根因 | 解决方案 |
|------|------|---------|
| 消息丢失 | Producer未确认、网络抖动、Broker宕机 | Producer端同步确认（`sendCallback`）+ Broker副本（`acks=all`）+ 消费者手动提交 offset |
| 消息重复 | Consumer消费超时自动重试、网络抖动 | 业务层幂等（消息ID去重 + 状态机） |
| 消息积压 | Consumer消费能力不足、故障 | 扩容Consumer + 消费者多线程并发消费 + 临时写入HBase/ES降级 |

### 追问方向

1. **RocketMQ 的顺序消息和事务消息可以同时实现吗？** —— 答：可以，但事务消息内部也是半消息机制，确认后才投递到消费队列；顺序性由 MessageQueue 保证。
2. **Kafka 的消费者 OFFSET 提交时机怎么选：自动提交还是手动提交？** —— 答：at-most-once 用自动提交（消费前提交，若消费失败则消息丢）；at-least-once 用手动提交（消费成功后再提交，消息不会丢但可能重复）。
3. **如果消费者处理失败，怎么设计重试策略？** —— 答：死信队列（DLQ）兜底，重试超过 N 次后进入 DLQ，人工介入；避免无限重试。

### 避坑提示

- ❌ 不能在事务中嵌套 RPC 远程调用（RocketMQ 事务消息的本地事务必须是本地数据库操作），否则事务消息可靠性下降。
- ❌ 顺序消息的性能瓶颈在单 Partition 只能单线程消费，QPS 受限；需要全局有序时可以多 Partition 路由到同一 Consumer 队列。
- ❌ 消息积压时不能盲目重启 Consumer，会导致消费位点重置，必须先解决积压根因。

---

## 第9题：设计一个优惠券系统

### 题目

在电商/WMS 系统中，优惠券模块需要支持：券模板创建、发放策略（定时发放/触发发放）、核销链路（订单抵扣）、库存扣减。请设计优惠券系统的完整架构，说明券模板的数据模型、发放链路（依赖库存扣减）、核销时如何与订单系统联动，以及防刷券的策略。

### 核心答案

**优惠券系统架构：**

```
┌──────────────────────────────────────────────────────────────┐
│                     优惠券系统                                │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐   │
│  │ 券模板管理  │  │  发放服务   │  │   核销服务            │   │
│  │ CouponTemplate│ │ IssueService │ │ RedeemService       │   │
│  │ - 面额      │  │ - 定时发放  │  │ - 规则校验            │   │
│  │ - 使用条件  │  │ - 触发发放  │  │ - 库存回滚            │   │
│  │ - 有效期    │  │ - 库存扣减  │  │ - 订单金额抵扣        │   │
│  └────────────┘  └─────┬──────┘  └──────────────────────┘   │
│                        │                                    │
└────────────────────────┼────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐    ┌──────────┐
   │ 用户表   │   │ 库存服务  │    │ 订单服务  │
   │ coupon   │   │ inventory │    │ order    │
   └──────────┘   └──────────┘    └──────────┘
```

**核心数据模型（MySQL）：**

```sql
-- 券模板表
CREATE TABLE coupon_template (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),           -- 券名称
    type ENUM('CASH','DISCOUNT'), -- 满减/折扣
    discount_amount DECIMAL(10,2), -- 面额
    min_order_amount DECIMAL(10,2), -- 最低订单金额
    total_stock INT,             -- 总库存
    remain_stock INT,            -- 剩余库存
    start_time DATETIME,
    end_time DATETIME,
    version INT DEFAULT 0        -- 乐观锁
);

-- 用户优惠券表
CREATE TABLE user_coupon (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    template_id BIGINT,
    status ENUM('UNUSED','USED','EXPIRED'),
    receive_time DATETIME,
    use_time DATETIME,
    order_id BIGINT              -- 核销订单ID
);
```

**发放链路（Redis 库存扣减 + 数据库写入）：**

```java
public void issueCoupon(Long userId, Long templateId) {
    CouponTemplate template = templateMapper.selectById(templateId);

    // 1. Redis 预减库存（原子扣减）
    String redisKey = "coupon:stock:" + templateId;
    Long remain = redisTemplate.opsForValue().decrement(redisKey);
    if (remain < 0) {
        redisTemplate.opsForValue().increment(redisKey, 1); // 加回
        throw new CouponStockExhaustedException();
    }

    // 2. 数据库扣减（乐观锁）
    int rows = templateMapper.decreaseStock(templateId);
    if (rows == 0) {
        redisTemplate.opsForValue().increment(redisKey, 1); // 回滚Redis
        throw new CouponStockExhaustedException();
    }

    // 3. 发放优惠券给用户（事务）
    userCouponService.grantToUser(userId, templateId);
}
```

**核销链路（订单抵扣）：**

```java
public OrderResult redeemCoupon(Long userId, Long orderId, Long couponId) {
    UserCoupon coupon = userCouponMapper.selectById(couponId);

    // 规则校验
    if (!coupon.getUserId().equals(userId)) throw new CouponNotOwnedException();
    if (!coupon.getStatus().equals("UNUSED")) throw new CouponAlreadyUsedException();
    if (coupon.isExpired()) throw new CouponExpiredException();

    Order order = orderService.getOrder(orderId);
    if (order.getAmount().compareTo(coupon.getMinOrderAmount()) < 0) {
        throw new CouponAmountNotMetException();
    }

    // 扣减订单金额
    orderService.deductAmount(orderId, coupon.getDiscountAmount());

    // 更新券状态
    userCouponMapper.updateStatus(couponId, "USED", orderId);
    return new OrderResult(orderId, coupon.getDiscountAmount());
}
```

### 追问方向

1. **优惠券超卖问题：Redis 扣减成功但数据库写入失败，怎么处理？** —— 答：发放消息写入 MQ，消费者重试数据库写入；若超过重试次数，触发 Redis 库存补偿脚本。
2. **优惠券薅羊毛：同一个用户重复领取怎么防？** —— 答：用户ID+模板ID 联合唯一索引 + Redis SETNX 去重；每日发放上限控制。
3. **券模板修改后，已发放的券怎么处理？** —— 答：已发放的券不受影响（基于发放时的模板快照）；只有未发放的券使用新模板。

### 避坑提示

- ❌ 不能在发放时直接 UPDATE 库存在高并发下会导致大量行锁；必须 Redis 预减 + 数据库最终扣减。
- ❌ 优惠券有效期不能仅靠前端控制，后端也必须校验 `use_time <= end_time`，防刷券。
- ❌ 折扣券的面额计算要用 `BigDecimal`，不能用 `double`，否则出现 0.06 + 0.01 = 0.069999999 问题。

---

## 第10题：设计一个权限控制系统

### 题目

在 **xms-starter** 平台中，需要为不同角色的用户（仓库管理员、操作员、审计员）配置菜单、按钮、数据权限。请设计一个权限控制系统，说明 RBAC 模型（Role-Based Access Control）、菜单树的构建、按钮级权限（细粒度控制）、数据权限（范围约束）的实现方案。

### 核心答案

**RBAC 模型：**

```
用户 ──属于──→ 角色 ──关联──→ 权限
                  │
                  ├──→ 菜单权限（能看到哪些菜单）
                  ├──→ 按钮权限（能看到哪些操作按钮）
                  └──→ 数据权限（能看哪些范围的数据）
```

**核心数据模型：**

```sql
-- 用户表
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    org_id BIGINT,           -- 组织ID（数据权限用）
    PRIMARY KEY (id)
);

-- 角色表
CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY,
    role_name VARCHAR(50),
    role_code VARCHAR(50),
    data_scope ENUM('ALL','ORG','SELF')  -- 数据权限范围
);

-- 用户-角色关联
CREATE TABLE sys_user_role (
    user_id BIGINT,
    role_id BIGINT,
    PRIMARY KEY (user_id, role_id)
);

-- 菜单表（树形结构）
CREATE TABLE sys_menu (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    parent_id BIGINT,         -- 父菜单ID
    path VARCHAR(200),        -- 路由路径
    type ENUM('MENU','BUTTON'), -- 菜单 or 按钮
    permission VARCHAR(100),  -- 权限标识（如 system:user:add）
    sort_order INT,
    PRIMARY KEY (id)
);

-- 角色-菜单关联
CREATE TABLE sys_role_menu (
    role_id BIGINT,
    menu_id BIGINT,
    PRIMARY KEY (role_id, menu_id)
);
```

**菜单树构建（递归）：**

```java
public List<MenuNode> buildMenuTree(Long roleId) {
    // 查询角色关联的所有菜单
    List<Menu> menus = menuMapper.findByRoleId(roleId);

    // 转换为树形结构
    Map<Long, List<Menu>> grouped = menus.stream()
        .filter(m -> m.getType() == "MENU")
        .collect(Collectors.groupingBy(Menu::getParentId));

    return grouped.getOrDefault(0L, Collections.emptyList()).stream()
        .map(menu -> buildNode(menu, grouped))
        .collect(Collectors.toList());
}

private MenuNode buildNode(Menu menu, Map<Long, List<Menu>> grouped) {
    MenuNode node = new MenuNode(menu);
    List<Menu> children = grouped.getOrDefault(menu.getId(), Collections.emptyList());
    node.setChildren(children.stream()
        .map(c -> buildNode(c, grouped))
        .collect(Collectors.toList()));
    return node;
}
```

**按钮级权限控制（Spring Security / 注解）：**

```java
// 自定义权限注解
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresPermission {
    String value(); // 如 "wms:inventory:deduct"
}

// 权限校验切面
@Aspect
@Component
public class PermissionAspect {
    @Around("@annotation(requiresPermission)")
    public Object checkPermission(ProceedingJoinPoint pjp,
                                   RequiresPermission requiresPermission) {
        String permission = requiresPermission.value();
        UserDetails user = SecurityContextHolder.getCurrentUser();
        if (!permissionService.hasPermission(user, permission)) {
            throw new AccessDeniedException("无操作权限");
        }
        return pjp.proceed();
    }
}

// 使用示例
@RequiresPermission("wms:inventory:deduct")
public void deductStock(Long skuId, Integer qty) { ... }
```

**数据权限实现（MyBatis 拦截器）：**

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare")})
public class DataScopeInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation inv) throws Throwable {
        String sql = inv.getTarget().getSql();
        UserDetails user = SecurityContextHolder.getCurrentUser();

        // 动态追加数据权限条件
        if (user.getDataScope() == DataScope.ORG) {
            sql = sql.replace("WHERE", "WHERE org_id = " + user.getOrgId() + " AND");
        } else if (user.getDataScope() == DataScope.SELF) {
            sql = sql.replace("WHERE", "WHERE create_user_id = " + user.getId() + " AND");
        }
        return inv.proceed();
    }
}
```

### 追问方向

1. **如果一个用户有多个角色，权限怎么合并？** —— 答：取并集（用户能看到所有角色权限的合集）；但如果涉及互斥角色（如同时拥有管理员和普通用户角色），取并集可能导致权限过大，需特殊处理。
2. **菜单树如何支持无限层级和性能优化？** —— 答：数据库存储父ID + 前端懒加载（按需加载子节点）；或预加载全部到 Redis，修改时刷新缓存。
3. **如何实现字段级权限（如普通操作员看不到价格字段）？** —— 答：在 VO/DTO 层做字段脱敏 + MyBatis 结果映射时过滤；或使用 JSON 动态字段过滤。

### 避坑提示

- ❌ 不能把权限数据全部塞进 Session，多实例部署时权限更新无法实时生效，应该每次从 DB/Redis 读取。
- ❌ 数据权限的 SQL 拦截不能拼接原始参数（SQL 注入风险），必须使用参数绑定。
- ❌ 超级管理员角色不要走权限校验逻辑，否则删错数据权限配置会导致系统无法管理。

---

## 第11题：设计一个商品SKU系统

### 题目

在 **item-master-backend** 中，商品主数据需要管理 SPU（标准化产品单元）和 SKU（库存量单位）的层级关系。请设计商品 SKU 系统，说明 SPU/SKU 的关系建模、多规格属性（颜色×尺码等）的笛卡尔积展开、库存统一管理方案，以及 SKU 维度库存与 SPU 维度聚合查询的实现。

### 核心答案

**SPU/SKU 关系模型：**

```
SPU（商品主数据）——规格模板——→ SKU（具体商品）
  商品名称          颜色:红/蓝       SKU_001(红色_42码)
  品牌              尺码:40/41/42    SKU_002(红色_41码)
  分类              材质             SKU_003(蓝色_42码)
  主图              ...
```

**数据库设计：**

```sql
-- SPU表（商品公共属性）
CREATE TABLE mdm_spu (
    id BIGINT PRIMARY KEY,
    spu_code VARCHAR(50) UNIQUE,
    name VARCHAR(200),
    brand_id BIGINT,
    category_id BIGINT,
    description TEXT,
    PRIMARY KEY (id)
);

-- SKU表（单规格库存属性）
CREATE TABLE mdm_sku (
    id BIGINT PRIMARY KEY,
    sku_code VARCHAR(50) UNIQUE,
    spu_id BIGINT,
    spec_json JSON,           -- 规格值JSON: {"颜色":"红","尺码":"42"}
    price DECIMAL(10,2),
    weight DECIMAL(10,3),     -- 重量(kg)
    PRIMARY KEY (id),
    FOREIGN KEY (spu_id) REFERENCES mdm_spu(id)
);

-- SPU规格模板表（定义有哪些规格维度）
CREATE TABLE mdm_spec_template (
    id BIGINT PRIMARY KEY,
    spu_id BIGINT,
    spec_name VARCHAR(50),    -- 规格名称: 颜色、尺码
    spec_values TEXT,         -- 可选值: 红,蓝 / 40,41,42
    sort_order INT
);

-- 库存统一表（按SKU维度）
CREATE TABLE inv_sku_stock (
    sku_id BIGINT PRIMARY KEY,
    available_stock INT,       -- 可用库存
    locked_stock INT,          -- 锁定库存（占用中）
    total_stock INT,           -- 总库存 = available + locked
    warehouse_id BIGINT,
    version INT DEFAULT 0,     -- 乐观锁
    PRIMARY KEY (sku_id)
);
```

**笛卡尔积展开（Java 实现）：**

```java
public List<SkuSpec> generateSkuCartesian(Long spuId) {
    SpecTemplate template = specTemplateMapper.selectBySpuId(spuId);

    // 收集所有规格维度
    List<List<String>> specValueLists = template.getSpecs().stream()
        .map(SpecTemplate::getSpecValues) // List<String> 每个规格的可选值
        .collect(Collectors.toList());

    // 笛卡尔积计算
    List<List<String>> cartesian = cartesianProduct(specValueLists);

    // 生成SKU列表
    return cartesian.stream().map(values -> {
        Map<String, String> specMap = new LinkedHashMap<>();
        for (int i = 0; i < template.getSpecs().size(); i++) {
            specMap.put(template.getSpecs().get(i).getSpecName(), values.get(i));
        }
        String skuCode = generateSkuCode(spuId, specMap);
        return new SkuSpec(skuCode, spuId, specMap);
    }).collect(Collectors.toList());
}

// 笛卡尔积核心算法
private List<List<String>> cartesianProduct(List<List<String>> lists) {
    List<List<String>> result = new ArrayList<>();
    result.add(new ArrayList<>());
    for (List<String> list : lists) {
        List<List<String>> newResult = new ArrayList<>();
        for (List<String> current : result) {
            for (String item : list) {
                List<String> newItem = new ArrayList<>(current);
                newItem.add(item);
                newResult.add(newItem);
            }
        }
        result = newResult;
    }
    return result;
}
```

**SKU 库存统一管理（聚合查询）：**

```java
// SPU维度聚合库存（查询某商品总可用库存）
public Integer getSpuTotalAvailableStock(Long spuId) {
    List<Long> skuIds = skuMapper.selectIdsBySpuId(spuId);
    if (skuIds.isEmpty()) return 0;
    return skuStockMapper.sumAvailableStockBySkuIds(skuIds);
}

// SKU维度精确扣减（批量扣减时分配到具体库位）
public Map<Long, Integer> allocateStockToLocations(Long skuId, Integer totalQty) {
    // 查询该SKU在所有库位的库存，按FIFO排序
    List<SkuLocationStock> locationStocks =
        skuLocationStockMapper.selectBySkuIdFIFO(skuId);

    Map<Long, Integer> allocation = new LinkedHashMap<>();
    int remain = totalQty;
    for (SkuLocationStock loc : locationStocks) {
        if (remain <= 0) break;
        int deduct = Math.min(remain, loc.getAvailableStock());
        allocation.put(loc.getLocationId(), deduct);
        remain -= deduct;
    }
    return allocation;
}
```

### 追问方向

1. **如果SKU规格维度很多（颜色×尺码×材质×版本），笛卡尔积数量爆炸怎么办？** —— 答：拆分为多级 SPU（如大类→中类→SKU）；或限制规格组合数量（商家只能选预定义的组合）；或使用矩阵表替代笛卡尔积。
2. **SKU的规格属性（JSON）如何支持高效查询？** —— 答：MySQL 8.0+ JSON 索引；或 ES 存储用于多维度搜索；或预解析到独立字段（颜色、尺码各一列）。
3. **SKU库存扣减时，如何选择从哪个库位出库？** —— 答：FIFO（按生产日期）、就近库位（距离打包台最近）、库位类型优先级。参考第2题库位推荐算法。

### 避坑提示

- ❌ 笛卡尔积展开不能每次请求都重新算，必须缓存或预生成结果，否则 SKU 上万时性能严重下降。
- ❌ SPU 修改规格模板后，已生成的 SKU 规格 JSON 必须保持不变（基于快照），不能追溯修改历史 SKU。
- ❌ SKU 的 `sku_code` 必须是唯一索引，且不可修改，因为下游系统（订单、WMS）依赖 SKU Code 做关联。

---

## 第12题：设计一个订单履约系统

### 题目

订单创建后，需要经过拆单、库存占用、出库、状态流转等履约流程。请设计订单履约系统，说明：拆单规则（按仓库/按商品类型/按配送时效）、库存占用（预占+实占+释放）、出库流程（波次→拣货→打包→发货）、状态流转（状态机+事件驱动），并讨论拆单后如何保证最终一致性。

### 核心答案

**订单履约全链路状态机：**

```
CREATE_PENDING → 库存占用 → 分拨 → 波次合并 → 拣货 → 打包 → 出库 → 配送 → 签收
     │                  │                              │
     ↓                  ↓                              ↓
  已取消            占用失败                        部分出库
```

**拆单规则引擎：**

```java
public List<Order> splitOrder(Order originalOrder) {
    List<SubOrder> subOrders = new ArrayList<>();

    // 按仓库拆分
    Map<Long, List<OrderItem>> byWarehouse = originalOrder.getItems().stream()
        .collect(Collectors.groupingBy(item -> item.getWarehouseId()));

    for (Map.Entry<Long, List<OrderItem>> entry : byWarehouse.entrySet()) {
        SubOrder subOrder = new SubOrder();
        subOrder.setWarehouseId(entry.getKey());
        subOrder.setItems(entry.getValue());

        // 同一仓库内，按配送时效再次拆
        Map<String, List<OrderItem>> byDeliveryType =
            entry.getValue().stream()
                .collect(Collectors.groupingBy(OrderItem::getDeliveryType));

        for (Map.Entry<String, List<OrderItem>> typeEntry : byDeliveryType.entrySet()) {
            subOrder.getItems().addAll(typeEntry.getValue());
            subOrder.setDeliveryType(typeEntry.getKey());
        }

        subOrders.add(subOrder);
    }
    return subOrders;
}
```

**库存占用三阶段：**

| 阶段 | 库存类型 | 说明 | 超时释放 |
|------|---------|------|---------|
| 预占 | `locked_stock` | 下单时锁定，不允许其他订单使用 | 15分钟未支付自动释放 |
| 实占 | `available_stock -= qty` | 支付后正式扣减，从可用转为已用 | 不释放 |
| 释放 | 回滚 | 取消/退货时还原库存 | 即时 |

```java
// 库存预占（order-service-backend）
public void preAllocateStock(Long orderId, List<OrderItem> items) {
    for (OrderItem item : items) {
        int rows = stockMapper.update(
            "UPDATE inv_sku_stock SET available_stock = available_stock - ?, " +
            "locked_stock = locked_stock + ?, version = version + 1 " +
            "WHERE sku_id = ? AND available_stock >= ?",
            item.getQty(), item.getQty(), item.getSkuId(), item.getQty()
        );
        if (rows == 0) throw new StockInsufficientException(item.getSkuId());
    }
    // 记录预占流水
    stockFlowService.recordPreAllocate(orderId, items);
}

// 支付成功后：预占转实占
public void confirmAllocation(Long orderId) {
    List<OrderItem> items = orderMapper.selectItemsByOrderId(orderId);
    for (OrderItem item : items) {
        stockMapper.update(
            "UPDATE inv_sku_stock SET locked_stock = locked_stock - ? " +
            "WHERE sku_id = ?", item.getQty(), item.getSkuId()
        );
    }
}
```

**出库流程（事件驱动）：**

```
┌─────────────────────────────────────────────────────┐
│  波次创建事件 (WaveCreatedEvent)                     │
│       ↓                                             │
│  拣货任务生成 → RabbitMQ/WarehouseTaskTopic          │
│       ↓                                             │
│  库库员工PDA确认拣货 → 拣货完成事件                   │
│       ↓                                             │
│  打包复核 → 打包完成事件                             │
│       ↓                                             │
│  出库交接 → 配送发货事件 (DeliveryShippedEvent)      │
│       ↓                                             │
│  订单状态更新 → 已发货                               │
└─────────────────────────────────────────────────────┘
```

### 追问方向

1. **拆单后部分子订单失败（库存不足），如何保证一致性？** —— 答：拆单失败子订单标记"缺货"，触发告警和库存调拨；主订单进入"部分履约"状态；或人工介入处理。
2. **订单超时未支付自动取消，如何避免库存重复释放？** —— 答：预占时写入 `order_stock_lock` 记录（含订单ID、SKU、预占数量）；释放时先查询是否存在且状态为"预占中"，再执行释放（幂等）。
3. **出库后如果发生退货，库存怎么回滚？** —— 答：逆向物流入库后，`available_stock += qty`；退货入库验收通过后，更新订单状态为"已退货"并同步财务退款。

### 避坑提示

- ❌ 拆单逻辑不能无限递归（按仓库→按品类→按时效→按批次），一般最多三层，否则订单系统复杂度爆炸。
- ❌ 库存预占超时释放必须用定时任务（而非 MQ 延迟消息），因为 MQ 消息在进程重启时会丢失，定时任务更可靠。
- ❌ 订单状态流转必须用状态机（不会发生非法状态如"已签收"回到"拣货中"），不能用简单的 `UPDATE status = ?`。

---

## 第13题：设计一个搜索推荐系统

### 题目

**item-master-backend** 中的商品搜索需要支持全文检索（按名称/描述/规格搜索）、相关性排序（销量+评分+新品）、热词统计（实时热搜榜）。请设计搜索推荐系统，说明 Elasticsearch 的使用、分词器选型（IKAnalyzer）、相关性排序公式、热词统计的实现。

### 核心答案

**搜索系统架构：**

```
商品数据写入 → MySQL → Canal监听Binlog → ES索引更新
                                              ↓
用户搜索请求 → Nginx → Search API → ES查询 → 返回结果
                                    ↓
                           热词统计 → Redis Sorted Set
```

**ES 索引 Mapping：**

```json
{
  "mappings": {
    "properties": {
      "sku_id": {"type": "long"},
      "spu_name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {"type": "keyword"}
        }
      },
      "description": {"type": "text", "analyzer": "ik_max_word"},
      "category_id": {"type": "long"},
      "brand_id": {"type": "long"},
      "specs": {"type": "nested"},
      "price": {"type": "double"},
      "sales_count": {"type": "integer"},
      "rating": {"type": "double"},
      "create_time": {"type": "date"}
    }
  }
}
```

**相关性排序公式（Function Score）：**

```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "should": [
            {"match": {"spu_name": {"query": "运动鞋", "boost": 3}}},
            {"match": {"description": "运动鞋"}},
            {"term": {"category_id": 100}}
          ]
        }
      },
      "functions": [
        {"filter": {"range": {"sales_count": {"gt": 1000}}}, "gauss": {"sales_count": {"origin": 10000, "scale": 5000, "decay": 0.5}}, "weight": 2},
        {"filter": {"range": {"rating": {"gt": 4.5}}}, "field_value_factor": {"field": "rating", "factor": 1.5, "modifier": "sqrt"}},
        {"gauss": {"create_time": {"origin": "now", "scale": "30d", "decay": 0.5}}, "weight": 1}
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

**热词统计（Redis Sorted Set + 滑动窗口）：**

```java
// 用户搜索词实时入队（每搜索一次，score +1）
public void recordSearchKeyword(String keyword) {
    String today = LocalDate.now().toString();
    String key = "hotwords:" + today;
    // 使用 ZINCRBY，关键词 score +1
    redisTemplate.opsForZSet().incrementScore(key, keyword, 1);
}

// 获取实时热搜TOP10
public List<String> getHotKeywords(int topN) {
    String today = LocalDate.now().toString();
    Set<String> hot = redisTemplate.opsForZSet()
        .reverseRange("hotwords:" + today, 0, topN - 1);
    return new ArrayList<>(hot);
}

// 滑动窗口：合并最近7天热词（每日权重递减）
public List<String> getHotKeywordsWeighted(int topN) {
    String[] keys = new String[7];
    double[] weights = {1.0, 0.6, 0.4, 0.3, 0.2, 0.1, 0.05};
    for (int i = 0; i < 7; i++) {
        keys[i] = "hotwords:" + LocalDate.now().minusDays(i).toString();
    }
    // 使用 ZUNIONSTORE 合并加权后的热词
    return Arrays.asList(keys);
}
```

### 追问方向

1. **IKAnalyzer 的分词策略：ik_max_word vs ik_smart 怎么选？** —— 答：`ik_max_word` 穷尽所有分词可能（召回率高，适合搜索建议）；`ik_smart` 智能切分（精度高，适合最终搜索结果）。
2. **ES 搜索结果与数据库不一致（商品已下架但 ES 仍有），怎么处理？** —— 答：Canal 监听 Binlog 实时同步，删除/下架时主动删除 ES 文档；定时全量对比修正。
3. **搜索降级方案：ES 挂了怎么办？** —— 答：降级到 MySQL LIKE 模糊查询（牺牲性能保功能）；或返回缓存的历史搜索结果。

### 避坑提示

- ❌ ES 的 `text` 类型不支持聚合，如需按品牌/分类聚合，需要额外定义 `keyword` 子字段。
- ❌ 热词统计不能用普通 String 存储并 INCR，会导致单 Key QPS 热点，必须用 Redis ZSet 按关键词分桶。
- ❌ 搜索结果的缓存不能用固定 TTL（否则更新后缓存不刷新），必须结合 Canal 监听变更主动淘汰。

---

## 第14题：设计一个数据同步方案

### 题目

**item-master-backend**（商品主数据）和 **inventory-service-backend**（库存数据）需要实时同步，库存数据来自 ERP 系统（MySQL）。当前方案是定时批量拉取，但延迟太大（分钟级）。请设计一个高效的数据同步方案，对比 Canal 和 Binlog 监听，说明增量同步、全量同步的触发机制，以及同步冲突的处理策略。

### 核心答案

**数据同步架构：**

```
ERP(MySQL) ──Binlog──→ Canal Server ──→ MQ ──→ 消费服务 ──→ 目标库
                                    │
                                    └──→ 增量同步（实时）
                                    │
                                    └──→ 全量同步（定时/触发）
```

**Canal 工作原理：**

```
MySQL 主库开启 Binlog（ROW 模式）
    ↓
Canal Server 伪装成 MySQL Slave，接收 Binlog 事件
    ↓
Binlog 解析为 Entry（行数据变化）
    ↓
发送到 Kafka / RocketMQ（分区按 Table 名）
    ↓
消费服务消费消息，更新目标库
```

**Canal 配置（canal.properties）：**

```properties
canal.serverMode = kafka
kafka.bootstrap.servers = kafka:9092
canal.destinations = inventory-sync,item-sync
# 按表过滤，只同步需要的表
item-master-backend.canal.filter.regex = item\\.mdm_spu,item\\.mdm_sku
inventory-service-backend.canal.filter.regex = inventory\\.inv_sku_stock
```

**增量同步实现（消费 MQ 消息）：**

```java
@RocketMQListener(topic = "canal-inventory-sync", concurrency = "10")
public void processInventorySync(Message message) {
    CanalEntry.Entry entry = CanalEntry.Entry.parseFrom(message.getBody());

    // 遍历每一条变更记录
    for (CanalEntry.RowChange rowChange : entry.getRowChangesList()) {
        CanalEntry.EventType eventType = rowChange.getEventType();

        for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
            if (eventType == CanalEntry.EventType.UPDATE) {
                // 提取变更后的值
                Map<String, String> afterValues = parseRow(rowData.getAfterColumnsList());
                inventoryService.syncUpdate(
                    Long.parseLong(afterValues.get("sku_id")),
                    Integer.parseInt(afterValues.get("available_stock"))
                );
            }
        }
    }
}
```

**全量同步策略：**

| 触发时机 | 方式 | 场景 |
|---------|------|------|
| 定时全量 | 每日凌晨跑 ETL | 数据修复、初始同步 |
| 增量水位标记 | 记录 Binlog 位点，宕机后从标记点恢复 | 故障恢复 |
| 人工触发 | 管理后台一键全量 | 重大数据修复 |
| 实时比对 | 每小时对比源库和目标库行数，差异告警 | 数据质量监控 |

**同步冲突处理：**

| 冲突类型 | 处理策略 |
|---------|---------|
| 主键冲突（INSERT） | 目标库已有记录 → 更新（UPSERT）而非插入 |
| 数据覆盖冲突（UPDATE） | 以时间戳判断：谁更新用谁；以业务字段判断优先级 |
| 删除冲突 | 软删除优先，不物理删除；必须硬删除时记录删除日志 |
| 顺序乱序 | Binlog 有序号字段，按序号顺序重放；Canal 支持按事务维度顺序消费 |

### 追问方向

1. **Canal 和 Maxwell、Debezium 对比，有什么优劣？** —— 答：Canal 是阿里巴巴开源，Java 实现，运维成本低但文档少；Debezium 功能最全（支持多种 DB）但资源消耗高；Maxwell 轻量但功能有限。
2. **如果 MQ 消费失败，怎么保证不丢消息？** —— 答：手动 ACK 模式，消费成功后再 ACK；消费失败进入重试队列（延迟重试）；超过重试次数进入死信队列。
3. **Binlog 同步的延迟怎么监控？** —— 答：Canal Server 记录每次同步的延迟时间戳，写入 Prometheus；或目标库对比 `update_time` 与 Canal 消息时间戳的差值。

### 避坑提示

- ❌ MySQL Binlog 必须使用 ROW 模式（完整行数据），STATEMENT 模式无法获取变更后的完整数据。
- ❌ 增量同步不能忽略 DDL（表结构变更），需要 Canal 配置支持 DDL 解析，并触发索引重建。
- ❌ 全量同步时必须停服或暂停写入，否则会导致数据不一致；正确的做法是：先停写→全量同步→校验一致→恢复写入。

---

## 第15题：设计一个高可用架构

### 题目

WMS 系统（wms-backend、inventory-service-backend、item-master-backend）需要保证高可用，支持多地部署（多活）、故障自动转移、服务降级熔断、监控告警。请设计完整的高可用架构方案，说明：多活部署方案（同城双活/异地多活）、故障转移（健康检查 + DNS/负载均衡切换）、降级熔断（Sentinel/Hystrix）、监控告警（Prometheus + Grafana + 链路追踪）。

### 核心答案

**高可用架构总览：**

```
                    ┌─────────────────┐
                    │   全局负载均衡    │  DNSPod / AWS Route53
                    │   (GSLB)        │
                    └────────┬────────┘
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
      ┌──────────┐   ┌──────────┐   ┌──────────┐
      │  主站点A  │   │  主站点B  │   │  灾备站点C│
      │(同城双活) │   │(异地多活) │   │ (冷备)   │
      │          │   │          │   │          │
      │ wms-app  │   │ wms-app  │   │          │
      │ inventory│   │ inventory│   │          │
      │ item-mast│   │ item-mast│   │          │
      │ MySQL主库│   │ MySQL主库│   │          │
      │ Redis集群│   │ Redis集群│   │          │
      └──────────┘   └──────────┘   └──────────┘
              │              │
              └──────┬───────┘
                     │
            Binlog同步 / DTC事务
```

**多活部署方案：**

| 方案 | 原理 | RTO | RPO | 复杂度 |
|------|------|-----|-----|--------|
| 同城双活 | 两机房同地域，延迟 < 1ms，共享 MySQL 主库 | 分钟级 | 秒级 | 中 |
| 异地多活 | 两地机房独立 MySQL，Binlog 同步 | 分钟级 | 秒级~分钟级 | 高 |
| 两地三中心 | 同城双活 + 异地灾备 | 小时级 | 小时级 | 中 |
| 冷备灾备 | 主挂了切到灾备，恢复时间长 | 小时级 | 天级 | 低 |

**故障转移（健康检查 + 自动切换）：**

```yaml
# Kubernetes 健康检查配置
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3  # 连续3次失败则重启Pod

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  failureThreshold: 5  # 连续5次失败则踢出Service

# Nginx upstream 健康检查
upstream wms-backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;  # 备份节点
}
```

**服务降级熔断（Sentinel）：**

```java
// Sentinel 流控规则：QPS > 1000 则拒绝
@RestController
public class InventoryController {
    @SentinelResource(value = "deductStock",
        blockHandler = "deductStockBlockHandler",
        fallback = "deductStockFallback")
    public Response deductStock(Long skuId, Integer qty) {
        return inventoryService.deductStock(skuId, qty);
    }

    // 触发限流时降级
    public Response deductStockBlockHandler(Long skuId, Integer qty,
                                            BlockException ex) {
        return Response.fail("系统繁忙，请稍后再试");
    }

    // 触发异常时降级
    public Response deductStockFallback(Long skuId, Integer qty,
                                        Throwable ex) {
        // 降级方案：跳过库存扣减，直接返回成功（允许后续对账）
        return Response.success("降级处理: 库存已预占");
    }
}
```

**监控告警体系：**

| 层级 | 工具 | 监控内容 |
|------|------|---------|
| 基础设施 | Prometheus + NodeExporter | CPU/内存/磁盘/网络 |
| 中间件 | Redis/ES/Kafka Exporter | 内存使用、QPS、延迟、错误率 |
| 应用层 | Spring Boot Actuator + Micrometer | JVM GC、线程池、HTTP QPS/延迟 |
| 链路追踪 | SkyWalking / Jaeger | 全链路请求耗时、调用链 |
| 日志 | ELK (Elasticsearch+Logstash+Kibana) | 错误日志、关键词告警 |
| 告警 | AlertManager + 飞书/钉钉 | 阈值触发自动通知 |

**核心告警规则（Prometheus）：**

```yaml
groups:
  - name: inventory-alerts
    rules:
    - alert: InventoryDeductHighLatency
      expr: histogram_quantile(0.99,
        rate(inventory_deduct_seconds_bucket[5m])) > 0.5
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "库存扣减P99延迟超过500ms"

    - alert: InventoryStockAbnormal
      expr: abs(inventory_redis_stock - inventory_db_stock) > 100
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Redis与DB库存差值异常"
```

### 追问方向

1. **多活架构中，Session 怎么管理？** —— 答：不能粘性 Session（多节点无效）；必须使用 Redis Session 共享或 JWT 无状态 Token。
2. **Sentinel 和 Hystrix 怎么选？** —— 答：Sentinel 是阿里巴巴开源，与 Spring Cloud Alibaba 集成更好，功能更全面（流控+熔断+系统保护）；Hystrix 已停止维护，不推荐新项目使用。
3. **故障转移时，如何保证不重复扣减库存？** —— 答：幂等设计（每笔请求带唯一幂等ID）+ 分布式锁 + 状态机校验，三道防线兜底。

### 避坑提示

- ❌ 多活部署不能简单理解为"多部署几套"，数据一致性、流量路由、故障判定都需要精心设计，否则多活反而增加复杂度。
- ❌ 熔断降级不能返回空数据（`null`），必须有明确的降级响应，否则下游无法判断是超时还是降级。
- ❌ 监控告警不能只有大屏展示，必须配置值班制度和 On-Call 机制，否则告警发出无人处理等于没有。

---

> **面试官点评维度**
> - **架构完整性**：是否能画出完整的数据流和调用链
> - **技术选型能力**：是否理解各方案优劣，能给出明确取舍理由
> - **结合项目**：是否联系 WMS 系统真实场景（库位、库存、订单）
> - **工程落地**：是否有关键代码/配置细节，而非只会背概念
> - **问题应变**：追问方向能否答出细节，避坑提示是否踩过真实坑

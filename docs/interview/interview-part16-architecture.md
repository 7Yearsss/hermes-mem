# 架构设计面试题（上）

> 本系列面试题面向 Java 开发者，结合 WMS 仓库管理系统实际场景，考察架构设计能力。

---

## 第 1 题：单体架构 vs 微服务架构 — 拆分时机与微服务划分原则

### 题目

你们 WMS 项目早期是单体架构，后来演进到了多服务架构。请描述：**什么情况下应该从单体拆分为微服务？拆分时如何划分服务边界？领域驱动设计（DDD）在其中起到了什么作用？**

### 核心答案

**拆分时机判断：**

| 征兆 | 说明 |
|------|------|
| 代码规模超过 50 万行 | 编译时间超过 10 分钟，团队超过 20 人 |
| 需求变更频繁冲突 | 多团队在同一个代码库上并行开发，merge 冲突频繁 |
| 独立发布需求强 | 库存模块和订单模块发布节奏完全不同 |
| 技术栈需要多元化 | 某些模块需要 Python/ML，某些需要 Java 高并发 |
| 可用性要求差异大 | 核心链路（如出库）要求 99.99%，而报表仅需 99% |

**微服务划分原则（DDD 限界上下文）：**

1. **业务边界优先于技术边界**：按业务能力（Domain）划分，而非按分层（Controller/Service/DAO）
2. **高内聚低耦合**：相关功能放一起（如库存锁定、出库校验、库存扣减），跨上下文通过 API/消息交互
3. **独立数据存储**：每个服务拥有独立数据库 schema，不共享表
4. **上下文边界识别**：
   - 仓库上下文（Warehouse）：库位管理、入库策略
   - 库存上下文（Inventory）：库存查询、冻结、扣减
   - 订单上下文（Order）：波次生成、出库单
   - 报表上下文（Report）：统计指标、数据聚合

**DDD 在 WMS 中的实际应用：**
- 库存上下文的核心聚合根是 `InventoryItem`（库存项），包含 SKU、仓库、可用数量、冻结数量
- 库存扣减作为聚合内的领域事件 `InventoryDeductedEvent`，触发订单状态变更
- 跨上下文通过 `ApplicationService` 编排，而非在领域层直接调用

### 追问方向

1. 你们 WMS 拆分过程中，遇到最大的"拆不开"的耦合是什么？怎么解决的？
2. 服务拆分后，数据一致性是如何保证的？有没有出现"分布式事务"的坑？
3. 如果让你重新设计，会不会把某些服务合并？为什么？

### 避坑提示

- ❌ **不要**说"因为别人都在用微服务，所以我们要拆"——没有业务驱动就不要拆
- ❌ **不要**把 DDD 当成"新潮词汇"，实际项目可能只是按表拆分服务，而不是按业务边界
- ✅ **要**能说清楚拆分前后的**量化指标**（发布频率、团队协作效率、故障隔离范围）

---

## 第 2 题：服务治理 — 注册中心、配置中心、路由网关、限流熔断

### 题目

WMS 微服务架构中涉及库存服务、订单服务、调度服务等多个节点。请描述：**如何实现完整的服务治理体系？注册中心、配置中心、路由网关、限流熔断各自解决什么问题？**

### 核心答案

**四类组件的职责对比：**

| 组件 | 核心问题 | 选型举例 | 关键能力 |
|------|---------|---------|---------|
| 注册中心 | 服务发现 | Nacos/Eureka/Zookeeper | 健康检查、服务地址管理 |
| 配置中心 | 动态配置 | Nacos Config/Apollo | 热更新、环境隔离、版本管理 |
| 路由网关 | 流量入口 | Spring Cloud Gateway/Kong | 路由转发、协议转换、认证 |
| 限流熔断 | 稳定性 | Sentinel/Hystrix | 流控、降级、熔断恢复 |

**注册中心（Nacos 为例）：**
```
服务启动 → 注册到 Nacos → 心跳保活 → 下线时主动注销
客户端：每 10s 拉取服务列表，缓存 + 主动通知
```

**配置中心：**
- WMS 中如库存阈值、调度策略参数需要动态调整
- 配置变更后通过 HTTP Long Polling 推送到客户端，无需重启服务

**路由网关（Spring Cloud Gateway）：**
```yaml
routes:
  - id: inventory-service
    uri: lb://inventory-service
    predicates:
      - Path=/inventory/**
    filters:
      - StripPrefix=1
      - RequestRateLimiter=10/s
```

**限流熔断（Sentinel）：**
- 流控：QPS、并发线程数、调用关系
- 熔断：RT 超过阈值自动熔断，恢复后逐步放流量
- 降级：兜底方法，如库存查询降级到走缓存

### 追问方向

1. Nacos 的注册心跳间隔是多少？服务下线后消费者多久能感知？
2. 配置中心的配置变更如何保证事务一致性（配置更新和代码逻辑同步）？
3. Sentinel 的滑动窗口算法是什么？和 Hystrix 的线程池隔离有什么区别？

### 避坑提示

- ❌ **不要**只背概念，要能画出服务治理的**完整数据流图**
- ❌ **不要**说"所有服务都往注册中心注册"，要说明哪些服务需要注册、哪些不需要（如定时任务节点）
- ✅ **要**能说出实际配置（如 Nacos 心跳超时、Sentinel 熔断阈值）和为什么这么配置

---

## 第 3 题：分布式事务 — 2PC/TCC/可靠消息/最大努力通知

### 题目

WMS 中"创建出库单→冻结库存→生成调度任务"涉及多个服务。请描述：**分布式事务的几种主流解决方案是什么？各自的适用场景是什么？**

### 核心答案

**四种方案对比：**

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **2PC** | 两阶段提交（Prepare→Commit） | 强一致 | 同步阻塞、单点故障、数据锁定 | 不推荐，理论方案 |
| **TCC** | Try-Confirm-Cancel 补偿 | 异步化、性能高 | 业务侵入、幂等性要求高 | 强一致性场景：库存扣减 |
| **可靠消息** | 本地消息表 + 消息队列 | 最终一致、异步 | 实现复杂、延迟 | 异步链路：订单→物流 |
| **最大努力通知** | 定期重试 + 人工兜底 | 实现简单 | 最终一致、可能丢数据 | 非核心业务：日志同步 |

**WMS 中的实际应用：**

**场景：出库单创建时冻结库存**
```
TCC 方案：
Try：库存服务预留库存（freeze_count++）
Confirm：出库单确认后扣减冻结、可用库存同步变化
Cancel：出库单取消后释放冻结库存
```

**场景：出库完成通知下游物流系统**
```
可靠消息方案：
1. 本地事务入库（出库单 + 消息记录同为一条事务）
2. 定时任务扫描待发送消息 → 发送到 MQ
3. 消费者 ACK 后更新消息状态
4. 生产端重试 + 消费端幂等
```

**核心实现细节：**
- 本地消息表：`outbox` 模式保障消息可靠性
- 消息状态机：PENDING → SENT → CONFIRMED / FAILED
- 幂等处理：消息唯一 ID + 去重表

### 追问方向

1. TCC 的 Try 失败了怎么办？Confirm 和 Cancel 失败了如何处理？
2. 可靠消息方案中，如果消息已经发送但本地事务还没提交，怎么处理？
3. Seata 的 AT 模式和 TCC 模式有什么区别？你们用过吗？

### 避坑提示

- ❌ **不要**说"为了保证强一致性就用 2PC"——2PC 在生产环境几乎是禁忌
- ❌ **不要**忽略幂等性设计——每一种方案都必须回答"如何保证幂等"
- ✅ **要**能结合 WMS 场景说明为什么选某种方案（如库存用 TCC、通知用可靠消息）

---

## 第 4 题：CAP 定理与 BASE 理论

### 题目

分布式系统中存在 CAP 不可能三角。请描述：**CAP 定理是什么？BASE 理论如何指导实际系统设计？WMS 的库存一致性是如何在 CAP 之间做权衡的？**

### 核心答案

**CAP 定理：**

```
CAP 不可能三角：
一个分布式系统最多同时满足以下两个：
- Consistency（一致性）：所有节点看到同一份数据
- Availability（可用性）：每个请求都能获得响应
- Partition Tolerance（分区容错）：网络分区时系统仍能运行

现实：分区必然发生 → 实际上只能在 C 和 A 之间权衡
```

**两种典型策略：**

| 策略 | 选择 | 典型场景 |
|------|------|---------|
| **CP 系统** | 放弃可用性 | Zookeeper、HBase、Redis Sentinel |
| **AP 系统** | 放弃强一致 | Eureka、Nacos（默认 AP）、Cassandra |

**BASE 理论：**

```
BASE = Basically Available + Soft State + Eventually Consistent

核心思想：
- 允许系统在不同阶段呈现不同状态（软状态）
- 在一定时间窗口后最终达到一致
- 通过牺牲强一致性换取可用性和性能
```

**WMS 库存一致性的实际权衡：**

```
强一致场景（选 CP）：
出库扣减库存 → 库存必须准确，不能超卖
→ 解决方案：分布式锁（Redisson）+ 乐观版本号

最终一致场景（选 AP）：
库存查询展示 → 允许短暂不一致（缓存与 DB 差几秒）
→ 解决方案：读请求走本地缓存 + 异步同步

柔性事务实现：
1. 库存服务先行冻结（本地事务）
2. 出库确认后真正扣减（异步 + 幂等补偿）
3. 差异通过定时对账任务修复
```

### 追问方向

1. Redis Cluster 是 AP 还是 CP？为什么？
2. Nacos 的 CP/AP 模式切换是怎么回事？你们用的哪种？
3. 如果要实现"库存不超卖"且"高可用"，怎么设计？有没有 trade-off？

### 避坑提示

- ❌ **不要**说"CAP 三个都能满足"——这是不可能的，要承认 trade-off
- ❌ **不要**把 BASE 等同于"不要一致性"，BASE 的"最终一致"是有明确时间窗口的
- ✅ **要**能针对具体业务场景（如 WMS 库存）说出 C 和 A 的权衡理由

---

## 第 5 题：领域驱动设计（DDD）

### 题目

请解释 DDD 中的核心概念：**限界上下文（Bounded Context）、聚合根（Aggregate Root）、实体（Entity）、值对象（Value Object）、领域事件（Domain Event）**，并结合 WMS 库存场景举例说明。

### 核心答案

**核心概念定义：**

| 概念 | 定义 | 标识特征 |
|------|------|---------|
| **限界上下文** | 业务边界的清晰划分，每个上下文有独立的领域模型 | 独立数据库、独立团队、独立发布 |
| **聚合根** | 聚合内实体的根，负责外部引用和状态变更 | ID 全局唯一，外部只能通过它修改聚合 |
| **实体** | 有唯一标识，生命周期内状态可变 | 如 `Order`、`InventoryItem` |
| **值对象** | 无唯一标识，属性值决定相等性，不可变 | 如 `Money`、`Address`、`Quantity` |
| **领域事件** | 领域中发生的业务事件，用于解耦 | `InventoryDeductedEvent`、`OrderCreatedEvent` |

**WMS 库存上下文 DDD 建模：**

```java
// 聚合根：库存项
public class InventoryItem {
    private InventoryItemId id;          // 唯一标识（聚合根 ID）
    private SkuId skuId;                  // 关联 SKU
    private WarehouseId warehouseId;     // 仓库 ID
    private Quantity availableQty;       // 值对象：可用数量
    private Quantity frozenQty;          // 值对象：冻结数量
    
    // 聚合根方法：冻结库存
    public void freeze(Quantity qty) {
        if (availableQty.lessThan(qty)) {
            throw new InsufficientStockException(...);
        }
        availableQty = availableQty.subtract(qty);
        frozenQty = frozenQty.add(qty);
        
        // 发布领域事件
        DomainEvents.publish(
            new InventoryFrozenEvent(this.id, qty, ...)
        );
    }
}

// 值对象：数量（不可变，运算后返回新实例）
public class Quantity {
    private final int value;
    
    public Quantity subtract(Quantity other) {
        return new Quantity(this.value - other.value);
    }
}
```

**限界上下文划分（WMS）：**
```
┌──────────────────────────────────────────────┐
│         仓库管理上下文 (Warehouse)              │
│   库位、库区、巷道、存储策略                      │
├──────────────────────────────────────────────┤
│         库存上下文 (Inventory)                  │
│   库存项、批次、库存快照、冻结/扣减               │
├──────────────────────────────────────────────┤
│         订单上下文 (Order)                      │
│   出库单、入库单、波次、调度任务                   │
├──────────────────────────────────────────────┤
│         主数据上下文 (Item Master)              │
│   SKU、商品规格、供应商、计量单位                 │
└──────────────────────────────────────────────┘
```

### 追问方向

1. 聚合根和实体在代码层面的区别是什么？聚合根内部的实体可以直接被外部修改吗？
2. 领域事件是如何传递给其他上下文的？通过 MQ 还是直接调用？
3. 你们项目有没有"聚合根膨胀"的问题？一个聚合根承载了太多业务逻辑怎么办？

### 避坑提示

- ❌ **不要**把 DDD 当成"给代码加几个注解"——DDD 是思维方式的转变，核心是业务边界
- ❌ **不要**在聚合根里写太多方法导致"上帝对象"，一个聚合根应该只管理一个核心实体
- ✅ **要**能用 WMS 的具体场景（如出库流程）画出完整的领域模型图和上下文交互图

---

## 第 6 题：CQRS 模式

### 题目

请解释 **CQRS（Command Query Responsibility Segregation）** 模式：**什么是命令查询分离？读写分离模型如何设计？事件溯源（Event Sourcing）在其中扮演什么角色？**

### 核心答案

**CQRS 核心思想：**

```
传统架构：同一个模型用于读写
┌─────────────┐
│  Repository │ ←── 同一个 Order 实体
│  OrderService│
└─────────────┘
    ↓ 读         ↓ 写
  OrderDTO    OrderEntity

CQRS：读模型和写模型分离
┌──────────────┐      ┌──────────────┐
│ Command Side │      │  Query Side  │
│  (写优化)    │  →   │  (读优化)    │
│ OrderService │ 事件  │ OrderReadDAO │
│ OrderEntity  │      │ OrderDetailDTO│
└──────────────┘      └──────────────┘
```

**WMS 中的 CQRS 应用：**

**写模型（命令端）：**
```java
// 出库单命令处理
public class CreateOutboundCommand {
    private OutboundOrderId orderId;
    private List<OutboundItem> items;
}

// 命令处理器
public class OutboundCommandHandler {
    public void handle(CreateOutboundCommand cmd) {
        OutboundOrder order = new OutboundOrder(...);
        orderRepository.save(order);
        
        // 发布事件到 MQ
        eventBus.publish(new OutboundOrderCreatedEvent(order));
    }
}
```

**读模型（查询端）：**
```java
// 读模型：通过 MQ 异步同步到读库
@RabbitListener
public void onOutboundOrderCreated(OutboundOrderCreatedEvent event) {
    // 构建读模型 DTO
    OutboundOrderSummary summary = OutboundOrderSummary.builder()
        .orderId(event.getOrderId())
        .status(event.getStatus())
        .itemCount(event.getItems().size())
        .createdAt(event.getCreatedAt())
        .warehouseName(warehouseCache.get(event.getWarehouseId()))
        .build();
    
    readRepository.save(summary); // 写入读库
}
```

**读库设计（MySQL + Elasticsearch）：**
- MySQL 读库：结构化查询（按状态查、按时间范围查）
- ES：模糊搜索、复杂条件组合查询（如多仓库多状态组合）

**Event Sourcing（事件溯源）：**

```
传统：只存储当前状态
Order → {status: COMPLETED, qty: 100}

Event Sourcing：存储所有状态变更历史
OrderCreated → {qty: 100}
InventoryFrozen → {qty: 50}
InventoryDeducted → {qty: 50}
OrderCompleted → {}

通过事件重放可以还原任意时间点的状态
```

### 追问方向

1. CQRS 读写模型的数据一致性如何保证？是最终一致还是准实时？
2. Event Sourcing 的事件存储如何做分页查询？快照机制是什么？
3. 如果 CQRS 的两个模型之一挂了，如何恢复数据？

### 避坑提示

- ❌ **不要**在简单场景下强行上 CQRS——CQRS 适合复杂业务，写少读多或读写模型差异大的场景
- ❌ **不要**忽略命令端和查询端的幂等性——事件重复投递时读模型要能正确处理
- ✅ **要**能说明 CQRS 相比传统方案的**性能收益**（如复杂查询下读 QPS 提升 10 倍）

---

## 第 7 题：事件驱动架构（EDA）

### 题目

请解释 **事件驱动架构（Event-Driven Architecture）**：**事件总线如何设计？Event Sourcing 在 CQRS 中如何应用？EDA 架构有哪些坑？**

### 核心答案

**EDA 三要素：**

```
Event（事件）："发生了什么"，如 OrderCreated
Command（命令）："要做什么"，如 FreezeInventory
Query（查询）："要什么数据"

事件驱动：用事件解耦生产者和消费者
生产者 → 事件 → 事件总线 → 消费者
```

**事件总线设计：**

```java
// 统一事件接口
public interface DomainEvent {
    EventId getId();        // 事件唯一 ID（雪花算法）
    LocalDateTime getOccurredOn();
    String getAggregateId(); // 关联聚合根 ID
}

// 事件总线
public interface EventBus {
    void publish(DomainEvent event);
    void subscribe(Class<? extends DomainEvent> eventType, 
                   EventHandler handler);
}

// Spring 实现（ApplicationEventPublisher）
@Service
public class SpringEventBus implements EventBus {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    @Override
    public void publish(DomainEvent event) {
        publisher.publishEvent(event);
    }
}
```

**EDA 在 WMS 中的实际流程：**

```
出库单创建
    ↓ DomainEvent: OutboundOrderCreated
    ↓
┌─────────────────────────────────────────┐
│  事件总线（RabbitMQ Topic Exchange）      │
├─────────────────────────────────────────┤
│  消费者1: 库存服务 → 冻结库存              │
│  消费者2: 调度服务 → 生成波次任务           │
│  消费者3: 消息服务 → 发送通知              │
│  消费者4: 审计服务 → 记录操作日志           │
└─────────────────────────────────────────┘
```

**Event Sourcing + CQRS 组合：**

```
命令端：
Order.create() → 存储 OrderEvent[]（事件流）
                  不存储 Order 实体本身

查询端：
订阅 OrderEvent[] → 重放事件 → 构建 OrderReadModel
                  → 提供多种查询视图（订单详情、统计 Dashboard）
```

**EDA 的坑：**

| 坑 | 说明 | 解决方案 |
|----|------|---------|
| 事件乱序 | 网络问题导致事件到达顺序和发送顺序不一致 | 事件版本号 + 重排序 |
| 重复消费 | 消费者重启导致重复处理 | 幂等表 + 唯一事件 ID |
| 事件膨胀 | 长期运行后事件数量巨大，重放慢 | 快照机制（每 1000 个事件打一个快照） |
| 死事件 | 消费者永久失败，事件无法处理 | DLQ（死信队列）+ 人工干预 |

### 追问方向

1. 你们 WMS 的事件总线是基于 RabbitMQ 还是 Kafka？为什么？
2. 如果消费者处理失败，事件如何重试？重试次数用完后呢？
3. 事件 Schema 变更怎么办（如增加字段）？有没有版本兼容策略？

### 避坑提示

- ❌ **不要**把所有事件都当成"最终一致"处理——有些事件是强同步的，不能为了解耦而解耦
- ❌ **不要**忽略事件的**幂等性设计**，这是 EDA 的生死线
- ✅ **要**能画出完整的事件流图，包括事件的生产、存储、消费、死信处理全链路

---

## 第 8 题：API 网关设计

### 题目

WMS 对外暴露了 REST API，对内服务间也有 RPC 调用。请描述 **API 网关的职责**：**路由转发、协议转换、认证鉴权、日志审计如何实现？**

### 核心答案

**API 网关核心职责：**

```
┌────────────────────────────────────────────────┐
│                 API Gateway                    │
├────────────────────────────────────────────────┤
│  路由转发    │ 协议转换   │ 认证鉴权  │ 日志审计  │
│  /order/**  │ REST→gRPC │ JWT验证   │ 请求日志  │
│  → order-svc│ JSON→Proto│ 限流      │ 链路追踪  │
└────────────────────────────────────────────────┘
```

**Spring Cloud Gateway 配置示例：**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: inventory-service
          uri: lb://inventory-service  # 负载均衡到服务
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=2             # 去掉 /api/inventory 前缀
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
      
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: GET,POST,PUT,DELETE
```

**认证鉴权实现：**

```java
// 全局 Filter 校验 JWT
@Component
public class AuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, 
                             GatewayFilterChain chain) {
        String token = exchange.getRequest()
            .getHeaders().getFirst("Authorization");
        
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(
                HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // 验证 JWT，提取 userId、roles
        Claims claims = jwtUtil.verify(
            token.substring(7));
        
        // 将用户信息传递到下游服务（Header 透传）
        ServerRequest modifiedRequest = ServerRequest.from(exchange)
            .mutate()
            .header("X-User-Id", claims.getUserId())
            .header("X-User-Roles", String.join(",", claims.getRoles()))
            .build();
        
        return chain.filter(modifiedRequest);
    }
}
```

**日志审计设计：**

```java
// 审计日志：记录每个请求的详细信息
public class AuditLogFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(...) {
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();
        
        return chain.filter(exchange).then(
            Mono.fromRunnable(() -> {
                long cost = System.currentTimeMillis() - startTime;
                auditLogService.save(AuditLog.builder()
                    .requestId(requestId)
                    .path(exchange.getRequest().getPath())
                    .method(exchange.getRequest().getMethod())
                    .userId(getUserId(exchange))
                    .responseStatus(
                        exchange.getResponse().getStatusCode())
                    .costMs(cost)
                    .build());
            })
        );
    }
}
```

### 追问方向

1. 网关如何感知下游服务的健康状态？健康检查的频率和阈值是多少？
2. 如果网关本身挂了怎么办？如何实现网关的高可用？
3. gRPC 和 REST 混用时，协议转换的性能损耗有多少？有测过吗？

### 避坑提示

- ❌ **不要**把业务逻辑放在网关层——网关只做横切关注点（路由、认证、日志），业务逻辑下沉到服务
- ❌ **不要**忽略网关的**性能测试**——网关是所有流量的入口，延迟增加 10ms 对整体影响很大
- ✅ **要**能说明网关的限流算法（令牌桶/滑动窗口）和配置依据（如何确定 100 QPS）

---

## 第 9 题：消息队列在架构中的应用

### 题目

WMS 中大量使用消息队列（如库存变更通知下游调度系统）。请描述：**消息队列在架构中的核心价值是什么？异步解耦、流量削峰、数据一致性如何实现？**

### 核心答案

**消息队列三大核心价值：**

```
1. 异步解耦：生产者不关心消费者何时处理
   OrderService → MQ → InventoryService
                 → MQ → SchedulerService
                 → MQ → NotificationService

2. 流量削峰：平缓突发流量，保护下游系统
   瞬时 10000 请求
   ↓ 消息队列（缓冲）
   ↓ 匀速消费（每s 100条）
   InventoryService（正常处理）

3. 数据一致性：最终一致性替代强一致性
   本地事务 + MQ 消息 = 可靠消息模式
```

**RabbitMQ 核心配置（WMS 场景）：**

```java
@Configuration
public class RabbitMQConfig {
    
    // 交换机定义
    @Bean
    public TopicExchange wmsExchange() {
        return ExchangeBuilder
            .topicExchange("wms.topic")
            .durable(true)
            .build();
    }
    
    // 队列定义
    @Bean
    public Queue inventoryFreezeQueue() {
        return QueueBuilder
            .durable("inventory.freeze.queue")
            .withArgument("x-dead-letter-exchange", "wms.dlx")
            .withArgument("x-dead-letter-routing-key", "dlq.inventory")
            .build();
    }
    
    // 绑定
    @Bean
    public Binding inventoryFreezeBinding() {
        return BindingBuilder
            .bind(inventoryFreezeQueue())
            .to(wmsExchange())
            .with("inventory.freeze.#");
    }
}
```

**可靠消息实现（本地消息表 + MQ）：**

```java
// 出库单服务
@Transactional
public void createOutboundOrder(CreateOutboundOrderCmd cmd) {
    OutboundOrder order = new OutboundOrder(...);
    outboundOrderRepository.save(order);
    
    // 本地消息表（和订单同一事务）
    OutboxMessage message = OutboxMessage.builder()
        .eventType("InventoryFreezeRequired")
        .payload(JSON.toJSONString(cmd))
        .status("PENDING")
        .build();
    outboxRepository.save(message);
}

// 定时任务：扫描 PENDING 消息，发送到 MQ
@Scheduled(fixedDelay = 1000)
public void pollOutboxMessages() {
    List<OutboxMessage> messages = 
        outboxRepository.findByStatusPending(100);
    for (OutboxMessage msg : messages) {
        try {
            rabbitTemplate.convertAndSend(
                msg.getEventType(), msg.getPayload());
            msg.markAsSent();
            outboxRepository.save(msg);
        } catch (Exception e) {
            msg.incrementRetryCount();
            outboxRepository.save(msg);
        }
    }
}
```

**流量削峰配置：**

```yaml
# 消费者端限流
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 5        # 初始消费者数量
        max-concurrency: 20   # 最大消费者数量
        prefetch: 10          # 预取数量（控制消费速率）
        acknowledge-mode: manual  # 手动 ACK
```

### 追问方向

1. RabbitMQ 的消息持久化和 ACK 机制是什么？如何在消费者端正确处理？
2. 如果 MQ 挂了，生产者如何保证消息不丢失？
3. Kafka 和 RabbitMQ 在使用场景上有什么区别？你们 WMS 为什么选 RabbitMQ？

### 避坑提示

- ❌ **不要**把 MQ 当成数据库——MQ 是传输通道，不是存储，不要在消息体里放太多数据
- ❌ **不要**忽略消息的**幂等消费**——这是最容易出生产事故的地方
- ✅ **要**能画出消息的完整生命周期图（生产→存储→消费→ACK→DLQ）

---

## 第 10 题：数据库读写分离

### 题目

WMS 读请求量远大于写请求量（库存查询、库位查询）。请描述：**数据库读写分离如何实现？主从同步延迟如何处理？ShardingSphere 如何做路由策略？**

### 核心答案

**读写分离架构：**

```
┌────────────────────────────────────────────┐
│                   应用层                    │
│   DataSourceRouter（ShardingSphere）       │
│         ↓ 路由读          ↓ 路由写          │
│   ┌─────────────┐    ┌─────────────┐       │
│   │  读库 DS0   │    │  写库 DS1   │       │
│   │  (只读副本) │    │  (主库)     │       │
│   └──────┬──────┘    └──────┬──────┘       │
│          │                  │              │
│          └──────┬───────────┘              │
│                 ↓                          │
│          MySQL 主从同步                      │
└────────────────────────────────────────────┘
```

**ShardingSphere-JDBC 配置：**

```yaml
spring:
  shardingsphere:
    datasource:
      ds-master:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://master:3306/wms
        username: wms
      ds-slave-0:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://slave0:3306/wms
      ds-slave-1:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://slave1:3306/wms
    
    rules:
      read-write-splitting:
        data-sources:
          prds:
            type: Static
            props:
              write-data-source-name: ds-master
              read-data-source-names: ds-slave-0,ds-slave-1
            load-balancer:
              type: ROUND_ROBIN  # 读负载均衡策略
```

**主从同步延迟问题及解决方案：**

| 问题 | 解决方案 |
|------|---------|
| 读写都在主库（延迟感知不到） | 强制读走从库，但关键读（如库存下单前）走主库 |
| 从库延迟导致超卖 | 写后立即读走主库（Read-Your-Writes 一致性） |
| 延迟不确定 | 业务层面接受最终一致，监控从库 `seconds_behind_master` |
| 读写分离后连接数翻倍 | 使用 HikariCP 连接池，合理配置核心连接数 |

**读写分离的代码策略：**

```java
// 方法级别指定数据源
@ReadOnlyConnection
public List<InventoryItem> queryByWarehouse(WarehouseId whId) {
    // 强制走从库
    return inventoryRepository.findByWarehouseId(whId);
}

public void freezeInventory(InventoryFreezeCmd cmd) {
    // 走主库
    inventoryRepository.save(cmd.toEntity());
}
```

### 追问方向

1. MySQL 主从同步的原理是什么？基于 binlog 的三种模式（Statement/Row/Mixed）区别？
2. ShardingSphere-Proxy 和 ShardingSphere-JDBC 的区别是什么？什么时候用哪个？
3. 如果主库挂了，读写分离如何切换？RPO 是多少？

### 避坑提示

- ❌ **不要**默认"所有读都走从库"——下单扣库存这种强一致读必须走主库
- ❌ **不要**忽略连接池调优——主从各一套连接池，参数不一致会导致问题
- ✅ **要**能说出你们具体的延迟监控方案（如 Canal 订阅 binlog + Prometheus 报警）

---

## 第 11 题：分库分表

### 题目

WMS 数据量增长很快（订单表超过 5000 万行）。请描述：**分库分表策略如何设计？水平拆分和垂直拆分的区别是什么？分片键如何选择？跨分片查询如何处理？**

### 核心答案

**拆分策略对比：**

| 类型 | 拆分维度 | 适用场景 | 示例 |
|------|---------|---------|------|
| **垂直拆分** | 按业务/列 | 表字段过多、冷热数据分离 | 订单表拆为订单主表+订单详情表 |
| **水平拆分** | 按行 | 单表数据量过亿 | 按用户ID/时间分片 |

**ShardingSphere 分库分表配置：**

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          t_outbound_order:
            actual-data-nodes: ds-${0..1}.t_outbound_order_${0..7}
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: order_inline
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake
        
        sharding-algorithms:
          order_inline:
            type: INLINE
            props:
              algorithm-expression: |
                t_outbound_order_${order_id % 8}
```

**分片键选择原则：**

```
✓ 分片键要均匀分布（如 user_id、order_id）
✗ 避免单调递增/递减（如时间戳 auto_increment）导致热点分片

WMS 订单表分片键选择：
- order_id（雪花算法）→ 8 个分片均匀分布
- 时间范围分片 → 适合查询历史订单（按月/季度归档）
```

**跨分片查询处理策略：**

| 场景 | 方案 |
|------|------|
| 聚合查询（COUNT/SUM） | 广播查询（每个分片查一次，再汇总） |
| 跨分片 JOIN | 冗余字段（如把 user_name 冗余到订单表） |
| 分页 | 先在每个分片内部排序，再归并排序 |
| 唯一ID查询 | 路由到特定分片，无需广播 |

```java
// ShardingSphere 广播查询示例
List<Long> totalCounts = new ArrayList<>();
for (int i = 0; i < 8; i++) {
    Long count = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM t_outbound_order_" + i 
        + " WHERE warehouse_id = ?", 
        Long.class, warehouseId);
    totalCounts.add(count);
}
long total = totalCounts.stream().mapToLong(Long::longValue).sum();
```

**分库分表后的扩容方案：**

```
方案1：翻倍扩容（如从 8 表→16 表）
       数据迁移：一半数据重新分配
       优点：扩容平滑
       缺点：浪费一半资源

方案2：一致性哈希（如 Redis）
       虚拟节点映射到物理节点
       优点：迁移量最小
       缺点：实现复杂

实际 WMS 方案：按年份+仓库ID 复合分片
- 历史库：按年归档（2023、2024、2025）
- 当前库：按 warehouse_id 分 4 个库
- 扩容时新增年份/仓库，不影响历史数据
```

### 追问方向

1. 分库分表后，如何生成全局唯一ID？雪花算法的时间回拨问题如何处理？
2. 如果一个分片键的数据过热（如某个 SKU 销量特别大）怎么办？
3. ShardingSphere 的强制路由和hint 路由分别在什么场景下使用？

### 避坑提示

- ❌ **不要**过早拆分——分库分表后运维复杂度大幅上升，先考虑冷热分离、归档历史数据
- ❌ **不要**用业务主键做分片键（如 SKU_code），要选择数据分布均匀的字段
- ✅ **要**能画出完整的分片拓扑图，并说明为什么这么分（分片数如何计算）

---

## 第 12 题：缓存架构

### 题目

WMS 中高频查询（库存余量、库位信息）使用 Redis 缓存。请描述：**多级缓存如何设计？缓存穿透、缓存击穿、缓存雪崩分别是什么？如何解决？Redis Cluster 如何部署？**

### 核心答案

**多级缓存架构：**

```
┌──────────────────────────────────────────────────┐
│                   请求                           │
│                   ↓                              │
│  L1: Caffeine 本地缓存（TTL 30s，容量 1000）      │
│       命中 → 直接返回（延迟 < 1ms）                │
│       未命中 →                                   │
│                   ↓                              │
│  L2: Redis 分布式缓存（TTL 5min，集群模式）        │
│       命中 → 回填 L1 → 返回                      │
│       未命中 →                                   │
│                   ↓                              │
│  L3: MySQL 数据库                                │
│       查询 → 回填 L2 + L1 → 返回                 │
└──────────────────────────────────────────────────┘
```

**缓存问题及解决方案：**

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| **缓存穿透** | 查询不存在的数据，每次都打到 DB | 布隆过滤器（BloomFilter）/ 空值缓存 |
| **缓存击穿** | 热点 key 过期瞬间，大量请求打到 DB | 互斥锁（Redisson）/ 永不过期+异步更新 |
| **缓存雪崩** | 大量 key 同时过期 / Redis 宕机 | 过期时间加随机值 / Redis 高可用 + 熔断 |

```java
// 缓存穿透：布隆过滤器
@Configuration
public class BloomFilterConfig {
    @Bean
    public BloomFilter<Long> inventoryBloomFilter() {
        RedisBloomFilter<Long> bloomFilter = 
            new RedisBloomFilter<>(redisTemplate, "inventory:bloom");
        return bloomFilter;
    }
}

public InventoryItem getInventory(Long skuId) {
    if (!bloomFilter.mightContain(skuId)) {
        return null; // 一定不存在，直接返回
    }
    // 继续查询 Redis/MySQL
}

// 缓存击穿：分布式锁
public InventoryItem getInventoryWithLock(Long skuId) {
    String cacheKey = "inventory:" + skuId;
    InventoryItem item = redis.get(cacheKey);
    if (item == null) {
        RLock lock = redissonClient.getLock("lock:" + cacheKey);
        try {
            if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
                // 双重检查
                item = redis.get(cacheKey);
                if (item == null) {
                    item = db.findBySkuId(skuId);
                    redis.setex(cacheKey, 300, item);
                }
            }
        } finally {
            lock.unlock();
        }
    }
    return item;
}

// 缓存雪崩：过期时间随机化
public void setWithRandomExpire(String key, Object value) {
    int baseExpire = 300; // 5分钟
    int randomExpire = ThreadLocalRandom.current().nextInt(60);
    redis.setex(key, baseExpire + randomExpire, value);
}
```

**Redis Cluster 部署：**

```
Redis Cluster：3 主 3 从（最小生产配置）
┌─────────┐     ┌─────────┐     ┌─────────┐
│Master 1 │─────│Master 2 │─────│Master 3 │
│ (slot)  │     │ (slot)  │     │ (slot)  │
│ 0-5460  │     │5461-10922│    │10923-16383│
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
┌────┴────┐     ┌────┴────┐     ┌────┴────┐
│ Slave 1 │     │ Slave 2 │     │ Slave 3 │
│(副本)   │     │(副本)   │     │(副本)   │
└─────────┘     └─────────┘     └─────────┘

数据分布：16384 slots，按 CRC16(key) % 16384 路由
故障转移：主节点宕机 → 从节点自动升主（哨兵机制）
```

### 追问方向

1. Redis Cluster 的槽迁移过程中，客户端如何处理 MOVED 重定向？
2. 大 key（单个 value 超过 10MB）和 hot key 如何发现和处理？
3. Redis 持久化策略（RDB/AOF）如何选择？AOF 重写会影响性能吗？

### 避坑提示

- ❌ **不要**把缓存当数据库用——缓存丢了要从 DB 恢复，不是备份
- ❌ **不要**在 Redis 里存大 JSON（单 value 超过 1MB）——序列化/反序列化成本高
- ✅ **要**能说明 Redis Cluster 和 Codis/Redis Sentinel 的区别，以及为什么选 Cluster

---

## 第 13 题：降级熔断

### 题目

WMS 库存服务依赖外部供应商服务（如商品主数据查询）。请描述：**Sentinel 和 Hystrix 的降级熔断方案是什么？阈值如何配置？自动恢复如何实现？**

### 核心答案

**熔断器三种状态：**

```
CLOSED（正常）→ 请求通过，失败计数
    ↓ 失败率/慢调用率达到阈值
OPEN（熔断）→ 请求直接返回降级结果，不发往下游
    ↓ 过了熔断窗口期（5s）
HALF-OPEN（半开）→ 放少量请求试探下游
    ↓ 成功/失败
CLOSED / OPEN
```

**Sentinel 熔断配置：**

```java
// Sentinel + Spring Cloud Alibaba
@Configuration
public class SentinelConfig {
    
    @Bean
    public Customizer<PathPatternParser> sentinelFallback() {
        return registry -> registry.addCustomizer(
            (sentinelFilter, metricFetcher) -> {
                // 熔断规则：慢调用比例 50%，RT 超过 2000ms，持续 10s
                DegradeRule rule = DegradeRule.builder()
                    .resource("getItemMaster")
                    .grade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType())
                    .param(0.5)  // 50% 慢调用比例触发熔断
                    .count(2000) // RT 阈值 2000ms
                    .timeWindow(10) // 熔断窗口 10s
                    .build();
                DegradeRuleManager.loadRules(Collections.singletonList(rule));
            }
        );
    }
}

// 降级方法
@SentinelResource(value = "getItemMaster", 
                  fallback = "getItemMasterFallback")
public ItemMaster getItemMaster(Long itemId) {
    return itemMasterClient.getById(itemId);
}

public ItemMaster getItemMasterFallback(Long itemId, Throwable t) {
    // 降级逻辑：走本地缓存 / 返回默认数据 / 记录告警
    log.warn("getItemMaster 降级，itemId={}, error={}", 
             itemId, t.getMessage());
    return itemMasterCache.get(itemId) // 从本地缓存兜底
        .orElse(ItemMaster.DEFAULT);
}
```

**Hystrix 线程池隔离：**

```java
@HystrixCommand(
    commandProperties = {
        @HystrixProperty(
            name = "circuitBreaker.requestVolumeThreshold", 
            value = "20"),   // 最小请求量
        @HystrixProperty(
            name = "circuitBreaker.sleepWindowInMilliseconds", 
            value = "5000"), // 熔断后 5s 后尝试恢复
        @HystrixProperty(
            name = "circuitBreaker.errorThresholdPercentage", 
            value = "50")    // 50% 错误率触发熔断
    },
    threadPoolProperties = {
        @HystrixProperty(
            name = "coreSize", 
            value = "10"),   // 线程池核心大小
        @HystrixProperty(
            name = "maxQueueSize", 
            value = "100")   // 队列大小
    },
    fallbackMethod = "getItemMasterFallback"
)
public ItemMaster getItemMaster(Long itemId) {
    return itemMasterClient.getById(itemId);
}
```

**阈值配置依据：**

| 指标 | 配置原则 | 计算方法 |
|------|---------|---------|
| QPS 限流 | 不超过下游系统 TPS 的 80% | 压测得到单实例 TPS = 1000 → 限流 800 |
| 线程数 | CPU 密集型设小，IO 密集型可设大 | CPU 核数 8，IO 密集型线程数 = 8 × 2 = 16 |
| 熔断阈值 | 错误率 > 50% 或 RT > 3s | 根据业务 SLA 要求反推 |
| 熔断窗口 | 10s~60s，太短容易震荡，太长恢复慢 | 经验值 30s |

### 追问方向

1. Sentinel 的滑动窗口算法是什么？和 Hystrix 的计数桶算法有什么区别？
2. 如果下游服务是灰度发布状态，熔断器如何避免误触发？
3. 降级逻辑里可以调用另一个微服务吗？会有什么风险？

### 避坑提示

- ❌ **不要**把熔断阈值设得太低（如 1% 错误率）——正常波动也会触发，生产环境雪崩
- ❌ **不要**在降级方法里做耗时的 DB 查询——降级要快，否则失去降级意义
- ✅ **要**能画出完整的熔断状态机转换图，并说明每个转换的触发条件

---

## 第 14 题：全链路追踪

### 题目

WMS 微服务调用链路复杂（Order → Inventory → Scheduler → Report）。请描述：**traceId 如何生成和传递？采样率如何设置？全链路追踪的存储选型有哪些？**

### 核心答案

**traceId 生成与传递：**

```
traceId 生成规则：
- 格式：{timestamp}-{machineId}-{pid}-{random}
- 示例：20250412-173000-5a2f-8c3d-0014-8b7d
- 生成时机：请求入口（网关/第一个服务）

传递方式：
1. Header 透传（最常见）：traceId 通过 HTTP Header 传递
   Header: X-Trace-Id = abc123
   
2. RPC Context 透传（gRPC/Dubbo）：
   RpcContext.getContext().setAttachment("traceId", traceId)

3. MQ 消息透传：
   消息属性中添加 traceId，消费端提取
```

**Sleuth + Zipkin 集成：**

```yaml
# application.yml
spring:
  sleuth:
    sampler:
      probability: 0.1   # 采样率 10%
      rate: 100           # 每秒最大采样数
  zipkin:
    base-url: http://zipkin-server:9411
    sender:
      type: rabbit  # 通过 MQ 异步上报
```

**手动埋点（跨服务传递）：**

```java
// 方式1：使用 Tracer
@Autowired
private Tracer tracer;

public void doSomething() {
    Span span = tracer.nextSpan().name("inventory-freeze");
    try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
        span.tag("sku_id", skuId.toString());
        
        // 业务逻辑
        inventoryService.freeze(skuId, qty);
        
        span.end();
    }
}

// 方式2：OpenTelemetry（趋势主流）
@Autowired
private OpenTelemetry openTelemetry;

public void doSomething() {
    Span span = openTelemetry.getTracer("wms")
        .spanBuilder("inventory-freeze")
        .startSpan();
    try {
        // 自动注入 traceId 到 HTTP Header
        inventoryClient.freeze(skuId, qty);
    } finally {
        span.end();
    }
}
```

**采样率策略：**

| 场景 | 采样率 | 原因 |
|------|-------|------|
| 调试环境 | 100% | 全量排查问题 |
| 生产日常 | 10%~30% | 减少存储和传输开销 |
| 异常请求 | 100% | 异常链路必须记录 |
| 高流量链路 | 动态调整 | 流量高时降低采样，保护自身 |

**存储选型对比：**

| 存储 | 优点 | 缺点 | 适用规模 |
|------|------|------|---------|
| **Elasticsearch** | 全文搜索、聚合分析强大 | 存储成本高 | 中大型（百亿级） |
| **Cassandra** | 写入强、水平扩展 | 查询能力弱 | 超大型（千亿级） |
| **HBase** | 海量存储、低成本 | 实时查询弱 | 超大型离线分析 |
| **内存（Zipkin In-Memory）** | 零运维 | 数据易丢失、不能持久化 | 开发测试 |

```yaml
# Jaeger 收集器配置（生产推荐）
jaeger:
  collector:
    queue-size: 1000
    buffer-flush-time: 1s
  storage:
    type: elasticsearch
    es:
      server-urls: http://es:9200
      index-prefix: jaeger
      lookback: 30d
```

### 追问方向

1. 采样率 10% 会不会导致关键链路丢失？如何保证重要请求一定被采集？
2. 全链路追踪数据量巨大，如何控制存储成本？有没有冷热分层策略？
3. 如果 traceId 链路断了（如 RPC 调用没有透传），如何定位问题？

### 避坑提示

- ❌ **不要**在日志里打印 traceId 但不用它做检索——traceId 的价值在于串联日志
- ❌ **不要**忽略 MQ 消息里的 traceId 透传——MQ 消费是异步的，没有 traceId 链路就断了
- ✅ **要**能画出完整的数据流：埋点 → 采集 → 上报 → 存储 → 查询

---

## 第 15 题：架构演进史 — 从巨石应用到微服务到云原生

### 题目

请梳理 **WMS 系统的架构演进历程**：**从单体架构到微服务到云原生，技术选型的决策依据是什么？每个阶段的挑战和解决方案是什么？未来演进方向是什么？**

### 核心答案

**架构演进时间线：**

```
阶段1：单体架构（2019-2021）
阶段2：SOA 服务化（2021-2022）
阶段3：微服务（2022-2024）
阶段4：云原生探索（2024-）
```

**阶段 1：巨石应用（Monolith）**

```
┌─────────────────────────────────────────┐
│              WMS 单体应用                 │
├─────────────────────────────────────────┤
│  Controller: OrderController            │
│             InventoryController         │
│             WarehouseController         │
├─────────────────────────────────────────┤
│  Service: OrderService                  │
│          InventoryService               │
│          WarehouseService               │
├─────────────────────────────────────────┤
│  DAO: OrderMapper                       │
│       InventoryMapper                   │
│       WarehouseMapper                   │
├─────────────────────────────────────────┤
│  DB: MySQL (单库，所有表共享)              │
└─────────────────────────────────────────┘

挑战：
- 代码冲突：3 个团队同时改同一个模块
- 发布周期：每周一次全量发布，一个功能推迟所有功能
- 扩容问题：库存查询 QPS 高，拖垮整个系统
```

**阶段 2：SOA 服务化**

```
技术选型：Dubbo（RPC 框架）+ Zookeeper（注册中心）
动机：复用共享服务（如机构服务、用户服务）

┌────────────┐  ┌────────────┐  ┌────────────┐
│OrderService│  │Inventory   │  │Warehouse   │
│            │  │Service     │  │Service     │
└─────┬───────┘  └─────┬───────┘  └─────┬───────┘
      │                │                │
      └────────────────┼────────────────┘
                       ↓
              Zookeeper 注册中心
              Dubbo RPC 调用
              
问题：
- 服务粒度不合理（服务还是太大）
- 没有 API 网关，所有客户端直连服务
- 配置分散，每个服务独立配置文件
```

**阶段 3：微服务（2022-2024）**

```
技术栈：
- Spring Cloud Alibaba（SCA）
- Nacos（注册中心 + 配置中心）
- Sentinel（限流熔断）
- Seata（分布式事务）
- Apollo（配置中心，生产环境用）
- ShardingSphere（读写分离 + 分库分表）

┌──────────────────────────────────────────────────┐
│                   API Gateway                      │
│              (Spring Cloud Gateway)              │
└──────────────────────────────────────────────────┘
                          ↓
┌──────────┐  ┌──────────────┐  ┌───────────────┐
│ Order    │  │  Inventory   │  │  Scheduler    │
│ Service  │  │  Service     │  │  Service      │
└────┬─────┘  └──────┬───────┘  └───────┬───────┘
     │                │                  │
     ↓                ↓                  ↓
┌──────────┐  ┌──────────────┐  ┌───────────────┐
│ Order DB │  │ Inventory DB │  │ Scheduler DB │
│ (读写分离)│  │  (分库分表)  │  │  (单库)       │
└──────────┘  └──────────────┘  └───────────────┘

挑战与解决方案：
1. 分布式事务：库存扣减用 Seata AT 模式
2. 服务治理：Nacos + Sentinel 完整方案
3. 数据规模：ShardingSphere 按 warehouse_id 分 4 库
```

**阶段 4：云原生探索（当前）**

```
技术方向：
- Kubernetes (K8s)：容器编排，服务弹性伸缩
- Istio：Service Mesh，流量管理（替代 Spring Cloud Gateway）
- Prometheus + Grafana：监控告警
- ArgoCD/GitOps：Git 驱动部署
- 容器化：所有服务 Docker 镜像，K8s Deployment

K8s 部署示例：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inventory-service
  template:
    spec:
      containers:
      - name: inventory-service
        image: wms/inventory-service:v2.1.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
```

**技术选型决策框架：**

```
选型决策树：
1. 这个技术解决什么问题？不解决什么？
2. 团队有没有经验？学习曲线多高？
3. 社区活跃度？生产案例？
4. 和现有技术栈的兼容性？
5. 运维成本（监控、告警、故障恢复）？
6. 扩展性：未来 10 倍流量能撑住吗？

WMS 选型案例：
- 为什么选 Nacos 而不是 Eureka？→ Nacos 同时支持 AP/CP，Eureka 已停止维护
- 为什么选 Sentinel 而不是 Hystrix？→ Hystrix 停更，Sentinel 是阿里主推
- 为什么选 ShardingSphere 而不是 MyCat？→ ShardingSphere 是 JDBC 层，零代理开销
```

### 追问方向

1. 从 Spring Cloud 迁移到 Istio，最大的挑战是什么？收益是什么？
2. 如果让你从零设计 WMS，你会直接上微服务还是先单体？为什么？
3. 未来有没有考虑 Serverless 或 Faas？在 WMS 中哪些场景适合？

### 避坑提示

- ❌ **不要**为了云原生而云原生——没有 Kubernetes 运维能力之前不要上
- ❌ **不要**把架构演进当成"技术追新"——每次演进都要有明确的业务驱动和量化收益
- ✅ **要**能用一张图画出完整的架构演进史，并说明每个阶段**为什么**这么演进

---

## 附录：面试准备 checklist

```
□ 能画出 WMS 完整架构图（服务关系、数据流向、技术组件）
□ 能说出每个技术组件选型的原因（为什么是 Nacos，不是 Eureka）
□ 能量化架构改进的收益（延迟降低 X%，可用性提升到 99.99%）
□ 能解释自己踩过的坑（分库分表扩容、分布式事务回滚）
□ 能对比相似技术的差异（Kafka vs RabbitMQ，Sentinel vs Hystrix）
□ 能设计简单的故障演练（一个服务挂了，系统如何自动恢复）
```

> 面试提示：架构设计题没有标准答案，核心是考察"**为什么这样设计**"的思考过程。结合自己的项目经历，用数据说话，用图说话。

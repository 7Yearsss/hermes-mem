# 微服务架构进阶面试题

## 1. 服务注册发现：Eureka/Nacos/Consul对比，客户端发现vs服务端发现

### 题目
请对比 Eureka、Nacos、Consul 三种服务注册中心的选型差异，并说明客户端发现和服务端发现的原理与适用场景。

### 核心答案

**Eureka（Netflix）**
- AP 模型，保证可用性而非一致性
- 服务续约心跳间隔 30s，超过 90s 未续约则剔除
- 自我保护模式：15分钟内收到心跳比例低于阈值则停止剔除
- 仅支持 Java 客户端，不支持跨数据中心

**Nacos（Alibaba）**
- 同时支持 CP 和 AP 模式，可切换
- 支持临时实例（AP）和永久实例（CP）
- 内置配置管理、流量管理功能
- 支持 GRPC 推送，变更推送更实时
- 提供控制台，运维友好

**Consul（HashiCorp）**
- 强 CP 模型，使用 Raft 协议保证一致性
- 支持多数据中心，健康检查丰富（HTTP/TCP/Docker/Shell）
- 提供 DNS 和 HTTP 接口，内置 KV 存储
- 支持服务网格（Consul Connect）

**客户端发现 vs 服务端发现**

| 维度 | 客户端发现 | 服务端发现 |
|------|-----------|-----------|
| 原理 | 客户端从注册中心获取服务实例列表，自己实现负载均衡 | 客户端请求 LB/网关，由其从注册中心获取实例并转发 |
| 优点 | 无单点瓶颈，性能好 | 客户端轻量，逻辑简单 |
| 缺点 | 客户端复杂度高，每个语言都要实现 | LB 是新单点，需高可用部署 |
| 代表 | Eureka、Nacos | Spring Cloud LoadBalancer + Consul、Kubernetes Service |

### 追问方向
- 注册中心挂了服务还能调用吗？如何保证注册中心高可用？
- Nacos 的 namespace 和 group 如何用于环境隔离？
- Consul 的 Raft 协议在网络分区时的表现？

### 避坑提示
- 不要混淆 CAP 定理：Eureka 是 AP，Consul 是 CP，Nacos 可切换
- 实际生产环境自我保护模式可能掩盖故障，需配合告警
- 客户端缓存服务列表是常用优化，需说明缓存更新策略

---

## 2. 服务网格Service Mesh：Istio/Envoy，Sidecar模式，数据平面vs控制平面

### 题目
解释服务网格的 Sidecar 模式，以及 Istio 中数据平面和控制平面的职责划分。Envoy 在其中扮演什么角色？

### 核心答案

**Sidecar 模式**
- Sidecar 是与主容器共存于同一 Pod 的独立容器
- 接管主应用的网络通信：出站流量经 Sidecar 代理，入站流量先到 Sidecar
- 应用只需关注业务逻辑，网络能力由 Sidecar 统一提供
- 部署架构：每个服务实例旁部署一个 Envoy Sidecar 代理

**Istio 架构**

```
┌─────────────────────────────────────────────────────┐
│                   控制平面 (Control Plane)           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Pilot     │  │  Citadel    │  │  Galley     │ │
│  │ (配置下发)   │  │ (身份安全)   │  │ (配置验证)   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────┐
│                   数据平面 (Data Plane)             │
│  ┌─────────────────────────────────────────────┐  │
│  │ Pod: Service A                               │  │
│  │  ┌──────────┐    ┌──────────────────────┐    │  │
│  │  │ Proxy    │    │    App Container     │    │  │
│  │  │ (Envoy)  │◄──►│                      │    │  │
│  │  └──────────┘    └──────────────────────┘    │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**数据平面（Data Plane）**
- 由部署为 Sidecar 的 Envoy 代理组成
- 负责：流量拦截、路由、负载均衡、熔断、限流、mTLS 加密
- 动态配置来自控制平面，本地缓存保证离线可用

**控制平面（Control Plane）**
- Pilot：管理 Envoy 实例，下发路由规则和流量策略
- Citadel：证书管理，服务间 mTLS 认证
- Galley：配置验证和获取
- Mixer（已废弃旧版）：策略检查和遥测数据收集

### 追问方向
- Sidecar 注入方式：Init Container vs mutating webhook 各有什么优劣？
- Envoy 的 xDS 协议具体包含哪些 API？
- 如何在不引入完整 Istio 的情况下单独使用 Envoy？

### 避坑提示
- Istio 不是零侵入，Sidecar 会增加延迟（通常 1-3ms）和资源开销
- Mixer V1 已被废弃，不要再提 Mixer 的策略检查
- Ambient 模式下无需 Sidecar，连边车也省了，但生产稳定性待验证

---

## 3. API网关设计：请求路由/协议转换/认证鉴权/限流熔断，Spring Cloud Gateway

### 题目
描述 API 网关的核心功能，设计一个 API 网关需要考虑哪些要素？Spring Cloud Gateway 如何实现这些能力？

### 核心答案

**核心功能**
1. **请求路由**：基于路径、Host、Header 条件路由到后端服务（方向代理）
2. **协议转换**：HTTP → gRPC/WebSocket，SOAP → REST 等协议转换
3. **认证鉴权**：JWT 验签、OAuth2 令牌校验、黑名单拦截
4. **限流熔断**：基于时间/IP/用户维度的限速，熔断器保护后端
5. **日志监控**：请求耗时、状态码、traceId 记录

**Spring Cloud Gateway 核心概念**

```
请求 → Route Predicate（匹配条件）→ Gateway Filter Chain（过滤器链）→ 目标服务
```

- **Route**：路由定义，包含 ID、目标 URI、Predicate 列表、Filter 列表
- **Predicate**：匹配条件（Path、Query、Method、Cookie、Header、After/Before 等）
- **Filter**：网关过滤器，分为 GatewayFilter（单路由）和 GlobalFilter（全局）
- **DispatcherHandler**：请求分发入口

**关键配置示例**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1           # 去掉前缀
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            - name: CircuitBreaker
              args:
                name: userCircuitBreaker
                fallbackUri: forward:/fallback/user
```

**认证鉴权实现**
- 使用 GatewayFilter 拦截，校验 JWT 签名和有效期
- 提取用户角色信息，设置请求头传递给下游服务
- 网关不解析业务数据，仅验证 token 有效性

### 追问方向
- Gateway 如何解决跨域问题（CORS）？
- Gateway 的过滤器执行顺序如何确定？
- 灰度路由如何实现（金丝雀路由）？

### 避坑提示
- Gateway 基于 Spring WebFlux 非阻塞模型，不要在 Filter 中使用阻塞操作
- 限流要区分服务级别和用户级别，避免单个用户压垮整个服务
- 网关是所有流量的入口，必须高可用部署，单点网关是灾难

---

## 4. 分布式事务：Seata AT/TCC/XA三模式对比，各自适用场景

### 题目
Seata 提供了 AT、TCC、Saga 三种事务模式，请对比它们的实现原理、性能开销和适用场景。

### 核心答案

**AT 模式（Automatic Transaction）**
- 对业务 SQL 进行解析，生成反向 SQL 记录到 undo_log 表
- 一阶段：解析 SQL → 执行 → 记录 undo_log → 提交
- 二阶段：异步回滚时根据 undo_log 逆向执行
- 优点：对业务零侵入，像本地事务一样使用
- 缺点：需要数据库支持（如 MySQL SELECT ... FOR UPDATE）
- 适用场景：短事务、高并发、对性能要求高的场景

**TCC 模式（Try-Confirm-Cancel）**
- Try：预留资源（冻结库存、锁定金额）
- Confirm：确认执行（真正扣减）
- Cancel：回滚（释放冻结）
- 优点：性能好，不依赖数据库锁
- 缺点：业务侵入大，每个分支事务需实现三个接口
- 适用场景：订单支付、库存扣减等需要明确资源预留的场景

**XA 模式**
- 基于数据库 XA 协议，两阶段提交（2PC）
- 一阶段：所有分支事务执行但不提交，TM 收集各分支结果
- 二阶段：TM 发出 commit 或 rollback
- 优点：强一致，数据库原生支持
- 缺点：性能差，长事务锁定资源；MySQL XA 稳定性问题
- 适用场景：对一致性要求极高的金融场景

**三种模式对比**

| 维度 | AT | TCC | XA |
|------|-----|-----|-----|
| 一致性 | 弱一致性 | 最终一致性 | 强一致性 |
| 性能 | 好 | 很好 | 差 |
| 侵入性 | 无 | 高 | 低 |
| 依赖 | 数据库 | 业务实现 | 数据库 XA 支持 |
| 隔离性 | 依赖全局锁 | 业务保证 | 数据库原生 |

### 追问方向
- Seata 的 TC（Transaction Coordinator）如何保证高可用？
- TCC 的空回滚和悬挂问题如何解决？
- Saga 模式和 TCC 的区别是什么？

### 避坑提示
- AT 模式的回滚是逆向 SQL，不是真正的回滚，有局限性（如批量更新）
- 不要在 AT 模式下使用 SELECT COUNT(*) 作为业务判断，可能查到脏数据
- 高并发下全局锁是性能瓶颈，考虑拆分事务链减少锁范围

---

## 5. 接口幂等性：Token机制/唯一键/乐观锁/状态机，如何保证幂等

### 题目
在高并发场景下，如何保证接口的幂等性？请列举常用方案并说明各自适用场景。

### 核心答案

**Token 机制（Token-based Idempotency）**
- 客户端请求前先从服务端获取唯一 Token（如 UUID）
- 提交业务请求时携带 Token，服务端存入 Redis 并设置 TTL
- 相同 Token 重复请求直接返回第一次的结果
- 适用场景：前端重复提交、MQ 消费幂等

```java
// 示例
String token = uuidGenerator.generate();
redis.setex("idem:order:" + token, 3600, "PROCESSING");
try {
    orderService.create(token, orderDTO);
} finally {
    // 成功或失败后不删除 token，由 TTL 清理
}
```

**唯一键约束**
- 数据库唯一索引或主键约束防止重复插入
- 前置查询 + 唯一索引双重保险
- 适用场景：创建订单、用户注册等数据库写入场景

**乐观锁**
- 在更新 SQL 中加入版本号或时间戳条件
- `UPDATE account SET balance = balance - ? WHERE id = ? AND version = ?`
- 更新失败说明被其他请求修改，触发重试
- 适用场景：余额扣减、库存扣减等高并发更新

**状态机幂等**
- 订单状态流转：待支付 → 支付中 → 已支付 → 已完成
- 只允许合法状态流转，重复支付在已支付状态直接返回成功
- 适用场景：订单、支付、流转型业务

**分布式幂等Token实现**
```
客户端                        服务端
  |                            |
  |--请求获取Token------------->|
  |<--返回 token_123 ----------|
  |                            |
  |--提交业务+token_123------->|
  |<--返回业务结果 ------------|
  |                            |
  |--重复提交+token_123------->|
  |<--返回首次结果（查缓存）---|
```

### 追问方向
- Token 过期时间如何设计？过短导致正常请求失败，过长浪费存储
- 唯一键方案在高并发下如何避免主键冲突异常？
- 如何区分幂等和重复消费？它们的关系是什么？

### 避坑提示
- 幂等是接口设计原则，不是缓存问题，不要把幂等实现完全依赖 Redis
- 失败重试需要配合幂等，接口本身也要考虑失败后重入的可能性
- HTTP GET/DELETE 默认幂等，POST/PUT 需特别处理

---

## 6. 分布式锁：Redis SETNX + TTL + Lua，Redisson，ZooKeeper

### 题目
如何用 Redis 实现分布式锁？Redisson 相比原生 Redis 有什么优势？ZooKeeper 实现分布式锁的原理是什么？

### 核心答案

**Redis 原生实现（SETNX + TTL）**
```lua
-- 加锁 Lua 脚本
if redis.call('SETNX', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) == 1 then
    redis.call('SET', KEYS[1] .. ':owner', ARGV[1])
    return 1
else
    return 0
end

-- 解锁 Lua 脚本（必须校验持有者，防止误删他人锁）
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```
- 必须设置 TTL 防止死锁（锁永不过期是灾难）
- 解锁必须校验 owner，防止持有者过期后其他进程获取锁被误删
- 推荐 Redisson 等成熟库，原生实现坑多

**Redisson 分布式锁**
- 基于 Watch Dog 机制实现锁自动续期（看门狗续命）
- 底层封装完善：SETNX + TTL + Lua + 自动续期 + 公平锁/读写锁
- 提供 `RLock` 接口：`lock()`, `tryLock()`, `unlock()`
- 支持公平锁（按请求顺序获取锁）、读写锁

```java
RLock lock = redisson.getLock("order:lock:" + orderId);
try {
    lock.lock(30, TimeUnit.SECONDS); // 自动续期
    // 业务逻辑
} finally {
    lock.unlock();
}
```

**ZooKeeper 分布式锁**
- 利用临时顺序节点 + Watch 机制
- 最小编号节点获得锁，其他节点监听前一个节点
- 获取锁：创建临时顺序节点 → 获取所有节点 → 判断是否最小
- 释放锁：删除自身节点（连接断开自动删除）

**ZooKeeper vs Redis 锁**

| 维度 | Redis 锁 | ZooKeeper 锁 |
|------|---------|-------------|
| 可靠性 | 半可靠（master 挂了可能丢锁）| 可靠（CP 模型） |
| 性能 | 极高 | 中等 |
| 特性 | 原子操作丰富 | 临时节点 + Watch |
| 适用 | 毫秒级高性能场景 | 强一致、低频锁 |

### 追问方向
- Redis 主从切换时分布式锁会失效，如何解决（RedLock 算法）？
- ZooKeeper 的脑裂问题如何处理？
- 分布式锁的粒度如何设计，锁粒度太粗/太细各有什么问题？

### 避坑提示
- `SET key value NX PX timeout` 是原子加锁，不要分两步（SET + EXPIRE）操作
- 锁的 TTL 要大于业务处理时间，否则自动释放后业务还在执行
- 不要在锁内执行远程服务调用或复杂计算，锁是稀缺资源，持有时间越短越好

---

## 7. 配置中心：Apollo/Nacos/Spring Cloud Config，配置动态生效

### 题目
对比 Apollo、Nacos、Spring Cloud Config 的配置管理能力，说明配置动态生效的原理以及如何处理配置变更推送失败的情况。

### 核心答案

**Apollo（携程）**
- 业界成熟度最高的配置中心
- 概念：App → AppId（应用） → Environment（环境） → Cluster（集群） → Namespace（配置分组）
- 支持热发布、多版本管理、灰度发布、权限管理
- 配置变更推送：长轮询（Client 定时拉）或 WebSocket 推送
- 客户端缓存：本地缓存 + 远程配置，读取优先级：内存 → 本地缓存 → 远程

**Nacos**
- 注册中心和配置中心一体
- 配置变更推送：长轮询（默认 30s），支持 gRPC 推送
- 命名空间隔离 + 配置分组，默认分组 public
- 配置持久化：嵌入式数据库（测试）或 MySQL（生产）

**Spring Cloud Config**
- Git 后端：配置存储在 Git 仓库，版本化管理
- 配合 Spring Cloud Bus（基于 MQ）实现配置刷新
- Config Server 访问 Git，Config Client 通过 @RefreshScope 刷新
- 支持对称加密（.key）和非对称加密（rsaKey）

**三大配置中心对比**

| 维度 | Apollo | Nacos | Spring Cloud Config |
|------|--------|-------|---------------------|
| 配置变更推送 | 长轮询 + 通知 | 长轮询 + gRPC | Spring Cloud Bus |
| 灰度发布 | 支持 | 支持 | 不支持 |
| 权限管理 | 完整 | 基础 | 依赖 Git |
| 多环境 | App + Env + Cluster | namespace | profile |
| 运维复杂度 | 高 | 低 | 中 |

**配置动态生效原理**
1. 客户端长轮询拉取变更（通常 30s 或 60s）
2. 服务端检测到配置变更，返回变更的 key 列表
3. 客户端拉取最新配置，更新到内存和本地缓存
4. Spring 中 @RefreshScope 注解的 Bean 会被重建

**配置推送失败处理**
- 本地缓存兜底：即使推送失败，客户端也能读到上一版本配置
- 重试机制：Apollo 客户端内置指数退避重试
- 告警监控：配置变更推送失败率超过阈值触发告警
- 手动回滚：通过控制台回滚到上一版本配置

### 追问方向
- Apollo 的多环境（DEV/FAT/UAT/PRO）如何配置？
- 配置中心挂了应用能正常启动吗？启动时配置的加载顺序是什么？
- 如何在配置变更时避免业务脏读（如正在使用的配置被覆盖）？

### 避坑提示
- 不要把密码等敏感配置明文放在配置中心，使用加密配置或 SOPS/Sealed Secrets
- 配置变更后需要考虑 Bean 重建的开销，@RefreshScope 重建是非懒加载 Bean 的
- 业务启动时如果配置中心不可用，要有降级策略（本地默认配置）

---

## 8. 链路追踪：SkyWalking/Zipkin/Jaeger，traceId/spandId传递，采样率

### 题目
分布式链路追踪的核心原理是什么？traceId 和 spanId 如何在服务间传递？SkyWalking、Zipkin、Jaeger 各有什么特点？如何设计合理的采样率？

### 核心答案

**链路追踪核心原理**
- traceId：全局唯一标识一次请求链路（整个调用树共享）
- spanId：当前操作的标识（每个 RPC/方法调用生成一个 span）
- 父子关系：通过 parentSpanId 建立树形结构

```
traceId: abc123
├── span1 (HTTP GET /order)         parentSpanId: null
│   ├── span2 (DB SELECT)           parentSpanId: span1
│   └── span3 (RPC /inventory)      parentSpanId: span1
│       └── span4 (Redis GET)      parentSpanId: span3
```

**traceId 传递方式**
- HTTP 头：`-D uber-trace-id: {traceId}:{spanId}:{parentSpanId}:{flags}`
- MQ 消息头：消息体中携带 traceId（如 T hawkular.traceid）
- RPC 框架拦截：OpenFeign/Grpc 拦截器自动透传
- 必须保证 traceId 在整条链路透传，即使某服务调用失败也要传递

**三大链路追踪工具对比**

| 维度 | SkyWalking | Zipkin | Jaeger |
|------|-----------|--------|--------|
| 存储 | Elasticsearch/MySQL/H2 | Cassandra/MySQL/内存 | Elasticsearch/Cassandra |
| 协议 | SkyWalking 协议（gRPC）| Zipkin 协议（HTTP/gRPC）| Jaeger 协议（gRPC/thrift）|
| UI | 强大（拓扑图/泳道图/告警）| 基础 | 中等 |
| 语言支持 | Java/.Net/Node.js | 多语言（tracer SDK）| 多语言（官方 + OpenTracing）|
| 自动探针 | Java Agent 字节码增强 | 需代码侵入 | 需代码侵入 |
| 性能开销 | 中 | 低 | 中 |

**采样率设计**
- 全量采样（100%）：测试/小流量环境
- 固定采样（10%/1%）：生产常规场景，减少存储
- 动态采样：根据错误率自动调高采样率
- 尾部采样：优先保留慢请求和错误请求（如 1% 快成功 vs 100% 慢/错误）

```java
// SkyWalking 采样配置示例
plugin.sofa4.agent.trace.ignore_path=/health,/metrics
agent.sample_n_per_3_secs=100  # 每3秒最多100个采样
```

### 追问方向
- OpenTracing 和 OpenTelemetry 的关系是什么？
- 如何在链路追踪中找到最慢的调用链？
- 异步线程池中的调用如何串联到主 trace？

### 避坑提示
- 采样率太低可能漏掉偶发性慢调用，排查问题要结合日志
- traceId 必须在日志中输出，否则链路追踪和日志对不上
- MQ 消费场景 traceId 需要手动埋入消息头，很多框架不会自动传递

---

## 9. 灰度发布：蓝绿部署/金丝雀/ABTest，流量染色，Istio灰度

### 题目
描述蓝绿部署、金丝雀发布和 A/B 测试的区别？如何实现基于 Istio 的灰度发布？流量染色的原理是什么？

### 核心答案

**三种发布策略对比**

| 维度 | 蓝绿部署 | 金丝雀发布 | A/B 测试 |
|------|---------|-----------|---------|
| 原理 | 两套环境，切换流量 | 新版本小流量，逐步放量 | 不同版本服务不同用户群 |
| 目的 | 快速回滚，零停机 | 验证新版本稳定性 | 验证业务假设 |
| 流量划分 | 100% 切换 | X% 到新版本 | 按用户属性/规则 |
| 资源成本 | 双倍资源 | 小比例额外资源 | 小比例额外资源 |
| 回滚速度 | 秒级（切换 DNS/LB） | 逐步切回，较慢 | 立即切回 |

**金丝雀发布流程**
1. 部署金丝雀版本（通常 5-10% 流量）
2. 监控核心指标（错误率、延迟、CPU）
3. 指标正常则逐步扩大比例（10% → 30% → 50% → 100%）
4. 指标异常立即回滚到稳定版本

**Istio 灰度发布实现**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1        # 稳定版本
          weight: 90         # 90% 流量
        - destination:
            host: reviews
            subset: v2        # 灰度版本
          weight: 10         # 10% 流量

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

**流量染色**
- 在请求头中植入标记（如 `x-canary-version: v2`）
- 网关/ Sidecar 根据标记路由到指定版本
- 未染色的请求默认走稳定版本
- 可扩展：基于 Header/Cookie/用户属性做更细粒度染色

```yaml
# 基于 Header 的染色路由
- match:
    headers:
      x-canary:
        exact: "true"
  route:
    - destination:
        host: user-service
        subset: canary
```

### 追问方向
- 金丝雀发布时如何设置合理的指标阈值来触发自动回滚？
- 蓝绿部署中数据库 schema 变更如何处理？
- A/B 测试和灰度发布的本质区别是什么？

### 避坑提示
- 灰度发布时服务注册中心可能同时存在多个版本实例，注册发现要能区分
- 灰度版本和稳定版本共用数据库时，要注意数据兼容（字段新增必须向后兼容）
- 金丝雀比例太低可能测不出问题（如只在压测时暴露的性能问题）

---

## 10. 服务降级：自动降级/手动降级，Hystrix/Sentinel降级策略

### 题目
什么是服务降级？自动降级和手动降级有什么区别？Hystrix 和 Sentinel 的降级策略是如何实现的？

### 核心答案

**服务降级定义**
- 当下游服务不可用或响应过慢时，上游服务主动返回兜底数据而非错误
- 降级是**有损**的，目标是保障核心链路可用
- 降级 ≠ 熔断：降级是返回替代方案，熔断是快速失败不再请求

**自动降级（系统自适应）**
- 基于实时指标自动触发：QPS 超限、线程池满、响应超时
- Sentinel 的自适应策略：
  - 系统 Load 超阈值
  - CPU 使用率超阈值
  - 平均 RT 超阈值
  - 异常比例/数量超阈值

**手动降级（业务配置）**
- 手动开关降级预案，配置降级规则到配置中心
- 降级规则可实时下发，无需重启
- 常见降级策略：
  - 返回静态兜底数据（默认值、空值）
  - 走备用逻辑（查缓存/降级数据库）
  - 返回友好提示（"服务繁忙，请稍后重试"）

**Hystrix 降级**
- 基于线程池隔离（Command 模式）
- 降级逻辑在 `getFallback()` 方法中定义
- 触发条件：超时、异常、线程池拒绝、断路器打开

```java
public class GetUserCommand extends HystrixCommand<User> {
    @Override
    protected User getFallback() {
        return defaultUser; // 返回兜底用户
    }
}
```

**Sentinel 降级策略**
- RT 降级：响应时间超过阈值触发降级
- 异常比例：异常比例超过阈值触发降级
- 异常数：分钟内异常数超过阈值触发降级
- 熔断策略：半开状态允许一个请求去探测

```java
@SentinelResource(value = "getUser", fallback = "getUserFallback")
public User getUser(Long id) {
    return userService.getById(id);
}

public User getUserFallback(Long id, Throwable t) {
    return defaultUser; // 降级方法
}
```

**降级与熔断的关系**
```
正常 → 熔断触发 → 降级生效 → 熔断半开 → 探测成功 → 恢复正常
```

### 追问方向
- 如何设计降级预案的开关？开关存储在哪里？
- 降级返回的兜底数据如何保证时效性（缓存数据过期问题）？
- 服务降级和接口熔断有什么区别？

### 避坑提示
- 降级方法要和主方法解耦，不要在降级方法里再调用同样会失败的服务
- 降级要考虑数据一致性，如库存服务降级时不能返回超卖的数量
- 不要滥用降级，所有服务都降级等于没有降级，要梳理核心链路

---

## 11. 服务限流：令牌桶/漏桶/滑动窗口，Guava RateLimiter，Sentinel

### 题目
令牌桶、漏桶和滑动窗口三种限流算法的原理是什么？各自的优缺点？Guava RateLimiter 和 Sentinel 如何实现？

### 核心答案

**令牌桶算法（Token Bucket）**
- 桶内有一定数量的令牌，请求消耗令牌
- 令牌以固定速率补充（`r` 个/秒），上限为桶容量
- 优势：允许突发流量（前提是有足够令牌）
- 适用场景：API 限流、MQ 消费限流

```
桶容量 = 10，令牌补充速率 = 5/秒

时刻 0: 10 令牌 → 收到 5 请求，消耗 5，剩 5
时刻 1: 补充 5 令牌，现有 10 → 收到 10 请求，全通过
时刻 2: 补充 5 令牌，现有 10 → 收到 3 请求，剩 7
```

**漏桶算法（Leaky Bucket）**
- 请求以任意速率进入桶，以固定速率漏出
- 桶满则拒绝（溢出）
- 优势：流量输出平滑，稳定
- 缺点：无法处理突发，所有请求排队等待

**滑动窗口算法（Sliding Window）**
- 将时间窗口切分为多个小窗口（如 60s 分成 60 个 1s）
- 计算滑动窗口内所有小窗口的请求总数
- 比固定窗口更精确，避免窗口边界突发

**三大算法对比**

| 维度 | 令牌桶 | 漏桶 | 滑动窗口 |
|------|--------|------|---------|
| 突发处理 | 支持（拿令牌）| 不支持（均匀漏出）| 支持 |
| 平滑性 | 允许突发 | 严格均匀 | 较平滑 |
| 实现复杂度 | 中 | 低 | 高（需要存储每个小窗口计数）|
| 典型实现 | Guava RateLimiter | Sentinel | Sentinel（改进版）|

**Guava RateLimiter**
```java
RateLimiter limiter = RateLimiter.create(100); // 每秒100个令牌
limiter.acquire(); // 获取令牌，支持阻塞
limiter.tryAcquire(); // 非阻塞获取
limiter.tryAcquire(1, 100, TimeUnit.MILLISECONDS); // 带超时
```

**Sentinel 限流**
```java
// 注解方式
@SentinelResource(value = "api", blockHandler = "blockedHandler")
public String api() {
    return "ok";
}

// 配置限流规则
flowRules.add(new FlowRule("api")
    .setGrade(RuleConstant.FLOW_GRADE_QPS)
    .setCount(100)
    .setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT));
```

### 追问方向
- 分布式限流如何实现（多节点限流 vs 单点限流）？
- Sentinel 的匀速排队模式（冷启动）是什么原理？
- 限流后返回 429 Too Many Requests，客户端如何处理？

### 避坑提示
- 限流值要经过压测确定，不要拍脑袋设置
- 限流粒度要考虑（用户级/服务级/IP级），单一维度限流可能误杀
- 降级限流要配合告警，触发限流说明系统已经在承受压力

---

## 12. 多租户架构：数据隔离方案（schema隔离/数据库隔离/表隔离）

### 题目
多租户架构下，数据隔离有哪几种方案？各自的优缺点和适用场景是什么？

### 核心答案

**三种数据隔离方案**

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| 数据库隔离 | 每个租户独立数据库实例 | 隔离性最好，性能无影响 | 成本高，运维复杂 | 大型租户，金融级 |
| Schema 隔离 | 同一实例，不同 Schema | 成本较低，隔离较好 | 跨租户备份恢复复杂 | 中型租户 |
| 表隔离（行级）| 共享数据库，表加 tenant_id | 成本最低，运维简单 | 隔离性差，查询开销 | 小型租户，SaaS |

**Schema 隔离（PostgreSQL/MySQL）**
```sql
-- MySQL: 不同租户不同数据库实例
tenant_a_db.order
tenant_b_db.order

-- PostgreSQL: 不同 schema
CREATE SCHEMA tenant_a;
CREATE SCHEMA tenant_b;
tenant_a.order
tenant_b.order
```

**行级租户隔离（最常见）**
```java
// 所有查询自动带上 tenant_id
@PreAuthorize("hasRole('USER')")
public List<Order> listOrders() {
    Long tenantId = SecurityContext.getTenantId();
    return orderRepository.findByTenantId(tenantId);
}
```

**实现方式：TenantColumn + AOP 拦截**
```java
// MyBatis Plus 租户插件
@Configuration
public class MyBatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(new TenantLineHandler() {
            @Override
            public Expression getTenantId() {
                return new LongValue(SecurityContext.getTenantId());
            }
        }));
        return interceptor;
    }
}
```

**租户路由**
- 租户 ID → 数据源路由（ShardingSphere 分库分表）
- ShardingSphere 虚拟数据源 + 分片策略
- 路由规则：`ds_${tenantId}.order_${tenantId % 10}`

### 追问方向
- 跨租户查询（超级管理员查看所有租户数据）如何实现？
- 租户切换时如何避免缓存穿透？
- 新增租户的数据库初始化流程是什么？

### 避坑提示
- 不要在 SQL 中拼接 tenant_id，容易注入漏洞，用参数化查询
- 多租户索引设计要考虑 tenant_id 作为索引前缀列
- 数据导出时要校验租户权限，防止越权导出

---

## 13. 服务契约：Swagger/OpenAPI规范，SpringDoc，契约测试

### 题目
什么是服务契约（API Contract）？Swagger/OpenAPI 规范的核心概念是什么？SpringDoc 如何生成 OpenAPI 文档？契约测试的意义和实现方式？

### 核心答案

**服务契约定义**
- API 提供方和消费方共同遵守的接口规范
- 契约优先（Contract First）：先定义契约再实现代码
- 代码优先（Code First）：代码实现后生成契约
- 核心价值：前后端解耦，服务间解耦，自动化测试

**OpenAPI 规范核心概念**
```yaml
openapi: 3.0.3
info:
  title: 用户服务 API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: 获取用户信息
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
```

**SpringDoc 集成（SpringBoot 2.6+）**
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```java
// 注解方式增强 API 文档
@RestController
@RequestMapping("/api/users")
@Tag(name = "用户管理", description = "用户 CRUD 操作")
public class UserController {
    @Operation(summary = "获取用户", description = "根据 ID 获取用户详情")
    @ApiResponse(responseCode = "404", description = "用户不存在")
    @GetMapping("/{id}")
    public User getUser(@Parameter(description = "用户ID") @PathVariable Long id) {
        return userService.getById(id);
    }
}
```

**契约测试（Contract Testing）**
- Provider：服务提供方，验证自己的 API 符合契约
- Consumer：服务消费方，验证自己对 API 的假设符合契约
- 工具：Pact（Consumer-Driven Contract）、Spring Cloud Contract

**Pact 契约测试流程**
```
Consumer 团队:
1. 编写 Pact 测试（期望的响应）
2. 发布 Pact 文件到 Pact Broker
3. 自动通知 Provider

Provider 团队:
1. 从 Broker 获取 Pact 文件
2. 运行 Provider 测试验证契约
3. 测试通过则 CI 通过
```

### 追问方向
- OpenAPI 3.0 和 Swagger 2.0 的主要区别是什么？
- 契约测试和单元测试/集成测试的区别是什么？
- 如何管理 API 版本（URL 路径 vs Header vs Content-Negotiation）？

### 避坑提示
- 契约文档要纳入 CI/CD 流程，API 变更必须更新契约并审查
- 不要让契约文档过时，过时的文档比没有文档危害更大
- 契约测试不能替代集成测试，它只测接口契约，不测业务逻辑

---

## 14. 容器化部署：Docker镜像构建优化，多阶段构建，.dockerignore

### 题目
如何编写高效的 Dockerfile？多阶段构建的原理是什么？.dockerignore 的作用和常见配置？

### 核心答案

**Dockerfile 最佳实践**
```dockerfile
# 多阶段构建示例（Java 应用）
# Stage 1: 构建阶段
FROM maven:3.9-eclipse-temurin AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline  # 先复制依赖，再复制源码
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: 运行阶段（只复制 jar 包）
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "-Xms256m", "-Xmx512m", "app.jar"]
```

**多阶段构建核心原则**
1. 最小化镜像层数：`RUN` 指令合并（`&&` 链式执行）
2. 最小化基础镜像：Alpine / distroless / scratch
3. 利用构建缓存：先复制依赖文件，再复制源码
4. 不在镜像中安装构建工具（构建阶段完成后丢弃）
5. 以非 root 用户运行

**Docker 镜像层级优化**
```
基础镜像层（不变）
    ↓
依赖层（不变）
    ↓
源码层（频繁变化）← 构建缓存容易失效点
```

**优化策略**
```dockerfile
# 坏例子：每次代码变更都重新下载依赖
COPY . .
RUN mvn package

# 好例子：利用缓存，先复制依赖文件
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src .
RUN mvn package
```

**.dockerignore**
```
# Git
.git
.gitignore

# IDE
.idea/
.vscode/
*.iml

# 构建产物
target/
build/
*.class

# 文档
*.md
docs/

# 其他
Dockerfile
docker-compose.yml
.env
*.log
```

**.dockerignore 核心作用**
- 避免将临时文件、构建产物复制进镜像（增大镜像体积）
- 避免复制敏感文件（.env、凭据文件）
- 加快构建（COPY 操作跳过无关文件）

### 追问方向
- 如何减小 Java 镜像体积（JDK vs JRE、Alpine 兼容性）？
- 什么是 distroless 镜像？和 Alpine 相比各有什么优劣？
- Docker 镜像构建如何加速（构建缓存、远程构建缓存）？

### 避坑提示
- 不要在 Dockerfile 中使用 `ADD`，除非需要 URL 自动下载或解压功能（COPY 更安全）
- 镜像构建顺序要合理，把不常变化的层放前面，充分利用缓存
- 生产镜像不要包含调试工具（curl、telnet），最小化攻击面

---

## 15. Kubernetes深入：Pod/Deployment/Service/Ingress，HPA/PA/VPA自动扩缩容

### 核心答案

**题目**
Kubernetes 的核心资源对象是什么？Pod、Deployment、Service、Ingress 各自的职责和关系是什么？HPA、PA（PodDisruptionBudget）、VPA 三种自动扩缩容机制有何区别？

### 核心答案

**核心概念层级关系**
```
Namespace
    └── Deployment（无状态副本管理）
            └── ReplicaSet（副本管理）
                    └── Pod（最小调度单元）
                            └── Container
    └── Service（服务发现 + 负载均衡）
    └── Ingress（HTTP 层七层路由）
```

**Pod**
- 最小调度单元，一个 Pod 可以包含多个容器（Sidecar 模式）
- 共享网络命名空间（localhost 通信）、存储卷
- 生命周期短暂，Pod IP 不固定
- 三种重启策略：Always / OnFailure / Never

**Deployment**
- 声明式管理 Pod 副本数、更新策略、滚动回滚
- 滚动更新：`maxSurge`（最多超出期望多少）、`maxUnavailable`（最多不可用多少）
- 回滚：`kubectl rollout undo deployment/name`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user
          image: user-service:v2
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

**Service**
- Pod IP 会漂移，Service 提供固定访问入口
- 四种类型：
  - ClusterIP（集群内部访问）
  - NodePort（节点端口映射）
  - LoadBalancer（云厂商负载均衡）
  - ExternalName（CNAME 映射）
- 负载均衡策略：`sessionAffinity: ClientIP`

**Ingress**
- 七层 HTTP/HTTPS 路由，基于域名/路径转发到 Service
- 替代 NodePort 的生产级入口

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /user
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

**HPA / PA / VPA 对比**

| 机制 | 全称 | 作用 | 触发条件 |
|------|------|------|---------|
| HPA | HorizontalPodAutoscaler | 水平扩缩容（增减 Pod 副本数）| CPU/Memory/自定义指标 |
| VPA | VerticalPodAutoscaler | 垂直扩缩容（调整 Pod 资源配额）| 历史资源使用率 |
| PA | PodDisruptionBudget | 限制最大 disruption，保护关键 Pod | 主动驱逐时保证最小可用数 |

**HPA 示例**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 追问方向
- Pod 调度过程：调度器如何选择最优节点？
- Deployment 的 maxUnavailable=0 在滚动更新时的表现？
- VPA 和 HPA 能否同时使用？

### 避坑提示
- HPA 需要 metrics-server 支持，确保集群已安装
- 资源 limits 不要设太大，否则 Pod 调度困难（OOMKilled vs 调度失败）
- 生产环境 minReplicas 至少 2，避免单点故障

---

## 16. 分布式缓存：本地缓存（Caffeine/Guava Cache）+ Redis二级缓存

### 题目
为什么需要本地缓存 + Redis 的二级缓存架构？Caffeine 和 Guava Cache 各自的特点是什么？二级缓存的更新策略如何设计？

### 核心答案

**二级缓存架构**
```
请求 → L1 本地缓存（Caffeine/Guava） → L2 分布式缓存（Redis） → DB
         ↓命中直接返回                    ↓命中返回并回填L1
         ↓未命中                          ↓未命中查DB并回填L1+L2
```

**为什么需要二级缓存**
- Redis 作为分布式缓存，减少数据库压力
- 本地缓存减少网络开销，应对 Redis 不可用/延迟高的情况
- 热点数据在本地缓存命中，响应极快（微秒级）

**Caffeine vs Guava Cache 对比**

| 维度 | Caffeine | Guava Cache |
|------|---------|------------|
| 性能 | 更高（ConcurrentHashMap 优化）| 稍低 |
| 功能 | 异步加载、统计、驱逐策略丰富 | 功能完善 |
| 淘汰算法 | W-TinyLFU（近似 LFU） | LRU/LFU/FIFO |
| TTL | 支持写入/访问过期 | 支持写入/访问过期 |
| 淘汰监听 | WeakValues/SoftValues | WeakReference/SoftReference |

**Caffeine 使用示例**
```java
LoadingCache<Long, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)                      // 最大条目数
    .expireAfterWrite(10, TimeUnit.MINUTES)   // 写入后过期
    .refreshAfterWrite(5, TimeUnit.MINUTES)   // 刷新（异步重加载）
    .recordStats()                             // 开启统计
    .build(key -> userService.getById(key));   // 加载函数

User user = cache.get(1L); // 命中则返回，未命中调用 load
```

**二级缓存更新策略**

| 策略 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Cache Aside | 读：先缓存后DB；写：先DB后删缓存 | 简单 | 缓存雪崩风险 |
| Read Through | 读：缓存未命中自动加载 | 应用层代码简单 | 缓存层负责加载逻辑 |
| Write Through | 写：写缓存同时写DB | 数据一致性好 | 写入延迟增加 |
| Write Behind | 写：先写缓存，异步写DB | 写入性能极高 | 可能丢数据 |

**典型 Cache Aside + 双删**
```java
// 更新
userDAO.update(user);
cache.invalidate(user.getId()); // 删除缓存
// 延迟双删（解决并发脏读）
Thread.sleep(100);
cache.invalidate(user.getId());
```

### 追问方向
- 缓存雪崩、缓存穿透、缓存击穿各自定义和解决方案？
- Redis 分布式锁和本地缓存一起使用时如何保证一致性？
- L1 缓存如何感知 L2 缓存的数据变更（推送/过期通知）？

### 避坑提示
- L1 缓存 TTL 不能太长，否则数据更新后短期无法感知
- 不要在 L1 缓存中存经常变化的数据（如库存数量）
- 多节点部署时，本地缓存无法跨节点同步，L1 只适合存几乎不变的数据

---

## 17. 消息可靠性：RabbitMQ/Kafka消息确认机制，消息持久化，消费幂等

### 题目
RabbitMQ 和 Kafka 的消息确认机制有什么不同？如何保证消息不丢失？消费端如何实现幂等消费？

### 核心答案

**RabbitMQ 消息确认**
- **生产者确认**：Publisher Confirm，消息写入队列后 broker 返回 ACK
  - 同步：`channel.confirmSelect()` + `waitForConfirms()`
  - 异步：注册 `ConfirmCallback`
- **消费者确认**：Manual Ack 模式，消费者处理完成后手动 ACK
  - `basicAck`：确认消息已被处理
  - `basicNack` / `basicReject`：拒绝消息，可选是否重入队列
- **队列持久化**：`durable=true`，队列元数据持久化到磁盘
- **消息持久化**：`PERSISTENT_TEXT_PLAIN`，消息体写入磁盘

```java
// 生产者确认
channel.confirmSelect();
channel.basicPublish(exchange, routingKey,
    MessageProperties.PERSISTENT_TEXT_PLAIN,
    message.getBytes());
channel.waitForConfirmsOrDie(5_000);

// 消费者手动确认
channel.basicConsume(queue, false, // autoAck=false
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(...) {
            try {
                process(message);
                channel.basicAck(envelope.getDeliveryTag(), false);
            } catch (Exception e) {
                channel.basicNack(envelope.getDeliveryTag(), false, true); // 重入队列
            }
        }
    });
```

**Kafka 消息可靠性**
- **acks 配置**：
  - `acks=0`：生产者不等 broker 确认（最快，可能丢数据）
  - `acks=1`：Leader 副本写入后确认（默认，可能丢 follower 未同步数据）
  - `acks=all`（`-1`）：ISR 列表所有副本写入后确认（最安全）
- **持久化**：`log.flush.interval.messages` 控制刷盘策略
- **消费者**：手动提交 offset，不自动提交
- **重试**：`retries >= 3`，配合 `幂等生产者`

```java
// Kafka 生产者配置
props.put("acks", "all");
props.put("retries", 3);
props.put("enable.idempotence", true); // 幂等生产者，避免重复
```

**消费幂等实现**
1. **数据库唯一索引**：消费记录表，以消息 ID 作为唯一键
2. **分布式锁**：消费前加锁，处理完成释放
3. **Redis 去重**：消息 ID 存入 Redis，TTL 覆盖消息最大生命周期

```java
// 唯一索引方案
@Transaction
public void consumeOrderCreated(Message msg) {
    if (idempotentRecordDao.existsByMessageId(msg.getMessageId())) {
        return; // 已消费，直接返回
    }
    orderService.createOrder(msg.getOrder());
    idempotentRecordDao.save(MessageIdRecord.builder()
        .messageId(msg.getMessageId())
        .status("PROCESSED")
        .build());
}
```

### 追问方向
- RabbitMQ 队列积压如何处理？如何避免消息丢失又避免无限积压？
- Kafka 的 Exactly-Once 语义是什么？它和幂等生产者有什么关系？
- 消费端重试次数耗尽后消息如何处理？死信队列（DLQ）的设计？

### 避坑提示
- RabbitMQ 的 autoAck=true 是万恶之源，生产环境必须用手动确认
- 消息持久化和刷盘会增加延迟，不要对每条消息都刷盘（批量刷盘更好）
- 消费者幂等和消息幂等是两回事，都要处理

---

## 18. 服务健康检查：存活探针/就绪探针，探针配置，故障注入测试

### 题目
Kubernetes 的存活探针（Liveness Probe）和就绪探针（Readiness Probe）有什么区别？故障注入测试的意义和常用工具？

### 核心答案

**Liveness Probe vs Readiness Probe**

| 维度 | Liveness Probe | Readiness Probe |
|------|----------------|-----------------|
| 目的 | 判断容器是否需要重启（进程存活）| 判断容器是否接收流量（服务就绪）|
| 失败后 | 重启容器 | 从 Service 移除，不接收流量 |
| 适用场景 | 应用进程僵死、OOM | 应用启动中、依赖未就绪、过载 |
| 建议 | 必须配置，避免僵死进程 | 建议配置，流量调度更精准 |

**三种探针方式**
```yaml
livenessProbe:
  httpGet:                      # HTTP GET 探测
    path: /health/live
    port: 8080
  initialDelaySeconds: 30       # 启动后等待 30s 再探测
  periodSeconds: 10             # 每 10s 探测一次
  failureThreshold: 3            # 连续 3 次失败才重启
  successThreshold: 1            # 成功 1 次即恢复
  timeoutSeconds: 5              # 超时 5s

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3

# TCP Socket 探测（适合非 HTTP 服务）
readinessProbe:
  tcpSocket:
    port: 3306

# Exec 探测（执行命令）
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
```

**Spring Boot 健康检查端点**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always      # 显示组件健康详情
  health:
    db:
      enabled: true
    redis:
      enabled: true
```

**故障注入测试（Chaos Engineering）**
- 目的：在可控范围内注入故障，验证系统韧性和监控告警有效性
- 核心原则：只影响实验范围，不影响生产其他服务

**常用故障注入工具**

| 工具 | 厂商 | 特点 |
|------|------|------|
| Chaos Monkey | Netflix | 随机终止 K8s Pod/Spring Boot 实例 |
| ChaosBlade | 阿里 | 支持多云、K8s、基础资源（CPU/网络/磁盘/进程）|
| Gremlin | 商业 | 可视化故障注入平台 |
| Toxiproxy | Shopify | 网络延迟/断开注入 |
| Pumba | Docker | 容器级故障（暂停、杀容器、网络故障）|

**ChaosBlade 使用示例**
```bash
# 注入 Pod 网络延迟
blade create k8s pod-network-delay --name nginx --time 3000 --offset 1000 --interface eth0

# 注入 CPU 负载
blade create cpu load --cpu-percent 80

# 销毁 Pod（模拟随机终止）
blade create k8s pod delete --name nginx --namespace default
```

### 追问方向
- 就绪探针失败后 Pod 会被驱逐吗？和污点（Taints）有什么关系？
- 故障注入实验的步骤是什么（steady state → hypothesis → run experiment → verify）？
- 如何在不影响生产的前提下做故障注入（隔离环境 vs 流量染色 + 故障注入）？

### 避坑提示
- 故障注入必须有限影响范围（namespace/标签/Label Selector 过滤）
- 故障注入前必须确认监控告警已就绪，否则发现问题也无法感知
- 故障注入后要清理实验状态，否则故障持续影响正常服务

---

## 19. 配置加密：SOPS/Sealed Secrets，对称加密vs非对称加密

### 题目
在 GitOps 场景下，如何安全地管理 Kubernetes Secret？SOPS 和 Sealed Secrets 的原理是什么？对称加密和非对称加密在配置加密中各自扮演什么角色？

### 核心答案

**对称加密 vs 非对称加密**

| 维度 | 对称加密 | 非对称加密 |
|------|---------|-----------|
| 密钥 | 同一密钥加密解密 | 公钥加密，私钥解密 |
| 速度 | 快（适合大体量数据）| 慢（不适合大数据）|
| 密钥分发 | 需要安全通道分发 | 公钥可公开分发 |
| 典型算法 | AES-256-GCM | RSA-2048/4096, ECC |
| 用途 | 数据加密 | 密钥交换、数字签名 |

**SOPS（Secret OPerationS）**
- Mozilla 开发的加密工具，支持 PGP、AWS KMS、GCP KMS、Azure Key Vault、HashiCorp Vault
- 原理：加密 YAML/JSON 中的特定字段，存储加密后的文件到 Git
- 加密粒度：可加密单个字段，不是整个文件

```yaml
# SOPS 加密前
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  password: cGFzc3dvcmQxMjM=  # base64 明文

# SOPS 加密后（.sops.yaml 规则）
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
sops:
  encrypted_regex: "^(data|stringData)$"
  kms:
    - resource: arn:aws:kms:us-east-1:123456:key/abc
      created_at: "2024-01-01T00:00:00Z"
      arn: "arn:aws:kms:..."
data:
  password: ENC[AES256_GCM,...]  # 加密后的密文
```

**Sealed Secrets（Bitnami）**
- 加密后的 Secret 只能由 Sealed Secrets Controller（集群内）解密
- 使用非对称加密：Sealer 公钥加密，Controller 私钥解密
- Workflow：
  1. `kubeseal --certificate=pub-cert.pem` 加密 Secret
  2. SealedSecret 提交到 Git
  3. Controller 解密为原生 Secret

```bash
# 安装 Sealed Secrets Controller
helm install sealed-secrets -n kube-system bitnami/sealed-secrets

# 加密 Secret
kubectl create secret generic db-credentials \
  --from-literal=password=secret123 \
  --dry-run=client \
  -o yaml | kubeseal --cert pub-cert.pem -o yaml > sealed-secret.yaml

# sealed-secret.yaml 可以安全提交到 Git
# Controller 会自动解密为原生 Secret
```

**SOPS vs Sealed Secrets 对比**

| 维度 | SOPS | Sealed Secrets |
|------|------|---------------|
| 加密位置 | 本地加密工具 | 集群 Controller 解密 |
| 密钥管理 | 外部 KMS（AWS/GCP/Azure）| 集群内自动管理 |
| Git 兼容性 | 高 | 高 |
| 轮换密钥 | 需手动重新加密 | Controller 定期自动轮换 |
| 适用场景 | 多云/混合云配置加密 | K8s Secret 加密 |

### 追问方向
- 密钥轮换（Key Rotation）如何在不停机的情况下完成？
- 如何防止 Sealed Secrets Controller 被恶意删除（单点密钥丢失）？
- 在 CI/CD 流水线中如何集成 SOPS/Sealed Secrets 解密？

### 避坑提示
- 不要把加密密钥直接提交到 Git
- Sealed Secrets 加密后文件不是完全不可读的（metadata 未加密），敏感信息不要放 metadata
- 定期轮换加密密钥，密钥生命周期管理是安全运维的重要部分

---

## 20. 混沌工程：故障注入（Chaos Monkey/ChaosBlade），混沌实验

### 题目
什么是混沌工程？混沌工程的核心原则是什么？Chaos Monkey 和 ChaosBlade 的使用场景有什么区别？设计一个混沌实验的标准流程是什么？

### 核心答案

**混沌工程定义**
- 在生产环境或类生产环境中，通过可控实验验证系统韧性
- 核心假设：故障不可避免，与其被动等待故障发生，不如主动验证
- 与故障注入测试的区别：混沌工程是持续性的实验文化，不是一次性测试

**混沌工程核心原则（Netflix 原则）**
1. 建立稳定状态假设（Define Steady State）
2. 假设故障会影响稳态（Hypothesize）
3. 在生产环境或接近生产环境运行实验（Run Experiment）
4. 验证稳态，收集证据（Verify）
5. 自动化实验，周期性执行（Automate）

**Chaos Monkey**
- Netflix 开源的随机故障注入工具（最早）
- 专注于随机终止生产环境中的实例
- 可配置：终止概率、服务类型、时间窗口
- Spring Cloud 版本：Spinnaker + Chaos Monkey

**ChaosBlade**
- 阿里巴巴开源，多云支持
- 支持 K8s、Docker、主机、虚拟机
- 故障类型丰富：
  - 网络故障：延迟、断连、DNS 篡改、丢包
  - 资源故障：CPU、内存、磁盘 IO、进程
  - JVM 故障：GC、Fatal Error、抛异常
  - 基础资源：Kafka 延迟、MySQL 慢查询

**ChaosBlade 典型场景**
```bash
# 1. 演练场景：Pod 被随机删除
blade create k8s pod-kill --name nginx --namespace default --label "app=nginx"

# 2. 演练场景：网络延迟注入
blade create k8s network-delay --name nginx --namespace default \
  --remote-ip 10.0.0.1 --time 3000 --offset 500

# 3. 演练场景：MySQL 连接数打满
blade create mysql delay --database testdb --time 30000
blade create process kill --process mysql

# 4. 查看演练结果
blade destroy <experiment-uid>

# 混沌实验平台（ChaosBlade-Box）
# 支持可视化编排实验、监控实验状态、生成报告
```

**混沌实验标准流程**
```
1. 定义稳态指标
   → QPS > 1000, P99 < 200ms, 错误率 < 0.1%

2. 构建假设
   → "如果 Redis Master 被 kill，缓存失效，后端数据库能承接吗？"

3. 设计和执行最小爆炸半径实验
   → 先在测试环境，再在生产灰度节点
   → 准备回滚方案（止血预案）

4. 观察和记录
   → 监控指标、告警触发、日志链路追踪

5. 复盘改进
   → 通过实验发现的问题 → 修复/优化 → 持续验证
```

**混沌实验分级**
| 级别 | 影响范围 | 风险 | 场景 |
|------|---------|------|------|
| L1 | 单容器/Pod | 低 | 验证探针恢复 |
| L2 | 单节点/单AZ | 中 | 验证节点故障不影响服务 |
| L3 | 多AZ/多节点 | 高 | 验证核心链路冗余 |
| L4 | Region 级 | 极高 | 验证容灾恢复能力（RTO/RPO）|

### 追问方向
- 混沌工程和传统故障测试的本质区别是什么？
- 混沌实验如何做到不影响正常用户（流量隔离 + 实验组）？
- 如何评估混沌实验的效果（稳态指标 vs SLA/SLO）？

### 避坑提示
- 混沌实验必须有回滚方案和止血预案，否则实验失控是灾难
- 不要一次性做太多故障场景，保持最小爆炸半径
- 混沌实验要逐步升级（先 L1 再 L3），不要一开始就 Region 级
- 实验后必须复盘，否则实验白做，问题下次依然会出现

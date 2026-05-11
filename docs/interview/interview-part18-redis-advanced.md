# Redis 深度知识面试题

> 本文档涵盖 Redis 高频深度面试题 20 道，结合 WMS 仓库管理系统实际场景，助你备战高级/专家级别面试。

---

## 1. Redis 数据结构：SDS / 链表 / 压缩列表 / 跳表 / 整数集合 / 对象系统

### 题目

请描述 Redis 6种底层数据结构（SDS、链表、压缩列表、跳表、整数集合）的实现原理，以及 Redis 如何通过对象系统统一封装它们。

### 核心答案

**SDS（Simple Dynamic String，简单动态字符串）：**
- `sdshdr` 结构体：`len`（已用长度）、`alloc`（总分配长度）、`flags`（类型标识）、`buf`（柔性数组）
- 3种编码类型：`SDS_TYPE_5`（≤32B）、`SDS_TYPE_8`/`16`/`32`/`64`（按字符串长度动态升级）
- 优势：O(1) 获取长度、二进制安全、杜绝缓冲区溢出（空间不够时先扩展再写入）、内存预分配（长度<1MB时加倍扩展，长度≥1MB时多分配1MB）

**链表（Linked List）：**
- `listNode` 双向链表节点：prev/next 指针
- `list` 列表头：头尾指针、长度、dup/free/match 函数指针（实现多态）
- 特性：双向、无环、带头尾指针、支持迭代遍历

**压缩列表（Zip List）：**
- 连续内存块：zlbytes（四字节总长度）+ zltail（到尾节点偏移）+ zllen（节点数量）+ entries + zlend
- 每个 entry：`prevlen`（前节点长度，1或5字节）+ `encoding` + `content`
- 连锁更新风险：删除/插入节点时，若 `prevlen` 从1字节扩展到5字节（或反向），后续节点需依次重新分配内存
- 使用场景：小整型、短字符串（String/Hash/List/ ZSet 的小数据量情况）

**跳表（Skip List）：**
- 多层有序链表：L0 层包含所有元素，L1/L2/... 每层是下层的子集
- `zskipnode`：后退指针、层高（随机算法决定，约 1/4 概率升层）、柔性数组存 score+member
- `zskiplist`：head/tail 指针、最大层数（32）、长度
- 查询：从最高层开始向右，若下一节点 > 目标则下沉，层层下沉至 L0

**整数集合（Int Set）：**
- `intset` 连续内存：小端存储的整数数组
- 编码方式：`INTSET_ENC_INT16` / `INT32` / `INT64`，插入时若超出当前编码范围则触发升级
- 升级过程：所有元素转换为新编码类型，再插入，保持有序（二分查找定位）
- 只支持整数、去重、有序

**对象系统：**
```c
typedef struct redisObject {
    unsigned type:4;       // 类型：STRING/LIST/HASH/SET/ZSET
    unsigned encoding:4;   // 编码方式
    unsigned lru:LRU_BITS; // LRU时钟或LFU计数器
    int refcount;          // 引用计数
    void *ptr;             // 指向底层数据结构的指针
} robj;
```
- type 对外展现5种数据类型，encoding 对内决定具体物理存储格式
- 每种 type 可对应多种 encoding，实现动态底层适配

### 追问方向

- SDS 的空间预分配和惰性释放分别解决了什么问题？
- 压缩列表的连锁更新最坏时间复杂度是多少？如何规避？
- 跳表和平衡树查询复杂度都是 O(log N)，为什么 Redis 选跳表而不是红黑树？
- 整数集合升级后还能降级吗？为什么？

### 避坑提示

- ❌ 不能混淆"SDS是C字符串包装"——SDS是二进制安全的，buf不以`\0`结尾判断长度
- ❌ 误以为压缩列表是链表——压缩列表是连续内存，支持随机访问但删除/插入代价高
- ❌ 混淆对象类型(type)与编码(encoding)——type是用户视角的数据类型，encoding是内部实现
- ✅ 面试高频：跳表 vs 红黑树，SDS vs C字符串

---

## 2. Redis 5种底层数据结构及对象类型对应关系

### 题目

Redis 的 5 种数据类型（String、List、Hash、Set、ZSet）分别对应哪些底层实现？同一种数据类型在什么情况下会切换编码？

### 核心答案

| 对象类型 | 底层编码 | 触发条件 |
|---------|---------|---------|
| **String** | `int` | 整数值且 ≤ `INT_MAX` |
| **String** | `embstr` | 字符串长度 ≤ 44字节（Redis 3.0+） |
| **String** | `raw` | 字符串长度 > 44字节 |
| **List** | `ziplist` | 列表长度 ≤ `list-max-ziplist-entries`（默认512）且每个元素 ≤ `list-max-ziplist-value`（默认64B） |
| **List** | `quicklist` | 不满足 ziplist 条件时。quicklist = 多段 ziplist 双向链表（每个 ziplist 节点叫 `quicklistNode`），允许参数控制深度 |
| **Hash** | `ziplist` | 键值对数量 ≤ `hash-max-ziplist-entries`（默认512）且 field 和 value 各自 ≤ `hash-max-ziplist-value`（默认64B） |
| **Hash** | `hashtable` | 超过 ziplist 阈值后自动转换，**不支持降级** |
| **Set** | `intset` | 集合元素全部为整数且数量 ≤ `set-max-intset-entries`（默认512） |
| **Set** | `hashtable` | 元素非整数或超过阈值 |
| **ZSet** | `ziplist` | 有序集合元素数 ≤ `zset-max-ziplist-entries`（默认128）且每个元素 ≤ `zset-max-ziplist-value`（默认64B），ziplist 按 score 排序存储 |
| **ZSet** | `skiplist` + `hashtable` | 超过 ziplist 阈值。hashtable 存 member→score 便于 O(1) 查找 score，跳表用于范围操作和排名 |

**编码升级（只能升级不能降级）：**
- String：`int` → `embstr` → `raw`
- Hash：`ziplist` → `hashtable`
- Set：`intset` → `hashtable`
- List：`ziplist` → `quicklist`
- ZSet：`ziplist` → `skiplist+hashtable`

**`embstr` vs `raw`：**
- `embstr`：只调用一次 `zmalloc`，将 redisObject 和 SDS 连续存储在同一个内存块中（≤44B 时使用）
- `raw`：调用两次 `zmalloc`，redisObject 和 SDS 独立分配内存

### 追问方向

- quicklist 的 `list-compress-depth` 参数是什么意思？设为0和设为5有什么区别？
- ZSet 同时使用 skiplist 和 hashtable 不会浪费内存吗？两个结构如何配合？
- embstr 既然连续存储，为什么 44 字节是分界线？
- 用 `OBJECT ENCODING` 命令查看一个 key 的编码，这个命令本身会不会修改 key 的编码？

### 避坑提示

- ❌ 记错 embstr 分界线——Redis 3.0 前是 32 字节，3.0+ 优化为 64-19=45 字节（jemalloc 最小分配单位）
- ❌ 混淆 List 的 quicklist 和 LinkedList——Redis 没有 LinkedList 编码，quicklist 是 ziplist 的双向链表
- ❌ 以为 Hash 可以从 hashtable 降级回 ziplist——encoding 只能升级不能降级
- ✅ 高频追问：ZSet 为什么需要 hashtable + skiplist 双写？

---

## 3. Redis 淘汰策略：noeviction / allkeys-lru / allkeys-random 等，LRU/LFU 算法实现

### 题目

Redis 的 maxmemory 淘汰策略有哪些？LRU 和 LFU 算法在 Redis 中是如何实现的？WMS 库存缓存场景下如何选型？

### 核心答案

**8种淘汰策略：**

| 策略 | 含义 |
|-----|------|
| `noeviction` | 不淘汰，写入报错（默认） |
| `volatile-lru` | 仅设置过期时间的 key 中淘汰最近最少使用 |
| `volatile-lfu` | 仅设置过期时间的 key 中淘汰最不常用 |
| `volatile-random` | 仅设置过期时间的 key 中随机淘汰 |
| `volatile-ttl` | 优先淘汰 TTL 最小的 key |
| `allkeys-lru` | 所有 key 中淘汰最近最少使用 |
| `allkeys-lfu` | 所有 key 中淘汰最不常用 |
| `allkeys-random` | 所有 key 中随机淘汰 |

**LRU 实现（近似 LRU）：**
- Redis 使用采样法而非精确 LRU：每次淘汰时随机取 `maxmemory-samples`（默认5，可配置到10）个 key，淘汰其中最旧的
- `object idletime` 可查看 key 的空闲时间（不精确，访问时更新）
- LRU 链表方案被放弃——维护成本太高，近似算法在命中率上差异不大

**LFU 实现：**
- `lru_bits[24位]` 分两部分：`counter`（16位，0~255饱和计数器）+ `last decrement time`（8位，对数递减时间戳）
- 访问频率递增：`LFUDecrAndReturn()` 根据距上次访问时间计算衰减，LFUIncr()` 做非线性增长（避免瞬间打满255）
- 配置项：`lfu-log-factor`（控制增长率）、`lfu-decay-time`（控制衰减速率，单位分钟）
- 与 LRU 对比：高访问频率的 key 更难被淘汰，适合写少读多的缓存场景（如WMS商品信息）

**WMS 场景选型建议：**
- 库存缓存：选 `volatile-lru` + 设置 TTL，避免冷数据占满内存
- 会话/Token：选 `allkeys-lru`，无 TTL 概念但容量可控
- 秒杀/高频热点：选 `allkeys-lfu`，防止偶发高热度 key 驱逐真正热点

### 追问方向

- LRU 采样数量是不是越大越好？5 和 10 的实际命中率差异？
- LFU 的 counter 饱和后怎么办？会降回低值吗？
- `maxmemory-policy` 和 `maxmemory` 的关系是什么？
- 淘汰执行在哪个时机？是主线程还是后台线程？

### 避坑提示

- ❌ 误以为 Redis LRU 是精确的——是近似算法，采样5个中最旧的淘汰
- ❌ 忘记设置 `maxmemory`——未配置时默认使用物理内存，可能导致 OOM
- ❌ volatile-ttl 淘汰的是 TTL 最小的 key，不是最大——TTL 越小越急迫淘汰
- ✅ 生产经验：LFU 在热点场景比 LRU 稳定，但需要合理调 `lfu-decay-time`

---

## 4. Redis 持久化：RDB（fork/COW）/ AOF（appendfsync）/ 混合持久化

### 题目

Redis 持久化机制有几种？RDB 的 fork 机制和 COW 是怎么回事？AOF 三种刷盘策略如何选型？混合持久化解决了什么问题？

### 核心答案

**RDB（Redis Database Snapshot）：**

触发方式：
- `BGSAVE`：fork 子进程执行，主进程继续服务（fork 期间）
- `SAVE`：阻塞主进程，不推荐生产使用
- 自动触发：配置 `save m n`（m秒内n次写操作则触发）

**fork + COW 原理：**
- fork 创建子进程时，Kernel 将父进程地址空间（包含所有 Redis 数据页）复制一份到子进程，**共享物理内存页**（read-only view）
- COW：父进程或子进程对某页进行写操作时，Kernel 触发缺页中断，复制该页到新物理地址，各自持有独立副本
- fork 瞬间不复制数据，内存越大 fork 越慢（Linux 使用 Copy-on-Write vfork/huge pages 优化）
- fork 完成后子进程遍历内存生成 RDB 文件，父进程正常处理请求

**AOF（Append Only File）：**

`appendfsync` 三种策略：
| 策略 | 刷盘时机 | 性能 | 安全性 |
|-----|---------|------|-------|
| `always` | 每个命令执行后同步刷盘 | 最慢 | 最多丢一个命令 |
| `everysec`（默认） | 每秒一次后台线程刷盘 | 中等 | 最多丢1秒数据 |
| `no` | 依赖 OS 刷盘策略 | 最快 | 最多丢一个 AOF 缓冲区 |

**重写机制（BGREWRITEAOF）：**
- 目的：消除冗余命令（如多次 SET 同一 key，只保留最后一次）
- 原理：fork 子进程读取当前数据库状态，将命令序列写入新 AOF 文件
- 增量重写：子进程重写期间，主进程每执行写命令就记录到 AOF 重写缓冲区，重写完成后将缓冲区内容追加到新 AOF 文件，原子替换旧 AOF

**混合持久化（Redis 4.0+）：**
- 开启：`aof-use-rdb-preamble yes`
- 原理：RDB + AOF 混合文件格式
  - AOF 重写时，先写入 RDB 格式的全量数据，再追加后续增量 AOF 命令
  - 加载时：识别 RDB magic number 则先加载 RDB 部分，再加载后续 AOF 命令
- 优势：RDB 恢复快（不用逐条回放命令），AOF 不丢数据，结合两者优点
- 缺点：RDB 部分不兼容低版本；混合文件无法用 `redis-check-aof` 修复

### 追问方向

- fork 在数据量多大时会有明显卡顿？10GB 内存服务器 fork 8GB RDB 的实际耗时？
- AOF 重写期间主进程挂了怎么办？会不会丢失重写期间的增量命令？
- 为什么 RDB 恢复比 AOF 快？加载 10GB RDB 和回放 10GB AOF 差距有多大？
- 混合持久化下，RDB 格式部分能否单独修复？

### 避坑提示

- ❌ fork 期间不是"复制整个内存"——是共享页表，子进程写入时才复制（Copy-on-Write）
- ❌ 误以为 AOF always 模式下每个命令都刷盘——实际上是调用 `fsync(fd)`，刷盘由 OS 决定
- ❌ 认为混合持久化 = RDB + AOF 两个文件——是混合成一个文件
- ✅ 生产推荐：AOF everysec + RDB 定时备份，或直接用混合持久化

---

## 5. Redis 事务：MULTI / EXEC / WATCH，ACID 特性，Redis vs MySQL 事务

### 题目

Redis 事务如何实现？WATCH 的机制是什么？为什么说 Redis 事务不满足 ACID 中的 C（一致性）和 I（隔离性）？和 MySQL 事务的关键区别在哪？

### 核心答案

**Redis 事务流程：**

```
MULTI        # 开启事务，客户端进入事务状态
SET k1 v1    # 命令入队
GET k1
HINCRBY k2 f 1
EXEC         # 执行队列中的所有命令
```

- `MULTI` 后命令入队而非执行
- `EXEC` 触发队列执行：失败命令返回错误，但**不会中断后续命令**（不像 MySQL 的原子性）
- `DISCARD` 清空命令队列，退出事务状态
- `WATCH key [key...]`：乐观锁，监视 key，若 EXEC 前被其他客户端修改则事务被取消（返回空）

**WATCH 实现：**
- 每个被 WATCH 的 key 在 Redis 中维护一个 `watched_keys` 字典
- 客户端执行写命令时，Redis 检查是否有关联的 watcher，如有则标记该客户端的 `REDIS_DIRTY_CAS`
- `EXEC` 时检测 `REDIS_DIRTY_CAS`，若被设置则返回空数组（整个事务失败）

**Redis 事务满足的 ACID：**

| ACID 属性 | Redis 事务 | MySQL(InnoDB) |
|-----------|-----------|---------------|
| **Atomic（原子性）** | 部分支持：EXEC 批量执行，但错误命令不中止其他命令 | 完全支持：回滚机制 |
| **Consistency（一致性）** | ❌ 不满足：无回滚机制，命令错误无法撤销 | ✅ 满足 |
| **Isolation（隔离性）** | ❌ 不满足：EXEC 执行期间无隔离，并发命令可能穿插 | ✅ 满足（MVCC） |
| **Durability（持久性）** | ❌ 不满足：取决于持久化策略（RDB/AOF/混合） | ✅ 满足（redo log） |

**Redis vs MySQL 事务关键区别：**
- Redis 事务是**命令队列批量执行**，MySQL 是基于 undo/redo log 的完整事务系统
- Redis 无 rollback——错误命令（如类型错误）会继续执行，不会撤销
- Redis 无真正的并发隔离——EXEC 期间其他命令可执行穿插
- Redis 事务可嵌套（内层 MULTI/EXEC）

### 追问方向

- Redis 事务中一个命令失败了，EXEC 返回值是什么？应用程序如何判断？
- WATCH 多个 key 时是 AND 还是 OR 关系？任一 key 被改就取消？
- Redis 7.0 的 MULTI/EXEC 相比之前版本有什么改进？
- Lua 脚本能否替代 Redis 事务？Lua 脚本与事务的本质区别？

### 避坑提示

- ❌ 把 Redis 事务等同于 MySQL 事务——两者 ACID 特性完全不同
- ❌ 误以为 EXEC 失败会回滚——Redis 没有回滚机制，失败命令的错误被返回，但队列继续执行
- ❌ 混淆 WATCH 的作用——WATCH 只在 EXEC 时检查，不阻止其他客户端写入
- ✅ 实战经验：Lua 脚本可替代事务实现原子性，且更灵活（支持条件判断）

---

## 6. Redis 发布订阅：模式、缺点、替代方案（Streams）

### 题目

Redis 的 PUBSUB 机制是什么？有哪些使用模式？为什么说它不适合做可靠消息队列？Redis 5.0 的 Streams 解决了什么问题？

### 核心答案

**PUBSUB 两种模式：**

1. **Channel（频道）模式**：
   - `PUBLISH channel message`：向频道发布消息
   - `SUBSCRIBE channel [channel...]`：订阅一个或多个频道（阻塞）
   - `UNSUBSCRIBE [channel...]`：取消订阅

2. **Pattern（模式匹配）模式**：
   - `PSUBSCRIBE pattern [pattern...]`：按模式订阅（如 `news.*`）
   - `PUBSUB NUMSUB channel`：查看频道订阅者数量
   - `PUBSUB CHANNELS [pattern]`：列出当前活跃频道

**PUBSUB 缺点：**

| 缺陷 | 说明 |
|-----|------|
| 无消息持久化 | 订阅者离线期间的消息全部丢失 |
| 无消息确认机制 | 发布者不知道谁收到了消息 |
| 消息堆积无上限 | 消费者若处理慢，消息堆积在 buffer 中 |
| 无消息 ID | 无法精确追踪消费位置 |
| 水平扩展困难 | 多个订阅者无法负载均衡，各自分别收到全量消息 |

**Streams（Redis 5.0+）：**

`XADD key group * field value [field value...]`：追加消息（`*` 表示自动生成 ID）
`XREAD [COUNT n] [BLOCK ms] STREAMS key id`：读取新消息
`XREADGROUP GROUP group consumer STREAMS key ID`：消费组读取（阻塞）
`XGROUP CREATE key group id MKSTREAM`：创建消费组
`XACK key group id [id...]`：确认消息已处理

**Streams vs PUBSUB：**
- Streams 有消息 ID、支持持久化、消费组 ACK 机制
- Streams 支持消息追溯（`XREVRANGE`）
- Streams 消息可删除（`XDEL`）或限制长度（`XTRIM`）

**WMS 场景应用：**
- PUBSUB：实时推送仓库作业指令到拣货员 PDA（可容忍短暂丢失）
- Streams：订单状态变更事件，需可靠消费（支付完成后触发库存锁定）

### 追问方向

- Streams 的消息 ID 格式是什么？为什么这样设计？
- 消费组（Consumer Group）如何实现消息的分发和负载均衡？
- 如果一个消费者挂了，消息会丢失吗？Streams 如何处理？
- Streams 的 MAXLEN 参数和 XTRIM 命令有什么区别？

### 避坑提示

- ❌ 用 PUBSUB 做可靠消息队列——它天生不保证送达，离线消息丢失
- ❌ 混淆 Streams 和 List 作队列——List 的 BRPOPLPUSH 可以做可靠队列，但Streams 功能更完整
- ❌ Streams 消息 ID 是全局递增但不连续——删除消息不会补空缺
- ✅ 生产场景：消息可靠性要求高时用 Streams，要求低时用 PUBSUB

---

## 7. Redis 主从复制：全量复制 / 增量复制 / psync2.0，断线重连

### 题目

Redis 主从复制原理是什么？全量复制和增量复制的触发条件分别是什么？psync2.0 如何实现断线重连后的部分同步？

### 核心答案

**主从复制配置：**
```bash
replicaof master_ip master_port   # 旧版命令
replicaof no one                 # 取消复制
```
或配置文件中 `replicaof <masterip> <masterport>`

**复制流程：**

1. **连接建立阶段**：从库发送 `PING` → 主库响应 → 认证（如需）→ 从库发送 `PORT` 和 `CIP`（IP）→ 进入在线配置阶段
2. **同步阶段**：从库向主库发送 `psync <runid> <offset>`
3. **命令传播阶段**：主库将写命令以 `REPLCONF ACK <offset>` 形式发给从库

**全量复制（sync）：**
- 触发条件：从库第一次连接，或 `runid` 未知，或 offset 为 -1，或主库 `repl_backlog_buffer` 缓冲区不足
- 过程：`PSYNC ? -1` → 主库执行 `BGSAVE` 生成 RDB → 发送 RDB 文件（磁盘或 socket 直接传输）→ 主库将 RDB 期间的写命令存入 `repl-diskless-load` 缓冲区 → 从库加载 RDB → 主库发送缓冲区增量 → 进入命令传播

**增量复制（psync）：**
- 触发条件：从库 offset 在主库 `repl_backlog_buffer` 环形缓冲区范围内
- 依赖：`repl_backlog_buffer`（主库所有写命令环形缓冲）+ `repl_offset`（主库已传播偏移量）+ 从库维护的 `master_repl_offset`
- 环形缓冲区大小由 `repl-backlog-size` 控制（默认 1MB）

**psync2.0（Redis 4.0+）改进：**

psync2.0 在 `repl_backlog_buffer` 基础上增加了 `repl_buffer`：
- `repl_backlog_buffer`：持久化的增量缓冲区，master 重启后仍可用
- `repl_buffer`：从库与 master 断开期间的缓冲区，存储从库未 ACK 的命令（从库越多，内存越大）

**psync2.0 断线重连流程：**
1. 从库断开，积累写命令到 `repl_buffer`（从库私有）
2. 重连后，从库发送 `psync <master_runid> <repl_offset>`
3. 若 master_runid 与本地一致，且 `repl_offset` 在 `repl_backlog_buffer` 内 → 部分同步
4. 若 master_runid 变了或 offset 超界 → 全量复制

**WMS 场景：**
- 读写分离：主库承担写，从库承担读（报表查询、库存快照读取）
- `repl-backlog-size` 要设置足够大，防止从库抖动期间缓冲区溢出导致全量复制

### 追问方向

- 全量复制期间主库能否处理写请求？RDB 期间的新写入如何同步？
- `repl-diskless-load` 的 `repl-diskless-load on-empty-db` 和 `swapdb` 区别是什么？
- 从库宕机恢复后会自动重连吗？重连的间隔和次数？
- 主从复制中，主库挂了怎么办？

### 避坑提示

- ❌ 混淆 `repl_backlog_buffer` 和 `repl_buffer`——前者是共享环形缓冲区，后者是从库私有缓冲区
- ❌ 忘记设置 `repl-backlog-size`——默认值 1MB，大流量场景容易溢出触发全量复制
- ❌ 以为从库只能读——从库默认可读，但写入不会同步回主库（有 `replica-read-only`）
- ✅ 生产经验：读写分离时，读从库要做好延迟监控，主从延迟过大会导致脏读

---

## 8. Redis Sentinel：故障自动切换、主观下线 / 客观下线、哨兵选主

### 题目

Redis Sentinel 如何实现故障自动切换？主观下线（SDOWN）和客观下线（ODOWN）的判断逻辑是什么？哨兵是如何选主的？

### 核心答案

**Sentinel 核心概念：**

- 哨兵本身是 Redis 实例的监控进程，不存储数据，只监控
- 最小部署：3 节点（保证多数派票数）
- 配置：`sentinel monitor <master-name> <ip> <port> <quorum>`
- `quorum`：判断 ODOWN 需要的哨兵数量（不是哨兵总节点数）
- `down-after-milliseconds`：主观下线阈值（毫秒）

**主观下线（SDOWN）：**
- 定义：单个 Sentinel 认为 master 已不可达（`PING` 超时）
- 触发：`sentinelTimer` 每次向 master 发送 `PING`，若超过 `down-after-milliseconds` 无有效回复，则标记 SDOWN
- SDOWN 状态仅本地有效，不同步给其他 Sentinel

**客观下线（ODOWN）：**
- 定义：多个 Sentinel 都认为 master SDOWN，达到 quorum 票数后升级为 ODOWN
- 计算：Sentinel 向其他 Sentinel 询问 `SENTINEL is-master-down-by-addr`；若超过 quorum 个 Sentinel 回应 SDOWN，则 ODOWN
- ODOWN 是触发故障转移的充分条件

**故障转移（选主）流程：**

1. **Leader 选举**：Sentinel 集群通过 `sentinel  is-master-down-by-addr` 投票，拿到多数票（≥ (哨兵数/2+1)）的 Sentinel 成为 leader
2. **选出新主**：
   - 过滤掉主观下线的从库
   - 按 `priority` 降序（配置文件中 `replica-priority`）
   - priority 相同则选 `info_repl_offset` 最大的（数据最新）
   - 再相同则选 `runid` 字典序最小的
3. **执行切换**：
   - 选中的从库执行 `replicaof no one` 成为主库
   - 其他从库向新主库执行 `replicaof new_master_ip port`
   - 原主库降为从库（若恢复）

**Sentinel + 客户端：**
- 客户端连接 Sentinel 获取 master 地址：`SENTINEL get-master-addr-by-name <name>`
- `SENTINEL master <name>` 查看 master 详情
- `SENTINEL sentinels <name>` 查看同集群其他哨兵

### 追问方向

- Sentinel 选 leader 时，为什么需要多数派（>哨兵数/2）而不是简单多数？
- `sentinel monitor` 的 quorum 和 `min-replicas-to-write` 是什么关系？
- Sentinel 主观下线的计数器是如何工作的？为什么超时时间要远大于 PING 周期？
- Sentinel 选主时，`runid` 字典序最小这个规则会不会导致每次都选同一个从库？

### 避坑提示

- ❌ 混淆 SDOWN 和 ODOWN——SDOWN 是本地判断，ODOWN 是分布式多数派决策
- ❌ 以为 Sentinel 可以监控从库故障——Sentinel 监控的是主库，从库故障由主库下线触发
- ❌ 忘记 `replica-priority` 默认值——默认是 100，priority=0 的从库永远不会被选为主
- ✅ 最小部署：3个 Sentinel + 1主2从，quorum 设为 2

---

## 9. Redis Cluster：槽指派 16384、ASK/MOVED 重定向、故障转移

### 题目

Redis Cluster 为什么用 16384 个槽而不是更多？客户端收到 MOVED 或 ASK 重定向时应该如何处理？故障转移是如何在 Cluster 层面完成的？

### 核心答案

**为什么是 16384 个槽：**

1. **心跳包开销**：每个 Sentinel/Node 定时向集群所有节点广播 Gossip 消息，包含节点 bitmap（槽指派信息）。16384 个槽 = 2048 字节（16384/8），每秒心跳约 50 字节/节点，开销可接受。若用 65536（65536/8=8KB），开销成倍增加
2. **足够分布式**：65536 槽意味着平均每个主库分到的槽数很少，迁移时迁移槽的后续命令需要遍历大量 key，迁移复杂度高
3. **Redis 作者经验**：16384 在实际集群规模下足够且高效，Node 不超过 1000 个时完全够用

**槽指派：**
- 集群有 N 个主库，每个主库负责 `16384/N` 个槽（不平均，由 `CLUSTER ADDSLOTS` 分配）
- `CLUSTER INFO` 查看集群状态
- `CLUSTER SLOTS` 查看槽分配

**MOVED 重定向（永久重定向）：**
- 场景：客户端访问的 key 所在的槽已迁移到其他节点
- 返回：`MOVED <slot> <ip:port>`
- 行为：**客户端更新本地槽映射表**，直接转向正确节点（大多数客户端库自动处理）
- 槽映射表初始化：客户端发送 `CLUSTER SLOTS` 获取全量槽映射

**ASK 重定向（临时重定向）：**
- 场景：槽正在迁移中（迁移是渐进的，不是原子切换），本节点没有该 key 但有迁移中数据
- 返回：`ASK <slot> <ip:port>`
- 行为：客户端**先向源节点发送 ASKING 命令**，再发送原请求到目标节点（允许客户端在迁移期间访问到中间态数据）
- ASK 不会更新客户端槽映射（因为迁移尚未完成）

**故障转移（Cluster 层面）：**
- 触发：某主库的从库（PONG 超时）认为主库下线
- 流程：
  1. 从库广播 `FAILOVER_AUTH_REQUEST` 请求其他主库投票
  2. 获得多数主库（≥N/2+1）投票的从库成为新的主库
  3. 新主库执行 `replicaof no one`，切换为独立主库
  4. 其他从库向新主库发起复制请求

**集群选举 vs Sentinel 选主：**
- Cluster 故障转移基于 Raft 变种（拉票制），Sentinel 基于多数派投票
- Cluster 从库发起故障转移，Sentinel 是哨兵节点发起

### 追问方向

- MOVED 和 ASK 的本质区别是什么？客户端处理策略有什么不同？
- 为什么 Cluster 要求至少 3 主 3 从的部署？2 主 2 从可以吗？
- 槽迁移过程中，某 key 被访问时同时在源节点和目标节点都不存在会返回什么？
- `cluster-require-full-coverage` 参数的作用是什么？设为 no 会怎样？

### 避坑提示

- ❌ 混淆 MOVED 和 ASK——MOVED 是槽已完全迁移，ASK 是迁移中间态
- ❌ASK 处理时忘记发 ASKING 命令——ASK 是临时重定向，需要客户端显式发送 `ASKING` 才会处理
- ❌ 以为 ASK 和 MOVED 返回后客户端会自动重试——某些低级客户端不会自动处理，需要手动处理
- ✅ 经验：生产环境不要裸用 Cluster 命令，用 `redis-cli --cluster` 工具或 Redis Cluster 客户端库

---

## 10. Redis 哈希冲突：链式寻址 / 渐进式 rehash，扩容缩容过程

### 题目

Redis 字典（dict）如何解决哈希冲突？什么是渐进式 rehash？为什么需要两个哈希表？扩容和缩容的触发条件是什么？

### 核心答案

**字典数据结构：**
```c
typedef struct dict {
    dictType *type;      // 类型特定函数（hash_fn, compare_fn, ...）
    void *privdata;
    dictEntry **ht_table[2]; // 两个哈希表，ht[1] 用于 rehash
    long ht_size_exp[2];     // 哈希表大小（2^n）
    long ht_used[2];        // 已用节点数
    long rehashidx;         // rehash 进度，-1 表示未进行
    int iterators;          // 正在迭代的迭代器数量
} dict;
```

**链式寻址解决冲突：**
- 相同哈希值的 entry 以单向链表形式挂在 `ht_table[bucket]` 下
- 新 entry 始终插入链表头部（O(1)）
- 查询时先计算 bucket，再遍历链表比对 key（`dictEntry`：key、v（union）、next 指针）
- 负载因子 = `ht_used[0] / ht_size[0]`

**扩容触发条件：**
- 未执行 `BGSAVE` / `BGREWRITEAOF` 时：`load_factor >= 1`
- 执行 `BGSAVE` / `BGREWRITEAOF` 时：`load_factor >= 5`（避免频繁 rehash 与 RDB/AOF 重写冲突）
- 扩容目标：找到第一个 >= `used * 2` 的 2^n

**缩容触发条件：**
- `load_factor < 0.1` 时自动缩容（`dict.c` 中常量 `DICT_HT_MIN_SIZE = 4`）
- 缩容目标：找到第一个 >= `used` 的 2^n

**渐进式 rehash：**
- 原因：ht[0] 数据量大时，一次性迁移会造成服务阻塞（Redis 单线程）
- 过程：
  1. `rehashidx = 0`，开始渐进式 rehash
  2. 每次对字典的增删改查操作，在完成正常操作后，**顺带**将 `ht_table[0][rehashidx]` 链表迁移到 `ht_table[1]`，`rehashidx++`
  3. 所有 bucket 迁移完成后，ht[0] 和 ht[1] 指针交换，`rehashidx` 置 -1
- 两个哈希表同时存在：查找时先查 ht[0]，找不到再查 ht[1]（因为新数据可能写入 ht[1]）
- 写操作只写入 ht[1]，保证 ht[0] 最终变为空

### 追问方向

- 渐进式 rehash 期间，如果只有读操作没有写操作，rehash 会进行吗？
- `dictEntry` 中的 `v` 是 union，如何区分存储的是指针还是整数值？
- 渐进式 rehash 过程中，如果 HT[0] 和 HT[1] 同时存在相同 key 怎么办？
- rehashidx 在什么情况下会回退（停止 rehash）？

### 避坑提示

- ❌ 误以为 rehash 是在 EXEC 或某命令执行完后一次性完成——是分散到每次访问时逐步迁移
- ❌ 混淆扩容和缩容的负载因子阈值——扩容是 >=1（无后台进程）或 >=5（有后台进程），缩容是 <0.1
- ❌ 以为 rehash 期间写操作不会影响 rehash 进度——写操作只往 ht[1] 写入，ht[0] 越来越空
- ✅ 面试核心：渐进式 rehash 的设计动机是避免阻塞，不是并发安全（单线程本身安全）

---

## 11. Redis 跳跃表：为什么比平衡树省内存，并发安全

### 题目

Redis 为什么用跳跃表而不是红黑树/AVL 树作为 ZSet 的底层结构？跳跃表的并发安全性如何？它比平衡树的内存优势在哪里？

### 核心答案

**跳跃表 vs 平衡树内存对比：**

| 结构 | 每节点额外开销 |
|-----|--------------|
| 红黑树 | 2个指针（左右孩子）+ 1个颜色标记 = ~24字节 |
| AVL 树 | 2个指针 + 平衡因子 = ~24字节 |
| 跳跃表 | 平均 `log n` 个指针（每层概率 1/4 升层），期望约 1.33 个指针 = ~16字节 |

Redis 跳跃表层高随机算法：`randomLevel()`，最大 32 层，每层有 1/4 概率升层：
- 期望层数 = `1 / (1-1/4) = 1.33`
- 相比平衡树固定 2 个孩子指针，跳跃表指针数量期望值更低

**跳跃表结构（Redis）：**
```c
typedef struct zskiplistNode {
    double score;         // 分值
    robj *obj;            // 成员对象（sds）
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前向指针
        unsigned int span;               // 到下一个节点的跨度（用于 rank）
    } level[];
    struct zskiplistNode *backward;    // 后退指针
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;    // 节点数
    int level;               // 最大层数
} zskiplist;
```

**并发安全性：**
- Redis 单线程模型天然无并发问题，不需要额外加锁
- 相比之下，平衡树（如红黑树）在并发环境下需要 RWLock 或其他同步机制
- 跳跃表的 insert/remove 操作只涉及相邻节点的局部指针修改，不需要全局锁
- 若 Redis 未来多线程化，跳跃表比平衡树更容易实现无锁化（lock-free 数据结构研究方向）

**ZSet 为什么用跳表而不是红黑树：**

| 考量 | 跳跃表 | 红黑树 |
|-----|-------|-------|
| 范围查询（ZRANGE） | 天然支持：只需定位起点，沿 L0 层遍历 | 需要中序遍历，需要栈 |
| 排名（ZRANK） | `span` 字段直接累计，无需额外计算 | 需要从根向下累计 |
| 实现复杂度 | 极低（约 200 行） | 高（旋转、染色等） |
| 内存开销 | 期望 O(1) 指针 | 固定 2 个指针 |
| 并发友好性 | 高（局部修改） | 低（可能触发全局 rebalance） |

### 追问方向

- 跳跃表的 `span` 字段是如何在 rank 计算中起作用的？
- 跳跃表最坏时间复杂度是多少？和平衡树的区别？
- Redis 跳跃表的 `level` 最大为什么是 32？这和节点数量有什么关系？
- 如果要实现一个 lock-free 跳跃表，主要难点在哪？

### 避坑提示

- ❌ 误以为跳跃表最坏情况是 O(n)——它的最坏情况是所有节点在同一层，退化为链表，但随机化保证概率极低
- ❌ 混淆并发安全和内存布局——Redis 单线程天然安全，跳跃表的并发优势是对比多线程场景下的平衡树
- ❌ 认为跳表不能做范围查询——恰恰相反，跳表做范围查询比平衡树更自然
- ✅ 面试高频：Redis 作者 antirez 选择跳表的核心原因是"实现简单，比红黑树少很多代码"

---

## 12. Redis 大 Key 处理：SCAN / UNLINK / 异步删除，bigkey 危害

### 题目

什么是 Redis 大 Key？它有哪些危害？如何用 SCAN 替代 KEYS 查找大 Key？UNLINK 和 DEL 的区别是什么？异步删除的实现原理？

### 核心答案

**大 Key 定义：**
- String 类型：value 超过 10KB（经验值，业界通常以 10KB~1MB 为阈值）
- 复合类型：Hash/List/Set/ZSet 元素个数超过 5000~10000，或单元素过大

**大 Key 危害：**

| 危害 | 场景描述 |
|-----|---------|
| 内存不均 | 集群模式下大 key 导致槽分布不均，引发节点内存差异 |
| 阻塞主线程 | DEL/SCAN/GET 对大 key 操作耗时过长，阻塞 Redis 单线程 |
| 网络带宽 | 读取大 value 占用大量带宽，影响同实例其他请求 |
| 主从延迟 | 大 key 同步给从库时造成从库OPS暴涨 |
| 持久化阻塞 | BGSAVE/RDB 期间 COW 复制大页面，fork 耗时增加 |

**SCAN 替代 KEYS：**
```bash
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```
- `KEYS pattern`：一次性遍历全库，O(N) 阻塞，**生产禁用**
- `SCAN cursor`：基于游标的增量迭代，每次返回一小批 key，不阻塞
- `COUNT` 提示每次返回数量（实际可能略少），默认 10
- 遍历完返回 `cursor=0`

**SCAN 实现原理：**
- Redis 字典（dict）rehash 时，SCAN 会在 rehashidx 两侧交替遍历（反向采样），保证不遗漏不重复
- 安全：SCAN 期间可以正常处理请求，不会漏掉 rehash 中迁移到 ht[1] 的新 key

**UNLINK vs DEL：**
- `DEL`：同步删除，遍历并释放 key 及其 value 引用，阻塞主线程
- `UNLINK`（Redis 4.0+）：**异步删除**，流程：
  1. 将 key 从全局字典中解除引用（O(1)），立即返回
  2. 释放 value 的工作交给后台线程 `bioCloseFiles` / `bioLazyFree`（根据数据类型）
  3. 对于 String（embstr/raw）：直接释放 SDS 内存
  4. 对于复合类型：把 full-expiry robin 任务投入后台队列

**异步删除后台队列：**
- Redis 启动时创建 2 个后台 I/O 线程：`BIO_CLOSE_FILE`（关闭文件描述符）、`BIO_LAZY_FREE`（释放内存）
- 大 key 的 value 释放进入 `lazy free` 队列，由后台线程完成

**WMS 场景排查：**
```bash
# 找出大 key（Redis 4.0+）
redis-cli --bigkeys

# 扫描指定数据库
SCAN 0 MATCH "stock:*" COUNT 1000

# 查看 key 大小（需要保证 redis-cli 版本支持）
MEMORY USAGE key_name
```

### 追问方向

- UNLINK 真的完全异步吗？什么情况下会退化成同步 DEL？
- SCAN 的 COUNT 参数是精确的吗？如果遍历过程中 key 被删除/新增会怎样？
- `lazy free` 线程会不会导致内存泄漏？什么情况下会失败？
- 大 key 造成主从复制延迟，如何在主从链路层面做优化？

### 避坑提示

- ❌ 生产环境用 KEYS 命令——无论任何理由，都是严重失误
- ❌ 认为 UNLINK 是完全异步的——如果 key 的 value 引用计数复杂或处于 active expire cycle，仍可能在主线程处理
- ❌ 忽略大 key 对集群槽分布的影响——大 key 导致数据倾斜，单节点内存暴涨
- ✅ 最佳实践：拆分大 hash（按 field 数量拆分），或对大 list 使用pipeline分段操作

---

## 13. Redis 热点 Key：热点探测、热点Key解决方案

### 题目

什么是 Redis 热点 Key 问题？WMS 仓库系统中哪些场景容易产生热点 Key？如何探测热点 Key？有哪些解决方案？

### 核心答案

**热点 Key 定义：**
- 某些 key 被频繁访问（如爆款商品库存、热门 SKU 详情），QPS 远超其他 key
- 单节点 Redis 承受不住热点 key 的请求，导致 CPU 过载、请求堆积

**WMS 热点场景：**
- 爆款商品库存扣减（秒杀场景）：`stock:sku:12345` 大量并发 HINCRBY
- 仓库路由热力图：`warehouse:route:{user_id}` 被反复读取
- 缓存的订单状态：`order:status:{order_id}` 高频查询

**热点探测方案：**

1. **客户端层**：在 Redis 客户端 SDK 埋点，统计 key 访问频次（`client-side caching`）
2. **Proxy 层**：Codis/Proxy 层做 key 统计（如阿里云 Redis 热点 Key 探测）
3. **Redis 层**：`redis-cli --hotkeys`（Redis 4.0+）扫描热 key（基于 `LFU` 策略）
4. **监控平台**：定期 `DEBUG OBJECT key` 查看 `lfu` 值

**热点 Key 解决方案：**

**方案1：本地缓存 + Redis（两极缓存）：**
```
热点 key → 本地缓存（Guava/Caffeine/LRU）→ Redis → DB
```
- 优点：热点 key 直接在 JVM 堆命中，不走网络
- 缺点：数据一致性难以保证（本地缓存过期/更新不一致）、内存浪费
- 适用：读多写少、一致性要求不高的场景

**方案2：热点 Key 分散（key 后缀分片）：**
```
hot:sku:{sku_id}:{shard_id}   # 分成 N 个 key
hot:sku:{sku_id}               # 写入时打散到 N 个分片
```
- 客户端用一致性 hash 路由到不同分片
- 缺点：复杂度增加，需要改造客户端

**方案3：读写分离 + 从库探测：**
- 写主库，读从库
- 当从库延迟增大时说明有热点打到从库
- 对从库做热 key 标记

**方案4：热点 key 发现后临时打散：**
- 监控系统发现热点后，自动将该 key 的请求打散到多个 key（key + random 后缀）
- 访问时随机路由到不同分片

**方案5：HashTag 强制分片（Cluster 场景）：**
- 若热点 key 所在槽集中在某一节点，可利用 HashTag 将其分散到不同节点
- `hotkey{user_id}:info` — 大括号内的 user_id 决定槽位

### 追问方向

- 本地缓存和 Redis 缓存一致性问题如何处理？延迟双删？canal 订阅？
- 热点 key 打散后，如何保证数据的原子性（如库存扣减）？
- Redis Cluster 下热点 key 发现和打散有什么特殊挑战？
- LFU 热度统计精度和 `lfu-log-factor` 的关系？

### 避坑提示

- ❌ 不要在 Redis 热 key 逻辑里做复杂计算——Redis 单线程受不了
- ❌ 忽视热点 key 的"二八定律"——80% 请求集中在 20% 的 key
- ❌ 随意用 `FLUSHDB` 清空——热 key 会瞬间打满数据库
- ✅ 最佳实践：事前（key 设计避免热点）+ 事中（监控 + 自动分散）+ 事后（大 key 拆分）

---

## 14. Redis 分布式锁：SETNX + Lua、RedLock、红锁实现

### 题目

Redis 分布式锁如何用 SETNX + Lua 实现？RedLock 算法是什么？Redisson 的 RLock 又是怎么实现的？为什么单节点 Redis 分布式锁不是绝对安全的？

### 核心答案

**基础分布式锁实现：**
```lua
-- 获取锁（SETNX + EXPIRE 原子化）
if redis.call('SETNX', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[3])
    return 1
else
    -- 防死锁：已存在则检查是否过期，过期则覆盖
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        redis.call('EXPIRE', KEYS[1], ARGV[3])
        return 1
    end
    return 0
end
```
- `SET key value NX PX timeout`：原子获取锁，避免先 SETNX 后 EXPIRE 的 race condition
- value 用唯一标识（如 UUID），释放锁时验证 owner

**释放锁（Lua 原子验证 + 删除）：**
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```
- 验证 value 是否匹配，防止误删他人持有的锁

**RedLock 算法（Redis 作者 antirez 提出，5 个独立 Redis 实例）：**

1. 获取当前时间 `T1`
2. 依次向 N 个 Redis 实例（多数独立部署）执行 `SET resource_name unique_id NX PX timeout`
3. 获取成功响应的实例数 `M`，记录每个实例的响应时间 `T2 - T1`
4. 若 `M >= N/2 + 1` 且总耗时 < 锁有效期，则认为成功获取锁
5. 锁实际有效期 = 初始 TTL - 获取耗时
6. 失败时向所有实例发送 `DEL` 释放锁

**为什么单节点分布式锁不安全：**
- Redis 主节点故障 + 哨兵/主从切换期间，锁可能丢失（主库还没同步到从库就被判定下线）
- 场景：客户端 A 获取了锁，主库宕机，哨兵提升从库为新主，锁数据未同步，客户端 B 也拿到同一把锁

**RedLock 安全性争议（Martin Fowler 与 antirez 的讨论）：**
- 争议点：RedLock 依赖系统时钟，假设 N 台机器时钟同步（CLOCK_DRIFT_FACTOR = 0.01）
- 批评：RedLock 不是真正的分布式锁，更像是"多节点 lease 机制"
- 实际生产：多数场景用 Zookeeper / etcd 做分布式锁，或使用 Chubby

**Redisson RLock 实现：**
```java
RLock lock = redisson.getLock("myLock");
lock.lock();  // 自动续期（watchdog 机制）
// 底层：异步线程每 10 秒续期 TTL/3
// 释放：finally 中 unlock()
```
- `lock()`：阻塞等待，支持 watchdog 自动延期（默认 30 秒，看门狗每 10 秒续 2/3）
- `tryLock()`：非阻塞，获取不到立即返回
- `lockInterruptibly()`：可中断加锁

### 追问方向

- 锁续期（watchdog）的实现原理？如果持锁客户端挂了，锁多久会自动释放？
- RedLock 的 5 个 Redis 实例是否需要主从复制？还是独立单节点？
- SETNX + EXPIRE 不是原子操作，为什么实际可以接受？什么情况下会出问题？
- 如何实现一个"可重入锁"？Redis 中如何实现？

### 避坑提示

- ❌ 简单使用 `SETNX + EXPIRE` 两步操作——不是原子的，中间掉电会导致死锁
- ❌ 认为 RedLock 是绝对安全的——它只是降低了单机故障概率，仍有争议
- ❌ 忘记释放锁后检查返回值——释放他人持有的锁是严重 bug
- ✅ 生产推荐：用 Redisson 或直接用 RedLock 算法（理解其局限性），不要自己写底层实现

---

## 15. Redis Lua 脚本：EVAL / EVALSHA、原子性保证、预编译脚本

### 题目

Redis Lua 脚本如何工作？为什么它是原子的？EVAL 和 EVALSHA 的区别是什么？SCRIPT LOAD 和 SCRIPT EXISTS 如何用于脚本缓存？

### 核心答案

**Lua 脚本基础：**
```bash
EVAL "return redis.call('GET', KEYS[1])" 1 "mykey"
# EVAL script numkeys [key ...] [arg ...]

# KEYS[1] = "mykey"，ARGV[1] = nil（无参数时）
```

**EVAL vs EVALSHA：**
| 命令 | 特点 |
|-----|------|
| `EVAL` | 直接传脚本内容，每次都要发送完整脚本 |
| `EVALSHA` | 传脚本 SHA1 摘要，先 `SCRIPT LOAD` 加载到 Redis，返回 sha1；后续直接用 sha1 调用 |
| `SCRIPT LOAD script` | 将脚本预加载到脚本缓存，返回 sha1 |
| `SCRIPT EXISTS sha1 [sha1...]` | 检查脚本是否已缓存 |
| `SCRIPT FLUSH` | 清空脚本缓存 |

**原子性保证：**
- Redis 在执行 Lua 脚本期间**禁用其他所有命令**（相当于单命令独占 Redis）
- 执行 Lua 脚本时：
  1. 将 Lua 脚本参与的所有 key 注册到 `lua_scripts` 字典
  2. 执行 `evalCommand()`，Lua 脚本被当作单条命令执行，执行期间不处理其他客户端请求
  3. 脚本中的 `redis.call()` 立即执行，`redis.call()` 错误会中断脚本
  4. `redis.pcall()` 类似 call，但捕获错误并以表形式返回
- **不存在"部分执行"**——Lua 脚本要么完全执行，要么完全不执行

**脚本执行时间限制：**
- `lua-time-limit`（默认 5 秒）：超时后向其他客户端返回错误（BUSY）
- Redis 6.2+：`ALLOW BUSY` 命令允许在超时后继续执行

**Lua 脚本的副作用（幂等性）：**
- `MULTI/EXEC` 事务无法嵌套 if-else，Lua 脚本支持条件判断，灵活性更高
- 多次执行同一脚本，`MULTI/EXEC` 每次重新入队，Lua 脚本每次重新执行
- Lua 脚本中不要使用随机函数（`math.random`）或时间函数（`os.time`）——因为主从复制时从库执行结果可能不同（需开启 `SCRIPT FLUSH` 后同步）

**WMS 库存扣减 Lua 脚本示例：**
```lua
local stock = redis.call('GET', KEYS[1])
if not stock then
    return -1  -- key不存在
end
if tonumber(stock) < tonumber(ARGV[1]) then
    return 0   -- 库存不足
end
redis.call('DECRBY', KEYS[1], ARGV[1])
return 1       -- 扣减成功
```

### 追问方向

- Lua 脚本中 `redis.call()` 和 `redis.pcall()` 的区别？什么情况下需要用 pcall？
- 主从复制模式下，Lua 脚本的复制是复制脚本内容还是复制 CALL 命令？
- Redis 7.0 的 FUNCTION 命令和 Lua 脚本是什么关系？
- 如果 Lua 脚本执行超时，Redis 会怎么处理？杀掉脚本还是回滚？

### 避坑提示

- ❌ 以为 Lua 脚本内部可以用 `EVAL` 嵌套——Lua 脚本不能嵌套另一个 EVAL，嵌套 call 是可以的
- ❌ 忘记 `SCRIPT LOAD` 做预加载——每次 EVAL 传输完整脚本浪费带宽，复杂脚本建议预加载
- ❌ 在 Lua 脚本中使用 `math.randomseed` 导致主从不一致——但 4.0+ 的 Redis 已解决此问题
- ✅ 最佳实践：库存扣减、分布式锁等原子操作优先用 Lua 脚本，而非 MULTI/EXEC

---

## 16. Redis 与 MySQL 一致性：Cache Aside / Read Through / Write Through / 异步双写

### 题目

如何保证 Redis 缓存与 MySQL 数据库的一致性？Cache Aside、Read Through、Write Through、Write Behind 分别是什么？WMS 库存场景下应该用哪种方案？

### 核心答案

**四种缓存读写策略：**

| 策略 | 读 | 写 | 一致性 | 性能 |
|-----|----|----|----|----|
| **Cache Aside** | 先读缓存，miss 再读DB，set 缓存 | 先更新DB，再删缓存 | 最终一致 | 高 |
| **Read Through** | 缓存层自动加载 miss 数据 | - | 最终一致 | 高 |
| **Write Through** | - | 先写缓存，再同步写DB（同步） | 强一致 | 低 |
| **Write Behind** | - | 先写缓存，异步批量写DB | 最终一致 | 最高 |

**Cache Aside（旁路缓存，最常用）：**

读流程：
```
GET key → 命中 → 返回
GET key → 未命中 → SELECT DB → SET key → 返回
```

写流程：
```
UPDATE DB → DELETE key（不是 SET）→ 返回
```
- 为什么是 DELETE 而不是 SET？因为 SET 可能导致脏数据（如 DB 更新了，SET 了旧值），DELETE 让下次读自然 load 新数据
- **删除而非更新**：更新操作在并发场景下可能出现"先更新 DB，后删除缓存"时序错误，导致旧值被写回缓存

**延迟双删（解决并发写问题）：**
```
1. DELETE key
2. UPDATE DB
3. sleep(N) → DELETE key  # 等待主从延迟后再次删除
```

**异步双写（Write Behind）：**
- 更新 DB 和设置缓存都放入队列，异步批量执行
- 问题：丢失更新（队列数据丢失）、数据脏读
- 适用：对一致性要求极低、日活低、允许最终一致的场景

**Canal 订阅方案（MySQL → Redis）：**
```
MySQL → Binlog → Canal → Redis
```
- MySQL 开启 binlog，Canal 伪装成从库订阅 binlog，解析后写入 Redis
- 优点：完全解耦，对业务代码无侵入
- 缺点：架构复杂、Canal 自身单点需要集群部署

**WMS 库存场景一致性分析：**

```
场景：订单系统扣减库存
Cache Aside 风险：
  T1: 查缓存 miss，读 DB 库存=10
  T2: 订单A 扣减库存，更新 DB 库存=9
  T3: 删除缓存（删除的是 T1 时刻的旧缓存）
  T4: 订单B 查缓存 miss，读 DB（库存=9，正确）
  
但如果 T3 删除的是 T2 之后的值，或 T1~T3 之间缓存被重新 SET：
  T1: 读 DB 库存=10
  T2: SET 缓存 = 10
  T3: 订单A 更新 DB=9，DELETE key
  T4: 订单B GET 缓存 miss，读 DB=9（正确）
  
危险时序：
  T1: 读 DB 库存=10
  T2: SET 缓存 = 10
  T3: 订单A 更新 DB=9
  T4: 订单A DELETE key（删除新值！）
  T5: 订单B GET miss，读 DB=9（但 T1 的 SET 还没被删，命中的是旧缓存=10）
  T6: 订单B 错误地认为库存=10，扣减成功
```
- **解决方案**：分布式锁 + 延迟删除（sleep 后再删一次）

### 追问方向

- Cache Aside 中为什么"删除缓存"而不是"更新缓存"？两者有什么区别？
- 延迟双删的 sleep 时间怎么定？主从延迟通常是多久？
- Canal 订阅相比双写方案的优势是什么？有什么缺点？
- Redis 缓存和 MySQL 数据一致性还有哪些工程上的陷阱？

### 避坑提示

- ❌ 用 SET 而不是 DELETE 更新缓存——并发时序下会导致脏数据
- ❌ 忽视主从同步延迟——DB 写完但从库还没同步完成，此时读到的是旧数据
- ❌ 缓存永不过期——除非有主动更新机制，否则数据会永远陈旧
- ✅ 推荐方案：Cache Aside + 延迟双删（库存一致性要求高），或 Canal（解耦但复杂）

---

## 17. Redis 消息队列：BLPOP / BRPOP、Streams 命令、XADD / XREADGROUP

### 题目

Redis 如何实现消息队列？List 的 BLPOP/BRPOP 和 Streams 的 XADD/XREADGROUP 有什么区别？XREADGROUP 的消费组如何实现消息可靠消费？

### 核心答案

**List 实现队列（LPUSH + BRPOP）：**
```bash
LPUSH queue "msg1"    # 生产者写
BRPOP queue 0         # 消费者阻塞阻塞等待（0=永久等待）
# 返回: ["queue", "msg1"]
```
- `BRPOPLPUSH source dest timeout`：阻塞弹出并推送到另一个列表（可靠模式，支持消息确认后删除）
- `BLPOP`/`BRPOP`：从左/右弹出一个元素，超时返回 nil

**List 可靠队列模式（BRPOPLPUSH + ACK）：**
```lua
-- 消费者：弹出到处理队列
WHILE TRUE
    msg = BRPOPLPUSH "queue" "processing" 0
    process(msg)
    LREM "processing" 1 msg  -- 处理成功后从 processing 删除
```
- 将消息从主队列转移到"处理中"队列，处理完成后显式删除
- 缺点：处理中队列无限增长（需监控+定时清理）

**Streams（Redis 5.0+）：**
```bash
# 生产者：添加消息
XADD mystream * field1 "value1" field2 "value2"
# * 表示自动生成消息 ID（格式：timestamp-sequence）

# 消费者：读取新消息
XREAD STREAMS mystream 0        # 从 ID=0 开始读所有历史
XREAD BLOCK 5000 STREAMS mystream $  # $ 表示只读新消息（阻塞）

# 消费者组（可靠消费）
XGROUP CREATE mystream mygroup $ MKSTREAM  # 创建消费组，从最新消息开始
XREADGROUP GROUP mygroup consumer1 STREAMS mystream ">"  # 只读未投递的消息
XACK mystream mygroup msg_id  # 确认消息已处理
```

**Streams vs List 消息队列对比：**

| 特性 | List (BLPOP) | Streams |
|-----|-------------|---------|
| 消息持久化 | 依赖 AOF/RDB | 自身持久化（类似 List） |
| 消息确认 | 手动处理 | `XACK` 确认机制 |
| 消息追溯 | 只能处理队首 | `XREVRANGE` 支持历史 |
| 消费组 | 不支持 | `XREADGROUP` 支持多消费者负载均衡 |
| 消息删除 | `LREM` | `XDEL` 或 `XTRIM` |
| 广播模式 | 不支持 | 不支持（但可用 PUBSUB） |
| 消息 ID | 无 | 自增 ID 支持排序 |

**消费组（Consumer Group）机制：**
- `XREADGROUP GROUP <gname> <cname> STREAMS <stream> ">"`：只读取**新消息**（未投递）
- `XREADGROUP GROUP <gname> <cname> STREAMS <stream> 0`：从**Pending Entry List (PEL)** 读取所有**未 ACK** 的消息（客户端下线重连后可继续处理）
- `XACK`：从 PEL 中移除消息，表示成功处理
- PEL（Pending Entry List）：每个消费者维护自己待确认的消息 ID 列表

**XADD 消息 ID 格式：**
- `<millisecondsTime>-<sequenceNumber>`
- `1672531200000-0`
- 毫秒时间戳基于本地 Redis 实例时钟（不是真实时间，可漂移）
- 手动指定 ID 可控制消息顺序：`XADD mystream "1700000000000-0" ...`

### 追问方向

- XREADGROUP 中 `>` 和具体 ID 分别代表什么？
- 如果消费者崩溃了，未 ACK 的消息会永久留在 PEL 中吗？如何处理？
- XADD 能否实现延迟消息（如延迟队列）？如何实现？
- Streams 消费组能否实现"消息只能被一个消费者处理"的互斥语义？

### 避坑提示

- ❌ 用 BLPOP/BRPOP 做可靠队列时忘记处理"处理中"队列——消息可能丢失
- ❌ 混淆 Streams 的消息 ID 和消息内容的地位——ID 只是标识符，不保证单调递增（多实例时间戳不同步）
- ❌ `XTRIM` 是异步的——命令返回后删除不一定完成，不要误以为可以精确控制消息数量
- ✅ 生产队列推荐：Streams + 消费组 + ACK 机制

---

## 18. Redis GEO：GEOADD / GEORADIUS / GEODIST，LBS 应用

### 题目

Redis GEO 如何存储和查询地理位置信息？GEOADD/GEORADIUS/GEODIST 的实现原理是什么？在 WMS 仓库系统中可以用于哪些场景？

### 核心答案

**Redis GEO 核心命令：**
```bash
GEOADD warehouse:locations 116.4074 39.9042 "仓库A"  # 添加一个仓库
GEOADD warehouse:locations 116.4164 39.9142 "仓库B"  # 添加另一个仓库
GEODIST warehouse:locations "仓库A" "仓库B" km  # 计算两仓库直线距离（km/m/cm/mm/ft）

# 查询附近仓库（半径搜索）
GEORADIUS warehouse:locations 116.4100 39.9100 5 km WITHDIST WITHCOORD ASC COUNT 10
# 参数：中心坐标 半径 单位 km/m/ft/mi
# WITHDIST：返回距离
# WITHCOORD：返回坐标
# ASC：按距离升序
# COUNT：限制返回数量

# 查询成员附近（无需知道坐标）
GEOPOS warehouse:locations "仓库A"  # 获取仓库A的坐标

# 海鲜/米范围内搜索（替代 GEORADIUS）
GEOSEARCH warehouse:locations FROMLONLAT 116.4100 39.9100 BYRADIUS 5 km WITHDIST ASC
```

**GEO 实现原理：**
- 底层使用 **Sorted Set（ZSet）**，score = **Geohash 编码值**
- Geohash：将经纬度编码为一个 52 位整数，既能排序（按距离粗筛），又能范围查询
- Redis GEO 命令实际是 ZSet 命令的封装：
  - `GEOADD` = `ZADD`（将 geohash(score) + member 存入 ZSet）
  - `GEOPOS` = `ZSCORE` + `ZSCAN` 解析
  - `GEORADIUS` = `ZRANGEBYSCORE` + 过滤 + Haversine 距离公式重排

**Geohash 算法：**
- 经度 [-180, 180] → 二进制交叉编码
- 纬度 [-90, 90] → 二进制交叉编码
- 52位整数按 5bit 分组映射到 Base32 字符
- 相邻格子 Geohash 值接近，适合范围查询
- 精度：geohash 长度 6 位时，精度约 ±0.61 米（Redis 默认 11 位）

**GEORADIUS 限制（6.2+）：**
- Redis 6.2 引入 `GEOSEARCH` 和 `GEOSEARCHSTORE` 作为 GEORADIUS 的替代
- GEORADIUS 在未来版本可能废弃

**WMS 仓库系统 LBS 应用场景：**

| 场景 | Redis GEO 命令 | 说明 |
|-----|--------------|------|
| 仓库位置管理 | `GEOADD` | 录入仓库坐标 |
| 附近仓库查询 | `GEORADIUS/GEOSEARCH` | 给定用户位置，查 N 公里内所有仓库 |
| 仓库间距离 | `GEODIST` | 计算仓库间配送距离 |
| 配送员调度 | `GEORADIUS` | 查找附近空闲配送员 |
| 网格化围栏 | `GEOSEARCH BYBOX` | 矩形范围搜索 |

**性能特点：**
- GEO 操作是 O(log N + M)，N=集合大小，M=结果集大小
- `GEORADIUS` 结果集不宜过大，先用 `COUNT` 限制
- 大规模点位查询：按城市/区域分 key，避免单 key 数据量过大

### 追问方向

- Geohash 的原理是什么？为什么相邻格子 Geohash 值接近？
- GEODIST 计算的是直线距离还是道路距离？如何计算实际配送距离？
- Redis GEO 在集群模式下有什么限制？槽迁移会影响 GEO 数据吗？
- 如果数据量达到千万级别，Redis GEO 如何优化？

### 避坑提示

- ❌ 认为 GEODIST 计算的是精确道路距离——它用的是 Haversine 公式算的是地球表面弧线距离，不是实际道路距离
- ❌ 在大集合上频繁用 GEORADIUS——结果集过大会阻塞单线程
- ❌ 混淆 GEO 和轨迹追踪——GEO 只存点位，轨迹需要其他数据结构（如 List + GEOHASH）
- ✅ 建议：全国范围仓库先用城市维度分 key（如 `warehouse:locations:beijing`），再在子 key 内搜索

---

## 19. Redis HyperLogLog：PFADD / PFCOUNT，基数统计，误差率

### 题目

Redis HyperLogLog 是什么？PFADD 和 PFCOUNT 的原理是什么？它的误差率如何计算？为什么 HyperLogLog 可以用极少的内存实现基数统计？

### 核心答案

**HyperLogLog 核心命令：**
```bash
PFADD page:uv "user_id_1" "user_id_2" "user_id_3"  # 添加用户
PFCOUNT page:uv              # 获取独立用户数（估算值）
PFMERGE destination source1 source2  # 合并多个 HLL 的 UV
```

**HyperLogLog 原理（概率基数统计）：**
- 核心思想：对于一个随机均匀哈希函数，如果哈希结果的二进制表示中第一个 `1` 出现在第 K 位，则说明集合中至少有 `2^K` 个不同元素
- Redis 使用 **64 位哈希**（MurmurHash64），只关心二进制中连续 0 的个数
- 维护 **16384 个桶**（2^14），每个桶存储 **6 位寄存器**（最大值 63），记录该桶内观察到的最大 0 前缀
- 最终基数估算：`PF estimation = 2^register_avg * 16384 * 调和平均数修正`
- 内存占用：固定 `16384 * 6 / 8 = 12KB`（无论统计多少元素）

**为什么只需 12KB：**
| 对比 | 记录 2^64 个元素需要内存 |
|-----|----------------------|
| 精确计数（HashSet） | 几十 GB |
| HyperLogLog | 12KB |

**误差率：**
- Redis HyperLogLog 标准误差率：**0.81%**（`1.04 / sqrt(桶数)` = `1.04 / sqrt(16384)` = `0.81%`）
- 误差与集合大小无关，永远是 ±0.81%
- 对于 WMS 统计 UV、DAU（日活）场景，0.81% 误差完全可接受

**PFADD 实现细节：**
```bash
PFADD key element [element ...]
# 每个元素 MurmurHash64 得到 64 位哈希
# 取低 14 位决定桶编号（0~16383）
# 取高 50 位统计第一个 1 的位置，更新该桶的寄存器
# 返回 1 表示寄存器被更新（即至少有一个新元素），返回 0 表示无变化
```

**PFCOUNT 实现：**
```bash
PFCOUNT key [key ...]
# 读取所有 key 对应的 16384 个桶
# 对每个桶取调和平均数
# 乘以桶数得到估算基数
# 多 key 时先 MERGE 再 COUNT
```

**使用场景：**
- UV 统计（页面/接口独立访客数）
- DAU 统计（日活用户数）
- 仓库设备在线数统计
- 不适合需要精确计数的场景（如库存数量）

**调优参数：**
- `hyperloglog-max-memory`（默认 1KB）：每个 HLL 最大内存
- 误差不可配置，精度由桶数固定决定

### 追问方向

- HyperLogLog 为什么需要 16384 个桶？桶数和精度是什么关系？
- PFADD 时，如果某个元素的哈希值低 14 位落在已有最多零的桶里，会发生什么？
- PFCOUNT 多 key 合并时，是精确合并还是估算合并？
- HyperLogLog 的误差在什么场景下不可接受？

### 避坑提示

- ❌ 用 HyperLogLog 统计精确数量——它是概率算法，不保证精确
- ❌ PFCOUNT 返回整数以为它是精确的——返回值是估算值，误差 ±0.81%
- ❌ 在小数据集上使用 HyperLogLog——数据量小时，用 Set 更精确且内存差别不大
- ✅ 最佳实践：UV/DAU 等"允许少量误差"的统计场景用 HyperLogLog；精确计数用 Set 或 Bitmap

---

## 20. Redis 安全：密码认证、命令重命名、ACL 细粒度权限

### 题目

Redis 有哪些安全机制？如何配置密码认证和命令重命名？Redis 6.0 的 ACL 权限控制系统有哪些能力？如何避免生产环境 Redis 被攻击？

### 核心答案

**密码认证（Redis 3.0+）：**
```bash
# 配置文件设置密码
requirepass your_password

# 客户端连接后认证
AUTH your_password

# 命令行单次认证
redis-cli -a your_password   # 不推荐，会暴露在 shell 历史

# CONFIG SET 运行时修改（临时，重启失效）
CONFIG SET requirepass "new_password"
```

**命令重命名（`rename-command` 配置）：**
```bash
# 配置文件
rename-command FLUSHALL ""           # 禁用 FLUSHALL
rename-command KEYS ""                # 禁用 KEYS（生产必须）
rename-command CONFIG "CONFIG_ABC123" # 重命名 CONFIG
rename-command SHUTDOWN "SHUTDOWN_MY" # 重命名 shutdown
```
- 禁用比重命名更安全——即使重命名后，攻击者可以通过 `FLUSHALL` 别名执行
- 重命名后，客户端必须用新命令名访问

**ACL 权限控制（Redis 6.0+）：**
```bash
# 创建用户
ACL SETUSER alice ON >password ~cached:* -@all +GET +SET +DEL
# 解释：用户 alice，密码 password，允许操作 cached:* 前缀的 key，只允许 GET/SET/DEL 命令

# 权限符号
# + 添加命令
# - 禁用命令
# @command category（如 @read @write @admin @dangerous）
# ~ key pattern（如 ~cache:* ~session:*）
# > password
# off 禁用用户
# nocommands 禁用所有命令

# 权限分类
@read: GET, MGET, HGET, HGETALL, LRANGE, SMEMBERS, ZRANGE ...
@write: SET, SETNX, HSET, INCR, LPUSH, SADD, ZADD ...
@admin: CONFIG, SAVE, BGSAVE, FLUSHALL, SYNC, REPLICAOF ...
@dangerous: FLUSHALL, FLUSHDB, KEYS, SHUTDOWN, DEBUG ...

# 查看当前连接用户权限
CLIENT LIST   # 查看所有客户端
CLIENT KILL   # 杀掉可疑连接

# 查看 ACL 规则
ACL LIST       # 列出所有用户及规则
ACL GETUSER username
```

**默认用户（default user）：**
- Redis 6.0+ 新默认用户 `default`：有完整权限，密码为空（可设置密码或禁用）
- 可以创建只读用户、监控用户等

**生产环境安全最佳实践：**

| 措施 | 说明 |
|-----|------|
| 设置 `requirepass` | 必须设置强密码 |
| `bind 127.0.0.1` | 仅允许本地访问，禁止公网 |
| `protected-mode yes` | 2.6 后默认开启，无密码时只接受本地连接 |
| `rename-command` | 生产必须禁用 `KEYS/FLUSHALL/FLUSHDB/CONFIG` |
| `acl-dir /path` | 将 ACL 规则写入文件，`ACL LOAD` 加载 |
| 网络层 | 防火墙、VPC 网络隔离，不暴露公网端口 |
| 最小化用户权限 | 给业务应用分配只读或按需权限的 ACL 用户 |
| 定期轮换密码 | 通过 `ACL SETUSER` 定期更新密码 |
| 禁用危险命令 | `FLUSHALL/FLUSHDB/KEYS/CONFIG/SCAN` |

**Redis 6.0 额外安全特性：**
- **ACL per connection**：每个连接可认证不同用户
- **ACL + TLS**：支持 TLS 加密连接
- **RESP3 协议**：更多数据类型，减少数据转换风险
- **Re山上**：`no-ignore-warnings` 配置

**WMS 生产安全配置示例：**
```bash
# 业务应用 Redis 用户（仅读缓存和部分写）
ACL SETUSER wms_app ON >WmsStrongPass2024 \
  ~stock:* ~order:* ~warehouse:* \
  +@read +@write +INCR +DECRBY +EXPIRE +EXISTS \
  -@dangerous -@admin -FLUSHALL -FLUSHDB -KEYS -CONFIG -DEBUG

# 运维 Redis 用户（完全权限）
ACL SETUSER wms_admin ON >AdminStrongPass2024 \
  +@all -@dangerous
```

### 追问方向

- `protected-mode` 和 `bind` 的关系是什么？bind 127.0.0.1 时 protected-mode 还有效吗？
- `rename-command CONFIG ""` 禁用 CONFIG 后，运维如何临时修改配置？
- ACL 的 `~key_pattern` 支持通配符吗？`~*` 和 `~cached:*` 有什么区别？
- Redis 6.0 的 ACL 如何与外部认证系统（如 LDAP）集成？

### 避坑提示

- ❌ 以为设置 `requirepass` 就安全了——内网攻击、命令注入仍可能突破
- ❌ 生产环境不禁用 KEYS——任何 KEYS 操作都会阻塞 Redis
- ❌ bind 0.0.0.0 且无密码——相当于公网裸奔，会被植入挖矿程序
- ✅ 最小权限原则：每个应用用专用 ACL 用户，禁止共享 default 用户

---

> 本文档共 20 道 Redis 深度面试题，涵盖数据结构、持久化、集群、安全等核心知识。建议配合 WMS 仓库管理系统源码（`/usr/src/`）中的 Redis 使用案例复习。

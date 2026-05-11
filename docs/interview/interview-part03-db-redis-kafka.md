# Database + Middleware + System Design Interview Questions
## Java Full-Stack Developer (3-5 Years Experience)

---

## 1. MySQL / 数据库

### Q1. 请解释B+树的索引结构，为什么MySQL选择B+树而不是B树或二叉树？ [Basic]

B+树是一种多路平衡查找树，所有数据都存储在叶子节点，且叶子节点之间通过链表相连。非叶子节点只存储索引（键），这使得每个节点可以存储更多的索引指针，树的高度更低，磁盘IO次数更少。相比B树，B+树的叶子节点包含所有数据且有序，范围查询更高效。相比二叉树，B+树的树高更小（通常3-4层），减少磁盘IO次数，适合数据库存储。

**追问：什么是索引命中？什么情况下索引会失效？**

---

### Q2. 什么是最左前缀原则？举例说明如何利用它进行索引优化？ [Basic]

最左前缀原则是指复合索引从最左边开始匹配，查询条件必须包含索引的最左列才能使用该索引。例如对于索引(idx_a_b_c) on (a, b, c)，查询条件(a=1 AND b>2 AND c=3)可以使用索引的(a,b)部分，但(b=1 AND c=2)无法使用索引。优化建议是将区分度高的列放在前面，范围查询列放在最后。

**追问：如果查询是 WHERE a=1 AND c=1，能使用索引吗？**

---

### Q3. 什么是覆盖索引？如何利用它减少回表查询？ [Intermediate]

覆盖索引是指查询的所有列都包含在索引中，不需要回表查询主键索引。例如对于SELECT id, name FROM users WHERE name='John'，如果存在索引(name, id)，则可以直接从索引返回数据，避免回表。可以通过EXPLAIN的Extra列显示Using index来确认是否使用了覆盖索引。

**追问：如何判断一个查询是否需要回表？**

---

### Q4. 请解释MySQL事务的ACID特性，以及InnoDB如何实现这些特性？ [Basic]

ACID指原子性(Atomicity)、一致性(Consistency)、隔离性(Isolation)、持久性(Durability)。InnoDB通过MVCC（多版本并发控制）保证隔离性，通过redo log保证持久性，通过undo log保证原子性，通过锁机制辅助隔离性实现。事务开始时会创建一个快照，读已提交和可重复读隔离级别都依赖MVCC实现。

**追问：可重复读隔离级别下，事务开始时的快照是基于事务ID还是基于开始时间？**

---

### Q5. MySQL的四种隔离级别分别是什么？各有什么特点？ [Basic]

READ UNCOMMITTED（读未提交）：最低级别，存在脏读问题，事务可以看到其他未提交事务的修改。READ COMMITTED（读已提交）：只能看到已提交事务的修改，但不可重复读。可REPEATABLE READ（可重复读）：MySQL默认级别，事务期间多次读取结果一致，但可能有幻读问题（InnoDB通过MVCC和间隙锁解决幻读）。SERIALIZABLE（串行化）：最高级别，事务强制顺序执行，完全避免幻读，但性能最差。

**追问：MySQL默认的隔离级别是什么？Oracle默认的呢？**

---

### Q6. 什么是MVCC？它是如何工作的？ [Intermediate]

MVCC（Multi-Version Concurrency Control）通过保存数据在某个时间点的快照来实现。每个数据行有两个隐藏列：创建版本号和删除版本号。读操作根据事务ID判断可见性：读已提交每次SELECT创建新快照，可重复读事务开始时创建快照。INSERT时设置创建版本号为当前事务ID，DELETE时将删除版本号设为当前事务ID，UPDATE相当于DELETE+INSERT。

**追问：undo log在MVCC中扮演什么角色？**

---

### Q7. 什么是脏读、不可重复读和幻读？分别在什么隔离级别下发生？ [Basic]

脏读：读到其他事务未提交的数据，只在READ UNCOMMITTED下发生。不可重复读：同一个事务中两次读取同一行数据结果不同，因其他事务提交了修改，在READ COMMITTED下发生。幻读：同一事务中两次查询返回的记录数不同，因其他事务插入了新记录，在READ COMMITTED和REPEATABLE READ下都可能发生，但InnoDB在REPEATABLE READ下通过间隙锁解决了幻读问题。

**追问：如何避免脏读？事务隔离级别如何设置？**

---

### Q8. MySQL的锁机制有哪些类型？行锁和表锁分别什么时候使用？ [Intermediate]

MySQL InnoDB支持行锁和表锁。行锁锁定单行数据，锁开销大但并发度高，包括共享锁(S)和排他锁(X)。表锁锁定整张表，锁开销小但并发度低，包括意向共享锁(IS)和意向排他锁(IX)。行锁在查询条件命中索引时使用，表锁在无索引查询、锁溢出或使用LOCK TABLE时触发。InnoDB行锁基于索引实现，如果查询不走索引会升级为表锁。

**追问：什么是意向锁？它的作用是什么？**

---

### Q9. 什么是间隙锁(Gap Lock)和临键锁(Next-Key Lock)？ [Advanced]

间隙锁锁定索引记录之间的间隙，防止幻读。例如对于有100, 200, 300值的索引，间隙锁锁定(-∞,100)、(100,200)、(200,300)、(300,+∞)。临键锁是记录锁和间隙锁的组合，锁定一个前开后闭的区间。InnoDB在REPEATABLE READ隔离级别下使用临键锁作为默认行锁算法，防止幻读。

**追问：什么情况下间隙锁不会产生？**

---

### Q10. 什么是死锁？如何检测和处理MySQL死锁？ [Intermediate]

死锁是两个或多个事务相互等待对方持有的锁，导致循环依赖。InnoDB使用等待图(Wait-for Graph)算法检测死锁，当检测到死锁时会主动回滚undo量最小的事务。避免死锁的方法：按固定顺序访问表；减少锁持有时间；使用低隔离级别；尽量使用索引而非全表扫描。

**追问：如何查看MySQL的死锁日志？**

---

### Q11. 如何分析慢查询？EXPLAIN各列分别代表什么？ [Intermediate]

使用EXPLAIN分析SQL执行计划，重要列包括：type（访问类型，从好到差：system>const>eq_ref>ref>range>index>ALL）；key（实际使用的索引）；rows（扫描行数）；Extra（额外信息，如Using filesort、Using temporary、Using index）。还可以使用SHOW PROFILE查看SQL各阶段耗时。

**追问：什么是EXPLAIN ANALYZE？它和普通EXPLAIN有什么区别？**

---

### Q12. 什么是SQL优化？常见的SQL优化手段有哪些？ [Basic]

SQL优化包括：选择合适索引、避免SELECT *、减少JOIN、使用LIMIT分页、避免子查询改用JOIN、用EXISTS代替IN、避免LIKE以%开头、使用覆盖索引、减少函数和计算、批量操作代替循环单条。生产环境中通过开启慢查询日志、配置performance_schema、使用工具如pt-query-digest分析。

**追问：为什么应避免在索引列上使用函数或计算？**

---

### Q13. 什么是分库分表？常见的分片策略有哪些？ [Intermediate]

分库分表是解决单库单表数据量过大的方案。垂直拆分按业务将表或字段分到不同库/表；水平拆分将数据按某个字段（如用户ID、时间）分散到多个库/表。常见分片策略：哈希分片（ID%N）、范围分片（按时间/ID区间）、一致性哈希（减少扩缩容数据迁移）。中间件有ShardingSphere、MyCat。

**追问：分库分表后，跨分片查询如何实现？**

---

### Q14. 什么是读写分离？如何实现？ [Basic]

读写分离将读操作分流到从库，写操作到主库，实现读写分离提高性能。主从复制基于binlog实现，从库IO线程拉取主库binlog并写入relay log，SQL线程重放relay log。Java中可以使用ShardingSphere-JDBC、Spring的AbstractRoutingDataSource、或使用工具如MySQL Proxy。

**追问：主从延迟如何处理？什么情况下会出现延迟？**

---

### Q15. HikariCP连接池的工作原理是什么？核心参数有哪些？ [Intermediate]

HikariCP使用ThreadLocal+ConcurrentBag管理连接，复用连接避免频繁创建销毁。核心参数：maximumPoolSize（最大连接数，通常设置为CPU核心数*2+磁盘IO线程数）、minimumIdle（最小空闲连接）、connectionTimeout（等待连接超时，默认30秒）、idleTimeout（空闲连接超时）、maxLifetime（连接最大生命周期）、cachePrepStmts（启用预编译语句缓存）。

**追问：如何根据实际业务场景调优HikariCP参数？**

---

### Q16. MySQL的redo log和undo log分别是什么？它们的作用？ [Intermediate]

redo log记录物理修改（页面修改），用于崩溃恢复，保证持久性。事务提交前先将redo log写入磁盘，宕机后重做未刷盘的修改。undo log记录逻辑修改（SQL的逆操作），用于事务回滚和MVCC，存储在系统表空间中。redo log用于恢复已提交事务，undo log用于回滚未提交事务和读取历史版本。

**追问：为什么redo log可以提高数据库性能？**

---

### Q17. 什么是数据库连接池？为什么要使用连接池？ [Basic]

数据库连接池预先创建并管理一批数据库连接，应用程序需要时从池中获取，使用完毕后归还池中而非关闭连接。这样避免每次请求都创建销毁连接的开销，减少数据库服务器压力。常用连接池有HikariCP（性能最优）、Druid（功能丰富，包含监控）、DBCP、C3P0。

**追问：连接池会带来什么问题？比如数据库连接泄漏如何排查？**

---

### Q18. MySQL如何实现分布式事务？ [Advanced]

MySQL支持XA事务（两阶段提交）实现分布式事务，通过XA START/END/PREPARE/COMMIT命令配合使用。但XA事务性能开销大，更常用的是柔性事务：TCC（Try-Confirm-Cancel）通过业务代码实现补偿；Saga模式通过编排子事务的逆操作；可靠消息最终一致性方案（如RocketMQ事务消息）。Seata是常用的分布式事务解决方案。

**追问：2PC和3PC的区别是什么？为什么3PC更高效？**

---

### Q19. 什么是数据库的雪崩、穿透和击穿？ [Intermediate]

缓存穿透：查询不存在的数据，绕过缓存直击数据库，可使用布隆过滤器或缓存空值解决。缓存击穿：热点key过期瞬间大量请求涌入数据库，可使用互斥锁或永不过期+异步更新解决。缓存雪崩：大量缓存同时过期或缓存服务宕机，可使用多级缓存、过期时间随机化、熔断降级、Redis Cluster高可用解决。

**追问：如何在Redis中实现一个互斥锁防止缓存击穿？**

---

### Q20. 请解释MySQL的COUNT(*)和COUNT(1)性能差异？ [Intermediate]

在InnoDB引擎下，COUNT(*)和COUNT(1)性能几乎相同，MySQL优化器会自动选择最快速的计数方式。COUNT(*)会选取最小的索引统计行数，通常直接走二级索引。真正影响性能的是是否需要过滤条件和表的大小，而非COUNT的具体形式。COUNT(col)会统计非NULL值，性能略差。

**追问：为什么建议在主键上进行COUNT操作？**

---

## 2. Redis

### Q21. Redis支持哪些数据结构？分别适用于什么场景？ [Basic]

String（字符串）：缓存、分布式锁、计数器。Hash（哈希）：对象存储、购物车。List（列表）：消息队列（lpush/brpop）、最新列表、排行榜。Set（集合）：标签系统、去重、共同好友。ZSet（有序集合）：排行榜、延迟队列、权重队列。Bitmap：签到、用户在线状态。HyperLogLog：UV统计。Geospatial：地理位置。

**追问：ZSet的底层实现是什么？为什么使用跳表而不是红黑树？**

---

### Q22. Redis的SDS（简单动态字符串）相比C字符串有什么优势？ [Advanced]

SDS结构包含：len（已用长度）、alloc（分配空间）、flags（类型）、buf（实际存储）。优势：O(1)获取字符串长度（不遍历）；避免缓冲区溢出（自动扩容）；减少内存重新分配（空间预分配和惰性释放）；二进制安全（不仅限文本）。

**追问：SDS的扩容策略是什么？**

---

### Q23. Redis的持久化机制有哪些？RDB和AOF的区别？ [Basic]

RDB（Redis Database）：定时生成数据快照，fork子进程遍历内存写入RDB文件，恢复快但可能丢失最后一次快照后的数据。AOF（Append Only File）：记录每个写操作到文件，通过重写(rewrite)压缩文件。可配置always/everysec/no三种策略。生产环境通常使用RDB+AOF混合持久化。

**追问：为什么Redis使用子进程而不是线程进行RDB持久化？**

---

### Q24. Redis的过期策略有哪些？惰性删除和定期删除的区别？ [Basic]

惰性删除：访问key时检查是否过期，过期则删除。优点是节省CPU，缺点是过期key长期占用内存。定期删除：每隔一段时间扫描expires字典随机检查部分key，删除过期key。默认每100ms检查有过期时间的key。Redis同时使用两种策略，实际是惰性删除作为兜底，定期删除控制内存。

**追问：Redis的8种淘汰策略是什么？**

---

### Q25. 什么是Redis的LFU和LRU淘汰算法？如何配置？ [Intermediate]

LRU（Least Recently Used）：最近最少使用，淘汰最长时间未被访问的key。LFU（Least Frequently Used）：最不经常使用，淘汰访问频率最低的key。Redis 4.0引入LFU，通过volatile-lfu和allkeys-lfu两种淘汰模式。可以通过maxmemory-policy配置淘汰策略，maxmemory-samples配置采样数量平衡精度和性能。

**追问：如何选择LRU和LFU？在什么场景下LFU更优？**

---

### Q26. Redis的缓存穿透及解决方案？ [Basic]

缓存穿透：查询不存在的数据（如恶意请求或业务漏洞），每次都直击数据库。解决方案：接口参数校验；布隆过滤器（内存高效但有误判）；缓存空值（TTL设置短）；Google的Guava布隆过滤器或RedisBloom插件。

**追问：布隆过滤器的原理？它的误判率如何计算？**

---

### Q27. Redis的缓存击穿及解决方案？ [Intermediate]

缓存击穿：热点key过期瞬间，大量并发请求涌入数据库。解决方案：互斥锁（setnx+过期时间）；永不过期+异步更新（逻辑过期）；双缓存（主从）；热点数据永不过期。互斥锁实现：SETNX key value EX 10，只允许一个线程回源加载数据。

**追问：互斥锁方案会降低系统并发能力，如何优化？**

---

### Q28. Redis的缓存雪崩及解决方案？ [Intermediate]

缓存雪崩：大量缓存同时过期或Redis集群宕机，导致数据库压力骤增。解决方案：过期时间随机化（+随机1-5秒）；多级缓存（本地+Redis）；Redis高可用部署（哨兵/集群）；服务降级/熔断（Hystrix/Sentinel）；预热热点数据。

**追问：如何设计一个缓存架构能够应对Redis完全宕机的情况？**

---

### Q29. Redis主从复制原理是什么？ [Intermediate]

主从复制流程：从库发送PSYNC命令给主库；主库fork子进程生成RDB文件并发送；从库接收并加载RDB文件；主库将复制缓冲区中的增量数据发送给从库。Redis 2.8+支持增量复制，通过master_repl_offset和replica_offset维护复制进度。复制风暴问题可通过树形拓扑解决。

**追问：从库宕机恢复后如何增量同步？**

---

### Q30. Redis哨兵模式的工作原理？ [Advanced]

哨兵集群通过PING/INFO/PUBLISH命令监控主从节点健康。主观下线（SDown）：单个哨兵认为主节点不可用。客观下线（ODown）：多个哨兵投票认为主节点不可用。故障转移：由选举产生的领头哨兵执行，选择优先级最高/偏移量最大/runid最小的从库升级为新主库，其他从库重新配置主从关系。

**追问：哨兵模式下客户端如何感知主从切换？**

---

### Q31. Redis Cluster集群原理？数据如何分布和访问？ [Advanced]

Redis Cluster采用16384个槽位进行数据分片，通过CRC16(key) % 16384计算槽位。集群中每个节点负责一部分槽位，客户端访问时先向任意节点获取槽位信息（MOVED重定向），然后直接访问目标节点。集群支持节点间通信(Gossip协议)，自动发现节点和故障转移。最小集群需要6个节点（3主3从）保证高可用。

**追问：为什么Redis Cluster使用16384个槽而不是65536个？**

---

### Q32. Redis如何实现分布式锁？有什么问题？ [Intermediate]

简单实现：SET key value NX PX timeout，value使用唯一标识（如UUID），释放时用Lua脚本原子性检查并删除。问题：单机Redis锁不可靠（需RedLock）；时钟漂移；锁过期但业务未完成（需看门狗延期）。Redisson封装了这些细节，提供可重入锁、公平锁等。

**追问：Redisson的看门狗机制是什么？**

---

### Q33. Redisson的RLock相比SETNX有什么优势？ [Advanced]

RLock是Redisson实现的分布式锁，特性：可重入（基于Hash结构存储重入次数）；公平锁（按请求顺序获取）；等待锁自动延期（看门狗机制，默认30秒，每次续期1/3）；读写锁（支持读并发、写互斥）；信号量/CountDownLatch。

**追问：Redisson的公平锁如何实现？**

---

### Q34. Redis的大key问题如何处理？ [Intermediate]

大key指单个key存储数据过大（string类型value>10KB，多元素collection>10000元素）。发现方法：SCAN + STRLEN/SCARD；Redis-cli --bigkeys；monitor日志分析。处理方案：拆分大key（hash拆成多个field）；压缩value（serialize压缩）；异步删除（UNLINK代替DEL）；定期清理而非一次性删除。

**追问：删除大key会阻塞吗？UNLINK和DEL的区别？**

---

### Q35. Redis的热key问题如何处理？ [Advanced]

热key指访问频率极高的key，导致单节点压力过大。解决方案：Redis集群分片（热点key分散到多个节点）；热点key备份（对热key加随机后缀分发到多个节点）；应用层本地缓存（Guava/Caffeine）；监控预警（提前发现）。

**追问：如何识别热key？发现后如何处理？**

---

### Q36. Redis的事务支持如何？MULTI/EXEC/WATCH的原理？ [Intermediate]

Redis事务通过MULTI/EXEC/WATCH实现。MULTI开始事务，命令进入队列，EXEC执行队列中所有命令。WATCH实现乐观锁，监视key若在事务执行前被修改则事务被打回。Lua脚本原子执行多条命令，但不支持回滚（Redis作者认为这是缺陷也是特性）。Redis事务不能保证原子性，EXEC会执行所有命令。

**追问：为什么Redis事务不支持回滚？**

---

### Q37. Redis与Memcached的区别？何时选择哪个？ [Intermediate]

Memcached：纯内存、支持多线程、只支持string、无持久化、无集群支持。Redis：支持多种数据结构、有持久化、支持主从/集群、数据类型丰富。单值缓存选两者皆可；需要复杂数据结构、持久化、集群选Redis；纯并发高性能缓存场景可考虑Memcached。

**追问：Redis 6.0的多线程IO是什么？它和Memcached的多线程有何不同？**

---

### Q38. Redis的GEO命令如何实现附近的人功能？ [Advanced]

GEO使用GeoHash算法，将二维经纬度编码成一维分数存入ZSet。GEOADD添加位置，GEORADIUS/GEOSEARCH查询附近的人。GeoHash编码将地球划分为多个层级网格，每个网格有唯一编码，相邻区域GeoHash编码前缀相同，误差范围约2km-5000km。Redis通过zrem删除geo key。

**追问：GeoHash的精度如何计算？为什么相邻格子边界附近的查询可能不准确？**

---

### Q39. Redis的Pipeline有什么作用？与事务有何区别？ [Basic]

Pipeline是客户端将多个命令打包一次发送，减少网络往返次数提升性能，但命令之间无原子性保证。事务(MULTI/EXEC)是服务端原子执行多条命令，具有原子性但不支持回滚，且事务期间会阻塞其他命令。Pipeline适用于批量读取/写入无关联的场景，事务适用于需要原子性的场景。

**追问：Pipeline和Lua脚本的区别？**

---

### Q40. Redis的SCAN命令相比KEYS有什么优势？ [Intermediate]

KEYS会遍历整个键空间匹配模式，时间复杂度O(N)，在生产环境会阻塞Redis所有命令造成严重问题。SCAN使用游标分批次遍历，时间复杂度O(1)每次，支持COUNT参数控制每次返回数量，不会阻塞但可能重复/漏元素。生产环境必须用SCAN代替KEYS。

**追问：SCAN的COUNT参数是精确的吗？**

---

## 3. Kafka

### Q41. Kafka的整体架构是怎样的？核心概念有哪些？ [Basic]

Kafka是分布式消息队列，采用发布-订阅模式。核心概念：Broker（服务节点）、Topic（消息主题）、Partition（分区，物理存储单元）、Producer（生产者）、Consumer（消费者）、Consumer Group（消费组）、Offset（消息消费进度）。消息以追加方式写入分区日志文件，分区可跨Broker分布实现高可用。

**追问：Topic的Partition数和Consumer数量如何规划？**

---

### Q42. Kafka的生产者如何保证消息发送可靠性？ [Intermediate]

生产者通过acks配置可靠性：acks=0（只发不管到达率，性能最高）；acks=1（等待leader副本写入成功，可能丢数据）；acks=all/-1（等待leader+ISR所有副本写入成功，最可靠但延迟高）。还可以配置retries重试次数、enable.idempotence=true启用幂等producer防止重复。

**追问：什么是ISR？为什么ISR是动态变化的？**

---

### Q43. Kafka消费者的消费模型是什么？如何实现消息顺序消费？ [Intermediate]

Kafka采用拉取(pull)模型，消费者主动从broker拉取消息。同一Consumer Group内消息被分区分配，一个分区只能被一个消费者消费，实现负载均衡。不同Consumer Group独立消费，互不影响。顺序消息保证需要：单分区+单消费者+同步发送。

**追问：Consumer Group的分区分配策略有哪些？**

---

### Q44. Kafka如何保证消息不丢失？ [Intermediate]

消息丢失的三个环节：生产者发送丢失（配置acks=all+重试）；Broker存储丢失（副本机制，replication.factor>=3，unclean.leader.election.enable=false）；消费者丢失（手动提交offset，先处理再提交）。关键配置：min.insync.replicas>=2、replication.factor>=3、retries配置合理值。

**追问：如何检测消息是否丢失？**

---

### Q45. Kafka如何处理消息重复消费？幂等消费者如何实现？ [Advanced]

Kafka At Least Once（至少一次）模式可能重复消费。解决方案：业务幂等（数据库唯一索引/Redis去重/消息表）；开启幂等producer（enable.idempotence=true）防止生产者重复；消费者手动提交offset+业务处理成功才提交；使用事务（producer事务+consumer事务）。

**追问：为什么Kafka默认是At Least Once而不是Exactly Once？**

---

### Q46. Kafka的消息分区策略有哪些？如何自定义分区？ [Intermediate]

默认分区策略：轮询（RoundRobin，保证负载均衡）；按key哈希（Hash(key)%numPartitions，相同key进同一分区）；粘性分区（Sticky Partition，批次内轮询）。自定义分区实现Partitioner接口，重写partition方法。分区数建议设置为Broker数量倍数，便于负载均衡。

**追问：为什么生产环境不建议过多Partition？**

---

### Q47. Kafka如何实现事务消息？ [Advanced]

Kafka事务于0.11引入，用于 Exactly Once 语义。Producer事务：使用transactionId配合幂等producer，开启enable.idempotence+transactional.id配置。Consumer事务：隔离模式isolation.level=read_committed可跳过未提交事务消息。Kafka事务相比RocketMQ较轻量，用于跨分区原子写。

**追问：Kafka事务和幂等producer的区别？**

---

### Q48. Kafka的零拷贝(Zero Copy)原理？ [Advanced]

传统数据发送：磁盘→内核缓冲区→用户空间→Socket缓冲区→网卡，需4次上下文切换和4次数据拷贝。零拷贝：磁盘→内核缓冲区→网卡，通过transferTo()直接传递，利用DMA（直接内存访问）。Kafka使用Java的FileChannel.transferTo()实现零拷贝，大幅提升性能。

**追问：Kafka哪些场景无法使用零拷贝？**

---

### Q49. Kafka的Rebalance机制是什么？有哪些触发条件？ [Intermediate]

Rebalance是Consumer Group内分区所有权重新分配的过程。触发条件：消费者加入/离开；订阅Topic数量变化；消费者心跳超时（session.timeout.ms）。Rebalance期间无法消费，有STABLE状态保护。问题：分区分配不均；消费暂停；重平衡风暴。可通过合理配置max.poll.interval.ms、heartbeat.interval.ms优化。

**追问：如何避免Rebalance？**

---

### Q50. Kafka顺序消息如何实现？ [Intermediate]

Kafka只保证分区内的顺序，不保证Topic级别的全局顺序。实现顺序消息：发送端指定同一key到同一分区；消费端单线程消费或自定义顺序处理；业务层加序号和版本号。RocketMQ支持消息队列（MessageQueueSelector）实现严格顺序。

**追问：Kafka分区内的消息顺序性如何保证？**

---

### Q51. Kafka的Lag是什么？如何监控和处理？ [Intermediate]

Lag是消费者消费进度滞后的字节数，Lag=HW-消费offset。Lag过大说明消费能力不足或生产者发送过快。监控：Kafka自带JMX指标；Kafka Eagle；Prometheus+Grafana。处理：增加Consumer并发；优化消费逻辑；增加分区数；扩容消费者。

**追问：HW和LEO是什么？它们的作用？**

---

### Q52. Kafka的日志存储结构是怎样的？ [Intermediate]

每个Partition对应一个日志目录，目录下有多个日志分段(segment)，每个segment包含.index（索引文件）、.timeindex（时间索引）、.log（数据文件）。索引稀疏存储，定时定位到目标偏移量所在的segment后顺序扫描。日志保留策略：按时间（retention.ms）或大小（retention.bytes）。

**追问：为什么Kafka能支持高吞吐？**

---

### Q53. Kafka如何实现高可用？ [Advanced]

多副本机制：每个Partition有N个副本，分布在不同Broker；ISR列表保持与Leader同步的副本；Leader故障时从ISR中选举新Leader；不完全选举配置unclean.leader.election.enable=false防止数据丢失。Controller管理整个集群元数据，Broker宕机由Controller触发选举。

**追问：Controller是如何选举的？Controller故障怎么办？**

---

### Q54. Kafka消息压缩有什么作用？支持哪些算法？ [Intermediate]

消息压缩在Producer端执行，Broker存储压缩消息，消费者自行解压。优点：减少磁盘占用和网络传输；提升传输效率。缺点：增加CPU开销。支持的算法：GZIP（压缩率最高，CPU开销大）；Snappy（平衡压缩比和速度）；LZ4（速度最快）；ZSTD（Twitter开发，压缩率高且速度不错）。

**追问：消息压缩会影响Kafka的Lag吗？**

---

### Q55. Kafka的Exactly Once语义如何实现？ [Advanced]

Kafka 0.11+通过幂等producer+事务实现Exactly Once。幂等producer：开启enable.idempotence=true，Producer发生重试时去重。事务：原子性写多个Partition，Consumer通过隔离级别跳过未提交消息。适用场景：跨Partition原子写入；Exactly Once消费+处理+写入。

**追问：Kafka事务的开销如何？适用于什么场景？**

---

### Q56. Kafka Streams是什么？有什么特点？ [Intermediate]

Kafka Streams是基于Kafka的消息处理库，用于构建实时流处理应用。特点：轻量级（无需独立集群）；Exactly Once语义；状态存储（支持本地状态+changelog流）；低延迟。适合场景：实时数据管道、流处理、微服务间事件驱动通信。

**追问：Kafka Streams的State Store是什么？**

---

### Q57. Kafka与RocketMQ的区别？何时选择哪个？ [Intermediate]

Kafka：吞吐量最高（百万级TPS），生态成熟，适合大数据场景；但消费模型功能较少，延迟略高。RocketMQ：阿里开源，事务消息、分区顺序消息、延迟消息功能完善；消费模型更丰富（广播/集群）；延迟消息支持特定时间。事务消息场景选RocketMQ，大数据流处理选Kafka。

**追问：RocketMQ的事务消息原理是什么？**

---

### Q58. Kafka如何实现延迟队列？ [Advanced]

Kafka本身不支持延迟消息，业界方案：时间轮算法（ScheduableMessage）；按时间分Topic；RocketMQ延迟消息。Kafka时间轮实现：使用Set、List、HashMap三层结构，每层精度不同，通过DelayQueue驱动。

**追问：时间轮算法相比JDK DelayQueue的优势？**

---

## 4. Docker / K8s

### Q59. Dockerfile的常用指令有哪些？ [Basic]

FROM（基础镜像）、MAINTAINER（维护者）、RUN（执行命令构建镜像）、COPY（复制文件）、ADD（复制+解压）、WORKDIR（工作目录）、EXPOSE（暴露端口）、ENV（环境变量）、CMD/ENTRYPOINT（启动命令）。CMD会被docker run参数覆盖，ENTRYPOINT配置容器入口更规范。

**追问：COPY和ADD的区别？什么情况用ADD？**

---

### Q60. Docker镜像的分层结构原理？ [Intermediate]

Docker镜像采用分层(Layer)结构，每个指令创建新层，层可以被多个镜像共享。写时复制(COW)：容器层可写，镜像层只读，修改时复制到容器层。UnionFS（Overlay2）实现层的合并挂载。分层存储使得镜像构建、传输、存储高效，节省磁盘空间。

**追问：为什么尽量减少Dockerfile的层数？**

---

### Q61. Docker的存储驱动有哪些？Overlay2的原理？ [Advanced]

存储驱动：aufs（早期）、overlay、overlay2、devicemapper、btrfs、zfs。Overlay2：上层container、下层image，各层独立存在。同一文件在upper层可直接修改，不同文件merge后展示。生产环境推荐overlay2，性能和稳定性好。

**追问：Docker的文件系统是什么？容器和宿主机的文件系统关系？**

---

### Q62. Docker的网络模式有哪些？ [Basic]

bridge（默认，NAT模式，容器有独立IP）；host（共享宿主机网络命名空间，性能最好但端口冲突）；container（共享其他容器网络）；none（禁用网络）。自定义bridge可使用docker network create。overlay用于Docker Swarm跨主机通信。

**追问：如何让容器使用宿主机的网络命名空间？**

---

### Q63. Docker的持久化存储方案？ [Intermediate]

Docker提供两种存储机制：Volume（宿主机文件系统，docker volume create管理）；Bind Mount（绑定宿主目录）。生产环境推荐Named Volume，便于迁移备份。数据容器方式已过时。K8s中使用PersistentVolume和PersistentVolumeClaim。

**追问：如何备份和恢复Docker Volume？**

---

### Q64. Docker Compose的使用场景？ [Intermediate]

Docker Compose用于定义和运行多容器应用。通过docker-compose.yml定义服务、网络、卷。使用docker-compose up/down启动/停止所有服务。适用于本地开发、微服务架构演示、CI/CD测试环境。

**追问：docker-compose如何实现服务依赖和启动顺序？**

---

### Q65. K8s的核心概念有哪些？ [Basic]

Pod（最小调度单元，可包含多个容器）；Service（服务发现和负载均衡）；Deployment（无状态部署，声明式更新）；StatefulSet（有状态部署）；DaemonSet（每节点一个Pod）；Job/CronJob（批处理）；ConfigMap/Secret（配置管理）；Namespace（资源隔离）。

**追问：Pod和Container的区别？为什么K8s不直接管理Container？**

---

### Q66. K8s的核心组件有哪些？ [Intermediate]

etcd：分布式键值存储，保存集群所有状态。API Server：集群入口，接收请求并验证。Scheduler：调度Pod到合适节点。Controller Manager：运行各种控制器维护期望状态。Kubelet：节点代理，汇报节点状态。Kube-proxy：维护网络规则，服务发现。Container Runtime：运行容器（Docker/containerd）。

**追问：如果API Server挂了，集群还能正常工作吗？**

---

### Q67. K8s的Service类型有哪些？ [Intermediate]

ClusterIP（默认，内部集群IP）；NodePort（节点端口，30000-32767）；LoadBalancer（云厂商负载均衡）；ExternalName（CNAME映射）。Headless Service（clusterIP=None）用于有状态服务，让客户端直接连接Pod IP。

**追问：ExternalName和ClusterIP的区别？**

---

### Q68. K8s的Pod调度机制？亲和性和反亲和性？ [Advanced]

调度流程：预选(Predicates)过滤不符合条件节点；优选(Priorities)给节点打分；选择最高分节点绑定Pod。亲和性：nodeAffinity（节点亲和性）；podAffinity（Pod亲和性，同一拓扑域优先）；podAntiAffinity（Pod反亲和性，分散部署）。硬亲和性required，soft亲和性preferred。

**追问：Taints和Tolerations的作用？和亲和性有何区别？**

---

### Q69. K8s的Horizontal Pod Autoscaler (HPA)原理？ [Advanced]

HPA基于Metrics Server采集的指标自动扩缩容Pod副本数。指标类型：CPU利用率（requests百分比）、内存利用率、自定义指标（Prometheus）。工作流程：HPA Controller定期轮询→计算目标副本数→更新Deployment。扩缩容算法：ceil(当前副本数 * 当前指标/目标指标)。

**追问：HPA的冷却机制是什么？如何避免抖动？**

---

### Q70. K8s的Ingress是什么？如何工作？ [Intermediate]

Ingress是集群外部访问内部服务的HTTP/HTTPS路由规则。代替NodePort提供统一入口。Ingress Controller（如Nginx Ingress）解析Ingress规则并配置HTTP服务。IngressRule定义域名、路径、目标Service。TLS配置支持HTTPS。

**追问：Ingress和Service的区别？何时用Ingress？**

---

### Q71. K8s的ConfigMap和Secret如何使用？ [Intermediate]

ConfigMap存储非敏感配置，Secret存储敏感数据（Base64编码）。使用方式：环境变量注入；命令行挂载；Volume挂载文件。Secret有Opaque（自定义）、kubernetes.io/tls（证书）、docker-registry（镜像仓库认证）类型。生产环境建议配合外部秘密管理（Vault/AWS Secrets Manager）。

**追问：ConfigMap挂载和注入环境变量的区别？更新后生效方式？**

---

### Q72. K8s的Health Check机制？ [Intermediate]

存活探针(livenessProbe)：检测应用存活，失败重启容器。读inessProbe：检测应用就绪，失败从Service移除。启动探针(startupProbe)：应用启动慢时，初始阶段代替livenessProbe。探测方式：Exec（执行命令）；TCP（端口检测）；HTTP（GET请求）。健康检查影响Pod调度和流量分发。

**追问：为什么生产环境需要配置健康检查？**

---

### Q73. K8s的有状态部署StatefulSet特点？ [Intermediate]

StatefulSet特点：稳定的唯一网络标识（PVC保留hostname）；有序部署/扩展/删除（序号命名）；有序自动滚动更新。应用场景：数据库主从、有序集合、需要持久化的中间件。每个Pod有独立存储，通过volumeClaimTemplates动态创建PVC。

**追问：StatefulSet和Deployment的核心区别？什么场景必须用StatefulSet？**

---

### Q74. K8s的存储方案？PV/PVC/StorageClass？ [Intermediate]

PersistentVolume(PV)是集群级存储资源；PersistentVolumeClaim(PVC)是Pod对存储的请求；StorageClass动态 Provision PV。访问模式：ReadWriteOnce、ReadOnlyMany、ReadWriteMany。回收策略：Retain（保留手动清理）、Delete（自动删除）、Recycle（基本擦除）。

**追问：NFS存储和云厂商存储如何对接K8s？**

---

### Q75. K8s的安全机制？RBAC原理？ [Advanced]

RBAC基于角色(Role/ClusterRole)和角色绑定(RoleBinding/ClusterRoleBinding)控制权限。Role定义命名空间内权限，ClusterRole定义集群级权限。ServiceAccount是Pod访问API Server的身份，结合TokenReview API认证。SecurityContext设置容器运行特权、Capabilities。NetworkPolicy控制Pod网络流量。

**追问：如何限制Pod只能访问特定命名空间的服务？**

---

### Q76. K8s的资源配额和限制？LimitRange/ResourceQuota？ [Intermediate]

LimitRange设置命名空间内Pod/容器的默认和最大资源限制（CPU/内存）。ResourceQuota设置命名空间总资源配额（Pod数量、CPU/内存总量）。Pod可设置resources.requests（调度依据）和resources.limits（运行时限制）。超过limits会被OOMKill。

**追问：CPU limits设置为0意味着什么？**

---

### Q77. Docker和K8s的日志收集方案？ [Intermediate]

Docker日志驱动：json-file（默认）、syslog、fluentd、awslogs。K8s日志收集方案：节点级（DaemonSet部署Fluentd/Fluent Bit）；应用级（Sidecar容器收集）；应用内直推（SDK直推）。常见组合：ELK（Elasticsearch+Logstash+Kibana）、EFK（Elasticsearch+Fluentd+Kibana）、Loki。

**追问：如何排查K8s中Pod的日志？**

---

### Q78. K8s集群如何升级？Rolling Update原理？ [Intermediate]

K8s版本升级策略：小版本递进。升级流程：升级Master→升级Node→验证。Node升级方式：手动kubectl cordon/drain；使用kubeadm upgrade；使用集群管理工具（Rancher/kubesphere）。Rolling Update通过Deployment实现，逐步替换Pod，配置maxSurge和maxUnavailable控制更新节奏。

**追问：Rolling Update失败如何回滚？**

---

### Q79. K8s的网络模型原理？CNI插件？ [Advanced]

K8s网络模型要求：Pod间可直接通信（NAT）；Pod和Node可互通；Service ClusterIP仅集群内部可路由。CNI（容器网络接口）实现网络插件：Flannel（VXLAN简单方案）；Calico（高性能，支持网络策略）；Cilium（eBPF，深度集成Linux内核）。CNI负责Pod网络配置、IP分配、跨节点通信。

**追问：K8s的Service如何实现负载均衡？**

---

### Q80. K8s的Operator模式原理？ [Advanced]

Operator是K8s的自定义控制器，通过CRD（自定义资源定义）扩展K8s API。原理：Controller循环监控自定义资源状态→计算期望状态和实际状态差异→采取行动使两者一致。Operator SDK提供框架简化开发。常见Operator：Prometheus Operator、CERT-Manager、ExternalDNS。

**追问：为什么需要Operator？它和内置Controller的区别？**

---

## 5. 系统设计

### Q81. 如何设计一个秒杀系统？ [Advanced]

整体架构：前端限流（验证码/按钮防抖）→CDN缓存静态资源→网关层（限流熔断）→业务层（库存预扣减）→缓存层（Redis减库存）→消息队列（异步下单）→数据库层（最终扣减）。热点数据放缓存，库存分段，异步下单避免数据库压力。分布式锁保证库存一致性。

**追问：超卖问题如何解决？**

---

### Q82. 如何设计一个分布式ID生成器？ [Intermediate]

方案：UUID（无序，存储空间大）；数据库自增（单机OK，分布式需改造）；雪花算法（时间戳+机房ID+机器ID+序列号，趋势递增）。雪花算法优点：趋势递增、64位Long支持分片。挑战：时钟回拨（可通过记录上次时间解决）。

**追问：雪花算法在分布式环境下各部分含义？**

---

### Q83. 如何设计一个延迟任务系统？ [Intermediate]

方案：RabbitMQ延迟插件；Redis ZSet（score为执行时间）；时间轮算法（如Kafka）；JDK ScheduledThreadPool（单机）。分布式延迟任务：Redis ZSet + 多消费者竞争抢单；消息队列TTL+死信队列。考虑：任务持久化、失败重试、任务幂等性。

**追问：时间轮算法在Kafka中如何实现？**

---

### Q84. 如何设计一个接口幂等性方案？ [Intermediate]

幂等实现：前端防重复提交（token）；数据库唯一约束（如支付流水号）；Redis去重（SETNX）；乐观锁（version字段）；悲观锁。Token方案：下单前获取token，提交时携带token，Redis删除token返回成功表示处理。Token有生命周期需合理设置。

**追问：POST接口如何保证幂等？**

---

### Q85. 如何设计一个分布式Session方案？ [Intermediate]

方案：粘性Session（负载均衡器基于IP/cookie绑定）；Session复制（广播同步，樱桃大了性能差）；Session持久化（数据库/Redis）；Session集中管理（Spring Session + Redis，推荐）。最佳方案：Token/JWT无状态方案，JWT缺点是注销和续期复杂。

**追问：JWT和Session的区别？JWT适用场景？**

---

### Q86. 如何设计一个抢红包系统？ [Advanced]

资金类场景关键点：事务保证原子性；乐观锁（version或金额）防止超发；分布式锁（Redisson）。架构：Redis缓存用户余额，异步扣减确保一致性。红包算法：二倍均值法（每次随机金额=0.01~2*剩余金额/剩余人数），保证公平性。

**追问：如何防止并发同时抢同一个红包？**

---

### Q87. 如何设计一个排行榜系统？ [Intermediate]

Redis ZSet实现，score为分数，member为用户ID。ZINCRBY更新分数，ZREVRANGE获取TopN。分页用ZREVRANGE key start end。复杂需求：日榜/周榜/月榜用不同key前缀；需要历史排名用ZADD直接写入。更新频率高时考虑批量更新+定时刷新。

**追问：如何实现用户排名上升下降提醒功能？**

---

### Q88. 如何设计一个消息推送系统？ [Intermediate]

架构：设备token管理（存储设备标识和用户关联）；推送服务（HTTP/SDK调用三方）；离线消息（设备不在线时存储）；送达/点击回执。推送方式：在线（WebSocket/长连接）；离线（APNs/FCM）。消息优先级和分组。用户分群精准推送。

**追问：如何保证消息的送达率？**

---

### Q89. 如何设计一个商品库存系统？ [Intermediate]

库存扣减策略：下单时预占库存（Redis），支付后真正扣减。超卖问题：乐观锁UPDATE SET stock=stock-1 WHERE stock>0。库存回滚：超时释放（定时任务扫描未支付订单）；失败补偿。缓存一致性问题：Cache Aside + 延迟双删。

**追问：库存为0时用户还能下单如何处理？**

---

### Q90. 如何设计一个分布式锁？ [Intermediate]

Redis实现：SET key value NX PX timeout + 唯一value + Lua原子释放。问题：单机不可靠需RedLock（多机多数实现）。Redisson封装看门狗机制。数据库实现：唯一索引+过期时间。ZooKeeper实现：临时有序节点+Watch。ZK更可靠但性能低于Redis。

**追问：为什么ZooKeeper不适合高并发场景？**

---

### Q91. 如何设计一个接口限流方案？ [Intermediate]

限流算法：计数器（简单，有临界问题）；滑动窗口（Redis ZSet实现）；漏桶（恒定速率）；令牌桶（允许突发流量）。限流维度：IP、用户ID、接口、参数组合。分布式限流：Redis原子计数+Lua。接入层：Nginx的limit_req/limit_conn。应用层：Guava RateLimiter、Sentinel。

**追问：滑动窗口和令牌桶的区别？**

---

### Q92. 如何设计一个分布式事务方案？ [Intermediate]

方案选择取决于场景：同库事务无需分布式；跨库强一致选2PC/3PC（性能差）；最终一致选可靠消息（TCC/Saga）。TCC：Try预占资源、Confirm确认扣减、Cancel取消释放。Saga：正向编排+逆操作补偿。Seata AT模式：自动生成回滚SQL，对业务无侵入。

**追问：TCC的空回滚和悬挂问题如何解决？**

---

### Q93. 如何设计一个幂等支付系统？ [Advanced]

幂等key：订单号作为支付幂等key。流程：检查支付流水是否存在→存在直接返回成功→不存在则开启事务→创建流水→调用支付→更新状态。关键：唯一索引防止重复；状态机防重复回调；异步通知需幂等处理。

**追问：支付回调的幂等性如何保证？**

---

### Q94. 如何设计一个CDN缓存策略？ [Intermediate]

CDN缓存内容：静态资源（CSS/JS/图片/视频）；动态内容缓存（边缘计算，需cache-control或query string）。缓存策略：强制缓存（Cache-Control/Expires）；协商缓存（Last-Modified/ETag）。CDN架构：就近访问、智能DNS解析、回源策略。缓存失效：URL加密、版本号、命名空间。

**追问：如何解决CDN缓存的hit rate低的问题？**

---

### Q95. 如何设计一个秒杀优惠券系统？ [Intermediate]

优惠券发放：预生成券码+库存放Redis。领取限流：一人一券（用户ID+券ID唯一约束）。核销：订单完成后异步核销，需幂等。重复领取：Redis SET防止。优惠券归还：退单/过期后归还库存（定时任务+消息队列）。

**追问：如何防止黄牛党薅羊毛？**

---

### Q96. 如何设计一个长连接网关？ [Intermediate]

长连接类型：WebSocket、Socket.IO、SSE。网关功能：连接管理（连接映射、Session存储）；协议转换（私有协议↔HTTP）；消息路由（广播、单播、组播）；心跳检测（保活、断线重连）。实现：Netty（高性能NIO）；Spring WebFlux（非阻塞响应式）。分布式部署需Session同步或无状态设计。

**追问：长连接如何实现水平扩展？**

---

### Q97. 如何设计一个灰度发布系统？ [Intermediate]

灰度策略：用户ID取模、IP段、地区、Cookie、Header。实现方式：Nginx Lua动态路由；Spring Cloud Zuul/Gateway；Service Mesh(Istio)。渐进式发布：小流量验证→全量。回滚机制：配置中心推送关闭/降级规则；版本管理+快速切换。

**追问：如何做ABTest的流量分配？**

---

### Q98. 如何设计一个分布式链路追踪系统？ [Advanced]

核心概念：Trace（完整调用链）、Span（单个服务调用）。实现方式：Agent无侵入（JavaAgent埋点）；SDK侵入。数据上报：MQ缓冲减少性能影响；采样率控制。存储：Elasticsearch/TiDB。UI展示：调用拓扑图、调用链详情。开源方案：SkyWalking（APM）、Jaeger、Zipkin。

**追问：如何实现跨服务的上下文传递？**

---

### Q99. 如何设计一个配置中心？ [Intermediate]

核心功能：配置发布/读取/订阅/回滚；多环境多版本管理；配置加密；灰度发布。实现原理：发布时写入DB/配置存储→轮询/推送更新→SDK回调监听器→本地缓存。推送方式：长轮询（如Apollo）；MQ（如Nacos）。配置变更需幂等处理。

**追问：如何处理配置推送的延迟问题？**

---

### Q100. 如何设计一个高可用架构？ [Advanced]

高可用措施：冗余（多副本、多机房）；Failover（自动切换）；限流熔断（Sentinel/Hystrix）；降级（非核心功能关闭）；超时重试+幂等；灾备（数据异地容灾）。架构原则：无单点（所有组件高可用）；水平扩展；资源隔离；告警监控。演练：Chaos Engineering破坏性测试。

**追问：如何设计一个异地多活架构？**

---

## 6. WMS项目相关问题

### Q101. WMS系统的核心业务模块有哪些？请描述其架构 [Intermediate]

WMS核心模块：入库管理（收货→验收→上架）；出库管理（下架→复核→出库）；库内管理（盘点、移库、库位调整）；拣选管理（PDA扫描、RF操作）；增值服务（换标、拼盘、组合）。系统架构通常采用微服务，Spring Cloud或Spring Boot多模块，通过消息队列解耦。

**追问：WMS如何与ERP/TMS系统对接？**

---

### Q102. WMS的库位管理是如何设计的？库位编码规则 [Intermediate]

库位编码通常采用库区→通道→货架→层→位的层级结构，如A01-01-01-01代表A库区1号通道1号货架1层1位。库位属性：存储类型（存储区/拣选区/周转区）；温度类型（常温/冷藏/冷冻）；货物类型限制。库位状态：可用/占用/冻结/维护。库位容量管理：重量上限、体积上限、货格数量限制。

**追问：库位容量超限如何处理？**

---

### Q103. WMS的入库流程是怎样的？如何确保数据准确？ [Intermediate]

标准入库：送货预报→收货单创建→PDA扫描收货（数量核对）→验收（质量检查）→箱规/托盘绑定→RF上架（扫描库位）→系统过账。数据准确性保证：条码/RFID扫描校验；二次复核；批次管理；SN码管理（序列号追踪）；库存事务日志。

**追问：收货数量与送货单不符如何处理？**

---

### Q104. WMS的出库流程是怎样的？如何提高拣选效率？ [Intermediate]

出库流程：订单接收→订单波次（按规则合并订单）→生成拣选任务→库内拣选→复核打包→称重出库→运单绑定→发货确认。提高拣选效率策略：黄金拣选位（高频SKU放于人体工程学位置）；RF拣选优化路径；拣选单合并；分区拣选+汇总；边拣边分。

**追问：什么是波次拣选？波次策略有哪些？**

---

### Q105. WMS的库存冻结和解冻机制是如何设计的？ [Intermediate]

冻结场景：订单占用（预占库存）；质检冻结（质量待定）；盘点冻结（实物与系统差异待确认）；冻结规则配置（触发条件、冻结比例）。实现方式：库存表增加冻结字段或冻结类型表。解冻：人工确认解冻/自动解冻（条件触发）。冻品不影响正常库存扣减，但订单分配和发货时跳过冻结库存。

**追问：库存冻结和解冻如何保证事务一致性？**

---

### Q106. WMS的盘点功能是如何实现的？ [Intermediate]

盘点类型：明盘（停止作业全面盘点）；暗盘（不停业随机抽查）；动态盘点（边作业边盘点）。流程：创建盘点单→生成盘点任务→PDA扫描库位/SKU→系统核对数量→差异分析→调整审批→系统过账。差异处理：明盘差异直接调整；暗盘需复盘确认。ABC盘点法优化（重点商品高频盘点）。

**追问：盘点差异如何处理？是否需要回溯调整历史数据？**

---

### Q107. WMS如何处理库内移库和库位调整？ [Intermediate]

移库类型：库位调整（优化库位利用）；库区调拨（区域间移动）；库间调拨（仓库间移动）。移库流程：创建移库单→PDA扫描源库位→扫描目标库位→系统记录事务。库位调整：SKU与库位属性不符时触发。异常处理：目标库位容量不足、SN码必须一起移动、批次追溯。

**追问：移库过程中如何防止库存丢失？**

---

### Q108. WMS的订单同步和履约流程是怎样的？ [Intermediate]

订单来源：OMS/商城/ERP；订单同步方式：MQ实时推送/API轮询/定时同步。订单履约：订单审核→库存锁定→智能分仓→仓库分配→出库单下发→WMS处理→运单回传→订单完成。异常处理：缺货订单挂起/等待/拆单；超时未发货自动取消。退货处理：RMA流程→逆向物流→验收→退款/重新入库。

**追问：多仓库订单如何智能分配仓库？**

---

### Q109. WMS的库存预警机制有哪些？ [Intermediate]

库存预警类型：库存不足预警（低于安全库存）；库存积压预警（超过库存上限）；临期预警（效期低于阈值）；库位容量预警。实现方式：定时任务扫描+消息推送；实时计算（库存变动触发）。配置灵活度：按SKU设置预警规则；按供应商设置；按仓库设置。推送方式：邮件/短信/系统通知。

**追问：安全库存如何计算？有哪些算法？**

---

### Q110. WMS系统如何与PDA/RF设备集成？ [Intermediate]

PDA集成方式：RF（Radio Frequency）实时无线连接；应用程序通过HTTP/MQTT与WMS服务端通信。典型操作：入库扫描→收货/上架；出库扫描→拣选/复核；库内扫描→移库/盘点。关键点：条码规则（Code128/QR）；扫描响应速度（<1秒）；离线模式（断网续传）。

**追问：RF扫描的数据一致性如何保证？**

---

### Q111. WMS的批次管理如何实现？ [Intermediate]

批次属性：生产日期、失效日期、供应商、批次号、采购单号。批次策略：先进先出（FIFO）；后进先出（LIFO）；效期优先；指定批次。批次追溯：正向追溯（原材料→成品）；逆向追溯（成品→原材料）。批次检查：出库校验（是否指定批次）；入库校验（效期录入）。

**追问：多批次合并出库如何处理？**

---

### Q112. WMS的复核打包流程是怎样的？ [Intermediate]

复核方式：播种式（按订单分货，一个复核台对应多个拣选位）；摘果式（按商品遍历所有订单）；边拣边分（拣货同时完成分拨）。复核操作：扫描订单/运单→扫描商品校验→称重复核→打包贴标→完成确认。复核效率指标：复核通过率；复核速度（件/小时）；差错率。

**追问：复核发现数量不符如何处理？**

---

### Q113. WMS如何实现与AGV/自动化设备的集成？ [Intermediate]

集成方式：标准化接口（HTTP REST/MQTT）；WCS中间层系统调度。AGV调度：任务下发（WMS→WCS→AGV）；路径规划（WCS负责）；状态回传（AGV→WCS→WMS）。立库AS/RS集成：WMS生成入库任务→WCS执行堆垛机/输送线→WMS确认入库。设备异常处理：异常上报→人工介入→任务重试。

**追问：AGV拣选和人工拣选如何协同？**

---

### Q114. WMS的退货处理流程（RMA）是如何设计的？ [Intermediate]

RMA流程：退货预报→退货接收（扫描RMA单）→验收（检查退货原因/商品状态）→分类处理（可销售/残损/销毁）→退款/重新入库/报废。退货验收：与原订单SKU/数量核对；外观检查；功能测试。财务结算：原路退款/余额退回；退款金额与订单金额差异处理。退货率分析：退货原因统计。

**追问：已出库商品退货如何处理库存回滚？**

---

### Q115. WMS的效期管理如何实现？ [Intermediate]

效期管理策略：按批次管理效期；先进先出（FEFO）出库；临期预警；效期冻结（超期自动冻结）。效期规则：近效期定义（如30天）；超效期禁止出库；允售期计算。效期冻结：超效期系统自动冻结；临效期预警。报表：效期库存分布；滞销库存分析；效期预警报表。

**追问：FIFO和FEFO的区别？食品行业用什么策略？**

---

### Q116. WMS如何支持多货主管理？ [Intermediate]

多货主模式：库存按货主隔离；同一SKU可多货主共存。货主配置：货主信息（名称、协议、结算方式）；货主库存隔离规则；货主优先级。业务流程：订单指定货主或系统自动分配；入库分配货主；库存报表按货主统计。结算：货主库存租金/操作费；月结对账单。

**追问：多货主场景下如何避免超卖？**

---

### Q117. WMS的计费引擎是如何设计的？ [Intermediate]

计费项：仓储费（按库容/按件数）；操作费（收货/发货/翻包）；增值服务费（换标/贴标）；配送费；耗材费。计费规则：按SKU/按重量/按体积；阶梯计费；协议价格。结算方式：月结/预付费；费用优惠/减免。核销：费用审核→对账单生成→开票→付款。

**追问：仓储费如何避免重复计费？**

---

### Q118. WMS的报表分析体系有哪些？ [Intermediate]

库存报表：库存分布/周转率/呆滞料分析。作业报表：收发存汇总/作业效率/差错率。财务报表：费用明细/客户对账/利润分析。KPI报表：履约时效/库存准确率/复核通过率。BI分析：数据仓库+多维分析；实时大屏展示。报表生成：定时任务+实时查询结合。

**追问：如何设计一个库存准确率的计算公式？**

---

### Q119. WMS系统的高可用和容灾方案是怎样的？ [Intermediate]

高可用架构：多实例部署（无状态服务）；数据库主从+读写分离；Redis Cluster/Sentinel；消息队列集群。容灾方案：数据库异地备份；应用多机房部署；数据实时同步。双11等活动预案：系统扩容评估；历史数据清理；压测演练。应急预案：故障分级+响应SLA。

**追问：如何设计一个WMS的降级策略？**

---

### Q120. WMS系统如何保证7*24小时运行？ [Intermediate]

热部署/不停机发布：蓝绿部署；灰度发布。数据库维护：在线表结构变更（pt-online-schema-change）；大表DDL提前处理。数据清理：冷数据归档；日志表定期分区清理。定时任务优化：错峰执行；分布式任务调度；无锁设计。监控告警：系统监控+业务监控；异常自动告警+人工介入。

**追问：高峰期如何控制定时任务的执行？**

---

### Q121. WMS的条码/RFID管理是如何设计的？ [Intermediate]

条码规则：SKU条码（商品标识）；库位条码（位置标识）；箱条码（物流箱）；托盘条码。条码生成：系统自动生成；条码规则配置（前缀+流水号+校验位）。条码打印：标签模板设计；批量打印/按需打印。RFID应用：标签绑定；批量识别；门禁监控。条码校验：校验位算法（Luhn等）。

**追问：如何防止条码重复打印导致的问题？**

---

### Q122. WMS的货物组装/拆解功能如何实现？ [Intermediate]

组装（组合入库/库内组装）：BOM定义（父件+子件+数量）；组装任务→子件扣减+父件增加。拆解（拆零出库/库内拆解）：拆解规则；拆解任务→父件扣减+子件增加。库存事务：父子件库存联动。成本核算：组装成本=子件成本之和；拆解按比例分摊成本。

**追问：组装件和子件之间如何保持库存一致性？**

---

### Q123. WMS如何与输送线/分拣机集成？ [Intermediate]

集成方式：WMS生成包裹分拣指令→WCS控制设备→分拣机执行→反馈状态。分拣策略：按目的地/按配送商/按尺寸。分拣流程：称重+扫码→系统比对→分拣指令→分拣机执行。异常处理：无法识别→异常区；尺寸超限→人工处理。效率指标：分拣效率（件/小时）；分拣准确率。

**追问：输送线堵包如何处理？**

---

### Q124. WMS的货位优化策略有哪些？ [Intermediate]

优化目标：缩短拣选路径；提高库位利用率；降低劳动强度。优化策略：ABC分类（高频SKU放黄金拣选位）；相似SKU相邻存放；热卖商品前置。动态优化：移库任务+库位评分；定时任务推荐优化。揩选路径算法：S型路径；椎字型路径；最省路径算法。

**追问：库位优化任务如何避免影响正常作业？**

---

### Q125. WMS系统如何处理峰谷时段的差异？ [Intermediate]

高峰期策略：波次合并减少拣选次数；增加临时工+RF设备；拣选任务优先队列；延长作业时间。系统优化：缓存热点数据；异步处理非核心任务；数据库连接池调优。低谷期策略：盘点任务安排；库位优化执行；历史数据归档。弹性扩容：云部署弹性伸缩；作业任务错峰。

**追问：双11等大促期间WMS如何准备？**

---

### Q126. WMS的拣选路径算法有哪些？ [Intermediate]

常见算法：S型路径（跨通道往返）；椎字型/Z字型路径；越库路径；最短路径（TSP求解）。算法选择依据：仓库布局（单深度/双深度）；通道方向；拣选策略。路径优化：按巷道分组；黄金拣选位优先；订单优先级排序。软件实现：VRP（车辆路径问题）算法。

**追问：S型路径和Z型路径各适用于什么场景？**

---

### Q127. WMS如何实现精准的库存库容管理？ [Intermediate]

库容维度：重量（库位承重）；体积（库位容积）；货格数量（SKU件数上限）。动态库容：收货时计算（件数*单件体积）；预留库容（波次预留）。上架推荐：库位容量匹配；货品属性匹配（温度/批次）；库位使用率均衡。超容处理：预警提示；禁止上架；强制分托。

**追问：如何计算一个库位还能放多少货物？**

---

### Q128. WMS的RF拣选与Batch拣选各有什么优缺点？ [Intermediate]

RF拣选（按单拣选）：一人一单，边走边拣；复核单一订单；准确率高但效率低。Batch拣选（批量拣选）：一人多单，汇总后分拨；一次行走，多单共享；效率高但复核复杂。适用场景：RF适合订单量小SKU多；Batch适合订单相似度高。C电商平台：大促期间Batch优先。

**追问：订单相似度如何定义？**

---

### Q129. WMS系统如何实现SN码（序列号）管理？ [Intermediate]

SN码应用：3C/数码/汽车配件等需要单品追溯的商品。SN管理：入库扫描绑定批次；出库扫描核销；库存查询可精确到SN。业务流程：采购入库→扫描SN→关联供应商/批次；销售出库→扫描SN→关联订单/客户。SN校验：重复扫描预警；格式校验（预设规则）。

**追问：SN码如何实现全链路追溯？**

---

### Q130. WMS的越库作业（Cross Docking）是如何实现的？ [Intermediate]

越库模式：货物从入库到出库不经过存储区，直接分拨。适用场景：大批量同质化货物；供应商直送；门店配送。流程：送货预报→到达越库区→分拨指令→装车出库。时间窗口管理：到货时间窗口；发车时间窗口。系统协同：TMS配送调度+WMS越库指令。

**追问：越库模式可以提高效率但也带来风险，如何控制？**

---

### Q131. WMS如何处理多币种/多汇率的跨境业务？ [Intermediate]

多币种处理：商品SKU维护多币种价格；采购单/销售单指定币种；汇率日更新配置。结算方式：按下单汇率锁定；按发货日汇率结算；按月平均汇率。财务核算：本位币换算；汇率损失计入；毛利重算。关务对接：清关申报价值；HS编码管理。

**追问：汇率波动带来的成本差异如何处理？**

---

### Q132. WMS如何与海关系统对接实现保税管理？ [Intermediate]

保税仓库类型：公共保税仓；出口监管仓；自用保税仓。系统对接：申报数据报送（进库/出库/库存）；EDI报文或API接口。核销管理：账册管理（料号级）；核销周期；与非保税库存隔离。合规要求：实时库存报送；批次追溯；有效期管理。

**追问：保税转一般贸易如何操作？**

---

### Q133. WMS的质检管理模块如何设计？ [Intermediate]

质检触发：到货质检（IQC）；过程质检（IPQC）；出货质检（OQC）。质检项目：外观/数量/功能/效期/包装。质检流程：创建质检单→抽样方案→执行检验→结果判定→系统处理。质检结果：合格→直接入库；不合格→退供/降级/报废。质检规则：按SKU/按供应商/按批次设置抽检比例。

**追问：抽检比例如何确定？**

---

### Q134. WMS如何实现灵活的计费规则配置？ [Intermediate]

计费引擎设计：计费规则引擎（规则+公式+条件）；计费项模板（定义费用类型）；账单模板（定义对账单格式）。规则配置：按SKU/按货主/按仓库；阶梯计费（量多优惠）；协议价格。费用计算：事务触发（作业发生时）；定时计算（月结）。异常处理：费用调整/减免/追溯。

**追问：仓储费计算时库存数量如何确定？**

---

### Q135. WMS的逆向物流处理有哪些特点？ [Intermediate]

逆向物流特点：退货来源分散；处理流程复杂；时效要求低但成本高。退货处理：RMA申请→退货运回→仓库接收→分类处理→退款/入库/报废。特殊商品处理：危化品；冷藏品；高价值商品。残值处理：可销售品→重新上架；残次品→报废/拆解；部件回收。

**追问：逆向物流的成本如何核算？**

---

### Q136. WMS系统如何实现多语言/多时区支持？ [Intermediate]

多语言：国际化资源文件（i18n）；前端静态文案+后端动态消息分离；语言切换配置。多时区：数据库存储UTC时间戳；前端显示按用户时区转换；报表按时区汇总。时区问题：跨时区订单时间显示；跨时区库存变动时间。境外仓：本地货币/计量单位。

**追问：同一SKU在中英文仓库的库位编码如何统一？**

---

### Q137. WMS的权限管理是如何设计的？ [Intermediate]

权限模型：RBAC（角色权限控制）+ 数据权限。功能权限：菜单/按钮/接口级别权限。数据权限：仓库维度；货主维度；业务类型维度。角色设计：仓库管理员；拣选员；复核员；财务。特殊权限：超管权限；委托权限；跨仓库权限。权限控制：Spring Security/Shiro；前端路由守卫。

**追问：如何实现操作日志审计？**

---

### Q138. WMS如何实现精准的库内作业时间统计？ [Intermediate]

作业时间采集：RF设备记录操作时间点（领取任务→开始→完成）；PDA扫描节点。作业分析：人均效率（件/人/小时）；任务平均处理时长；库内移动时间占比。瓶颈分析：哪类作业耗时最长；哪个环节效率低。实时监控：大屏实时展示作业进度；异常报警。

**追问：作业效率KPI如何制定才合理？**

---

### Q139. WMS的异常处理机制有哪些？ [Intermediate]

异常分类：系统异常（网络/硬件）；业务异常（库存不足/库位锁定）；操作异常（扫描错误/数量不符）。异常处理：实时报警+人工介入；异常任务挂起/重试；操作日志完整记录。库存差异：即时冻结差异；差异审核；调账审批。常见异常：上架库位被占用；拣选找不到货；复核数量不对。

**追问：如何避免RF操作异常导致的效率问题？**

---

### Q140. WMS的移库任务如何优化以减少仓库运营成本？ [Intermediate]

移库优化策略：收货端合并（同一目的地多批次合并）；发货端合并（同一目的地订单合并）。路径优化：移库任务+拣选路径规划；减少重复路线。时机选择：作业低谷执行；配合盘点执行。自动化：输送线自动搬运；AGV自动搬运。效果评估：移库成本统计；库位利用率提升。

**追问：频繁移库说明初期库位规划有问题，如何优化初期上架策略？**

---

## 综合设计题

### Q141. 假设你设计的WMS系统需要支持日均100万订单量，请描述你的技术架构 [Advanced]

技术架构设计要点：微服务架构（订单服务/仓储服务/库存服务/计费服务）；消息队列削峰（Kafka）；读写分离+分库分表；Redis集群缓存热点数据；多级缓存（本地+Redis）；弹性扩容（K8s HPA）；异步化（日志/通知/报表）。数据库：MySQL分片（按订单ID/仓库ID）；ClickHouse用于报表分析。

**追问：高峰期100万订单集中下单如何处理？**

---

### Q142. 请描述WMS与自动化立库（AS/RS）的集成架构 [Advanced]

集成架构：WMS作为大脑负责任务指令；WCS作为调度层负责设备控制；PLC/RTU负责底层执行。核心交互：WMS下发入库任务→WCS分解为堆垛机指令→执行完成后WMS确认；WMS下发出库任务→WCS调度堆垛机→输送线输送→WMS确认。异常处理：设备故障上报→WCS重试→WMS告警人工介入。

**追问：立库爆仓时WMS如何处理？**

---

### Q143. 如何设计一个支持多仓库统一管理的WMS平台？ [Intermediate]

平台架构：总仓（数据中心） + 多个运营仓（独立部署或SaaS）；统一主数据（SKU/货主/供应商）；分布式库存（各仓独立库存）；跨仓调拨流程。数据同步：库存实时同步；订单统一接收分发；报表汇总。运维：多仓库独立发布；灰度策略；统一监控。

**追问：多仓库库存如何防止超卖？**

---

### Q144. 请设计一个WMS的库存一致性保障方案 [Advanced]

一致性保障：数据库事务保证单库ACID；分布式事务保证跨库一致性；消息队列保证最终一致。实现方式：库存服务强一致（同步）；订单与库存一致性（消息队列+幂等）；跨仓调拨（两阶段提交或TCC）。补偿机制：定时对账发现差异；异步补偿修正。

**追问：Redis缓存与数据库库存如何保证一致性？**

---

### Q145. 如何设计一个WMS的弹性架构应对双11等大促？ [Advanced]

弹性架构要点：计算资源弹性（云服务器+K8s自动扩缩）；数据库读写分离+连接池调优；Redis集群提升缓存吞吐；消息队列削峰；限流熔断保护（Spring Cloud Gateway+Sentinel）。库存热点问题：热点SKU分散到多个Redis分片；库存预分片。压测演练：提前进行全链路压测。

**追问：大促期间如何控制成本？**

---

## MySQL深入问题

### Q146. MySQL的MVCC在REPEATABLE READ下如何实现快照读和当前读？ [Advanced]

快照读：普通SELECT语句，使用事务开始时的快照，读取版本链中可见的数据。当前读：SELECT...FOR UPDATE、INSERT、UPDATE、DELETE，读取最新版本数据并加锁。RC级别下每次SELECT重新生成快照，RR级别下事务开始时生成快照。ReadView包含m_ids（活跃事务ID列表）、min_trx_id、max_trx_id、creator_trx_id。

**追问：ReadView的可见性判断规则是什么？**

---

### Q147. Explain Extended和Explain Analyze的区别？ [Advanced]

EXPLAIN显示查询计划，EXPLAIN ANALYZE（MySQL 8.0+）执行查询并返回实际运行信息：actual rows（实际返回行数）、actual time（实际执行时间）、成本估算vs实际。EXPLAIN ANALYZE是诊断工具，可以发现估算与实际的偏差，如索引选择错误、统计信息过期。

**追问：统计信息过期会导致什么问题？**

---

### Q148. MySQL的optimizer hint有哪些？如何使用？ [Intermediate]

Hint用于引导优化器生成更优执行计划。常用Hint：STRAIGHT_JOIN（固定表顺序）；USE INDEX/IGNORE INDEX/FORCE INDEX（索引选择）；LEADING（指定驱动表）；SEMIJOIN/NO_SEMIJOIN（子查询优化）；MERGE/NO_MERGE（派生表合并）。Hint在复杂查询或优化器判断失误时有用。

**追问：什么情况下优化器会选择错误的索引？**

---

### Q149. MySQL的在线DDL如何实现？ [Intermediate]

MySQL 5.6+支持在线DDL，方式：INPLACE（不复制表，修改元数据）；COPY（复制表重建）。在线DDL选项：ALGORITHM=INPLACE/CLONE；LOCK=NONE/SHARED/EXCLUSIVE。典型场景：加字段（在线加字段）；加索引（先建索引再补充数据）；修改字段类型（需重建表）。

**追问：pt-online-schema-change和gh-ost工具的区别？**

---

### Q150. MySQL半同步复制的工作原理？ [Advanced]

半同步复制（Semi-sync Replication）在主库commit时等待至少一个从库确认收到binlog后才返回客户端。流程：主库事务commit→写入binlog→发送给从库→从库ACK→主库返回客户端。相比异步复制，提高可靠性但增加延迟。参数：rpl_semi_sync_master_wait_point（AFTER_COMMIT/AFTER_SYNC）。

**追问：半同步复制能保证不丢数据吗？**

---

## Redis深入问题

### Q151. Redis的GEO底层实现？GeoHash算法原理？ [Advanced]

GEO使用ZSet存储，score为GeoHash编码值（52位整数）。GeoHash将经纬度编码成字符串：地球纬度[-90,90]、经度[-180,180]；二进制交叉切分形成网格；相邻格子编码前缀相同。精度：编码长度决定格子大小（如6位≈~1.2km*0.6km）。Redis使用8个level的sorted set + 52位整数。

**追问：GeoHash编码相邻但物理距离远的边界情况如何处理？**

---

### Q152. Redis的Lua脚本原子性保证？与事务的区别？ [Advanced]

Lua脚本原子执行：Redis执行Lua脚本时不会执行其他命令，整个脚本作为单一操作。事务(MULTI/EXEC)虽然批量执行但中间可能插入其他客户端命令。脚本优点：原子性+可编程逻辑；支持读取+计算+写入；减少网络往返。脚本缓存：SCRIPT LOAD + EVALSHA避免重复传输。

**追问：Lua脚本执行时间过长会阻塞Redis吗？**

---

### Q153. Redis 6.0的多线程IO原理？ [Intermediate]

Redis 6.0多线程IO：主线程accept客户端连接→分配给IO线程处理读/写→主线程执行命令。默认关闭，通过io-threads-do-reads配置。瓶颈在网络IO而非CPU，6.0之前单线程处理网络IO。命令执行仍是单线程，保持Redis的简单性和一致性。

**追问：多线程IO线程数如何配置？**

---

### Q154. Redis的Lazy Free机制？ [Intermediate]

Lazy Free（惰性删除）异步释放内存而非同步阻塞。命令：UNLINK（异步DEL）；FLUSHDB ASYNC；FLUSHALL ASYNC。场景：大key删除；过期key内存释放。原理：标记待删除对象→后台线程释放→主线程继续处理请求。避免DEL阻塞导致Redis卡顿。

**追问：Lazy Free和fork COW的关系？**

---

### Q155. Redis Cluster的故障检测和自动故障转移流程？ [Advanced]

故障检测：节点间定期PING/PONG；超过阈值未响应标记为疑似下线(PFAIL)；集群内传播；多数节点认为PFALL则标记为FAIL。自动故障转移：集群选举领头哨兵→从节点发起选举→获得多数投票→升级为主节点→通知其他节点更新配置→处理原主节点的从节点。

**追问：Redis Cluster的选举机制与Raft有什么异同？**

---

## Kafka深入问题

### Q156. Kafka的Leader Epoch机制？ [Advanced]

Leader Epoch解决数据丢失问题（尤其是Leader切换时的截断问题）。每个Epoch包含start offset，Follower请求Leader最新Epoch，Leader返回该Epoch开始的日志段。Follower收到Leader Epoch后截断多余数据，避免数据丢失。

**追问：没有Leader Epoch时会出现什么问题？**

---

### Q157. Kafka的Exactly Once实现原理？ [Advanced]

Exactly Once = 幂等producer + 事务。幂等producer：每个Producer分配唯一PID，消息带序列号，Broker去重。事务：原子写多个Partition，引入协调者(Transaction Coordinator)管理事务状态（begin/commit/abort）。Consumer通过isolation.level过滤未提交消息。

**追问：Exactly Once能保证跨Partition的原子性吗？**

---

### Q158. Kafka的配额控制(Quota)原理？ [Intermediate]

Kafka配额控制Producer/Consumer的速率。配额类型：字节速率（bytes/sec）；请求速率。配额作用域：User+Client ID（默认）；User；Client ID。配额分配：Broker侧计量，超过配额被阻塞。动态修改：alterConfigs命令。配额保护：避免个别客户端拖垮整个集群。

**追问：配额限速在Broker侧如何实现？**

---

### Q159. Kafka的分区再均衡（Rebalance）协议？ [Advanced]

Rebalance协议流程：Consumer加入/离开触发→Group Coordinator计算新分配方案→Group Leader（被选出的Member）执行分配→Coordinator通知所有Member新方案。分配策略：Range（按Topic分）；RoundRobin（跨Topic轮询）；Sticky（保持原有分配减少变动）。Rebalance期间消费者无法消费。

**追问：如何减少Rebalance对业务的影响？**

---

### Q160. Kafka的高水位(HW)和Leader Epoch如何配合保证数据一致性？ [Advanced]

HW是已 committed 的消息位置，Consumer只能看到HW之前的数据。Leader Epoch用于解决HW截断问题：Follower故障恢复后，发送FetchRequest带Leader Epoch；Leader比较Epoch，如果Follower的Epoch小于Leader，则只截断到Leader Epoch开始的offset。防止故障恢复时数据被错误截断。

**追问：Follower和Leader差距过大如何处理？**

---

## 综合扩展问题

### Q161. 如何设计一个高并发的抢购系统，Redis和数据库如何配合？ [Advanced]

架构：前端限流→CDN缓存静态页→网关限流→库存预扣Redis→异步下单MQ→数据库最终扣减。Redis库存扣减：DECR原子操作+Lua脚本校验；超卖防护：乐观锁+库存校验。Redis与DB一致性：TCC事务或可靠消息最终一致；延迟双删处理并发写。

**追问：Redis库存扣减成功但MQ发送失败如何处理？**

---

### Q162. 如何设计一个每日亿级流量的短链服务？ [Intermediate]

短链生成：哈希或自增ID转62进制；分布式ID生成器保证不冲突。存储：Redis缓存热数据；MySQL持久化；ClickHouse统计。跳转：302重定向或301（缓存友好）。容量估算：亿级UV约100亿次/天跳转；Redis集群QPS需达百万级。

**追问：短链如何实现永久有效？**

---

### Q163. 如何设计一个消息幂等消费的系统？ [Intermediate]

幂等方案：业务表唯一键+INSERT IGNORE；Redis SETNX key+过期时间；防重表+唯一索引。消息处理：先查询防重表→处理业务→标记已处理。消息去重窗口：根据业务容忍度设置去重时间（如24小时）。幂等性保证：消费者处理前先检查；处理中记录处理中状态；失败重试不重复处理。

**追问：Redis去重和数据库去重各有什么优缺点？**

---

### Q164. 如何设计一个延迟消息队列系统？ [Intermediate]

实现方案：Redis ZSet（score为执行时间戳）；RabbitMQ延迟插件（TTL+DLX）；Kafka+定时轮询。Redis ZSet方案：消费者用ZRANGEBYSCORE获取到期消息→处理→删除。分布式问题：多消费者竞争抢单。定时轮询间隔：100-500ms，平衡精度和性能。

**追问：如何保证延迟消息的可靠性（不丢失）？**

---

### Q165. 如何设计一个支持千万级用户的即时通讯系统？ [Intermediate]

架构要点：长连接网关（Netty）；用户路由（用户ID→网关服务器）；消息路由（单聊/群聊/广播）；消息存储（Timeline模型）；离线消息（用户上线后拉取）。单聊消息流：发送方→API→消息存储→推送（WebSocket/IM服务）→接收方。群聊：消息→群成员路由→多端投递。

**追问：消息顺序性如何保证？**

---

### Q166. 如何设计一个分布式搜索建议系统？ [Intermediate]

搜索建议实现：前缀匹配；Trie树（内存占用大）；Elasticsearch completion suggester。热词缓存：Redis Sorted Set存储搜索热词；前缀查询返回TopN。异步更新：用户搜索词→MQ→实时统计→更新热词库。容灾：本地缓存+L1；Redis L2。

**追问：如何防止恶意搜索刷词？**

---

### Q167. 如何设计一个商品比价系统？ [Intermediate]

数据采集：爬虫抓取竞品价格；API对接供应商。数据存储：商品信息（MySQL）；价格时序数据（InfluxDB/TimescaleDB）。价格策略：最低价/平均价/最低价趋势。比价查询：商品ID关联竞品价格；实时返回。监控预警：价格变化超过阈值告警。

**追问：数据一致性如何保证？**

---

### Q168. 如何设计一个积分兑换系统？ [Intermediate]

积分系统：积分获取（消费/活动）；积分消耗（兑换礼品/抵扣）；积分过期（定时任务）。事务保证：积分扣减+订单创建原子性（分布式事务）。防刷：单用户获取积分上限；异常检测。库存保护：礼品库存独立扣减。

**追问：积分过期如何设计才合理？**

---

### Q169. 如何设计一个外卖配送调度系统？ [Intermediate]

调度算法：就近分配（骑手距商家/顾客距离）；时间窗约束（期望送达时间）；骑手负载均衡。实时调度：订单派发→骑手接单/系统自动派；骑手位置实时更新。异常处理：骑手无法完成→重新派单；配送超时预警。系统协同：接单→取餐→配送→完成全链路。

**追问：高峰期骑手资源不足如何优化？**

---

### Q170. 如何设计一个账户扣费系统保证事务一致性？ [Intermediate]

扣费流程：余额校验→冻结金额→扣减余额→记录流水→解冻/扣除冻结。幂等保证：流水号唯一索引防重复。分布式事务：TCC（Try冻结/Confirm扣减/Confirm解冻）或可靠消息最终一致。幂等+事务：事务开启→幂等检查→执行→状态更新。异常处理：超时补偿任务；财务对账。

**追问：余额和流水如何对账？**

---

## Docker/K8s深入问题

### Q171. Kubernetes的Pod驱逐策略？ [Intermediate]

驱逐触发：Node资源不足（soft/ hard eviction）；Node资源不足时先驱逐BestEffort Pod，再驱逐Burstable Pod。软驱逐（soft eviction）：memoryPressure/ DiskPressure持续超过阈值；可配置grace period。硬驱逐（hard eviction）：立即终止Pod。OOMKilled：容器内存超limits。

**追问：如何避免Pod被频繁驱逐？**

---

### Q172. Kubernetes的调度器如何工作？Predicates和Priorities算法？ [Advanced]

调度流程：过滤（Filter/Predicates）→打分（Scoring/Priorities）→选择。常见Predicates：PodFitsResources、PodFitsHostPorts、MatchNodeSelector、NoDiskConflict、CheckNodeMemoryPressure。常见Priorities：LeastRequestedPriority（资源最少优先）；BalancedResourceAllocation（资源均衡）；ImageLocality（镜像本地优先）。

**追问：如何自定义调度算法？**

---

### Q173. K8s的Etcd数据存储原理？一致性保证？ [Advanced]

Etcd使用Raft共识算法保证分布式一致性。数据模型：B+树的键值存储（mvcc-keyIndex、mvcc-keyBucket）。写流程：客户端请求→Leader写入WAL日志→日志复制到Follower→多数确认后提交→应用状态机。读流程：可配置线性读（与Leader确认）或本地读（可能读到过期数据）。

**追问：Etcd为何不适合存储大量数据？**

---

### Q174. K8s的NetworkPolicy网络策略原理？ [Advanced]

NetworkPolicy是Pod级别的防火墙规则。实现依赖CNI网络插件（Calico/Cilium等）。策略组成：ingress（入流量规则）；egress（出流量规则）；podSelector（目标Pod）；namespaceSelector（命名空间）；port（端口）。默认拒绝所有出入流量，配置白名单。

**追问：如何测试NetworkPolicy是否生效？**

---

### Q175. K8s的Custom Resource(CRD)如何扩展API？ [Intermediate]

CRD允许定义新的资源类型。定义：YAML声明apiVersion/group/name/scope/versions/validation。Controller监听CR资源变化：Informer机制；资源增删改触发 Reconcile。代码生成：operator-sdk自动生成DeepCopy/Clients。示例：PrometheusOperator扩展了ServiceMonitor、Prometheus等资源。

**追问：为什么需要Custom Resource而不是直接存Etcd？**

---

### Q176. K8s Pod的Init Container和Container有什么区别？ [Intermediate]

Init Container在Pod启动前运行，完成初始化任务。特点：必须运行完成才能启动业务Container；多个Init Container按顺序执行；可以包含敏感数据和工具镜像。应用：拉取依赖镜像；初始化配置；等待服务依赖。业务Container全部启动后才对外服务。

**追问：Init Container失败会怎样？**

---

### Q177. Kubernetes的Service Account和User Account区别？ [Intermediate]

User Account：人为系统用户，用于管理员/开发访问集群。Service Account：机器/进程使用的身份，用于Pod访问API Server。Service Account关联Token（JWT），Token挂载到Pod的/run/secrets/kubernetes.io/serviceaccount/。每个Namespace有默认Service Account。

**追问：Pod如何通过Service Account访问API Server？**

---

### Q178. K8s ConfigMap的热更新如何实现？滚动更新会自动生效吗？ [Intermediate]

ConfigMap更新：kubectl apply -f cm.yaml。挂载方式不同生效方式不同：环境变量（不自动更新，需重启Pod）；Volume挂载（自动更新，kubeelet每分钟同步）。Volume更新：大约30-60秒延迟；应用需支持配置Reload（监听文件变化/主动reload）。滚动更新不会自动触发，需要重建Pod。

**追问：Spring Boot应用如何监听ConfigMap更新？**

---

### Q179. K8s的PodPreset作用？如何使用？ [Intermediate]

PodPreset自动注入默认配置到Pod：环境变量；Volume挂载；资源限制。配置：定义PodPreset YAML；selector选择目标Pod。限制：Pod已创建则Preset不生效；不支持集群级别Preset；initContainers不支持注入。替代方案：MutatingWebhook（更灵活）。

**追问：PodPreset在哪个阶段注入配置？**

---

### Q180. Docker容器化如何优化镜像大小？ [Intermediate]

优化策略：使用多阶段构建（编译镜像→运行镜像分离）；选择合适基础镜像（alpine/distroless）；减少层数（合并RUN指令）；清理缓存和临时文件；.dockerignore排除无关文件；使用COPY而非ADD。distroless镜像仅包含运行时依赖，无法登录shell。

**追问：多阶段构建的原理是什么？**

---

## MySQL扩展问题

### Q181. MySQL如何处理热点行更新？ [Intermediate]

热点行问题：多个事务竞争同一行导致锁等待。优化方案：读已提交隔离级别减少锁范围；将大事务拆小；应用层排队（如Redis SETNX）；减少更新频率（批量更新）；业务层面分散热点（库位随机选择）。Long Latch问题：热点索引页面锁。

**追问：短行长更新的热点问题如何优化？**

---

### Q182. MySQL的覆盖索引能否加速COUNT(*)查询？ [Basic]

可以。如果索引包含所有需要统计的列（如主键索引），查询只需扫描索引而不需要回表。InnoDB的COUNT(*)会选择最小的索引统计，因为主键索引包含所有数据。复合覆盖索引比单列主键索引更小，扫描更快。

**追问：SELECT COUNT(DISTINCT col)能用覆盖索引优化吗？**

---

### Q183. MySQL的binlog格式有哪些？如何选择？ [Intermediate]

Statement格式（STATEMENT）：记录SQL语句。优点日志量小，缺点函数/触发器结果不确定。Row格式（ROW）：记录行的变更。优点安全，缺点日志量大。Mixed格式（MIXED）：Statement不安全时自动切换Row。推荐使用Row格式，数据恢复和复制更可靠。

**追问：Row格式下误删数据如何恢复？**

---

### Q184. MySQL的performance_schema监控哪些指标？ [Intermediate]

Performance Schema是MySQL性能监控引擎。监控项：语句事件（events_statements_*）；等待事件（events_waits_*）；阶段事件（events_stages_*）；存储引擎事件；连接信息（threads）。常用表：performance_schema.events_statements_summary_by_digest（按SQL摘要统计）；setup_actors（配置监控过滤）。

**追问：如何排查MySQL CPU使用率高的原因？**

---

### Q185. MySQL的Table Cache机制？连接和Table Cache的关系？ [Intermediate]

Table Cache缓存已打开表的对象（TABLE对象），减少重复打开文件开销。通过table_open_cache参数配置。Table Cache与线程关联：一个线程打开的表对其他线程不可见（不同线程需要分别打开）。连接数和Cache大小关系：max_connections * JOIN_BUFFER + table_cache。连接关闭时表放回Cache。

**追问：Table Cache过小会有什么影响？**

---

## Redis扩展问题

### Q186. Redis的HyperLogLog实现原理？空间复杂度？ [Intermediate]

HyperLogLog用于基数估算（UV统计）。原理：哈希函数映射后取前导零数量，利用概率统计。空间复杂度：固定12KB（2^14个桶，每个桶6bit）。标准误差约0.81%。命令：PFADD添加元素；PFCOUNT获取基数估计；PFMERGE合并。适合大数据量场景（如独立访客统计）。

**追问：HyperLogLog为什么不能获取具体元素？**

---

### Q187. Redis的Pub/Sub实现原理？ [Intermediate]

发布订阅：PUBLISH channel message；SUBSCRIBE channel。原理：维护channel→clients订阅关系；PUBLISH时遍历订阅者发送。问题：消息不持久化（离线丢失）；无确认机制；无消息堆积能力。适用场景：实时通知/广播。替代方案：Redis Stream（支持消费组和ACK）。

**追问：Redis Stream相比Pub/Sub有什么优势？**

---

### Q188. Redis Stream的实现原理？ [Intermediate]

Redis Stream是5.0引入的数据结构，类似持久化消息队列。结构：RadixTree存储消息ID→消息内容；Consumer Group支持多消费者分组；ACK机制；支持监控。命令：XADD/XREAD/XREADGROUP/XACK。适用场景：消息队列替代Pub/Sub和BRPOPLPUSH。

**追问：Redis Stream的消费组如何实现消息分配？**

---

### Q189. Redis的GEO命令如何用于配送范围判断？ [Intermediate]

GEO命令：GEOADD添加位置；GEODIST计算距离；GEORADIUS/GEOSEARCH查询范围内成员；GEOPOS获取成员位置。配送范围：店铺维护自己GEO位置；用户下单时计算到各店铺距离；选择配送范围内店铺。边界处理：模糊范围+距离过滤。

**追问：两个地理位置距离计算用的是球面还是平面？**

---

### Q190. Redis Cluster的MOVED重定向和ASK重定向区别？ [Intermediate]

MOVED：槽永久迁移，客户端缓存槽位信息，下次直接请求正确节点。ASK：槽临时迁移（迁移中），客户端仍需去目标节点获取数据，但不更新本地缓存。ASKING：客户端发送ASKING命令表示愿意接受数据。迁移期间两种重定向共存。

**追问：客户端需要监听MOVED消息并更新槽位缓存吗？**

---

## Kafka扩展问题

### Q191. Kafka的日志段文件管理？日志压缩原理？ [Intermediate]

日志段：每个Partition分为多个segment（.log/.index/.timeindex）。写入机制：当前segment写满（默认1GB）后创建新segment。删除策略：基于时间（retention）/大小（retention.bytes）。日志压缩（Log Compaction）：针对消息key，保留每个key最新消息，删除旧消息，常用于状态存储。

**追问：日志压缩会阻塞正常写入吗？**

---

### Q192. Kafka的消息格式？消息批次(Batch)结构？ [Advanced]

消息格式：Record Base Offset(8B)+Length(4B)+CRC(4B)+Magic(1B)+Attributes(1B)+Key+Value。批次(Batch)：多个Record组成Batch，共享RecordBatch Header（First Offset、Length、Partition Leader Epoch、CRC、Attributes、Last Offset Delta、First Timestamp、Max Timestamp、Producer ID、Producer Epoch、Base Sequence、Record Count）。压缩在批次级别。

**追问：为什么Kafka消息支持批量压缩？**

---

### Q193. Kafka的Controller控制器职责？ [Advanced]

Controller是集群中第一个感知其他Broker存活的节点。职责：Broker注册/下线处理；Leader选举；分区分派；ISR变更通知；LeaderAndIsr请求处理。选举：Broker启动时向Zookeeper抢注，谁先创建/controller节点谁获胜。Controller通过Zookeeper Watch机制感知变化。

**追问：Controller挂了集群会怎样？**

---

### Q194. Kafka的副本同步机制？ISR动态调整？ [Intermediate]

Follower通过FetchRequest拉取Leader数据。同步条件：心跳正常；追上Leader最新数据（replica.lag.max.messages配置）；延迟<replica.lag.time.max.ms。ISR：与Leader保持同步的副本集合。ISR动态收缩：Follower落后太多被踢出；ISR扩张：Follower追回。unclean.leader.election.enable=false防止数据丢失。

**追问：Follower长时间故障恢复后如何同步数据？**

---

### Q195. Kafka的连接管理？TCP连接复用原理？ [Intermediate]

Kafka客户端维护到Broker的连接池。复用机制：同一Broker的请求共用同一连接；Selector多路复用。连接创建时机：元数据刷新时；元数据过期+请求失败。连接复用优势：减少TCP握手开销；请求批量发送提升吞吐。客户端参数：connections.max.idle.ms（空闲连接清理）。

**追问：Kafka的认证机制有哪些？**

---

## 系统设计扩展问题

### Q196. 如何设计一个抢优惠券系统？ [Advanced]

系统架构：券预减库存Redis；用户资格校验（一人一券）；MQ异步下单；库存回滚机制。热点问题：券库存分桶；用户ID打散。防刷：限流；验证码；设备指纹。超卖防护：Redis Lua原子扣减；数据库乐观锁兜底。

**追问：如何防止机器人抢券？**

---

### Q197. 如何设计一个投票系统？ [Intermediate]

投票类型：单选/多选/ ranking。防刷：IP限流；设备指纹；验证码。计数存储：Redis Hash（field=用户ID）；DB存储投票记录。实时统计：Redis INCR；定时持久化到DB。分布式场景：Redis原子操作+DB幂等校验。

**追问：投票结果如何防止被篡改？**

---

### Q198. 如何设计一个评论系统？ [Intermediate]

评论存储：树形结构（邻接表/嵌套集合）；MongoDB更适合复杂评论结构。展示：分页展示+懒加载。点赞：Redis Sorted Set。敏感词过滤：DFA算法/Trie树。点赞防刷：用户维度限流。

**追问：如何实现盖楼评论（嵌套评论）？**

---

### Q199. 如何设计一个爬虫系统？ [Intermediate]

爬虫架构：调度器（URL管理）→下载器→解析器→存储器。分布式爬虫：URL去重（Redis/BloomFilter）；多Worker并行抓取；增量抓取。效率优化：异步IO（aiohttp）；连接池复用；分布式限流。反爬应对：代理IP池；User-Agent轮换；验证码识别。

**追问：URL去重使用BloomFilter有什么问题？**

---

### Q200. 如何设计一个商品搜索系统？ [Intermediate]

搜索架构：Elasticsearch存储商品索引；分词（IK/ANSJ）；排序（相关性/销量/价格）。搜索优化：热词缓存；自动补全（suggest）；拼音搜索；同义词扩展。排序策略：类目打分+文本相关性+商业权重。实时性：数据变更→MQ→ES更新。

**追问：搜索结果如何保证相关性？**

---

DONE_PART3

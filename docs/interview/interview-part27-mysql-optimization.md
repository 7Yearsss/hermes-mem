# MySQL 深度优化面试题（第27期）

> 覆盖 20 道高频面试题，每题包含题目、核心答案、追问方向、避坑提示

---

## 1. MySQL 架构：连接层 / 服务层 / 存储引擎层

### 题目
描述 MySQL 的三层架构（连接层、服务层、存储引擎层），以及 Server 层和 Engine 层的职责划分。

### 核心答案

MySQL 逻辑架构从上到下分为三层：

| 层次 | 组件 | 核心职责 |
|------|------|----------|
| **连接层** | 连接处理器、线程池、认证模块 | 接收客户端连接、SSL 加密、认证、线程管理 |
| **服务层** | SQL 接口、解析器、优化器、缓存 | SQL 解析、查询优化、缓存管理（8.0 已移除 query cache）、跨引擎功能 |
| **存储引擎层** | InnoDB、MyISAM、Memory 等 | 数据读写、事务支持、索引管理、崩溃恢复 |

**Server 层**（服务层）负责：
- SQL 解析与重写（Parser → Preprocessor）
- 查询优化（Optimizer）生成执行计划
- 内置函数（日期、数学、加密等）
- 跨引擎的表管理（DDL、Federated 等）
- 缓存（Table cache、Metadata lock）

**Engine 层**（存储引擎层）负责：
- 数据的物理存储与读取
- 索引的维护（B+Tree、Hash 等）
- 事务的 ACID 实现（InnoDB）
- 崩溃恢复与redo/undo日志
- 锁粒度实现（行锁、页锁、表锁）

### 追问方向
- 服务层和引擎层之间通过什么接口通信？ → **Handler Socket / InnoDB API**
- 为什么 MySQL 要将优化器放在 Server 层而不是 Engine 层？ → **解耦——优化器需要看到所有引擎的统计信息**
- Connector/J、libmysql、PHP-mysqli 分别属于哪一层？ → **连接层**

### 避坑提示
- 不要混淆"服务层"和"Server 层"——两者同义，指的是中间那层
- 8.0 移除了 query cache，但 table cache 和 buffer pool 仍然在，不要把 query cache 和 InnoDB buffer pool 搞混
- 不同引擎共用同一 Server 层，所以 Stored Procedure、Trigger 等对象跨引擎行为一致

---

## 2. InnoDB vs MyISAM

### 题目
对比 InnoDB 和 MyISAM 的核心区别，包括事务支持、锁粒度、索引结构。

### 核心答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务** | 支持 ACID（BEGIN/COMMIT/ROLLBACK） | 不支持 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **并发能力** | 高（MVCC + 行锁） | 低（表锁阻塞写） |
| **索引结构** | 聚簇索引（Clustered）：数据文件即索引 | 非聚簇（Secondary）：索引叶节点存物理行号 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动通过 redo log 恢复 | 需要手动修复（myisamchk） |
| **COUNT(*)** | 需扫描全表（无内部计数器） | 维护了全局行数计数器 |
| **全文索引** | 5.6+ 支持 | 原生支持 |
| **存储限制** | 64TB | 256TB（MyISAM 支持到 256TB，但单文件最大 256GB） |
| **适用场景** | OLTP、高并发、需事务 | OLAP、只读表、全文检索 |

**聚簇索引 vs 非聚簇索引的本质区别**：
- InnoDB：主键索引的叶子节点直接存储完整行数据（数据和索引合一）
- MyISAM：所有索引叶子节点存储的是物理行号，需二次查找获取数据

### 追问方向
- InnoDB 的主键设计原则是什么？ → **自增主键或 UUID（但 UUID v7 更优），避免随机插入导致页分裂**
- 为什么 InnoDB 建议使用自增主键？ → **顺序插入减少页分裂，提升范围查询性能**
- MyISAM 的表锁什么时候会升级？ → **MyISAM 没有锁升级机制，但会导致大量连接阻塞**

### 避坑提示
- 面试中"COUNT(*) 的性能差异"是高频追问——InnoDB 虽然需要扫描但不会锁表，MyISAM 虽然快但不支持并发写
- 不要说"InnoDB 没有表锁"——InnoDB 在意向锁、MDL（元数据锁）、大字段更新时也会升级为表锁
- 很多面试题说"MyISAM 比 InnoDB 快"——这只在纯只读、全表扫描、简单查询场景成立，OLTP 场景 InnoDB 完胜

---

## 3. 索引失效场景

### 题目
列举导致索引失效的常见场景，并解释原理。

### 核心答案

**四大索引失效场景**：

1. **LIKE 模糊查询以通配符开头**
   ```sql
   -- 失效
   SELECT * FROM t WHERE name LIKE '%wang';
   -- 生效（后缀匹配可利用索引）
   SELECT * FROM t WHERE name LIKE 'wang%';
   ```
   原理：前缀通配符导致无法确定索引前缀，引擎无法使用 B+Tree 有序性做二分查找。

2. **隐式类型转换**
   ```sql
   -- phone 是 varchar(20) 类型，传入整数
   SELECT * FROM t WHERE phone = 13000000000;  -- 失效！
   ```
   原理：MySQL 对字符串列比较时会将右边的数字转为字符串，导致全表扫描。**等号左边列发生了隐式 CAST**。

3. **OR 连接条件**
   ```sql
   -- 只有 age 有索引，name 无索引
   SELECT * FROM t WHERE age = 20 OR name = 'Tom';  -- 失效！
   ```
   原理：OR 要求两侧都有索引才能走 index_merge，否则全表扫描。优化方式：用 UNION ALL 拆分或为 name 建索引。

4. **索引列参与计算或函数**
   ```sql
   -- 失效：索引列被包裹在函数内
   SELECT * FROM t WHERE YEAR(create_time) = 2024;
   SELECT * FROM t WHERE LEFT(name, 4) = 'wang';

   -- 失效：参与算术运算
   SELECT * FROM t WHERE price + 10 > 100;  -- 应改为 price > 90
   ```
   原理：索引存储的是原始值，参与计算后无法匹配索引树。

**其他常见失效场景**：
- 使用 `!=` 或 `<>`（某些情况下可利用索引反转扫描）
- `IS NULL` / `IS NOT NULL`（InnoDB 对 NULL 值有专门处理，有时可利用索引）
- 联合索引不遵循最左前缀原则
- 优化器估算全表扫描成本更低时主动放弃索引（统计信息过期）

### 追问方向
- 如何验证索引是否生效？ → **EXPLAIN 查看 type=ref/range 则生效，type=all 则失效**
- 联合索引 `(a,b,c)`，查询 `WHERE a=1 AND b>2 AND c=3` 如何走索引？ → **a 和 b 生效，c 在索引中无效（b 的范围中断了后续列）**
- 字符串类型 phone 的查询，传入字符串能用到索引吗？ → **能，但需保证不隐式转换**

### 避坑提示
- OR 失效是一个高频陷阱，面试时务必说出"OR 两边都要有索引或改成 UNION"这个优化方案
- 隐式类型转换是 SQL 注入之外的另一个安全考点，很多人忽略
- 优化器自动选择不走索引不代表索引没用——可能是统计信息过期，用 `ANALYZE TABLE` 重建

---

## 4. 覆盖索引与索引下推

### 题目
解释什么是覆盖索引（Covering Index）和索引下推（Index Condition Pushdown，ICP），以及 `Using index` 和 `Using index condition` 的区别。

### 核心答案

**覆盖索引**：
查询所需的所有列都包含在索引中，无需回表（不访问主数据页）。

```sql
-- 表：idx(name, age, email)
SELECT name, age FROM t WHERE name = 'Tom';  -- 覆盖索引，Using index
SELECT name, age, phone FROM t WHERE name = 'Tom';  -- 需要回表，Using index condition
```

**索引下推（ICP，MySQL 5.6+）**：
在索引遍历过程中，对索引包含的列先做过滤，减少回表次数。

```sql
-- 表：idx(name, age)
SELECT * FROM t WHERE name LIKE 'T%' AND age = 20;
```
- **无 ICP**：先根据 `name LIKE 'T%'` 找到所有匹配的索引记录，然后逐条回表，再在 server 层过滤 `age = 20`
- **有 ICP**：在 InnoDB 索引树遍历时，直接用 `age = 20` 过滤，只对符合条件的索引项回表

**EXPLAIN 输出区别**：

| Extra 值 | 含义 |
|----------|------|
| `Using index` | 覆盖索引——完全不需要回表 |
| `Using index condition` | 索引下推——先在引擎层过滤，再回表 |
| `Using index condition; Using where` | 引擎层过滤 + Server 层有额外 WHERE 条件 |
| `Using where` | 在 Server 层过滤，无索引优化 |

### 追问方向
- ICP 适用于哪些存储引擎？ → **InnoDB 和 MyISAM 都支持，Memory/NDB 不支持**
- 覆盖索引是否是 ICP 的超集？ → **不是同一维度，覆盖索引描述查询列与索引列的关系，ICP 描述过滤下推的行为**
- 什么情况下 ICP 会失效？ → **索引是辅助索引且查询列包含主键（需回表获取主键再做 ICP 过滤）**

### 避坑提示
- `Using index` 和 `Using index condition` 看起来很像但本质不同——前者完全不回表，后者仍需回表但减少了回表次数
- ICP 只在二级索引场景有收益，聚簇索引本身就是完整数据无需下推
- 联合索引设计时要考虑"查询覆盖"和"ICP 过滤"的列顺序trade-off

---

## 5. 索引合并（Index Merge）

### 题目
什么是索引合并（Index Merge）？有哪几种类型？

### 核心答案

索引合并是 MySQL 优化器在单一查询中同时使用多个索引，并将结果合并的操作。

**三种类型**：

1. **Intersection（交集）**
   ```sql
   SELECT * FROM t WHERE id = 1 AND name = 'Tom';
   -- id 主键索引 + name 二级索引
   -- 先分别取 id=1 的行和 name='Tom' 的行，再取交集
   ```
   条件：等值查询（=），OR 条件下也可能触发。

2. **Union（并集）**
   ```sql
   SELECT * FROM t WHERE id = 1 OR id = 5;
   -- 分别从 id 索引取 id=1 和 id=5 的行，再合并去重
   ```
   条件：OR 连接的多个等值条件。

3. **Sort-Union（排序并集）**
   ```sql
   SELECT * FROM t WHERE id < 1 OR id > 5;
   -- 非等值的范围查询，先分别取范围结果，再排序合并
   -- 比 Union 多了排序步骤
   ```

**EXPLAIN 示例**：
```
type: index_merge
key: idx_id, idx_name
key_len: 4, 20
Extra: Using intersect(idx_id, idx_name)
```

**何时使用**：
- 查询条件包含多个独立索引列
- 优化器认为分别扫描多个索引比全表扫描更优

### 追问方向
- 索引合并和联合索引的取舍？ → **联合索引要求列顺序固定，索引合并更灵活但有额外开销（多次索引扫描 + 合并排序）**
- 索引合并一定比单索引扫描好？ → **不一定——intersection 需要取交集，适合交集比例高的场景，否则随机IO反而更多**
- `OR` 一定触发索引合并吗？ → **不一定，当优化器认为全表扫描成本更低时会放弃索引合并**

### 避坑提示
- 不要把"索引合并"和"多个索引同时使用"混淆——索引合并是结果的合并，不是并列扫描
- 索引合并在某些场景下反而是"索引失效的遮羞布"，说明单列索引设计可能不合理
- 8.0 优化器进一步增强，但核心原则没变——优先考虑联合索引而不是依赖索引合并

---

## 6. 执行计划 EXPLAIN

### 题目
解释 EXPLAIN 输出中 `type`、`ref`、`const`、`eq_ref`、`ref_or_null` 等关键字段的含义。

### 核心答案

**type（连接类型，从最优到最差）**：

| type 值 | 含义 |
|---------|------|
| `system` | 系统表，仅一行（const 的特例） |
| `const` | **主键或唯一索引等值查询，最多返回 1 行** |
| `eq_ref` | **多表关联时，被驱动表用主键或唯一索引等值关联** |
| `ref` | **非唯一索引等值查询，返回匹配的多行** |
| `ref_or_null` | ref 查询但包含 NULL 值扫描（如 `WHERE col = 'x' OR col IS NULL`） |
| `range` | 索引范围扫描（`>`、`<`、`BETWEEN`、IN） |
| `index` | 全索引扫描（遍历整个索引树，不访问数据行） |
| `ALL` | **全表扫描（最差）** |

**ref 字段**：
显示哪个列或常量被用于索引查找。例如 `ref: const, func` 表示第一个用常量，第二个用函数。

**key_len**：
实际使用的索引长度，可用于判断联合索引使用了前几列。

**rows**：
估算 MySQL 需要扫描的行数，不是精确值。

**Extra**：
- `Using filesort`：需额外排序（内存或磁盘）
- `Using temporary`：需临时表
- `Using index`：覆盖索引
- `Using index condition`：索引下推
- `Using where`：Server 层过滤

### 追问方向
- `const` 和 `eq_ref` 的本质区别是什么？ → **const 是单表最多1行，eq_ref 是多表关联时驱动表每行对应被驱动表最多1行（用了主键/唯一索引）**
- `ref_or_null` 为什么要单独一类？ → **因为 OR IS NULL 的语义导致需要额外扫描索引中最左侧的 NULL 值链，成本高于普通 ref**
- `index` 和 `ALL` 的区别？ → **index 扫描整个索引（但索引通常比表小），ALL 扫描整个数据行**

### 避坑提示
- 面试时能快速说出 type 的排序（system > const > eq_ref > ref > ref_or_null > range > index > ALL）是加分项
- 很多人把 `ref` 和 `eq_ref` 混淆——**eq 意味着被驱动表用的是主键或唯一索引，不会返回多行**
- rows 字段是估算值，不准确，8.0 引入 `EXPLAIN ANALYZE` 可看实际执行统计

---

## 7. 慢查询优化

### 题目
如何定位和优化慢查询？请描述从分析到优化的完整流程。

### 核心答案

**慢查询定位工具**：

```sql
-- 1. 查看慢查询日志配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 2. 开启慢查询日志（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 3. 查看慢查询日志文件
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**优化三板斧**：

1. **EXPLAIN 分析**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE user_id = 100;
   -- 关注：type(应<=ref)、key(是否用到索引)、rows(扫描行数)、Extra
   ```

2. **EXPLAIN ANALYZE（MySQL 8.0+）**
   ```sql
   EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 100;
   -- 输出实际执行时间 vs 估算，找出成本瓶颈
   ```

3. **Optimizer Trace 跟踪优化器决策**
   ```sql
   SET OPTIMIZER_TRACE = 'enabled=on';
   SET OPTIMIZER_TRACE_MAX_SIZE = 1000000;

   SELECT * FROM orders WHERE user_id = 100;
   SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
   ```
   查看优化器如何评估每种执行计划的成本。

4. **PROFILING 定位 CPU/IO 耗时**
   ```sql
   SET profiling = 1;
   SELECT * FROM orders WHERE user_id = 100;
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;  -- 各阶段耗时
   ```

**常见优化手段**：
- 添加合适索引（覆盖索引减少回表）
- 拆分或重写复杂 WHERE 条件
- 避免 SELECT *，只查需要的列
- 分解大查询（分页处理）
- 利用 EXPLAIN 调整 JOIN 顺序

### 追问方向
- `EXPLAIN` 和 `EXPLAIN ANALYZE` 的区别？ → **EXPLAIN 只显示估算计划，EXPLAIN ANALYZE 执行查询并返回实际耗时**
- 慢查询日志和 Performance Schema 的关系？ → **慢查询日志是文件记录，Performance Schema 提供更细粒度的指标（锁、IO、内存等）**
- `optimizer_trace` 的限制？ → **JSON 体积大需设 `max_payload`，且开启后对性能有影响仅用于调试**

### 避坑提示
- 不要只盯着 EXPLAIN 的 type=ALL——有时候 type=ref 但 rows 很大（比如100万），同样需要优化
- `Using filesort` 不一定慢，内存排序（`max_length_for_sort_data` 内）很快，只有磁盘排序才是真正瓶颈
- `SHOW PROFILES` 在 8.0 已废弃，改用 Performance Schema 的 `events_statements_history`

---

## 8. SQL 优化

### 题目
比较 `COUNT(*)` vs `COUNT(1)`，如何优化 LIMIT 大偏移量查询？子查询如何改写为 JOIN？

### 核心答案

**COUNT(*) vs COUNT(1) vs COUNT(列)**：

| 形式 | InnoDB 行为 | 性能差异 |
|------|-------------|----------|
| `COUNT(*)` | **最优**，MySQL 内部优化，不取任何列值，只统计行数 | 最快 |
| `COUNT(1)` | 同样只统计行数，但多了一步参数传递 | 和 COUNT(*) 基本等价 |
| `COUNT(id)` | 需要取出 id 列值判断是否为 NULL（主键永不为 NULL） | 最慢，因为要读主键列 |

**结论**：`COUNT(*)` 是 MySQL 官方推荐写法，InnoDB 做了专门优化，`COUNT(1)` 几乎等价但不如 `COUNT(*)` 保险。

**LIMIT 大偏移量优化**：

```sql
-- 原始（深分页，偏移量大时极慢）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;
-- 问题：MySQL 先扫描前100020行，丢弃前100000行

-- 优化方案1：延迟关联（利用索引覆盖）
SELECT * FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) AS t
ON o.id = t.id;

-- 优化方案2：游标分页（基于上一页最大ID）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;
-- 前提：客户端维护上一页最后一条的 id

-- 优化方案3：InnoDB 记录主键的物理位置
SELECT * FROM orders WHERE pk > #{last_pk} ORDER BY pk LIMIT 20;
```

**子查询改写 JOIN**：

```sql
-- 子查询（可能触发全表扫描）
SELECT * FROM users u WHERE u.id IN (SELECT o.user_id FROM orders o);

-- 改写为 JOIN（MySQL 优化器自动转换，但显式写法更可控）
SELECT DISTINCT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 更复杂的子查询改写示例
-- 原始：SELECT * FROM products WHERE price > (SELECT AVG(price) FROM products)
-- 改写：SELECT p1.* FROM products p1
--       INNER JOIN (SELECT AVG(price) AS avg_price FROM products) p2
--       ON p1.price > p2.avg_price;
```

### 追问方向
- `COUNT(*)` 在 MyISAM 和 InnoDB 中性能一样吗？ → **MyISAM 有全局行数计数器秒回，InnoDB 必须扫描全表或索引**
- 延迟关联的原理是什么？ → **子查询只查主键，利用索引覆盖不回表，再关联原表获取其他字段**
- 子查询改写 JOIN 一定等价吗？ → **不完全等价——子查询的 NULL 处理、重复值语义、相关子查询等场景需注意**

### 避坑提示
- `COUNT(col)` 会过滤该列为 NULL 的行，而 `COUNT(*)` 不过滤——语义不同不能随意替换
- 深分页优化是面试高频陷阱，很多人只记得"延迟关联"，忘了游标分页更简单高效
- 8.0 对子查询优化大幅改进，但显式 JOIN 写法语义更清晰、也更易于优化器理解

---

## 9. 关联查询优化

### 题目
描述嵌套循环连接（NLJ）、哈希连接（Hash Join）和排序合并连接（Sort-Merge Join）的原理，以及如何选择驱动表。

### 核心答案

**三种 JOIN 算法**：

1. **嵌套循环连接（Nested Loop Join，NLJ）**
   ```
   FOR row1 IN table1:          -- 驱动表/outer table
     FOR row2 IN table2:        -- 被驱动表/inner table
       IF join_condition:       -- 逐行匹配
         output row
   ```
   - 驱动表越小越好（减少外循环次数）
   - 被驱动表的 join 列**必须有索引**（否则退化为 O(N×M)）
   - MySQL 5.6+ 支持 **Block Nested Loop（BNL）**：一次读取多行驱动表数据到 join buffer，减少内循环次数

2. **哈希连接（Hash Join，MySQL 8.0+）**
   ```
   FOR row1 IN table1:                   -- 构建阶段
     hash_table[hash(join_key)] = row1

   FOR row2 IN table2:                   -- 探测阶段
     IF hash_table.contains(hash(row2.join_key)):
       output row1 + row2
   ```
   - 适用于等值连接（=）
   - 小表作为构建侧（build），大表作为探测侧（probe）
   - 内存不足时溢出到磁盘（Grace Hash Join）

3. **排序合并连接（Sort-Merge Join）**
   ```
   1. 对 table1 的 join 列排序
   2. 对 table2 的 join 列排序
   3. 双指针扫描合并（类似归并排序的 merge 步骤）
   ```
   - 适用于非等值连接（>、<、BETWEEN）
   - MySQL 极少使用（需要显式 hint `STRAIGHT_JOIN` 或优化器选择）

**驱动表选择原则**：
- **小表驱动大表**（减少循环次数）
- **在被驱动表的 join 列上有索引**
- 用 `EXPLAIN` 查看 join 类型：`type=ref/eq_ref` 说明被驱动表走了索引

```sql
-- 强制使用 NLJ（禁用 BNL/Hash Join）
SELECT /*+ NO_BNL() */ * FROM t1 STRAIGHT_JOIN t2 ON t1.id = t2.t1_id;

-- 强制使用 Hash Join
SELECT /*+ HASH_JOIN(t1) */ * FROM t1 INNER JOIN t2 ON t1.id = t2.t1_id;
```

### 追问方向
- NLJ 和 BNL 的区别？ → **NLJ 一次读一行驱动表，BNL 一次读多行到 join buffer 减少内循环次数**
- MySQL 8.0 Hash Join 能替代所有 NLJ 吗？ → **不能——Hash Join 只支持等值连接，且内存消耗大，大数据量仍需 NLJ + 索引**
- JOIN 顺序会影响性能吗？ → **会，改变驱动表顺序可能让被驱动表的索引被利用或不利用，用 STRAIGHT_JOIN 强制顺序**

### 避坑提示
- MySQL 8.0 的 Hash Join 是重大特性，但很多旧系统还在用 5.7，面试时要根据面试官问的版本回答
- `STRAIGHT_JOIN` 是双刃剑——强制顺序但绕过了优化器，谨慎使用
- 小表驱动大表不是绝对的——如果被驱动表的 join 列无索引，即使大表驱动小表也可能更优（因为可以利用被驱动表的索引）

---

## 10. 事务隔离级别

### 题目
解释 MySQL 的四种事务隔离级别，以及 MVCC 的实现原理。

### 核心答案

**四种隔离级别**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|----------|------|-----------|------|----------|
| **READ UNCOMMITTED (RU)** | 可能 | 可能 | 可能 | 无锁，直接读最新值 |
| **READ COMMITTED (RC)** | 不可能 | 可能 | 可能 | MVCC（每次 SELECT 生成新 ReadView） |
| **REPEATABLE READ (RR)** | 不可能 | 不可能 | 可能* | MVCC（事务开始生成单一 ReadView） |
| **SERIALIZABLE** | 不可能 | 不可能 | 不可能 | 锁，所有读取都加锁 |

> * InnoDB 的 RR 级别通过 Next-Key Lock 解决幻读，标准 SQL RR 仍可能有幻读。

**脏读 vs 不可重复读 vs 幻读**：
- **脏读**：读到其他事务**未提交**的数据
- **不可重复读**：同一事务中两次读取同一行，数据**不一致**（被其他事务更新并提交）
- **幻读**：同一事务中两次查询，结果集**行数不一致**（被其他事务插入或删除）

**MVCC 原理概述**（详细在下节）：
- 每行数据有 `DB_TRX_ID`（最近修改的事务ID）和 `DB_ROLL_PTR`（回滚指针）
- 读取时根据 ReadView 判断哪个版本可见
- 通过 undo log 链构建数据的历史快照

### 追问方向
- MySQL 默认隔离级别是什么？ → **InnoDB 默认 REPEATABLE READ**
- RC 和 RR 的本质区别是什么？ → **ReadView 生成时机——RC 每次 SELECT 重新生成，RR 事务内只生成一次**
- 为什么大多数互联网公司使用 RC 而不是 RR？ → **RR 在大并发下容易导致间隙锁竞争，且 RC 配合业务对幻读的容忍度更实用**

### 避坑提示
- SERIALIZABLE 在 InnoDB 中实际是通过 Next-Key Lock 实现的，不是真正的串行执行
- 隔离级别和锁是独立的两个概念——MVCC 实现快照读，锁实现当前读，两者结合
- 不要把"脏读"和"幻读"混淆——前者是读到未提交，后者是两次读取行数不同

---

## 11. MVCC 深度解析

### 题目
详细解释 MVCC 的核心组件（ReadView、undo 日志链、活跃事务列表）以及 RC 和 RR 的区别。

### 核心答案

**MVCC（Multi-Version Concurrency Control）核心组件**：

1. **隐藏字段**（每行数据自带）
   - `DB_TRX_ID`：最近修改的事务 ID（6字节）
   - `DB_ROLL_PTR`：指向 undo log 链的指针（7字节）
   - `DB_ROW_ID`：隐藏主键（如果表没有主键），自增

2. **undo log 链**
   - 每次修改数据时，InnoDB 将旧值写入 undo log，并用 `DB_ROLL_PTR` 链接
   - 形成版本链：`最新数据 → 历史版本1 → 历史版本2 → ... → 初始版本`
   - 不同事务可以构建出不同历史版本的数据

3. **ReadView（读视图）**
   事务执行快照读时生成的快照，包含：
   - `m_ids`：创建 ReadView 时**活跃事务 ID 列表**（未提交的事务）
   - `min_trx_id`：活跃事务列表中的最小 ID
   - `max_trx_id`：创建 ReadView 时**已分配的最大事务 ID + 1**（不是最大活跃事务ID）
   - `creator_trx_id`：当前事务 ID

**可见性判断规则**：
```
IF (row.trx_id == creator_trx_id)         → 可见（自己的修改）
ELIF (row.trx_id < min_trx_id)            → 可见（已提交）
ELIF (row.trx_id IN m_ids)                → 不可见（活跃事务修改）
ELSE                                        → 可见（已提交事务修改）
```

**RC vs RR 的本质区别**：

| | READ COMMITTED | REPEATABLE READ |
|---|---|---|
| **ReadView 生成时机** | **每次 SELECT 重新生成** | **事务第一次 SELECT 时生成** |
| **同一事务中两次读取** | 可能读到其他事务已提交的新数据 | 始终读到事务开始时的快照 |
| **不可重复读** | 可能（其他事务提交了新修改） | 不可能 |
| **幻读** | 可能 | InnoDB 通过 Next-Key Lock 防止 |

**图示（RC 下两次 SELECT）**：
```
T1 时刻 ReadView1：活跃 [T2]  → 看到 旧版本
T2 时刻：T2 提交
T3 时刻 ReadView2：活跃 []    → 看到 新版本（已提交）
→ 同一事务中两次 SELECT 结果不同！
```

### 追问方向
- 如果一个事务的 ReadView 活跃列表很大（如10000个事务），对性能有影响吗？ → **有——每次快照读都需要遍历 m_ids 判断可见性，8.0 通过 trx_id 排序和二分查找优化**
- update 语句是快照读还是当前读？ → **当前读（读取最新版本并加排他锁）**
- undo log 何时被 purge？ → **当没有事务需要某个历史版本时，purge 线程异步清理**

### 避坑提示
- MVCC 并不完全消除锁——它解决的是**快照读的并发问题**，当前读仍需加锁
- RC 每次 SELECT 都重新生成 ReadView，所以同一事务中两个 SELECT 可能看到不同数据（"不可重复读"的真正含义）
- RR 的"单一 ReadView"保证了同一事务内快照的一致性，但 Next-Key Lock 才是防止幻读的机制（不是 MVCC 本身）

---

## 12. 锁机制

### 题目
描述 MySQL InnoDB 的锁类型：共享锁/排他锁、记录锁/间隙锁/临键锁。

### 核心答案

**按模式分类**：

| 锁类型 | 英文 | 作用 | 兼容情况 |
|--------|------|------|----------|
| **共享锁** | S锁 | 允许其他事务读取行（加 S锁） | 与 S锁兼容，与 X锁互斥 |
| **排他锁** | X锁 | 允许持有事务更新/删除行 | 与任何锁互斥 |

```sql
-- 共享锁：SELECT ... LOCK IN SHARE MODE
SELECT * FROM orders WHERE id = 1 LOCK IN SHARE MODE;

-- 排他锁：SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
```

**按粒度分类**：

| 锁类型 | 作用范围 | 说明 |
|--------|----------|------|
| **记录锁（Record Lock）** | 单行记录 | 锁住索引记录，如 `WHERE id = 5` |
| **间隙锁（Gap Lock）** | 索引记录之间的间隙 | 防止其他事务在间隙插入新记录 |
| **临键锁（Next-Key Lock）** | 记录 + 前一间隙 | Record Lock + Gap Lock，MySQL 默认的锁算法 |
| **意向锁（Intention Lock）** | 表级 | IS（意向共享）/ IX（意向排他），表明"表中某行正在加锁" |

**记录锁 vs 间隙锁**：
```sql
-- 记录锁（等值查询，精确匹配）
SELECT * FROM t WHERE id = 5 FOR UPDATE;  -- 锁住 id=5 这条记录

-- 间隙锁（范围查询，锁住区间）
SELECT * FROM t WHERE id BETWEEN 3 AND 7 FOR UPDATE;
-- 锁住 (3,5)、(5,7) 这样的间隙，以及 id=5, id=7 记录
```

**Next-Key Lock 工作原理**：
- 对于 `WHERE id = 5 FOR UPDATE`，如果 id 是唯一索引/主键，实际退化为 Record Lock
- 如果 id 是普通索引或范围查询，锁住：`(-∞, 5] ∪ (5, +∞)`（临键）
- InnoDB 默认在 RR 隔离级别使用 Next-Key Lock 防止幻读

### 追问方向
- 意向锁的作用是什么？ → **让表锁和行锁共存——事务 A 申请行 X 的 X锁，需要先在表上加 IX，事务 B 看到 IX 知道有行锁，不会去申请全表 X锁**
- X锁和 S锁的兼容性矩阵？ → **S-S兼容，S-X互斥，X-X互斥**
- 为什么 InnoDB 的锁信息在 `information_schema.INNODB_LOCKS` 和 `INNODB_LOCK_WAITS` 中查看？ → **存储引擎层的锁表，通过内存结构暴露给 Server 层**

### 避坑提示
- 很多人以为 InnoDB 只有行锁——实际上**行锁是通过索引实现的**，如果 UPDATE 没有索引列会锁全表
- 间隙锁在 RC 隔离级别下不生效（只锁记录），但 RR 隔离级别下会锁索引范围
- 意向锁是表级锁，但不会阻塞普通 DML（如 SELECT）——只有表级锁（如 `LOCK TABLE`）才会和意向锁冲突

---

## 13. Next-Key Lock（临键锁）

### 题目
解释 Next-Key Lock 如何解决幻读问题，以及锁升级的条件。

### 核心答案

**Next-Key Lock = 记录锁（Record Lock）+ 间隙锁（Gap Lock）**

对于索引 `(10, 20, 30)`：
```
Next-Key Lock 锁住的区间：(-∞, 10] (10, 20] (20, 30] (30, +∞)
```
每个临键是左开右闭区间 `(前一个值, 当前值]`。

**幻读问题的本质**：
同一事务中，两次执行 `SELECT * FROM orders WHERE price > 100`，第二次比第一次多了新插入的行（其他事务插入了 price=150 的记录）。

**Next-Key Lock 解决幻读**：
```sql
-- 事务 A：
SELECT * FROM orders WHERE price > 100 FOR UPDATE;
-- 锁住 (某个值, +∞) 的临键，阻止其他事务插入 price > 100 的新记录

-- 事务 B：
INSERT INTO orders VALUES (..., 150);  -- 被阻塞，等待 A 释放锁
```

**锁退化（Lock Downgrade）**：
Next-Key Lock 在以下情况退化为 Record Lock：
- **唯一索引等值查询**：如 `WHERE id = 5`，只锁 id=5 这一行
- **主键索引等值查询**：同样退化为 Record Lock
- **范围查询的边界**：最后一个临键可能退化为 Gap Lock

**为什么退化？因为唯一索引保证键值唯一，不需要锁定间隙。**

**锁升级（Lock Escalation）**：
MySQL **没有显式的锁升级机制**（与 SQL Server 不同），但以下情况会导致锁范围扩大：
- 锁数量超过 `innodb_lock_wait_timeout` 或内存限制
- 事务持有大量行锁时，新申请的行锁可能阻塞直到超时（变相锁升级）
- 大范围 UPDATE（如 `UPDATE t SET col = x` 无 WHERE）会锁全表

### 追问方向
- RR 隔离级别下是否就不会幻读了？ → **InnoDB 通过 Next-Key Lock 基本解决了幻读，但极端场景（如 `SELECT ... FOR UPDATE` 之后 `INSERT`）仍有理论可能**
- Next-Key Lock 锁住的是索引还是数据行？ → **锁住索引项（index entry），不是数据行本身——所以如果 id 无索引，锁会升级到表级**
- `SELECT ... LOCK IN SHARE MODE` 会加 Next-Key Lock 吗？ → **会的，S锁同样遵循 Next-Key Lock 算法**

### 避坑提示
- 不要把"锁升级"和"锁退化"混淆——退化是好事（范围缩小），升级是负面（范围扩大导致更多阻塞）
- 很多人误以为 InnoDB 有 SQL Server 那样的锁升级（行锁→页锁→表锁）——实际上 InnoDB 没有这个机制，是通过计数器监控锁数量
- 面试时说出"唯一索引的等值查询会退化为 Record Lock"是加分项，说明理解了锁退化的原理

---

## 14. 死锁

### 题目
描述死锁产生的条件、InnoDB 的死锁检测机制，以及减少死锁的策略。

### 核心答案

**死锁的四个必要条件（同时满足）**：
1. **互斥**：资源一次只能被一个事务持有
2. **持有并等待**：事务持有资源 A，同时等待资源 B
3. **不可抢占**：资源不能被强制从持有者手中夺走
4. **循环等待**：事务 T1 等 T2，T2 等 T1，形成循环链

**InnoDB 死锁检测**：
- **等待图（Wait-For Graph）**：InnoDB 维护一个事务等待图，检测环路
- **超时回滚**：默认 `innodb_lock_wait_timeout = 50s`，死锁时一个事务超时回滚，另一个获得锁继续
- **死锁回滚**：检测到死锁时，**回滚 UNDO 量最小的事务**（最轻量）而非等待超时
- 查看死锁日志：`SHOW ENGINE INNODB STATUS`（最近一次死锁的详细信息）

**减少死锁的策略**：

1. **按固定顺序访问资源**（打破循环等待）
   ```sql
   -- 所有事务先更新 id=1 的表，再更新 id=2 的表
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 固定顺序
   UPDATE orders SET status = 'paid' WHERE order_id = 100;
   ```

2. **降低隔离级别**（RC 下间隙锁不生效，减少锁竞争）

3. **避免大事务，减少锁持有时间**
   ```sql
   -- 不好：长事务持有锁
   START TRANSACTION;
   SELECT * FROM t;  -- 加锁
   -- ... 长时间业务处理 ...
   UPDATE t SET col = x;  -- 锁被持有很久

   -- 好：拆小事务，及时提交
   START TRANSACTION;
   UPDATE t SET col = x;
   COMMIT;
   ```

4. **合理使用索引**（减少锁的粒度，避免全表锁）

5. **避免子查询和 DISTINCT**，减少复杂 SQL 带来的意外锁

### 追问方向
- 死锁日志在哪里看？ → **`SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 部分**
- InnoDB 死锁检测是每次加锁都检测吗？ → **不是，增量检测——每次有新的等待边加入时检测是否形成环路**
- 为什么回滚 UNDO 量最小的事务而不是最先开始的事务？ → **最小化回滚成本，更快释放锁让其他事务继续**

### 避坑提示
- 死锁和锁等待不是一回事——死锁是两个事务互相等待，锁等待是事务等另一个事务释放锁（可能不是死锁）
- `innodb_lock_wait_timeout` 设太短会导致误判（正常锁等待被误杀），设太长会影响吞吐，50s 是常见默认值
- 面试时能写出"按固定顺序访问"和"拆小事务"两个实战策略，说明不仅懂理论还懂工程

---

## 15. 分库分表

### 题目
描述分库分表的核心概念、ShardingSphere 的实现方式，以及 sharding-key 选择和跨分页查询的解决方案。

### 核心答案

**分库分表的核心问题**：数据水平拆分后，单表变多表，路由和聚合是核心挑战。

**ShardingSphere**（ShardingSphere-JDBC / ShardingSphere-Proxy）：

| 组件 | 架构 | 特点 |
|------|------|------|
| **ShardingSphere-JDBC** | Java 客户端分片，嵌入应用 | 轻量，性能好，主流选择 |
| **ShardingSphere-Proxy** | 代理层（MySQL 协议），部署独立 | 多语言支持，运维复杂 |

**分片策略**：

```yaml
# ShardingSphere 典型配置
dataSources:
  ds_0:
    url: jdbc:mysql://localhost:3306/ds_0
  ds_1:
    url: jdbc:mysql://localhost:3306/ds_1

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_${0..1}.t_order_${0..15}  # 2库 × 16表 = 32物理表
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_mod
```

**sharding-key 选择原则**：
1. **业务查询最频繁的过滤条件**（避免跨分片查询）
2. **高基数**（值分布广，如 user_id 优于 status）
3. **非自增**（避免写入集中在最后一个分片）
4. **均匀分布**（避免热点数据导致单分片压力过大）

常见分片算法：
- **取模**：`hash(key) % sharding_count`（最简单，但扩缩容需全量迁移）
- **范围**：`key / 10000`（按时间分片，热点在最新分片）
- **一致性哈希**：解决取模的扩容痛点

**跨分页查询问题**：

```sql
-- 需求：分页查询订单，第100页每页20条
-- 问题：数据分布在16个分片上

-- 方案1：两次查询（主流）
-- 第一步：从各分片取前 (100*20) = 2000 条的 id
SELECT id FROM t_order_${i} ORDER BY create_time DESC LIMIT 2000;
-- 第二步：聚合排序后取第100页的20条

-- 方案2：点击后查询（游标分页，避免跨分片）
-- 记录上一页最后一条的 id
SELECT * FROM t_order WHERE id < #{last_id} ORDER BY id DESC LIMIT 20;

-- 方案3：异构索引表（汇总表）
-- 将订单按时间+id 汇总到 ES 或另一库，专门用于分页查询
```

### 追问方向
- 分库分表后 JOIN 怎么处理？ → **先在应用层拆分查询（分片键相同），或用广播表 / 异构存储**
- 扩容（resharding）怎么处理？ → **一致性哈希环、倍扩法（4→8表）、在线双写等方案，各有取舍**
- ShardingSphere-Proxy 和 DRDS 的区别？ → **ShardingSphere-Proxy 是 MySQL 协议代理，DRDS 是阿里巴巴闭源商业产品，功能类似但更完善**

### 避坑提示
- sharding-key 选择错误是分库分表最大的坑——选错后大量跨分片查询导致性能反而下降
- "跨分片聚合"是面试高频追问，要能说清楚"先各分片查询再应用层合并"的分层思想
- 分库分表不是银弹——增加了运维复杂度（数据迁移、跨库事务、分布式 ID 生成），考虑是否真的需要

---

## 16. 主从复制

### 题目
描述 MySQL 主从复制的三种模式（异步、半同步、同步）以及 GTID 复制原理。

### 核心答案

**三种复制模式**：

| 模式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **异步复制** | 主库提交事务后立即返回客户端，**不等从库响应** | 性能最高 | 可能丢数据（主库宕机从库未收到） |
| **半同步复制** | 主库提交后**等待至少一个从库 ACK** 才返回客户端 | 至少1个从库有数据 | 主从切换时可能丢数据（从库未ACK） |
| **同步复制** | 主库提交后等待**所有从库 ACK** | 几乎不丢数据 | 性能差，任一从库延迟都阻塞主库 |

**半同步配置**：
```sql
-- 主库安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

-- 从库安装插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- 开启半同步
SET GLOBAL rpl_semi_sync_master_enabled = 1;  -- 主库
SET GLOBAL rpl_semi_sync_slave_enabled = 1;   -- 从库
```

**GTID 复制（MySQL 5.6+，全局事务ID）**：
- 每个事务在主库提交时生成一个全局唯一 ID：`source_id:transaction_id`
- 例如：`3E11FA47-71CA-11E1-9E33-C80AA9429562:23`
- 从库通过 GTID 定位要执行的事务，无需指定 `log_file` 和 `log_pos`

**GTID 复制优势**：
1. **故障转移简单**：新主库通过 GTID 集合确定同步位置
2. **并行复制**：MySQL 5.7+ 的 `WRITESET` 并行复制基于 GTID
3. **一致性保证**：不会漏复制也不会重复复制

```sql
-- GTID 配置
gtid_mode = ON
enforce_gtid_consistency = ON
```

**复制格式**：
- **Statement Based Replication (SBR)**：复制 SQL 语句（可能导致函数/触发器结果不一致）
- **Row Based Replication (RBR)**：复制行变化（binlog 大但安全）
- **Mixed**：混合模式，默认用 SBR，不确定时切换 RBR

### 追问方向
- 半同步复制中，如果从库 ACK 超时怎么办？ → **退化为异步复制，超时时间由 `rpl_semi_sync_master_timeout` 控制**
- GTID 如何保证不重复复制？ → **从库记录已执行的 GTID 集合 (`gtid_executed`)，主库事务的 GTID 在从库已存在则跳过**
- MySQL 8.0 的 `MTS`（多线程从库复制）原理？ → **按 Schema 或 WRITESET 哈希分配 worker 线程，并行执行 binlog events**

### 避坑提示
- 异步复制不等于不可靠——大多数场景（电商、社交）异步复制足够，但金融场景必须半同步
- GTID 复制不是银弹——从库开启 GTID 后不能向主库写主键冲突的数据（需 `auto_position=1`）
- binlog 格式选择 SBR vs RBR 是另一个高频面试点，RBR 是 8.0 默认值

---

## 17. 读写分离

### 题目
描述读写分离的架构，以及主从延迟问题的应对方案（延迟复制、Read-Your-Writes）。

### 核心答案

**读写分离架构**：

```
应用程序
   ↓
读写分离中间件（如 ShardingSphere-Proxy / Atlas / MySQL Router）
   ↓
  /       \
主库(写)  从库1 从库2 从库N (读)
```

**主从延迟的原因**：
1. 从库单线程 relay-log 同步（5.6），5.7+ 支持多线程（并行复制）
2. 从库执行大事务比主库慢（主库提交后从库才开始应用）
3. 从库服务器配置低或负载高
4. 大表 DDL 同步（pt-osc / gh-ost 工具应运而生）

**Read-Your-Writes（读己所写）一致性**：

用户写完数据后立即读取，应该读到自己的写入，而不是从库的旧数据。

```sql
-- 解决方案1：会话一致性（主库读取）
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
-- 写操作
COMMIT;
-- 立即读取：强制走主库（应用层标记或中间件支持）

-- 解决方案2：GTID + 等待
-- MySQL 5.7+ 可以让从库等待特定 GTID 应用完成
SELECT WAIT_FOR_EXECUTED_GTID_SET('gtid_set', 1);  -- 最多等1秒

-- 解决方案3：半同步复制（保证至少一个从库收到）
```

**延迟复制（Delayed Replication）**：

配置从库比主库"慢"固定时间，用于：
- 误操作恢复（从库停止在 DML 执行前）
- 测试环境验证
- 配合 `CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600`（延迟1小时）

```sql
-- 配置延迟复制（从库执行）
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;  -- 1小时延迟
START REPLICA;
```

### 追问方向
- 读写分离和主从复制是同一个东西吗？ → **主从复制是底层数据同步机制，读写分离是基于主从复制的上层架构设计**
- 延迟复制在容灾恢复中的实际价值？ → **可以恢复到误操作前的任意时间点，比 PITR 更精细**
- 如何监控主从延迟？ → **`SHOW REPLICA STATUS` 的 `Seconds_Behind_Master`（不准确），用 `pt-heartbeat` 或 `MASTER_POS_WAIT`**

### 避坑提示
- `Seconds_Behind_Master` = 0 不代表没有延迟——它反映的是 SQL 线程执行 relay-log 的延迟，不包括网络传输延迟
- 读写分离中间件如果不做"写后读主库"处理，会导致用户刚发完帖子立刻刷新看不到（经典的"已发内容消失"bug）
- 延迟复制是运维神器，但要记住它是主动配置的，不是默认行为

---

## 18. SQL 注入

### 题目
解释 SQL 注入的原理，以及如何防御（PreparedStatement），什么是宽字节注入？

### 核心答案

**SQL 注入原理**：

用户输入被拼接到 SQL 语句中，破坏原本 SQL 的结构，执行攻击者期望的恶意 SQL。

```sql
-- 原始查询（Java）
String sql = "SELECT * FROM users WHERE name = '" + name + "' AND pwd = '" + pwd + "'";

-- 正常输入：name='Tom', pwd='123'
-- SELECT * FROM users WHERE name = 'Tom' AND pwd = '123'

-- 注入输入：name=' OR '1'='1', pwd=' OR '1'='1'
-- SELECT * FROM users WHERE name = '' OR '1'='1' AND pwd = '' OR '1'='1'
-- '1'='1' 恒真，绕过认证！
```

**常见注入类型**：
1. **字符型注入**：`' OR '1'='1`（字符串闭合）
2. **数字型注入**：`id = 1 OR 1=1`
3. **UNION 注入**：获取其他表数据
4. **布尔盲注**：根据页面真假响应推断数据
5. **时间盲注**：`SLEEP()` 函数推断

**PreparedStatement 防御**：

预编译 SQL，参数绑定，恶意输入被当作**字面值**而非 SQL 语法。

```java
// 恶意输入被当作字符串字面值，不会被执行
String sql = "SELECT * FROM users WHERE name = ? AND pwd = ?";
PreparedStatement ps = connection.prepareStatement(sql);
ps.setString(1, name);  // ' OR '1'='1' 作为字符串值
ps.setString(2, pwd);
ResultSet rs = ps.executeQuery();
```

**为什么 PreparedStatement 有效？**
- SQL 预编译阶段已确定语法结构（骨架）
- 参数绑定时，数据库只把输入当作**数据**处理，不会有语法解释

**宽字节注入**：

当应用程序使用 GBK/GB2312 等宽字节编码时，一个中文字符占2字节。

```sql
-- PHP 连接数据库使用 GBK 编码
mysql_query("SET NAMES 'gbk'");

-- 输入：name = %df' OR 1=1 --
-- %df' → df 27，其中 27 是单引号 ASCII
-- df27 在 GBK 中是合法双字节字符（%df%27）
-- 等价于：0xdf27，在 GBK 中可能被解释为 '繮'（0xdf=valid, 0x27='）

-- 如果过滤层用 addslashes() 将 ' 转义为 \'（0x5c27）
-- df5c27 在 GBK 中是 0xdf5c + 0x27，5c 是反斜杠但 df5c 是合法 GBK 字符
-- 最终绕过！
```

**防御宽字节注入**：
1. 使用 UTF-8 编码（MySQL 的 `utf8mb4`）——最根本
2. 使用 `PreparedStatement`（参数绑定，宽字节无意义）
3. 过滤层用 `mysql_real_escape_string`（需要正确字符集）

### 追问方向
- ORM 框架（如 MyBatis `#{}` vs `${}`）如何避免注入？ → **`#{}` 是预编译参数绑定，`${}` 是字符串替换——后者有注入风险**
- SQL 注入的防御层面有哪些？ → **三层：应用层（PreparedStatement）、中间件（WAF）、数据库层（最小权限原则）**
- UNION 注入的原理？ → **通过 UNION 合并恶意查询结果集到原查询，UNION 必须列数相同、类型兼容**

### 避坑提示
- `addslashes()` 和 `mysql_escape_string()` 在处理宽字节时都不安全，面试时能说出"宽字节注入的根因是字符集和转义函数不兼容"是加分项
- MyBatis 中 `${}` 是直接拼接，**永远不要**用 `${}` 处理用户输入
- SQL 注入是 OWASP Top 10 常年第一，不要只说"用 PreparedStatement"——还说清楚是**参数绑定**，强调"把用户输入当数据而不是 SQL 语法"

---

## 19. MySQL 8.0 新特性

### 题目
列举 MySQL 8.0 的重要新特性：CTE（公用表表达式）、窗口函数、JSON_TABLE，以及角色管理。

### 核心答案

**1. CTE（Common Table Expression，公用表表达式）**

```sql
-- 普通 CTE（类似子查询但可读性更好）
WITH order_stats AS (
  SELECT user_id, COUNT(*) as order_count, SUM(amount) as total_amount
  FROM orders
  GROUP BY user_id
)
SELECT u.name, o.order_count, o.total_amount
FROM users u
INNER JOIN order_stats o ON u.id = o.user_id;

-- 递归 CTE（树形/层次结构查询）
WITH RECURSIVE cte AS (
  SELECT id, name, manager_id, 1 AS level FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, c.level + 1
  FROM employees e INNER JOIN cte c ON e.manager_id = c.id
)
SELECT * FROM cte;
```

**2. 窗口函数（Window Functions）**

```sql
-- 不使用 GROUP BY 也能进行分组计算
SELECT
  name,
  department,
  salary,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg,   -- 窗口聚合
  RANK() OVER (ORDER BY salary DESC) AS salary_rank,        -- 排名
  LAG(salary, 1) OVER (PARTITION BY department ORDER BY hire_date) AS prev_salary,  -- 前一行
  SUM(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total  -- 累计
FROM employees;

-- 常用窗口函数：ROW_NUMBER()、RANK()、DENSE_RANK()、LAG()、LEAD()、FIRST_VALUE()、LAST_VALUE()
```

**3. JSON_TABLE（JSON 转表）**

```sql
-- 将 JSON 数组展开为关系表
SELECT jt.*
FROM orders,
JSON_TABLE(
  items,                                    -- JSON 列
  '$[*]'                                    -- JSONPath
  COLUMNS (
  item_name VARCHAR(100) PATH '$.name',
  item_qty INT PATH '$.qty',
  item_price DECIMAL(10,2) PATH '$.price'
  )
) AS jt;

-- 配合 JSON 函数：JSON_ARRAYAGG(), JSON_OBJECTAGG()（8.0.21+）
```

**4. 角色（Role）管理**

```sql
-- 创建角色
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- 授权
GRANT SELECT ON db.* TO 'app_read';
GRANT SELECT, INSERT, UPDATE, DELETE ON db.* TO 'app_write';
GRANT ALL ON db.* TO 'app_admin';

-- 将角色授予用户
GRANT 'app_read' TO 'user1'@'%';
GRANT 'app_write' TO 'user2'@'%';

-- 激活角色（可设置默认角色）
SET DEFAULT ROLE 'app_read' FOR 'user1'@'%';

-- 查看角色权限
SHOW GRANTS FOR 'app_read';
```

**其他重要新特性**：
- **撤销表空间管理**：InnoDB 支持独立撤销表空间（`innodb_undo_tablespaces`）
- **不可见索引（Invisible Index）**：`ALTER TABLE t ALTER INDEX idx INVISIBLE` 用于灰度测试索引影响
- **降序索引**：`CREATE INDEX idx ON t(col DESC)` 支持倒序排序索引
- **JSON 聚合函数**：`JSON_ARRAYAGG()`, `JSON_OBJECTAGG()`
- **窗口函数直接用于 ORDER BY / WHERE 子句**

### 追问方向
- CTE 和派生表（子查询）的区别？ → **CTE 可递归、可复用、语义更清晰；派生表每次都要写完整子查询**
- MySQL 8.0 的角色和传统用户权限的区别？ → **角色是权限的集合，不对应登录用户，应用角色前需 `SET ROLE` 或设默认角色**
- 窗口函数能否替代所有 GROUP BY？ → **不能——聚合函数减少行数，窗口函数不减少行数（每行都有输出）**

### 避坑提示
- 窗口函数是 MySQL 8.0 最重要的功能升级，很多面试官会追问"Lead/Lag函数在报表中的实际用法"
- JSON_TABLE 之前处理 JSON 只能用 JSON_EXTRACT() 等函数，JSON_TABLE 让 JSON 查询变成了标准 SQL 形式
- MySQL 8.0 的角色功能之前只能通过 User+Privilege 实现，面试时能说出"角色是权限集合，支持集中管理"是加分项

---

## 20. 性能指标

### 题目
列举 MySQL 的核心性能指标：QPS、TPS、连接数、缓冲池命中率、锁等待，并说明如何查看和计算。

### 核心答案

**QPS（Queries Per Second）**：
```sql
-- 方法1：基于 Com_show status
SHOW GLOBAL STATUS LIKE 'Questions';
-- 每隔1秒取差值
-- QPS = (Questions2 - Questions1) / interval

-- 方法2：基于 Performance Schema（MySQL 5.6+）
SELECT * FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Questions', 'Queries');

-- 方法3：MySQL Enterprise Monitor / prometheus_exporter
```

**TPS（Transactions Per Second）**：
```sql
-- 需要开启 InnoDB 监控
SHOW GLOBAL STATUS LIKE 'Innodb_rows%';
-- TPS = (Com_commit + Com_rollback) / seconds
-- 或基于：Innodb_data_fsync, Innodb_log_written 等计算
```

**连接数**：
```sql
-- 当前连接数
SHOW STATUS LIKE 'Threads_connected';  -- 已建立连接数
SHOW STATUS LIKE 'Threads_running';    -- 正在执行的连接数（不含 sleeping）

-- 最大连接数配置
SHOW VARIABLES LIKE 'max_connections';  -- 默认 151

-- 连接数使用率
-- 使用率 = Threads_connected / max_connections * 100%
-- 使用率 > 80% 需告警
```

**缓冲池命中率（Buffer Pool Hit Rate）**：
```sql
SHOW ENGINE INNODB STATUS;

-- 关键指标：
-- Buffer pool hit rate（InnoDB 页面缓存命中率）
-- 如果 < 95% 说明缓冲池可能不足

-- 也可以从 Performance Schema 查看：
SELECT * FROM performance_schema.memory_summary_by_event_name
WHERE EVENT_NAME LIKE 'memory/innodb/buf%';

-- 缓冲池大小配置
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

**锁等待（Lock Waits）**：
```sql
-- 当前锁等待数量
SHOW STATUS LIKE 'Innodb_row_lock%';
-- Innodb_row_lock_current_waits: 当前等待锁的行数
-- Innodb_row_lock_time: 累计锁等待时间（毫秒）
-- Innodb_row_lock_time_avg: 平均锁等待时间

-- 正在等待的锁详情
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 正在持有的锁
SELECT * FROM information_schema.INNODB_LOCKS;

-- 事务详情
SELECT * FROM information_schema.INNODB_TRX;
```

**综合监控查询（单条 SQL）**：
```sql
SELECT
  SUBSTRING_INDEX(event_name, '/', -1) AS metric,
  count_star AS total_count,
  sum_timer_wait / 3600000000000000 AS total_seconds,
  avg_timer_wait / 3600000000000000 AS avg_seconds
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_seconds DESC
LIMIT 10;
```

**关键阈值汇总**：

| 指标 | 告警阈值 | 处理建议 |
|------|----------|----------|
| QPS | 持续 > 50000（视硬件） | 水平扩展、分库分表 |
| 连接使用率 | > 80% | 增加 max_connections 或连接池 |
| 缓冲池命中率 | < 95% | 增大 innodb_buffer_pool_size |
| 锁等待时间 | 平均 > 100ms | 优化慢查询、检查死锁 |
| 慢查询数量 | > 100条/分钟 | 分析慢查询日志 |
| 主从延迟 | > 5s | 优化从库性能、并行复制 |

### 追问方向
- QPS 和 TPS 的区别？ → **QPS 统计所有查询，TPS 只统计有事务的提交/回滚——一个 TPS 可能包含多个 QPS**
- `innodb_buffer_pool_size` 设置多少合适？ → **通常是物理内存的 60-80%，注意留内存给 OS；Docker 环境中需精确计算**
- Threads_running 和 Threads_connected 的区别？ → **running 是瞬时正在执行的 SQL 数量，connected 是所有活跃连接（含 sleeping）**

### 避坑提示
- 不要只看 QPS——**平均响应时间**（Average Response Time）和 QPS 同样重要，高 QPS 但快响应 ≠ 没问题
- 缓冲池命中率低不一定只是缓冲池小——也可能是全表扫描太多，热点数据无法常驻
- 监控指标一定要建立**基线**（baseline），异常通常是相对于基线的偏离，而非绝对阈值

---

## 附录：面试速查表

### EXPLAIN type 排序（最优→最差）
```
system > const > eq_ref > ref > ref_or_null > range > index > ALL
```

### 事务隔离级别与锁/MVCC 对应
```
RU       → 无MVCC，无锁读
RC       → MVCC (每次SELECT新ReadView)，Record Lock
RR       → MVCC (单一ReadView) + Next-Key Lock
SERIALIZABLE → Next-Key Lock（所有读都加锁）
```

### 锁类型速记
```
共享锁 S ←→ 共享锁 S  兼容
共享锁 S ←→ 排他锁 X  互斥
排他锁 X ←→ 排他锁 X  互斥

Record Lock   → 锁单行
Gap Lock      → 锁间隙（RR下生效）
Next-Key Lock → Record Lock + Gap Lock（默认锁算法）
```

### 索引失效速查
```
LIKE '%abc'   → 失效（前导通配符）
隐式类型转换  → 失效（字符串列用数字比较）
OR 无索引侧   → 失效（除非 index_merge）
索引列计算    → 失效（函数/算术）
```

### MySQL 8.0 三剑客
```
CTE           → WITH ... AS (...) 递归查询
窗口函数      → OVER (PARTITION BY ... ORDER BY ...)
JSON_TABLE    → JSON → 关系表 转换
角色管理      → CREATE ROLE / GRANT role
```

# Elasticsearch 搜索引擎面试题（第20部分）

> 本文档涵盖 20 道 Elasticsearch 高频面试题，每题包含题目、核心答案、追问方向和避坑提示。建议配合实际集群操作和源码阅读加深理解。

---

## 1. ES 核心概念：倒排索引、正排索引、Index/Document/Type/Field

### 题目
请解释 Elasticsearch 的核心概念：倒排索引与正排索引的区别，以及 Index、Document、Type、Field 的含义。

### 核心答案

**倒排索引（Inverted Index）**
- 以词项（Term）为索引键，存储包含该词项的文档列表
- 结构：`Term → (DocId, TermFrequency, Position, Offset)`
- 支持全文检索，是 ES 搜索性能的核心保障

**正排索引（Forward Index）**
- 以文档（DocId）为索引键，存储文档的完整字段信息
- 即 Lucene 的 Stored Fields，用于返回完整文档内容
- 写入时直接写入，读取时根据 DocId 取出原始值

**Index（索引）**
- 逻辑命名空间，等同于 MySQL 的 Database
- 包含多个分片（Shard）和副本（Replica）
- 通过 `index name` 进行 CRUD 操作

**Document（文档）**
- ES 中的最小可搜索单元，JSON 格式
- 等同于 MySQL 的一行记录
- 必须有一个 `_id` 唯一标识

**Type（类型，已废弃）**
- 7.x 之前用于逻辑隔离，同 Index 下可有多个 Type
- 8.x 完全移除此概念，统一为 `_doc`

**Field（字段）**
- Document 的数据列
- 有数据类型（keyword, text, integer, date, geo 等）
- text 类型会经过分词器处理，keyword 不会

### 追问方向
- 倒排索引和正排索引在查询流程中如何配合？
- 为什么 text 字段需要分词而 keyword 不需要？
- Document 的 `_id` 是如何生成的？有几种生成方式？

### 避坑提示
- 不要混淆 Index 和 Type 的概念，Type 在新版本中已不存在
- 倒排索引的查询过程是从词项到文档列表，而不是从文档到词项
- 正排索引不是倒排索引的反向，它们是互补关系，共同构成 Lucene 索引

---

## 2. ES 集群架构：节点类型、分片、副本

### 题目
描述 Elasticsearch 集群的架构组成，包括节点类型、分片（Shard）和副本（Replica）的作用。

### 核心答案

**节点类型**

| 节点类型 | 职责 | 配置参数 |
|---------|------|---------|
| Master 节点 | 集群管理（创建/删除索引、节点调度、分片分配） | `node.master: true` |
| Data 节点 | 数据存储和 CRUD、搜索、聚合 | `node.data: true` |
| Coordinator 节点 | 请求转发、结果聚合（仅协调） | `node.master: false, node.data: false` |
| Ingest 节点 | 文档预处理（pipeline 管道） | `node.ingest: true` |
| ML 节点 | 机器学习任务 | `node.ml: true`（需 x-pack） |

**分片（Shard）**
- Primary Shard：主分片，数据写入先到主分片
- ES 7.x 默认 `number_of_shards: 1`
- 分片数量在索引创建后**不可更改**（除非 reindex）
- 分片大小建议 30GB~50GB

**副本（Replica）**
- Replica Shard：主分片的副本，支持故障转移和读写分离
- 默认 `number_of_replicas: 1`
- 副本数量可以动态调整
- 读请求可打到主分片或副本，实现负载均衡

**集群健康状态**
```
green：主分片 + 副本分片全部正常
yellow：主分片正常，但有副本分片未分配
red：存在未分配的主分片
```

### 追问方向
- 3 个 Data 节点、1 个主分片、1 个副本，集群健康状态是什么？
- 分片数设置过多或过少各有什么问题？
- Coordinator 节点和 LB 负载均衡器的区别是什么？

### 避坑提示
- 分片数不可变是面试高频陷阱，不要说"可以随意修改"
- 不要把所有节点都配置为 Master-eligible，否则容易引发脑裂
- 副本为 0 时集群是 yellow 状态，不是 green，但数据是安全的

---

## 3. ES 文档读写流程：写入路由、读取流程、乐观并发

### 题目
详细描述 Elasticsearch 文档的写入路由过程、读取流程，以及如何处理并发冲突。

### 核心答案

**写入流程（Write Pipeline）**

```
客户端 → Coordinator Node → Routing (hash(_id) % shards) → Primary Shard
  ↓
Primary 写入 Lucene (translog) → 同步到 Replica Shard
  ↓
响应 Coordinator → 返回客户端
```

1. 客户端请求任意节点（Coordinator）
2. 根据 `公式：shard = hash(_id) % number_of_shards` 计算目标分片
3. 请求转发到包含该主分片的 Data 节点
4. 主分片写入 Lucene index 并记录 translog（保证持久性）
5. 并行转发到所有副本分片，等副本确认后返回客户端

**读取流程（Read Pipeline）**

```
客户端 → Coordinator Node → Routing → Primary/Any Replica（随机）
  ↓
从选中的分片读取文档 → 聚合结果 → 返回客户端
```

1. Coordinator 接收搜索请求
2. 广播请求到所有相关分片（Scatter 阶段）
3. 各分片执行查询，返回 DocId + Score
4. Coordinator 收集并排序，取 Top N 的 DocId
5. 去对应分片获取完整文档（Fetch 阶段）
6. 返回结果给客户端

**乐观并发控制（Optimistic Concurrency）**

```json
PUT /index/_doc/1?if_seq_no=5&if_primary_term=1
{
  "field": "value"
}
```

- ES 使用 `seq_no`（序列号）+ `primary_term`（主分片代次）实现乐观锁
- `_version` 是旧版方案，不支持严格顺序
- 写入时带版本号，版本冲突时返回 409 Conflict

### 追问方向
- 写入流程中主分片写入成功但副本写入失败会怎样？
- `wait_for_active_shards` 参数的作用是什么？
- Translog 的刷盘策略是什么？如何平衡性能和数据安全？

### 避坑提示
- 读取不一定要到主分片，副本也能提供读取服务
- 乐观并发不是悲观锁，不会阻塞写入
- 副本写入失败不会导致主分片回滚，但会导致副本数据暂时不一致

---

## 4. ES 分片分配：Hash 路由、故障转移

### 题目
解释 Elasticsearch 分片分配的原理，包括 Hash 路由算法和故障转移机制。

### 核心答案

**Hash 路由算法**

```
shard_num = hash(_routing) % number_of_shards
```

- 默认使用文档 `_id` 作为路由字段
- 可自定义 `_routing` 字段，将相同业务数据写入同一分片
- 分片数决定数据分布的桶数量

**分片分配策略（Shard Allocation）**

1. **分配器（Allocator）**：Master 节点负责执行分配决策
2. **均衡器（Balancer）**：自动在节点间均匀分布分片
3. **权重过滤器（Allocation Filtering）**：
   - `index.routing.allocation.include.*`：可分配到哪些节点
   - `index.routing.allocation.exclude.*`：不能分配到哪些节点
   - `index.routing.allocation.require.*`：必须满足哪些条件

**故障转移（Failover）**

1. Master 检测到节点失联（默认 30s 内无响应）
2. 将失联节点上的主分片在其他节点提升为新主分片（主副本切换）
3. 触发新主分片的副本重分配
4. 集群状态变为 yellow → green

**强制重新分配**

```bash
POST /_cluster/reroute?retry_failed=true
```

### 追问方向
- 自定义路由有什么优缺点？
- 均衡器何时会触发分片迁移？
- 节点重启后分片恢复流程是什么？

### 避坑提示
- 自定义路由会导致数据倾斜，应谨慎使用
- 故障转移期间集群可能短暂不可用
- 不要将 `_routing` 设为经常更新的字段，否则会导致更新失败

---

## 5. Lucene 倒排索引：分词、倒排列表、TF/IDF

### 题目
深入讲解 Lucene 倒排索引的核心结构：分词过程、倒排列表存储结构，以及 TF/IDF 评分原理。

### 核心答案

**分词（Tokenization）**

```
原始文本 → Character Filter → Tokenizer → Token Filter → Tokens
```

- **Character Filter**：处理特殊字符（HTML 标签移除等）
- **Tokenizer**：分词器，将文本切分为词条（例：`the cat sat` → `["the", "cat", "sat"]`）
- **Token Filter**：过滤停用词、同义词变换、词形还原、小写转换等

**倒排列表（Posting List）结构**

```
Term → Sorted DocIds → [DocId, TermFreq, Positions, Offsets, Payloads]
```

每个词项对应一个倒排列表，存储：
- **DocId**：文档 ID
- **Term Frequency (TF)**：词项在文档中出现次数
- **Positions**：词项在文档中的位置（用于 phrase 查询）
- **Offsets**：词项的字符偏移（用于高亮）
- **Payloads**：自定义扩展数据

**倒排列表存储格式**
- FST（Finite State Transducer）：存储词项字典，节省内存
- Lucene 4+ 使用 FST+Postings 格式
- 跳过列表（Skip List）：加速多词查询

**TF/IDF 评分公式（BM25）**

ES 5.x 后默认使用 BM25（Okapi BM25）：

```
score = IDF * (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * dl/avgdl))

其中：
- IDF = log((N - df + 0.5) / (df + 0.5))
- tf = 词项频率
- k1 = 词频饱和度参数（默认 1.2）
- b = 文档长度归一化参数（默认 0.75）
- dl = 文档长度
- avgdl = 平均文档长度
```

### 追问方向
- FST 和 B-Tree 在倒排索引中的区别？
- BM25 相比 TF/IDF 的改进点是什么？
- position 数据在什么场景下必须使用？

### 避坑提示
- 不是所有字段都需要 position，关闭 position 可以节省存储
- TF 不是越高越好，BM25 有词频饱和机制
- 分词结果直接影响搜索质量，需要结合业务选择合适的分词器

---

## 6. 分词器：IK 分词、停用词、同义词

### 题目
介绍 Elasticsearch 分词体系，重点说明 IK 分词器的配置、停用词处理和同义词扩展。

### 核心答案

**分词器架构**

```
Analyzer = Tokenizer + CharFilter(s) + TokenFilter(s)

analyze API:
GET /_analyze
{
  "text": "中华人民共和国",
  "analyzer": "ik_max_word"  // 或 "ik_smart"
}
```

**IK 分词器**

- `ik_max_word`：最细粒度分词（穷举所有可能的词）
  - `"中华人民共和国"` → `["中华人民共和国", "中华人民", "华人", "共和国", "共和国"]`
- `ik_smart`：最粗粒度分词（按语义最大切分）
  - `"中华人民共和国"` → `["中华人民共和国"]`
- 安装：`elasticsearch-plugin install analysis-ik`

**停用词（Stopwords）**

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": ["的", "了", "在", "是"]
        }
      }
    }
  }
}
```

- 停用词可减少索引体积，但可能影响召回率
- 英文停用词列表：`english` 内置
- 禁用停用词过滤：`"stopwords": "_none_"`

**同义词（Synonyms）**

```json
{
  "filter": {
    "my_synonym": {
      "type": "synonym",
      "synonyms": [
        "电脑, 计算机, PC",
        "手机, 移动电话, 智能手机"
      ]
    }
  }
}
```

- **双向同义词**：`"电脑, 计算机"` 表示两者等价
- **单向同义词**：`"iPhone, 苹果手机"`（查询 iPhone 会匹配苹果手机，反之不成立）
- 同步方式：`elasticsearch-plugin install analysis-synonym`

### 追问方向
- IK 分词器的词典更新后如何生效？
- 同义词和分词器的执行顺序是什么？
- 如何实现动态同义词（不重启生效）？

### 避坑提示
- 同义词文件修改后需要重启 ES 或调用 reload synonyms API
- 不要在大规模索引上使用过于细粒度的分词，会导致索引膨胀
- 停用词要根据语言和业务场景选择，中文停用词表与英文不同

---

## 7. ES 搜索类型：query_then_fetch、深分页问题

### 题目
解释 Elasticsearch 的搜索类型（search type），重点说明 `query_then_fetch` 流程和深分页问题。

### 核心答案

**ES 搜索类型**

| 搜索类型 | 描述 |
|---------|------|
| `query_then_fetch`（默认） | 分两阶段执行：先收集 DocId，再获取完整文档 |
| `query_and_fetch` | 每个分片返回本地 Top N，Coordinator 直接汇总（不推荐） |
| `dfs_query_then_fetch` | 预计算 IDF（分布式频率）再执行查询，评分更准确但更慢 |
| `count` | 仅返回匹配文档数，不返回文档 |
| `scan`（已废弃） | 使用 scroll 替代 |

**query_then_fetch 流程（默认）**

```
Phase 1 (Query):
  Coordinator → 所有相关分片 → 各分片执行 query，取 Top N (from+size) DocIds
  ↓
Phase 2 (Fetch):
  Coordinator → 获取的 DocIds 排序 → 取 Top M → 对应分片 fetch 完整文档
```

**深分页问题**

```
# 禁止的深分页：
GET /index/_search?from=10000&size=10
```

**问题根源**：
- 假设 5 个分片，每个分片返回 from+size=10010 条 DocId
- Coordinator 需要在内存中合并、排序所有分片返回的 50050 条 DocId
- from 越大，需要排序的数据量越大，内存和时间复杂度 O(from * shards)

**解决方案**：
1. **Scroll API**（适合导出大量数据）：
   ```bash
   POST /index/_search?scroll=1m
   {
     "size": 1000,
     "sort": ["_doc"]
   }
   ```
2. **Search After**（适合实时翻页）：
   ```bash
   # 第一页
   GET /index/_search?size=10
   # 下一页，使用上次的 sort values
   GET /index/_search?size=10&search_after=[1234567890, "doc_id"]
   ```
3. **Point In Time (PIT)**：搭配 search_after 使用，保证数据一致性视图

### 追问方向
- 为什么 from+size 超过 10000 会报错？如何调整？
- Scroll 和 Search After 的区别是什么？
- `track_total_hits` 参数对深分页的影响？

### 避坑提示
- 永远不要在生产环境使用大 from 值
- Scroll 不适合实时搜索，适合离线数据导出
- Search After 需要一个唯一且排序的字段（如 `_id` 或时间戳）作为 sort 的一部分

---

## 8. ES 聚合：Bucketing / Metrics Aggregation

### 题目
描述 Elasticsearch 聚合查询的类型，重点区分 Bucketing（桶聚合）和 Metrics（指标聚合）。

### 核心答案

**聚合查询结构**

```json
GET /index/_search
{
  "size": 0,
  "aggs": {
    "my_aggregation": {
      "aggregator_type": {
        "field": "field_name",
        "params": ...
      }
    }
  }
}
```

**Bucketing Aggregation（桶聚合）**
- 按条件将文档分组，每组一个桶
- 桶内包含满足条件的文档

| 聚合类型 | 说明 |
|---------|------|
| `terms` | 按字段值分桶（类似 SQL GROUP BY） |
| `histogram` | 按数值区间分桶 |
| `date_histogram` | 按日期时间分桶 |
| `range` | 自定义范围分桶 |
| `filter` / `filters` | 按过滤条件分桶 |
| `nested` | 嵌套文档聚合 |
| `significant_text` | 显著性文本聚合 |

**Metrics Aggregation（指标聚合）**
- 对桶内文档计算标量指标

| 聚合类型 | 说明 |
|---------|------|
| `avg` | 平均值 |
| `sum` | 求和 |
| `min` / `max` | 最小/最大值 |
| `cardinality` | 去重计数（类似 COUNT(DISTINCT)） |
| `stats` | 同时返回 count, min, max, avg, sum |
| `percentiles` | 百分位数 |
| `percentile_ranks` | 百分位排名 |
| `top_hits` | 每桶返回 top N 文档 |

**Pipeline Aggregation（管道聚合）**
- 对其他聚合结果进行二次聚合

```json
{
  "aggs": {
    "sales_per_month": {
      "date_histogram": { "field": "date", "calendar_interval": "month" }
    },
    "max_monthly_sales": {
      "max_bucket": { "buckets_path": "sales_per_month>sales" }
    }
  }
}
```

### 追问方向
- 为什么聚合查询需要设置 `"size": 0`？
- `cardinality` 聚合的精度问题（HyperLogLog 算法）？
- 嵌套聚合（sub-aggregation）的执行顺序？

### 避坑提示
- 不设置 `size: 0` 会返回大量文档，影响性能
- `terms` 聚合返回的是近似值（因分片独立计算后再合并）
- 嵌套层数过深会影响聚合性能

---

## 9. ES 全文搜索：match / term / phrase / multi_match

### 题目
对比 Elasticsearch 的全文搜索查询类型：match、term、match_phrase、multi_match 的区别和使用场景。

### 核心答案

**match 查询（分词搜索）**

```json
{
  "query": {
    "match": {
      "title": "Elasticsearch 教程"
    }
  }
}
```

- 对查询词先分词，再搜索
- `"Elasticsearch 教程"` → 分词为 `["elasticsearch", "教程"]`
- 文档 title 字段中包含任一分词结果即匹配（OR 逻辑）
- **OR → AND**：`"operator": "and"` 可改为 AND 逻辑

**term 查询（精确词项搜索）**

```json
{
  "query": {
    "term": {
      "status": "published"
    }
  }
}
```

- 不分词，直接精确匹配倒排索引中的词项
- 适用于 keyword、numeric、date、boolean 等精确值字段
- **不会**分析查询词，适用于需要精确匹配的场景

**match_phrase 查询（短语搜索）**

```json
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "Elasticsearch 教程",
        "slop": 1
      }
    }
  }
}
```

- 查询词作为整体（不分词），按顺序匹配 phrase
- `slop`：允许词项间插入的最大词数（默认 0）
- `"Elasticsearch 教程"` 要求 title 中有连续的"elasticsearch"和"教程"（中间允许 1 个其他词）

**match_phrase_prefix（前缀短语）**

```json
{
  "query": {
    "match_phrase_prefix": {
      "title": "Elastic"
    }
  }
}
```

- 最后一个词项做前缀匹配（类似自动补全）

**multi_match 查询（多字段搜索）**

```json
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 教程",
      "fields": ["title^2", "content", "description"],
      "type": "best_fields"
    }
  }
}
```

- 在多个字段上执行 match 查询
- `type` 类型：
  - `best_fields`（默认）：取最高分字段的分数
  - `most_fields`：各字段分数相加
  - `cross_fields`：跨字段匹配（词项必须在不同字段中出现）
  - `phrase` / `phrase_prefix`：按短语匹配
- `^2` 表示 title 字段权重加倍

### 追问方向
- `match` 和 `term` 在 keyword 字段上的行为差异？
- `minimum_should_match` 参数的作用？
- `zero_terms_query` 什么时候会触发？

### 避坑提示
- 不要对 text 字段使用 term 查询，查询词不会分词
- 不要对 keyword 字段使用 match 查询，会被分析导致匹配失败
- phrase 搜索依赖 position 信息，关闭 position 的字段无法做 phrase 查询

---

## 10. ES 排序：相关性评分、字段排序

### 题目
解释 Elasticsearch 的排序机制，包括相关性评分（_score）和字段排序的实现方式。

### 核心答案

**相关性评分（_score）**

- 默认按相关性评分降序排序
- ES 使用 BM25 算法计算评分
- 可通过 `"explain": true` 查看评分细节

```json
{
  "query": { "match": { "title": "elasticsearch" } },
  "explain": true
}
```

**BM25 评分解释**

```json
{
  "explanation": {
    "value": 8.5,
    "description": "weight(title:elasticsearch in 0) [PerFieldSimilarityProvider], result of:",
    "details": [...]
  }
}
```

**字段排序**

```json
GET /index/_search
{
  "sort": [
    { "publish_date": "desc" },
    { "_score": "desc" },
    { "_doc": "asc" }
  ]
}
```

- 支持多字段排序，按顺序优先级
- `"_doc"` 排序：按 Lucene 文档顺序（写入顺序），性能最优
- 排序字段类型：数值、日期、keyword、geo_distance

**嵌套字段排序**

```json
{
  "sort": [
    {
      "nested": {
        "path": "offers",
        "filter": { "range": { "offers.price": { "lte": 100 } } }
      },
      "offers.price": "asc"
    }
  ]
}
```

**自定义评分（function_score）**

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "elasticsearch" } },
      "functions": [
        { "field_value_factor": { "field": "popularity", "factor": 1.2 } },
        { "gauss": { "location": { "origin": "0,0", "scale": "200km" } } }
      ]
    }
  }
}
```

### 追问方向
- `script_field` 排序的性能问题？
- 如何实现随机排序（shuffle）？
- 排序时 `_score` 为 null 的情况有哪些？

### 避坑提示
- 排序会禁用 Query Cache，因为结果顺序依赖查询
- 对 text 字段排序需要将字段同时设为 keyword 类型（多字段类型）
- 大量文档排序会消耗较多内存和 CPU

---

## 11. ES 映射：动态映射 vs 显式映射

### 题目
说明 Elasticsearch 的映射（Mapping）机制，对比动态映射和显式映射的优缺点。

### 核心答案

**Mapping 概述**
- 定义索引中文档的字段结构（字段名、数据类型、分词器等）
- 类似于关系数据库的 Schema

**动态映射（Dynamic Mapping）**

```json
PUT /index/_doc/1
{
  "title": "Hello World",
  "count": 123,
  "date": "2024-01-01",
  "flag": true
}
```

ES 自动推断类型：
| JSON 数据 | 推断类型 |
|----------|---------|
| `"string"` | text + keyword |
| `123` | long |
| `123.0` | double |
| `"true"` / `"false"` | boolean |
| `"2024-01-01"` | date |
| `[1, 2, 3]` | long array |
| `{"key": "value"}` | object |

**动态模板（Dynamic Templates）**

```json
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "unmatch_pattern": ".*_text$",
          "mapping": { "type": "keyword" }
        }
      },
      {
        "strings_as_text": {
          "match_mapping_type": "string",
          "match_pattern": ".*_text$",
          "mapping": { "type": "text" }
        }
      }
    ]
  }
}
```

**显式映射（Explicit Mapping）**

```json
PUT /index
{
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "ik_max_word" },
      "price": { "type": "float" },
      "status": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "create_time": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss||epoch_millis" }
    }
  }
}
```

**动态映射参数**

| 参数 | 说明 |
|-----|------|
| `dynamic` | `true`（默认）/ `false` / `runtime`：控制新字段处理 |
| `dynamic_date_formats` | 日期格式列表 |
| `dynamic_templates` | 自定义字段映射规则 |
| `date_detection` | 是否自动检测日期（默认 true） |
| `numeric_detection` | 是否自动检测数字（默认 false） |

### 追问方向
- 动态映射推断错误的类型如何修正？
- 已有字段能否修改类型？
- `runtime` 字段（运行时字段）的作用？

### 避坑提示
- 生产环境强烈建议使用显式映射，避免动态映射导致类型错误
- 已存在字段的类型不能直接修改，只能 reindex
- 动态映射对复杂嵌套对象的支持有限

---

## 12. ES 数据建模：nested 对象、denormalization

### 题目
讨论 Elasticsearch 的数据建模策略，重点说明 nested 对象和反规范化（denormalization）的使用。

### 核心答案

**嵌套对象（nested）**

```json
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested",
        "properties": {
          "name": { "type": "keyword" },
          "age": { "type": "integer" }
        }
      }
    }
  }
}
```

**问题场景**：普通 object 类型的字段，数组内各对象属性会混合存储，无法精确匹配数组内的特定对象。

```json
{
  "user": [
    { "name": "Alice", "age": 30 },
    { "name": "Bob", "age": 25 }
  ]
}
```

普通查询 `{ "term": { "user.name": "Alice" } }` 可能匹配 Bob 的文档（因为 Lucene 将所有属性打平存储）。

**nested 查询**：

```json
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "term": { "user.name": "Alice" } },
            { "term": { "user.age": 30 } }
          }
        }
      }
    }
  }
}
```

**nested 限制**：
- 性能开销较大（每层 nested 都会生成独立索引段）
- 嵌套层数不宜过深

**反规范化（Denormalization）**

ES 是非关系型存储，鼓励通过冗余数据换取查询性能：

```json
{
  "order": {
    "order_id": "1001",
    "customer_name": "张三",
    "customer_region": "北京",  // 反规范化：从 customer 表冗余
    "items": [
      { "product_name": "商品A", "qty": 2 },
      { "product_name": "商品B", "qty": 1 }
    ]
  }
}
```

**数据更新问题**：冗余字段更新时需要同步更新所有相关文档。

**父子关系（join）**

```json
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": "answer"
        }
      }
    }
  }
}
```

- 适合 1:N 或 N:1 的关联查询
- `has_child` / `has_parent` 查询
- 父文档和子文档可在不同分片（通过 parent_id 路由）

### 追问方向
- nested 和 join 类型的性能对比？
- 如何选择 nested vs 冗余数据？
- 数据同步更新的实现方案？

### 避坑提示
- 避免深层嵌套，建议不超过 2 层
- nested 查询的 sort 只能在 nested 内部字段排序
- 反规范化后需要考虑数据一致性问题，通常配合消息队列解决

---

## 13. ES 性能优化：refresh_interval、段合并、bulk 批量

### 题目
介绍 Elasticsearch 的性能优化策略，重点涉及 refresh_interval、段合并机制和 bulk 批量操作。

### 核心答案

**refresh_interval**

```json
PUT /index/_settings
{
  "refresh_interval": "1s"   // 默认 1 秒
}
```

- `refresh`：将内存中的 Lucene segment 写入文件系统缓存，使其可搜索
- 默认每秒刷新一次（适合实时性要求高的场景）
- **调优方向**：
  - 写入高峰期可临时设为 `-1`（禁用自动刷新）
  - 批量导入时设为 `30s` 或 `-1`，导入完成后再调回 `1s`
  - 使用 `?refresh=true` 对单次操作立即生效

**段合并（Segment Merging）**

Lucene 写入流程：
```
写入请求 → Memory Buffer → Refresh（生成新 segment） → segment 持久化
```

- 每个 segment 都是一个完整的 Lucene 索引
- 小 segment 越多，搜索性能越差（需要打开更多文件描述符和 merge 开销）
- ES 后台执行段合并，将小 segment 合并为大 segment

**合并策略**：

```json
{
  "index": {
    "merge_scheduler.max_thread_count": 1,  // 合并线程数
    "refresh_interval": "30s"
  }
}
```

- `TieredMergePolicy`（默认）：按层合并
- 大量写入时可临时降低 merge 频率
- Force Merge（强制合并）用于优化只读场景：

```bash
POST /index/_forcemerge?max_num_segments=1
```

**bulk 批量操作**

```json
POST /_bulk
{ "index": { "_index": "index", "_id": "1" } }
{ "field": "value" }
{ "update": { "_index": "index", "_id": "2" } }
{ "doc": { "field": "new_value" } }
{ "delete": { "_index": "index", "_id": "3" } }
```

**bulk 最佳实践**：
- 单批次大小建议 5~15MB
- 文档数量建议 1000~5000 条/批
- 并发 bulk 建议 2~4 个线程
- 监控 bulk 队列：`GET /_cat/thread_pool?v`

**其他优化点**：

| 优化项 | 说明 |
|-------|------|
| `index.translog.durability: async` | 异步写 translog，提升写入性能（有丢数据风险） |
| `index.number_of_replicas: 0` | 批量导入时关闭副本，导入后开启 |
| 文档 ID 避免使用 UUID | UUID 随机导致写入无法利用 refresh 缓存 |
| 使用 routing | 将相同属性文档路由到同一分片，减少查询分片数 |

### 追问方向
- `index.translog.durability: async` 和 `request` 的区别？
- Force merge 对正在写入的索引有什么影响？
- 如何监控 merge 进度？

### 避坑提示
- refresh_interval 设置过大会导致新写入文档搜索不到
- Force merge 会消耗大量 IO 和 CPU，建议在低峰期执行
- bulk 批量过大会导致 OOM，需要控制单批次大小

---

## 14. ES 安全：用户认证、角色权限

### 题目
描述 Elasticsearch 的安全机制，包括用户认证（Authentication）和角色权限（Authorization）管理。

### 核心答案

**安全架构（X-Pack Security）**

ES 7.x+ 内置 Basic License，提供基础安全功能：
- 用户认证（User Authentication）
- 角色权限（Role-Based Access Control）
- 传输层加密（TLS）
- IP 白名单（允许/禁止列表）

**内置用户**

| 用户 | 角色 |
|-----|------|
| `elastic` | superuser（最高权限） |
| `kibana` | `kibana_user` |
| `logstash_system` | `logstash_system` |
| `beats_system` | `beats_system` |

**创建用户**

```bash
POST /_security/user/jack
{
  "password": "强密码",
  "roles": ["read_role"],
  "full_name": "Jack",
  "email": "jack@example.com"
}
```

**角色（Role）定义**

```bash
POST /_security/role/read_role
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["index-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}
```

**权限类型**：

| Cluster 权限 | Index 权限 |
|-------------|-----------|
| `all` | `all` |
| `monitor` | `read` |
| `manage` | `write` |
| `manage_security` | `delete` |
| `manage_ilm` | `create_index` |
| `manage_rollup` | `index` |
| `manage_ccr` | `view_index_metadata` |

**API 密钥认证**

```bash
POST /_security/api_key
{
  "name": "my-app-key",
  "role_descriptors": {
    "app_reader": {
      "index": [{ "names": ["app-*"], "privileges": ["read"] }]
    }
  }
}
```

返回：`{"id": "xxx", "api_key": "yyy"}`

客户端使用：
```bash
curl -H "Authorization: ApiKey <base64(id:api_key)>" http://localhost:9200/index/_search
```

**LDAP/Active Directory 集成**

```yaml
# elasticsearch.yml
xpack.security.authc.realms:
  native:
    type: native
    order: 0
  ldap:
    type: ldap
    order: 1
    url: "ldap://ldap.example.com:389"
    bind_dn: "cn=admin,dc=example,dc=com"
```

### 追问方向
- Basic License 和 Platinum License 的安全功能差异？
- 如何审计用户操作日志？
- 安全模式（security disabled）和开启安全的区别？

### 避坑提示
- 生产环境务必开启安全功能
- 不要使用默认 elastic 密码，应立即修改
- API 密钥不要硬编码在代码中，应使用密钥管理服务

---

## 15. ES 集群故障：脑裂问题、Master 选举

### 题目
分析 Elasticsearch 集群的脑裂（Split-Brain）问题，以及 Master 节点的选举机制。

### 核心答案

**脑裂问题（Split-Brain）**

**定义**：集群被网络分区分割成多个子集，每个子集都认为自己是主集群，各自独立运行，导致数据不一致。

**发生场景**：
- 3 个 Master-eligible 节点
- 网络分区导致节点 A 和 B 能互相通信，但 C 被隔离
- A 和 B 认为对方正常，继续处理写请求
- C 被隔离后尝试成为 Master，形成多个独立集群

**ES 防止脑裂的机制**

1. **最小 Master 节点数（minimum_master_nodes）**：
   ```
   minimum_master_nodes = (master_eligible_nodes / 2) + 1
   ```
   - 3 个 Master-eligible 节点：`minimum_master_nodes = 2`
   - 只有获得 >= 2 票的节点才能成为 Master
   - 网络分区后，任何分区最多只能有一方满足此条件

2. **ZenDiscovery 选举**：
   - 节点发现：通过 Ping 机制检测节点存活
   - Master 选举：先预选（Pre-Vote），再正式投票（Vote）
   - 发现其他 Master 后会加入，除非发现自己的票数更多

3. **ES 7.x 改进**：
   - 移除了 `minimum_master_nodes` 配置，改为内部自动计算
   - 采用 Raft 风格的选举算法

**Master 选举流程**

```
1. 节点启动，加入集群，发送 join 请求
2. 如果没有 Master，当前节点发起选举
3. 节点根据(cluster_name, nodeId)排序，选最小的为 Master
4. 得票数 >= minimum_master_nodes 的节点成为 Master
5. 其他节点收到新 Master 通知后加入
```

**选举优先级**

```
Master 按以下顺序优先：
1. 最低的 clusterStateVersion（处理过最多变更）
2. 相同的 cluster_state_version，优先选择 master node
3. 节点 ID 最小的
```

**故障检测（Fault Detection）**

- Master 定期 Ping Data 节点和 Master-eligible 节点
- 默认 30s 内无响应则判定节点失联
- 触发故障转移：重新分配失联节点的分片

**恢复策略**

```yaml
# elasticsearch.yml
cluster.routing.allocation.enable: none  # 停止分片分配
# 修复后手动开启
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

### 追问方向
- `discovery.seed_hosts` 和 `cluster.initial_master_nodes` 的区别？
- 脑裂后如何手动修复？
- 为什么 3 个 Master-eligible 节点比 2 个更安全？

### 避坑提示
- 不要将所有节点都设为 master-eligible
- `minimum_master_nodes` 设置错误是脑裂的主要原因
- 脑裂恢复后，可能需要手动处理副本不一致问题

---

## 16. ES 与 MySQL 数据同步：Logstash / Canal

### 题目
介绍 Elasticsearch 与 MySQL 等关系型数据库的数据同步方案，重点对比 Logstash 和 Canal 的实现方式。

### 核心答案

**方案对比**

| 方案 | 原理 | 延迟 | 复杂度 | 数据量 |
|-----|------|-----|-------|-------|
| Logstash | JDBC 定期全量/增量查询 | 秒~分钟 | 低 | 中小 |
| Canal | MySQL Binlog 监听 | 毫秒~秒 | 中 | 大 |
| Debezium | Kafka + CDC | 秒~分钟 | 高 | 大 |
| Python 脚本 | 自定义轮询 | 可控 | 中 | 中小 |

**Logstash JDBC 方案**

```conf
input {
  jdbc {
    jdbc_driver_library => "/mysql-connector-java-8.0.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://mysql:3306/shop"
    jdbc_user => "es_sync"
    jdbc_password => "password"
    
    # 追踪字段，记录上次查询位置
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
    last_run_metadata_path => "/root/.logstash_jdbc_last_run"
    
    # SQL 语句
    statement => "
      SELECT * FROM products 
      WHERE updated_at > :sql_last_value
      AND updated_at < NOW()
    "
    
    # 定时调度
    schedule => "*/5 * * * * *"
  }
}

filter {
  # 可添加字段转换
  mutate { remove_field => ["@version", "@timestamp"] }
  date { match => ["updated_at", "UNIX"] target => "sync_time" }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "products"
    document_id => "%{id}"  # 使用 MySQL 主键作为 ES _id
  }
}
```

**Logstash 增量同步原理**
- 使用 `sql_last_value` 记录上次同步的时间戳/ID
- 每次执行只查询大于上次值的记录
- 适合定时同步，延迟取决于调度频率

**Canal 方案（Binlog 监听）**

Canal 是阿里巴巴开源的 MySQL Binlog 增量订阅组件：

```
MySQL → Canal Server → Canal Adapter → Elasticsearch
         (解析 binlog)   (数据转换)      (写入)
```

**Canal Server 配置**

```yaml
# canal.properties
canal.destinations = example
canal.serverMode = kafka  # 或直连模式
```

**instance.properties**（instance 级配置）：
```properties
canal.instance.master.address = mysql:3306
canal.instance.master.journal.name = mysql-bin.000001
canal.instance.master.position = 12345
canal.instance.dbUsername = canal
canal.instance.dbPassword = canal
canal.instance.filter.regex = shop\\.products
```

**Canal Adapter 配置（ES 适配器）**

```yaml
# application.yml
dataSourcesKey:
  defaultDS:
    url: jdbc:mysql://mysql:3306/shop
    username: canal
    password: canal

esKey:
  targetHosts: es:9200
  properties:
    clusterName: my-cluster
    index: products

etlFiles:
  - name: products
    sqlFile: products.yml
```

**products.yml**（ETL 映射配置）：
```yaml
dataSourceKey: defaultDS
esIndex: products
opType: insert
sql: SELECT id, name, price, category, updated_at FROM products

mapping:
  _id: id
  name: name
  price: price
  category: category
  updated_at: updated_at
```

**同步模式对比**

| 特性 | Logstash | Canal |
|-----|---------|-------|
| 延迟 | 秒~分钟 | 毫秒~秒 |
| 全量同步 | 支持 | 支持（需配置） |
| 增量同步 | 基于时间字段轮询 | 基于 Binlog 实时 |
| 删除同步 | 依赖软删除字段 | 支持（binlog 有 delete 事件） |
| 架构复杂度 | 低 | 中 |
| 依赖 | MySQL 需开启 Binlog | MySQL 需开启 Binlog + GTID |

### 追问方向
- Canal 如何保证 exactly-once 语义？
- 如何处理 Binlog 位点漂移问题？
- Logstash 全量同步时 ES 索引压力大如何处理？

### 避坑提示
- MySQL 必须开启 Binlog 且配置 `binlog_format=ROW`
- Canal 只能同步增量数据，全量同步需要额外处理
- 同步字段类型必须一一对应，否则数据写入会失败

---

## 17. ES 在 WMS 中的应用：商品搜索、订单搜索

### 题目
结合仓库管理系统（WMS）场景，说明 Elasticsearch 在商品搜索和订单搜索中的具体应用。

### 核心答案

**WMS 系统概述**

WMS（Warehouse Management System）核心模块：
- 商品管理（SKU、库存）
- 订单管理（入库单、出库单、调拨单）
- 库位管理
- 报表统计

**ES 在 WMS 中的应用架构**

```
MySQL (WMS 业务库) → Canal → Elasticsearch
                              ↓
                       Kibana (可视化看板)
                              ↓
                       业务系统 API (商品搜索、订单查询)
```

---

### 商品搜索

**商品索引设计**

```json
{
  "mappings": {
    "properties": {
      "sku_id": { "type": "keyword" },
      "sku_name": { "type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_smart" },
      "sku_code": { "type": "keyword" },
      "category_l1": { "type": "keyword" },
      "category_l2": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "barcode": { "type": "keyword" },
      "price": { "type": "float" },
      "stock_qty": { "type": "integer" },
      "warehouse_code": { "type": "keyword" },
      "location_code": { "type": "keyword" },
      "specs": {
        "type": "nested",
        "properties": {
          "attr_name": { "type": "keyword" },
          "attr_value": { "type": "keyword" }
        }
      },
      "status": { "type": "keyword" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" }
    }
  }
}
```

**典型查询场景**

1. **模糊搜索（商品名称）**
   ```json
   {
     "query": {
       "multi_match": {
         "query": "蓝牙耳机",
         "fields": ["sku_name", "sku_code", "barcode"],
         "type": "best_fields",
         "fuzziness": "AUTO"
       }
     }
   }
   ```

2. **分类筛选 + 排序**
   ```json
   {
     "query": {
       "bool": {
         "must": [
           { "term": { "category_l1": "数码" } }
         ],
         "filter": [
           { "range": { "price": { "gte": 100, "lte": 500 } } },
           { "term": { "status": "on_sale" } }
         ]
       }
     },
     "sort": [{ "sales_count": "desc" }]
   }
   ```

3. **嵌套属性筛选（如颜色、尺寸）**
   ```json
   {
     "query": {
       "nested": {
         "path": "specs",
         "query": {
           "bool": {
             "must": [
               { "term": { "specs.attr_name": "颜色" } },
               { "term": { "specs.attr_value": "黑色" } }
             ]
           }
         }
       }
     }
   }
   ```

4. **条码扫描（精确查询）**
   ```json
   {
     "query": { "term": { "barcode": "6921234567890" } }
   }
   ```

**商品搜索优化**：
- 使用 `completion suggester` 实现搜索自动补全
- 使用 `function_score` 实现按库存、销量加权排序

---

### 订单搜索

**订单索引设计**

```json
{
  "mappings": {
    "properties": {
      "order_id": { "type": "keyword" },
      "order_type": { "type": "keyword" },
      "warehouse_code": { "type": "keyword" },
      "customer_name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "customer_phone": { "type": "keyword" },
      "receiver_name": { "type": "text" },
      "receiver_phone": { "type": "keyword" },
      "receiver_address": { "type": "text" },
      "status": { "type": "keyword" },
      "total_amount": { "type": "float" },
      "items": {
        "type": "nested",
        "properties": {
          "sku_id": { "type": "keyword" },
          "sku_name": { "type": "text" },
          "qty": { "type": "integer" }
        }
      },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" },
      "expected_delivery_date": { "type": "date" }
    }
  }
}
```

**典型查询场景**

1. **订单号精确查询**（入库员扫码查询）
   ```json
   { "query": { "term": { "order_id": "IN202401150001" } } }
   ```

2. **客户手机号搜索**
   ```json
   {
     "query": {
       "bool": {
         "should": [
           { "term": { "customer_phone": "13800138000" } },
           { "term": { "receiver_phone": "13800138000" } }
         ]
       }
     }
   }
   ```

3. **订单状态聚合看板**
   ```json
   {
     "size": 0,
     "aggs": {
       "status_count": {
         "terms": { "field": "status" }
       },
       "daily_trend": {
         "date_histogram": {
           "field": "created_at",
           "calendar_interval": "day"
         }
       }
     }
   }
   ```

4. **地址关键词搜索**
   ```json
   {
     "query": {
       "match": { "receiver_address": "北京市朝阳区" }
     }
   }
   ```

5. **商品级联查询（查询包含某 SKU 的所有订单）**
   ```json
   {
     "query": {
       "nested": {
         "path": "items",
         "query": { "term": { "items.sku_id": "SKU001" } }
       }
     }
   }
   ```

**性能优化策略**：
- 订单索引按 `warehouse_code` 做 routing，减少搜索分片数
- 使用别名（alias）隔离读写，零 downtime 重建索引
- 冷热分离：近期订单在热节点，历史订单迁移到冷节点

### 追问方向
- WMS 中如何处理商品多仓库库存的聚合查询？
- 订单状态变更后 ES 同步延迟如何监控？
- ES 和 MySQL 数据不一致时如何处理？

### 避坑提示
- 商品和订单索引量级差异大，建议分开独立索引
- 嵌套查询性能开销大，商品规格不建议超过 3 层
- 订单搜索需要控制返回字段，避免返回过多无用数据

---

## 18. ES 7.x 新特性

### 题目
列举 Elasticsearch 7.x 相比 6.x 的重大新特性，以及 7.x 自身的版本演进。

### 核心答案

**ES 7.0 重大变化**

| 特性 | 说明 |
|-----|------|
| **移除 Type** | 索引只能有一个隐式 type `_doc`，mapping 中不再需要 type |
| **7.0 默认分片数改为 1** | `index.number_of_shards` 默认为 1 |
| **性能改进** | 整体性能提升约 30%（减少跨分片协调开销） |
| **Cluster Coord Purge** | 新增 `cluster.coordination.throttling.*` 控制集群协调 |
| **间隔查询（Interval Query）** | 支持按固定时间间隔查询文档 |
| **.keyword 支持 ignore_above** | keyword 字段支持长度限制 |
| **移除 Yellow/Green 状态语义** | 健康状态含义变化 |

**7.1 ~ 7.5 特性**

- **永续性改进**：向量距离、功能评分查询优化
- **7.3**：在 mapping 中支持条件判断，节省索引大小
- **7.4**：新增 Painless 脚本调试 API
- **7.5**：Search Template 改进

**7.6 ~ 7.10 特性**

| 特性 | 版本 | 说明 |
|-----|------|------|
| **async_search** | 7.6 | 异步搜索，适合长时间运行的聚合 |
| **ES|QL** | 7.12+ | 类 SQL 查询语言（实验阶段） |
| ** searchable snapshots** | 7.10 | 搜索快照（无需完整副本数据） |
| **Runtime Fields** | 7.11+ | 运行时字段，不存储即可计算 |

**7.12+ 特性**

- **ES|QL（实验）**：
  ```sql
  POST /_query
  {
    "query": "SELECT * FROM products WHERE price > 100"
  }
  ```
- **Search Template 增强**
- **默认关闭 JDK17 的 GC**（ZGC 改进）

**7.15 ~ 7.17**

- **ES 7.15**：
  - 新的默认分词器：`analysis-icu`（国际化支持）
  - 减少 JVM 堆内存占用约 8%
- **7.16 / 7.17**：
  - Bugfix 为主，性能稳定版

**8.x 兼容性提示**：
- 8.x 完全移除 Type（但 7.x 索引可升级）
- 8.x 默认启用 Security（免费 Basic）
- 8.x 不支持 `_type` 参数

### 追问方向
- 7.x 默认 1 个分片，6.x 是 5 个，这个变化对迁移有什么影响？
- 7.x 的 async_search 和 scroll 的区别？
- Searchable Snapshots 的适用场景？

### 避坑提示
- 从 6.x 升级到 7.x 时，需要先处理多 type 索引
- 7.x 默认分片数为 1 并不意味着所有场景都用 1 个分片
- ES|QL 仍是实验功能，生产环境慎用

---

## 19. SearchGuard 安全插件

### 题目
介绍 SearchGuard 插件的功能、架构，以及与 X-Pack Security 的对比。

### 核心答案

**SearchGuard 概述**

- 开源 ES 安全插件（AGPLv3 许可）
- 提供用户认证、权限管理、TLS 加密、审计日志
- 支持 Elasticsearch 5.x ~ 7.x（也有 8.x 版本）

**SearchGuard 核心功能**

| 功能 | 说明 |
|-----|------|
| **认证（Authentication）** | 支持 Basic Auth、LDAP/AD、Kerberos、JWT、Proxy |
| **授权（Authorization）** | RBAC 角色权限控制 |
| **传输加密（TLS）** | 节点间加密通信（节点证书） |
| **HTTP 层加密** | HTTPS 支持 |
| **审计日志** | 记录所有操作日志，可输出到文件/ES 本身/Syslog |
| **Doc-Level Security** | 基于查询结果的文档级访问控制 |
| **Field-Level Security** | 字段级访问控制（屏蔽敏感字段） |

**Search Guard 架构**

```
Search Guard Plugin (每个节点) ←→ sg_config.yml
                                        ↓
                              sg_internal_users.yml
                              sg_roles.yml
                              sg_roles_mapping.yml
                              sg_action_groups.yml
```

**核心配置文件**

- `sg_config.yml`：主配置（auth_backend、HTTP 过滤、审计等）
- `sg_internal_users.yml`：用户列表（用户名 + hash 密码）
- `sg_roles.yml`：角色定义（集群权限 + 索引权限）
- `sg_roles_mapping.yml`：用户-角色映射
- `sg_action_groups.yml`：权限动作组

**安装**

```bash
bin/elasticsearch-plugin install search-guard-7
# 或使用离线安装
bin/elasticsearch-plugin install file:///path/to/search-guard-7.zip
```

**初始化 Search Guard**

```bash
./plugins/search-guard-7/tools/sgadmin.sh \
  -cd /path/to/sgconfig/ \
  -cacert /path/to/root-ca.pem \
  -cert /path/to/admin.pem \
  -key /path/to/admin-key.pem \
  -icl \
  -nhnv
```

**与 X-Pack Security 对比**

| 特性 | Search Guard | X-Pack Security |
|-----|-------------|----------------|
| 基础功能 | 开源免费 | 需 Basic+ License |
| TLS 加密 | 免费 | Basic+ License |
| LDAP 集成 | 免费 | Gold+ License |
| 审计日志 | 免费 | Platinum License |
| Doc-Level Security | 免费 | Platinum License |
| 商业支持 | 付费 | 官方付费支持 |
| 云服务商兼容 | 好 | 好 |

**Search Guard 租户（Tenants）**

- 支持 Kibana 多租户隔离
- 每个租户有独立索引访问权限

### 追问方向
- Search Guard 和 X-Pack Security 能同时安装吗？
- 如何迁移从 X-Pack Security 到 Search Guard？
- Search Guard 的 TLS 证书如何配置？

### 避坑提示
- Search Guard 和 X-Pack Security 不能同时启用
- sgadmin 执行后需要重启 ES 节点
- Search Guard 7.x 需对应 ES 版本安装，版本不匹配会导致集群不可用

---

## 20. 跨集群搜索 CCS（Cross Cluster Search）

### 题目
详细说明 Elasticsearch 跨集群搜索（Cross Cluster Search）的原理、配置和使用场景。

### 核心答案

**CCS 概述**

- Cross Cluster Search：允许单个查询同时搜索多个远程集群
- ES 5.3+ 支持，7.x 完全可用
- 不需要数据迁移，实现跨集群联合搜索

**架构原理**

```
本地集群 (Local)          远程集群 A (Remote A)    远程集群 B (Remote B)
┌─────────────┐           ┌─────────────┐          ┌─────────────┐
│ Coordinator │ ──CCS──→ │ Data Node   │          │ Data Node   │
│   Node      │           └─────────────┘          └─────────────┘
└─────────────┘
```

- 本地 Coordinator 节点接收请求
- 通过 `cCSRemoteCluster` 连接远程集群
- 远程集群执行查询后返回 DocId 和 Score
- 本地 Coordinator 合并结果并返回

**配置步骤**

**1. 在本地集群配置远程集群连接**

```yaml
# elasticsearch.yml（本地集群）
cluster.remote.cluster_a:
  seeds: ["remote_a_node1:9300", "remote_a_node2:9300"]
  transport.sniff: true
  proxy_address: ""   # 可选：代理地址

cluster.remote.cluster_b:
  seeds: ["remote_b_node1:9300"]
  mode: sniff  # sniff / proxy
```

**模式说明**：
- `sniff`（默认）：自动发现远程集群节点
- `proxy`：使用固定地址代理

**2. 使用 CCS 查询**

```json
# 语法：<remote_cluster>:<index>
GET /cluster_a:products,cluster_b:products/_search
{
  "query": {
    "match": { "sku_name": "蓝牙耳机" }
  }
}
```

**3. 跨集群查询用户认证**

```yaml
# elasticsearch.yml（本地集群）
cluster.remote.cluster_a:
  seeds: ["remote_a_node1:9300"]
  username: es_ccs_user
  secure_password: "password"  # 或使用 keystore
```

**CCS 过滤（Skip unavailability）**

```json
GET /cluster_a:products,cluster_b:products/_search?ignore_unavailable=true
{
  "query": { ... }
}
```

- `ignore_unavailable=true`：忽略不可用的集群或索引
- 默认情况下，远程集群不可用会导致整个查询失败

**跨集群复制（CCR）vs CCS**

| 特性 | CCS | CCR |
|-----|-----|-----|
| 数据复制 | 不复制 | 实时复制 |
| 查询延迟 | 跨集群网络延迟 | 本地读取 |
| 用途 | 联合搜索 | 灾备/热备 |
| 数据一致性 | 实时性差 | 实时性好 |
| 配置复杂度 | 低 | 高 |

**使用场景**

1. **多数据中心搜索**：不同地区的数据中心各自索引，跨数据中心联合搜索
2. **异构集群**：测试环境和生产环境独立，通过 CCS 联合查询
3. **数据分层**：热数据在本地集群，冷数据在快照集群，通过 CCS 查询

**性能优化**

- 减少跨集群请求次数：将多个远程集群查询合并
- 使用 `pre_filter_shard_size` 控制分片过滤
- 远程集群网络质量影响查询延迟

**监控**

```bash
GET /_remote/info
# 返回已配置的远程集群信息

GET /_nodes/stats/transport
# 查看 CCS 请求统计
```

### 追问方向
- CCS 的延迟如何评估？
- 远程集群和本地集群用户权限如何映射？
- CCS 能搜索没有配置连接的其他集群吗？

### 避坑提示
- CCS 不支持数据 JOIN，只能在本地合并结果
- 远程集群网络不稳定会导致 CCS 查询超时
- 不要用 CCS 替代 CCR 做实时数据同步
- 远程集群和本地集群的 ES 大版本应尽量一致

---

## 附录：ES 面试高频关键词速查

| 关键词 | 关联问题 |
|-------|---------|
| 倒排索引 | #1, #5 |
| 分片路由 | #3, #4 |
| BM25 | #5, #10 |
| refresh_interval | #13 |
| 段合并 | #13 |
| bulk | #13 |
| 深分页 | #7 |
| search_after | #7 |
| nested | #12 |
| join | #12 |
| dynamic mapping | #11 |
| IK 分词 | #6 |
| 聚合 | #8 |
| bucket/metric | #8 |
| X-Pack Security | #14 |
| SearchGuard | #19 |
| 脑裂 | #15 |
| Master 选举 | #15 |
| Logstash | #16 |
| Canal | #16 |
| CCS | #20 |
| CCR | #20 |
| 7.x 新特性 | #18 |

---

> 本文档由 Hermes Agent 生成，建议配合官方文档和实际集群操作加深理解。如发现内容错误或过时，请及时反馈更新。

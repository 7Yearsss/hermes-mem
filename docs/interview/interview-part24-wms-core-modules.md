# WMS核心模块深度面试题（第24期）

> 适用岗位：WMS产品经理、WMS实施顾问、WMS开发工程师、仓储运营管理  
> 难度等级：中高级 | 覆盖模块：入库/出库/库位/批次/效期/盘点/调拨/退货/波次/计费

---

## 1. WMS入库流程：预收货→确认收货→质检→上架，状态机流转

### 题目
描述WMS入库全流程的状态机流转，并说明预收货与确认收货的核心区别是什么？

### 核心答案
```
入库单状态机：
┌──────────┐    ASN到达     ┌──────────┐   车到货确认   ┌──────────┐
│ 待预收货  │ ───────────→  │  预收货   │ ───────────→ │ 确认收货  │
└──────────┘               └──────────┘               └──────────┘
                                                              │
                                              质检类型分流    ▼
                            ┌──────────────┬───────────────┴────┐
                            ▼              ▼                    ▼
                      ┌──────────┐   ┌──────────┐         ┌──────────┐
                      │   免检   │   │   抽检   │         │   全检   │
                      └────┬─────┘   └────┬─────┘         └────┬─────┘
                           └──────────┬──┴──────────────────────┘
                                      ▼
                               ┌──────────┐   上架完成   ┌──────────┐
                               │   上架   │ ──────────→ │  已入库   │
                               └──────────┘              └──────────┘
```

**预收货（Pre-Receive）**：供应商 ASN 到货前提前在系统中创建预入库单，分配库位意向，但**不扣减供应商库存**，也**不产生实际库存账**。核心目的是提前预警、占用库位资源。

**确认收货（Confirm Receive）**：实物到达并扫描/录入后，**实际库存账产生**，关联供应商批次信息，启动质检流程。

**关键差异**：
- 预收货：不产生库存凭证，不参与库存计算
- 确认收货：生成库存台账，触发质检WIP

### 追问方向
- 预收货取消/修改时如何处理已占用库位？
- 质检不合格品如何回退到供应商？
- 上架时库位满了如何处理（库位替代策略）？
- PO（采购单）与 ASN 的对应关系是一对一还是一对多？

### 避坑提示
- **不要混淆**：预收货不产生库存，很多新手误以为 ASN 到就扣库存
- **注意时序**：有些WMS设计里"确认收货"与"质检"是并行，有些是串行，要说清楚
- **批次继承**：预收货时没有批次信息，确认收货后才生成批次号，批次生成规则要提前约定

---

## 2. WMS出库流程：订单接收→库存分配→拣货→复核→打包→出库

### 题目
详细描述WMS出库流程各环节的核心逻辑，特别是库存分配与拣货策略的关系。

### 核心答案
```
订单接收 → 库存预占 → 波次分组 → 拣货任务生成 → RF拣货 → 复核打包 → 集货出库
    │           │           │            │           │        │        │
  API/REST   库存锁定    订单聚合      路径规划    扫描确认  称重复核  交接签收
  格式校验   占用可用量   批次/效期    库位排序    差异处理   打包规则  出库确认
```

**库存预占（Alloction）**：
- 订单下行后，系统检查可用库存（含在途、待检）
- 按策略分配：FIFO / 效期优先 / 指定批次
- 预占后从可用库存中扣除，形成**待出库库存**（soft allocation）
- 若库存不足：全单预占失败 or 部分预占（看策略配置）

**拣货任务生成**：
- 按波次合并订单，生成拣货单
- 拣货路径算法：S型/蛇形/距最近优先
- 拣货单位：整托 / 整箱 / 拆零

**复核打包**：
- 复核：扫描比对，确保发货SKU/数量正确
- 打包：按订单或合并打包，称重校验（重量阈值拦截）

### 追问方向
- 库存分配时遇到库存不足如何处理？是阻塞还是部分分配？
- 拣货时发现库存实际为0（账实差异）如何处理？
- 多批次先入先出时，同SKU不同批次如何确定拣货顺序？
- 打包规则如何配置（按品类/重量/体积/订单数）？

### 避坑提示
- **预占≠占用**：预占库存（soft allocation）在拣货完成前仍是可释放的，确认拣货后才转为实占（hard allocation）
- **路径算法不是玄学**：面试中能说出S型/蛇形差异的是加分项
- **复核≠称重**：复核是核验SKU/数量，称重是校验总重是否合理，两者需分开设计

---

## 3. 库位管理：库位编码规则（仓+区+排+列+层）、库位状态管理

### 题目
设计一个完整的库位编码体系，并说明库位状态各状态之间的转换逻辑。

### 核心答案
**库位编码规则（推荐方案）**：
```
格式：WW-WW-PP-CC-LL
示例：A区-01排-B列-3层 → A-01-B-03

分解：
├── WW = 仓库代码（Warehouse）   2位  A~Z, 01~99
├── WW = 库区代码（Zone）        2位  如：RF(冷冻)/RE(冷藏)/DR(干仓)/HA(高架)
├── PP = 排（Passage/Aisle）     2位  01~99
├── CC = 列（Column/Stack）      2位  01~99
├── LL = 层（Level）             2位  01~09
└── 特殊位：可加批次/方向后缀
```

**库位状态流转**：
```
┌────────────┐   上架完成   ┌────────────┐
│   空闲     │ ──────────→ │   占用     │
└────────────┘              └────────────┘
     ▲                           │
     │    出库/调拨完成          │
     │                           ▼
     │                      ┌────────────┐   盘点差异/冻结  ┌────────────┐
     │                      │   占用     │ ───────────────→ │   冻结     │
     │                      └────────────┘                  └────────────┘
     │                                                     │
     │  解冻完成                                           │  清理/报废
     │                                                     ▼
     │                                              ┌────────────┐
     └───────────────────────────────────────────── │   占用     │
                                                     └────────────┘
```

**关键状态定义**：
| 状态 | 含义 | 可否上架 | 可否拣货 |
|------|------|---------|---------|
| 空闲 | 可用 | ✅ | ✅ |
| 占用 | 有货 | ✅ | ✅ |
| 冻结 | 锁定 | ✅ | ❌ |
| 维修 | 设备维护 | ❌ | ❌ |
| 禁用 | 废弃不用 | ❌ | ❌ |

### 追问方向
- 库位容量（容积/承重）如何管理？超容时如何处理？
- 动态库位 vs 固定库位如何选择？
- 库位绑定（固定某个SKU只能放某库位）如何实现？
- 库位与容器（托盘）的关系：一个库位放多个托盘or一个托盘放多个库位？

### 避坑提示
- **编码不要太长**：超过10位扫码效率低，容易出错
- **层数从1开始**：地面层=1，不是0（仓库常见错误）
- **一维码 vs 二维码**：库位码建议用二维码，一个码包含仓库+库区+排+列+层信息
- **禁用≠删除**：库位禁用后历史记录要保留，不能物理删除

---

## 4. 批次管理：批次号生成规则、批次属性（生产日期/效期/供应商）

### 题目
批次管理的核心是什么？批次号有哪些主流生成规则，各自优缺点是什么？

### 核心答案
**批次号生成规则**：

| 规则类型 | 格式示例 | 优点 | 缺点 |
|---------|---------|------|------|
| 日期+流水号 | `20230515-001` | 可读性强，便于追溯 | 位数较长，跨年需重置 |
| 供应商编码+日期+流水 | `SUP-A-230515-001` | 快速识别供应商 | 依赖供应商编码规范 |
| 入库单号继承 | `RK20230515001-01` | 与入库单强关联 | 批次与入库单耦合 |
| 随机码+日期 | `X7K9-20230515` | 防伪强 | 不可读，无法人工核查 |
| 生产批次ID透传 | `LOT20230515A001` | 厂商原批次保留 | 多系统批次编码不一致 |

**批次核心属性**：
```json
{
  "batch_no": "BTH20230515001",
  "sku_id": "SKU-001",
  "supplier_id": "SUP-A",
  "production_date": "2023-05-15",
  "expiry_date": "2025-05-14",
  "manufacture_date": "2023-05-15",
  "qty": 1000,
  "unit": "EA",
  "warehouse_id": "WH-01",
  "inbound_no": "RK20230515001",
  "customs_info": { },
  "temperature_range": "2-8℃"
}
```

### 追问方向
- 批次号在入库时由谁生成：系统自动or供应商提供？
- 同SKU不同批次是否可以合并？合并后批次属性如何处理？
- 批次追溯链：从原料批次追踪到成品批次，如何实现正向反向双向追溯？
- 批次与效期的关系：批次号能推算效期吗？

### 避坑提示
- **批次号唯一性**：批次号在同一仓库内必须唯一，不能简单用日期+流水（跨年/跨供应商会重复）
- **批次属性继承**：生产日期/效期必须从入库时录入，不能靠批次号推算（除非规则强制约束）
- **批次冻结不等于库位冻结**：可以按批次冻结，也可以按库位冻结，要分清楚
- **先进先出依赖批次**：FIFO策略必须依赖准确的批次数据，批次错了整仓数据都错

---

## 5. 效期管理：近效期预警、效期分级（30天/60天/90天）、效期冻结

### 题目
效期管理的核心逻辑是什么？如何设计近效期预警和效期冻结策略？

### 核心答案
**效期分级标准（常见配置）**：
```
临期警戒等级：
├── 冻结区（Stop）    ≤ 30天    禁止出库，系统自动拦截
├── 近效期（Warning） 31-60天  订单分配时降权，优先出老货
├── 观察区（Watch）   61-90天  黄色预警，不影响分配
└── 正常区（Normal）  > 90天   正常参与FIFO分配
```

**预警触发机制**：
```sql
-- 效期预警每日定时任务
SELECT sku_id, batch_no, expiry_date, 
       DATEDIFF(expiry_date, CURDATE()) AS days_to_expire,
       qty,
       CASE 
         WHEN days_to_expire <= 30 THEN 'FROZEN'
         WHEN days_to_expire <= 60 THEN 'WARNING'  
         WHEN days_to_expire <= 90 THEN 'WATCH'
         ELSE 'NORMAL'
       END AS expiry_level
FROM wms_batch_stock
WHERE warehouse_id = ?
  AND qty > 0
  AND expiry_date <= DATE_ADD(CURDATE(), INTERVAL 90 DAY);
```

**效期冻结流程**：
```
效期检查时点：
1. 入库时 → 录入生产日期/效期，计算剩余天数
2. 分配时 → 库存分配前检查效期等级，冻结区不允许分配
3. 拣货时 → 复核效期，冻结区批次再次校验拦截
4. 出库时 → 出库记录关联批次效期，供追溯查询
```

### 追问方向
- 效期预警的推送方式有哪些（系统消息/邮件/短信/WMS看板）？
- 效期冻结后如何处理：只能等过期还是可以通过审批释放？
- 效期与批次的关系：同SKU不同批次效期不同，如何统一管理？
- 食品/药品/化妆品的效期法规要求有何不同？

### 避坑提示
- **效期计算基准**：用到期日还是生产日+保质期推算？两种方式结果一致但需约定清楚
- **冻结≠报废**：冻结是禁止出库，报废是实物处理，两个概念要区分
- **效期预警不要太多**：90天/60天/30天三个阈值足够，太多档位运营人员记不住
- **系统拦截时机**：是在分配时拦截（推荐）还是拣货时拦截（事后补救）？必须说清楚

---

## 6. 库存冻结：预占库存/实占库存/冻结库存，三种状态转换

### 题目
库存冻结的三种类型是什么？它们之间的状态转换逻辑是怎样的？

### 核心答案
**三种库存状态**：
```
库存状态维度：
┌─────────────────────────────────────────────────────┐
│                   总库存 (Total Stock)                 │
│  ┌─────────────────┐      ┌─────────────────────┐   │
│  │   冻结库存       │      │    可用库存          │   │
│  │  (Frozen Stock) │      │  (Available Stock)   │   │
│  │                 │      │  ┌─────┬─────────┐  │   │
│  │  - 质检冻结     │      │  │预占 │ 实占    │  │   │
│  │  - 效期冻结     │      │  │Soft │ Hard    │  │   │
│  │  - 库存冻结     │      │  │     │ Allocated│  │   │
│  │  - 司法冻结     │      │  └─────┴─────────┘  │   │
│  └─────────────────┘      └─────────────────────┘   │
└─────────────────────────────────────────────────────┘

公式：
总库存 = 可用库存 + 冻结库存
可用库存 = 实占库存 + 预占库存 + 自由库存
```

**状态转换流程**：
```
订单下达 → 预占库存（Soft Allocation）
    │           库存从"可用"转入"预占"
    │           预占后可释放（取消订单时）
    │
拣货确认 → 实占库存（Hard Allocation）
    │           从预占转为实占
    │           实占后一般不可逆（除非取消出库）
    │
出库完成 → 扣减实占库存
    │           库存从账上扣减
    │
质检不合格 / 效期到期 → 冻结库存（Frozen）
    │                        冻结库存不参与任何分配
    │                        冻结后可解冻（质检合格/效期续期）
    │                        或报废（实物处理）
```

### 追问方向
- 预占库存有没有时效？超时不释放怎么办？
- 实占库存和预占库存的数据库表设计有何不同？
- 冻结库存时是否需要同时冻结对应批次？
- 多货主场景下，冻结是按整体库存冻结还是按货主分拆冻结？

### 避坑提示
- **预占不是真实库存**：预占库存只是占用配额，不实际减少实物库存
- **状态变更要原子**：库存状态转换必须在一个事务内完成，避免出现中间状态
- **冻结要可逆**：设计时要把冻结和报废分开，冻结后要能解冻，不能做成单向
- **虚占库存**：预占后长期不拣货会形成"虚占"，占用库位但不产生出库，要定期清理

---

## 7. 盘点流程：初盘/复盘/抽盘、盲盘实现、盘点差异处理

### 题目
WMS盘点流程的核心步骤是什么？盲盘如何实现？盘点差异如何处理？

### 核心答案
**盘点类型**：
| 类型 | 定义 | 频率 | 参与人员 |
|------|------|------|---------|
| 全盘 | 对仓库所有库存逐一盘点 | 年度/季度 | 全员 |
| 抽盘 | 随机抽查部分库位/批次 | 每日/每周 | 专员 |
| 循环盘 | 按计划轮流盘点不同区域 | 每日 | 保管员 |
| 动态盘 | 出入库时顺便核对 | 实时 | 作业员 |

**盲盘实现**：
```python
# 盲盘核心逻辑
def blind_count(stocktake_task_id):
    """盘点员只能看到库位，不知道系统账面数量"""
    
    # 1. 盘点员录入
    actual_data = {
        "location_code": "A-01-B-03",  # 知道库位
        "sku_id": None,                 # 不知道SKU（不知道这个位置有什么）
        "qty": 100,                     # 盘点实际数量
        "batch_no": None,               # 不知道批次
    }
    
    # 2. 盘点后系统比对
    system_qty = get_system_stock(location_code)
    difference = actual_data["qty"] - system_qty["qty"]
    
    # 3. 生成差异报告
    if abs(difference) > threshold:
        trigger_recount(difference)  # 触发复盘
```

**盘点差异处理流程**：
```
初盘录入 → 系统比对 → 差异判断
    │              │
    │         差异≤阈值    差异>阈值
    │              │         │
    │              ▼         ▼
    │         差异确认    发起复盘
    │              │         │
    │              │    复盘录入 → 再次比对
    │              │         │
    │              │    仍差异 → 复盘确认
    │              │         │
    │              │         ▼
    │              │    差异审批（主管/经理）
    │              │         │
    └──────────────┴─────────┘
                   │
                   ▼
            调账处理（红字冲销/蓝字补充）
                   │
                   ▼
            生成盘点报告（盈亏原因分析）
```

### 追问方向
- 盘点时仓库是否需要停止作业？不停业盘点如何实现？
- 盲盘和明盘的使用场景有什么区别？
- 盘点差异率有没有行业标准？超过多少需要追责？
- 盘点差异如何做会计处理（盘盈/盘亏）？

### 避坑提示
- **盲盘不是完全盲**：盘点员知道库位就行，SKU可以从系统中隐藏，防止提前修改账面
- **差异处理要审批**：盘点差异超过一定金额/数量必须走审批，不能由盘点员自行调账
- **盘盈盘亏要分开**：盘盈和盘亏的账务处理方向相反，要分别记录和审批
- **抽盘要随机**：抽盘库位选择要有随机算法，不能让作业人员提前知道，否则失去抽盘意义

---

## 8. 移库调拨：库间调拨/库内移位、库存账务分离

### 题目
库间调拨与库内移位的核心区别是什么？调拨过程中如何保证库存账务一致？

### 核心答案
**移库类型对比**：
```
┌─────────────────────────────────────────────────────┐
│                    移库调拨                          │
├──────────────────────────┬──────────────────────────┤
│     库间调拨              │      库内移位             │
│  (Transfer Out/In)       │    (Location Move)        │
├──────────────────────────┼──────────────────────────┤
│ 跨仓库：WH-A → WH-B       │ 同一仓库内：              │
│ 产生调出单+调入单         │  A-01 → A-02              │
│ 财务需要成本记账          │  只产生移库单             │
│ 涉及运输/物流环节         │  不涉及财务成本           │
│ 审批流程复杂              │  可批量作业               │
└──────────────────────────┴──────────────────────────┘
```

**调拨账务分离（库间调拨）**：
```
调拨出库（调出仓库）          调拨入库（调入仓库）
     │                              ▲
     ▼                              │
┌──────────┐                  ┌──────────┐
│ 调拨在途  │ ───运输中───→  │  收货待验  │
└──────────┘                  └──────────┘
     │                              │
     ▼                              ▼
出库冻结 + 账务移出            入库确认 + 账务移入
（不冲减总库存，状态变为在途）  （增加目标仓库库存）
```

**库内移位账务处理**：
```sql
-- 库内移位：一条记录 OLD库位减，一条记录 NEW库位增
BEGIN TRANSACTION;
  -- 减旧库位
  UPDATE wms_inventory 
     SET location_id = 'NEW-LOC', qty = qty - @move_qty
   WHERE location_id = 'OLD-LOC' AND sku_id = @sku AND batch_no = @batch;
  
  -- 增新库位  
  INSERT INTO wms_inventory (location_id, sku_id, batch_no, qty, ...)
  VALUES ('NEW-LOC', @sku, @batch, @move_qty, ...);
COMMIT;
```

### 追问方向
- 调拨在途如何处理？运输损耗/丢失由谁承担？
- 库间调拨的成本如何计价（调出方成本or调入方采购价）？
- 移库时发现实际数量与系统不符如何处理？
- 移库任务如何生成和执行（RF扫描还是批量指令）？

### 避坑提示
- **库间调拨≠即时调账**：库间调拨有在途状态，调出方账已减但调入方未增，不能做成实时同时增减
- **库内移位要原子**：同一事务内完成减库位+增库位，避免并发导致库存丢失
- **调拨单据要留痕**：调拨申请→审批→出库→在途→入库全流程单据必须完整
- **调拨与采购/销售的区别**：调拨是企业内部行为，不产生收入，财务处理方式不同

---

## 9. 退货处理：RMA流程、退货验收、不合格品处理

### 题目
RMA（退货授权）流程的核心是什么？退货验收与不合格品如何处理？

### 核心答案
**RMA流程状态机**：
```
客户申请退货 ──→ RMA审核 ──→ RMA开具 ──→ 退货入库 ──→ 验收 ──→ 分类处理
     │             │            │            │           │          │
  退货运输   审核数量/原因    生成RMA单    扫描匹配   质检判定    合格/不合格
  费用核算   批准/拒绝        预占库存     ASN匹配    不合格登记
```

**退货验收核心逻辑**：
```python
def rma_validate(rma_no, returned_items):
    """退货验收：比对RMA单与实际退货"""
    
    # 1. RMA单查询
    rma_order = get_rma_order(rma_no)
    
    # 2. 比对清单
    for item in returned_items:
        expected = rma_order.items.find(sku=item.sku)
        if not expected:
            raise ValidationError(f"SKU {item.sku} 不在RMA范围内")
        
        if item.qty > expected.qty:
            raise ValidationError(f"SKU {item.sku} 退货数量超出RMA数量")
        
        # 3. 品质检查
        item.quality = quality_check(item)
        if item.quality == 'DEFECTIVE':
            item.disposition = determine_disposition(item)  # 报废/退供应商/降价处理

def determine_disposition(item):
    if item.defect_type == 'DAMAGED':
        return 'SCRAP'           # 报废
    elif item.defect_type == 'WRONG_ITEM':
        return 'RETURN_TO_SUPPLIER'  # 退供应商
    elif item.defect_type == 'MINOR_DEFECT':
        return 'REWORK'         # 返修
    else:
        return 'HOLD'           # 暂存待定
```

**不合格品处理路径**：
```
不合格品登记 → 不良原因分析 → 处置决策
                              │
        ┌──────────┬──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼          ▼
     退供应商    报废处理    返修再加工   降价销售    转样品
```

### 追问方向
- RMA退货入库后库存如何处理？直接回库还是走质检流程？
- 客户丢失RMA单但实物退回，如何处理？
- 不合格品处置产生的费用（报废/退供应商）走谁的账？
- RMA逆向物流（上门取件）如何与WMS联动？

### 避坑提示
- **RMA必须先审批**：不能让客户随意退货，必须有RMA单号才能收货
- **退货数量≠RMA数量**：实际退货可能少于RMA批准数量，也可能多出（多出部分拒收）
- **不合格品不能再卖**：质量判定为不合格的SKU必须冻结，不能重新参与出库分配
- **RMA账务复杂**：退货涉及退款/换货/赔偿等多种场景，账务处理要分清楚

---

## 10. 复核打包：RF枪操作、称重校验、打包规则匹配

### 题目
复核打包环节的核心功能是什么？称重校验和打包规则如何配合工作？

### 核心答案
**复核打包流程**：
```
拣货集货 ──→ 复核扫描 ──→ 称重校验 ──→ 打包操作 ──→ 贴标出库
    │            │              │            │           │
  订单合流    SKU/数量确认   重量阈值比对   规则匹配    面单打印
             批次效期扫描   超重/欠重拦截  包装箱选择  交接签收
```

**RF复核操作**：
```python
def rf_verify(order_no, scanned_barcode):
    """RF枪复核核心逻辑"""
    
    # 1. 获取订单待复核明细
    pending_items = get_pending_verify_items(order_no)
    
    # 2. 扫描匹配
    scanned_sku = parse_barcode(scanned_barcode)['sku']
    scanned_qty = scanned_barcode['qty']  # 可能含数量信息
    
    # 3. 比对
    match_item = find_match(pending_items, scanned_sku)
    if not match_item:
        raise VerifyError("SKU不匹配，请检查")
    
    if scanned_qty != match_item.pending_qty:
        # 数量差异
        handle_qty_diff(order_no, match_item, scanned_qty)
    
    # 4. 批次效期校验（食品/药品必须）
    if requires_expiry_check(match_item.sku):
        verify_expiry(scanned_barcode['batch'])
        verify_batch_frozen_status(scanned_barcode['batch'])
    
    # 5. 标记复核完成
    mark_verified(order_no, match_item, scanned_qty)
```

**称重校验逻辑**：
```
理论重量 = Σ(商品重量 × 数量) + 包装材料重量
允许误差 = ±X% (行业标准通常 3-5%)

if actual_weight <理论重量 × (1 - tolerance):
    → 欠重报警：少货风险，系统拦截
elif actual_weight > 理论重量 × (1 + tolerance):
    → 超重报警：多货/包装错误，系统拦截
else:
    → 放行：进入打包环节
```

**打包规则匹配**：
| 规则维度 | 说明 |
|---------|------|
| 商品特性 | 易碎/液体/食品/药品 → 专用包装 |
| 商品尺寸 | 匹配包装箱型（1-5号箱） |
| 商品重量 | 超重自动拆包，分箱打包 |
| 订单数 | 多品订单按合包规则合并 |
| 温层要求 | 冷冻/冷藏/常温分区打包 |

### 追问方向
- RF复核时发现数量不对，是直接修改还是走差异流程？
- 称重数据与WMS联动后，如何处理称重设备故障时的备援方案？
- 打包规则是固定规则还是可配置？多规则冲突时优先级如何定？
- 一单多件（一个订单多个包裹）如何处理？

### 避坑提示
- **复核≠只扫条码**：复核要扫数量，有些WMS只扫一件但实际要发多件，容易漏发
- **称重要校准**：称重设备每日/每周需校准，否则校验失效
- **批次效期必扫**：食品/药品/化妆品复核时必须校验批次效期，否则合规风险
- **复核与打包分离**：不要把复核和打包混成一个环节，出问题难以定位

---

## 11. 波次计划：订单波次策略（数量/时间/路线）、波次生成算法

### 题目
波次计划的核心目标是什么？有哪些主流波次策略？波次生成算法的关键点是什么？

### 核心答案
**波次计划目标**：
```
核心目标：提高拣货效率，降低行走距离，缩短订单履约时效

传统逐单拣货：N个订单 → N次行走 → 效率低
波次合单拣货：N个订单 → 1次行走 → 按波次合并 → 分单下楼
```

**波次策略类型**：

| 策略 | 触发条件 | 适用场景 | 优点 | 缺点 |
|------|---------|---------|------|------|
| 数量波次 | 凑满X个订单 | 订单量大、品类相似 | 拣货效率高 | 等待时间长 |
| 时间窗口波次 | 每X分钟/每小时 | 追求时效 | 履约及时 | 订单碎片化，效率低 |
| 路线波次 | 同区域/同配送路线 | 配送路线稳定 | 集货装车方便 | 受路线限制 |
| 品类波次 | 同品类/同温层 | 冷链/医药/食品 | 温层管理简单 | 品类分散时无效 |
| 重量波次 | 总重达到阈值 | 卡车载重优化 | 装载率高 | 计算复杂 |
| 混合策略 | 上述组合 | 复杂业务 | 综合最优 | 配置复杂 |

**波次生成算法（伪代码）**：
```python
def generate_wave(orders, strategy_config):
    """波次生成核心算法"""
    
    # 1. 订单池过滤
    eligible_orders = filter_orders(orders, 
        status='PAID', 
        ship_time>=current_time,
        not_in_any_wave=True
    )
    
    # 2. 按策略分组
    if strategy == 'QUANTITY':
        groups = group_by_quantity(eligible_orders, 
                                   max_orders_per_wave=20)
    elif strategy == 'TIME_WINDOW':
        groups = group_by_time_window(eligible_orders,
                                       window_minutes=30)
    elif strategy == 'ROUTE':
        groups = group_by_route(eligible_orders)
    elif strategy == 'ZONE':
        groups = group_by_pick_zone(eligible_orders)
    
    # 3. 各组内订单排序（按库位路径优化）
    for group in groups:
        group.orders.sort(key=lambda o: o.pick_path)  # S型路径排序
    
    # 4. 生成波次单
    wave_id = create_wave(group.orders, strategy)
    
    # 5. 生成拣货任务
    generate_pick_tasks(wave_id)
    
    return wave_id
```

### 追问方向
- 波次订单数量设多少合适？过多/过少有什么问题？
- 波次生成后可以临时加减订单吗？加减订单的逻辑是什么？
- 波次与拣货路径优化算法是什么关系？
- 多仓库/多温层如何做波次？

### 避坑提示
- **波次不能无限等**：时间窗口波次要设最大等待时间，否则订单积压
- **波次生成后要锁定**：已生成波次的订单不能被其他波次重复占用
- **波次回退要谨慎**：波次取消/回退时，涉及的库存预占要全部释放
- **拣货单元 ≠ 发货单元**：拣货时可能整箱拣，但发货是按订单件数，要分清楚

---

## 12. 摘果法vs播种法：适用场景对比、WMS中如何选择

### 题目
摘果法（Pick to Order）和播种法（Pick and Pack）的核心区别是什么？各自的优缺点和适用场景？

### 核心答案
**摘果法（Discrete Picking / Pick to Order）**：
```
作业员路径：A库位→B库位→C库位（按顺序）
每个库位取完一个订单的全部商品
到复核台/打包台后打包成一个包裹
```

**播种法（Batch Picking / Pick and Pack）**：
```
作业员路径：A库位取X件（所有需要此SKU的订单总和）
到播种墙：按订单播种（一个库位→N个订单格口）
每个订单格口累积完整后打包
```

**对比表格**：

| 维度 | 摘果法 | 播种法 |
|------|--------|--------|
| 订单数 | 1单 | 多单（一个波次） |
| 行走距离 | 长（每个订单走全程） | 短（一次走完所有库位） |
| 拣货效率 | 低 | 高 |
| 复核难度 | 低（单一订单） | 高（需播种墙+复核） |
| 适用场景 | 大件/高价值/复杂订单 | 小件/标准化/高频订单 |
| 系统复杂度 | 低 | 高（需播种墙管理） |
| 订单延迟 | 无 | 有（等波次） |

**WMS策略选择决策树**：
```
订单特征判断：
├── 单均件数 ≥ 10件 → 摘果法（播种法需大量格口）
├── 单均件数 < 5件 → 播种法（高频高效）
├── 商品尺寸大（家具/家电）→ 摘果法（播种墙放不下）
├── 效期敏感（药品）→ 摘果法（批次追溯简单）
├── 品类单一/相似 → 播种法（SKU集中）
└── 订单时效要求极高 → 摘果法（不等待波次）
```

### 追问方向
- 摘果法如何与复核打包环节衔接？
- 播种墙的格口数量如何配置？少了怎么办？
- 播种法如何处理错播（播错格口）？
- 两种方式如何混合使用（半摘果半播种）？

### 避坑提示
- **不是二选一**：很多仓库是混合模式，大件摘果+小件播种
- **播种墙投资**：播种墙需要物理空间和维护，初始投入不小
- **错播风险**：播种法错播率比摘果法高，需要二次复核
- **时效与效率的权衡**：摘果快但效率低，播种效率高但有等待延迟，要看业务优先级

---

## 13. 库存异常处理：库存为负数原因、库存账实差异排查

### 题目
WMS中库存出现负数的原因是什么？如何系统性地排查库存账实差异？

### 核心答案
**库存为负数的常见原因**：

| 原因分类 | 具体场景 | 发生环节 |
|---------|---------|---------|
| 作业逆向 | 拣货确认数量>实际可用 | 拣货 |
| 批次错误 | 分配时批次搞错，扣错批次 | 分配 |
| 并发冲突 | 多用户同时操作同一SKU+批次 | 并发 |
| 回退失败 | 取消订单/回退时事务失败 | 回退 |
| 盘点漏盘 | 盘点时漏扫但已确认完成 | 盘点 |
| 上架虚报 | 上架数量>实际入库数量 | 上架 |
| 冻结失败 | 冻结操作被回滚但已预占 | 冻结 |

**账实差异排查流程**：
```
Step 1: 确定差异范围
├── SKU / 批次 / 库位 / 时间窗口
└── 查询：系统库存 vs 实际库存

Step 2: 追踪流水
SELECT operation_type, qty_change, reference_no, 
       created_at, created_by
FROM wms_inventory_transaction
WHERE sku_id = ? AND batch_no = ? AND location_id = ?
  AND created_at BETWEEN ? AND ?
ORDER BY created_at;

Step 3: 逐笔核对
├── 入库单据 vs 上架实数
├── 拣货单据 vs 复核确认数
├── 调拨单据 vs 移库确认数
└── 盘点单据 vs 差异确认数

Step 4: 定位异常单据
├── 某笔入库/出库单据数量与实际不符
└── 某笔单据状态异常（部分回退/超时未完成）

Step 5: 差异处理
├── 账务调整（经审批后）
└── 实物核查（必要时复盘）
```

**预防负库存机制**：
```sql
-- 库存扣减前置检查（防止负库存写入）
UPDATE wms_inventory 
   SET qty = qty - @reduce_qty
 WHERE location_id = @loc 
   AND sku_id = @sku 
   AND batch_no = @batch
   AND qty >= @reduce_qty;  -- 必须满足此条件

-- 影响行数为0时 → 库存不足，抛出异常
```

### 追问方向
- 库存为负数时系统是否应该阻止？还是有负即查？
- 负库存发现后如何处理账务（直接调账vs原因追溯）？
- 如何设计负库存预警（每日/实时）？
- 跨仓库调拨时库存一出一进，入库方负库存怎么处理？

### 避坑提示
- **负库存不可怕，但要追源**：负库存是结果，不是原因，要找到导致负库存的那笔操作
- **不要直接调账**：未经排查直接调账是正负抵消，掩盖问题，下次还会发生
- **并发是常见原因**：多线程/多设备同时操作同一库存时，必须加锁或乐观锁
- **定时任务要幂等**：库存校验/同步的定时任务必须可重复执行，否则重复运行会出问题

---

## 14. ASN预先到货通知：ASN与采购单关联、到货扫码匹配

### 题目
ASN（Advanced Shipping Notification）的核心价值是什么？ASN与采购单如何关联？到货扫码匹配流程是什么？

### 核心答案
**ASN核心价值**：
```
ASN = 供应商提前告知：我将在什么时间送什么货

核心目的：
1. 提前分配库位资源
2. 缩短到货验收时间
3. 实现采购-到货-入库全流程追溯
4. 为供应商KPI提供数据（准时率/准确率）
```

**ASN与采购单关联**：
```
采购单（PO）主数据：
├── PO号、供应商信息、约定交期
├── 采购明细：SKU/数量/单价
└── 采购类别：常规/紧急/VMI

ASN数据结构：
├── ASN号（供应商生成 or 系统分配）
├── 关联PO号（支持一ASN对一PO or 一ASN对多PO）
├── 预计到货时间、车牌号、司机信息
├── ASN明细：SKU/数量/批次/效期/生产日期
└── 包装信息：件数/托数/箱规
```

**到货扫码匹配流程**：
```python
def asn_receiving(scan_barcode):
    """到货扫码匹配核心逻辑"""
    
    # 1. 解析条码（ASN号/PO号/箱码）
    parsed = parse_barcode(scan_barcode)  
    
    # 2. 匹配ASN
    asn = match_asn(parsed.asn_no)
    if not asn:
        raise Error("ASN不存在，请核实")
    
    # 3. 校验ASN状态
    if asn.status == 'RECEIVED':
        raise Error("ASN已完成收货，不能重复扫描")
    
    # 4. 匹配采购单
    po = match_po(asn.po_no)
    
    # 5. 逐件扫描比对
    for item in parsed.items:
        po_line = find_po_line(po, item.sku)
        if not po_line:
            raise Error(f"SKU {item.sku} 不在PO {po.po_no} 中")
        
        # 累计收货数量
        received_qty = get_received_qty(asn.asn_no, item.sku)
        if received_qty + item.qty > po_line.ordered_qty:
            raise Warning(f"SKU {item.sku} 超出PO数量：已收{received_qty}+本票{item.qty}>订单{po_line.ordered_qty}")
        
        # 更新ASN明细
        update_asn_line(asn.asn_no, item.sku, item.qty)
    
    # 6. ASN收货完成确认
    if is_asn_complete(asn):
        close_asn(asn.asn_no)
        trigger_quality_check(asn)

def match_asn(barcode):
    """ASN匹配规则"""
    # 优先精确匹配ASN号
    # 其次匹配供应商箱码（映射ASN）
    # 最后手工录入ASN号
```

### 追问方向
- ASN数量与实际到货不符如何处理（少送/多送/错送）？
- ASN未到但货物先到如何处理？
- 一张PO分多次到货，ASN如何处理？
- ASN与入库单的对应关系是什么？

### 避坑提示
- **ASN是通知，不是入库**：ASN到货不等于实际入库，预收货≠确认收货
- **ASN号要规范**：供应商ASN号必须唯一且可追溯，否则到货匹配会乱
- **ASN数量是参考**：实际收货数量以到货扫码为准，ASN数量仅做预校验
- **ASN完成不等于入库完成**：ASN收货后还要走质检上架，才算真正入库

---

## 15. 容器管理：周转箱/托盘编码、容器与库位绑定

### 题目
容器管理的核心目标是什么？周转箱和托盘的编码体系如何设计？容器与库位如何绑定？

### 核心答案
**容器类型**：
```
┌────────────────────────────────────────┐
│              容器类型                    │
├──────────────┬──────────────┬──────────┤
│   托盘（Pallet）│  周转箱（Bin） │  纸箱（Carton）│
├──────────────┼──────────────┼──────────┤
│  层级：库位    │  层级：库位    │ 层级：库位    │
│  承重：1-1.5T  │  承重：5-50kg │  承重：按规格 │
│  尺寸：标准    │  尺寸：多种    │  尺寸：不定   │
│  可循环       │  可循环       │  一次性      │
└──────────────┴──────────────┴──────────┘
```

**容器编码规则（推荐）**：
```
托盘编码：PLT-{供应商前缀}-{年月}-{流水号}
示例：PLT-SUPA-202305-00001

周转箱编码：BIN-{仓库}-{箱型代码}-{流水号}
示例：BIN-WH01-A-00001
（箱型代码：A=小号, B=中号, C=大号, D=特大号）

纸箱编码（一次性）：CTN-{ASN号}-{序号}
示例：CTN-ASN20230515-001
```

**容器与库位绑定关系**：
```sql
-- 容器当前位置表
CREATE TABLE wms_container_location (
    container_id    VARCHAR(50) PRIMARY KEY,
    container_type  ENUM('PALLET','BIN','CARTON'),
    location_id     VARCHAR(50),          -- 当前库位
    status          ENUM('IN_USE','EMPTY','SCRAPPED'),
    bind_sku_id     VARCHAR(50),          -- 绑定SKU（可选）
    bind_batch_no   VARCHAR(50),          -- 绑定批次（可选）
    last_update     DATETIME,
    last_operator   VARCHAR(50)
);

-- 容器与库位绑定查询
SELECT c.container_id, c.location_id, l.zone, l.aisle, l.column, l.level
FROM wms_container_location c
JOIN wms_location l ON c.location_id = l.location_id
WHERE c.container_id = 'PLT-SUPA-202305-00001';
```

**容器作业流程**：
```
托盘：入库上架 → 绑定库位 → 库存存放 → 整托出库/拆托 → 解绑库位
周转箱：空箱发出 → 收货装箱 → 绑定库位 → 库内流转 → 拣货集货 → 循环回收
```

### 追问方向
- 容器循环回收如何管理（回收率/丢失率）？
- 托盘和周转箱的容量如何设定（最大SKU数/重量）？
- 容器与批次的绑定关系：一个容器能否混放多批次？
- 容器追溯如何实现（条码/RFID）？

### 避坑提示
- **容器状态要管理**：空容器和满容器要分开状态，否则不知道容器里有没有货
- **容器编码要唯一**：同一编码不能同时存在于两个地方（除非是复制品要作废）
- **托盘不要超载**：托盘有承重和体积限制，超载入库时要预警
- **一次性容器要标记**：纸箱等一次性容器不能按循环容器管理，要单独分类

---

## 16. 作业策略：先进先出FIFO、后进先出LIFO、效期优先

### 核心答案
**三大库存策略对比**：

| 策略 | 定义 | 适用场景 | 优点 | 缺点 |
|------|------|---------|------|------|
| FIFO（先进先出） | 先入库先出库 | 食品/药品/化妆品等效期管理 | 效期风险最低 | 通道利用率低 |
| LIFO（后进先出） | 后入库先出库 | 大宗商品/建材（价格波动） | 库存成本接近现价 | 效期风险高 |
| 效期优先（FEFO） | 最近效期先出 | 药品/生鲜/高值商品 | 效期管控最严 | 计算复杂 |

**FIFO实现逻辑**：
```sql
-- 库存分配时按批次时间排序
SELECT batch_no, production_date, qty,
       DATEDIFF(expiry_date, CURDATE()) AS days_to_expire
FROM wms_batch_stock
WHERE sku_id = @sku
  AND qty > 0
  AND frozen_status = 'NORMAL'
  AND location_id NOT IN (SELECT location_id FROM wms_location WHERE status = 'DISABLED')
ORDER BY production_date ASC,  -- FIFO：按生产日期升序
         expiry_date ASC       -- 效期相近时优先出
LIMIT @required_qty;
```

**效期优先（FEFO）实现**：
```sql
-- 效期优先分配：按到期日期升序
SELECT batch_no, expiry_date, qty,
       DATEDIFF(expiry_date, CURDATE()) AS days_to_expire
FROM wms_batch_stock
WHERE sku_id = @sku
  AND qty > 0
  AND frozen_status = 'NORMAL'
  AND days_to_expire > 0  -- 排除已过期
ORDER BY days_to_expire ASC  -- 最近效期优先
LIMIT @required_qty;
```

**策略配置示例**：
```python
# WMS作业策略配置
strategy_config = {
    "default_strategy": "FIFO",           # 默认策略
    "sku_override": {
        "SKU-MEDICINE-001": "FEFO",       # 药品效期优先
        "SKU-BUILD-MATERIAL": "LIFO",     # 建材后进先出
    },
    "zone_override": {
        "ZONE-FRESH": "FEFO",             # 生鲜区效期优先
    },
    "emergency_override": {
        "allow_expired": False,            # 紧急订单不允许出过期品
        "allow_near_expiry": True          # 近效期需审批
    }
}
```

### 追问方向
- FIFO一定保证批次先进先出吗？库位分散时如何保证？
- LIFO在仓储管理中是否合规（会计准则不允许，但实际操作存在）？
- FEFO和FIFO混合使用时如何处理？
- 拣货时发现FIFO顺序的批次库位较远，是走远路还是打破FIFO？

### 避坑提示
- **FIFO不是按批次号大小**：而是用生产日期/入库日期排序，有些系统批次号≠入库顺序
- **策略可叠加**：同SKU可以同时设置FIFO+效期优先（先进先出，但效期不足时跳仓）
- **紧急订单要设例外**：FIFO严格按序，但紧急插单时要有审批旁路
- **策略要与库位规划配合**：FIFO仓库通道设计要单向或S型，不能让拣货员来回折返

---

## 17. 库存预留：订单预留/活动预留、预留释放策略

### 题目
库存预留的几种类型是什么？订单预留与活动预留如何区分？预留释放的策略有哪些？

### 核心答案
**预留类型**：
```
┌─────────────────────────────────────────────────┐
│                 库存预留                          │
├────────────────────┬────────────────────────────┤
│    订单预留         │      活动预留               │
│ (Order Reservation) │   (Campaign Reservation)    │
├────────────────────┼────────────────────────────┤
│ 客户已下单，锁定库存 │ 大促/活动前提前锁定库存     │
│ 预留量=订单需求量    │ 预留量=预估销量×安全系数   │
│ 订单取消自动释放    │ 活动结束手动/定时释放      │
│ 时效：分钟~天       │ 时效：天~周               │
└────────────────────┴────────────────────────────┘
```

**预留释放策略**：

| 释放条件 | 触发时机 | 释放方式 |
|---------|---------|---------|
| 订单超时未支付 | 支付超时（30分钟/1小时） | 系统自动释放 |
| 订单取消 | 用户主动取消 | 即时释放 |
| 活动结束 | 活动到期时间点 | 定时释放 |
| 库存不足释放 | 库存被更高优先级占用 | 自动挤占 |
| 人工干预释放 | 客服/运营手动操作 | 审批后释放 |
| 永久保留 | 特殊业务（样品） | 永不过期 |

**预留库存计算**：
```sql
-- 可用库存 = 总库存 - 订单预占 - 活动预占 - 冻结库存
SELECT 
    i.sku_id,
    i.total_qty AS 总库存,
    i.total_qty - 
        COALESCE(r.order_reserved_qty, 0) -
        COALESCE(r.campaign_reserved_qty, 0) -
        COALESCE(r.frozen_qty, 0) AS 可用库存
FROM wms_inventory i
LEFT JOIN (
    SELECT sku_id,
           SUM(CASE WHEN reservation_type = 'ORDER' THEN qty ELSE 0 END) AS order_reserved_qty,
           SUM(CASE WHEN reservation_type = 'CAMPAIGN' THEN qty ELSE 0 END) AS campaign_reserved_qty,
           SUM(CASE WHEN reservation_type = 'FROZEN' THEN qty ELSE 0 END) AS frozen_qty
    FROM wms_reservation
    WHERE status = 'ACTIVE'
    GROUP BY sku_id
) r ON i.sku_id = r.sku_id;
```

### 追问方向
- 预留与预占的区别是什么？
- 活动预留给多了（浪费）/少了（超卖）如何处理？
- 高优先级订单如何"抢"低优先级的预留库存？
- 预留释放后实物已拣货，如何处理？

### 避坑提示
- **预留≠预占**：预留是逻辑概念（告诉系统我要用），预占是物理动作（实际扣库存）
- **预留要设时效**：预留不能无限期，否则库存被无效占用
- **释放要幂等**：预留释放操作要可重复执行，防止并发导致重复释放
- **活动预留要谨慎**：活动预留占用真实库存，会影响正常销售，要设置合理上限

---

## 18. 计费引擎：仓储费/作业费/增值服务费、计费规则配置

### 题目
WMS计费引擎的核心功能是什么？仓储费、作业费、增值服务费分别如何计算？

### 核心答案
**计费三大维度**：

| 费用类型 | 计算基础 | 常见计费项 |
|---------|---------|-----------|
| 仓储费 | 库存体积/重量/托数/库位数 | 存储费、仓租 |
| 作业费 | 作业数量/重量/次数 | 入库费、出库费、拣货费、复核费 |
| 增值服务费 | 服务类型 | 贴标费、组套费、质检费、销毁费 |

**计费规则配置模型**：
```python
# 计费规则配置
billing_rules = {
    "storage": {
        "model": "VOLUME_MONTH",  # 按体积·月计费
        "unit_price": 15.0,       # 元/m³/月
        "min_charge": 0.5,        # 最低起算量
        "free_days": 30,          # 入库前30天免仓租（促销期）
    },
    "inbound": {
        "model": "PER_ORDER",     # 按单计费
        "unit_price": 5.0,        # 元/单
        "free_threshold": 1,      # 首单免费
    },
    "outbound": {
        "model": "PER_ITEM",      # 按件数计费
        "tiers": [
            {"max_qty": 100, "price": 0.5},
            {"max_qty": 500, "price": 0.4},
            {"max_qty": None, "price": 0.3},  # 量越大越便宜
        ]
    },
    "value_added": {
        "LABEL": {"model": "PER_ITEM", "price": 0.2},
        "REPACK": {"model": "PER_ITEM", "price": 1.5},
        "QUALITY_CHECK": {"model": "PER_ORDER", "price": 3.0},
    }
}
```

**计费触发与计算**：
```sql
-- 入库计费触发
INSERT INTO billing_record (tenant_id, charge_type, reference_no, 
                             sku_id, qty, unit_price, amount, 
                             billing_date, status)
SELECT 
    i.tenant_id,
    'INBOUND',
    i.inbound_no,
    i.sku_id,
    SUM(i.qty),
    r.unit_price,
    SUM(i.qty) * r.unit_price,
    CURDATE(),
    'PENDING'
FROM wms_inbound_detail i
JOIN billing_rules r ON r.charge_type = 'INBOUND' 
                     AND r.tenant_id = i.tenant_id
WHERE i.status = 'COMPLETED'
  AND NOT EXISTS (
      SELECT 1 FROM billing_record b 
      WHERE b.reference_no = i.inbound_no 
        AND b.charge_type = 'INBOUND'
  )
GROUP BY i.tenant_id, i.inbound_no, i.sku_id;
```

### 追问方向
- 仓储费是按毛重还是净重/体积重计费？
- 计费周期如何确定（自然月/账单月）？
- 货主对账时发现差异如何处理（账单异议流程）？
- 增值服务费如何与作业单据关联（证明完成了服务）？

### 避坑提示
- **计费规则要可配置**：不能写死代码，要支持界面配置和版本管理
- **计费数据要独立**：计费明细要与作业单据分离，便于对账和追溯
- **优惠/折扣要记录**：任何优惠都要有依据，不能直接改金额
- **账单要支持导出**：货主对账需要标准格式（Excel/CSV/PDF）

---

## 19. 货主多租户：数据隔离方案（tenant_id）、跨货主查询

### 题目
WMS多货主（多租户）场景下，数据隔离方案是什么？跨货主查询如何实现？

### 核心答案
**数据隔离方案**：

| 隔离级别 | 实现方式 | 优点 | 缺点 |
|---------|---------|------|------|
| 独立数据库 | 每个货主一个DB | 完全隔离，安全最高 | 成本高，管理复杂 |
| Schema隔离 | 每个货主一个Schema | 隔离较好，成本适中 | 跨货主查询麻烦 |
| tenant_id列隔离 | 所有数据同一表，加tenant_id | 查询灵活，资源共享 | 隔离性较弱 |

**推荐方案：tenant_id列级隔离**：
```sql
-- 所有业务表都包含tenant_id
CREATE TABLE wms_inventory (
    id              BIGINT PRIMARY KEY,
    tenant_id       VARCHAR(50) NOT NULL,     -- 货主ID（必加）
    sku_id          VARCHAR(50) NOT NULL,
    batch_no        VARCHAR(50),
    qty             DECIMAL(18,4),
    location_id     VARCHAR(50),
    created_at      DATETIME,
    updated_at      DATETIME,
    INDEX idx_tenant_sku (tenant_id, sku_id),  -- 复合索引优化
    INDEX idx_tenant_loc (tenant_id, location_id)
);

-- 所有查询默认带上tenant_id
SELECT * FROM wms_inventory 
WHERE tenant_id = @current_tenant_id  -- 全局过滤器
  AND sku_id = @sku;
```

**跨货主查询（仅限授权场景）**：
```python
# 跨货主查询需要显式授权
def cross_tenant_query(requester_tenant_id, target_tenant_ids, query_params):
    """跨货主查询 - 仅对账/报表场景"""
    
    # 1. 权限校验：requester是否有权限查看target货主数据
    if not has_cross_tenant_permission(requester_tenant_id, target_tenant_ids):
        raise PermissionDenied("无权跨货主查询")
    
    # 2. 查询时明确列出target货主ID
    sql = """
        SELECT 
            tenant_id,
            sku_id,
            SUM(qty) as total_qty
        FROM wms_inventory
        WHERE tenant_id IN :target_tenant_ids
          AND sku_id = :sku_id
        GROUP BY tenant_id, sku_id
    """
    return execute_query(sql, target_tenant_ids=target_tenant_ids, **query_params)
```

**全局过滤器实现**：
```python
# MyBatis风格的全局过滤器
class TenantContextFilter:
    def do_filter(request, response):
        # 从登录态获取当前货主ID
        tenant_id = get_current_tenant(request)
        
        # 注入到ThreadLocal
        TenantContext.set_tenant_id(tenant_id)
        
        try:
            chain.do_filter(request, response)
        finally:
            TenantContext.clear()

# 所有Repository查询自动追加tenant_id条件
class InventoryRepository:
    def find_by_sku(sku_id):
        return db.query(
            f"SELECT * FROM wms_inventory WHERE tenant_id = '{TenantContext.get()}' AND sku_id = '{sku_id}'"
        )
```

### 追问方向
- 货主切换时Session如何处理？
- 货主间库存是否可以互借/调拨？
- 多货主场景下如何做跨货主报表？
- 货主数据隔离级别是否支持不同等级（金融级vs普通商业级）？

### 避坑提示
- **tenant_id必加索引**：所有查询都以tenant_id为前置条件，不加索引查询极慢
- **跨租户操作要审批**：库存调拨/数据导出等跨货主操作必须审批留痕
- **数据初始化要隔离**：创建货主时必须初始化独立的货主配置，不能共用
- **测试要造多货主数据**：测试多货主功能时必须用多个货主数据测，不能只用一个货主测

---

## 20. WMS接口设计：REST API规范、WebService接口、消息通知机制

### 题目
WMS对外接口设计有哪些方式？REST API规范如何定义？消息通知机制如何实现？

### 核心答案
**接口设计三种方式对比**：

| 接口类型 | 协议 | 数据格式 | 适用场景 | 特点 |
|---------|------|---------|---------|------|
| REST API | HTTP/HTTPS | JSON | 轻量级对接/互联网 | 灵活、易调试 |
| WebService | SOAP | XML | 企业级/ESB对接 | 规范、事务支持 |
| 消息队列 | MQ | JSON/二进制 | 异步解耦/高并发 | 松耦合、削峰填谷 |

**REST API规范设计**：
```yaml
# OpenAPI 3.0 规范示例
openapi: 3.0.0
info:
  title: WMS API
  version: 2.1.0
  description: 仓储管理系统对外接口

paths:
  /api/v2/inbound/orders:
    post:
      summary: 创建入库单
      tags: [入库管理]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/InboundOrder'
      responses:
        '201':
          description: 创建成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InboundOrderResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  schemas:
    InboundOrder:
      type: object
      required: [tenant_id, warehouse_id, supplier_id, items]
      properties:
        tenant_id:
          type: string
          description: 货主ID
          example: "TENANT-001"
        warehouse_id:
          type: string
          description: 仓库编码
          example: "WH-02"
        expected_arrival_time:
          type: string
          format: date-time
        items:
          type: array
          items:
            $ref: '#/components/schemas/InboundOrderItem'

    InboundOrderItem:
      type: object
      required: [sku_id, expected_qty]
      properties:
        sku_id:
          type: string
          example: "SKU-A001"
        expected_qty:
          type: integer
          minimum: 1
        batch_no:
          type: string
        expiry_date:
          type: string
          format: date
```

**WebService接口（WSDL示例）**：
```xml
<!-- SOAP WSDL片段 -->
<message name="createInboundRequest">
  <part name="tenantId" type="xsd:string"/>
  <part name="warehouseId" type="xsd:string"/>
  <part name="asnNo" type="xsd:string"/>
  <part name="items" type="tns:ArrayOfInboundItem"/>
</message>

<portType name="WMSPort">
  <operation name="createInboundOrder">
    <input message="tns:createInboundRequest"/>
    <output message="tns:createInboundResponse"/>
  </operation>
</portType>
```

**消息通知机制（RabbitMQ示例）**：
```python
# WMS事件发布
def publish_wms_event(event_type, payload):
    """WMS事件发布到消息队列"""
    message = {
        "event_id": generate_uuid(),
        "event_type": event_type,  # INBOUND_CREATED/OUTBOUND_SHIPPED
        "tenant_id": get_current_tenant(),
        "timestamp": datetime.now().isoformat(),
        "payload": payload
    }
    
    exchange = 'wms.events'
    routing_key = f"wms.{event_type.lower()}"
    
    channel.basic_publish(
        exchange=exchange,
        routing_key=routing_key,
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # 持久化消息
            content_type='application/json'
        )
    )

# 消费方订阅
def on_inbound_completed(ch, method, properties, body):
    event = json.loads(body)
    if event['event_type'] == 'INBOUND_COMPLETED':
        # 更新ERP采购到货状态
        erp.update_po_receipt(event['payload']['po_no'])
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

**API版本管理策略**：
```
URL版本 vs Header版本：
✗ /api/inbound/orders (变更多个版本后混乱)
✓ /api/v1/inbound/orders (推荐，清晰可追溯)
✓ X-API-Version: 2.0 (Header方式，不污染URL)

版本兼容策略：
├── 新增字段：向后兼容 ✅
├── 删除字段：向后不兼容 ❌
├── 修改字段类型：不兼容 ❌
└── 废弃字段：标记deprecated，过渡期保留
```

### 追问方向
- WMS接口如何做幂等性设计？
- 消息队列消费失败如何处理？重试机制是什么？
- 接口限流和熔断如何实现？
- WMS与ERP/MES的集成架构是怎样的？

### 避坑提示
- **接口幂等必须做**：入库/出库/调拨等操作接口必须支持幂等，否则重复调用会账务混乱
- **异步消息要可靠**：MQ消息不能丢，需要Confirm/Ack机制
- **接口文档要实时**：API规范要与代码同步，建议用Swagger/OpenAPI自动生成
- **错误码要统一**：全局错误码规范，便于调用方处理

---

## 附录：面试高频追问TOP10

1. **库存为负数了怎么办？** → 先查流水定位原因，不直接调账
2. **FIFO和FEFO哪个更好？** → 看业务场景，药品用FEFO，常规品用FIFO
3. **波次订单数量设多少合适？** → 看单均件数和时效要求，20-50单是常见区间
4. **拣货时发现实物与系统不符？** → 停止该库位作业，标记差异，触发盘点
5. **预收货和确认收货的区别？** → 预收货不产生库存，确认收货才入账
6. **盘点时仓库要不要停业？** → 不停业方案更常见，但需要冻结受影响库位
7. **ASN匹配失败如何处理？** → 人工介入，查明是ASN未发还是到货信息错误
8. **多货主库存可以互借吗？** → 技术上可以，但需要严格的审批和成本结算
9. **称重校验失败怎么办？** → 复核员检查，必要时开包验货，记录差异
10. **接口超时如何处理？** → 重试3次+幂等ID，永久失败进入异常处理队列

---

*本文档由Hermes Agent生成 | 版本：v1.0 | 生成时间：2026-05-11*

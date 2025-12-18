## Nonce Management 设计说明

把系统拆成两层：

1. **路由层**：尽量把 signer 请求打到“应负责的节点”（性能）
2. **执行权层**：最终以“租约与 fencing”决定谁能对该 signer 做关键写入（正确性）

## 目标

### 1 目标
- 多实例环境下, 对同一 submitter 提供全局串行化的关键写入能力
- 关键写入具备硬性隔离能力(旧节点写入必须失败)
- 交易生命周期可追踪: 分配 nonce, 写入 txHash, receipt 落库, 终局推进, 卡住治理
- 组件边界清晰, 允许后续演进(例如 终局确认, 分叉处理, 严格连续模式)

---

## 3. 总体架构

### 3.1 架构总览

```mermaid
flowchart LR
  A["业务方"] --> B["负载均衡"]
  B --> N1["交易管理节点 A"]
  B --> N2["交易管理节点 B"]
  B --> N3["交易管理节点 C"]

  subgraph C1["集群"]
    N1 --> P["PostgreSQL(权威状态)"]
    N2 --> P
    N3 --> P
    N1 --> R["链 RPC(连接器)"]
    N2 --> R
    N3 --> R
  end
```

设计原则: 负载均衡仅负责转发流量, 不参与正确性. 正确性由 PostgreSQL 的执行权与栅栏机制保证.

### 3.2 节点内并发模型

```mermaid
flowchart TD
  I["入口请求"] --> Q["工作队列(按 submitter 分片)"]
  Q --> W["单线程工作者"]
  W --> L["租约管理"]
  W --> D["数据库写入(带栅栏校验)"]
  W --> R["链交互(发送和查询)"]
```

节点内串行化用于降低数据库冲突与便于批处理, 但不作为唯一正确性依赖.

---

## 5. 核心机制

### 5.1 执行权: 租约与栅栏

#### 5.1.1 抢占与续租流程

```mermaid
flowchart TD
  S["开始获取或续租"] --> L["锁定租约记录(行锁)"]
  L --> E{"租约存在"}
  E -->|否| I["插入新租约(持有者=本节点 栅栏号=1)"]
  E -->|是| O{"owner 是自己"}
  O -->|是| V{"租约未过期"}
  V -->|是| R["续租(更新 expires_at)"]
  V -->|否| T["抢占(栅栏号自增)"]
  O -->|否| X{"租约已过期"}
  X -->|是| T
  X -->|否| F["失败(非 owner)"]
```

约束: 过期判断使用数据库时间, 避免依赖本地时钟.

#### 5.1.2 栅栏写入规则

所有关键写入必须携带以下校验:
- owner_node 等于当前节点
- expires_at 大于当前数据库时间
- fencing_token 等于本次持有的栅栏号

若更新影响行数为 0, 视为被栅栏拒绝, 当前节点必须停止对该 submitter 的推进, 由新 owner 接管.

### 5.2 Nonce 分配: 条件更新 + 唯一约束 + 超时回收

```mermaid
flowchart TD
  classDef biz fill:#FADBD8,stroke:#C0392B,stroke-width:1px,color:#641E16;
  classDef core fill:#D6EAF8,stroke:#2E86C1,stroke-width:1px,color:#0B3D91;
  classDef rule fill:#D5F5E3,stroke:#1E8449,stroke-width:1px,color:#0B3D2E;
  classDef db fill:#FCF3CF,stroke:#B7950B,stroke-width:1px,color:#7D6608;
  classDef chain fill:#FDEBD0,stroke:#B9770E,stroke-width:1px,color:#6E2C00;

  U[业务调用 开始一次需要nonce的操作]:::biz --> N[Nonce组件 申请并自动释放]:::core
  N --> S[Nonce分配服务]:::core

  S --> A0[分配尝试 短事务 可重试]:::rule

  A0 --> R1[回收超时占用]:::rule
  R1 --> ALLOC[nonce生命周期表]:::db

  A0 --> R2[查找最小可复用nonce]:::rule
  R2 --> ALLOC

  R2 --> D1{是否存在可复用nonce}:::rule
  D1 -->|是| R3[抢占可复用nonce 条件更新]:::rule
  R3 --> ALLOC

  D1 -->|否| R4[读取submitter游标]:::rule
  R4 --> CUR[submitternonce状态表]:::db
  R4 --> R5[乐观推进submitter游标 条件更新]:::rule
  R5 --> CUR
  R5 --> R6[写入占用记录 新nonce占用]:::rule
  R6 --> ALLOC

  ALLOC --> OUT1[返回nonce给业务逻辑]:::core
  OUT1 --> CH[业务发送交易或调用链]:::chain

  CH --> D2{业务调用结果}:::rule
  D2 -->|成功| M1[标记已使用 记录交易哈希]:::rule
  M1 --> ALLOC
  D2 -->|不可重试失败| M2[标记可复用 释放nonce]:::rule
  M2 --> ALLOC
  D2 -->|可重试失败| M3[保持占用 等待业务重试]:::rule

  M1 --> RESP[返回成功结果给业务]:::biz
  M2 --> RESP2[返回失败结果给业务]:::biz
  M3 --> RESP3[返回可重试结果给业务]:::biz
```

关键点:
- next_local_nonce 通过条件更新(比较后更新)保证不重号
- allocation 表唯一约束保证不会重复占号
- markUsed 与 markRecyclable 要求 lock_owner 匹配, 防止超时回收后的迟到回写污染状态

状态名对照(便于与库字段对齐):
- 预留 = RESERVED
- 已使用 = USED
- 可复用 = RECYCLABLE

---

## 6. 交易生命周期

### 6.1 状态机

当前仓库交易状态包含:
- 已分配(ALLOCATED): 已分配 nonce
- 跟踪中(TRACKING): 已持有 tx_hash, 等待回执
- 终局成功(CONFIRMED): 达到终局条件
- 终局失败(FAILED_FINAL): 链上执行失败且已可判定为终局
- 需要治理(STUCK): 长时间无法推进, 需要人工或策略介入

### 6.2 创建与提交(概念流程)

```mermaid
flowchart TD
  A["接收创建请求"] --> B["获取或续租租约"]
  B --> C{"是 owner"}
  C -->|否| X["拒绝处理(非 owner)"]
  C -->|是| D["分配 nonce 并写入交易(带栅栏校验)"]
  D --> E["发送交易到链"]
  E --> F["写入 tx_hash 并进入跟踪(带栅栏校验)"]
```

说明: 当前仓库 txmgr 部分以 service 形态提供写入与提交示例, 对外 API 形态在后续演进中补齐.

### 6.3 ReceiptChecker

目标: 异步查询回执. 查不到回执时不阻塞队列; 出错时逐步延长等待再重试.
- 扫描条件: TRACKING 且 receipt 为空
- receipt 为 null 视为查不到回执(可能尚未上链或节点未同步), 记录检查时间并延迟重试
- receipt 获取成功后仅落库, 终局推进交给 FinalityManager

### 6.4 ResubmitScheduler

目标: 长时间未确认时, 重发同 nonce 的交易, 并避免多节点重复执行外部动作.
- 先在数据库中占位(推进 next_resubmit_at, 带栅栏校验), 再发送链上请求
- 提交成功后写回 tx_hash 或更新重试信息(带栅栏校验)
- 超过最大尝试次数后进入卡住决策

### 6.5 FinalityManager

目标: 将回执推进到终局状态, 并提供最小链重组检测能力.
- 可选 终局确认判定(例如按确认数, 链实现支持时)
- 可选 链重组检测(链实现支持按高度查询区块哈希时)
- 终局写入必须带栅栏校验

### 6.6 卡住治理(StuckResolution)

目标: 提供统一治理入口, 支持业务注入 hook 决策.
- 默认策略不自动取消或替换, 仅延迟或继续尝试
- 若业务选择补救动作, 系统先在数据库中占位再执行发送, 且关键写入仍带栅栏校验

---

## 7. 故障与竞态场景

| 场景                              | 风险                 | 设计处理                                                     |
| --------------------------------- | -------------------- | ------------------------------------------------------------ |
| 故障切换窗口旧节点仍在运行        | 旧节点迟到回写覆盖   | 栅栏写入影响行数为 0, 旧节点无法写入                         |
| 多节点重复重提                    | 重复副作用与写乱状态 | 先在库中占位再发送, 且所有写入带栅栏校验                     |
| 发送灰区(超时但可能已进入 txpool) | 本地缺失 tx_hash     | 预留灰区补齐端口(派生预期 txHash)                            |
| 回执长期不存在                    | 交易长期未确认       | 回执轮询 + 重发 + 卡住治理                                   |
| 链重组                            | 终局判断不稳定       | 最小链重组检测, 不一致进入 STUCK 并等待治理或演进为可回退确认链 |

---

## 8. 可观测性

通过指标端口采集关键事件:
- 租约获取结果(插入, 续租, 抢占, 非 owner)
- 栅栏拒绝次数(按操作区分)
- receipt 查询结果(命中, 未命中, 错误)
- 重提结果(成功, 错误)
- 终局推进结果
- 卡住决策动作
- 队列深度(可选)

日志建议包含: submitter, txId, nodeId, fencing_token.

---

## 9. 演进路线

### 9.1 严格连续模式(single in-flight)

按最终目标设计, 每个 submitter 同时仅允许 1 个 in-flight nonce:
- 创建请求只写入队列态
- 后台按 submitter 串行推进
- 当前 in-flight 被链确认消费后才分配下一个 nonce

### 9.2 黑盒化对外接口

对业务侧仅暴露 txId 与查询接口, 隐藏 nonce 与重试细节:
- CreateTx 返回 txId
- 查询接口返回生命周期状态与可观测信息

### 9.3 可回退确认链与分叉标记

从回执即终局演进到可回退的确认链:
- 确认链列表可覆盖(发生分叉时用新链替换旧链)
- 分叉标记触发下游全量覆盖, 避免拼接两条不同分叉的确认链

### 9.4 数据库迁移管理

引入 Flyway 管理 DDL 与版本演进, 将手工 DDL 迁入 db/migration.

---

## 10. 仓库代码对应关系

- core nonce 模块: `com.work.nonce.core`
  - NonceService, PostgresNonceRepository, NonceComponent
- txmgr 执行权与交易闭环: `com.work.nonce.txmgr`
  - LeaseManager, TransactionWriter, ReceiptChecker, ResubmitScheduler, FinalityManager, StuckResolutionService
- 数据访问: `src/main/resources/mapper/**`
















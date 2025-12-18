### 分布式 Nonce 管理与终局回写方案（Review 版）

### 1. TL;DR（结论先行）
- **仅靠“按 signer 路由到同一台机器”不够**：扩缩容、故障切换、网络分区会产生短暂双活，导致 nonce 分配/重提/终局回写竞态。
- 推荐方案：**路由分片（性能）+ 分布式租约 lease（执行权）+ fencing token（硬隔离旧主写入）**  
  目标是把 FireFly/FFTM 的“同 signer 串行化（单进程）”升级为“同 signer 单主串行化（分布式）”。

---

### 2. 背景与问题定义
#### 2.1 背景
- 系统需要对同一 signer（同一私钥/账户）的交易做 **nonce 分配、提交、重提、receipt/confirmations 跟踪、终局状态回写**。
- 同 signer 的行为天然是强序列：nonce 单调递增，且终局回写不能乱序/覆盖错误。

#### 2.2 核心问题
- 在分布式场景下，同 signer 的请求与后台任务可能被多个节点同时处理：
  - **重复分配 nonce**
  - **重复 resubmit**
  - **旧节点迟到回写覆盖新节点（最危险）**
  - **脑裂/双活窗口内写入互相打架**

#### 2.3 “仅路由到同节点”为什么不够
- 稳态（节点健康、路由稳定）下有效：并发显著降低，接近“单点串行”。
- 但在以下窗口会失效：
  - **故障转移窗口**：路由已切走，但旧节点仍在跑
  - **网络分区**：两边都以为自己负责该 signer
  - **扩缩容/一致性哈希重平衡**：短时间内责任边界不稳定  
→ 这些都会直接破坏 nonce 与终局回写的一致性。

---

### 3. 目标 / 非目标
#### 3.1 目标
- **正确性优先**：任何时刻同 signer 的关键写入只能由一个“合法主”执行。
- **可用性**：主节点故障后能安全接管，不需要全局停机。
- **高性能稳态**：尽量走本地分片，减少全局锁争用。
- **与现有机制兼容**：支持 resubmit、receipt/confirmations、reorg 等闭环。

#### 3.2 非目标（本方案不直接解决）
- gas bump / 动态费率策略细节（可作为后续策略插件）
- 外部系统与本系统共享 signer 的彻底根治（只能“更保守 + 更频繁链上对齐”降低风险窗口）

---

### 4. 方案概览：两层治理
把系统拆成两层（强烈建议按此复用既有架构）：

#### 4.1 路由层（性能优化）
- 目标：尽量把 `signer -> shard -> node` 路由到“应负责节点”
- 手段：一致性哈希/固定分片/网关路由/SDK 路由
- 重要原则：**路由错了也没关系**，因为下一层会挡住错误写入

#### 4.2 执行权层（正确性保证）
- 目标：以分布式方式保证“同一时刻只有一个节点拥有该 signer 的关键写入权”
- 手段：**lease + fencing token**
- 关键原则：**所有关键写操作必须做 fencing 校验**；旧 token 写入一律拒绝（硬切断双活竞态）

---

### 5. 核心机制设计：Lease + Fencing
#### 5.1 数据模型（建议）
表：`signer_lease`

- `signer`：signer 标识（主键）
- `owner_node`：当前持有 lease 的节点 ID
- `fencing_token`：单调递增令牌（每次抢占 ++）
- `expires_at`：lease 过期时间

#### 5.2 抢占/续租规则（语义）
- 节点在处理 signer 前：
  - **续租**：如果自己是 owner 且未过期，则延长 `expires_at`
  - **抢占**：如果 lease 已过期或 owner 不可用，则通过 CAS 更新 owner，并 **`fencing_token++`**
- 关键点：
  - 抢占必须是原子条件更新（CAS）
  - `fencing_token` 必须单调递增，作为“新主身份”的唯一凭据

#### 5.3 Fencing 校验（强制）
- **所有关键写入**必须带上 `fencing_token` 并校验“仍为当前 token”，否则拒绝：
  - nonce 分配落库
  - txHash 写入 / 状态推进
  - resubmit 调度与落库（如 `next_resubmit_at`）
  - receipt/confirmations 的“终局回写/覆盖/回退”
- 结果：即便出现短暂双活，旧节点因 token 过期无法写入，竞态被硬切断。

---

### 6. 关键路径：哪些逻辑必须 leader-only
#### 6.1 必须 leader-only（强一致写）
- **nonce 分配**：只有持 lease 的节点能分配；分配写入必须做 fencing 校验
- **resubmit（同 nonce 重提）**：只有 leader 执行；调度写入必须做 fencing 校验，避免重复调度
- **终局确认回写**：计算可以并行，但“写回 USED / 回退 / 覆盖 confirmations / 状态推进”必须 leader-only + fencing

#### 6.2 可多节点并行（读/计算）
- receipt 拉取、confirmations 计算、链上区块查询等“读 + 计算”可以多节点并行
- 但最终落库/状态推进必须通过 leader-only 写入口

---

### 7. 借鉴 FireFly Transaction Manager（FFTM）的可复用方法论（浓缩）
> 以下是从 FFTM 在 nonce 分配与终局跟踪上的关键设计抽取，用于指导本方案落地。

#### 7.1 “同 signer 串行化”是根本原则
- FFTM 在 Postgres 下通过“确定性路由 writer worker”让同 signer 的写入串行化（进程内保证）
- 本方案将其升级为：**分布式单主 + fencing**，从而在故障切换/脑裂窗口仍正确

#### 7.2 Nonce 分配：max(chain, cache, db) 的保守对齐
- 当缓存有效：使用 `nextNonce` 递增
- 缓存缺失/过期：拉链上 `NextNonceForSigner`，并与本地最高 nonce+1 做 max
- 批次失败要清缓存，避免“缓存递增但 DB 未提交”导致跳号/错乱  
→ 建议作为本系统 nonce 分配策略的参考实现（与 leader-only 写入结合）

#### 7.3 幂等与重复提交的语义分离
- 外部 requestID 需要预查幂等，避免重复请求消耗 nonce
- submit 失败但属于“已知交易 / nonce too low”等场景，需要将“节点已接收”与“本地返回成功/失败”解耦，减少误判引发的错误回收/风暴

#### 7.4 长期 pending：同 nonce 周期性 resubmit
- FFTM simple handler 会在超过间隔后进入 stale 并 resubmit（同 nonce）
- 本系统也应明确：未终局不轻易回收 nonce；超时回收需要更强链上诊断与策略

#### 7.5 终局确认与重组：确认链可回退（newFork 语义）
- confirmations 不应假设单调递增；遇到 reorg 需要支持“回退并替换”
- 下游协议建议：
  - `newFork=true` → **全量覆盖 confirmations**
  - 否则 → 增量追加  
→ 对“终局回写/下游一致性”非常关键

---

### 8. 风险点与缓解
#### 8.1 最大风险：故障转移/脑裂导致双活写入
- **缓解**：fencing token + 关键写入强校验（本方案核心价值）

#### 8.2 迟到写覆盖（旧节点 late commit）
- **缓解**：所有关键写入都以“当前 token”为条件，旧 token 写不进去

#### 8.3 性能担忧：全局锁争用
- **缓解**：稳态依赖路由分片，lease 操作是“按 signer 粒度”，不会变成全局大锁

#### 8.4 共享 signer（外部系统也发交易）
- **缓解**：更保守的 `max(chain, db/cache)`，缩小本地盲增窗口；必要时降低 nonceStateTimeout/提高链上对齐频率  
- **现实边界**：无法彻底消除冲突，只能做风险控制与可观测告警

---

### 9. 配置建议（对齐 FFTM 的“正确性/性能杠杆”思路）
建议将以下作为可调参数（名字可按你们系统规范落地）：
- **confirmations.required**：终局确认阈值
- **confirmations.staleReceiptTimeout**：pending 交易强制重查 receipt 周期
- **confirmations.fetchReceiptUponEntry**：入队立即查 receipt（降低延迟但增压）
- **confirmations.receiptWorkers**：receipt 并行 worker 数
- **block/notification queue length**：削峰缓冲，避免关键路径阻塞
- **transactions.nonceStateTimeout**：本地 nonce 状态“新鲜度”窗口（越小越保守）

---

### 10. 落地建议（实现顺序）
#### 10.1 最小可行落地（MVP）
- 引入 `signer_lease` 表（或等价存储）
- 实现 lease 抢占/续租 + fencing token 生成
- 将以下写入全部纳入“带 token 的条件更新”：
  - nonce 分配落库
  - resubmit 调度写入
  - 终局回写（状态推进/confirmations 覆盖）

#### 10.2 第二阶段（增强与优化）
- 路由层一致性哈希/网关路由优化（提升稳态命中）
- 引入幂等 requestID 双重检查（入口预查 + 写入前再查）
- 增强 submit 的错误分类（known tx / nonce too low 的幂等语义）
- 引入 confirmations 的 newFork 覆盖协议（如还未具备）

#### 10.3 观测与告警（上线必备）
- lease 抢占频率、续租失败率、token 跳变速率（异常提示网络分区/抖动）
- fencing 校验失败计数（应接近 0，出现说明存在双活/迟到写）
- signer 粒度吞吐、积压、receipt 查询延迟、终局延迟分布

---

### 11. 待确认问题（Review 需要拍板）
- **存储选型**：`signer_lease` 落库是用 DB 还是 Redis（或两者组合）？需要的原子语义/CAS 能否保证？
- **lease TTL 与续租周期**：过短会抖动，过长会延迟故障接管；需结合 SLO 与心跳机制定值
- **节点身份**：`owner_node` 如何生成与保证唯一（实例重启/滚动升级/多 AZ）？
- **关键写入边界**：现有代码里哪些写路径会影响 nonce/状态/终局？是否存在“漏加 fencing 校验”的风险点清单？
- **与现有 worker/队列模型集成方式**：是把 leader-only 写入集中到“writer 入口”，还是分散在各模块加 guard？

--- 

如果你愿意，我可以按你们现有工程结构，把“哪些写操作算关键写入、需要 fencing 校验”的清单也整理成一页 review checklist（方便评审逐条过）。
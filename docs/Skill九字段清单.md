# Skill 九字段清单

> 赛道要求：每个 Skill 说明 名称/用途/输入与输出/调用条件/依赖工具/失败处理/安全边界/复用价值/与多Agent协同流程的关系
> 用途：PPT P10 详细展开 db-jit-grant 或 db-hypo-index-validate（二选一贴页）；本清单为全量底稿
> 状态：核心 2 个完整展开，其余 6 个精简版，复赛补全

## 一、db-jit-grant ★（PPT 建议展开项）

| 字段 | 内容 |
|---|---|
| **名称** | db-jit-grant |
| **用途** | 审批通过后，为变更 Worker 发放"单实例+单动作"的临时执行权限，用完自动作废——JIT 权限的执行器 |
| **输入与输出** | 输入：动作定义（实例ID、动作类型、命令指纹）、分级（L1/L2）、有效期（默认≤900s）。输出：临时凭证句柄 + Higress 白名单登记记录；到期/撤销后输出撤销回执 |
| **调用条件** | 仅在复核闸门（Matrix 房间点头/批准）通过之后调用；L0 无需权限、L3 只出方案不执行，均不走本 Skill |
| **依赖工具** | RAM STS AssumeRole（DurationSeconds + Policy 收窄）、Higress Console API（allowedConsumers 增删）、AgentTeams 任务状态机（登记定时撤销任务） |
| **失败处理** | AssumeRole 失败→中止执行并升级人工；Higress 登记失败→不发放凭证（fail-closed，宁可不做不可失控）；撤销失败→告警并冻结该实例自动执行 |
| **安全边界** | 凭证有效期≤15 分钟；Policy 收窄到单实例+单动作；凭证不落 Agent 文件系统；全程默认拒绝（fail-closed） |
| **复用价值** | 不依赖数据库类型——任何"Agent 临时授权"场景（云资源变更、文件操作、其他有状态系统）均可复用；可沉淀为通用安全 Skill |
| **与协同流程的关系** | 变更 Worker 在闸门通过后调用 → 执行动作 → 撤销回执写入任务档案 → 知识流水线归档；Manager 监控撤销回执，缺失即告警 |

## 二、db-hypo-index-validate ★（PPT 备选展开项）

| 字段 | 内容 |
|---|---|
| **名称** | db-hypo-index-validate |
| **用途** | 不真实建索引，在验证实例上用 HypoPG 虚拟索引比对执行计划成本，产出"加索引是否有收益"的量化证据 |
| **输入与输出** | 输入：验证实例句柄、目标 SQL 列表、候选索引定义。输出：cost 对比表（建前/建后）、预估收益百分比、写放大评估、verdict（recommended / not_improved / needs_manual_review） |
| **调用条件** | 诊断验证 Worker 产出"索引嫌疑"结论且已有候选索引；或加索引工单进入预检阶段 |
| **依赖工具** | db-snapshot-verify（提供验证实例）、HypoPG 扩展、EXPLAIN |
| **失败处理** | HypoPG 不可用→降级为"验证实例真实建索引+流量回放"；EXPLAIN 报错→该条标记未定论转人工，不影响批次其他条目 |
| **安全边界** | 只在验证实例执行；生产库永久禁止安装 HypoPG 或做任何验证动作；生产零写入 |
| **复用价值** | 任何 PostgreSQL 索引优化的标准验证动作；证据格式统一（verdict + cost 对比），可被门禁直接消费；MySQL 场景以等价实现替换 |
| **与协同流程的关系** | 诊断验证 Worker 调用补齐证据 → 证据进入证据闸门布尔清单 → 达标后变更 Worker 作为选型依据 → 核验阶段复用同一证据做落地后比对 |

## 三、其余 Skill 精简版（复赛补全九字段）

| 名称 | 用途 | 关键安全边界 |
|---|---|---|
| db-alert-correlate | 多源告警聚合降噪 + 噪声短路，产出结构化工单 | 只读；噪声判定带置信度，低置信不短路 |
| db-six-domain-diagnose | 六域诊断引擎：seed calls + 假设-验证循环 + 冲突消解 | 只读；补查≤2 轮；空结果显式声明 |
| db-snapshot-verify | 备份拉起临时验证实例，TTL 自动销毁回收 | 生产零写入零 attach；到期强制回收 |
| db-hemostat | L1 止血：DAS 限流/kill 会话 + 定时撤销模拟 TTL | 白名单内；必须可逆限时；保命SQL比对 |
| db-change-risk-grade | 按锁范围/影响面/可逆性输出 L0-L3 分级 | 纯规则，无副作用；分级从严 |
| db-knowledge-writeback | 四元组回写 + 负样本 + 规则升华 | 仅数据层写权限；升华需人工复核 |

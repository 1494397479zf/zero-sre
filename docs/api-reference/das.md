# DAS（数据库自治服务）API 参考

> 产品代号：hdm · API 版本：2020-01-16 · 签名风格：RPC
> 来源：阿里云官方 API 概览（help.aliyun.com/zh/das/developer-reference/api-das-2020-01-16-overview）
> 整理日期：2026-08-06
> 调用约束：SDK 调用时地域需指定 **cn-shanghai**；权限点格式 `hdm:<API名>`

## 与 ZeroDBA 方案的关系总览

| 方案环节 | 关键 API | 说明 |
|---|---|---|
| 实例接入 | AddHDMInstance | 程序化接入 ECS 自建库到 DAS |
| 告警感知 | GetAutonomousNotifyEventsInRange | ML 异常检测事件（四级紧急度） |
| 止血 ★ | EnableSqlConcurrencyControl / CreateKillInstanceSessionTask / UpdateAutoThrottleRulesAsync | SQL 限流 + kill 会话 + 自动限流 |
| 诊断 | CreateRequestDiagnosis / DescribeQueryExplain / GetDeadLockDetailList / DescribeSlowLogRecords | SQL 诊断/执行计划/锁/慢日志 |
| 验证 ★ | RunCloudBenchTask / CreateCloudBenchTasks | 压测回放，可做故障演练与变更验证 |
| 复盘 | CreateDiagnosticReport / GetInstanceInspections | 诊断报告与巡检评分 |

## 1. 实例接入

| API | 用途 | 方案映射 |
|---|---|---|
| AddHDMInstance | 将数据库实例接入 DAS | Manager 初始化时自动接入 ★待验证自建库支持 |

## 2. 事件通知（异常检测）

| API | 用途 | 关键参数 |
|---|---|---|
| SetEventSubscription | 配置事件订阅 | InstanceId、事件类型 |
| GetEventSubscription | 获取事件订阅配置 | InstanceId |
| GetAutonomousNotifyEventsInRange | 按时间范围/紧急度查异常事件 ★ | StartTime/EndTime（毫秒时间戳）、MinLevel（Notice/Optimization/Warn/Critical）、InstanceId、PageNo/PageSize |
| GetAutonomousNotifyEventContent | 获取单个自治事件详情 | 事件 ID |

**方案映射**：db-oncall-manager 的告警源之一。ML 异常检测优于规则告警，事件自带紧急度分级，可直接映射到 L0–L3 风险分级。
**前提**：需开启自治中心。

## 3. SQL 限流 ★★ 止血核心

| API | 用途 | 风险级别 |
|---|---|---|
| EnableSqlConcurrencyControl | 启用 SQL 限流（控制访问量与并发量） | L1 可逆 |
| DisableSqlConcurrencyControl | 关闭指定限流规则 | L1 |
| DisableAllSqlConcurrencyControlRules | 关闭全部执行中的限流规则 | L1（紧急回滚用） |
| GetRunningSqlConcurrencyControlRules | 获取执行中的限流规则 | 只读 |
| GetSqlConcurrencyControlRulesHistory | 获取执行中或曾触发的规则 | 只读 |
| GetSqlConcurrencyControlKeywordsFromSqlText | 从 SQL 文本生成限流关键词 | 只读 |

**方案映射**：L1 止血第一手段。限流规则可随时关闭（DisableAll 提供紧急全量回滚），符合"可逆、见效快"的止血特征。
**关键设计**：止血用 EnableSqlConcurrencyControl，恢复验证通过后由 executor 调 Disable 撤销——形成"止血-撤销"闭环。

## 4. 实例会话 ★★ kill 会话

| API | 用途 | 风险级别 |
|---|---|---|
| GetMySQLAllSessionAsync | 异步获取实例当前会话（多维度统计） | 只读 |
| GetRedisAllSession | Redis 会话 | 只读 |
| GetMongoDBCurrentOp | MongoDB 会话 | 只读 |
| CreateKillInstanceSessionTask | 创建结束会话任务 ★ | L1 |
| GetKillInstanceSessionTaskResult | 获取结束会话任务结果 | 只读 |
| KillInstanceAllSession | 结束全部会话 ⚠️ | L2 高危 |

**方案映射**：kill 慢查询/阻塞源头会话是 L1 止血第二手段。任务式 API（Create+GetResult）天然适合 Agent 异步执行与结果验证。
**注意**：KillInstanceAllSession（全杀）风险高，应划入 L2 且需审批；常规止血用 CreateKillInstanceSessionTask 精确杀指定会话。

## 5. SQL 诊断

| API | 用途 | 方案映射 |
|---|---|---|
| CreateRequestDiagnosis | 发起 SQL 诊断请求 | diagnostician |
| GetRequestDiagnosisResult | 查询 SQL 诊断结果 | diagnostician |
| GetRequestDiagnosisPage | 分页获取诊断历史 | retrospector |
| DescribeQueryExplain | 查询 SQL 执行计划 ★ | 执行计划基线比对的云 API 版本 |

**方案映射**：DescribeQueryExplain 是"db-plan-baseline-diff"自研 Skill 的云侧能力基础——取当前执行计划与基线比对。

## 6. 锁分析

| API | 用途 | 备注 |
|---|---|---|
| GetDeadLockDetailList | SQL Server 死锁列表 | 仅 SQL Server |
| GetBlockingDetailList | SQL Server 锁阻塞列表 | 仅 SQL Server |
| CreateLatestDeadLockAnalysis | 创建最近死锁分析任务 | |
| GetDeadLockHistory | 死锁分析任务列表 | |
| GetDeadLockDetail | 单个死锁详情 | |
| GetDeadlockHistogram | 死锁数量趋势 | 基于错误日志全量分析 |

**注意**：MySQL 的锁等待分析在控制台能力里有（InnoDB 锁等待），但 API 清单中未见对应接口——⚠️ 待验证（可能需通过 DAS Agent Chat 或控制台）。

## 7. 慢日志

| API | 用途 |
|---|---|
| DescribeSlowLogHistogramAsync | 慢日志趋势（异步） |
| DescribeSlowLogStatistic | 慢日志统计 |
| DescribeSlowLogRecords | 慢日志记录查询（多条件过滤排序） |

## 8. 诊断报告与巡检

| API | 用途 | 方案映射 |
|---|---|---|
| CreateDiagnosticReport | 创建诊断报告 | diagnostician 综合输出 |
| DescribeDiagnosticReportList | 查询诊断报告 | retrospector 归档 |
| GetDBInstanceConnectivityDiagnosis | 网络连通性诊断 | 连接类故障 |
| GetInstanceInspections | 巡检评分结果 | 健康度基线 |

**注意**：文档标注诊断报告支持引擎为 RDS MySQL / PolarDB MySQL / Redis——⚠️ ECS 自建库是否支持待验证。

## 9. 空间分析

| API | 用途 |
|---|---|
| CreateStorageAnalysisTask | 创建空间分析任务 |
| GetStorageAnalysisResult | 获取分析结果 |
| GetAutoIncrementUsageStatistic | 表自增 ID 使用率 |

**方案映射**：磁盘满故障的根因定位（哪张表占空间）；自增 ID 耗尽预警。

## 10. 查询治理与 SQL 洞察

| API | 用途 |
|---|---|
| GetQueryOptimizeExecErrorStats | 执行失败的模板统计 |
| GetQueryOptimizeExecErrorSample | 执行失败样本 |
| GetQueryOptimizeSolution | 治理建议 |
| GetQueryOptimizeDataTrend | 治理趋势 |
| GetQueryOptimizeDataStats | 模板统计 |
| GetErrorRequestSample | 错误 SQL 样本（≤20 条） |
| GetAsyncErrorRequestStatResult | 指定 SQL 错误次数 |
| GetAsyncErrorRequestStatByCode | MySQL 错误码统计 |
| GetFullRequestStatResultByInstanceId | 全量请求统计 |
| GetFullRequestSampleByInstanceId | SQL 样本 |

## 11. 自动 SQL 限流 ★

| API | 用途 |
|---|---|
| UpdateAutoThrottleRulesAsync | 设置自动限流配置 |
| DisableAutoThrottleRules | 关闭自动限流 |
| GetAutoThrottleRules | 获取自动限流规则 |

**方案映射**：可配置为"常驻自动止血策略"——异常时 DAS 自动限流，Agent 只做事后确认与撤销决策。

## 12. 压测 ★★ 故障演练场执行器

| API | 用途 |
|---|---|
| RunCloudBenchTask | 执行压测任务 |
| CreateCloudBenchTasks | 创建压测任务 |
| DescribeCloudbenchTask | 查询压测任务 |
| DescribeCloudBenchTasks | 压测任务列表 |
| DescribeCloudbenchTaskConfig | 压测任务配置 |
| DeleteCloudBenchTask | 删除压测任务 |

**方案映射**：这是"故障演练场 + 变更验证"的官方执行器——在独立验证实例上用 CloudBench 回放生产流量，验证修复动作的效果与副作用。把"备节点验证"（技术上不成立）替换为"验证实例 + 流量回放"。

## 13. 错误日志与企业版

| API | 用途 |
|---|---|
| DescribeErrorLogRecords | 错误日志明细 |
| DescribeAuditLogs | 审计告警日志 |
| ModifySqlLogConfig / DescribeSqlLogConfig | DAS 企业版（SQL 洞察全量审计）配置 |
| GetDasSQLLogHotData | 审计日志热数据 |

**注意**：SQL 洞察全量审计需 DAS 企业版（付费）。

## 14. DAS Agent（大模型接口）

| API | 用途 | 权限点 |
|---|---|---|
| Chat | DAS Agent 自然语言交互（异步） | das:Chat |
| GetInstanceGroupInspectReportList | DAS Agent 运维报告列表 | |
| GetInstanceGroupInspectReportDetail | 运维报告明细 | |

**方案映射**：官方开源的 `alibabacloud-das-agent` Skill 即封装此 API。可作为 diagnostician 的"兜底大脑"——结构化 API 查不到的问题，用自然语言问 DAS Agent。
**费用**：有免费额度（默认 Agent ID），生产需购买订阅。
**注意**：实例需先在 DAS 控制台关联到 Agent（错误码 -1810006 = 未关联）。

## 15. 自动 SQL 优化 / 空间优化 / 弹性伸缩

| API | 用途 | 风险级别 |
|---|---|---|
| UpdateAutoSqlOptimizeStatus | 开关自动 SQL 优化 | 配置类 |
| GetSqlOptimizeAdvice | 自动优化诊断建议 | 只读 |
| UpdateAutoResourceOptimizeRulesAsync | 空间碎片自动回收配置 | 配置类 |
| ModifyAutoScalingConfig | 弹性伸缩配置 | L2（仅云数据库） |

## 待验证清单（自建库场景）

| # | 项目 | 验证方式 |
|---|---|---|
| 1 | AddHDMInstance 接入 ECS 自建 PG/MySQL 的参数与限制 | OpenAPI Explorer 实测 |
| 2 | 异常检测/诊断报告对自建实例是否生效 | 接入后触发异常观察 |
| 3 | kill 会话 API 对自建实例是否生效 | 控制台 + API 实测 |
| 4 | MySQL InnoDB 锁等待分析的 API 入口 | 控制台抓包 / 问 DAS 团队 |
| 5 | CloudBench 压测对自建实例的支持 | 实测 |
| 6 | DescribeQueryExplain 对自建实例的支持 | 实测 |

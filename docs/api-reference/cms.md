# 云监控 CMS API 参考

> 产品：云监控 · API 版本：2019-01-01
> 整理日期：2026-08-06
> 定位：ZeroDBA 的**指标/告警通道**之一

## 与方案的关系

| 方案环节 | 关键 API | 说明 |
|---|---|---|
| 告警接入 | DescribeAlertLogList / 报警规则 API | Manager 的告警源之一 |
| 指标查询 | DescribeMetricList / DescribeMetricData | diagnostician 查主机指标 |
| 数据库内部指标 ★ | PutCustomMetric + PutCustomMetricRule | 连接数/缓冲池等进程级指标靠自定义上报 |

## 1. 指标查询

| API | 用途 | 关键参数 |
|---|---|---|
| DescribeMetricList | 查询指定监控项的监控数据（批量） | Namespace（如 acs_ecs_dashboard）、MetricName（如 cpu_total）、Dimensions（实例维度）、StartTime/EndTime、Period |
| DescribeMetricData | 查询监控数据（聚合） | 同上 + 聚合方式 |
| DescribeMetricLast | 查询最新一条监控数据 | 同上 |

ECS 基础指标（CPU/内存/磁盘/网络）免安装自动采集，Namespace=acs_ecs_dashboard。

## 2. 自定义监控 ★（数据库内部指标的关键）

云监控只自动采集主机级指标。**数据库进程级指标**（连接数、QPS、缓冲池命中率、复制延迟、锁等待）需要自建采集脚本 + 自定义上报：

| API | 用途 |
|---|---|
| PutCustomMetric | 上报自定义监控数据 |
| PutCustomMetricRule | 创建自定义监控报警规则 |
| DescribeCustomMetricList | 查询已上报的自定义监控数据 |

**方案映射**：这是 CMS 侧最大的工作量——需要一个指标采集器（定时脚本，经云助手部署），把 `SHOW STATUS` / `pg_stat_*` 的关键指标转成自定义监控上报。上报后即可复用 CMS 的告警规则与可视化。
**替代选项**：数据库内部指标也可只进 SLS（查询更灵活），CMS 自定义监控的优势是能直接挂告警规则。两者可并存：采集器双写。

## 3. 报警规则管理

| API | 用途 |
|---|---|
| PutResourceMetricRule | 设置阈值报警规则（单条） |
| PutResourceMetricRules | 批量设置 |
| PutGroupMetricRule | 应用分组维度报警规则 |
| EnableMetricRules / DisableMetricRules | 启用/禁用规则 |
| DeleteMetricRules | 删除规则 |
| DescribeMetricRuleList | 查询规则列表 |

**方案映射**：报警规则可编程管理——Agent 可以根据实例重要性动态调整阈值（核心实例阈值更敏感），复盘后把误报规则自动调优。

## 4. 报警历史与事件

| API | 用途 |
|---|---|
| DescribeAlertLogList | 查询报警历史 |
| DescribeAlertLogCount | 统计报警历史 |
| DescribeAlertLogHistogram | 报警历史直方图 |

**方案映射**：Manager 的告警聚合输入之一；retrospector 复盘时拉取报警时间线。

## 5. 待验证

| # | 项目 | 验证方式 |
|---|---|---|
| 1 | 自定义监控上报频率与配额限制 | 实测 |
| 2 | 报警规则触发到回调/查询的延迟 | 实测（决定告警时效性） |

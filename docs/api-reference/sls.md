# SLS 日志服务 API 参考

> 产品：日志服务 SLS · API 版本：2020-12-30
> 整理日期：2026-08-06
> 定位：ZeroDBA 的**日志通道**——数据库日志/系统日志的采集、查询与分析

## 与方案的关系

| 方案环节 | 关键能力 | 说明 |
|---|---|---|
| 日志采集 | LoongCollector | 部署在 ECS 上采集数据库日志 |
| 日志查询 ★ | GetLogsV2 | diagnostician 查错误日志/慢日志上下文 |
| 日志告警 | SLS 告警 | 基于日志模式的告警（如 "ERROR replication"） |
| 复盘归档 | Logstore 保留策略 | retrospector 证据留存 |

## 1. 数据模型

```
Project（项目，与 ECS 同地域走内网采集）
└── Logstore（日志库：采集/存储/查询单元）
    └── 日志数据（带索引，支持 SQL 式查询）
```

**规划**：一个 Project（如 zerodba-{env}），按数据类型分 Logstore：`db-error-log`、`db-slow-log`、`sys-log`、`db-metrics`（指标也可进 SLS）。

## 2. 采集

| 组件/API | 用途 |
|---|---|
| LoongCollector ★ | 官方采集器，部署在 ECS 上，采集文本日志（数据库错误日志、系统日志） |
| PutLogs / PutLogsV2 | SDK 方式写入日志（自研采集器用） |

**采集对象规划**：
- PG：`postgresql.log` / `pg_log/`（错误 + 慢查询日志）
- MySQL：error log + slow query log
- 系统：`/var/log/messages`、dmesg（OOM 记录）
- **关键**：OOM kill 记录只在系统日志里——数据库被 OOM 杀掉的根因定位必须靠它

**约束**：Project 与 ECS 同地域可走内网采集（速度快、免公网流量费）。

## 3. 查询 ★

| API | 用途 | 备注 |
|---|---|---|
| GetLogsV2 ★ | 查询 Logstore 日志（返回压缩传输） | GetLogs 已废弃，用 V2 |
| GetHistograms | 查询日志分布直方图 | 先看分布再下钻 |
| GetIndex / UpdateIndex | 索引管理 | 查询前提：字段建索引 |

**查询语法**：`查询语句 | 分析语句`——前半段关键词/条件过滤，后半段标准 SQL 分析（GROUP BY / 聚合 / TOP N）。

**方案映射**：
- diagnostician 的典型查询：故障时间窗内的 ERROR 日志聚类（`level:ERROR | select pattern, count(*) group by pattern order by count desc`）
- 慢日志上下文分析：慢查询发生时刻前后的错误日志关联
- SQL 式分析让 LLM 可以生成查询语句而不是拉全量日志进上下文——**缓解上下文爆炸**

## 4. 告警与加工

| 能力 | 用途 |
|---|---|
| SLS 告警 | 基于查询结果的定时告警（如每 5 分钟查一次 ERROR 数量） |
| 定时 SQL | 定时聚合任务（如每小时慢查询统计） |
| 数据加工 | 日志脱敏/富化（⚠️ 敏感字段脱敏后再入库） |

**方案映射**：数据加工做**日志脱敏**——生产日志入库前把 SQL 字面量、业务数据脱敏，这既是合规要求也呼应仓库的脱敏红线。

## 5. 权限

RAM 权限点格式 `log:<API名>`，资源到 Project/Logstore 级（`acs:log:{region}:{account}:project/{project}/logstore/{logstore}`）——可以做到"诊断 Agent 只读日志、不能写"。

## 6. 待验证

| # | 项目 | 验证方式 |
|---|---|---|
| 1 | LoongCollector 对 PG/MySQL 日志轮转（log rotation）的跟踪 | 实测 |
| 2 | 大日志文件（慢日志突发暴增）的采集延迟 | 实测 |
| 3 | GetLogsV2 单次查询返回量上限与分页策略 | 实测 |

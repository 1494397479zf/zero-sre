# API 参考索引

> ZeroDBA 方案依赖的阿里云 API 全盘梳理，持久化供方案设计与实现参考。
> 每个文档按「API 清单 + 关键参数 + 方案映射 + 待验证项」组织。

## 文档清单

| 文档 | 产品 | 状态 | 说明 |
|---|---|---|---|
| [das.md](das.md) | 数据库自治服务 DAS（hdm） | ✅ 完成 | 诊断/止血/验证核心，含 90+ API |
| ecs-cloud-assistant.md | ECS 云助手 | 📋 待整理 | 执行通道：RunCommand 等 |
| cms.md | 云监控 CMS | 📋 待整理 | 指标/告警通道 |
| sls.md | 日志服务 SLS | 📋 待整理 | 日志采集与查询 |
| sts.md | RAM/STS | 📋 待整理 | 临时凭证与细粒度授权 |
| skills.md | 官方云 Skills | 📋 待整理 | aiops-skills 仓库结构与安装方式 |

## 通道分工总览

```
感知层：CMS（指标/告警）+ SLS（日志）+ DAS 异常事件（ML 检测）
诊断层：DAS 结构化 API（优先）+ DAS Agent Chat（兜底）
执行层：云助手 RunCommand（主机/数据库操作）+ DAS 止血 API（限流/kill 会话）
验证层：CloudBench 压测回放（验证实例）+ 恢复探针
权限层：Higress allowedConsumers（网关）+ RAM/STS（云凭证）双层 JIT
审计层：云助手执行记录（30 天）+ Matrix 消息 + ActionTrail
```

## 整理原则

1. **只收录方案会用到的 API**，不做全产品搬运
2. 每个 API 标注**方案映射**（哪个 Agent / 哪个环节用）和**风险级别**（对应 L0–L3 分级）
3. 对 ECS 自建库场景存疑的能力，一律标注 ⚠️ 并进入待验证清单
4. 参数详情以官方文档为准，本文档记录关键参数与调用约束（如 DAS 需 cn-shanghai 地域）

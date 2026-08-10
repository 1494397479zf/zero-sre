# ECS 云助手 API 参考

> 产品：云服务器 ECS · API 版本：2014-05-26
> 来源：help.aliyun.com/zh/ecs/developer-reference（RunCommand 等官方文档）
> 整理日期：2026-08-06
> 定位：ZeroDBA 的**执行通道**——Agent 对 ECS 上数据库的一切主机级操作都走云助手，不直接 SSH

## 与 ZeroDBA 方案的关系总览

| 方案环节 | 关键 API | 说明 |
|---|---|---|
| 前置检查 | DescribeCloudAssistantStatus | 执行前确认 Agent 在线 |
| 执行 ★ | RunCommand | 改参数/重启/DDL/备份等一切主机操作 |
| 预检 ★ | RunCommand + RepeatMode=DryRun | 只预检不执行，天然支持"执行前检查" |
| 文件下发 | SendFile | 配置文件/脚本/SQL 文件下发 |
| 结果查询 | DescribeInvocations / DescribeInvocationResults | 异步查结果 |
| 中止 | StopInvocation | 停止待执行/定时命令 |
| 审计 | 执行记录保留 30 天（上限 1 万条） | 天然审计 |

## 1. API 清单

| API | 用途 | 访问级别 |
|---|---|---|
| RunCommand ★ | 创建并执行命令（一步完成） | update |
| CreateCommand | 仅创建命令（不执行） | create |
| InvokeCommand | 执行已有命令 | update |
| SendFile | 上传文件到 ECS 实例 | update |
| StopInvocation | 停止待执行/定时执行的命令 | update |
| DescribeInvocations | 查询命令执行信息列表 | read |
| DescribeInvocationResults | 查询命令执行结果（含输出） | read |
| DescribeCloudAssistantStatus | 查询云助手 Agent 安装/在线状态 | read |
| InstallCloudAssistant | 安装云助手 Agent | update |

**权限点格式**：`ecs:RunCommand` 等，资源级授权到实例（`acs:ecs:{region}:{account}:instance/{instanceId}`），支持条件键 `ecs:CommandRunAs`（可限制以哪个用户执行）。

## 2. RunCommand 关键参数（方案相关）

| 参数 | 类型/取值 | 方案用法 |
|---|---|---|
| RegionId | string，必填 | 实例所在地域 |
| Type | RunShellScript / RunPowerShellScript / RunBatScript | Linux 用 RunShellScript |
| CommandContent | string，必填。明文或 Base64；编码后 ≤18KB（保留）/24KB（不保留） | 命令模板 |
| EnableParameter + Parameters | 自定义参数 {{name}}，≤20 个 | **命令模板化**：同一模板不同参数，如 kill 指定 pid、改指定参数 |
| InstanceId | array，1~100 | 批量执行（三节点滚动变更） |
| Timeout | 秒，默认 **60** | ⚠️ 数据库操作必须显式调大（备份/DDL） |
| Username | 默认 Linux=root | ★ **最小权限**：指定 mysql/postgres 等普通用户执行 |
| WorkingDir | 默认 /root | 指定数据目录 |
| RepeatMode | Once / Period / NextRebootOnly / EveryReboot / **DryRun** | ★ DryRun = 只预检不执行 |
| Frequency | rate(5m) / at(...) / Cron | 定时巡检、定期备份验证 |
| KeepCommand | bool，默认 false | 常用命令保留复用 |
| ClientToken | string | ★ **幂等控制**：防网络重试导致重复执行 |
| OssOutputDelivery | oss://bucket/prefix | 大输出投递 OSS，避免截断 |
| ContainerId / ContainerName | string | 容器化部署时在容器内执行 |
| TerminationMode | Process / ProcessTree | 超时终止时杀整个进程树 |
| ResourceTag | array | 按标签批量选实例（与 InstanceId 互斥） |

**返回**：CommandId + InvokeId（异步接口，用 InvokeId 查结果）。

**内置环境参数**（命令模板内可直接引用）：`{{ACS::InstanceId}}`、`{{ACS::InstanceName}}`、`{{ACS::InvokeId}}`、`{{ACS::RegionId}}`、`{{ACS::AccountId}}`。

## 3. 方案设计要点

### 3.1 执行用户最小权限 ★
`Username` 参数允许以普通用户执行。数据库操作应**禁止 root**，统一用数据库属主用户（mysql/postgres）执行；配合 RAM 条件键 `ecs:CommandRunAs` 在策略层强制。这直接回应"Agent 权限过大"的评审疑虑。

### 3.2 DryRun 预检 ★
RepeatMode=DryRun 只检查参数、实例环境、Agent 状态，不实际执行。可作为 executor 的**强制前置步骤**：任何 L2+ 动作先 DryRun，通过才真执行——把 AWS OODA 式 pre-flight check 落地为 API 能力。

### 3.3 命令模板化
EnableParameter=true 后命令内容用 `{{参数}}` 占位，执行时传键值对。意味着 Skill 可以沉淀为**参数化命令模板**（如"kill 会话模板"、"在线 DDL 模板"），Agent 只填参数不现写命令——大幅降低 LLM 生成错误命令的风险，也是 Skill"可复用"的具体形态。

### 3.4 幂等与审计
- ClientToken 保证重试不重复执行（止血动作尤其重要）
- 执行记录保留 30 天、上限 1 万条，DescribeInvocations 可查——审计证据白送
- 支持 EventBridge 事件订阅拿结果，Agent 不必轮询

### 3.5 文件下发
SendFile 可下发配置文件/脚本/SQL 文件到实例（前提：实例 Running + Agent 版本满足）。用于：下发 my.cnf 补丁、gh-ost 脚本、恢复探针脚本。

## 4. 约束与限制

| 约束 | 值 | 影响 |
|---|---|---|
| 命令内容大小 | Base64 后 ≤18KB（保留）/24KB（不保留） | 大脚本需拆分或 SendFile 下发 |
| 批量实例数 | ≤100/次 | 三节点场景无压力 |
| 命令保有量 | 单地域 500~50000 条 | 用 KeepCommand=false 控制 |
| 超时 | 10~86400 秒，默认 60 | ⚠️ 超过 24h 的操作（超大表 DDL）需拆分 |
| Agent 版本 | 定时任务新特性需 Linux ≥2.2.3.282；容器执行需 ≥2.2.3.344 | 新实例默认预装（2017-12 后公共镜像） |
| 实例状态 | 必须 Running | 宕机实例无法执行——故障场景注意 |

**最后一条是盲区**：数据库把机器拖死（OOM/夯机）时云助手也执行不了，此时止血只能走 DAS API（限流/kill 会话，不依赖主机 Agent 存活度）或重启实例（ECS RebootInstance）。方案中止血优先级应为：**DAS API > 云助手 > 实例级操作**。

## 5. 待验证清单

| # | 项目 | 验证方式 |
|---|---|---|
| 1 | Username 指定 mysql/postgres 用户执行的权限边界 | 实测 |
| 2 | DryRun 预检的检查项覆盖范围 | 实测 |
| 3 | OssOutputDelivery 的授权配置 | 实测 |
| 4 | 实例夯死（load 极高）时云助手的可用性 | 压测模拟 |

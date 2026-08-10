# RAM / STS API 参考

> 产品：访问控制 RAM / 安全令牌服务 STS · STS API 版本：2015-04-01
> 整理日期：2026-08-06
> 定位：ZeroDBA 的**权限通道**——JIT 权限的云侧底座（与 Higress 网关层组成双层 JIT）

## 与方案的关系

```
双层 JIT 权限模型：
┌─────────────────────────────────────────┐
│ Higress 网关层：哪个 Agent 能调哪个工具      │
│ （allowedConsumers，秒级增删，管工具路由）   │
├─────────────────────────────────────────┤
│ RAM/STS 云凭证层：调用时持什么权限、多久过期  │
│ （AssumeRole 临时凭证，管云 API 权限范围）   │
└─────────────────────────────────────────┘
```

## 1. 核心 API

| API | 用途 | 关键参数 |
|---|---|---|
| AssumeRole ★ | 扮演 RAM 角色获取临时凭证（STS Token） | RoleArn、RoleSessionName、DurationSeconds（有效期）、Policy（可再收窄权限） |
| AssumeRoleWithOIDC | OIDC 联合身份扮演 | 对接外部身份系统 |
| AssumeRoleWithSAML | SAML 联合身份扮演 | 企业 SSO |

**STS Token 三要素**：AccessKeyId + AccessKeySecret + SecurityToken，带过期时间。

**关键能力**：AssumeRole 时可传 `Policy` 参数**在角色权限基础上进一步收窄**——即同一个角色，不同场景扮演时给不同范围。这是 JIT 的精髓：
> executor 执行"改实例 A 的参数"时，AssumeRole 附带 Policy 限定 `Resource=instance/A`——凭证权限精确到单次动作。

## 2. 角色与策略设计（方案规划）

**一个 Agent 一个 RAM 角色**（最小权限）：

| 角色 | 权限范围 | 对应 Agent |
|---|---|---|
| zerodba-readonly | hdm:Get*/Describe*、cms:Describe*、log:Get*、ecs:DescribeInvocations | diagnostician |
| zerodba-hemostat | hdm:EnableSqlConcurrencyControl、hdm:CreateKillInstanceSessionTask、hdm:Disable* | executor（止血模式） |
| zerodba-operator | ecs:RunCommand（限指定实例 + CommandRunAs 条件键）、hdm 变更类 | executor（变更模式） |
| zerodba-manager | 只读 + 事件订阅 | manager |

**条件键的妙用**：
- `ecs:CommandRunAs`：限制云助手命令只能以 mysql/postgres 用户执行，策略层禁止 root
- Resource 级授权：角色权限限定到指定实例 ID，越权调用直接被拒

## 3. JIT 权限流程（方案设计）

```
L2/L3 变更审批通过
  → Manager 调 AssumeRole（Policy 收窄到目标实例+动作）
  → 获得短时效 STS Token（如 900 秒）
  → 注入 executor 的执行环境（经 Higress 网关）
  → executor 执行变更
  → Token 到期自动失效（无需主动回收）
  → ActionTrail 记录全程调用
```

**与 Higress 的分工**：Higress 管"哪个 Agent 能访问哪个工具入口"（网络层，秒级吊销），STS 管"这次调用持什么权限"（凭证层，到期自动失效）。两层独立失效，任一层出问题不影响另一层的防护。

## 4. 审计

| 产品 | 用途 |
|---|---|
| ActionTrail（操作审计） | 记录所有云 API 调用（谁、何时、调了什么、参数） |
| RoleSessionName | AssumeRole 时传入的会话名——**用来标记"哪次故障处置的哪一步"**，审计可溯源到具体工单 |

**方案映射**：RoleSessionName 用 `zerodba-{task-id}-{agent}-{step}` 格式，把云 API 审计与 Matrix 房间的处置记录关联起来——形成完整证据链。

## 5. 待验证

| # | 项目 | 验证方式 |
|---|---|---|
| 1 | AssumeRole 附带 Policy 收窄的生效范围与限制 | 实测 |
| 2 | STS Token 最短有效期（秒级吊销的替代方案） | 实测 |
| 3 | Higress 网关注入 STS 凭证的集成方式 | 设计验证 |

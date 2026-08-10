# 官方云 Skills 参考

> 来源：github.com/aliyun/alibabacloud-aiops-skills（主仓库）、alibabacloud-ecs-troubleshoot-skills、skills.aliyun.com 门户
> 整理日期：2026-08-06
> 定位：赛道要求的"阿里云官方用云 Skills"——可直接安装使用，也可作为自研 Skill 的结构范本

## 1. 仓库与目录结构

**主仓库 `aliyun/alibabacloud-aiops-skills`**，按云产品分类：

```
skills/
├── database/          ← 数据库（重点）
│   ├── hdm/alibabacloud-das-agent/      ★ DAS 自然语言诊断
│   ├── rds/alibabacloud-rds-copilot/    RDS AI 助手
│   ├── rds/alibabacloud-rds-instances-manage/
│   ├── polardb/ mongodb/ kvstore/ dms/ dts/ drds/ clickhouse/ adb/ gpdb/ hitsdb/
│   └── solutions/
├── computing/         ← ECS 等
├── entcmc/            ← 云监控等
├── migrationom/       ← 迁移与运维管理
├── security/ container/ storage/ netcdn/ middleware/ ...
└── playbooks/
```

**单个 Skill 的标准结构**（自研 Skill 照抄此范式）：

```
{skill-name}/
├── SKILL.md                    # 能力定义：name/description/license/compatibility/metadata
├── scripts/                    # 可执行脚本（如 call_das_agent.py）
└── references/
    ├── api-reference.md        # API 签名与事件文档
    ├── related-apis.md         # 相关 API 清单
    ├── ram-policies.md         # 所需 RAM 权限
    ├── cli-installation-guide.md
    ├── troubleshooting-workflow.md
    ├── verification-method.md  # 验证方法
    └── acceptance-criteria.md  # 验收标准
```

## 2. ZeroDBA 可用的官方 Skills

| Skill | 仓库 | 用途 | 依赖/权限 |
|---|---|---|---|
| alibabacloud-das-agent ★★ | aiops-skills/database/hdm | 自然语言诊断数据库（高 CPU/慢查询/连接异常/锁等待/磁盘/健康巡检/安全基线），支持 kill 会话 | uv + AliyunHDMFullAccess + das:Chat；有免费额度 |
| alibabacloud-rds-copilot | aiops-skills/database/rds | RDS AI 助手（SQL 优化/诊断/性能分析） | RDS 相关权限 |
| alibabacloud-rds-instances-manage | aiops-skills/database/rds | RDS 实例管理操作 | RDS 相关权限 |
| alibabacloud-ecs-troubleshoot-skills ★ | 独立仓库 aliyun/alibabacloud-ecs-troubleshoot-skills | ECS Linux 排查（SSH/网络/磁盘/性能/宕机夯机/时钟）+ 安全检测（51 分析器/103+ ATT&CK 映射）+ 内核 CVE 检测 | aliyun CLI + 只读/诊断权限 |
| alibabacloud-cms-manage | 门户（迁移与运维管理分类） | 云监控告警管理 | CMS 权限 |

**注意**：das-agent 与 rds 系 Skills 的官方支持范围是 RDS/PolarDB 等云数据库——ECS 自建库场景需实测（见 das.md 待验证清单）。ECS 排查 Skills 走 aliyun CLI + OpenAPI，对自建库无此限制。

## 3. 安装方式

```
# 方式一：skills.sh CLI（门户推荐）
npx skills add aliyun/alibabacloud-aiops-skills --skill alibabacloud-find-skills --full-depth

# 方式二：手动安装
git clone https://github.com/aliyun/alibabacloud-aiops-skills.git
mkdir -p ~/.agents/skills
cp -r alibabacloud-aiops-skills/skills/database/hdm/alibabacloud-das-agent ~/.agents/skills/
```

**AgentTeams 集成**：Worker CR 的 `skills` 字段挂载平台内置技能；自定义 Skill 可通过 `package` 字段（file/http/nacos 等 URI）下发到 Worker 的技能目录。

**alibabacloud-find-skills**：技能发现 Skill——让 Agent 自己搜索并安装所需技能，可在 Manager 上挂载。

## 4. 对方案的意义

1. **das-agent 是 diagnostician 的兜底大脑**：结构化 API 查不出的问题用自然语言问 DAS Agent
2. **ecs-troubleshoot 覆盖主机层故障**：数据库问题有时根因在主机（网络丢包/磁盘异常/夯机），此 Skill 补上这一层
3. **Skill 结构即规范**：references/ 下的 ram-policies.md、verification-method.md、acceptance-criteria.md 正好对应赛道评审对 Skill"安全边界、验证方式、失败处理"的核验字段——自研 Skill 直接套用此结构，评审对格式零学习成本
4. **开源分发路径现成**：自研的数据库运维 Skills 可以按同样结构开源，甚至提 PR 进官方仓库（开源贡献分）

## 5. 待办

| # | 项目 |
|---|---|
| 1 | 逐一核对门户「迁移与运维管理」分类的 11 个 Skills（门户 JS 渲染，需浏览器逐个查看） |
| 2 | 内网 aone skills 市场的 db-troubleshooting：获取后分析能力边界（公网渠道已确认不存在） |
| 3 | das-agent 在 AgentTeams Worker 内的安装实测（uv 依赖） |

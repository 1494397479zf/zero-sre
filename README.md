# ZeroDBA — 数据库零人工运维多 Agent 系统

> Agent Infra 赛道 · 方向一「零人工运维」· 数据库域
> GOAI 世界人工智能开源大赛参赛项目

## 项目简介

面向数据库运维场景的多 Agent 协同系统。以 **AgentTeams** 为协同底座、**Higress** 为安全网关、**阿里云 Skills** 为工具层，构建「告警聚合 → 根因定位 → 预案生成 → 备节点验证 → 生产执行 → 恢复验证 → 复盘沉淀」的端到端零人工运维闭环。

核心设计：**备节点先验 + JIT 权限 + 分级授权** 三重保险，回答"敢让 AI 动生产数据库"。

详细方案见 [docs/方案骨架.md](docs/方案骨架.md)。

## 仓库结构

```
zero_sre/
├── README.md                    # 本文件
├── docs/
│   ├── 方案骨架.md              # 初赛方案骨架
│   ├── 赛题要求.md              # 赛道要求摘要（待补）
│   ├── agentteams-mapping.md   # AgentTeams 映射详表（待补）
│   ├── skills/                 # Skill 清单（待补）
│   └── ppt/                    # PPT 底稿（待补）
├── samples/                    # 脱敏样例（告警/慢SQL，需脱敏后入库）
└── .gitignore
```

## 本地环境

- AgentTeams v1.2.0（embedded 模式，Docker）
  - Element Web: http://127.0.0.1:18088
  - Dashboard: http://127.0.0.1:13000
  - Higress 控制台: http://127.0.0.1:18001
- 数据库：阿里云 ECS 部署 PG/MySQL 三节点（规划中）

## Git 协作规范

- `main` 分支已保护，必须走 PR 合并（仅 squash merge）
- 分支命名：`feat/xxx` / `docs/xxx` / `fix/xxx`
- 合并后自动删除分支

## 脱敏红线（public 仓库，务必遵守）

- 不提交未脱敏的生产信息：实例名、IP、连接串、真实慢 SQL、告警原文
- 脱敏后的样例放 `samples/` 并显式 `git add`
- 文件一旦被提交，删除也无法从 git 历史中移除

## 许可证

待定（建议 Apache 2.0，与 AgentTeams 保持一致）

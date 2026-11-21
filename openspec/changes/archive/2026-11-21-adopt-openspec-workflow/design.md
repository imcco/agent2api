# Design

## 目标

把 OpenSpec 工作流落进本仓库，同时不扰乱现有开发纪律（release-flow、coding-conventions、六语文案同步）。

## 决策

- **方案**：用 OpenSpec 官方 CLI 初始化 + 手工补 `project.md`，不在仓库里自研一套规格目录。
- **不选**：直接复制别家规格目录格式；或与现有 Skill 并行维护两份事实。理由：OpenSpec CLI 能校验与归档，且 `openspec init` 已生成 OpenCode 集成，省去自己接线。
- **取舍**：`openspec/` 与 `.opencode/` 一起入库。`project.md` 是规格工作流依赖的上下文；`.opencode/` 让其他机器上同一仓库的 OpenCode 会话直接获得 `/opsx-*` 命令。不想入库可后续在 `.gitignore` 里例外。

## 结构

```text
openspec/
├── config.yaml            # schema + zh-CN + 自定义规则位
├── project.md             # 项目上下文（代码边界、验证命令、运行面）
├── specs/                 # 主规格库（只由 archive 更新）
└── changes/
    ├── archive/           # 已归档 change
    └── <change-id>/       # proposal.md / specs/ / design.md / tasks.md
.opencode/
├── skills/openspec-*/     # OpenSpec 生成的技能
└── commands/opsx-*.md     # OpenSpec 生成的命令
```

## 与现有规范的关系

- 规格内容仍遵守 `coding-conventions`：中文文案、英文标识符、最小改动。
- 发版仍走 `release-flow`；OpenSpec 只管「需求→实现」，不管版本号、tag、Release。
- AGENTS.md 只加一节指路，细则不重复写，避免两处漂移。

## 风险与回滚

- 风险：团队机器没装 `openspec` CLI。缓解：README/AGENTS 写明安装方式；不装 CLI 也能手动读写目录，只是不能校验/归档。
- 回滚：删除 `openspec/`、`.opencode/`、AGENTS.md 新增段落即可，无源码影响。

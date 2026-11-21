# Proposal

## Why

仓库目前靠 `AGENTS.md` + 若干 Skill 约束开发行为，需求与实现契约散落在会话与 Skill 里；新增功能或修 bug 时容易直接改代码，缺少先对齐规格再实现的轻量流程。需要在本项目引入 OpenSpec 的 spec-driven 工作流，把「要做什么」沉淀成可审的规格，再按任务实现，后续功能与 bug 修复都走这条路。

## What Changes

- 在项目根新增 `openspec/` 目录：保存 `config.yaml`、`project.md`、`specs/` 主规格库、`changes/` 变更提案。
- 用 `openspec init` 生成 OpenCode 集成（`.opencode/skills/openspec-*`、`.opencode/commands/opsx-*`），让 OpenCode 会话直接以命令/技能方式驱动提案、实现、归档。
- 新建 `spec-authoring` 能力的主规格，定义规格怎么写、变更怎么落地。
- `AGENTS.md` 增补「OpenSpec 工作流」一节，指向 `openspec/project.md` 与使用入口；不改动任何源码。

## Capabilities

### New Capabilities
- `spec-authoring`: 管理本仓库 OpenSpec 规格与变更的编写规则：主规格存放、变更提案结构、提案-实现-归档的流转约定。

### Modified Capabilities

（无）

## Impact

- 新增：`openspec/**`、`.opencode/**`（OpenSpec 生成的技能与命令，团队共享时一起入库）。
- 文档：`AGENTS.md` 新增一节。
- 依赖：开发机需 `openspec` CLI（brew/npm 安装，Node ≥ 22.20）；运行时不引入新依赖。
- 不影响：网关代码、面板、插件、CI 工作流。

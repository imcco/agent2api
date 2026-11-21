# spec-authoring Specification

## Purpose
定义本仓库如何用 OpenSpec 管理需求规格与变更：规格写在哪里、一次改动要交哪些产物、什么时候允许实现、什么时候归档。

## Requirements

### Requirement: 项目初始化 OpenSpec 目录结构
仓库根必须包含 `openspec/` 目录，内有 `config.yaml`（schema: spec-driven，语言 zh-CN）、`project.md`（项目上下文）、`specs/` 主规格库、`changes/` 变更提案。

#### Scenario: 新项目成员初始化
- **WHEN** 在本仓库首次运行 `openspec init --tools opencode --language zh-CN`
- **THEN** 生成上述目录与 OpenCode 集成文件，且不修改任何源码

### Requirement: 变更按提案-规格-设计-任务流程推进
每个新功能或非平凡 bug 修复必须先创建 change（`proposal.md`、`specs/<capability>/spec.md`、`design.md`、`tasks.md`），再实现；文档类、配置类、纯重构等无规格行为的改动可在 change 的 `.openspec.yaml` 标记 `skip_specs: true`。

#### Scenario: 新增功能时先对齐规格
- **WHEN** 要新增对外可见行为（新接口、新能力、行为变更）
- **THEN** 先创建对应 change，主规格以 `openspec/specs/<capability>/spec.md` 的增量描述，审阅通过后再按 `tasks.md` 实现

#### Scenario: 文档或纯重构改动
- **WHEN** 改动只影响文档、目录结构或内部实现，不改变外部行为
- **THEN** 该 change 的 `.openspec.yaml` 设置 `skip_specs: true`，不生成 specs 目录

### Requirement: 主规格随归档同步
`openspec/specs/` 下的主规格只能由已完成的 change 通过 `openspec archive` 同步；禁止手工直接编辑主规格文件来新增行为。

#### Scenario: 变更完成后归档
- **WHEN** 某 change 的 tasks 全部完成且验证通过
- **THEN** 运行 `openspec archive <change>`，把增量规格合并进 `openspec/specs/`，并把 change 移到 `openspec/changes/archive/`

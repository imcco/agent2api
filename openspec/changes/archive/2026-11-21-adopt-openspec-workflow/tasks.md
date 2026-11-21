# Tasks

- [x] 1. 用 `openspec init --tools opencode --language zh-CN --force` 初始化，生成 `openspec/` 与 `.opencode/` 集成。
- [x] 2. 编写 `openspec/project.md`：代码边界、运行面、验证命令、已修复未合入项。
- [x] 3. 创建 change `adopt-openspec-workflow`，产出 proposal、specs、design、tasks 四件套。
- [x] 4. `AGENTS.md` 增补「OpenSpec 工作流」一节，指向 `openspec/project.md` 与使用入口。
- [x] 5. `openspec validate adopt-openspec-workflow --strict` 通过；`openspec list --specs` 能看到新能力。
- [x] 6. 把 `openspec/`、`.opencode/`、AGENTS.md 新增段落一起提交到本地 beta 分支；先不推 origin。
- [x] 7. 归档：`openspec archive adopt-openspec-workflow`，确认主规格落到 `openspec/specs/spec-authoring/spec.md`。

## 验收

- `openspec validate adopt-openspec-workflow --strict` 输出 0 错误。
- `openspec list --specs` 列出 `spec-authoring`。
- `openspec list --changes` 归档后为空（只剩 archive）。
- `git status --porcelain openspec .opencode AGENTS.md` 在提交后干净。

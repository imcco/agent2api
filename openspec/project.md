# Agent2API 项目上下文

> 本文件供 OpenSpec 工作流使用：任何提案、规格、任务都基于这里的事实。保持更新，改动关键事实时同步修改。

## 一句话

本机运行的多提供商 AI 网关（Tauri 桌面端 + Rust 网关 + WASM 提供商插件），把 Loomy / Trae / Raccoon / WorkBuddy / Antigravity 等账号的能力聚合成 OpenAI 兼容的 `http://127.0.0.1:3065/v1` 接口，并带面板、签到、模型目录、容量控制。

## 代码边界

| 位置 | 作用 | 语言 |
|---|---|---|
| `desktop-tauri/src-tauri/server/` | 网关本体 crate `agent2api-server`：HTTP API、路由、插件宿主、容量控制、SQLite 存储 | Rust |
| `desktop-tauri/src-tauri/` | Tauri 壳 `workbuddy-proxy-desktop` | Rust |
| `desktop-tauri/ui/` | 面板前端（纯静态资源，不经打包） | JS/HTML/CSS |
| `desktop-tauri/ui-*`、`mobile/`、`i18n/` | 前端模块、移动端、六语文案 | TypeScript/JS |
| 提供商插件 | 独立仓库 `ProvidersPlugin/Agent2Providers`，按 git tag 引用，本仓库不含插件源码 | — |

## 本机运行面

- 配置与数据：`~/.agent2api/`（SQLite `agent2api.db`，账号、Key、请求、日志都在里面）。
- 安装路径：`/Applications/Agent2API.app`，LaunchAgent `~/Library/LaunchAgents/Agent2API.plist` 自启。
- 面板 API 鉴权：`kv.apiKeys` 里的 `enc1:` AES-256-GCM 信封，用 `~/.agent2api/account.key` 解。

## 编码与协作约定

- 动手前先读 `.agents/skills/coding-conventions/SKILL.md`，Rust 细则看 `references/rust.md`，前端看 `references/frontend.md`。
- 分支、版本号、发版一律按 `.agents/skills/release-flow/SKILL.md`。
- 提交信息中文、祈使句；改动最小化；测试测真实实现，不 Mock 出假绿。
- 六语文案改一份，其余五份同步。

## 验证方式

- Rust：`CARGO_TARGET_DIR=<仓库>/target cargo test -p agent2api-server <过滤>`。
- 桌面构建：`cd desktop-tauri && npx tauri build --config src-tauri/tauri.macos.conf.json --bundles app`（release 约 8–10 分钟，产物在仓库根 `target/release/bundle/macos/Agent2API.app`）。
- 运行验证优先打本地网关 HTTP（`/v1/models`、`/v1/chat/completions`），配合 `~/.agent2api/agent2api.db` 的 `logs` / `requests` / `maintenance_runs` 表取证。

## 已修复但上游未合入

- `fix/macos-memory-pressure`（d9ee142b + fce2d6d0）：macOS 内存压力误判，读 `kern.memorystatus_vm_pressure_level`，只在 level ≥ 4 判紧张。PR #627 → `aimod-cc/Agent2API:beta` 仍 OPEN。
- 本机跑的是基于 v3.5.0-beta.4 的本地修复版；官方发 beta.5 后先确认是否含该修复再升级。

---
name: coding-conventions
description: Agent2API（workbuddy）仓库的编程语言规范入口 —— Rust（desktop-tauri/src-tauri 桌面壳 + server 网关 crate）与 TypeScript/React（ui-islands / ui-kit / ui-shared / mobile / i18n）的目录落点、通用约定、入库产物纪律与提交前检查；语言细则见 references/rust.md 与 references/frontend.md。凡是新增或修改 Rust / TS 源码、新建模块或组件、加接口或文案、写测试、review 本仓库代码、或判断「这样写合不合本项目规矩」时，都先读本文件，再按改动语言读对应 references。
---

# Agent2API 代码规范（入口）

本 Skill 只写**这个仓库里真实存在**的做法，并给出证据路径；不写通用语言教程。
判断「该照谁写」的方法永远是：**找同层同域最近的文件，照它的形状写**，再回来对照本文。

| 文件 | 什么时候读 |
|---|---|
| 本文件 | 每次动代码前：落点、通用约定、产物纪律、验收命令 |
| [`references/rust.md`](references/rust.md) | 改 `desktop-tauri/src-tauri/**` 下任何 `.rs` |
| [`references/frontend.md`](references/frontend.md) | 改 `desktop-tauri/ui-islands`、`ui-kit`、`ui-shared`、`mobile`、`i18n` |

工程协作规则（怎么分工、怎么验收、什么时候能推送）由本机的 `project-rules` skill 管，**它不随仓库发布**；本文只写仓库里可验证的代码事实。发版见 `release-flow`；提供商接入见 `provider-adaptation`。

## 0. 硬事实先记住

- 仓库**没有** `rustfmt.toml` / `clippy.toml` / ESLint / Biome / Prettier —— 格式与 lint 都不是 CI 硬门槛，靠本文约定与 review。别提议「加个 lint 就解决了」。
- CI 只有三道闸：`rust-tests.yml`（测试 + 插件联调 + beta 覆盖率）、`frontend-artifacts.yml`（前端产物逐字节同源）、`build.yml`（只派发构建仓库，不编译）。
- 注释、日志、错误文案、用户可见文案一律**中文**；标识符一律英文。
- 本仓库大量刻意保留的「与 Node 版对齐」行为：**看不懂某个奇怪分支时，先找注释里的 `server.mjs` 行号**，那是原始依据，不要「顺手修正」。

## 1. 代码地图与落点

| 位置（相对仓库根） | 是什么 | 新代码落点 |
|---|---|---|
| `desktop-tauri/src-tauri/src/` | Tauri 桌面壳：窗口、托盘、command、window 桥、登录、更新 | 系统交互、Tauri command、桥接口 |
| `desktop-tauri/src-tauri/server/src/server/api/` | HTTP handler，**一域一文件**（`accounts.rs`、`chat.rs`、`plugin_login.rs`…） | 新增/修改 `/api`、`/v1` 接口 |
| `.../server/src/server/core/` | 域服务：`account_store/`、`billing/`、`providers/`、`upstream/`、`plugin_host/`、`egress.rs`、`auto_checkin.rs` | 业务逻辑 |
| `.../server/src/server/db/` | `mod.rs` 连接句柄、`schema.rs` 全量 DDL、`migrate/` 旧文件迁移 | 表结构、SQL |
| `.../server/src/server/{errors,http,logging}.rs` | 错误类型、路由组装、日志 | 横切改动 |
| `desktop-tauri/ui-islands/src/` | React 页面岛 + 壳（`islands/<域>-*.tsx` 一域多文件） | 桌面界面 |
| `desktop-tauri/ui-kit/` | 私有组件库 `@agent2api/ui`（源码随仓库，不走 npm） | 通用控件 |
| `desktop-tauri/ui-shared/` | 无 `package.json` 的纯源码，桌面与手机共享逻辑/面板 | 跨端复用 |
| `desktop-tauri/mobile/` | PWA / Capacitor 手机端 | 手机端界面 |
| `desktop-tauri/i18n/` | 词典扫描（`scan.mjs`）与编译（`build.mjs`） | 文案与词典 |
| `desktop-tauri/ui/` | **入库的构建产物**（Tauri 与 Docker 直接取这些字节） | **禁止手改**，见第 4 节 |

Rust 是 workspace 两个 crate：桌面壳 `workbuddy-proxy-desktop`（`src/`）与网关 `agent2api-server`（`server/`），桌面进程内跑网关、headless 二进制链接同一个 lib。提供商（provider）实现在另一个仓库 `ProvidersPlugin/Agent2Providers`，本仓按 **git tag** 引用协议库 —— 换协议版本走 `release-flow` / `provider-adaptation`，别在本仓里改插件实现。

## 2. 通用约定

### 2.1 注释解释「为什么」，且模块头必写

这是本仓库最显眼的规范：**文件/模块开头一段中文说明它解决什么问题、为什么这样分层、踩过什么坑**，长度常常 20–140 行。

- 分节线：`// ── 标题 ────────────────────────`（Rust）、`/* ─── 标题 ─── */`（TS 代码内分段）。仓库里成百上千处，别人一眼能扫到结构。
- 标准题材是「为什么 X 而不是 Y」：`db/mod.rs` 里「为什么单连接不用池、为什么 Mutex 不用 RwLock」、`Cargo.toml` 里「为什么选 rusqlite 而不是 sqlx / sea-orm」、`assistant-api.ts` 里「为什么绕开桥直连 `/api/*`」。
- 事故记录写进注释：`server/src/server/errors.rs` 里记录了「中文错误文案上切片 panic」这类事故的修复依据。
- **不写 TODO/FIXME 当挡箭牌**（全前端仅 1 处、还是引用后端 TODO）。没做完的事写在 commit / issue / 计划里，不埋在代码里。
- 公共 API 用 `///`（Rust）/ JSDoc（TS），写约束与理由，不复述签名。

### 2.2 命名总则

- Rust 细则（服务后缀、`serde`、command、事件名）见 [`references/rust.md`](references/rust.md) §2；TS 细则（类型、文件、组件）见 [`references/frontend.md`](references/frontend.md) §3。
- 跨语言的硬要求只有三条：**标识符英文、文案中文、对外 JSON 用 camelCase**（Rust 侧 `#[serde(rename_all = "camelCase")]`，内部枚举除外）。
- 新名字先在同域搜索：本仓库偏好「描述事实」的名字（`quota_reset_at`、`plugin_busy`、`port_occupant`），不用缩写和拼音。

### 2.3 可见性默认收窄

Rust 里 `pub(crate)` 是常态，`pub` 只给真正跨 crate 的面；TS 里组件/函数**具名导出**，除极少数页面岛外不用 `export default`。

## 3. 语言细则

- **Rust** → [`references/rust.md`](references/rust.md)：分层与文件形状、`GatewayError`、`spawn_task` / 锁 / 出网约束、SQLite 单连接与迁移、测试放哪、依赖与平台 cfg、风格硬事实。
- **TypeScript / React** → [`references/frontend.md`](references/frontend.md)：技术栈实况、包职责、组件与状态写法、桥与直连的边界、Tailwind 令牌、i18n「中文即键」、脚本与工具链。

## 4. 入库产物纪律（最容易踩）

**禁止手改**：`desktop-tauri/ui/islands/ui.js`、`ui/islands/ui.css`、`ui/islands/login.js`、`ui/i18n/**`、`ui/m/**`、`ui/sw.js`。它们是源码产物，Tauri 打包与 Docker 镜像直接取这些字节。

改了源码就重建（`.githooks/pre-commit` 也会自动做，失败即中止提交）：

| 改了什么 | 必须重建 |
|---|---|
| `ui-islands/src/**`、`ui-kit/src/**`、`ui-kit/styles/**` | `npm --prefix desktop-tauri/ui-islands run build` |
| `mobile/src/**` | `npm --prefix desktop-tauri/mobile run build` |
| `i18n/i18n-src/**` | `npm --prefix desktop-tauri/i18n run build` |

- 重建后 `git status --porcelain -- desktop-tauri/ui` 必须为空 —— CI 的判据就是「产物能由源码逐字节复现」，不是语义等价。
- 首次构建前先 `npm ci` 四个包（**包括 `ui-kit`**），版本漂移会导致产物不同源。
- 手机端原生工程（`mobile/android/**`、`mobile/ios/**`）不参与 Web 产物，改了不用重建。
- 已知过时注释：`frontend-artifacts.yml` 里说「仓库没有 .gitattributes」，现在有（只约束 `.githooks/**` 为 LF）—— 以现状为准，别照着改回 CRLF。

## 5. 提交前检查（精确命令）

```sh
# Rust 单测（CI 硬门槛①，隔离临时 HOME，内部就是 cargo test --locked -p agent2api-server）
npm run test:rust

# 插件联调（CI 硬门槛②，需先把测试插件编出来）
AGENT2API_TEST_PLUGIN_EXE=/abs/path/to/agent2api-providers npm run test:rust:plugins

# 覆盖率（只在 beta push / 手动跑）
npm run test:rust:coverage

# 前端：类型检查 + 重建 + 同源校验
npm --prefix desktop-tauri/ui-kit ci
npm --prefix desktop-tauri/ui-islands ci && npm --prefix desktop-tauri/ui-islands run build
npm --prefix desktop-tauri/mobile ci && npm --prefix desktop-tauri/mobile run build
npm --prefix desktop-tauri/i18n ci && npm --prefix desktop-tauri/i18n run build
git status --porcelain -- desktop-tauri/ui   # 必须为空
```

- Rust 改动的验证只跑受影响 crate 的测试：`cargo test --locked -p agent2api-server <模块路径>`；壳侧改动至少 `cargo check -p workbuddy-proxy-desktop`。
- 不跑 `cargo fmt` / `clippy -- -D warnings` 当门槛（仓库没配、CI 没跑）。真要格式化，先看同文件既有风格，别把整文件重排混进功能提交。
- 只改文档时不跑业务测试；只改前端文案时 `tsc --noEmit` + 同源校验即可，不必全量重建两遍（watch 模式更快）。
- 验收范围按改动实际影响选择，不默认跑全量：只改文档就不跑业务测试；改某模块先跑该模块测试，稳定后再跑受影响范围；动公共接口/共享逻辑才扩大到 `npm run test:rust`。

## 6. 跨语言违规对照

| 违规写法 | 本项目要求 |
|---|---|
| 新增无调用方的「通用工具」 | 已经有第二处真实复用再抽象 |
| 硬编码中文不进 `t()` | `t('中文')` + 登记域 + scan/build |
| 手改 `ui/islands/ui.js` 等产物 | 改源码后重建 |
| 注释只写「这里调用 xxx」 | 写「为什么这样而不是那样」；模块头必写 |
| 把 Node 版的怪行为「顺手修正」 | 先读注释里的 `server.mjs` 依据，改了要单独说明 |
| 提交里混入无关格式化 / 重排 | 拆开，或先问 |

## 7. 维护本 Skill

- 规范以代码现状为准：发现文中与代码不一致（例如某个「已知过时注释」），**改本文**，别迁就旧注释。
- 新增或变更的约定，先在一处落地并成为同层多数（≥3 个文件）再写进本文，避免把个人风格写成仓库规范。
- **不要写会过期的计数**（「N 处」「N%」）。要表达普遍性就用「绝大多数 / 个别遗留」，必要时标注「抽样于 YYYY-MM」。数字只会让文档三个月后开始说谎。
- 与 `release-flow`（分支/发版）、`provider-adaptation`（提供商/WASM/`supports_*`）重叠的部分，以那两个 Skill 为准，本文只做代码层指引。

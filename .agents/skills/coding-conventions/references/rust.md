# Rust 细则（desktop-tauri/src-tauri）

面向 `desktop-tauri/src-tauri/src/`（Tauri 壳）与 `desktop-tauri/src-tauri/server/`（网关 crate）。
通用约定（中文「为什么」注释、模块头、产物纪律、验收命令）见 `../SKILL.md`。

## 1. 分层与文件形状

- `api/` 只做「解析请求 → 调 core → 组响应」，业务不写在 handler 里；`core/` 是域服务；`db/` 只碰 SQL。
- 大模块用「`xxx.rs` 总装 + 同名目录」：`auto_checkin.rs` + `auto_checkin/`、`server/mod.rs` + `server/`。
- 新增受保护路由：往 `http::router` 的 `protected` 分组里 `route(...)`，**不要**另挂全局中间件 + 白名单（`server/http.rs` 模块头写了理由：放错组一眼能看出来）。
- 模块头写清「谁调我、我调谁、边界在哪」；`server/mod.rs` 的超长模块头是范本，`db/mod.rs` 的「为什么单连接 / 为什么 Mutex」是取舍类注释的范本。
- 桌面壳侧：Tauri command 放 `commands.rs` 并在 `lib.rs` 注册；状态放 `state.rs`；进程/端口/退出逻辑各自独立文件，不要把系统调用塞进 command 里。

## 2. 命名

| 对象 | 规则 | 例子 |
|---|---|---|
| 服务句柄类型 | 后缀 `Service` / `Store` / `Manager` | `AccountStore`、`AuthService`、`UpdateManager` |
| trait | 名词/角色 | `ProviderAdapter`、`GatewayCallbacks` |
| 常量 | `SCREAMING_SNAKE_CASE` | `TICK_MS`、`MAX_BODY_SIZE`、`KEY_CORS_ENABLED` |
| DB 表/列 | `snake_case` | `accounts(id, provider, priority, …)` |
| 对外 JSON | `#[serde(rename_all = "camelCase")]` | 内部枚举可用 snake_case |
| Tauri command | `snake_case`，动词开头 | `api_request`、`end_port_occupant`、`login_kuku_sms_send` |
| 事件名 | 冒号分域或 kebab | `backend:ready`、`login:progress`、`management-session-changed` |
| 错误变体/枚举 | 名词短语，不用 `Error1` 这类占位 | `UpstreamErrorClass`、`ErrorDecision` |

## 3. 错误处理：不用 `anyhow` / `thiserror`

仓库无这两个依赖，错误是**手写 struct + builder**：

```rust
pub struct GatewayError {
    pub message: String,
    pub status_code: i32,
    pub upstream_code: Option<i64>,
    pub code: Option<String>,
    pub retry_after_secs: Option<u64>,
    // …内部字段 pub(crate)，不序列化
}
```

- handler 直接返回 `Result<T, GatewayError>`，靠 `impl IntoResponse` 自动变成**状态码 + OpenAI 风格 body**（`server/errors.rs`）。
- 管理面用独立信封 `management_error(status, msg)` → `{success:false, error}`；壳侧与低层函数用 `Result<_, String>`（一句能打给人看的话，`db/mod.rs` 的模块头有理由）。
- 新增 `From<...>` 只覆盖「记日志没意义」的转换（io / serde_json → 500）。
- **`unwrap` / `expect` 在非测试路径基本禁用**：release profile 是 `panic = "abort"`，一次 panic 带走整个桌面应用（`errors.rs` 模块头明文）。
- 允许的 `expect` 只有「构造期不变量 + 附中文理由」，例如 `Method::from_bytes(b"PROPFIND").expect("PROPFIND 是合法 HTTP 方法")`。
- 日志与错误分开：先 `logging::log` 一条中文可读文案（❌/⚠️ 前缀），再 `return management_error(...)`；不要把同一句话既当日志又当异常。
- 错误只能**添加信息**，不能把上游正文、凭据、token 塞进 message 或内部字段。

## 4. 异步与并发

- **spawn 一律走 `spawn_task`**（`server/src/lib.rs`）：绝大多数后台任务都走它，裸 `tokio::spawn` 是个别遗留。它兼容 tauri 运行时与 headless `#[tokio::main]`，无运行时上下文时落惰性全局 runtime，**绝不 panic**。
- 阻塞操作（文件、SQLite 批量、进程）走 `spawn_blocking`：网关内用 `core/blocking.rs` 的 `run()`（包成 `GatewayError`）/ `in_place()`；壳侧用 `tauri::async_runtime::spawn_blocking`。
- 锁选型默认 `std::sync::Mutex`；`RwLock` 只在「读多写极少的纯内存字段」上用，且要写理由（`db/mod.rs` 的 Mutex vs RwLock 一节）。
- **硬约束：持锁期间绝不做网络请求**（跨模块一致要求）。闭包里只读写库，需要出网的事先在闭包内取快照、出闭包再请求、单独回写。
- 停机用 `tokio_util::sync::CancellationToken` + oneshot；channel 按语义选 oneshot / mpsc / watch / broadcast，不要一律 mpsc。
- 超时：一般用 `tokio::time::timeout`；涉及 SSE / 长流的 reqwest **只设 `connect_timeout`，不设总 `.timeout()`**（`core/egress.rs`：总超时会掐断流）。
- 要在同步代码里跑异步、或在异步里跑阻塞时，先找仓库里已有入口，不要自己 `block_on`。

## 5. 持久化

- 单连接 `Arc<Mutex<Connection>>`，`Db` 可 Clone 且共享同一连接；**不要引入连接池**（`db/mod.rs` 有完整取舍）。
- 访问形态固定为闭包：`db.with(|conn| …) -> Option<T>` / `with_mut(...)`；**锁中毒返回 `None`** 让调用方显式降级，不 `into_inner()`（`db/mod.rs` 解释了与其它 store 判据的不同）。
- 事务用 `unchecked_transaction()`（迁移框架签名是 `&Connection`）。
- 所有 DDL 集中 `db/schema.rs`，`PRAGMA user_version` 逐版本升级，新表 `CREATE TABLE IF NOT EXISTS`；旧 JSON 迁移一项一文件放 `db/migrate/`，启动只探测、用户确认后才执行。
- `ServerState` 里数据库是 `Option<Db>`：打不开就降级启动，不能让应用闪退；各 store 的接线点必须按「库可能不存在」写。

## 6. 测试

- 单元测试**同文件** `#[cfg(test)] mod xxx_tests { … }`，这是常态；仓库**没有**顶层 `tests/` 集成目录。模块名描述性，不叫清一色 `tests`：`panel_401_cookie_tests`、`round_budget_tests`。
- 测试名是完整英文句子，写清被保护的行为：`the_panel_401_clears_the_stale_access_cookie`、`a_same_day_window_already_in_the_past_is_not_invented`。
- 断言带中文说明，讲清「为什么必须是这个值」：`assert_eq!(cookies.len(), 1, "401 只该发一条清除头（多条会被中转拼坏）")`。
- 需要「现在」的函数拆成双入口：纯函数接收时间参数，线上入口取 `Utc::now()` 再转调（见 `parse_quota_reset_at_at`）。
- 隔离与注入：`server/src/test_support.rs` 提供 thread_local 临时 HOME、`with_key` 注入、`isolated()` 子进程隔离；路由级测试用 `tower::ServiceExt::oneshot` 直打 Router + 本地 Mock 上游。
- 插件联调用例标 `#[ignore]`，靠 `AGENT2API_TEST_PLUGIN_EXE` 与 `npm run test:rust:plugins` 单独跑。
- 不复制业务逻辑进测试，也不为测试在生产代码里留测试专用分支。

## 7. 依赖与平台

- workspace 共享 `target/` 与 `Cargo.lock`，`[profile.release]` 只在 root 定义（`panic="abort"`、`lto=true`、`opt-level="s"`、`strip`）。
- 每个依赖的 Cargo.toml 条目上方写**为什么选它 / 为什么关掉 default features**；关键库精确锁版（`=x.y.z`）。
- 平台差异走 `[target.'cfg(windows)'.dependencies]` 等分目标段，注释说明「两个平台特性会合并，别漏写」。
- `autobins = false` + 显式 `[[bin]]`；`rust-version` 必须等于真实下限（当前 1.88，由 libsqlite3-sys 决定）。

## 8. 风格硬事实

- 行宽事实上限 **约 100**（>120 的基本是注释、测试数据、SVG/base64）。
- import 三组、组间空行：`std` → 第三方 → `crate::`；跨模块一律 `use crate::server::…` 绝对路径，子模块内用 `use super::`。
- `format!` 用内联捕获 `{error}`，别写 `"{}", error`。
- **不刻意避免 `clone()`**：`Arc` + Clone 是显式设计（轻量句柄）；要省的地方用引用，不搞「零 clone 洁癖」。
- `unsafe` 只出现在 Win32 API、libc、插件宿主 native、SQL 导入等必要处，且必须写 `// SAFETY: <中文理由>`。
- `#[allow(dead_code)]` 必须附保留理由（`server/mod.rs` 要求逐个标注）。

## 9. 常见违规

| 违规写法 | 本项目要求 |
|---|---|
| handler 里 `unwrap()` / `expect("…")` | 返回 `Result<T, GatewayError>`，用 `.map_err` / `From` |
| 直接 `tokio::spawn(...)` | 走 `spawn_task`（否则 headless 下可能 panic） |
| 持锁时 `reqwest` / 命令出网 | 闭包内取快照，出闭包再请求 |
| 给 `Db` 加连接池 / 换 `RwLock<Connection>` | 单连接 + Mutex（`db/mod.rs` 有取舍） |
| 新建 `tests/` 集成目录 | 同文件 `#[cfg(test)] mod xxx_tests` |
| 测试名 `test_parse_ok` | 完整句子 + 断言带中文说明 |
| 在业务层重复校验已在边界校验过的输入 | 边界校验一次，内部靠类型流转 |
| 用 `.clone()` 换「零借用洁癖」 | 需要就 clone，但要能说清为什么 |

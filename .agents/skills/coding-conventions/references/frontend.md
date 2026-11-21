# TypeScript / React 细则（desktop-tauri 前端）

面向 `ui-islands`、`ui-kit`、`ui-shared`、`mobile`、`i18n`，以及 `scripts/*.mjs` 这类 Node 工具。
通用约定（中文「为什么」注释、模块头、产物纪律、验收命令）见 `../SKILL.md`。

## 1. 技术栈（别猜错）

- **React 19**（不是 Preact），`@base-ui/react` 交互原语 + `class-variance-authority` 变体。
- Vite 7 **lib 模式**打 **IIFE 单文件**（`ui/islands/ui.js`）：60 多个前端脚本都是同步经典脚本，混 ESM 会出时序问题（`ui-islands/vite.config.ts` 有注释）。
- Tailwind **v4**（`@tailwindcss/vite`），无 `tailwind.config.js`，令牌在 `ui-kit/styles/theme.css`。
- TS `strict: true`、`target ES2022`、`moduleResolution: "bundler"`、`jsx: "react-jsx"`、`noEmit`。
- 报表用 `recharts`，长列表用 `@tanstack/react-virtual`。

## 2. 包职责与依赖方向

`ui-islands` / `mobile` → `ui-kit`（vite alias `@ui` 指 `../ui-kit/src`，源码方式接入）+ `ui-shared`（相对路径，无 package.json）。

- 新增桌面界面 → `ui-islands/src/islands/`
- 跨端逻辑/面板 → `ui-shared`
- 通用控件 → `ui-kit`
- `ui-kit` 必须被安装（哪怕只改 islands），否则样式产物不同源。

## 3. 组件写法与命名

- 函数组件 + hooks，无 class 组件。
- props 用 **`type`，不用 `interface`**；`interface` 只用于 `declare global { interface Window … }` 这类声明合并。
- **具名导出**，组件不用 `export default`；ui-kit 组件末尾集中导出：`export { Button, buttonVariants, type ButtonProps }`。
- 文件名 **kebab-case**（`accounts-page.tsx`）；mobile 页面多为 `PascalCase`，同一目录内保持一致。
- 一域多文件、视图与状态分离：`accounts-page.tsx`（视图）/ `-panels.tsx`（子面板）/ `-store.ts`（状态）/ `-api.ts`（请求）/ `-hook.ts`（绑定）/ `-columns.tsx`（表格列），模块头写明各自分工。
- 页面岛在文件尾部用 `mountOnPage('keys', SECTION, mount)` 注册按需挂载（机制见 `island-lazy.ts`）；不要自己往 `document` 上挂全局监听。
- `any` 为 **0**，`enum` 为 **0**（一律字符串字面量 union）。桥对象统一窄类型：`(window as unknown as { workbuddyDesktop?: … })`，不要 `as any` / `@ts-ignore`。

## 4. 状态与后端通信

- **没有状态库**（Redux / Zustand 均无）：桌面端用模块级 store + `subscribe/patch/snapshot`；mobile 用手写 `class Store` + `React.useSyncExternalStore`（`mobile/src/state/store.ts` 头注释写了为什么不用库）。
- 不 import `@tauri-apps/*`。与 Rust 通信走经典脚本桥 `window.workbuddyDesktop` / `window.wbApp`；`invoke` 只用于 `api_request` / `start_login` 这类既有通道。
- 页面间通信用 DOM `CustomEvent`（`wb:page`、`wb:sidebar-layout`）。
- **流式 / 长请求绕开桥直连网关 `/api/*`**（桥有 60s 超时且不支持流式，`assistant-api.ts` 模块头有完整理由）；fetch 的 credentials 必须是 `same-origin`，不能 `include`（`*` 的 CORS 头 + 凭据会被浏览器判失败）。
- 请求态模式：模块级缓存 Map + 参数签名比对 + `generation` ref 防竞态；加载/错误内嵌为 `{ loading, error }`；轮询只在页面可见时进行且不叠加（`usePolling.ts`）。
- 不新增全局变量、不做隐式单例：需要跨岛共享就放 `ui-shared` 的显式导出。

## 5. 样式与设计令牌

- 类名合并一律 `cn()` = `clsx` + `tailwind-merge`（`ui-kit/src/lib/cn.ts`）；调用方覆盖组件默认类靠 twMerge，不靠 CSS 顺序。
- 令牌**双名制**：实值 `--ui-*` 在 `theme.css`，`aliases.css` 兼容旧名，`@theme inline` 映射成 Tailwind 色类。**不要写死颜色/间距**。
- 暗色跟 `data-theme="dark" | "system"`（`@custom-variant dark`），**不是** `.dark` class。
- 组件 API **照 shadcn/ui（Base UI 支）搬**，样式取项目令牌；不要自己发明 prop（`Input` 自造的 `icon` 已被标准 `InputGroup` 取代，见 `ui-kit/src/index.ts` 头注释）。
- 类名顺序惯例：布局 → 边框 → 字体 → 过渡 → 状态；尺寸走 cva 的 size 档，不散写 `h-8 px-3`。
- 手机断点 `max-width: 767px`；别新增断点。

## 6. i18n：中文即键

- 源码直接写 `t('账号')`，**键就是简体原文**；词典是扁平 JSON，按域拆 `i18n/i18n-src/<locale>/<domain>.json`，目标语言 en / zh-Hant / ja / ko / pt-BR（简体不建目录）。
- 域归属按 `scan.mjs` 的 `DOMAIN_RULES` 先见先得；**新文件必须登记**，否则归 `common` 并告警。
- 加一条文案：源码 `t('…')` → 跑 scan 补空骨架（只加键，从不删键改值）→ 人工填译文 → 跑 build 出 `ui/i18n/<locale>/<domain>.js`。zh-Hant 由 opencc 自动转 + `zh-hant-glossary.json` 纠术语，不逐条手翻。
- 运行时缺译回落中文、不报错；界面上的中文是键本身，不算硬编码。

## 7. 风格硬事实

- 单引号、无分号、2 空格缩进、行宽约 100–120。
- import 顺序：外部包 → `@ui` → 相对路径（父级 → 兄弟），不排字母序；类型用 `import type` / 内联 `type`。
- 别名只有 `@ui`；`ui-shared` 用相对路径（`../../../ui-shared/…`）是现状，别擅自加新别名。
- 日志/注释中文；`console.log` 只留在调试分支并带上下文，不留在渲染路径。

## 8. 脚本与工具链

- `scripts/*.mjs` 是 Node ESM 一次性工具：`#!/usr/bin/env node`、**只用 node 内置模块**（根 `package.json` 没有任何依赖，别往里加）、单引号无分号、中文 `console.error` 提示、失败用 `process.exitCode` / `process.exit(1)` 而不是抛裸异常。
- 需要隔离环境时自建临时目录并清理（`cargo-test.mjs` 是范本：临时 HOME + 清理 + 信号处理）。
- Shell：`scripts/release.sh`、`packaging/linux/install.sh` 用 `#!/usr/bin/env bash` + `set -euo pipefail`；`.githooks/pre-commit` 是 `#!/bin/sh` + `set -e`（保持 POSIX，别用 bash 专有语法）。脚本开头写清用法与「为什么需要它」。
- macOS 本机命令按 BSD 写：`sed -i ''`、没有 `grep -P` / `timeout`；`glab` 在 `~/.local/bin`、Homebrew 在 `/opt/homebrew/bin`。
- 平台相关的一次性工具（如 `packaging/linux/install.sh`、`scripts/check-linux-cpu.py`）不引入新运行时依赖；Linux 安装脚本必须能在无网环境执行完（离线包是产品承诺）。
- 不新增测试基建：前端没有 test runner / lint 配置，验收靠 `tsc --noEmit` + 产物同源 + 真机手点。

## 9. 常见违规

| 违规写法 | 本项目要求 |
|---|---|
| `interface Props` / `enum` / `any` / `export default` | `type` + 字面量 union + 窄类型 + 具名导出 |
| `import { invoke } from '@tauri-apps/api'` | 走 `window.workbuddyDesktop` 桥 |
| 写死 `#1f2937` / `p-4` 硬尺寸 | 用 `--ui-*` 令牌 / cva 档位 |
| 自己给 ui-kit 组件加新 prop | 先看 shadcn 有没有标准件 |
| 跨源 fetch 用 `credentials: 'include'` | `same-origin` + 请求头带凭证 |
| 硬编码中文不 `t()` | `t('中文')` + 登记域 + scan/build |
| 加 Zustand / Redux 管一点状态 | 模块级 store / `useSyncExternalStore` |
| 根 `package.json` 加依赖给脚本用 | 脚本只用 node 内置模块 |

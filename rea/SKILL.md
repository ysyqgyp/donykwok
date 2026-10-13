---
name: rea
description: "REA (Reverse Engineer Anything, morluto/rea)：让 AI 智能体逆向分析没有源码的软件。当用户想搞清某个软件/程序包（EXE、APK、IPA、Electron 应用、.NET 程序、固件、抓包文件、网站）里某个功能是怎么实现的、要求反编译或伪代码、对比两个版本的行为差异、分析 HAR/抓包接口，或提到 REA / Reverse Engineer Anything 时使用。普通源码仓库的架构分析不要用 REA。"
---

# REA 用法（已学，备用）

REA 是本地运行的逆向分析工具，以 MCP 服务器 + CLI 两种形态把分析引擎接到 AI 智能体上：智能体提问 → REA 检查目标并追踪代码 → 返回带证据（Evidence）的结论 → 智能体据此解释或重写实现。分析全程在本机，不上传目标文件（智能体所属模型厂商的数据政策另算）。

- 仓库：github.com/morluto/rea（MIT）；npm 包名 `rea-agents`；命令 `rea` / `rea-agents`。
- 版本基线：npm 发布版 6.4.0（2026-10-11 官方文档核查）。REA 更新极快，仓库 main 的能力可能领先发布版；判断功能以实际连接的服务器 tool list 和 schema 为准，不要凭本文断言某功能存在。
- 完整原文在本机学习副本 `~/workspace/rea-study/`：`docs/`（installation.md、cli.md、mcp-contracts.md 及各目标指南）、官方智能体技能 `.agents/skills/reverse-engineer-anything/`（含 5 份 references）。深挖细节先读那里，不要凭记忆编造命令参数；副本缺失时重新 `git clone --depth 1 https://github.com/morluto/rea`。

## 本机（云电脑）现状 — 2026-10-13 核查

- Node v24.20.0、npm 10.9.4，满足 REA 要求（Node 22.x ≥22.19 / 24.x ≥24.11 / 26+，另需 npm）。
- REA 本体**尚未安装**；Ghidra、IDA、Hopper、JADX、JDK 均未装。真要动手分析时按下面路线现装现配，并先向用户报计划。
- 常驻安装用本机惯例的 `~/.local` 前缀（默认 npm 全局目录在易失路径上）：`npm install --global --prefix ~/.local rea-agents`，之后命令为 `~/.local/bin/rea`。一次性调用可直接 `npx -y rea-agents@latest <命令>`。

## 两种用法

### A. 给用户自己电脑上的编程 Agent 注册（REA 的主用法）

在装有目标 Agent 的机器上运行 `npx rea-agents setup`：多选客户端 → 审阅计划（路径、备份、拟装内容）→ 批准后写入 → **重启该 Agent** 才生效。支持 22 个客户端（`--client` 值）：claude_code、claude_desktop、codex、cursor、gemini_cli、windsurf、devin、opencode、antigravity、copilot_cli、commandcode、qwen_code、vscode、grok_build、omp、pi、hermes、qoder、droid、trae、zcode，以及仅 main 有的 dsh。

- 脚本化先出只读计划：`rea setup --client codex --dry-run --json`；`--yes` 必须带明确范围（`--client` 或 `--all-detected`），无范围时只动 REA 已有注册。
- setup 永不安装/升级 Node、npm、Java、Ghidra、IDA、JADX 等无关软件；写配置前会备份、原子替换。Hopper 是单独的可选项，需另行明确批准（`--install-hopper`）才会装。
- 只装技能不注册 MCP：`npx skills add morluto/rea --skill reverse-engineer-anything`（只有说明书，没有工具）。

### B. 在本云电脑用 CLI 直接分析（无需 Agent 注册）

CLI 与 MCP 走同一套工作流和证据契约。每条命令是独立进程，结果要复用就落盘。按目标选命令见 `references/cli-and-routing.md`。最常用的免引擎路线：

```bash
npx -y rea-agents@latest analyze-javascript-application /abs/path/to/app --json   # JS/Electron 静态图谱
rea inspect-managed-artifact /abs/path/Assembly.dll --json                        # .NET 元数据
rea inspect-go-binary /abs/path/to/program --json                                  # Go 构建信息
rea inspect-sqlite-database /abs/path/db.sqlite --json                             # SQLite 结构
```

原生二进制（伪代码/汇编/字符串/调用/引用）必须先配一个 provider：Hopper、Ghidra 或 IDA 之一（均为用户自备；Ghidra 需 12.1.x + 64 位 JDK 21+，用 `GHIDRA_INSTALL_DIR` 指路）。命令形如 `rea analyze|search|function|decompile|instructions|xrefs|trace TARGET --provider ghidra --json`。

## 就绪检查与维护

- `rea doctor --json` 只读体检；按任务缩小范围：`--provider ghidra|hopper|ida`、`--client codex`、`--skill`。整体 healthy=false 不代表选定路线不可用，看 scope_checks。
- 更新：全局安装用 `rea update`；npx 形态用 `npx rea-agents@latest setup` 刷新注册后重启 Agent。卸载：`rea uninstall`（`--purge-data` 另删 `~/.rea` 缓存与状态；不动 Hopper、证据文件和其他技能）。

## 证据纪律（REA 的核心规矩，必须遵守）

- 每个结论区分**观察 / 推断 / 未知**，引用 Evidence ID，保留覆盖范围与局限；静态分析得出的结论不得说成"运行时看到的行为"。
- 先摘要后钻取：默认结果 + 内联证据起步；大结果（真实应用的证据文件可达数百 MB）先落盘，再用摘要/分页/单模块视图（`inspect_analysis_view`），不要把整份 JSON 灌进上下文。
- 目标身份先行：用户只说"我的 app"而没给名称/路径时先问清，不要拿附近的示例或旧报告里的路径擅自当目标。

## 操作规则

- **只分析用户拥有或已获授权检查的软件**（项目 DISCLAIMER 的立场：合法研究、教育、互操作、软件保存）。IDA Pro、Hopper、JEB 等商业引擎须用户自备合法授权，不协助绕过其授权限制。
- 任何写配置、装引擎的动作（setup apply、装 Hopper/Ghidra/JDK/JADX 等）先把计划给用户看、获批再执行；被拒即停，不重试。
- 运行时类分析（运行目标程序、浏览器/Electron 观察、进程捕获）会以用户权限真正执行目标，动手前与用户确认目标与动作范围，不擅自扩大。
- provider 有多个候选而 REA 报 ambiguous 时，从 `details.candidate_ids` 里显式选一个（CLI `--provider` / MCP `provider_id` / 环境变量 `REA_ANALYSIS_PROVIDER`），不要反复盲试。

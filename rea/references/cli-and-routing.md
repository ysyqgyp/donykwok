# REA 命令与路由速查

提炼自学习副本 `~/workspace/rea-study` 的 docs/cli.md、docs/installation.md 与官方技能
`.agents/skills/reverse-engineer-anything/SKILL.md`（基线：npm 6.4.0 / 仓库 main，2026-10-13）。
参数细节以 `rea <命令> --help` 和原文为准；本表只做选路。

## CLI 速查

通用：默认输出 TOON，给程序/存档用 `--json`；退出码 0 完成（可含部分证据）、1 失败、128+N 信号终止。

| 任务 | 命令 |
| --- | --- |
| JS/Electron 静态分析 | `rea analyze-javascript-application <目录或.asar> --json`（`rea analyze PATH` 对目录/.asar 也走这条） |
| 看大证据的摘要/分页 | 结果落盘后 `rea inspect-analysis-view <view.json>`（view 取 summary / page / 单模块） |
| .NET 元数据 / 成员 | `rea inspect-managed-artifact A.dll --json`；`rea inspect-managed-members A.dll [--type 全名] --json` |
| SQLite 快照 | `rea inspect-sqlite-database db.sqlite [--table 名 --row-limit 20] --json` |
| Go 构建信息 | `rea inspect-go-binary <程序> --json`（免 Go、免引擎） |
| 原生分析（需 provider） | `rea analyze / search "词" / function <名> / decompile <地址> / instructions / xrefs <地址> / trace "词"`，均带 `<目标> --provider ghidra|hopper|ida --json` |
| provider 与能力清单 | `rea providers --json`、`rea capabilities --json` |
| 快照复用 | 任一原生命令加 `--snapshot /abs/analysis.json`，同目标同参数第二次直接命中缓存 |
| 证据校验/导出/对比 | `rea evidence-import B.json`、`rea evidence-export B.json OUT.json`（覆盖需 `--overwrite`）、`rea compare L.json R.json` |
| 导入历史源码对比 | `rea import-reference-source /abs/source`（仅 Linux/macOS 的安全打开方式；Windows 需 WSL） |
| 体检 / 安装 / 更新 / 卸载 | `rea doctor [--provider X] [--client X] [--skill] --json`；`rea setup [--client X] [--dry-run] [--yes] [--install-hopper]`；`rea update`；`rea uninstall [--purge-data]` |

## 按目标选路（MCP 工具名；CLI 走上表对应命令）

| 目标 | 首选工具 | 前置条件 |
| --- | --- | --- |
| 原生二进制 / 程序包归档（ZIP/APK/IPA/MSIX/DMG 清单） | `open_binary` 绑定目标，后用聚焦工具；`binary_overview` 看总览（IDA 不提供） | 原生深度分析需 Hopper / Ghidra / IDA 之一 |
| ASAR 或解出的 JS/Electron 目录 | `analyze_javascript_application`，再用 `trace_application_feature` 追功能、`inspect_analysis_view` 分页看模块 | 仅 Node/npm，免引擎 |
| Android APK 代码/类/方法/引用 | `inspect_android_package` 后接 Android 聚焦工具 | 无头 JADX + 完整 JDK（Linux/macOS/Windows x64） |
| .NET PE/CLI 程序集 | `inspect_managed_artifact` | 静态，免引擎；不解包单文件宿主、不解 IL2CPP；NativeAOT 走原生 Ghidra |
| 固件镜像 | `inspect_firmware_regions`；要解包用 `extract_firmware`（输出目录须调用方新指定） | Linux 上调用方自备 Binwalk / Unblob |
| 已打开的用户浏览器页面 / Electron 运行时 | `list_browser_targets` / `list_electron_targets` | 对应浏览器/Electron 已在运行 |
| 抓包：HAR / SAZ / mitmproxy 记录 | `inspect_web_network_capture` | 只读已存文件，不实时抓、不回访其中 URL |
| PCAP 记录 | 先 `inspect_packet_fields` 选字段，再 `inspect_packet_capture`；汇总用 `inspect_pcap_capture` | Linux + TShark 4.x（+capinfos）+ prlimit，`REA_TSHARK_COMMAND` 指路 |
| WASM 产物 | `inspect_wasm_artifact` | 调用方提供 WABT 1.0.42（`REA_WABT_BIN_DIRECTORY`） |
| EVM 字节码 | `inspect_evm_interface`（raw/hex 载体） | 本地文件；选择器/参数/可变性均为推断候选 |
| Go 二进制元数据 | `inspect_go_binary` | 内置静态检查，免 Go 安装 |
| SQLite 数据库快照 | `inspect_sqlite_database`（先 schema，选表再取行，默认 20 行） | Node 24.x ≥24.15 或 26+；用静止快照，连同 WAL 一并看 |
| ELF 离线布局/符号/缓解 | `inspect_binary_layout` | Linux x64，调用方自备 pwntools，不开反汇编库 |
| Linux ELF core 崩溃记录 | `inspect_recorded_crash` | 读记录，不附加活进程 |
| PE 资源树 | `inspect_pe_resources` | 免引擎、免活动目标 |
| 逻辑信号记录 | 先 `inspect_signal_decoders` 选解码器，再 `decode_signal_recording` | Linux + sigrok-cli 0.7.2（`REA_SIGROK_COMMAND`）+ prlimit |
| JEB 工程 | 引擎类工具（经 `REA_JEB_MCP_URL`） | 用户自行启动 JEB 客户端并开 MCP |

会话类：`binary_session`（传 `{}` 查工具可用性与不可用原因）、`get_evidence_bundle`（按摘要+过滤找保留证据）、
`export_evidence_bundle`（整场导出到指定路径）、`close_binary`（收尾，可带 `snapshot_path` 保存）。

## 常见坑

- **证据体积**：真实应用的完整证据可达数百 MB（官方例子：334 MB 的证据摘要视图只有约 10 KB）。永远先落盘 + 摘要，不要整份读入上下文；MCP 报 `resource_constraint`（transport）时用同一会话的 Evidence 引用改看摘要/单节，或导出 bundle。
- **Ghidra 首次查询懒启动**：`open_binary` 不等导入与自动分析，第一个 Ghidra 查询才触发；默认启动截止 330,000 ms，大二进制调环境变量 `REA_GHIDRA_STARTUP_TIMEOUT_MS`；客户端自身的调用超时可能更短，超时不等于分析失败，可关会话重开恢复。
- **Ghidra 配置**：只认已有安装：Ghidra 12.1.x + 64 位 JDK 21+（以该 Ghidra 的 application.java.min/max 为准），`export GHIDRA_INSTALL_DIR=/abs/ghidra_12.1.4_PUBLIC`，`JAVA_HOME` 可选；REA 永不代装 Ghidra/Java。分析用临时只读工程（-readOnly、-deleteProject），不动用户原有工程。
- **provider 选择**：只有一个可用时自动选；多个都支持同一目标时必须显式选（`--provider` / `provider_id` / `REA_ANALYSIS_PROVIDER`）。报错先分清 `ambiguous`（选一个）还是 `provider_unavailable`（修引擎，`rea doctor --provider X --json`）。
- **发布版 vs main**：注册会把 MCP 钉在执行 setup 的那个版本；新技能文案可能描述发布版还没有的能力，以服务器实际 tool list 为准。升级走 `rea update` 或 `npx rea-agents@latest setup` 并重启 Agent。
- **静态 .NET/JS 检查不运行目标**；运行时类工具（浏览器场景、进程捕获）会真的运行/交互目标，属另一档授权。

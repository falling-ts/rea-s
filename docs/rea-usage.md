# REA 使用指南

> REA（Reverse Engineer Anything）是一个**本地优先**的逆向工程工具，提供统一的
> CLI 与 MCP Server，覆盖二进制、应用程序与运行时行为分析。本文档整理自
> `refs/rea`（上游 [`morluto/rea`](https://github.com/morluto/rea)）的 README 与
> `docs/` 文档，用于在本工作区快速查阅其使用方法。

---

## 一、REA 是什么

REA 通过 **MCP（Model Context Protocol）** 让你的智能体（agent）对目标程序发起
检查与代码追踪，并返回带**证据（Evidence）**的分析结果——每条结论都附带其来源、
推导与未决疑问，而不是凭空给出答案。CLI 命令与 MCP 工具使用**完全相同的工作流
与证据契约**，因此既可交给 agent 调用，也可直接在终端里跑。

核心特点：

- **本地分析**：目标在你本机被检查，REA 不上传你的应用。
- **证据优先**：结果区分观察（observation）、推导（derivation）、推断
  （inference）与未决（unknown），并保留来源证据与会话语义。
- **统一入口**：一条 `rea` 命令 + 一个 MCP Server，覆盖静态与运行时分析。
- **分析引擎可自带**：原生（native）深度分析复用你已有的 Hopper / Ghidra / IDA；
  静态 JavaScript 与 .NET 检查不需要任何原生引擎。

---

## 二、环境要求

- **Node.js**：22.19+ 、24.11+ 或 26+（不支持 23、25 及预发布版本）。需配套 npm。
- 用 Bun 全局安装时，要求 Bun 1.2.17+。
- 不同目标有各自的额外依赖（见下表「需求」列）。

---

## 三、安装与注册

### 1. 最快上手（引导式安装）

```bash
npx rea-agents setup
```

该命令会让你选择要配置的 agent，预览它要做的改动（MCP 注册、工作流技能、
可选的 Hopper 安装），**经你确认后才写文件**。支持的客户端包括 Claude Code、
Codex、Cursor、Gemini CLI、Windsurf、Devin、OpenCode、VS Code、Grok Build、
DeepSeek Harness（`dsh`）等（完整列表见 `refs/rea/docs/installation.md`）。

针对 DeepSeek Harness 单独注册：

```bash
rea setup --client dsh
```

它会向 `$DSH_HOME/cordis.patch.yml`（`DSH_HOME` 未设置时为 `~/.dsh/cordis.patch.yml`）
追加一条 REA 的 stdio MCP 条目；重启 Harness 后，工具以 `mcp__rea__<tool_name>`
的形式暴露。仅安装工作流技能（不放 MCP 注册）：

```bash
rea setup --skill --yes
```

技能会安装到 `~/.agents/skills`（`$DSH_AGENTS_HOME/skills` 已配置时优先），
Harness 的 `skill` 工具即可加载 `reverse-engineer-anything`。

### 2. 全局安装（得到一个 shell 可见的 `rea` 命令）

```bash
npm install --global rea-agents
rea --help
```

### 3. 免安装一次性使用

```bash
npx -y rea-agents@latest <command> [args] --json
# Bun 环境：
bunx rea-agents@latest <command> [args] --json
```

### 4. 更新

```bash
rea update                       # 全局/Bun 全局安装
npx rea-agents@latest setup      # 刷新 agent 注册与技能（npx 方式）
```

### 5. 健康检查 / 就绪诊断

`rea doctor` 只读检查本机前置条件与 agent 配置，**不修改任何东西**：

```bash
rea doctor --json                       # 全量审计
rea doctor --provider ghidra --json     # 只看某个原生引擎
rea doctor --client dsh --json          # 只看某个 agent 注册
rea doctor --skill --json               # 只看已安装的技能
```

---

## 四、能在哪些目标上分析

| 目标                      | REA 返回的内容                                                       | 需求 / 指南                                                            |
| ------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 原生二进制                | 伪代码、汇编、字符串、符号、调用与引用                              | Hopper / Ghidra / IDA                                                  |
| 离线 ELF 布局             | 段、节、原始符号/重定位、缓解措施候选                               | 调用方自备 pwntools（Linux x64）                                        |
| Go 二进制                 | 编译器版本、模块、替换项、构建设置                                  | 内置静态 ELF/PE/Mach-O 检查，无需 Go                                  |
| WASM 产物                 | 校验、节、导入/导出 WAT、解码 WAT                                   | 调用方自备 WABT 1.0.42                                                 |
| JavaScript / Electron     | 模块、导入、source map、路由、IPC、原生插件关系                     | 仅需 Node.js / npm                                                     |
| 网站                      | 页面结构、脚本、网络观测、截图                                      | Chrome 系浏览器（CDP）                                                 |
| .NET 程序集               | 元数据、CIL 指令、声明的原生依赖、构建对比                         | 无（仅静态）                                                          |
| Android APK               | Manifest 声明、类、反编译方法与引用                                 | 无头 JADX + 完整 JDK（Linux/macOS/Windows x64）                       |
| Android 设备              | 设备、包、进程、日志、转储、截屏、APK/文件传输                     | 调用方自备 adb                                                        |
| 固件                      | 区域、解包结果、原生分析交接                                        | Binwalk / Unblob（Linux）                                              |
| 进程行为                  | 终端输出、交互、退出与文件系统观测、运行对比                        | Linux/macOS 原生 PTY                                                   |

> 静态 JavaScript 与 .NET 检查**读取文件而不运行应用**；运行时捕获则使用你的用户
> 权限去运行或与目标交互，各运行时指南会说明其副作用。

---

## 五、常见命令速查

所有 MCP 工具名（下划线式，如 `analyze_javascript_application`）都有对应的 CLI
命令（连字符式，`rea analyze-javascript-application`），二者使用同一套工作流。

### 1. JavaScript / Electron 应用（无需原生引擎）

```bash
# 分析一个解包后的应用目录或 .asar
rea analyze-javascript-application /absolute/path/to/app --json

# 结果可能很大（数百 MB），先存盘，再投影摘要/分页/单模块
rea analyze-javascript-application /absolute/path/to/app --json > app-evidence.json
jq -c '{source: {kind: "inline", evidence: .}, view: {kind: "summary"}}' app-evidence.json > app-view.json
rea inspect-analysis-view app-view.json

# 特征追踪：从节点 ID / 路由 / 字符串 / API / IPC / 模块 / 原生导出 追踪可达子图
rea trace-application-feature app-evidence.json --seed "search" --json

# 语义关系追踪、版本对比、源码到产物对比等
rea trace-javascript-semantics app-evidence.json --seed "..." --json
rea compare-application-versions left.json right.json --json
rea compare-source-to-bundle left.json source-graph.json --json
```

### 2. .NET 程序集（静态，无需引擎）

```bash
rea inspect-managed-artifact /absolute/path/Assembly.dll --json        # 识别身份与部署形态
rea inspect-managed-members /absolute/path/Assembly.dll --json         # 类型/签名/方法体/CIL
rea inspect-managed-members /absolute/path/Assembly.dll --type Example.Parser --json
rea inspect-managed-native-boundaries /absolute/path/Assembly.dll --json
rea compare-managed-members a.dll b.dll --json
```

### 3. Go 二进制（无需 Go / 反汇编器）

```bash
rea inspect-go-binary /absolute/path/to/program --json
```

返回编译器版本、模块、替换项、构建设置、原始字节与 SHA-256。

### 4. 原生分析（需先配置 Hopper / Ghidra / IDA）

```bash
# 检查可用 provider 与能力
rea providers --json
rea capabilities --json

# 基础工作流：分析 / 搜索 / 函数档案 / 反编译 / 交叉引用 / 追踪
rea analyze   /absolute/path/to/program --provider ghidra --json
rea search    /absolute/path/to/program "search" --provider ghidra --json
rea function  /absolute/path/to/program main --provider ghidra --json
rea decompile /absolute/path/to/program 0x1000 --provider ghidra --json
rea xrefs     /absolute/path/to/program 0x1000 --provider ghidra --json
rea trace     /absolute/path/to/program "search" --provider ghidra --json
rea instructions /absolute/path/to/program 0x1000 --provider ghidra --json  # 仅汇编
```

macOS 的 `.app` 包可直接作为目标传入。

**选择 provider**：存在多个可用引擎时须显式指定 `--provider hopper|ghidra|ida`；
也可用环境变量 `REA_ANALYSIS_PROVIDER` 设置长期偏好（显式 `--provider` 优先）。
MCP 中则向 `open_binary` 传入 `provider_id`。若返回 `ambiguous` 选择错误，从
`details.candidate_ids` 中选一个；若是 `provider_unavailable`，用
`rea doctor --provider ID --json` 诊断该引擎。

### 5. 网站 / 浏览器（CDP 被动观测）

```bash
rea list-browser-targets --json                       # 发现页面目标
rea inspect-web-page <url-or-target> --json           # DOM/脚本/资源/网络/控制台
rea analyze-web-bundle <url> --json                   # 解析脚本产物、路由、端点
rea observe-web-session <target> --json               # 开启观测窗口记录用户操作
rea discover-webmcp-tools <target> --json             # 实验性 WebMCP 工具发现
rea capture-web-screenshot <target> --json            # 视口截图（内容寻址 PNG）
rea compare-web-screenshots a.png b.png --json        # 本地像素对比
rea compare-web-captures before.json after.json --json
```

需先启动一个**独立浏览器配置**并开启调试端口（Chrome 系，如
`chrome --remote-debugging-port=9222 --user-data-dir=/tmp/rea-chrome`）。

### 6. 证据导入 / 导出 / 对比

```bash
rea evidence-import  /path/to/bundle.json                 # 校验已有证据包
rea evidence-export  /path/to/bundle.json /path/to/canonical.json   # 导出规范形式
rea compare          /path/to/left.json /path/to/right.json         # 对比两个证据包
```

### 7. 导入历史源码做对比

```bash
rea import-reference-source /absolute/path/to/source
```

仅记录哈希与元数据（尊重 `.gitignore` 与默认排除项），不执行应用代码。
Windows 原生主机对该命令返回 `unsupported_host`，需在 WSL 或其他支持的主机上运行。

---

## 六、把 REA 接入你的 agent（MCP）

引导式 `rea setup` 会自动为支持的客户端写入 MCP 注册。也可以手动注册：REA 的
stdio 入口是 `scripts/rea.mjs`，其 MCP 子命令为 `mcp`。例如 DeepSeek Harness 的
条目（`$DSH_HOME/cordis.patch.yml`）形如：

```yaml
- insert:
    - id: rea-mcp
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: rea
        transport: stdio
        command: /absolute/path/to/node
        args: ["/absolute/path/to/rea/scripts/rea.mjs", "mcp"]
```

全局 npm 安装时入口位于 `$(npm root -g)/rea-agents/scripts/rea.mjs`。

注册后，向 `open_binary` 传入目标路径与 `provider_id` 打开原生会话；用
`binary_session` 查询当前可用工具、provider 绑定与能力；分析完成后用
`close_binary` 关闭并释放资源。会话内分析进度支持普通 `tools/call` 进度令牌，
调用方取消会传递给 provider。

---

## 七、输出格式与退出状态

- 默认终端格式为 **TOON**；保存结果给 JSON 消费者时加 `--json`。
  `--json` 在终端上缩进、在管道/重定向时紧凑，文件可读性可用 `jq .` 处理。
- 输出选择与格式**不影响**操作状态。

| 退出码            | 含义                                                                       |
| ----------------- | -------------------------------------------------------------------------- |
| `0`               | 操作完成（结果可能含部分证据、警告或未决疑问）                             |
| `1`               | 操作无法完成（结构化输出会指明无效输入、权限、取消、超时或其他失败）       |
| `128 + N`         | 信号 `N` 结束进程                                                          |

`setup --dry-run` 返回 `planned` 并退出 `0`；取消的 setup 也退出 `0`；
`needs_confirmation` / `needs_human` 则退出 `1`。`doctor` 在所选就绪范围内必要检查
失败时退出 `1`（不可用的可选 provider 仅作参考）。

> 建议在支持 `pipefail` 的 shell 中开启它，避免下游格式化工具吞掉 REA 的失败状态：
> `set -o pipefail`。

---

## 八、快照（缓存）与复用

快照把成功分析结果保存到本地文件（属主只读权限），当目标字节、操作、参数、provider
与设置完全匹配时直接复用，避免重复启动 provider。

```bash
rea analyze /absolute/path/to/program --provider ghidra \
  --snapshot /absolute/path/to/analysis/program.json
# 重复同一条查询即可命中已保存结果
rea analyze /absolute/path/to/program --provider ghidra \
  --snapshot /absolute/path/to/analysis/program.json
```

MCP 中 `open_binary` 接受 `snapshot_path` 原子导入；`close_binary` 接受
`snapshot_path` 与可选 `overwrite: true` 在释放前保存。注意：写入元数据会使当前
会话的快照缓存失效——导入或保存另一个快照前，应先关闭并重新打开目标。

---

## 九、给 agent 的提问范式

向你的 agent 提出目标驱动的请求即可，例如（来自官方 README）：

```text
Understand how search works in the Notes app, show me the evidence,
and build a similar feature for my project.
```

把 `Notes` 换成你的目标应用、把功能换成你想理解的特性。agent 会通过 MCP 调用
REA 检查目标、追踪相关代码，并基于带证据的结果进行解释、实现或测试。

---

## 十、关键文档索引（位于 `refs/rea/docs/`）

- `installation.md` —— 安装、各 agent 注册、provider 配置、更新与卸载、就绪诊断
- `cli.md` —— 命令行与证据契约：命令、provider 选择、快照、导入导出、退出状态
- `mcp-contracts.md` —— MCP 运行时契约、工具清单、会话生命周期、进度与取消
- `mcp-prompts.md` —— agent 引导式调查提示
- 目标专项指南：
  - `javascript-application-workflows.md` / `javascript-runtime-reconciliation.md`
  - `native-investigation.md` / `ghidra-*` / `ida-provider.md` / `windows-ghidra-p0.md`
  - `managed-code-analysis.md` / `go-binaries.md` / `wasm-artifacts.md` / `evm-bytecode.md`
  - `android-analysis.md` / `adb-device-analysis.md` / `firmware-analysis.md`
  - `browser-observation.md` / `browser-scenario-contract.md` / `electron-observation.md`
  - `process-capture.md` / `recorded-crashes.md` / `binary-diagnostics.md`

> 更精确的选项、前置条件与结果契约，请以对应指南与运行中的实际 `tools/list` /
> `binary_session` 返回为准。

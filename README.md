# kimchi

一款由 [kimchi](https://kimchi.dev/) 提供支持的编程智能体 CLI。基于 [pi-mono](https://github.com/badlogic/pi-mono) 编程智能体 SDK 构建，kimchi 让你在终端中拥有一个 AI 驱动的开发助手，并连接到 kimchi 的 LLM 基础设施。

[![kimchi](https://github.com/getkimchi/kimchi/raw/master/kimchi.png)](https://github.com/getkimchi/kimchi/blob/master/kimchi.png)

## 快速开始

安装最新版本：

**Homebrew（macOS / Linux）：**

```bash
brew install getkimchi/tap/kimchi
```

**安装脚本：**

```bash
curl -fsSL https://github.com/getkimchi/kimchi/releases/latest/download/install.sh | bash
```

然后配置你的 API 密钥并启动：

```bash
kimchi setup   # 一次性交互式配置
kimchi         # 启动编程智能体
```

运行 `kimchi --help` 查看所有可用的子命令和参数。

## 模型

### 模型选择

支持的模型列表在启动时从 kimchi 元数据服务获取。在交互式 CLI 中使用 `/model` 或 `ctrl+p` 在可用模型之间切换。

Kimchi 以以下两种模式之一运行：

| 模式 | 页脚指示 | 行为 |
|------|----------|------|
| **多模型** | `multi-model (orchestrator-id)` | 编排器将每个任务委派给分配给该角色的模型 |
| **单模型** | 模型名 | 所有工作直接在所选模型上运行 |

使用 `ctrl+p` 在模型之间循环切换。循环中的最后一项是 `multi-model`。你也可以打开 `/model` 选择器，从列表中选择特定模型或 `multi-model`。

在单模型模式下，编排系统提示词（环境、工具、研究规则、指南、阶段标记）保持激活，但任务分类和委派被禁用。如果显式要求智能体委派，子智能体工具仍然可用。

### 模型角色

在多模型模式下，每种任务类型由特定角色处理。每个角色可以有一个模型或**候选池**——编排器读取模型的等级和描述，为每个任务挑选最合适的模型。

在交互式 CLI 中使用 `/multi-model` 按角色启用/禁用模型，或直接编辑 `~/.config/kimchi/harness/settings.json`：

```json
{
  "modelRoles": {
    "orchestrator": "kimchi-dev/kimi-k2.6",
    "builder": ["kimchi-dev/minimax-m2.7", "anthropic/claude-sonnet-4-5"],
    "reviewer": "anthropic/claude-sonnet-4-5",
    "explorer": "kimchi-dev/nemotron-3-ultra-fp4"
  }
}
```

| 角色 | 默认 | 说明 |
|------|------|------|
| **orchestrator**（编排器） | `kimi-k2.6` | 运行主循环，分类任务，委派工作。单一模型。 |
| **planner**（规划器） | `kimi-k2.6` | 设计方案，编写规格。与编排器相同时，规划在进程内完成。 |
| **builder**（构建器） | `kimi-k2.6`、`minimax-m2.7` | 代码实现。对于复杂任务，编排器可能从池中挑选更重的模型。 |
| **reviewer**（审查器） | `kimi-k2.6`、`minimax-m2.7` | 代码审查。编排器按等级挑选最强的模型进行初次审查。 |
| **explorer**（探索器） | `nemotron-3-ultra-fp4` | 代码库探索——浏览文件、阅读代码、追踪架构。 |
| **researcher**（研究员） | `kimi-k2.6` | 代码库之外的研究——网络搜索、文档查阅、外部资源。 |

默认值硬编码在 `DEFAULT_MODEL_ROLES` 中。角色接受任何 `provider/model-id` 字符串或字符串数组。只需指定非默认值；缺失的键回退到默认值。

#### 委派的工作原理

编排器接收从角色配置生成的明确阶段指令。对于每个流水线阶段（plan、build、review、explore、research），系统提示词会精确地告诉编排器该做什么：

- **它拥有的角色**（其模型 ID 出现在角色池中）："请自行完成此工作。"
- **它不拥有的角色**："不要自行完成此工作。委派给 Agent(type: X, model: Y)。"——使用池中的具体模型 ID。
- **审查始终被委派**，即使编排器拥有 reviewer 角色，也是为了通过全新上下文确保独立性。

当角色池有多个模型时，编排器挑选最轻量且匹配任务的模型，仅对复杂工作（并发、算法）或作为标准模型失败后的重试才升级到重量级模型。

内置模型（kimchi-dev）已固化等级和描述。外部模型需要元数据——见下文。

#### 模型元数据

外部模型（Anthropic、OpenAI 或任何非内置提供方）没有内置的等级和描述。没有元数据时，它们默认是 `standard` 等级、`vision: false`，并根据分配的角色自动生成描述。要为编排器提供更好的路由信息，请在 `settings.json` 中添加 `modelMetadata` 段：

```json
{
  "modelRoles": {
    "builder": ["kimchi-dev/minimax-m2.7", "anthropic/claude-sonnet-4-5"],
    "reviewer": "anthropic/claude-sonnet-4-5"
  },
  "modelMetadata": {
    "anthropic/claude-sonnet-4-5": {
      "tier": "heavy",
      "description": "强大的通用模型——用于复杂构建和深度审查。"
    }
  }
}
```

| 字段 | 默认 | 说明 |
|------|------|------|
| `tier` | `standard` | `light`、`standard` 或 `heavy`。用于基于复杂度的路由。 |
| `description` | 自动生成 | 显示给编排器，便于将模型能力与任务需求匹配。 |
| `vision` | `false` | 模型是否支持图像输入。 |

在上述元数据下，编排器将对简单的构建块使用 minimax，复杂的则使用 Claude。若没有元数据，两个模型对编排器而言都呈现为 standard 等级，选择将是任意的。

元数据也可以通过 `/multi-model` → "Edit model metadata"（编辑模型元数据）以交互方式管理。自定义覆盖可以从同一菜单重置为默认值。切换到未知模型时，向导会提示配置元数据。内置模型的元数据也可以用同样的方式覆盖。

### 阶段追踪

Kimchi 为每个 LLM 请求打上 `phase:{name}` 标签，用于使用分析和成本归属。编排器在工作推进过程中设置阶段，并显示在页脚中。

| 阶段 | 说明 |
|------|------|
| `explore` | 浏览代码库，阅读文件以了解结构 |
| `plan` | 设计、拆解任务，编写规格 |
| `build` | 编写、修改或重构代码 |
| `review` | 分析输出，验证正确性 |
| `research` | 查阅文档，研究问题 |

子智能体从编排器继承当前阶段，但不能修改它。

## 标签

Kimchi 支持为 LLM 请求打标签，以便进行使用跟踪和成本归属。标签随每个请求一起发送，并按 key 分组、彩色编码后显示在页脚。

### 命令

| 命令 | 说明 |
|------|------|
| `/tags` | 列出所有活动标签 |
| `/tags add key:value ...` | 添加一个或多个标签（例如 `/tags add project:myapp team:backend`） |
| `/tags remove tag ...` | 移除一个或多个用户定义的标签 |
| `/tags clear` | 移除所有用户定义的标签 |

### 标签格式

标签使用 `key:value` 格式。键和值必须以字母数字字符开头和结尾（中间可包含 `-`、`_`、`.`），每个最多 64 字符，总共最多 10 个标签。

### 静态标签

通过 `KIMCHI_TAGS` 环境变量设置（逗号分隔）。静态标签在会话内只读，并以 `[static]` 标记显示。

```bash
export KIMCHI_TAGS="team:backend,project:api"
```

### 自动标签

以下两个标签会自动添加到每个请求，且不计入 10 个标签的限制：

- `model:{model_id}` —— 处理该请求的模型
- `phase:{phase}` —— 当前工作阶段

### 持久化

用户定义的标签（通过 `/tags add` 添加）持久化到 `~/.config/kimchi/tags.json`，跨会话保留。来自 `KIMCHI_TAGS` 的静态标签必须在每个会话中重新设置。

## Ferment —— 跨会话项目管理

Ferment 是 Kimchi 用于多会话工作的渐进细化项目模式。Ferment 不再每次对话都从零开始，而是将结构化的计划（目标、阶段、步骤）作为 JSON 状态文件持久化。

### 快速开始

```bash
kimchi --ferment "Build Tetris"
```

或者在活跃会话内：

```
/ferment new "Build Tetris"    # 创建一个 ferment
/ferment auto                  # 持续推进直到完成或阻塞
```

### 概念

- **Ferment** —— 顶层项目（例如 "Build Tetris"、"Auth rewrite"）
- **Phase（阶段）** —— 项目中的里程碑（例如 "Canvas & Grid"、"Movement"）
- **Step（步骤）** —— 阶段中的单个可执行任务（例如 "Create index.html"）
- **Decision（决策）** —— 为后人记录的架构选择
- **Memory（记忆）** —— 工作中遇到的坑、规范或模式

### 状态机

所有生命周期转换都由确定性有限状态机进行验证，强制执行合法状态变更并阻止非法操作（例如在步骤开始前完成、跳过已完成的阶段）。

```
draft -> planned -> running -> [paused] -> complete
```

1. **draft** —— 通过 `/ferment new` 创建，智能体以对话方式收集目标和阶段
2. **planned** —— `scope_ferment` 设置目标、标准、约束、阶段拆分
3. **running** —— `activate_ferment_phase` 启动阶段，智能体执行步骤
4. **paused** —— 需要用户介入，或通过 `/ferment pause` 暂停
5. **complete** —— 所有阶段终态

### 延续策略

控制活跃的 ferment 是否跨越阶段边界继续推进。

| 策略 | 行为 | 命令 |
|------|------|------|
| **manual**（手动） | 进入下一阶段前询问 | `/ferment manual` |
| **automated**（自动） | 持续推进直到完成、阻塞、暂停或需要用户输入 | `/ferment auto` |

暂停/恢复是独立的：`/ferment pause` 停止 ferment，`/ferment resume` 按当前策略继续。`/ferment exit` 在不删除或放弃 ferment 的情况下退出 Ferment 模式；已计划/运行中的工作会先被暂停，活跃的 Ferment UI/工具被清空，之后你可以从 `/ferment list` 或 `/ferment switch` 中再次选择它。

### 命令

| 命令 | 说明 |
|------|------|
| `/ferment` | 通过提示词启动新 ferment |
| `/ferment new "Name"` | 创建新 ferment |
| `/ferment switch <id>` | 按 ID 前缀或名称恢复 |
| `/ferment delete <id>` | 永久删除 |
| `/ferment export` | 将统计信息导出为 JSON |
| `/ferment progress` | 打开阶段/步骤导航浮层 |
| `/ferment manual` | 设置手动延续策略 |
| `/ferment auto` | 设置自动延续策略 |
| `/ferment pause` | 暂停活跃 ferment 的生命周期 |
| `/ferment resume` | 恢复活跃 ferment 的生命周期 |
| `/ferment exit` | 退出 Ferment 模式但不删除该 ferment |

### 恢复

每个会话都会在会话日志中写入一条 `ferment_reference` 条目。再次启动时，harness 读取该条目并从确切的状态恢复。

```bash
# Day 1
$ kimchi --ferment "Build Tetris"
# ... 智能体工作中，崩溃，终端关闭 ...

# Day 2
$ kimchi --ferment "Build Tetris"
# -> 恢复状态，精准地从上次中断的 Phase 2 继续
```

### 状态存放位置

```
.kimchi/
  ferments/
    <uuid>.json          -- 快照（机器可读的计划状态）
    <uuid>.events.jsonl  -- 每次转换的只追加审计日志
  sessions/
    <timestamp>.jsonl    -- 聊天记录 + 工具调用
```

每次变更都作为只追加事件持久化，并带有变更前/后的状态哈希，从而实现完全可审计。统计信息（阶段/步骤数、耗时、模型用量、评分分布）按需从快照计算，并通过 `/ferment export` 导出。

完整文档请参见 `docs/ferment.md` 与 `docs/ferment-storage-schema.md`。

## LSP 集成

kimchi 内置 Language Server Protocol（LSP）支持，为智能体提供具备类型感知的代码智能。该扩展默认加载——无需任何配置。

### 支持的语言

| 语言 | 服务 | 安装 |
|------|------|------|
| TypeScript / JavaScript | `typescript-language-server` | `npm i -g typescript-language-server typescript` |
| Go | `gopls` | `go install golang.org/x/tools/gopls@latest` |

服务通过 `PATH` 上的 `which` 自动检测。如果找不到服务二进制，相应工具将静默不可用。

### 工具

| 工具 | 说明 |
|------|------|
| `lsp_diagnostics` | 获取文件的类型错误、警告和诊断信息 |
| `lsp_hover` | 获取某位置符号的类型信息和文档 |
| `lsp_definition` | 跳转到符号的定义（支持 `typeDefinition` 和 `implementation` 变体） |
| `lsp_references` | 在整个代码库中查找符号的所有引用 |
| `lsp_rename` | 跨所有文件原子地重命名符号 |

智能体被引导优先使用 LSP 工具而非基于文本的方式（例如跳转定义时用 `lsp_definition` 而非 `grep`）。通过 `edit`、`write` 或 `read` 修改的文件会自动同步到语言服务器。

### 状态栏

LSP 服务激活时，状态栏显示服务名称和当前诊断错误数（例如 `LSP: typescript-language-server (3 diags)`）。

## 远程 Teleport（预览）

使用 `kimchi --teleport` 启动以启用会话多路复用命令。本地 TUI 仍是大本营；远端 worker 被派生、分离、再连接，无需重启 kimchi。

```bash
kimchi --teleport
```

TUI 内可用的斜杠命令：

| 命令 | 说明 |
|------|------|
| `/teleport [name] [flags]` | 将工作树同步到全新远端沙箱并置于前台。参数：`--allow-dirty`、`--exclude <glob>`、`--include-ignored`、`--abandon-pending`、`--force` |
| `/detach [--abandon-pending]` | 断开与前台远端的 WebSocket，返回本地大本营。服务器继续运行会话。 |
| `/attach <name-or-id>` | 重新连接到之前分离的远端 |
| `/sessions` | 列出全部内容：前台远端、已分离远端、服务端会话 |
| `/connect [name-or-id]` | 通过 teleport 代理在沙箱上打开交互式 SSH shell |

`--teleport` 与 `--remote` 互斥。启动时连接单个远端使用 `--remote --session <id>`；从本地大本营多路复用则使用 `--teleport`。

## 配置

### 身份认证

按以下顺序解析 API 密钥：

1. `KIMCHI_API_KEY` 环境变量（优先）
2. `~/.config/kimchi/config.json` 中的 `api_key` 字段

运行 `kimchi setup` 进行交互式首次配置。

### 智能体配置

Kimchi 将其配置（设置、会话、模型）存储在：

```
~/.config/kimchi/harness/
```

### 上下文文件

你可以提供自定义指令，这些指令会在每个会话注入到系统提示词中。Kimchi 会发现两类上下文文件：

**Global（全局）** —— 应用于每个会话，不分项目：

```
~/.config/kimchi/harness/AGENTS.md
```

将适用于所有场景的规则（例如你的姓名、代码风格偏好或全局工具默认设置）放在此文件中。它在任何项目级文件之前加载。

**Project-level（项目级）** —— 在特定目录树中工作时应用。Kimchi 从工作目录向上遍历至文件系统根，每个目录收集一个上下文文件：

```
AGENTS.md
CLAUDE.md
```

每个目录中，`AGENTS.md` 优先于 `CLAUDE.md`。`.local.md` 变体（例如 `AGENTS.local.md`）会附加到对应主文件后，用于用户特定、且被 git 忽略的覆盖。

当全局与项目文件同时存在时，全局指令首先出现在提示词中，其次是祖先目录，最后是工作目录。这意味着项目级规则可以细化或覆盖全局规则。

### 包

Kimchi 支持原生 Pi 包。`kimchi install npm:<package>` 只为 Kimchi 安装包，而 `pi install npm:<package>` 保持包归属于原版 Pi harness。Kimchi 也可以通过 **Pi package lookup**（Pi 包查找）资源加载原版 Pi 包，因此两个 CLI 都可以使用 Pi 包而无需共享 Kimchi 的安装作用域。

如果同一包在两处都安装了，Kimchi 安装的优先。使用以下命令禁用原版 Pi 包查找：

```bash
kimchi resources disable extensions.pi-package-lookup
```

使用以下命令更新包：

```bash
kimchi update                  # 更新已安装的包，然后更新 Kimchi 本身
kimchi update --extensions     # 仅更新已安装的包
kimchi update context-mode     # 按源或显示名称更新单个包
kimchi update self             # 仅更新 Kimchi 本身
```

### HTTP 代理

Kimchi 遵守 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量来处理网络请求。

### Token 优化（RTK）

Kimchi 在安装过程中安装 [RTK](https://github.com/rtk-ai/rtk)，并确保 `rtk` 命令在启动时可用。启用后，kimchi 在执行前通过 `rtk rewrite` 重写 bash 工具调用。这可将命令输出（git、cargo、npm、docker 等）压缩 60-90%，减少 LLM 上下文用量。

每次执行 bash 工具前，kimchi 会调用 `rtk rewrite "<command>"`。如果 RTK 返回重写后的命令（例如 `git status` 变为 `rtk git status`），则执行重写后的版本。

```bash
brew install rtk    # macOS / Linux
```

RTK rewrite 通过资源进行管理：

```bash
kimchi resources disable hooks.rtk-rewrite
kimchi resources enable hooks.rtk-rewrite
```

### 钩子

用户可以添加自定义 Bash 钩子，在 shell 命令运行前重写或阻止它们。全局钩子位于 `~/.config/kimchi/harness/hooks/bash/`；项目钩子位于 `.kimchi/hooks/bash/`，默认禁用，需通过 `/resources` 或 `kimchi resources enable ...` 启用。

钩子协议与示例请参见 `docs/hooks.md`。

### 从其它编程智能体迁移

首次运行时，kimchi 会查找现有的 **Claude Code**、**OpenCode** 或 **Cursor** 安装，并提供迁移其 MCP 服务器的选项。如果有可迁移内容，你将看到一次性提示：

```
+  Claude Code + OpenCode + Cursor configuration found
|
|  MCP servers: filesystem, github, ripgrep
|  Claude Code skills: 4 in ~/.claude/skills
|  OpenCode skills: 2 in ~/.config/opencode/skills
|  Cursor skills: 3 in ~/.cursor/skills
|
*  Migrate MCP servers to Kimchi?
|  * Migrate now
|  * Skip this time
|  * Never ask again
```

发现的 MCP 服务器会被合并到 `~/.config/kimchi/harness/mcp.json`。命名冲突时，已有的 Kimchi 条目始终优先，因此重复运行迁移是安全的。

只有在确实有可迁移内容时才会显示该提示。如果两个智能体都未安装或均为空，向导会静默跳过。

#### 扫描来源

| 智能体 | 配置文件（按顺序读取，结果合并） | Skills 目录 |
|------|--------------------------------|------------|
| Claude Code | `~/.claude.json`（顶层 `mcpServers` + 按项目的 `projects[*].mcpServers`） | `~/.claude/skills/` |
| OpenCode | `$OPENCODE_CONFIG`，然后 `~/.config/opencode/opencode.json`、`opencode.jsonc`、`config.json`、`~/.opencode.json` | `~/.config/opencode/skills/` |
| Cursor | `~/.cursor/mcp.json`，然后 `~/.config/cursor/mcp.json` | `.cursor/skills/`，然后 `~/.cursor/skills/` |

对于 OpenCode，同时支持现代（`mcp` 块）和旧版 Go 二进制（`mcpServers` 块）模式。`enabled: false` 的服务器会被跳过。对于 Cursor，`disabled: true` 的服务器会被跳过。

#### 冲突解决

当同一 MCP 服务器名出现在多个来源时：

1. **单个智能体内**：前面的文件优先；项目级条目优先于顶层（Claude Code）；现代 `mcp` 块优先于旧版 `mcpServers`（OpenCode）；`~/.cursor/mcp.json` 优先于 `~/.config/cursor/mcp.json`（Cursor）。
2. **跨智能体**：Claude Code 优先于 OpenCode，OpenCode 优先于 Cursor。
3. **与已有 Kimchi 配置冲突时**：你在 `~/.config/kimchi/harness/mcp.json` 中的条目始终优先。

#### "不再询问"

存储在 `~/.config/kimchi/config.json`（`migrationState: "skip-forever"`）。删除该字段即可重新触发提示。添加对其它智能体的支持改动很小——只需将新定义放入 `src/agent-discovery/agents/` 并追加到注册表中。

## 开发

### 环境要求

- Node.js 22 (LTS)
- [Bun](https://bun.sh/)（开发服务器与二进制编译）
- [corepack](https://nodejs.org/api/corepack.html) 已启用（`corepack enable`）
- pnpm（通过 corepack 自动安装）

### 快速设置

```bash
./scripts/dev-startup.sh
```

该脚本会检查并按需安装 node、pnpm 与 bun，然后运行 `pnpm install`、复制资源，并通过 `pnpm run dev` 启动 harness。

### 手动设置

```bash
git clone git@github.com:getkimchi/kimchi.git
cd kimchi
corepack enable
pnpm install
```

### 命令

| 命令 | 说明 |
|------|------|
| `pnpm run build` | 将 TypeScript 编译到 `dist/` 并复制主题资源 |
| `pnpm run dev` | 通过 Bun 在本地运行 CLI |
| `pnpm run check` | Biome lint + TypeScript 类型检查 |
| `pnpm run lint` | 仅 Biome lint |
| `pnpm run lint:fix` | Biome lint 并自动修复 |
| `pnpm run test` | 使用 vitest 运行测试 |
| `pnpm run test:smoke` | 端到端冒烟测试 |

### 本地运行

首次运行前请先传播资源：

```bash
node ./scripts/copy-resources.js --dev
```

通过 Bun 直接运行 CLI：

```bash
pnpm run dev
```

或者构建独立的二进制：

```bash
pnpm run build:binary
./dist/bin/kimchi
```

### 项目结构

```
src/
  entry.ts              -- 入口点
  cli.ts                -- CLI 逻辑与 harness 初始化
  cli-args.ts           -- 参数解析
  config.ts             -- 身份认证与配置加载
  models.ts             -- 模型元数据获取与注册
  setup-wizard.ts       -- 首次运行向导
  commands/             -- CLI 子命令（setup、login、config、update 等）
  extensions/           -- 智能体扩展
    agents/             -- 子智能体系统（角色、管理器、记忆）
    orchestration/      -- 任务分类、模型注册表、委派
    ferment/            -- Ferment 生命周期工具与 UI
    mcp-adapter/        -- MCP 服务器集成
    permissions/        -- 工具授权流程
    behaviours/         -- 上下文提示行为
    web-fetch/          -- Web 内容获取
    web-search/         -- Web 搜索
    lsp/                -- Language Server Protocol
    login/              -- OAuth 流程
    onboarding/         -- 会话模式启动向导
  ferment/              -- Ferment 状态机、事件存储、统计
  modes/
    interactive/        -- TUI harness
    acp/                -- 通过 stdio 的 JSON-RPC（IDE 集成）
    teleport/           -- 远端会话多路复用
  agent-discovery/      -- 检测与迁移其它编程智能体
  config/               -- 配置加载与合并
  auth/                 -- API 密钥管理
  utils/                -- 共享辅助函数
```

## 基准测试

`benchmark/` 目录包含对 kimchi 会话进行冒烟测试和质量审计的工具。

- **手动基准**（`benchmark/manual/`）—— 针对不同模型运行预定义任务并比较结果。参见 `benchmark/manual/README.md`。
- **Terminal-bench-2**（`benchmark/terminal-bench-2/`）—— 在 Docker 容器内针对 kimchi 运行 [terminal-bench](https://www.harborframework.com/) 套件（89 个任务）。参见 `benchmark/terminal-bench-2/README.md`。
- **会话审计**（`benchmark/audit-session/`）—— 审计已完成的会话的阶段纪律、代码质量、架构、测试、模型契合度与成本效率。参见 `benchmark/audit-session/README.md`。

## 发布

推送版本标签（`v*`）时，GitHub Actions 会自动构建独立二进制。二进制通过 `bun build --compile` 编译，在用户机器上无需任何运行时。

支持的平台：

- macOS（amd64、arm64）
- Linux（amd64、arm64）

发布资源遵循命名约定 `kimchi_{os}_{arch}.tar.gz`，并附带 `checksums.txt`（SHA256）用于校验。

## 许可证

[Apache License 2.0](https://github.com/getkimchi/kimchi/blob/master/LICENSE) —— 有关 CLA 与贡献者指南，请参见 [CONTRIBUTING.md](https://github.com/getkimchi/kimchi/blob/master/CONTRIBUTING.md)。

# 我的 Pi Coding Agent 配置

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/realchendahuang/pi-config?style=social)](https://github.com/realchendahuang/pi-config)
[![GitHub forks](https://img.shields.io/github/forks/realchendahuang/pi-config?style=social)](https://github.com/realchendahuang/pi-config/network/members)
[![GitHub issues](https://img.shields.io/github/issues/realchendahuang/pi-config)](https://github.com/realchendahuang/pi-config/issues)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/realchendahuang/pi-config/pulls)
[![Follow @realchendahuang](https://img.shields.io/badge/Follow-%40realchendahuang-1DA1F2?logo=x&logoColor=white)](https://x.com/realchendahuang)

<!-- ASCII 渲染测试：下面的 banner 用 Unicode 块字符，测试 GitHub 等宽渲染 -->

```
██████╗ ██╗     ██████╗ ██████╗ ███╗   ██╗███████╗██╗ ██████╗ 
██╔══██╗██║    ██╔════╝██╔═══██╗████╗  ██║██╔════╝██║██╔════╝ 
██████╔╝██║    ██║     ██║   ██║██╔██╗ ██║█████╗  ██║██║  ███╗
██╔═══╝ ██║    ██║     ██║   ██║██║╚██╗██║██╔══╝  ██║██║   ██║
██║     ██║    ╚██████╗╚██████╔╝██║ ╚████║██║     ██║╚██████╔╝
╚═╝     ╚═╝     ╚═════╝ ╚═════╝ ╚═╝  ╚═══╝╚═╝     ╚═╝ ╚═════╝ 
                                                               
   17 plugins · 18 global skills · 2 MCP servers · one-line installer
```

一份可直接复刻的 [Pi](https://pi.dev) 配置，整理成教程形式分享。包含 **17 个插件、18 个全局 Skill、2 个 MCP server**，配一键安装脚本。

仓库里的每样东西都经过实际使用筛选，目标是把 Pi 打造成一个能多代理协作、能省 token、能跑浏览器的全能终端编码代理。

---

## 一、Pi 是什么？

[Pi](https://pi.dev)（npm 包 `@earendil-works/pi-coding-agent`）是一个开源的终端 AI 编码代理。你给它一个任务，它在你本地的项目里读写文件、跑命令、调用工具来完成任务。

如果你还没装 Pi，先：

```bash
npm install -g @earendil-works/pi-coding-agent
```

然后跑 `pi` 进入交互界面。本仓库的所有插件都假设你已经有一个能正常工作的 Pi。

---

## 二、Pi 的整体架构

理解架构有助于你挑选和编写插件。Pi 是一个**插件驱动的代理循环**，核心分层：

```
┌─────────────────────────────────────────────────────┐
│  TUI / RPC / Print   ← 用户交互层（终端界面 / API）  │
├─────────────────────────────────────────────────────┤
│  Agent Loop          ← 代理循环：收 prompt → 调模型  │
│                         → 执行工具 → 回传结果         │
├─────────────────────────────────────────────────────┤
│  Providers / Models  ← 多模型抽象：OpenAI / Anthropic │
│                         / 本地 / 自定义 provider      │
├─────────────────────────────────────────────────────┤
│  Tools               ← 模型可调用的工具：bash/read/   │
│                         edit/web_search/mcp/自定义    │
├─────────────────────────────────────────────────────┤
│  Extensions          ← TypeScript 模块，能订阅事件、  │
│                         注册工具/命令/快捷键/主题      │
├─────────────────────────────────────────────────────┤
│  Skills / Prompts    ← Markdown 形式的「过程知识」，  │
│                         按需注入到模型上下文           │
├─────────────────────────────────────────────────────┤
│  Themes              ← JSON 配色方案                  │
├─────────────────────────────────────────────────────┤
│  Packages            ← 上面这些资源的分发单位          │
│                         （npm / git / 本地路径）       │
└─────────────────────────────────────────────────────┘
```

**关键概念：**

- **Extension（扩展）**：TypeScript 模块，最强力的改造方式。能订阅生命周期事件（如 `tool_call` 可以拦截/修改/阻止工具调用）、注册自定义工具、注册命令、改系统提示词、自定义渲染。
- **Skill（技能）**：一个 `SKILL.md` 文件，描述「某类任务怎么做」。模型按需加载，跨会话存活。Pi 实现了 [Agent Skills 标准](https://agentskills.io/specification)，所以 Claude Code / OpenAI Codex 的 skill 也能直接用。
- **Prompt（提示模板）**：可复用的 `.md` 模板，用 `/template` 唤起。
- **Theme（主题）**：JSON 配色文件。
- **Package（包）**：把上面这些打包，通过 npm / git / 本地路径分发。`pi install` 就是装包。

**代理循环的大致流程：**

1. 用户输入 prompt
2. `before_agent_start` 事件（扩展可注入消息 / 改系统提示词）
3. 进入 turn 循环：调模型 → 模型可能调用工具 → `tool_call` 事件（可拦截）→ 执行工具 → `tool_result` 事件（可改结果）→ 回传给模型
4. 模型不再调工具 → `agent_end` → `agent_settled`

扩展可以挂在任何一个环节上。

---

## 三、Pi 的生态与包市场

### 包市场（pi.dev/packages）

Pi 有一个官方的 [包市场](https://pi.dev/packages)，所有在 `package.json` 里带 `pi-package` 关键字的 npm 包都会自动出现在上面。每个包页面会展示版本、下载量、依赖、仓库链接、甚至 demo 视频/截图。

在 Pi 里也可以直接搜包：

```bash
# 用 pi-marketplace 扩展（本仓库已装）
# 或命令行：
pi install npm:@some/package
```

### 三种包来源

| 来源 | 格式 | 说明 |
| --- | --- | --- |
| npm | `npm:@scope/pkg@1.2.3` | 最常见，版本可 pin |
| git | `git:github.com/user/repo@v1` | 适合私有 / 未发布的包，SSH/HTTPS 都行 |
| 本地路径 | `/abs/path` 或 `./rel/path` | 开发调试用 |

### 配置文件位置

| 文件 | 作用 |
| --- | --- |
| `~/.pi/agent/settings.json` | 全局设置：packages 列表、主题、默认模型等 |
| `~/.pi/agent/tools.json` | 工具开关：哪些工具 active / inactive |
| `~/.pi/agent/models.json` | provider 和模型定义 |
| `~/.pi/agent/extensions/*.ts` | 全局自定义扩展（auto-discover，支持 `/reload`） |
| `~/.agents/skills/` | 全局自定义 skill |
| `~/.config/mcp/mcp.json` | MCP server 配置 |
| `.pi/settings.json` | 项目级设置（可跟团队共享） |
| `.pi/extensions/` | 项目级扩展 |

---

## 四、如何改造 / 扩展 Pi

Pi 提供了从轻到重三档改造方式，按你的需求选：

### 轻量：写一个 Skill

最简单的扩展。在 `~/.agents/skills/my-skill/SKILL.md` 写一个 markdown，描述某类任务的标准流程。模型会按需加载它。本仓库的 `chrome-devtools` skill 就是这种形式。

适合：沉淀「怎么调试某个框架」「怎么发版」这类流程知识，零代码。

### 中量：装 / 写 MCP Server

MCP（Model Context Protocol）是跨 agent 的工具协议。你写一个 MCP server（任何语言），在 `~/.config/mcp/mcp.json` 注册，Pi 通过 `pi-mcp-adapter` 把它的工具接进来。本仓库的 `context7`、`chrome-devtools` 两个 MCP 就是这么接的。

适合：接入外部服务（数据库、API、浏览器、文档源），工具跨 agent 复用。

### 重量：写一个 Extension

最强力的改造。创建 `~/.pi/agent/extensions/my-ext.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 拦截危险命令
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("危险!", "允许 rm -rf?");
      if (!ok) return { block: true, reason: "用户阻止" };
    }
  });

  // 注册自定义工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "按名字打招呼",
    parameters: Type.Object({ name: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: `Hello, ${params.name}!` }] };
    },
  });

  // 注册命令
  pi.registerCommand("hello", {
    description: "打招呼",
    handler: async (_args, ctx) => ctx.ui.notify("Hello!", "info"),
  });
}
```

测试：`pi -e ./my-ext.ts`；正式用放 `~/.pi/agent/extensions/` 然后 `/reload`。

适合：权限门禁、git checkpoint、自定义 compaction、外部集成、有状态工具。能订阅的事件覆盖了 session / agent / message / tool / model / provider 全生命周期。

### 打包分发

写好后加个 `package.json`，声明 `pi` manifest 和 `pi-package` 关键字，发到 npm 就成：

```json
{
  "name": "my-pi-ext",
  "keywords": ["pi-package"],
  "pi": { "extensions": ["./extensions"], "skills": ["./skills"] }
}
```

然后 `pi install npm:my-pi-ext`，别人就能用你的扩展了。

---

## 五、快速上手（3 步）

```bash
# 1. 克隆本仓库
git clone https://github.com/realchendahuang/pi-config.git
cd pi-config

# 2. 一键安装（装 17 个插件 + 合并 MCP 配置）
bash install.sh

# 3. 重启 pi，让所有插件和 MCP server 生效
```

脚本会自动把 17 个插件用 `pi install` 装好，并把 2 个 MCP server 的配置合并进 `~/.config/mcp/mcp.json`（如果该文件已存在，会保留你已有的 server，只做合并）。

> 安装完之后，模型的 provider / key 还需要你自己用 `pi config` 配一下——这部分因人而异，不在本仓库范围内。

---

## 六、插件目录（17 个，按用途分组）

下面逐个讲清楚每个插件是干嘛的、为什么选它，并附上仓库地址方便你深挖。

### 🤖 代理编排与工作流

#### `pi-subagents`

子代理委派框架。让主代理可以把任务派给子代理，支持五种工作模式：

- **single**：单个子代理独立完成一个任务
- **chain**：多个子代理串成流水线，前一个的输出喂给后一个
- **parallel**：多个子代理并行跑同一类任务
- **async**：后台异步执行，完成后通知
- **forked-context**：从父会话 fork 出独立上下文

适合做「先让一个代理研究、再让另一个代理实现、最后让第三个代理 review」这种复杂流程。配带一个 `pi-subagents` skill 教模型怎么编排。

- 📦 仓库：<https://github.com/nicobailon/pi-subagents>

#### `@narumitw/pi-goal`

目标驱动模式。用 `/goal` 设一个目标，Pi 会自主推进直到完成，中途遇到阻塞会主动停下来等你。适合那种「给我做完这个 feature」的长任务。

- 📦 仓库：<https://github.com/narumiruna/pi-extensions>

#### `@narumitw/pi-plan-mode`

只读的计划模式（类似 Codex 的 read-only collaboration）。让模型先出方案、跟你讨论清楚，再进入执行。避免一上来就乱改文件。

- 📦 仓库：<https://github.com/narumiruna/pi-extensions>

### 🔍 代码智能与上下文管理

#### `pi-lens`

Pi 的「IDE 眼睛」。给 Pi 加上 AST 级别的代码理解能力：

- 基于 **ast-grep** 的结构化搜索 / 替换（比纯文本 grep 精准）
- 基于 **tree-sitter** 的语法规则检查
- **LSP 诊断**：跑构建前主动查类型错误
- 符号搜索、模块报告、读符号体等「廉价读代码」工具

配套 4 个 skill：`pi-lens-ast-grep`、`pi-lens-lsp-navigation`、`pi-lens-write-ast-grep-rule`、`pi-lens-write-tree-sitter-rule`（教你写自定义规则）。

- 📦 仓库：<https://github.com/apmantza/pi-lens>

#### `context-mode`

省 token 的核心插件。把大输出（日志、构建结果、网页内容、git diff 等）路由进一个沙箱，用代码处理，只把摘要返回给模型。内置一个 **FTS5 全文检索知识库**，可以索引文档、网页、历史会话，之后按需检索。

效果：分析 47 个源文件，直接 read 要烧 ~700KB 上下文；走 context-mode 只回 ~3.6KB。配套 8 个 skill（ctx-search / ctx-index / ctx-stats / ctx-purge / ctx-insight / ctx-doctor / ctx-upgrade / context-mode 本体）。

- 📦 仓库：<https://github.com/mksglu/context-mode>

#### `pi-hermes-memory`

跨会话记忆。让 Pi 记住你之前告诉过它的事（你的偏好、项目约定、之前踩过的坑），下次开新会话还能用上。记忆分 user / memory / project / failure 四类，可搜索。默认基于策略的 token 感知记忆，带 SQLite FTS5 检索 + 自动合并。

- 📦 仓库：<https://github.com/chandra447/pi-hermes-memory>

### 🌐 浏览、检索与外部接入

#### `pi-web-access`

Web 访问全家桶：

- **web_search**：多引擎网页搜索（OpenAI / Brave / Exa / Tavily / Perplexity / Gemini）
- **fetch_content**：抓 URL 内容转 markdown，支持 YouTube 转录、GitHub 仓库克隆、PDF 提取、本地视频抽帧
- 配带 `librarian` skill：带 GitHub 永链的库研究，适合深挖某个开源库的内部实现
- 📦 仓库：<https://github.com/nicobailon/pi-web-access>

#### `pi-playwright`

Playwright 浏览器自动化。让 Pi 能开浏览器、填表单、点按钮、截图、查 console / network。配带 `playwright-browser` skill。适合测 Web 应用、做端到端自动化。

- 📦 仓库：<https://github.com/guwidoe/pi-playwright>

#### `pi-mcp-adapter`

MCP（Model Context Protocol）适配器。让 Pi 能连接任何 MCP server，把它的工具接进来。本仓库的 2 个 MCP server 就是通过它生效的。支持 OAuth、安全审查、按需懒启动。

- 📦 仓库：<https://github.com/nicobailon/pi-mcp-adapter>

#### `pi-marketplace`

Pi 包市场入口。在 Pi 里直接搜索、查看详情、安全审计、安装 npm 上的 pi 包。配套 `marketplace_search` / `marketplace_detail` / `marketplace_audit` / `marketplace_install` 工具。发现新插件很方便。

- 📦 仓库：<https://pi.dev/packages/pi-marketplace>

### ✨ 实用工具与主题

#### `@juicesharp/rpiv-todo`

给模型的 todo list，渲染成实时浮层，**扛得住 `/reload` 和会话压缩**。多步骤任务进度可视化，不容易跑偏。

- 📦 仓库：<https://github.com/juicesharp/rpiv-mono>

#### `@narumitw/pi-statusline`

状态栏增强。替换 Pi 默认 footer，在底部显示模型、上下文占用、git 状态等更丰富的信息，一眼看清当前状态。

- 📦 仓库：<https://github.com/narumiruna/pi-extensions>

#### `@narumitw/pi-github-pr`

在 Pi 里看 GitHub PR 的 review / checks / comment 状态，不用切浏览器。

- 📦 仓库：<https://github.com/narumiruna/pi-extensions>

#### `pi-simplify`

审最近改动的代码，从清晰度、一致性、可维护性角度给建议。改完代码跑一下，把烂味道扫干净。

- 📦 仓库：<https://github.com/MattDevy/pi-extensions>

#### `@firstpick/pi-prompts-git-pr`

一套可复用的 prompt 模板：提交信息、PR 描述、PR review 流程。直接 `/` 唤起对应模板，省得每次手敲。

- 📦 仓库：<https://github.com/Firstp1ck/pi-coding-agent-forge>

#### `@firstpick/pi-skill-deep-research`

带 `deep-research` skill：两阶段严谨研究流程，带 schema / policy 校验。适合需要多源证据、事实核查的高 stakes 研究。

- 📦 仓库：<https://github.com/Firstp1ck/pi-coding-agent-forge>

#### `@victor-software-house/pi-curated-themes`

精选暗色终端主题集（从 iTerm2-Color-Schemes 迁移），含我用的 `github-dark-colorblind`。还带 `adapt-ghostty-theme-to-pi` skill：把 Ghostty 终端主题迁移成 Pi 主题。

- 📦 仓库：<https://github.com/victor-software-house/pi-curated-themes>

---

## 七、全局 Skill 清单（18 个）

所有 skill 都是全局的——装了包就处处可用，跟当前在哪个项目无关。

| 类别 | Skill | 来源包 |
| --- | --- | --- |
| 研究/浏览 | `librarian`、`deep-research`、`chrome-devtools` | pi-web-access / @firstpick / 自定义 |
| 浏览器 | `playwright-browser` | pi-playwright |
| 上下文/知识库 | `context-mode`、`ctx-search`、`ctx-index`、`ctx-stats`、`ctx-purge`、`ctx-insight`、`ctx-doctor`、`ctx-upgrade` | context-mode |
| 代码智能 | `pi-lens-ast-grep`、`pi-lens-lsp-navigation`、`pi-lens-write-ast-grep-rule`、`pi-lens-write-tree-sitter-rule` | pi-lens |
| 代理编排 | `pi-subagents` | pi-subagents |
| 主题 | `adapt-ghostty-theme-to-pi` | @victor-software-house |

> 其中 `chrome-devtools` 是我放在 `~/.agents/skills/` 的自定义全局 skill，不在任何 npm 包里，需要自己创建（参考第二节「写一个 Skill」）。

---

## 八、MCP Server（2 个）

配置在 `mcp.json`，安装脚本会合并到 `~/.config/mcp/mcp.json`。

### `context7`

Upstash 的 Context7 MCP。给模型实时拉取第三方库的**最新文档**，避免它用过时的训练知识写代码。

```json
{
  "command": "npx",
  "args": ["-y", "@upstash/context7-mcp@latest"],
  "lifecycle": "lazy"
}
```

### `chrome-devtools`

Chrome DevTools 远程控制，29 个工具（点击、截图、网络抓包、性能分析等）。配合 `chrome-devtools` skill 调试网页很顺手。

> 这个 server 需要本地装一个 `chrome-devtools-mcp` 二进制，路径在 `mcp.json` 里是写死的，换机器后请改成你自己的路径。

两个都设了 `"lifecycle": "lazy"`——按需启动，不常驻，省资源。

---

## 九、仓库内容

| 文件 | 说明 |
| --- | --- |
| `install.sh` | 一键安装脚本：装 17 个插件 + 合并 MCP 配置 |
| `config.json` | 机器可读的完整配置（plugins / globalSkills / mcpServers / UI / tools） |
| `mcp.json` | MCP server 配置，可直接放到 `~/.config/mcp/mcp.json` |
| `README.md` | 本文件 |

---

## 十、装完之后怎么用

1. **先配模型**：跑 `pi config`，加你自己的 provider 和模型。本仓库不涉及这部分。
2. **重启 Pi**：让新装的插件和 MCP server 生效。
3. **试试 skill**：在 Pi 里直接描述任务，模型会自动匹配合适的 skill；也可以用 `/context-mode:ctx-stats` 这类斜杠命令手动调。
4. **调工具开关**（可选）：Pi 的 `tools.json` 可以把不常用工具设为 inactive，保持工具列表干净。我个人的做法是把 `ast_grep_*` / `grep` / `find` / `ls` / `lsp_navigation` 设为 inactive，需要时通过 `pi_lens_activate_tools` 按需唤起。`config.json` 里有我的完整 tools 配置可参考。

---

## 十一、Star 趋势

[![Star History Chart](https://api.star-history.com/svg?repos=realchendahuang/pi-config&type=Date)](https://star-history.com/#realchendahuang/pi-config&Date)

> 仓库刚建，曲线会随 star 增长实时更新。点击图片可跳转 star-history 交互页。

---

## 十二、贡献活动折线图

下面这张 ASCII 折线图由本仓库的真实 `git log` 生成（按小时分桶统计提交数），直接用等宽块字符绘制——既是贡献趋势可视化，也验证 GitHub 的等宽渲染。

```
按小时提交分布（2026-07-25）

 4 │      ●─
   │      ●─
   │      ●─
   │      ●─
   │  ╱   ●─
   │  ●─  ●─
    └┴───┴───
     17:00 18:00

总提交: 5 | 时间跨度: 17:34 → 18:13
```

> 随着提交增加，这张图会按小时/按天自动重绘。生成脚本思路：`git log --pretty=format:%ad` → 按小时分桶 → 用 `●` / `╱` / `─` 画线。

---

## License

MIT

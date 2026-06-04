# AI Agent Harness 技术评审

## Claude Code vs Codex vs OpenCode 深度对比

> 评审日期：2026-06-05
> 评审范围：Agent Harness 架构、Token 消耗、上下文管理、能力评估

---

## 一、评审对象信息

### 1.1 版本与来源

| 项目 | 来源仓库 | 版本/状态 | 语言 |
|------|---------|----------|------|
| **Claude Code** | [chauncygu/collection-claude-code-source-code](https://github.com/chauncygu/collection-claude-code-source-code) | v2.1.88（2026-03-31 泄露） | TypeScript |
| **Codex** | [openai/codex](https://github.com/openai/codex) | 最新 main 分支 | Rust |
| **OpenCode** | [anomalyco/opencode](https://github.com/anomalyco/opencode) | 最新 main 分支 | TypeScript |

### 1.2 项目背景

| 项目 | 开发方 | 定位 | 开源状态 |
|------|-------|------|---------|
| **Claude Code** | Anthropic | CLI Agent Harness（官方产品） | 源码泄露（非官方开源） |
| **Codex** | OpenAI | CLI Agent Harness（官方产品） | 开源（Apache-2.0 / MIT） |
| **OpenCode** | Anomaly | CLI Agent Harness（社区项目） | 开源（MIT） |

---

## 二、Agent Harness 架构对比

### 2.1 什么是 Agent Harness

Agent Harness 是包裹 LLM 的运行时基础设施，让模型能作为 agent 工作。

```
Agent Harness = Tools + Knowledge + Observation + Action + Permissions
```

核心循环：
```
消息 → LLM → 响应
           ├─ tool_use → 执行工具 → 结果回传 LLM
           └─ text → 返回用户
```

### 2.2 架构全景对比

| 组件 | Claude Code | Codex | OpenCode |
|------|------------|-------|----------|
| **Tools** | 40 个工具 | 42 个处理器 | 18 个工具 |
| **Knowledge** | CLAUDE.md + Skill + MCP + Memory | AGENTS.md + memories 扩展 | AGENTS.md |
| **Observation** | Git diff、文件系统、终端、浏览器 | 文件系统、终端、Git | 文件系统、终端 |
| **Action** | CLI、API、文件、Git、团队协作 | Shell、文件、MCP | Shell、文件 |
| **Permissions** | 3 种模式 + 工具级权限 + Hook | permissions.toml + 请求审批 | 基本权限 |
| **Context 管理** | 5 层压缩体系 | 2 层压缩体系 | 1 层压缩体系 |
| **多代理** | AgentTool + TeamCreate | agent_jobs + 层级代理 | 无 |
| **记忆系统** | Session Memory + 文件记忆 | memories 扩展（26 个文件） | 无 |
| **技能系统** | SkillTool | skills + core-skills | skill 工具 |

---

## 三、Token 消耗对比

### 3.1 测试场景

发送一条简单消息"你好"，估算首次请求的 token 消耗。

### 3.2 System Prompt 大小

| 项目 | 文件 | 大小 | 估算 Token 数 |
|------|------|------|--------------|
| **Claude Code** | `src/constants/prompts.ts` | 55,234 bytes | ~15,700 |
| **Codex** | `codex-rs/prompts/src/*.rs`（排除 tests） | 28,064 bytes | ~8,000 |
| **OpenCode** | `packages/opencode/src/session/prompt/default.txt` | 8,623 bytes | ~2,400 |

### 3.3 Tool 定义大小

| 项目 | 工具数 | 大小 | 估算 Token 数 |
|------|-------|------|--------------|
| **Claude Code** | 41 个 | 135,716 bytes（tool prompts） | ~38,700 |
| **Codex** | 42 个 | 集成在 system prompt 中 | ~15,000 |
| **OpenCode** | 18 个 | 15,285 bytes（tool txt files） | ~4,300 |

### 3.4 总 Token 消耗估算

| 项目 | System Prompt | Tool 定义 | 消息开销 | **总计** |
|------|--------------|----------|---------|---------|
| **Claude Code** | ~15,700 | ~38,700 | ~500 | **~54,900** |
| **Codex** | ~8,000 | ~15,000 | ~500 | **~23,500** |
| **OpenCode** | ~2,400 | ~4,300 | ~300 | **~7,000** |

### 3.5 Token 效率排名

| 排名 | 项目 | Token 消耗 | 评价 |
|------|------|-----------|------|
| 🥇 1 | **OpenCode** | ~7,000 | 最省 token，轻量级设计 |
| 🥈 2 | **Codex** | ~23,500 | 中等消耗，功能丰富 |
| 🥉 3 | **Claude Code** | ~54,900 | 最高消耗，功能最全 |

---

## 四、上下文管理能力对比

### 4.1 Claude Code — 5 层压缩体系

| 层级 | 机制 | 说明 |
|------|------|------|
| **L1** | MicroCompact | 选择性压缩特定工具结果（FileRead、Shell、Grep、Glob、WebSearch、WebFetch、FileEdit、FileWrite），清除旧输出保留最近的 |
| **L2** | Session Memory Compact | 会话记忆压缩，保留最近 5 条文本消息 + 最少 10K tokens，最多 40K tokens |
| **L3** | AutoCompact | 自动压缩，当 token 达到上下文窗口 ~93% 时触发，用 LLM 生成摘要 |
| **L4** | Reactive Compact | 响应式压缩，API 返回 prompt_too_long 时触发 |
| **L5** | Context Collapse | 上下文折叠（实验性），90% 时提交、95% 时阻塞 |

**关键参数：**
```
AUTOCOMPACT_BUFFER_TOKENS = 13,000
WARNING_THRESHOLD_BUFFER_TOKENS = 20,000
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000
POST_COMPACT_TOKEN_BUDGET = 50,000
POST_COMPACT_MAX_FILES_TO_RESTORE = 5
POST_COMPACT_MAX_TOKENS_PER_FILE = 5,000
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25,000
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

**独特能力：**
- **Prompt Cache 检测**：14 种缓存中断向量检测，防止不必要的缓存失效
- **图片/文档剥离**：压缩前自动剥离图片和文档，节省 token
- **技能截断**：压缩后重新注入技能时，每个技能最多 5K tokens
- **文件恢复**：压缩后自动恢复最近 5 个文件内容
- **断路器**：连续失败 3 次后停止自动压缩，避免浪费 API 调用
- **上下文窗口**：默认 200K，支持 1M（Claude Sonnet 4 / Opus 4.6）

### 4.2 Codex — 2 层压缩体系

| 层级 | 机制 | 说明 |
|------|------|------|
| **L1** | Auto Compact Window | 基于 token 使用量的自动压缩窗口追踪 |
| **L2** | Compact Task | 远程压缩（compact_remote）或内联压缩（compact） |

**关键参数：**
```
COMPACT_USER_MESSAGE_MAX_TOKENS = 20,000
```

**特点：**
- **远程压缩**：支持调用远程 API 进行压缩
- **内联压缩**：本地生成摘要
- **Pre/Post Compact Hooks**：压缩前后可执行钩子
- **压缩分析**：完整的压缩事件追踪（触发器、原因、状态、阶段）
- **Token 估算**：使用 `approx_token_count` 进行估算

**缺失能力：**
- 无 MicroCompact（不能选择性压缩工具结果）
- 无 Session Memory 独立压缩
- 无 Prompt Cache 优化
- 无断路器机制

### 4.3 OpenCode — 1 层压缩体系

| 层级 | 机制 | 说明 |
|------|------|------|
| **L1** | Compaction | 自动压缩 + Prune 机制 + 结构化摘要模板 |

**关键参数：**
```
PRUNE_MINIMUM = 20,000 tokens
PRUNE_PROTECT = 40,000 tokens
TOOL_OUTPUT_MAX_CHARS = 2,000 字符
DEFAULT_TAIL_TURNS = 2（保留最近 2 轮对话）
MIN_PRESERVE_RECENT_TOKENS = 2,000
MAX_PRESERVE_RECENT_TOKENS = 8,000
```

**特点：**
- **SUMMARY_TEMPLATE**：结构化摘要模板，包含 Goal → Constraints → Progress → Pending → Active Context
- **Prune 机制**：裁剪旧消息，保留最近 N 轮对话
- **工具输出截断**：工具输出最多 2,000 字符
- **保护机制**：skill 工具输出受保护不被裁剪

**缺失能力：**
- 无 MicroCompact（不能选择性压缩工具结果）
- 无 Session Memory 独立压缩
- 无 Prompt Cache 优化
- 无断路器机制
- 无远程压缩

### 4.4 上下文能力排名

| 排名 | 项目 | 评分 | 核心优势 | 核心短板 |
|------|------|------|---------|---------|
| 🥇 1 | **Claude Code** | ⭐⭐⭐⭐⭐ | 5 层压缩、Prompt Cache 优化、断路器、文件恢复 | 复杂度高 |
| 🥈 2 | **Codex** | ⭐⭐⭐⭐ | 远程压缩、Hook 系统、完整分析 | 缺少精细控制 |
| 🥉 3 | **OpenCode** | ⭐⭐ | 轻量、结构化摘要模板 | 功能相对简单 |

---

## 五、Harness 能力详细对比

### 5.1 工具系统

#### Claude Code（40 个工具）

| 类别 | 工具 |
|------|------|
| 文件操作 | FileReadTool、FileWriteTool、FileEditTool、NotebookEditTool |
| 代码搜索 | GrepTool、GlobTool、ToolSearchTool |
| 系统执行 | BashTool、PowerShellTool、REPLTool |
| Web 访问 | WebFetchTool、WebSearchTool |
| 任务管理 | TaskCreateTool、TaskGetTool、TaskListTool、TaskUpdateTool、TaskOutputTool、TaskStopTool |
| 多代理 | AgentTool、SendMessageTool |
| 团队协作 | TeamCreateTool、TeamDeleteTool |
| 计划模式 | EnterPlanModeTool、ExitPlanModeTool |
| Worktree | EnterWorktreeTool、ExitWorktreeTool |
| 技能 | SkillTool |
| 配置 | ConfigTool |
| 定时任务 | ScheduleCronTool（CronCreate、CronDelete、CronList） |
| 用户交互 | AskUserQuestionTool |
| 其他 | BriefTool、SleepTool、SyntheticOutputTool、TodoWriteTool、RemoteTriggerTool |
| MCP | MCPTool、ListMcpResourcesTool、ReadMcpResourceTool、McpAuthTool |
| LSP | LSPTool |

#### Codex（42 个处理器）

| 类别 | 工具 |
|------|------|
| 代码执行 | shell、shell_command、unified_exec |
| 文件操作 | apply_patch |
| 多代理 | agent_jobs、multi_agents、spawn_agents_on_csv、close_agent、resume_agent |
| 目标系统 | create_goal、update_goal、get_goal、goal |
| MCP | mcp、mcp_resource、read_mcp_resource、list_mcp_resources |
| 用户交互 | request_user_input、request_permissions |
| 插件 | request_plugin_install、list_available_plugins_to_install |
| 技能 | 5 个内置技能（imagegen、openai-docs、plugin-creator、skill-creator、skill-installer）+ core-skills 框架 |
| 搜索 | tool_search |
| 其他 | plan、send_message、send_input、view_image、write_stdin |

#### OpenCode（18 个工具）

| 类别 | 工具 |
|------|------|
| 文件操作 | read、write、edit、apply_patch |
| 代码搜索 | grep、glob |
| 系统执行 | shell |
| Web 访问 | webfetch、websearch |
| 任务管理 | task、todowrite |
| 用户交互 | question |
| 计划模式 | plan（enter/exit） |
| 技能 | skill |
| LSP | lsp |
| 其他 | external-directory、json-schema、mcp-websearch |

### 5.2 多代理能力

| 能力 | Claude Code | Codex | OpenCode |
|------|------------|-------|----------|
| 子代理生成 | ✅ AgentTool | ✅ agent_jobs | ❌ |
| 嵌套代理 | ✅ 支持 | ✅ 支持 | ❌ |
| 团队协作 | ✅ TeamCreate + SendMessage | ❌ | ❌ |
| 批量代理 | ❌ | ✅ spawn_agents_on_csv | ❌ |
| 代理通信 | ✅ 异步邮箱 | ✅ send_message | ❌ |
| 代理生命周期 | ✅ 创建/关闭/恢复 | ✅ 创建/关闭/恢复 | ❌ |
| 工作隔离 | ✅ Worktree | ❌ | ❌ |

### 5.3 记忆系统

| 能力 | Claude Code | Codex | OpenCode |
|------|------------|-------|----------|
| Session Memory | ✅ | ❌ | ❌ |
| 文件记忆 | ✅ Memory 文件 | ✅ memories 扩展 | ❌ |
| 记忆提取 | ✅ 自动提取 | ✅ ad_hoc_note | ❌ |
| 记忆搜索 | ✅ | ✅ search 工具 | ❌ |
| 记忆列表 | ✅ | ✅ list 工具 | ❌ |
| 记忆压缩 | ✅ Session Memory Compact | ❌ | ❌ |

### 5.4 权限控制

| 能力 | Claude Code | Codex | OpenCode |
|------|------------|-------|----------|
| 权限模式 | ✅ 3 种（default/bypass/strict） | ✅ permissions.toml | ⚠️ 基本 |
| 工具级权限 | ✅ | ✅ | ❌ |
| Hook 审批 | ✅ | ✅ Pre/Post Hooks | ❌ |
| 权限请求 | ✅ AskUserQuestion | ✅ request_permissions | ❌ |
| 沙箱隔离 | ✅ | ✅ | ❌ |

---

## 六、代码质量对比

### 6.1 Claude Code

**优点：**
- 完整的 TypeScript 类型系统
- 详细的错误处理和日志记录
- 丰富的配置选项和环境变量
- 完善的测试覆盖

**缺点：**
- 代码复杂度高（`print.ts` 有 5,594 行，单个函数 3,167 行，嵌套 12 层）
- 大量条件编译（`feature()` 宏）
- 部分代码有技术债务

### 6.2 Codex

**优点：**
- Rust 实现，性能优秀
- 清晰的模块化设计
- 完善的测试覆盖
- 良好的错误处理

**缺点：**
- 代码量大，学习曲线陡峭
- 部分功能仍在开发中

### 6.3 OpenCode

**优点：**
- 代码简洁，易于理解
- 轻量级设计
- 快速上手

**缺点：**
- 功能不完整
- 缺少关键组件
- 文档不足

---

## 七、综合评估

### 7.1 能力雷达图

```
                    Token 效率
                        ★★★★★
                           |
                           |
    上下文管理 ★★★★★ ----+---- ★★★★★ 工具丰富度
                           |
                           |
                        ★★★★★
                    多代理能力
```

| 维度 | Claude Code | Codex | OpenCode |
|------|------------|-------|----------|
| Token 效率 | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 上下文管理 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| 工具丰富度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 多代理能力 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ |
| 记忆系统 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| 权限控制 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| 代码质量 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 易用性 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 7.2 综合排名

| 排名 | 项目 | 综合评分 | 定位 |
|------|------|---------|------|
| 🥇 1 | **Claude Code** | ⭐⭐⭐⭐⭐ | 企业级全功能 Agent Harness |
| 🥈 2 | **Codex** | ⭐⭐⭐⭐ | 高性能多代理 Agent Harness |
| 🥉 3 | **OpenCode** | ⭐⭐⭐ | 轻量级入门 Agent Harness |

### 7.3 使用场景推荐

| 场景 | 推荐项目 | 原因 |
|------|---------|------|
| **企业级开发** | Claude Code | 功能最全面，上下文管理最好 |
| **长对话任务** | Claude Code | 5 层压缩体系，不会丢失上下文 |
| **多代理协作** | Codex | agent_jobs 系统强大，支持批量生成 |
| **追求性能** | Codex | Rust 实现，性能优秀 |
| **简单任务** | OpenCode | 最省 token，轻量级 |
| **学习研究** | OpenCode | 代码简洁，易于理解 |
| **预算有限** | OpenCode | Token 消耗最低 |

---

## 八、结论

### 8.1 核心发现

1. **Claude Code 的上下文管理是碾压级的**：5 层压缩体系（Micro → Session Memory → Auto → Reactive → Context Collapse）远超其他两个项目
2. **Codex 的多代理能力最强**：agent_jobs 系统支持批量生成、CSV 导入，适合大规模代理协作
3. **OpenCode 最省 token**：约 7,000 tokens，是 Claude Code 的 1/8，适合简单任务
4. **三者都有上下文管理**：Claude Code 5 层 > Codex 2 层 > OpenCode 1 层

### 8.2 技术趋势

1. **上下文管理将成为核心竞争力**：随着对话长度增加，有效的上下文压缩能大幅降低 token 成本
2. **多代理协作是发展方向**：Claude Code 和 Codex 都在加强多代理能力
3. **轻量级方案有其市场**：OpenCode 证明了简单、省 token 的方案也有用户需求

### 8.3 最终建议

- **如果只能选一个**：选 Claude Code，它的上下文管理能力无可替代
- **如果追求性价比**：选 Codex，功能和 token 消耗之间取得了平衡
- **如果只是入门**：选 OpenCode，简单易用，资源消耗少

---

## 附录

### A. 数据来源

| 数据项 | 来源 |
|-------|------|
| Claude Code 源码 | `chauncygu/collection-claude-code-source-code` 仓库 `original-source-code/` 目录 |
| Codex 源码 | `openai/codex` 仓库 |
| OpenCode 源码 | `anomalyco/opencode` 仓库 |
| Token 估算 | 基于源码中的文件大小和内容分析（~3.5 bytes/token） |
| 上下文管理分析 | 基于源码中的 compact 相关文件 |

### B. 关键文件索引

#### Claude Code
- System Prompt: `src/constants/prompts.ts`（55,234 bytes）
- Tool 定义: `src/tools/*/prompt.ts`（135,716 bytes）
- AutoCompact: `src/services/compact/autoCompact.ts`
- MicroCompact: `src/services/compact/microCompact.ts`
- Session Memory: `src/services/compact/sessionMemoryCompact.ts`
- 上下文窗口: `src/utils/context.ts`

#### Codex
- System Prompt: `codex-rs/prompts/src/*.rs`（28,064 bytes，排除 tests）
- Compact: `codex-rs/core/src/compact.rs`
- Auto Compact Window: `codex-rs/core/src/state/auto_compact_window.rs`
- Memories: `codex-rs/ext/memories/`（26 个 .rs 文件）

#### OpenCode
- System Prompt: `packages/opencode/src/session/prompt/default.txt`（8,623 bytes）
- Tool 定义: `packages/opencode/src/tool/*.txt`（16,575 bytes）
- Compaction: `packages/opencode/src/session/compaction.ts`（640 行）

### C. 术语表

| 术语 | 定义 |
|------|------|
| Agent Harness | 包裹 LLM 的运行时基础设施，提供工具、知识、观察、行动、权限 |
| MicroCompact | 选择性压缩特定工具结果的机制 |
| Session Memory | 会话记忆，跨对话保持的上下文信息 |
| AutoCompact | 当 token 使用量达到阈值时自动触发的压缩 |
| Reactive Compact | 当 API 返回 prompt_too_long 时触发的响应式压缩 |
| Context Collapse | 上下文折叠，实验性的高级压缩机制 |
| Prompt Cache | LLM 提供商的提示缓存机制，可降低重复请求的成本 |
| 断路器 | 连续失败 N 次后停止尝试的保护机制 |

---

*本文档基于 2026-06-05 的源码分析生成，各项目可能已有更新。*

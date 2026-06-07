# opencode 消息处理流程

```
用户输入 → TUI 提交
     │
     ▼
POST /api/session/{id}/prompt  (SDK v2)
     │
     ▼
HTTP handler → SessionPrompt.prompt()
     │
     ▼
构建请求
  ├─ System Prompt
  ├─ 消息历史
  ├─ 可用工具列表
  └─ 参数
     │
     ▼
LLM.stream()
  ├─ AI SDK streamText（默认路径）
  └─ 原生 LLM Runtime（实验性）
     │
     ▼
解析 LLM 事件流
  ├─ text-delta    → 拼装完整回复文本
  ├─ reasoning     → 推理过程
  └─ tool-call     → 进入工具调用循环
     │
     ▼
  有 tool-call? ──是──▶ 执行工具（read / write / shell ...）
     │                       │
     │                       ▼
     │                   工具结果入库
     │                       │
     └─── 否 ←───────── 再接回 LLM 下一轮
     │
     ▼
SSE 事件推送
  ├─ message.text
  ├─ message.tool_call
  └─ message.tool_result
     │
     ▼
TUI 渲染（实时更新）
```

---

## System Prompt 构建

### 问题1：Provider 模板是怎么选择的？

答：`src/session/llm/request.ts:60` 决定——**二选一**，不会叠加：

```
有 agent.prompt ? → 使用 agent.prompt（完全替代 Provider 模板）
无 agent.prompt ? → 按模型 ID 选择模板（system.ts:19-33）
```

模型选模板规则：

| 模型 ID 特征 | 模板文件 |
|--------------|----------|
| `claude` | `anthropic.txt` |
| `gpt`（含 `codex`） | `gpt.txt` / `codex.txt` |
| `gemini-` | `gemini.txt` |
| `kimi` | `kimi.txt` |
| `trinity` | `trinity.txt` |
| 以上都不匹配 | `default.txt` |

Agent prompt 在 `opencode.json` 中配置，配置后 LLM 收到的提示词不再是模板内容，而是你自己定义的 prompt：

```json
{
  "agent": {
    "build": {
      "prompt": "你是 OpenCode，一个中文编程助手。始终用中文回复。"
    }
  }
}
```

---

### 问题2：AGENTS.md 默认位置在哪？opencode.json 里怎么配 instructions？

答：**自动发现**的 `AGENTS.md`（`instruction.ts:58-66`）：

| 层级 | 路径 |
|------|------|
| 全局 | `~/.config/opencode/AGENTS.md` |
| 项目 | 从当前目录向上查找第一个 `AGENTS.md` |
| 兼容 | `~/.claude/CLAUDE.md`（可通过 flags 禁用） |
| 遗留 | `CONTEXT.md`（已弃用） |

**手动配置**在 `opencode.json` 的 `instructions` 字段（`instruction.ts:133-148`）：

```json
{
  "instructions": [
    "docs/coding-standards.md",
    "../team-guide.md",
    "*.rules.md",
    "https://company.com/guidelines.md"
  ]
}
```

支持的文件格式：文件路径、glob 匹配、远程 URL。最终以 `Instructions from: <路径>\n<内容>` 的格式注入 system prompt。

---

### 问题3：Instructions 和 Skills 是怎么动态加载的？Skills 有哪些工具？

答：**每次请求时重新加载**，不是编译时静态的。

#### Instructions 加载（`instruction.ts:153-167`）

```
① 扫描 systemPaths()：
   ├─ 检查全局 AGENTS.md 是否存在
   ├─ 向上查找项目 AGENTS.md
   └─ 展开 instructions 配置中的路径/glob/URL

② 读取所有文件内容：
   ├─ 本地文件 → fs.readFileString
   └─ 远程 URL  → http fetch（5s 超时）

③ 拼装为字符串数组，每项格式：
   "Instructions from: <路径>\n<文件内容>"
```

#### Skills 加载（`system.ts:65-77`）

```
① 检查 skill 工具是否被 permission 禁用
② 调用 skill.available(agent) 获取当前可用技能
③ 拼装为描述文本注入 system prompt
```

Skills 不是通过 `opencode.json` 配置的，而是通过 `.opencode/skills/` 目录或远程 URL 注册：

```
.opencode/
└── skills/
    ├── my-skill/
    │   ├── SKILL.md
    │   └── script.ts
    └── another-skill/
        └── SKILL.md
```

也可以通过 `opencode.json` 添加外部技能路径：

```json
{
  "skills": {
    "paths": ["/path/to/custom-skills"],
    "urls": ["https://example.com/.well-known/skills/"]
  }
}
```

每个技能目录下的 `SKILL.md` 定义了技能的名称、描述和触发条件。当 LLM 识别到任务匹配技能描述时，会调用 `skill` 工具加载该技能。

---

---

## 消息历史构造与压缩

### 消息怎么从数据库变成发给 LLM 的请求

```
每轮 runLoop 开始（prompt.ts:1229）:
  filterCompactedEffect(sessionID)
      │
      ▼
  stream(sessionID)  →  从 DB 分页加载全部消息
      │
      ▼
  filterCompacted()  →  如果有压缩记录，重排消息顺序
      │
      ▼
  得到一个 WithParts[] 数组（原始格式，还不是 LLM 的格式）
      │
      ▼
  toModelMessagesEffect()  →  转换成 LLM 认识的格式（user/assistant roles, text/image parts...）
      │
      ▼
  传给 llm.stream() 发出请求
```

纯文字解释：每次 `runLoop` 循环的第一步，都是从数据库把整个 session 的消息全读出来。如果有压缩记录（之前的摘要），`filterCompacted()` 会把消息重新排序，让 LLM 看到的是"摘要 → 最近几轮 → 最新消息"的顺序。然后 `toModelMessagesEffect()` 把内部格式转换成各 provider 能识别的消息格式，才发出去。

### 重排具体怎么排

`filterCompacted()`（`message-v2.ts:533-584`）的规则：

- 没有压缩记录 → 按时间正序，不变
- 有压缩记录（`summary: true` 的 assistant + 带 `compaction` part 的 user）→ 重排为：

```
[压缩摘要用户消息, 摘要回复, ...保留的旧消息..., 最新消息]
```

保留的旧消息 = 从 `compaction part` 上记录的 `tail_start_id` 开始到压缩点之前的部分。最新消息 = 压缩点之后到末尾的部分。

### 压缩什么时候触发

不是后台定时任务，也不是手动触发，而是在 **runLoop 的每次迭代里** 自动判断。有 4 个触发点：

**触发 1：预检查**（`prompt.ts:1297-1304`）

LLM 回复完后，下一轮开始前，看上一条 assistant 记录的实际 token 消耗：

```
判断条件: 上一轮 input + output + cache >= 模型上限 - buffer（默认 20K）
```

如果超了 → 插压缩标记 → `continue` 回到循环顶重新加载。这个检查是保守的，buffer 留出了本轮新增消息 + 输出空间。

**触发 2：LLM 回复后实际检查**（`processor.ts:753-758`）

每次 LLM 成功返回，processor 用 provider 返回的 `usage.tokens` 再查一次。如果超限 → 设 `needsCompaction = true`。

**触发 3：Provider 报 context overflow**（`processor.ts:924-934`）

如果 LLM 直接抛 `ContextOverflowError`（例如 Anthropic 的 "prompt is too long"）→ 设 `needsCompaction = true`。

**触发 4：收到 `"compact"` 结果**（`prompt.ts:1451-1460`）

触发 2/3 最终都会让 processor 返回 `"compact"`，runLoop 收到后调用 `compaction.create()` → `continue`。

### 从触发到真正压缩的完整流程

```
触发（任意一种方式）
  │
  ▼
compaction.create()
  │  写入一条新 user 消息到 DB，附带 type:"compaction" part
  │  不调 LLM，纯粹"插旗"
  ▼
continue → 回到循环顶
  │
  ▼
filterCompactedEffect() 再次加载消息
  │  现在消息列表里多了一条带 compaction flag 的用户消息
  ▼
MessageV2.latest() 发现这条 flag
  │  它比最后完成的 assistant 更新 → 放进 tasks 队列
  ▼
runLoop: const task = tasks.pop()
  │  task.type === "compaction"
  ▼
compaction.process()
  │
  ├─ 选要压缩的历史（select()）：
  │   保留最近 tail_turns 轮（默认 2 轮），
  │   预算 preserve_recent_tokens（默认 usable×25%，限 2000~8000）
  │   之前的历史全压缩
  │
  ├─ 调 LLM 生成摘要（用专门的 compaction agent）
  │
  └─ 写入 summary:true 的 assistant 消息
     必要时 replay 用户原始消息
      │
      ▼
  continue → 下次 filterCompactedEffect() 重排
             旧历史被摘要替代，发给 LLM 的请求变小了
```

### 一句话总结

> 压缩 = runLoop 发现 token 快超了 → 写个标记 → 下次循环让 LLM 把旧历史浓缩成摘要 → 后续只发摘要 + 最近几轮。消息构造每次都从 DB 全量加载 + 重排，压缩和构造是同一套流程的上下游。

---

## 可用工具列表构建

### ① 工具从哪来

来源有三层，最终合并成一个 `Record<string, AISDK.Tool>`：

**内建工具**（`tool/registry.ts:246-262`）— 写死在代码里的 16 个工具：

| 工具 ID | 说明 | 文件 |
|---------|------|------|
| `shell` | 执行命令 | `tool/shell.ts` |
| `read` | 读文件 | `tool/read.ts` |
| `write` | 写文件 | `tool/write.ts` |
| `edit` | 编辑文件 | `tool/edit.ts` |
| `apply_patch` | 应用 patch | `tool/apply_patch.ts` |
| `glob` | 文件搜索 | `tool/glob.ts` |
| `grep` | 内容搜索 | `tool/grep.ts` |
| `task` | 子任务 | `tool/task.ts` |
| `question` | 问用户 | `tool/question.ts` |
| `webfetch` | 抓取网页 | `tool/webfetch.ts` |
| `websearch` | 搜索网络 | `tool/websearch.ts` |
| `skill` | 加载技能 | `tool/skill.ts` |
| `todowrite` | 任务清单 | `tool/todo.ts` |
| `lsp` | LSP 查询（实验） | `tool/lsp.ts` |
| `plan` | 计划模式退出 | `tool/plan.ts` |
| `invalid` | 错误修复回退 | `tool/invalid.ts` |

**自定义工具**（`tool/registry.ts:199-220`）— 两种注册方式：

1. **文件脚本**：在 `~/.config/opencode/` 或项目目录下的 `tool/*.ts` / `tools/*.ts` 里导出，文件名做 namespace，默认导出用文件名当 ID，命名导出用 `文件名_函数名` 当 ID。
2. **插件注册**：通过 `plugin` 系统在 `tool` 字段注册。

**MCP 工具**（`session/tools.ts:121-206`）— 每次请求从连接的 MCP Server 拉取，工具 ID 格式 `server名_工具名`。

### ② 怎么自定义工具

**最简单的方式**：在项目或全局的 `tool/` 目录下新建 `.ts` 文件：

```ts
// ~/.config/opencode/tools/weather.ts
import { z } from "zod"

export default {
  id: "get_weather",
  description: "获取指定城市的天气",
  parameters: z.object({
    city: z.string().describe("城市名"),
  }),
  execute: async ({ city }) => {
    return `天气：晴天，25°C`
  },
}
```

也可以写插件注册，或者配 MCP Server。

### ③ 怎么开启 / 禁用工具

| 方式 | 位置 | 说明 |
|------|------|------|
| Agent permission | `opencode.json agent.{name}.permission` | `{ "shell": "deny" }` 禁用，`"ask"` 每次询问，`"allow"` 放行 |
| Agent 级 tools（已弃用） | `opencode.json agent.{name}.tools` | `{ "write": false }` 会自动转为 `permission: { edit: "deny" }` |
| 全局默认权限 | `agent/agent.ts:106-123` | 所有 agent 共享：`read` 对 `.env` 为 `ask`，`question`/`plan_enter`/`plan_exit` 默认为 `deny` |
| Runtime flag | `tool/registry.ts:248-262` | `question` 仅 cli/app/desktop 可用；`lsp` 需 `experimentalLspTool`；`plan` 需 `experimentalPlanMode` + cli |
| 模型路由 | `tool/registry.ts:319-322` | GPT 非 oss/4 用 `apply_patch` 替代 `edit`/`write` |
| Provider 限制 | `tool/registry.ts:58-60` | `websearch` 仅对 opencode provider 开放（除非开 exa/parallel flag） |
| 每请求禁用 | `request.ts:202` | 调用时传 `{ tools: { websearch: false } }` 临时关掉某个工具 |

permission 的优先级：**全局默认 < agent 配置 < 运行时 flag < 每请求禁用**。只要任一环节 deny，工具就不可用。

### ④ 请求里会发工具信息吗？发什么？

**会**，每次 LLM 请求都会带上完整的工具列表。`request.ts:174` 最终拼装成这样发给 AI SDK `streamText`：

```ts
{
  tools: {
    read: {
      description: "Read a file...",
      parameters: { /* JSON Schema */ },
      execute: /* 闭包，实际执行逻辑 */,
    },
    write: { description: "Write a file...", parameters: { ... }, execute: ... },
    // ... 所有可用的工具
    _noop: { description: "No-op tool for Copilot", parameters: { ... }, execute: ... },
  }
}
```

传递给 LLM 的**实际工具信息**只有 `description` 和 `parameters`（JSON Schema）。`execute` 是 AI SDK 内部用的 JS 闭包，**不会序列化发给 provider**。

工具列表经过三层过滤后最终包含：

1. **内建工具** — 经 permission / flag / 模型路由过滤后剩余的工具
2. **自定义工具** — 文件脚本 + 插件注册的工具
3. **MCP 工具** — 当前连接的 MCP Server 暴露的工具

加上 `_noop`（Copilot 兼容，当无工具可用但历史有 tool_call 时注入，`request.ts:155-165`）和 `invalid`（保留在 tools 里但不出现在 activeTools 中，作为 AI SDK `experimental_repairToolCall` 的回退，`llm.ts:290-309`）。

过滤后工具按 ID 字母排序再发出去（`request.ts:174`）：

```ts
tools: Object.fromEntries(Object.entries(tools).toSorted(([a], [b]) => a.localeCompare(b))),
```

---

### ⑤ 怎么配置 MCP 工具

两种方式，都在 `opencode.json` 的 `mcp` 字段配：

**本地 MCP Server**（最常用）：

```jsonc
{
  "mcp": {
    "servers": {
      "my-db": {
        "type": "local",
        "command": ["node", "path/to/mcp-server.js"],
        "environment": {
          "DB_URL": "postgres://..."
        }
      }
    },
    "timeout": 10000  // 全局超时，默认 5000ms
  }
}
```

每个 server 的 id 会作为 namespace，工具最终 ID 为 `server名_工具名`。

**远程 MCP Server**：

```jsonc
{
  "mcp": {
    "servers": {
      "my-api": {
        "type": "remote",
        "url": "https://example.com/mcp",
        "headers": { "Authorization": "Bearer sk-xxx" },
        "oauth": false  // 禁用 OAuth 自动检测
      }
    }
  }
}
```

**完整配置项**（`core/src/v1/config/mcp.ts`）：

| 字段 | 类型 | 适用 | 说明 |
|------|------|------|------|
| `type` | `"local"` / `"remote"` | 必填 | 连接方式 |
| `command` | `string[]` | local | 命令+参数，如 `["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]` |
| `url` | `string` | remote | MCP server 地址 |
| `environment` | `Record<string,string>` | local | 子进程环境变量 |
| `headers` | `Record<string,string>` | remote | 请求头 |
| `oauth` | `OAuthConfig \| false` | remote | OAuth 配置，`false` 关闭自动检测 |
| `timeout` | `number` | 两者 | 超时毫秒，默认 5000 |
| `enabled` | `boolean` | 两者 | 启动时是否连接，默认 `true` |

**关闭某个 MCP Server**：

```json
{
  "mcp": {
    "servers": {
      "old-server": {
        "type": "local",
        "command": ["node", "server.js"],
        "enabled": false
      }
    }
  }
}
```

---

## Tool-Call 在 opencode 中的完整流程

### 从 LLM 回复到工具执行

```
LLM 原始回复流
  │
  ├─ 普通文本 → { type: "text-delta" } → 拼装到 assistant 消息
  │
  └─ tool-call → { type: "tool-call", toolCallId, toolName, args }
                    │
                    ▼
                  processor.ts 收到 tool-call 事件
                    │
                    ├─ 把 tool-call 写入当前 assistant 消息的 parts
                    │
                    ├─ 判断是否要继续等待更多 tool-call（stream 可能分批到达）
                    │
                    └─ LLM stream 结束后，检查是否有未执行的 tool-call
                          │
                   有 ? ──┤
                          │
                          ▼
                        SessionTools.resolve() 从已注册工具列表里按 toolName 找到对应工具
                          │
                          ├─ permission 检查（当前 agent 允许执行吗？）
                          │    ├─ "allow" → 直接执行
                          │    ├─ "deny"  → 跳过，写错误结果
                          │    └─ "ask"   → 弹权限询问 UI，等用户确认
                          │
                          ├─ 执行工具的 execute()（shell / read / write / ...）
                          │    ├─ 工具结果以 tool-result 形式写回 DB
                          │    └─ 推送 SSE 事件给前端实时更新
                          │
                          └─ 全部工具执行完毕后
                               │
                               ▼
                            回到 runLoop 顶部
                              │
                              ├─ filterCompactedEffect() 重新加载消息
                              │   （现在历史里多了 tool-call + tool-result）
                              │
                              └─ 构造请求再次调 LLM
                                  LLM 看到 tool-result 后继续推理
                                  （可能继续调工具，也可能给出最终回复）
```

### tool-call 解析的细节

AI SDK 的 `fullStream` 已经帮 opencode 消化了 provider 差异。`ai-sdk.ts:220-232` 中收到的标准化事件结构：

```ts
case "tool-call":
  // event = { type: "tool-call", toolCallId, toolName, args, ... }
  LLMEvent.toolCall({
    id: event.toolCallId,      // 唯一标识，用于关联 result
    name: event.toolName,      // 工具名，如 "read"、"shell"
    input: event.args,         // 参数对象，如 { path: "src/index.ts" }
  })
```

不关心里面是 OpenAI 的 `function.name` 还是 Anthropic 的 `tool_use.name`——AI SDK 已经归一化了。

### 工具循环什么时候结束

有两种情况 runLoop 会跳出工具循环、把最终回复返回给用户：

1. **LLM 回复里没有 tool-call** → 纯文本回复，直接 break
2. **LLM 回复 finish 原因是非 `tool-calls`** → 如 `"stop"`、`"end_turn"` 等

判断在 `prompt.ts:1248-1266`：

```ts
if (
  lastAssistant?.finish &&
  !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls &&
  lastUser.id < lastAssistant.id
) break
```

---

## 请求生命周期：一次用户输入，多次 LLM 调用

### 核心概念

一个用户请求（发一条消息 = 一个 `prompt()` 调用），runLoop 可能会调 LLM **很多次**。每次调 LLM 叫一轮（step）。

```
用户发了一条消息 "帮我重构这个文件"
  │
  ▼
runLoop while(true) 开始
  │
  ├─ step 1: 调 LLM
  │           LLM 返回: [文本, tool-call: read, tool-call: grep]
  │           finish_reason = "tool-calls"
  │           ↓
  │           执行 read → 写入 DB
  │           执行 grep → 写入 DB
  │           ↓ continue（下一轮）
  │
  ├─ step 2: 调 LLM（现在历史里多了 tool 结果）
  │           LLM 返回: [tool-call: edit]
  │           finish_reason = "tool-calls"
  │           ↓
  │           执行 edit → 写入 DB
  │           ↓ continue
  │
  ├─ step 3: 调 LLM（历史里多了 edit 的结果）
  │           LLM 返回: [文本 "重构完成"]
  │           finish_reason = "stop"
  │           ↓
  │           finish 不是 "tool-calls" → break
  │
  ▼
runLoop 结束，最终回复 "重构完成" 返回给用户
```

### 为什么需要多次调 LLM

LLM 本身是**无状态的推理引擎**——它不知道执行 `shell` 命令后的输出是什么，必须把执行结果塞回给它，才能基于结果继续推理。

每次调用 → LLM 返回结果（文本/tool-call）→ 执行工具 → 结果写回 DB → 下一轮 LLM 看到工具结果 → 继续思考。

### 一轮（step）何时结束

`processor.ts` 处理完 LLM stream 后返回结果给 runLoop：

| processor 返回 | 含义 | runLoop 动作 |
|---------------|------|-------------|
| `"continue"` | LLM 返回了 tool-call，工具已执行完 | `continue` → 下一轮 |
| `"compact"` | 上下文超限，已标记压缩 | 压缩后 `continue`（prompt.ts:1452） |
| `"stop"` | LLM 正常结束或出错 | `break` 退出循环（prompt.ts:1451） |

### 整个请求何时完成

`prompt.ts:1248-1266` 的判断——**四个条件同时满足才退出循环**：

```ts
if (
  lastAssistant?.finish &&                           // ① LLM 有 finish 原因
  !["tool-calls"].includes(lastAssistant.finish) &&  // ② 原因不是 "tool-calls"
  !hasToolCalls &&                                    // ③ 没有未执行的 tool-call
  lastUser.id < lastAssistant.id                      // ④ assistant 比最后一条 user 新
) break
```

条件 ③ `!hasToolCalls` 是关键——某些 provider 返回 `finish: "stop"` 但消息里还带着 tool-call，此时不能退出，得先执行工具。

### finish 字段的来源

跟 `tool-call` 一样，是 LLM 原始回复里的标准字段。AI SDK 归一化成 `finishReason`，opencode 存到 assistant 消息的 `finish` 上。各 provider 命名不同：

| AI SDK finishReason | OpenAI | Anthropic | 含义 |
|-------------------|--------|-----------|------|
| `"stop"` | `stop` | `end_turn` | 正常结束 |
| `"tool-calls"` | `tool_calls` | （无，看 content 里是否有 tool_use）| 要调工具 |
| `"length"` | `length` | `max_tokens` | 输出截断 |
| `"content-filter"` | `content_filter` | — | 内容过滤 |
| `"error"` | — | — | 出错 |

### 还有哪些情况会退出

| 场景 | 触发位置 |
|------|---------|
| 达到 agent 设置的 `steps` 上限 | prompt.ts:1314-1316，注入 `MAX_STEPS` 消息 |
| 结构化输出收集完毕 | prompt.ts:1432-1437 |
| 用户中断（Esc） | runLoop 被 `Effect.onInterrupt` 捕获 |
| ContextOverflowError 且 auto compaction 失败 | compaction.ts:426-434，返回 `"stop"` |
| Journal replay 异常 | prompt.ts 内各种 throw |

### 能力分界

```
LLM 的责任:     能不能输出 tool-call 格式
框架的责任:     什么时候调、调哪个、传什么参数
```

**模型侧决定"能不能"**——训练数据里有没有工具调用样本。Claude 3+、GPT-4+、Gemini 1.5+ 等现代模型在 SFT 阶段大量注入了工具调用数据，所以原生支持。古早模型或没做过工具微调的模型，即使 API 传了 `tools` 参数，它也不知道该怎么输出。

**框架侧决定"好不好"**——同一个模型，不同框架/配置下工具调用效果差异很大，因为框架决定了：

```
① 工具名 + 描述 → 模型靠它匹配"当前该用哪个"
② 参数 schema   → 模型按它生成参数，好的 schema 提高参数准确率
③ tool_choice   → "auto" vs "required" 影响模型是否主动调工具
④ 系统 prompt   → 工具使用策略引导（如"优先读文件再编辑"）
⑤ 工具数量      → 一次塞太多工具，模型容易选错
```

### 怎么提高工具调用效果

| 方式 | 具体做法 |
|------|---------|
| **优化 tool description** | 告诉模型"什么时候该用"和"注意事项"。如 `shell` 的描述里强调 command 参数需要用引号包裹路径 |
| **优化参数 schema** | 字段加 `describe()`、必要字段标 `required`、取值范围用 `enum` 约束 |
| **精简工具列表** | 不用的工具通过 permission 关掉，减少模型选择面。MCP 工具太多时尤其重要 |
| **`tool_choice: "required"`** | 强制模型每次都调工具。对编程助手场景特别有效，防止模型跳过工具直接编造答案 |
| **Agent prompt 加引导** | 比如"当需要查看文件内容时，优先使用 read 工具" |
| **善用 MCP namespace** | `server名_工具名` 的 ID 格式帮模型区分不同来源的工具 |
| **模型选择** | 不同模型工具调用各有优劣：Claude 复杂参数稳、GPT-4o 大量工具时好、开源模型需配合 grammar |
| **错误修复机制** | opencode 的 `invalid` 工具 + `experimental_repairToolCall`，在模型返回格式错误的 tool-call 时自动修复重试 |

### 一句话总结

> 工具调用 = 模型能力（能不能）× 框架引导（好不好）。前者靠选模型，后者靠写好描述、schema 和 prompt。

---

## 关键文件索引

| 步骤 | 文件 |
|------|------|
| TUI 提交 | `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` |
| HTTP handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` |
| Session 核心循环 | `packages/opencode/src/session/prompt.ts` (runLoop) |
| LLM 调用 | `packages/opencode/src/session/llm.ts` |
| AI SDK 事件解析 | `packages/opencode/src/session/llm/ai-sdk.ts` |
| 请求构建 | `packages/opencode/src/session/llm/request.ts` |
| Provider/模型解析 | `packages/opencode/src/provider/provider.ts` |
| 工具注册 | `packages/opencode/src/tool/registry.ts` |
| 工具执行 | `packages/opencode/src/tool/tool.ts` |
| SSE 事件订阅 | `packages/opencode/src/cli/cmd/tui/context/sync-v2.tsx` |
| System Prompt 模板选择 | `packages/opencode/src/session/system.ts` |
| Instructions 加载 | `packages/opencode/src/session/instruction.ts` |
| 消息加载与重排 | `packages/opencode/src/session/message-v2.ts` (filterCompacted, filterCompactedEffect) |
| 压缩编排 | `packages/opencode/src/session/compaction.ts` (create, process, isOverflow, prune) |
| 压缩判断逻辑 | `packages/opencode/src/session/overflow.ts` (isOverflow, usable) |
| Processor 内 overflow 检查 | `packages/opencode/src/session/processor.ts` (needsCompaction) |
| 工具注册（内建/自定义） | `packages/opencode/src/tool/registry.ts` |
| 工具组装（权限过滤+转 AI SDK 格式） | `packages/opencode/src/session/tools.ts` (SessionTools.resolve) |
| 请求层工具过滤 | `packages/opencode/src/session/llm/request.ts` (resolveTools) |
| 权限系统 | `packages/opencode/src/permission/index.ts` |
| Agent 定义（默认权限） | `packages/opencode/src/agent/agent.ts` |
| Agent 配置 schema | `packages/core/src/v1/config/agent.ts` |
| 权限配置 schema | `packages/core/src/v1/config/permission.ts` |
| MCP 工具 | `packages/opencode/src/mcp/index.ts` |

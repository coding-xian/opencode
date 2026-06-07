# opencode 全局配置

## 文件位置

| 文件 | 路径 | 来源 |
|------|------|------|
| `opencode.json` | `~/.config/opencode/opencode.json` | `Global.Path.config`（`core/src/global.ts:12`，基于 `XDG_CONFIG_HOME`） |
| `auth.json` | `~/.local/share/opencode/auth.json` | `Global.Path.data`（`core/src/global.ts:10`，基于 `XDG_DATA_HOME`） |

---

## opencode.json — 完整示例

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-20250514",
  "small_model": "openai/gpt-4o-mini",
  "provider": {
    "anthropic": {
      "options": { "apiKey": "sk-ant-..." }
    },
    "openai": {
      "options": { "apiKey": "sk-..." }
    }
  },
  "agent": {
    "build": { "model": "anthropic/claude-sonnet-4-20250514" },
    "plan":  { "model": "openai/gpt-5.2" },
    "explore": { "model": "openai/gpt-4o-mini" }
  }
}
```

## 模型选择字段

| 字段 | 用途 | 执行方 |
|------|------|--------|
| `model` | **主模型**，所有默认会话使用此模型 | Provider |
| `small_model` | **轻量模型**，用于标题生成、摘要、路径选择等不涉及代码修改的简单任务 | Provider |
| `agent.<name>.model` | **Agent 级覆盖**，指定某个 agent 使用不同的模型（不设则走 `model`） | Provider |

`small_model` 常使用参数量小、成本低的模型（如 GPT-4o-mini、Claude Haiku），因为它处理的都是简单任务，响应速度更快且成本更低。

`agent` 配置在 `provider.ts:1936-1968` `defaultModel()` 中生效：

```
model（默认）→ agent.build.model（build agent 覆盖）→ /model 命令（会话级覆盖）
```

## 配置加载优先级（后覆盖前）

1. `~/.config/opencode/opencode.json`
2. 远程 well-known 配置
3. `OPENCODE_CONFIG` 环境变量指定的文件
4. 项目目录向上查找的 `opencode.json`
5. 所有 `.opencode/` 目录下的配置文件
6. `OPENCODE_CONFIG_CONTENT` 环境变量
7. 托管配置（macOS MDM）

见 `packages/opencode/src/config/config.ts:305` `loadInstanceState`。

## Provider 级参数

定义见 `packages/core/src/v1/config/provider.ts:76` `ConfigProviderV1.Info`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 显示名称 |
| `npm` | string | AI SDK 包名。内置支持见 `provider.ts:108`（`@ai-sdk/openai-compatible`, `@ai-sdk/anthropic` 等）；不在内置表中则自动 npm install |
| `api` | string | API 标识符 |
| `env` | string[] | 环境变量名列表，用于自动检测该提供商已配置 |
| `whitelist` | string[] | 仅启用列表中的模型 |
| `blacklist` | string[] | 禁用列表中的模型 |
| `options.apiKey` | string | API 密钥 |
| `options.baseURL` | string | 自定义端点 URL |
| `options.enterpriseUrl` | string | GitHub Enterprise URL（copilot 认证用） |
| `options.setCacheKey` | boolean | 启用 promptCacheKey（默认 false） |
| `options.timeout` | number \| false | 完整请求超时（毫秒），false 禁用 |
| `options.headerTimeout` | number \| false | 等待响应头超时（毫秒），false 禁用 |
| `options.chunkTimeout` | number | SSE 流 chunk 超时（毫秒） |
| `models` | object | 模型定义集合，见下方 |

## Model 级参数

定义见 `packages/core/src/v1/config/provider.ts:8` `Model`：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | string | `""` | 模型显示名称 |
| `id` | string | `""` | 覆盖模型在 API 请求中使用的 ID |
| `family` | string | `""` | 模型系列 |
| `status` | enum | `"active"` | `alpha` / `beta` / `active` / `deprecated` |
| `release_date` | string | `""` | 发布日期 |
| `tool_call` | boolean | `true` | 是否支持工具调用 |
| `reasoning` | boolean | `false` | 是否支持推理（thinking） |
| `temperature` | boolean | `false` | 是否支持温度参数 |
| `attachment` | boolean | `false` | 是否支持附件 |
| `interleaved` | boolean \| object | `false` | 是否支持流式推理；可指定 `{ field: "reasoning_content" \| "reasoning_details" }` |
| `experimental` | boolean | `false` | 标记为实验性模型 |

### cost

| 字段 | 类型 | 说明 |
|------|------|------|
| `input` | number | 输入价格（$/M tokens） |
| `output` | number | 输出价格 |
| `cache_read` | number | 缓存读取价格 |
| `cache_write` | number | 缓存写入价格 |
| `context_over_200k` | object | 超过 200K 上下文时的价格，含 `input`, `output`, `cache_read`, `cache_write` |

### limit

| 字段 | 类型 | 说明 |
|------|------|------|
| `context` | number | 上下文窗口大小（token） |
| `input` | number | 单次输入最大 token |
| `output` | number | 单次输出最大 token |

### modalities

| 字段 | 类型 | 说明 |
|------|------|------|
| `input` | string[] | 支持的输入模态：`text`, `audio`, `image`, `video`, `pdf` |
| `output` | string[] | 支持的输出模态：`text`, `audio`, `image`, `video`, `pdf` |

### variants

变体配置，键为变体名，值为任意参数集合，`{ disabled: true }` 可禁用某变体。

### provider

| 字段 | 类型 | 说明 |
|------|------|------|
| `provider.npm` | string | 该模型使用独立的 npm 包（覆盖提供商级 `npm`） |
| `provider.api` | string | 该模型使用独立的 API ID |

### options / headers

- `options` — 任意键值对，透传给 AI SDK 的 model 选项
- `headers` — 额外 HTTP 请求头

## auth.json — 凭据存储

`~/.local/share/opencode/auth.json`（`src/auth/index.ts:9`）

```json
{
  "anthropic": {
    "type": "api",
    "key": "sk-ant-..."
  },
  "github-copilot": {
    "type": "oauth",
    "refresh": "ghr_...",
    "access": "gho_...",
    "expires": 1700000000
  }
}
```

### 三种认证类型

| type | 字段 | 说明 |
|------|------|------|
| `api` | `key` | API Key |
| `api` | `metadata` | 额外元数据键值对（如 Azure resourceName） |
| `oauth` | `refresh`, `access`, `expires` | OAuth 刷新令牌、访问令牌、过期时间 |
| `oauth` | `accountId` | 可选的账户 ID |
| `oauth` | `enterpriseUrl` | GitHub Enterprise URL |
| `wellknown` | `key`, `token` | 远程企业配置（从 `/.well-known/opencode` 拉取） |

### 管理命令

```bash
opencode auth add anthropic        # 交互式添加
opencode auth list                 # 列出所有凭据
opencode auth remove anthropic     # 删除
```

### 数据流

`provider.ts:1683` 在 `resolveSDK` 中合并：

```
opencode.json 的 provider.<id>.options.apiKey → auth.json 的 key → 环境变量
```

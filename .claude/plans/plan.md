# 实现计划：Claude Code 桌面版 — 多模型支持 + 友好 UI

## 需求重述

将 Claude Code 从 CLI 终端工具改造为：
1. **补齐缺失模块** — 108 个 feature-gated 模块的处理策略
2. **多模型提供商支持** — 不再绑定 Anthropic，支持 OpenAI/Google/本地模型等
3. **桌面应用** — 从 CLI 转为 GUI 桌面应用
4. **新手友好** — LLM/MCP/Skill 配置可视化、界面直观易用

---

## 风险评估

| 风险 | 级别 | 说明 |
|------|------|------|
| 108 个缺失模块不可恢复 | **高** | 这些模块在 Anthropic 内部 monorepo 中，无法从 npm 包恢复。只能 stub 或重新实现 |
| API 类型深度耦合 Anthropic SDK | **高** | `BetaMessage`、`BetaContentBlock` 等类型贯穿全代码库，剥离成本极高 |
| 流式协议差异 | **高** | Anthropic SSE 事件（`message_start`、`content_block_delta`）与 OpenAI 等完全不同 |
| 工程量巨大 | **极高** | 这是三个大型项目的叠加（模块补齐 + 多模型 + 桌面应用） |
| 许可证限制 | **高** | 原始代码归 Anthropic 所有，商用受限 |

---

## Phase 0: 项目基础设施搭建

**目标**: 建立独立项目结构，不直接修改原始源码

**原因**: 原始代码受版权保护且工程复杂，直接修改风险极高。应创建独立项目引用原始代码。

### 0.1 创建独立桌面应用项目

```
desktop-claude-code/
├── packages/
│   ├── core/              # 核心引擎（从原始代码提取/重写）
│   │   ├── src/
│   │   │   ├── providers/  # 模型提供商抽象层
│   │   │   ├── agent/      # Agent 循环引擎
│   │   │   ├── tools/      # 工具系统
│   │   │   ├── mcp/        # MCP 协议
│   │   │   ├── skills/     # 技能系统
│   │   │   └── config/     # 配置管理
│   │   └── package.json
│   ├── desktop/           # Electron/Tauri 桌面壳
│   │   ├── src/
│   │   │   ├── main/      # 主进程
│   │   │   ├── preload/   # 预加载
│   │   │   └── renderer/  # 渲染进程（React UI）
│   │   └── package.json
│   └── shared/            # 共享类型和工具
│       ├── src/
│       └── package.json
├── package.json           # Monorepo root
└── turbo.json             # Turborepo 配置
```

### 0.2 技术选型决策

| 决策点 | 推荐 | 理由 |
|--------|------|------|
| 桌面框架 | **Tauri 2.0** | 体积小（~5MB vs Electron ~150MB）、性能好、原生 Webview |
| 前端框架 | **React + TypeScript** | 与原始代码技术栈一致，可复用知识 |
| UI 组件库 | **shadcn/ui + Tailwind** | 美观、可定制、社区活跃 |
| 构建工具 | **Vite** | 快速 HMR，Tauri 原生支持 |
| Monorepo | **Turborepo + pnpm** | 高效的 monorepo 管理 |
| 包管理 | **pnpm** | 严格依赖管理，磁盘效率 |

**复杂度**: 中
**依赖**: 无

---

## Phase 1: 多模型提供商抽象层

**目标**: 创建 Provider 接口，支持 Anthropic/OpenAI/Google/Ollama/国产模型/自定义端点

### 1.1 定义 Provider 接口 — 两种兼容协议

```typescript
// packages/core/src/providers/types.ts

// 协议类型：所有提供商分为两大类
type ProviderProtocol = 'anthropic' | 'openai'

interface LLMProvider {
  id: string                          // 唯一标识
  name: string                        // 显示名称
  protocol: ProviderProtocol          // 兼容协议类型
  models: ModelInfo[]                 // 可用模型列表
  baseUrl: string                     // API 端点 URL

  // 核心方法
  chat(params: ChatParams): AsyncGenerator<StreamEvent>
  validateConfig(config: ProviderConfig): ValidationResult

  // 能力声明
  capabilities: ProviderCapabilities
}

interface ProviderConfig {
  id: string
  name: string
  protocol: ProviderProtocol          // 'anthropic' 或 'openai'
  baseUrl: string                     // 自定义 base URL
  apiKey: string                      // API Key
  models: ModelInfo[]                 // 可用模型（可自动获取）
  defaultModel?: string
}

interface ChatParams {
  messages: Message[]
  model: string
  tools?: ToolDefinition[]
  systemPrompt?: string
  thinkingConfig?: ThinkingConfig
  signal?: AbortSignal
  temperature?: number
  maxTokens?: number
}

interface StreamEvent {
  type: 'text' | 'tool_use' | 'tool_result' | 'thinking' | 'error' | 'done'
  content: string
  toolCall?: ToolCall
  usage?: TokenUsage
}

interface ProviderCapabilities {
  streaming: boolean
  toolUse: boolean
  thinking: boolean
  vision: boolean
  promptCaching: boolean
  maxTokens: number
}
```

### 1.2 协议适配层架构

关键设计：所有提供商只需实现两种协议之一。国产模型（Kimi、GLM、Minimax 等）大多兼容 OpenAI 格式，因此归入 `openai` 协议。

```
providers/
├── base/
│   ├── AnthropicProtocol.ts    # Anthropic 协议基类
│   └── OpenAIProtocol.ts       # OpenAI 协议基类
├── anthropic/
│   └── index.ts                # Anthropic 官方
├── openai/
│   └── index.ts                # OpenAI 官方
├── google/
│   └── index.ts                # Google Gemini（OpenAI 兼容）
├── ollama/
│   └── index.ts                # Ollama 本地模型（OpenAI 兼容）
├── chinese/                    # 国产模型提供商
│   ├── zhipu.ts                # 智谱 GLM（OpenAI 兼容）
│   ├── moonshot.ts             # Moonshot Kimi（OpenAI 兼容）
│   ├── minimax.ts              # Minimax（OpenAI 兼容）
│   ├── deepseek.ts             # DeepSeek（OpenAI 兼容）
│   ├── qwen.ts                 # 通义千问（OpenAI 兼容）
│   └── baichuan.ts             # 百川（OpenAI 兼容）
├── custom/
│   └── index.ts                # 自定义端点（用户自选协议 + base URL）
└── registry.ts                 # 提供商注册中心
```

### 1.3 内置预设提供商

```typescript
// packages/core/src/providers/presets.ts

const BUILTIN_PROVIDERS: ProviderPreset[] = [
  // Anthropic 系列
  { id: 'anthropic', name: 'Anthropic', protocol: 'anthropic',
    baseUrl: 'https://api.anthropic.com', models: ['claude-sonnet-4-6', 'claude-opus-4-6', 'claude-haiku-4-5'] },

  // OpenAI 系列
  { id: 'openai', name: 'OpenAI', protocol: 'openai',
    baseUrl: 'https://api.openai.com/v1', models: ['gpt-4o', 'o3', 'o4-mini'] },

  // Google
  { id: 'google', name: 'Google Gemini', protocol: 'openai',
    baseUrl: 'https://generativelanguage.googleapis.com/v1beta/openai', models: ['gemini-2.5-pro', 'gemini-2.5-flash'] },

  // 国产模型
  { id: 'zhipu', name: '智谱 GLM', protocol: 'openai',
    baseUrl: 'https://open.bigmodel.cn/api/paas/v4', models: ['glm-4-plus', 'glm-4-flash', 'glm-4v'] },
  { id: 'moonshot', name: 'Moonshot Kimi', protocol: 'openai',
    baseUrl: 'https://api.moonshot.cn/v1', models: ['moonshot-v1-128k', 'moonshot-v1-32k'] },
  { id: 'minimax', name: 'Minimax', protocol: 'openai',
    baseUrl: 'https://api.minimax.chat/v1', models: ['MiniMax-Text-01'] },
  { id: 'deepseek', name: 'DeepSeek', protocol: 'openai',
    baseUrl: 'https://api.deepseek.com/v1', models: ['deepseek-chat', 'deepseek-coder', 'deepseek-reasoner'] },
  { id: 'qwen', name: '通义千问', protocol: 'openai',
    baseUrl: 'https://dashscope.aliyuncs.com/compatible-mode/v1', models: ['qwen-max', 'qwen-plus', 'qwen-coder-plus'] },

  // 本地模型
  { id: 'ollama', name: 'Ollama (本地)', protocol: 'openai',
    baseUrl: 'http://localhost:11434/v1', models: [] },  // 自动检测
]
```

### 1.4 自定义提供商配置

用户可通过 UI 添加任意自定义提供商：
- 选择协议类型（`anthropic` 或 `openai`）
- 填写 Base URL
- 填写 API Key
- 手动输入或自动获取模型列表
- 保存后即可使用

### 1.5 Provider 注册中心

```typescript
// packages/core/src/providers/registry.ts

class ProviderRegistry {
  private providers: Map<string, LLMProvider>

  register(preset: ProviderPreset, config: ProviderConfig): LLMProvider
  unregister(id: string): void
  get(id: string): LLMProvider
  list(): ProviderInfo[]

  // 测试连接
  async testConnection(id: string): Promise<ConnectionResult>

  // 自动发现模型列表
  async discoverModels(id: string): Promise<ModelInfo[]>
}
```

**复杂度**: 高
**依赖**: Phase 0

---

## Phase 2: Agent 引擎核心重写

**目标**: 从原始代码提取 agent 循环核心，解耦 Anthropic 依赖

### 2.1 统一消息格式

```typescript
// packages/core/src/types/message.ts

interface Message {
  role: 'user' | 'assistant' | 'system'
  content: MessageContent[]
}

type MessageContent =
  | { type: 'text'; text: string }
  | { type: 'tool_use'; id: string; name: string; input: Record<string, unknown> }
  | { type: 'tool_result'; tool_use_id: string; content: string; is_error?: boolean }
  | { type: 'thinking'; text: string }
  | { type: 'image'; data: string; media_type: string }
```

### 2.2 Agent 循环

从 `src/query.ts` 提取核心逻辑，改为：

```typescript
// packages/core/src/agent/loop.ts

async function* agentLoop(
  provider: LLMProvider,
  messages: Message[],
  tools: Tool[],
  options: AgentOptions
): AsyncGenerator<AgentEvent> {
  while (true) {
    // 1. 调用 provider.chat()
    // 2. 处理流式响应
    // 3. 如果 stop_reason === 'tool_use'，执行工具
    // 4. 追加工具结果到消息
    // 5. 如果 stop_reason === 'end_turn'，退出
  }
}
```

### 2.3 工具系统提取

从 `src/Tool.ts` 和 `src/tools/` 提取工具接口和核心工具：

- **保留的核心工具**: BashTool, FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool, AgentTool
- **新增工具**: WebSearchTool（内置，不依赖 MCP）
- **MCP 工具**: 动态加载，保持原有机制

### 2.4 上下文管理

- 提取 `src/services/compact/` 的压缩逻辑
- 实现基础的上下文窗口管理
- 为不同模型适配不同的 token 限制

**复杂度**: 高
**依赖**: Phase 1

---

## Phase 3: 桌面应用壳

**目标**: 搭建 Tauri 桌面应用框架

### 3.1 Tauri 主进程

```
packages/desktop/src/main/
├── main.ts           # Tauri 入口
├── commands/         # Tauri IPC commands
│   ├── agent.ts      # Agent 控制（启动/停止/发送消息）
│   ├── config.ts     # 配置管理
│   ├── mcp.ts        # MCP 管理
│   ├── files.ts      # 文件系统操作
│   └── shell.ts      # 终端操作
└── tray.ts           # 系统托盘
```

### 3.2 权限与安全

- 文件系统访问通过 Tauri 命令受控暴露
- Shell 命令需要用户确认弹窗
- API Key 存储使用系统 Keychain（macOS）/ Credential Manager（Windows）

### 3.3 进程间通信

```typescript
// Tauri IPC 命令示例
invoke('agent:start', { providerId, model, workspacePath })
invoke('agent:send_message', { message })
invoke('agent:stop')
invoke('config:get_providers')
invoke('config:save_provider', { provider })
invoke('mcp:list_servers')
invoke('mcp:add_server', { config })
```

**复杂度**: 中
**依赖**: Phase 0

---

## Phase 4: 桌面 UI 开发

**目标**: 构建友好的用户界面

### 4.1 页面结构

```
packages/desktop/src/renderer/
├── App.tsx
├── layouts/
│   ├── MainLayout.tsx         # 主布局（侧边栏 + 内容区）
│   └── SettingsLayout.tsx     # 设置页布局
├── pages/
│   ├── ChatPage.tsx           # 对话主页
│   ├── SettingsPage.tsx       # 设置总览
│   ├── ProviderConfigPage.tsx # 模型提供商配置
│   ├── McpConfigPage.tsx      # MCP 服务器配置
│   ├── SkillConfigPage.tsx    # 技能配置
│   └── OnboardingPage.tsx     # 新手引导
├── components/
│   ├── chat/
│   │   ├── ChatView.tsx       # 对话视图
│   │   ├── MessageList.tsx    # 消息列表
│   │   ├── MessageBubble.tsx  # 单条消息
│   │   ├── InputBar.tsx       # 输入栏
│   │   ├── ToolResult.tsx     # 工具执行结果展示
│   │   └── CodeBlock.tsx      # 代码块（语法高亮）
│   ├── settings/
│   │   ├── ProviderCard.tsx   # 提供商配置卡片
│   │   ├── ApiKeyInput.tsx    # API Key 输入（带验证）
│   │   ├── ModelSelector.tsx  # 模型选择下拉
│   │   ├── McpServerCard.tsx  # MCP 服务器卡片
│   │   └── SkillCard.tsx      # 技能卡片
│   ├── workspace/
│   │   ├── FileTree.tsx       # 文件树
│   │   └── TerminalPanel.tsx  # 内嵌终端（xterm.js）
│   └── common/
│       ├── Sidebar.tsx        # 侧边栏
│       ├── StatusBadge.tsx    # 状态标识
│       └── ConfirmDialog.tsx  # 确认对话框
└── hooks/
    ├── useAgent.ts            # Agent 状态管理
    ├── useProvider.ts         # 提供商状态
    └── useConfig.ts           # 配置管理
```

### 4.2 核心界面设计

**对话页面**:
- 左侧：历史会话列表 + 工作区文件树
- 中间：对话消息流（支持 Markdown 渲染、代码高亮、工具调用可视化）
- 底部：输入栏（支持多行、文件拖放、@提及工具）
- 右上角：模型选择器（快速切换提供商/模型）

**模型配置页面**:
- 卡片式布局，每个提供商一张卡片
- 填写 API Key 后自动验证连接
- 显示可用模型列表、Token 价格
- 支持自定义端点（OpenAI 兼容格式）

**MCP 配置页面**:
- 可视化添加/编辑/删除 MCP 服务器
- 支持导入 `.mcp.json`
- 实时显示连接状态
- 按服务器显示可用工具列表

**技能配置页面**:
- 显示已安装技能列表
- 支持启用/禁用
- 技能市场（浏览安装新技能）

### 4.3 新手引导流程

1. **欢迎页** — 选择语言
2. **配置模型** — 引导选择提供商、输入 API Key
3. **选择工作区** — 选择项目目录
4. **开始使用** — 简短的功能介绍

**复杂度**: 高
**依赖**: Phase 2, Phase 3

---

## Phase 5: 配置管理系统

**目标**: 完善配置持久化与管理

### 5.1 配置文件结构

```json
// ~/.desktop-claude-code/config.json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-...",
      "models": ["claude-sonnet-4-6", "claude-opus-4-6"],
      "default": "claude-sonnet-4-6"
    },
    "openai": {
      "apiKey": "sk-...",
      "endpoint": "https://api.openai.com/v1",
      "models": ["gpt-4o", "o3"],
      "default": "gpt-4o"
    },
    "ollama": {
      "endpoint": "http://localhost:11434",
      "models": ["llama3", "codellama"],
      "default": "llama3"
    },
    "custom": [
      {
        "id": "deepseek",
        "name": "DeepSeek",
        "endpoint": "https://api.deepseek.com/v1",
        "apiKey": "...",
        "models": ["deepseek-coder"]
      }
    ]
  },
  "activeProvider": "anthropic",
  "mcpServers": { ... },
  "skills": { ... },
  "workspace": { ... }
}
```

### 5.2 配置迁移

- 支持从现有 Claude Code 配置（`~/.claude.json`、`.mcp.json`）导入
- 自动检测已安装的 Claude Code 配置

**复杂度**: 中
**依赖**: Phase 1

---

## Phase 6: 108 个缺失模块处理策略

**目标**: 确定每个缺失模块的处理方式

### 6.1 分类处理

**A. 无需处理（保持 stub）— 约 70 个**

所有 `feature()` 门控模块在发布版本中本就是禁用的。这些模块的代码路径永远不会被执行。包括：
- `daemon/*` — 后台守护进程
- `proactive/*` — 主动通知
- `assistant/*` — KAIROS 自主模式
- `bridge/peerSessions` — 多人协作
- 所有 `commands/` 下的门控命令
- 所有 Anthropic 内部工具（`ant` 门控）
- 遥测模块

**B. 需要重新实现（高价值功能）— 约 8 个**

| 模块 | 功能 | 优先级 |
|------|------|--------|
| `compact/snipCompact` + `snipProjection` | 智能上下文裁剪 | 高 |
| `compact/reactiveCompact` | 响应式压缩 | 中 |
| `coordinator/workerAgent` | 多 Agent 协调 | 中 |
| `WebBrowserTool` | 浏览器自动化 | 高 |
| `VerifyPlanExecutionTool` | 计划验证 | 低 |
| `contextCollapse/*` | 上下文折叠 | 中 |

**C. 需要创建简化替代 — 约 5 个**

| 原始模块 | 替代方案 |
|----------|----------|
| `yolo-classifier-prompts/*` | 自主模式权限分类器（简化版） |
| `commands/workflows/*` | 简单工作流执行器 |
| `commands/fork/*` | 子 Agent 分支命令 |

### 6.2 实现优先级

1. **P0**: stub 所有 108 个模块确保构建通过
2. **P1**: 重新实现上下文压缩（`compact/*`）
3. **P2**: 实现 WebBrowserTool（使用 Playwright）
4. **P3**: 多 Agent 协调器
5. **P4**: 其他功能按需实现

**复杂度**: 高（整体），低（单个 stub）
**依赖**: Phase 0

---

## 执行顺序与里程碑

```
Phase 0: 项目基础设施 ──────────────────── 1 周
    ↓
Phase 3: Tauri 桌面壳 ──────────────────── 1 周
    ↓ (并行)
Phase 1: 多模型抽象层 ──────────────── 2 周
    ↓
Phase 2: Agent 引擎核心 ─────────────── 2 周
    ↓
Phase 5: 配置管理系统 ──────────────── 1 周
    ↓
Phase 4: 桌面 UI 开发 ───────────────── 3-4 周
    ↓
Phase 6: 缺失模块处理（持续） ──────────── 持续
```

**总预估**: 10-12 周（核心功能），之后持续迭代

---

## 立即可做的第一步

1. **初始化 monorepo 项目** — `pnpm create turbo`
2. **搭建 Tauri 2.0 骨架** — `pnpm tauri init`
3. **创建 Provider 接口定义** — 从 `src/services/api/client.ts` 和 `src/utils/model/providers.ts` 出发
4. **实现 Anthropic 适配器** — 作为第一个 provider，验证架构可行性

---

## 替代方案考虑

### 替代方案 A: 直接修改原始代码

**优点**: 不需要重写，工作量较小
**缺点**: 版权风险、代码理解成本高、108 个缺失模块无法恢复、维护困难
**结论**: 不推荐

### 替代方案 B: Electron 替代 Tauri

**优点**: Node.js 原生支持、更成熟的生态、可直接使用原始代码的 Node 依赖
**缺点**: 体积大（~150MB）、内存占用高、安全性较差
**结论**: 如果需要快速原型验证可以考虑，长期推荐 Tauri

### 替代方案 C: Web 应用（非桌面）

**优点**: 跨平台、无需安装、部署简单
**缺点**: 文件系统访问受限、无法运行 Shell 命令、安全限制多
**结论**: 不适合 Claude Code 的核心功能需求

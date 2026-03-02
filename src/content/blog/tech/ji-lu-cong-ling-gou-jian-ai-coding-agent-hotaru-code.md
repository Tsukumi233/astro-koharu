---
title: 记录从零构建 AI Coding Agent：Hotaru Code
link: ji-lu-cong-ling-gou-jian-ai-coding-agent-hotaru-code
date: 2026-03-02 21:45:47
tags:
  - ai
  - agent
  - 开发
categories:
  - 技术
---
# 从零构建 AI Coding Agent：Hotaru Code

> ==人与人之间最大的差异不是智力，而是他们在社会与环境中积累的上下文。== 如果一个人完全指导另一个人的上下文（人际关系、项目资料、遇到的各种事情等等），那么这个人大概率也可以替代了另一个人的大部分工作。高中时候读费孝通的《乡土中国》，书中有这样的观点：乡下人的行为模式并不是愚蠢，而是适应了他们的环境；城里人的行为模式也不是聪明，让他们去地里辨认庄稼也答不上来。这句话放到数十年后的 AI 时代来看更有深意。AI Agent 也是一样的道理——它的能力不仅仅是由模型决定的，还是由它的上下文（工具、历史等）塑造的。理解和设计好这个上下文，才是构建优秀 AI Agent 的关键。

> 本文是我开发 Hotaru Code 过程中的技术总结。Hotaru Code 是一个支持 TUI、WebUI 和 CLI 的 AI 编程助手，具备完整的 Agent Loop、工具调用、权限控制、MCP 协议集成和上下文压缩能力。本文将从架构设计到实现细节，完整剖析一个生产级 AI Agent 的全链路。

## 为什么要自己造一个？

作为 Claude Code 的重度用户，我很好奇它是如何做到这么优秀的。我经常想：如果自己从零开始构建一个 AI coding agent，会是什么样子？市面上虽然有一些开源的 agent 框架，但它们要么过于简化，难以扩展，要么抽象过于复杂。我决定只用普通的工具和库，自己实现一个专注于 coding 的 agent，来验证我的设计想法。

Hotaru Code 就是这些想法的落地。Python 3.12+，asyncio 全异步，SQLite WAL 持久化，约 2 万行核心代码。

## 整体架构：分层与解耦

```plain
┌─────────────────────────────────────────────────┐
│              Interface Layer                     │
│   TUI (Textual)  │  WebUI (React)  │  CLI Run   │
├─────────────────────────────────────────────────┤
│              Transport Layer                     │
│   SDKContext → FastAPI Server (SSE + WebSocket)  │
├─────────────────────────────────────────────────┤
│           Session Orchestration                  │
│   SessionPrompt → Processor → TurnRunner         │
│   ToolExecutor → Compaction                      │
├─────────────────────────────────────────────────┤
│              LLM Adapter                         │
│   ProviderTransform → AnthropicSDK / OpenAISDK   │
├─────────────────────────────────────────────────┤
│           Tool & Extension Layer                 │
│   ToolRegistry │ Permission │ MCP │ Skill        │
├─────────────────────────────────────────────────┤
│            Core Infrastructure                   │
│   ConfigManager │ Event Bus │ SQLite Storage     │
└─────────────────────────────────────────────────┘
```

核心设计原则：**上层依赖下层，同层之间通过 Event Bus 通信**。Session 层不知道自己跑在 TUI 还是 WebUI 里，Tool 层不关心 LLM 返回的是 Anthropic 还是 OpenAI 格式。这种解耦让每一层都可以独立测试和替换。

## 三端合一：一个 SessionPrompt 统治一切

三个入口最终汇聚到同一个函数：

```plain
TUI:  InputWidget → SessionScreen → SessionPrompt.prompt()
WebUI: POST /v1/sessions/{id}/messages → SessionService → SessionPrompt.prompt()
CLI:  hotaru run "msg" → run_command() → SessionPrompt.prompt()
```

这是经典的门面模式（Facade）。`SessionPrompt` 是整个系统的编排中心，负责：

1. 调用 `prepare_prompt_context()` 解析 provider/model、创建或恢复 session、构建系统提示词
2. 构造 `SessionProcessor` 并启动 agent loop
3. 循环结束后检查上下文溢出、生成标题、持久化消息

TUI 和 WebUI 的差异仅在传输层：TUI 通过 `SDKContext` 封装的 HTTP client 与内嵌的 FastAPI server 通信，WebUI 直接走 HTTP/SSE。这意味着 TUI 本质上也是一个"前端"，和 React WebUI 地位平等。

这个设计的好处是显而易见的——任何新增的前端（比如未来的 VS Code 插件）只需要对接 REST API，不需要碰任何业务逻辑。

## Agent Loop：LLM 驱动的自主循环

Agent Loop 是整个系统的心脏。它不是简单的"发请求-收回复"，而是一个 LLM 自主决策的循环：

```python
# SessionProcessor.process() 的核心逻辑（简化）
while result.status == "continue" and self.turn < self.max_turns:
    # 1. 准备本轮输入：解析可用工具，构建 StreamInput
    stream_input = self.turnprep.prepare(agent, provider_id, model_id, messages, ruleset)

    # 2. 流式调用 LLM
    async for chunk in self.turnrun.run(stream_input, observer):
        if chunk.type == "text":
            observer.on_text(chunk.text)          # 实时推送文本
        elif chunk.type == "tool_call_end":
            tool_result = await self.tools.execute(chunk.tool_call)  # 执行工具
            messages.append(tool_result)

    # 3. 有工具调用 → 继续循环；没有 → 停止
    if has_tool_calls:
        self.turn += 1
        continue
    result.status = "stop"
```

这就是 ReAct（Reasoning + Acting）模式的实现。LLM 在每一轮可以选择：
- 输出文本（思考/回复用户）
- 调用工具（执行操作）
- 两者兼有

关键的设计决策：

**为什么用流式而不是一次性返回？** 因为工具调用可能需要用户授权。如果等整个响应返回再处理，用户体验会很差——他们看不到 AI 在"思考什么"。流式处理让文本实时显示，工具调用一完成就立即执行。

**为什么 max_turns 默认 100？** 这是安全阀。复杂任务（比如重构整个模块）可能需要几十轮工具调用，但不应该无限循环。100 轮足够完成绝大多数任务，同时防止失控。

**TurnPreparer 做了什么？** 它不只是简单地把工具列表传给 LLM。它会：
- 根据当前 agent 的配置过滤可用工具
- 合并 MCP 工具和内置工具
- 通过 `ToolResolver` 生成 OpenAI 格式的 tool definitions
- 处理 agent 切换（比如从 build agent 切到 plan agent）

## LLM 适配层：一套代码接所有 Provider

LLM 世界的现实是：每家 provider 的 API 格式都不一样。Anthropic 用 `tool_use`/`tool_result` 块，OpenAI 用 `function` 调用。Moonshot 把推理过程放在 `reasoning_content` 字段，Anthropic 用 `thinking` 块。

`ProviderTransform` 是解决这个问题的核心：

```python
class ProviderTransform:
    """五步转换管线"""

    # 1. 规范化 tool_call_id（Mistral 要求 9 位字母数字，Claude 不允许特殊字符）
    # 2. 处理 interleaved reasoning（不同 provider 的推理字段名不同）
    # 3. 重映射 provider_options（通过 sdk_key 解析到正确的 SDK）
    # 4. 注入缓存控制（Anthropic: cacheControl, OpenAI: cache_control, Bedrock: cachePoint）
    # 5. 清理空内容（Anthropic 会拒绝空 content 的消息）
```

转换完成后，根据 `api_type` 分发到 `AnthropicSDK` 或 `OpenAISDK`。两个 SDK wrapper 都输出统一的 `StreamChunk`：

```python
@dataclass
class StreamChunk:
    type: str       # "text", "tool_call_start", "tool_call_end", "reasoning_delta", ...
    text: str | None
    tool_call: ToolCall | None
    usage: dict[str, int] | None
    stop_reason: str | None
```

上层代码完全不需要知道底层用的是哪个 provider。这种归一化的代价是 `ProviderTransform` 本身比较复杂（需要处理各种边界情况），但收益是巨大的——新增一个 provider 只需要在 transform 里加几行映射规则。

`SessionRetry` 提供指数退避重试，处理 429（rate limit）和 5xx 错误。重试逻辑和业务逻辑完全分离，通过组合而非继承实现。

## 工具系统：三层叠加的可扩展架构

工具是 Agent 的手和脚。Hotaru 的工具体系分三层：

**第一层：内置工具（20+）**

```plain
文件操作: list, glob, grep, read, edit, write, multiedit, apply_patch
执行环境: bash, task (子 agent 委派)
交互控制: question, todoread, todowrite, plan_enter, plan_exit
网络能力: webfetch, websearch
扩展集成: skill, lsp
```

每个工具是一个 `ToolInfo` dataclass，包含四个核心要素：

```python
@dataclass
class ToolInfo:
    id: str                          # 工具标识
    description: str                 # 给 LLM 看的描述
    parameters_type: type[BaseModel] # Pydantic 参数模型（自动校验）
    execute: Callable                # 实际执行函数
    permissions: Callable            # 返回所需权限列表
```

用 Pydantic 做参数校验是一个关键决策。LLM 生成的 JSON 参数经常有类型错误（比如把数字传成字符串），Pydantic 的 coercion 能自动修正大部分情况，省去了大量手动校验代码。

**第二层：MCP 工具**

MCP（Model Context Protocol）是 Anthropic 提出的工具扩展协议。Hotaru 实现了完整的 MCP client，支持两种传输：

- `stdio`：启动子进程，通过 stdin/stdout 通信（本地工具）
- `HTTP/SSE`：连接远程 MCP server（支持 OAuth 认证流程）

MCP 工具通过命名空间隔离避免冲突：`{client_name}__{tool_name}`。比如 `filesystem__read_file` 和内置的 `read` 不会混淆。

```python
# MCPManager 的连接策略
async def _connect_remote(self, name, config):
    # 先尝试 streamable HTTP（新协议）
    # 失败则回退到 SSE（旧协议）
    # 遇到 401/403 则触发 OAuth 流程
```

**第三层：自定义工具**

用户可以在项目目录的 `tool/` 或 `tools/` 下放 Python 文件，`ToolRegistry` 会自动发现并加载。约定优于配置——导出 `TOOLS` 列表、`tool` 函数、`Tool` 类或 `register_tools()` 函数均可。

## 权限系统：Human-in-the-Loop 的工程实现

让 AI agent 自动执行代码是危险的。权限系统是安全的最后一道防线。

**规则引擎**

权限规则由三部分组成：`permission`（权限类型）、`pattern`（glob 模式）、`action`（allow/deny/ask）。

```json
{
  "permission": {
    "bash": "ask",
    "edit": { "src/**": "allow", "*.env": "deny" },
    "read": { "*.env": "ask", "*.env.example": "allow" }
  }
}
```

匹配逻辑是 **last-match-wins**——和 CSS 优先级类似，后面的规则覆盖前面的。这比 first-match 更灵活，因为用户可以先设一个宽泛的 deny，再用更具体的 pattern 开白名单。

规则来源按优先级合并：agent 默认规则 → 用户配置规则 → 运行时已批准规则。

**异步授权流程**

当规则匹配到 `ask` 时，系统不会阻塞整个进程，而是：

```python
async def ask(self, *, session_id, permission, patterns, ruleset, ...):
    # 1. 创建 asyncio.Future
    future = asyncio.get_event_loop().create_future()
    self._pending[request_id] = future

    # 2. 通过 Bus 发布 PermissionAsked 事件（UI 层订阅并弹出对话框）
    await Bus.publish(PermissionAsked, PermissionAskedProperties(...))

    # 3. await Future（直到用户回复）
    return await future
```

用户的回复有三种：
- `once`：仅本次放行
- `always`：记住这个规则（作用域可配置：turn/session/project/persisted）
- `reject`：拒绝，并且拒绝当前 session 所有 pending 的权限请求

`always` 的实现特别巧妙——它不仅 resolve 当前的 Future，还会扫描所有 pending 请求，自动 resolve 匹配同一 pattern 的其他请求。这意味着如果 AI 连续调用了 5 次 `bash`，用户只需要批准一次。

**Doom Loop 检测**

LLM 有时会陷入死循环——反复调用同一个工具，传同样的参数。`DoomLoopDetector` 通过签名匹配检测这种情况：

```python
# 签名 = tool_name + JSON(sorted_input)
signature = f"{tool_name}:{json.dumps(tool_input, sort_keys=True)}"

# 最近 N 次调用中，如果连续 threshold 次签名相同 → 触发权限询问
recent = self.signatures[-self.threshold:]
if len(set(recent)) == 1:
    await self.permission.ask(permission="doom_loop", ...)
```

默认阈值是 3 次，滑动窗口 50 次。这不是简单地阻止重复调用（有些场景确实需要重复），而是把决定权交给用户。

## 上下文压缩：让 Agent 拥有"长期记忆"

LLM 的上下文窗口是有限的。一个复杂任务可能涉及几十轮工具调用，每轮的输入输出都会累积 token。不做压缩，很快就会撞到上限。

Hotaru 的压缩策略分两级：

**第一级：裁剪（Prune）—— 零成本**

```python
class SessionCompaction:
    PRUNE_MINIMUM = 20_000    # 至少裁剪 20k tokens 才值得动手
    PRUNE_PROTECT = 40_000    # 保护最近 40k tokens 不被裁剪
    PRUNE_PROTECTED_TOOLS = {"skill"}  # 某些工具输出永不裁剪
```

裁剪的目标是旧的工具输出——它们通常是最大的 token 消耗者（一次 `grep` 可能返回几千行）。裁剪只是把 `ToolPart` 的内容标记为 `compacted`，替换为 `"[Old tool result content cleared]"`。不需要调用 LLM，零额外成本。

**第二级：摘要（Summarize）—— 需要 LLM 调用**

如果裁剪后仍然溢出，系统会插入一个 `CompactionPart` 标记，然后调用专门的 compaction agent 生成对话摘要。后续加载历史时，`filter_compacted()` 会跳过标记之前的所有消息，只保留摘要。

```python
@classmethod
async def is_overflow(cls, *, tokens: TokenUsage, model: ProcessedModelInfo) -> bool:
    # usable = input_limit - reserved_buffer(默认 20k 或 model.limit.output)
    # 当 total_tokens >= usable 时触发
    usable = model.limit.input - reserved
    return tokens.total >= max(usable, 1)
```

为什么分两级？因为摘要需要额外的 LLM 调用（成本和延迟），而裁剪是免费的。大多数情况下，裁剪就够了——工具输出占了 token 的大头，清理掉旧的输出通常能释放足够空间。只有在对话本身非常长的时候才需要摘要。

## Event Bus：解耦的粘合剂

在一个多端架构中，组件之间的通信是最容易变成意大利面条的地方。Hotaru 用 `ContextVar` 作用域的事件总线解决这个问题：

```python
# 定义事件（类型安全，Pydantic schema）
class PermissionAskedProperties(BaseModel):
    id: str
    session_id: str
    permission: str
    patterns: list[str]

PermissionAsked = BusEvent.define("permission.asked", PermissionAskedProperties)

# 发布
await Bus.publish(PermissionAsked, PermissionAskedProperties(...))

# 订阅
unsubscribe = Bus.subscribe(PermissionAsked, on_permission_asked)
```

`ContextVar` 作用域意味着每个请求/session 有自己独立的 Bus 实例，不会串台。这在 WebUI 场景下至关重要——多个用户同时使用时，A 的权限请求不会弹到 B 的界面上。

事件总线承担了几个关键职责：
- 权限请求/回复的异步通信
- 消息更新的实时推送（Session → SSE → WebUI）
- MCP 工具变更通知
- PTY 输出转发

## 存储层：SQLite WAL 的务实选择

为什么选 SQLite 而不是 PostgreSQL 或 Redis？

1. **零部署成本**：用户 `pip install` 就能用，不需要额外启动数据库服务
2. **WAL 模式**：支持并发读写，读操作不阻塞写操作
3. **足够快**：对于单用户/少量并发的场景，SQLite 的性能绰绰有余

存储层通过命名空间路由实现表分离：

```python
# key[0] 决定写入哪张表
"sessions"           → sessions 表
"session_index"      → session_index 表
"messages"           → messages 表
"parts"              → parts 表
"permission_approval" → permission_approval 表
其他                  → kv 表（通用键值存储）
```

`update()` 方法使用 `BEGIN IMMEDIATE` 事务，确保读-改-写的原子性。这比乐观锁简单得多，在 SQLite 的单写者模型下也不会有性能问题。

## 配置系统：五层合并的优先级链

一个好的配置系统需要同时满足"开箱即用"和"深度定制"。Hotaru 的 `ConfigManager` 从五个来源加载配置，按优先级从低到高深度合并：

```plain
1. ~/.config/hotaru/hotaru.json          ← 全局用户配置
2. hotaru.json / hotaru.jsonc            ← 项目根目录（向上查找）
3. .hotaru/hotaru.json                   ← 项目私有配置（向上查找）
4. HOTARU_CONFIG_CONTENT 环境变量          ← JSON 字符串覆盖
5. 托管配置目录                            ← 最高优先级（企业场景）
```

配置值支持 `{env:VAR}` 占位符，在加载时解析为环境变量。这让 API key 可以安全地写在配置文件里而不暴露明文：

```json
{ "options": { "apiKey": "{env:OPENAI_API_KEY}" } }
```

`ConfigManager` 使用 `ContextVar` 实现实例隔离——测试中可以注入不同的配置而不影响全局状态。最终合并结果通过 Pydantic 校验，类型错误在启动时就能发现。

## 子 Agent 委派：Task Tool 的设计

复杂任务往往需要分而治之。`TaskTool` 允许 LLM 将子任务委派给专门的 agent：

```plain
父 Session (build agent)
  → LLM 决定委派
  → TaskTool(subagent_type="explore", prompt="找到所有 API 路由")
  → 创建子 Session（独立 session_id，同一 project）
  → 子 Processor 运行（受限工具集：grep, glob, read, bash）
  → 子 agent 完成 → 文本结果注入父 Session 的 tool_result
```

子 agent 有自己独立的工具集和权限规则，运行在隔离的 session 中。这意味着 explore agent 不能调用 edit/write（只读），plan agent 不能执行 bash。隔离既是安全措施，也是让 LLM 更专注的手段——工具越少，LLM 越不容易分心。

## WebUI：React + SSE 的实时架构

WebUI 的前端是标准的 React SPA，但通信模式值得一提：

- **REST API**：session CRUD、provider 列表、权限回复等操作
- **SSE（Server-Sent Events）**：消息流、工具执行状态的实时推送
- **WebSocket**：终端（xterm.js）的双向通信

为什么用 SSE 而不是全部走 WebSocket？因为 AI 回复本质上是单向流——服务端推送文本块到客户端。SSE 天然适合这个场景，而且比 WebSocket 更简单（自动重连、HTTP 兼容、不需要心跳）。WebSocket 只用在真正需要双向通信的终端场景。

前端的 hooks 设计也值得一提：

```typescript
useSession()      // session CRUD
useMessages()     // 消息列表 + 增量更新
useEvents()       // SSE 订阅，驱动整个实时更新
usePermissions()  // 权限对话框的状态管理
usePty()          // 终端生命周期
```

`useEvents` 是核心——它订阅 SSE 流，根据事件类型分发到其他 hooks。消息更新是增量的（delta），不是每次都拉全量，这对长对话的性能至关重要。

## AppContext：分阶段启动与优雅降级

一个有 20+ 子系统的应用，启动顺序很重要。`AppContext` 实现了分阶段启动：

```plain
Phase A: 绑定 ContextVar（Bus, ConfigManager）
Phase B: 加载配置 + 初始化 SQLite（WAL 模式，自动迁移）
Phase C: 注册内置工具（同步，快速）
Phase D: MCP + LSP 并行初始化（带健康追踪）
    ├── MCP 失败 → critical，回滚启动
    └── LSP 失败 → degraded，继续运行（LSP 是增强功能，不是核心）
Phase E: Skill + Agent 并行发现
```

关键设计：MCP 和 LSP 并行初始化（`asyncio.gather`），但健康等级不同。MCP 是工具系统的一部分，失败意味着用户配置的工具不可用，必须报错。LSP 只是代码智能提示的增强，降级运行完全可以接受。

这种分级健康追踪避免了"全有或全无"的脆弱性——系统能在部分子系统故障时继续提供核心功能。

## 开发中的经验与教训

**1. 延迟导入解决循环依赖**

`session/__init__.py` 用了一个巧妙的模式——lazy exports：

```python
_EXPORTS = {
    "SessionPrompt": (".prompting", "SessionPrompt"),
    "SessionProcessor": (".processor", "SessionProcessor"),
    ...
}

def __getattr__(name):
    module_name, attr_name = _EXPORTS[name]
    module = import_module(module_name, __name__)
    value = getattr(module, attr_name)
    globals()[name] = value  # 缓存，下次直接返回
    return value
```

session 包内部有大量交叉引用（processor 引用 compaction，compaction 引用 session，session 引用 processor）。传统的 `from .processor import SessionProcessor` 会导致循环导入。lazy export 把导入推迟到实际使用时，彻底解决了这个问题。

**2. 不要用 try/except 掩盖问题**

这是我在 CLAUDE.md 里写的第一条规则，也是开发过程中最深刻的教训。早期版本里到处是 `try/except Exception: pass`，看起来程序"不会崩溃"，实际上是把 bug 藏起来了。

一个真实的例子：MCP 连接偶尔超时，早期的做法是 catch 住然后返回空工具列表。结果用户发现"工具有时候能用有时候不能用"，排查了很久才发现是超时被吞了。正确的做法是让超时错误冒泡到 AppContext 的健康追踪系统，标记为 degraded 并通知用户。

**3. ContextVar 是多租户的利器**

Bus 和 ConfigManager 都用 `ContextVar` 实现实例隔离。这在测试中尤其有用——每个测试用例可以注入自己的 Bus 和 Config，完全不需要 mock。在 WebUI 的多用户场景下，每个请求的 ContextVar 自然隔离，不需要额外的租户管理代码。

**4. Pydantic 既是校验器也是文档**

工具参数用 Pydantic model 定义，一举三得：
- LLM 看到的是自动生成的 JSON Schema（tool definition）
- 运行时自动校验和类型转换
- 开发者看到的是带类型标注的 Python 类

不需要手写 JSON Schema，不需要手写校验逻辑，不需要维护文档。一个 Pydantic class 搞定一切。

## 全链路追踪：一条消息的完整旅程

最后，用一条用户消息的完整生命周期串联所有模块，作为全文的总结：

```plain
1. 用户在 WebUI 输入 "帮我重构 utils.py"
   → React ChatView → api.sessions.send() → POST /v1/sessions/{id}/messages

2. FastAPI 路由 → SessionService.send_message() → SessionPrompt.prompt()
   → prepare_prompt_context(): 解析 model=claude-sonnet, agent=build
   → Session.get(): 从 SQLite 加载会话历史
   → SystemPrompt.build_full_prompt(): 拼接系统提示词 + agent 指令

3. SessionProcessor.process(user_message)
   → Turn 1: TurnPreparer 解析 22 个可用工具
   → LLM.stream(): ProviderTransform 转换 → AnthropicSDK 发起 HTTP 流
   → LLM 返回: "我先读一下 utils.py" + tool_call(read, {path: "utils.py"})
   → 文本通过 Bus → SSE → WebUI 实时显示
   → ToolExecutor: Permission.evaluate("read", "utils.py") → allow（命中规则）
   → ReadTool.execute() → 返回文件内容
   → 工具结果追加到 messages → 继续循环

4. Turn 2: LLM 分析代码后返回 tool_call(edit, {path: "utils.py", ...})
   → Permission.evaluate("edit", "utils.py") → ask（需要用户确认）
   → Bus.publish(PermissionAsked) → SSE → WebUI 弹出权限对话框
   → 用户点击 "Always Allow" → Permission.reply(always, scope=session)
   → Future resolved → EditTool.execute() → 文件修改完成
   → 结果追加 → 继续循环

5. Turn 3: LLM 返回纯文本 "重构完成，主要改动是..."
   → 无工具调用 → status="stop" → 退出循环

6. SessionPrompt 收尾:
   → SessionCompaction.is_overflow() → false（token 未溢出）
   → SessionSummary.generate() → 异步生成会话标题
   → Session.update_message() → 持久化到 SQLite
   → Bus.publish(MessageUpdated) → SSE → WebUI 更新完成
```

从用户敲下回车到看到最终回复，经过了 Interface → Transport → Session → LLM → Tool → Permission → Storage → Bus → SSE 九个层次。每一层都有明确的职责边界，每一层都可以独立理解和测试。

## 写在最后

构建一个 AI coding agent 的过程，本质上是在回答一个问题：**如何让 LLM 安全、可控、高效地与真实世界交互？**

Hotaru Code 的答案是：用分层架构隔离关注点，用事件总线解耦通信，用权限系统保障安全，用 Provider Transform 屏蔽差异；

最最重要的还是 Context Engineering。Context Engineering 是构建 AI 应用的核心。通过合理运用 Skills、MCP、Subagents，并遵循工具设计和评估的最佳实践，才能构建真正生产可用的 Agent 系统。

代码开源在 GitHub，欢迎 star 和 PR。

---

*技术栈：Python 3.12+ / asyncio / FastAPI / Textual / React / TypeScript / SQLite / MCP Protocol*
*项目地址：[GitHub - hotaru-code](https://github.com/Tsukumi233/hotaru-code)*


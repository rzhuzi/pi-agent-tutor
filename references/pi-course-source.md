# pi 源码教案——面向 AI 的深度解析

> Skill 使用说明：这是创建 Skill 时所依据的课程快照。文中的源码路径与历史行号只用于理解原教案结构；涉及当前版本时，应重新检查现行源码。

> 仓库：[earendil-works/pi](https://github.com/earendil-works/pi)（课程创建时使用过本地源码副本；本仓库不包含该副本）
> 定位：**教学文档**。逐模块、逐函数、含代码片段与行号引用。适合作为 AI 或工程师系统学习"如何造一个 minimal coding agent"的教材。
> 本 Skill 已将课程所需的路线图、术语表与练习库单独整理到 `references/`；原分析阶段的其他配套笔记未随仓库发布。
>
> 所有历史行号以课程创建时使用的源码快照为准；公开版仅保留文件路径与快照行号，不保留本地文件链接。

---

## 课程目录

| 课 | 主题 | 核心文件 |
|---|---|---|
| 0 | 预备知识：包结构与依赖方向 | package.json |
| 1 | 类型系统：AgentMessage / AgentTool / AgentEvent | agent/src/types.ts |
| 2 | Agent 类：状态机与双队列 | agent/src/agent.ts |
| 3 | agentLoop 主循环：runLoop 逐行精读 | agent/src/agent-loop.ts |
| 4 | 工具执行流水线：prepare → execute → finalize | agent/src/agent-loop.ts（下半部） |
| 5 | 内置工具四件套：bash / read / write / edit | agent/src/harness/tools/ |
| 6 | Session 持久化：Mutation / JSONL / torn-tail | agent/src/harness/session/ |
| 7 | Compaction：估算、切点、摘要生成 | agent/src/harness/compaction/compaction.ts |
| 8 | pi-ai 层：Provider / Models / streamSimple | ai/src/models.ts |
| 9 | 扩展系统：jiti 加载 / 虚拟模块 / 事件总线 | coding-agent/src/core/extensions/ |
| 10 | Skills 与 System Prompt | coding-agent/src/core/skills.ts、agent/src/harness/system-prompt.ts |
| 11 | AgentSession：模式共享骨架与自动压缩 | coding-agent/src/core/agent-session.ts |
| 12 | durable harness：规格 vs 占位实现 | agent/src/harness/agent-harness.ts、docs/harness.md |
| 13 | 总复习：设计模式清单 + 练习 | — |

---

## 第 0 课 预备知识：包结构与依赖方向

pi 是 npm workspaces monorepo。构建顺序即依赖方向（`package.json`（课程快照 L16））：

```
tui → telemetry → ai → agent → session-backends/sqlite-node → protocol → client → server → coding-agent
```

分层职责：

```
┌──────────────────────────────────────────────┐
│ coding-agent  （CLI/TUI/print/JSON/RPC/SDK）  │  ← 用户面
├──────────────┬───────────────┬───────────────┤
│ server       │ client        │ protocol      │  ← 远程接口层
├──────────────┴───────────────┴───────────────┤
│ agent（pi-agent-core）                        │  ← 有状态 agent 运行时（本教案主线）
├──────────────────────────────────────────────┤
│ ai（pi-ai）                                   │  ← 40+ provider 统一 LLM API
├──────────────────────────────────────────────┤
│ tui / telemetry / session-backends            │  ← 基础设施
└──────────────────────────────────────────────┘
```

**教学要点**：
1. `agent` 包不依赖 `coding-agent`——它是可独立复用的运行时库；`coding-agent` 在其上组装产品。
2. `agent` 依赖 `ai`，但只通过 `StreamFn` 函数类型注入（见第 1 课），不直接 import provider——控制反转。
3. SQLite 后端独立成包（`packages/agent/README.md`），避免 native 依赖污染核心。

---

## 第 1 课 类型系统：AgentMessage / AgentTool / AgentEvent

**文件**：`packages/agent/src/types.ts`（444 行，全部类型）

### 1.1 AgentMessage：可扩展的消息联合类型

```typescript
// types.ts L316-L325
export interface CustomAgentMessages {
	// Empty by default - apps extend via declaration merging
}

export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

这是 pi 最精妙的类型设计之一。`Message` 是 pi-ai 的 LLM 消息（user/assistant/toolResult），而 `CustomAgentMessages` 默认为空接口，**应用通过 declaration merging 注入自定义消息**：

```typescript
declare module "@mariozechner/agent" {
  interface CustomAgentMessages {
    artifact: ArtifactMessage;
    notification: NotificationMessage;
  }
}
```

注入后 `AgentMessage` 联合类型自动扩大，transcript（`AgentMessage[]`）即可承载 UI 通知、artifact 等非 LLM 消息，而类型系统全程保持类型安全。

**教学要点**：这解决了"transcript 需要记录 UI 事件，但 LLM 不该看到它们"的矛盾——答案在第 3 课的 `convertToLlm` 管道。

### 1.2 StreamFn：LLM 边界的函数抽象

```typescript
// types.ts L28-L32
export type StreamFn = (
	model: Model<Api>,
	context: Context,
	options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

注释（L19-L27）明确了**契约**：
- **不得 throw 或返回 rejected promise**——请求/模型/运行时失败必须编码在返回流中；
- 失败以协议事件 + 最终 `stopReason: "error" | "aborted"` 的 AssistantMessage 表达。

这是"错误即数据"（errors as data）风格：循环层永远不用 try/catch 处理 LLM 失败，而是把失败消息当普通消息流转。

### 1.3 AgentTool：带生命周期的工具定义

```typescript
// types.ts L386-L409
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
	label: string;                      // UI 显示名
	prepareArguments?: (args: unknown) => Static<TParameters>;  // 校验前的参数兼容垫片
	execute: (
		toolCallId: string,
		params: Static<TParameters>,
		signal?: AbortSignal,                        // 中止信号
		onUpdate?: AgentToolUpdateCallback<TDetails>, // 流式部分结果
	) => Promise<AgentToolResult<TDetails>>;
	executionMode?: ToolExecutionMode;  // "sequential" | "parallel" 覆盖全局
}
```

`AgentToolResult`（L361-L375）：

```typescript
export interface AgentToolResult<T> {
	content: (TextContent | ImageContent)[];  // 模型可见
	details: T;                                // UI/日志可见
	usage?: Usage;                             // 工具自身 token 用量
	addedToolNames?: string[];                 // 本次结果引入的新工具（动态工具注入！）
	terminate?: boolean;                       // 批次早停提示位
}
```

**教学要点**：
1. **content 与 details 分离**——`content` 进 LLM 上下文，`details` 只进 UI/日志。同一工具调用的"给模型看的"与"给人看的"解耦。
2. `addedToolNames` 支持**工具结果的副作用是引入新工具**（如"装完包后出现新命令"），这是其他 agent 框架少见的动态能力。
3. 错误处理约定（注释 L394）："**Throw on failure instead of encoding errors in `content`**"——工具失败靠 throw，由循环层统一转成 `isError: true` 的 toolResult。

### 1.4 AgentEvent：完整事件序列

```typescript
// types.ts L428-L443
export type AgentEvent =
	| { type: "agent_start" }
	| { type: "agent_end"; messages: AgentMessage[] }
	| { type: "turn_start" }
	| { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
	| { type: "message_start"; message: AgentMessage }
	| { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
	| { type: "message_end"; message: AgentMessage }
	| { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
	| { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
	| { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

层级：`agent > turn > (message | tool_execution)`。一个 turn = 一次 assistant 响应 + 其全部工具调用/结果。

**教学要点**：事件的判别字段是 `type`（discriminated union），消费方 switch 后类型自动收窄——TypeScript 教科书用法。

### 1.5 AgentLoopConfig 钩子全景

`packages/agent/src/types.ts`（课程快照 L149-L293） 定义了循环的全部扩展点。每个钩子的 JSDoc 都写明契约（"must not throw or reject"）。汇总表：

| 钩子 | 时机 | 用途 |
|---|---|---|
| `convertToLlm` | 每次 LLM 调用前（必需） | AgentMessage[] → Message[]，过滤 UI-only 消息 |
| `transformContext` | convertToLlm 之前 | 修剪/注入上下文（compaction 钩子） |
| `getApiKey` | 每次请求 | 动态 API key（OAuth token 刷新） |
| `shouldStopAfterTurn` | turn_end 后、队列消费前 | 优雅停止（如上下文将满） |
| `prepareNextTurn` | turn_end 后 | 替换下一轮的 context/model/thinkingLevel |
| `getSteeringMessages` | 工具执行完后 | 运行中插话 |
| `getFollowUpMessages` | agent 将停时 | 追加消息续跑 |
| `beforeToolCall` | 参数校验后、执行前 | 拦截/阻断（`{ block: true }`） |
| `afterToolCall` | 执行后、事件发出前 | 改写结果（字段级覆盖，无深合并） |

**教学要点**：注意 `BeforeToolCallResult.terminate` 的注释（L66-L68）："Early termination only happens when **every** finalized tool result in the batch sets this to true"——terminate 是批次级 AND 语义，不是单工具即可停。

---

## 第 2 课 Agent 类：状态机与双队列

**文件**：`packages/agent/src/agent.ts`（592 行）

Agent 是"low-level agent loop 的有状态包装"（L167-L172 注释）：拥有 transcript、发生命周期事件、执行工具、暴露 steering/follow-up 队列。

### 2.1 可变状态的防御性设计

```typescript
// agent.ts L61-L66
type MutableAgentState = Omit<AgentState, "isStreaming" | "streamingMessage" | "pendingToolCalls" | "errorMessage"> & {
	isStreaming: boolean;
	streamingMessage?: AgentMessage;
	pendingToolCalls: Set<string>;
	errorMessage?: string;
};
```

```typescript
// agent.ts L68-L95（节选）
function createMutableAgentState(initialState?): MutableAgentState {
	let tools = initialState?.tools?.slice() ?? [];
	let messages = initialState?.messages?.slice() ?? [];
	return {
		// ...
		get tools() { return tools; },
		set tools(nextTools: AgentTool<any>[]) { tools = nextTools.slice(); },
		get messages() { return messages; },
		set messages(nextMessages: AgentMessage[]) { messages = nextMessages.slice(); },
		// ...
	};
}
```

**教学要点**：getter/setter 强制**赋值即拷贝**（`slice()`）——外部赋一个数组进来，内部存的是浅拷贝；外部再改原数组不影响内部。这是防御性编程，防止"外部持有引用偷偷改 transcript"。

### 2.2 PendingMessageQueue：双队列的统一实现

```typescript
// agent.ts L125-L159
class PendingMessageQueue {
	private messages: AgentMessage[] = [];
	public mode: QueueMode;  // "all" | "one-at-a-time"

	enqueue(message: AgentMessage): void { this.messages.push(message); }
	hasItems(): boolean { return this.messages.length > 0; }

	drain(): AgentMessage[] {
		if (this.mode === "all") {
			const drained = this.messages.slice();
			this.messages = [];
			return drained;
		}
		const first = this.messages[0];
		if (!first) return [];
		this.messages = this.messages.slice(1);
		return [first];   // one-at-a-time：每次只取最旧一条
	}
	clear(): void { this.messages = []; }
}
```

Agent 持有两个实例（L231-L232）：

```typescript
this.steeringQueue = new PendingMessageQueue(runtimeOptions.steeringMode ?? "one-at-a-time");
this.followUpQueue = new PendingMessageQueue(runtimeOptions.followUpMode ?? "one-at-a-time");
```

**语义区分**（L282-L290 注释）：
- `steer(msg)`："注入到当前 assistant turn 结束之后"——运行中插话；
- `followUp(msg)`："仅在 agent 本要停止时运行"——停止前追加。

`one-at-a-time` 模式下，多条排队消息每个 drain 点只放行一条——让模型逐条消化而不是一次塞一堆。

### 2.3 单飞行模型（single-flight）与运行生命周期

```typescript
// agent.ts L161-L165
type ActiveRun = {
	promise: Promise<void>;
	resolve: () => void;
	abortController: AbortController;
};
```

`prompt()` 入口（L350-L358）：

```typescript
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
	if (this.activeRun) {
		throw new Error(
			"Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
		);
	}
	const messages = this.normalizePromptInput(input, images);
	await this.runPromptMessages(messages);
}
```

`runWithLifecycle`（L486-L509）：

```typescript
private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
	if (this.activeRun) throw new Error("Agent is already processing.");
	const abortController = new AbortController();
	let resolvePromise = () => {};
	const promise = new Promise<void>((resolve) => { resolvePromise = resolve; });
	this.activeRun = { promise, resolve: resolvePromise, abortController };

	this._state.isStreaming = true;
	this._state.streamingMessage = undefined;
	this._state.errorMessage = undefined;

	try {
		await executor(abortController.signal);
	} catch (error) {
		await this.handleRunFailure(error, abortController.signal.aborted);
	} finally {
		this.finishRun();
	}
}
```

**教学要点**：
1. **同一时刻最多一个 run**——并发 prompt 直接 throw，想插话必须走队列。这极大简化了并发推理。
2. 失败也被"事件化"（`handleRunFailure` L511-L527）：构造一条 `stopReason: "aborted" | "error"` 的 assistant 失败消息，按完整事件序列（message_start → message_end → turn_end → agent_end）发出——**消费者永远看到合法事件流，哪怕是崩溃**。

### 2.4 事件处理与监听器串行语义

```typescript
// agent.ts L544-L591（节选）
private async processEvents(event: AgentEvent): Promise<void> {
	switch (event.type) {
		case "message_end":
			this._state.streamingMessage = undefined;
			this._state.messages.push(event.message);
			break;
		case "tool_execution_start": {
			const pendingToolCalls = new Set(this._state.pendingToolCalls);
			pendingToolCalls.add(event.toolCallId);
			this._state.pendingToolCalls = pendingToolCalls;
			break;
		}
		// ...
	}
	const signal = this.activeRun?.abortController.signal;
	if (!signal) throw new Error("Agent listener invoked outside active run");
	for (const listener of this.listeners) {
		await listener(event, signal);   // 按订阅顺序 await——串行！
	}
}
```

**教学要点**：监听器按订阅顺序 **await**（L588-L590），意味着监听器（如 session 持久化）完成后事件才算处理完。`agent_end` 发出后 run 的 promise 还不 resolve——要等所有 `agent_end` 监听器 settle（L240-L253 注释）。这就是"持久化完成前不认为 run 结束"的实现基础。

### 2.5 createLoopConfig：把 Agent 状态桥接为循环配置

```typescript
// agent.ts L445-L484（节选）
private createLoopConfig(options: { skipInitialSteeringPoll?: boolean } = {}): AgentLoopConfig {
	let skipInitialSteeringPoll = options.skipInitialSteeringPoll === true;
	return {
		model: this._state.model,
		reasoning: this._state.thinkingLevel === "off" ? undefined : this._state.thinkingLevel,
		// ...
		getSteeringMessages: async () => {
			if (skipInitialSteeringPoll) {
				skipInitialSteeringPoll = false;
				return [];     // continue() 已手动 drain 过，跳过首次轮询
			}
			return this.steeringQueue.drain();
		},
		getFollowUpMessages: async () => this.followUpQueue.drain(),
	};
}
```

**教学要点**：`skipInitialSteeringPoll` 闭包标志解决 `continue()` 的边界情况（L371-L385）：continue 时若末条是 assistant，先手动 drain steering 队列作为输入，再跑循环——循环开局的 steering 轮询必须跳过，否则同一批消息会被注入两次。

---

## 第 3 课 agentLoop 主循环：runLoop 逐行精读

**文件**：`packages/agent/src/agent-loop.ts`（796 行）

### 3.1 双层 API：stream 版与 run 版

- `agentLoop()`（L31-L54）/ `agentLoopContinue()`（L64-L93）：返回 `EventStream<AgentEvent, AgentMessage[]>`，用 `for await` 消费——低层流式 API；
- `runAgentLoop()`（L95-L118）/ `runAgentLoopContinue()`（L120-L143）：回调式（`emit: AgentEventSink`），await 完整结果——Agent 类用的是这层。

`runAgentLoop` 的开头（L103-L114）：

```typescript
const newMessages: AgentMessage[] = [...prompts];
const currentContext: AgentContext = {
	...context,
	messages: [...context.messages, ...prompts],
};

await emit({ type: "agent_start" });
await emit({ type: "turn_start" });
for (const prompt of prompts) {
	await emit({ type: "message_start", message: prompt });
	await emit({ type: "message_end", message: prompt });
}
```

**教学要点**：`newMessages`（本次运行新增的消息）与 `currentContext.messages`（全量 transcript）是两个数组——事件结束时 `agent_end` 带的是 newMessages，调用方据此知道"这次跑了什么"。

### 3.2 runLoop：双层 while 结构

`packages/agent/src/agent-loop.ts`（课程快照 L155-L275）。完整骨架：

```typescript
async function runLoop(initialContext, newMessages, initialConfig, signal, emit, streamFunction): Promise<void> {
	let currentContext = initialContext;
	let config = initialConfig;
	let firstTurn = true;
	// 开局先查一次 steering（用户可能在等待时就输入了）
	let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

	// 外层循环：agent 本要停止时，若 follow-up 队列有消息则续跑
	while (true) {
		let hasMoreToolCalls = true;

		// 内层循环：处理工具调用与 steering 消息
		while (hasMoreToolCalls || pendingMessages.length > 0) {
			if (!firstTurn) {
				await emit({ type: "turn_start" });
			} else {
				firstTurn = false;   // 首个 turn_start 已由外层入口发出
			}

			// 1. 注入 pending 消息（steering 或 follow-up）
			if (pendingMessages.length > 0) {
				for (const message of pendingMessages) {
					await emit({ type: "message_start", message });
					await emit({ type: "message_end", message });
					currentContext.messages.push(message);
					newMessages.push(message);
				}
				pendingMessages = [];
			}

			// 2. 流式获取 assistant 响应
			const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
			newMessages.push(message);

			// 3. 错误/中止：短路退出
			if (message.stopReason === "error" || message.stopReason === "aborted") {
				await emit({ type: "turn_end", message, toolResults: [] });
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

			// 4. 执行工具调用
			const toolCalls = message.content.filter((c) => c.type === "toolCall");
			const toolResults: ToolResultMessage[] = [];
			hasMoreToolCalls = false;
			if (toolCalls.length > 0) {
				const executedToolBatch =
					message.stopReason === "length"
						? await failToolCallsFromTruncatedMessage(toolCalls, emit)  // 截断的调用全部失败
						: await executeToolCalls(currentContext, message, config, signal, emit);
				toolResults.push(...executedToolBatch.messages);
				hasMoreToolCalls = !executedToolBatch.terminate;   // 全员 terminate 才停
				for (const result of toolResults) {
					currentContext.messages.push(result);
					newMessages.push(result);
				}
			}

			await emit({ type: "turn_end", message, toolResults });

			// 5. prepareNextTurn：允许换模型/换上下文/换思考等级
			const nextTurnSnapshot = await config.prepareNextTurn?.({ message, toolResults, context: currentContext, newMessages });
			if (nextTurnSnapshot) {
				currentContext = nextTurnSnapshot.context ?? currentContext;
				config = { ...config, model: nextTurnSnapshot.model ?? config.model, /* reasoning 同理 */ };
			}

			// 6. shouldStopAfterTurn：优雅停止点
			if (await config.shouldStopAfterTurn?.({ message, toolResults, context: currentContext, newMessages })) {
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

			// 7. drain steering 队列
			pendingMessages = (await config.getSteeringMessages?.()) || [];
		}

		// 内层退出 = 无工具调用且无 steering。查 follow-up
		const followUpMessages = (await config.getFollowUpMessages?.()) || [];
		if (followUpMessages.length > 0) {
			pendingMessages = followUpMessages;   // 变成下轮内层的 pending，continue 外层
			continue;
		}
		break;   // 真正结束
	}

	await emit({ type: "agent_end", messages: newMessages });
}
```

**教学要点（逐条）**：

1. **双层循环的分工**：内层 while = "还有活干"（工具调用/steering）；外层 while = "本要停了，看看有没有 follow-up"。`hasMoreToolCalls` 初始为 `true` 保证至少跑一轮。
2. **消息注入的时序**（步骤 1）：pending 消息在 assistant 响应**之前**注入——即 steering 消息出现在工具结果之后、下一次 LLM 调用之前的 transcript 位置。
3. **`stopReason === "length"` 的防御**（步骤 4，L207-L214 注释）："a 'length' stop means the output was cut off by the token limit, so every tool call in the message may carry truncated arguments. Fail them all instead of executing potentially borked calls."——被截断的消息即使参数恰好能解析通过也可能是残缺的，全部标记失败让模型重发。这是实战认知：JSON salvage parser 可能"成功"解析出静默不完整的参数。
4. **terminate 的 AND 语义**（L216）：`hasMoreToolCalls = !executedToolBatch.terminate`——批次早停需所有结果 terminate。
5. **`prepareNextTurn` 在 `shouldStopAfterTurn` 之前**（步骤 5→6）：即使本轮要停，下一轮的配置快照也已备好（供外层 continue 使用）。

### 3.3 streamAssistantResponse：AgentMessage 到 Message 的转换边界

`packages/agent/src/agent-loop.ts`（课程快照 L281-L372）。核心管道：

```typescript
// 1. transformContext（可选）：AgentMessage[] → AgentMessage[]
let messages = context.messages;
if (config.transformContext) {
	messages = await config.transformContext(messages, signal);
}
// 2. convertToLlm（必需）：AgentMessage[] → Message[]
const llmMessages = await config.convertToLlm(messages);
// 3. 组装 LLM Context
const llmContext: Context = { systemPrompt: context.systemPrompt, messages: llmMessages, tools: context.tools };
// 4. 动态解析 API key（应对过期 OAuth token）
const resolvedApiKey =
	(config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;
// 5. 调 streamFn
const response = await streamFunction(config.model, llmContext, { ...config, apiKey: resolvedApiKey, signal });
```

随后是流事件消费（L317-L361）：`start` 事件把 partial 消息 push 进 context 并 emit `message_start`；每个 delta 事件**原地替换** context 末尾（`context.messages[context.messages.length - 1] = partialMessage`）并 emit `message_update`；`done`/`error` 时用 `response.result()` 取最终消息替换。

**教学要点**：
- **partial 消息直接进 transcript**：流式过程中 transcript 末尾始终是"当前最新 partial"，结束时原位替换为 final。这样任何时刻快照 context 都能得到一致视图。
- **转换管道的三段式**是 pi 的"双层消息抽象"落地：
  ```
  AgentMessage[] --transformContext--> AgentMessage[] --convertToLlm--> Message[] → LLM
  ```
  compaction 在第一段做（操作 transcript 语义），消息过滤在第二段做（操作 LLM 语义）。

---

## 第 4 课 工具执行流水线：prepare → execute → finalize

**文件**：`packages/agent/src/agent-loop.ts`（课程快照 L411-L796）

### 4.1 三段式状态类型

```typescript
// agent-loop.ts L556-L580
type PreparedToolCall = {          // 已校验、已放行，待执行
	kind: "prepared";
	toolCall: AgentToolCall;
	tool: AgentTool<any>;
	args: unknown;
};

type ImmediateToolCallOutcome = {  // 无需执行即出结果（错误/被拦截）
	kind: "immediate";
	result: AgentToolResult<any>;
	isError: boolean;
};

type ExecutedToolCallOutcome = { result: AgentToolResult<any>; isError: boolean };  // 已执行
type FinalizedToolCallOutcome = { toolCall; result; isError };                      // 已终化（含 afterToolCall 覆盖）
```

工具调用状态机：`prepareToolCall` 返回 prepared 或 immediate；prepared → `executePreparedToolCall` → executed → `finalizeExecutedToolCall` → finalized。

### 4.2 prepareToolCall：查表 → 垫片 → 校验 → 拦截

`packages/agent/src/agent-loop.ts`（课程快照 L600-L668）：

```typescript
async function prepareToolCall(currentContext, assistantMessage, toolCall, config, signal) {
	const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
	if (!tool) {
		return { kind: "immediate", result: createErrorToolResult(`Tool ${toolCall.name} not found`), isError: true };
	}
	try {
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);   // ① prepareArguments 垫片
		const validatedArgs = validateToolArguments(tool, preparedToolCall); // ② schema 校验
		if (config.beforeToolCall) {                                        // ③ 拦截钩子
			const beforeResult = await config.beforeToolCall(
				{ assistantMessage, toolCall, args: validatedArgs, context: currentContext },
				signal,
			);
			if (signal?.aborted) { /* immediate aborted */ }
			if (beforeResult?.block) {
				const result = createErrorToolResult(beforeResult.reason || "Tool execution was blocked");
				if (beforeResult.terminate === true) result.terminate = true;
				return { kind: "immediate", result, isError: true };
			}
		}
		if (signal?.aborted) { /* immediate aborted */ }
		return { kind: "prepared", toolCall, tool, args: validatedArgs };
	} catch (error) {
		return { kind: "immediate", result: createErrorToolResult(String(error)), isError: true };
	}
}
```

**教学要点**：
1. **查不到工具、校验失败、被拦截、被中止**四种情况全部返回 `immediate` 错误结果——统一走"错误也是合法工具结果"路径，模型能看到并自行纠正。
2. `prepareToolCallArguments`（L586-L598）调用工具自带的 `prepareArguments` 垫片——给旧模型/不守格式的模型留兼容层（第 5 课 edit 工具有实战用例：把旧版顶层 oldText/newText 迁移到 edits 数组）。

### 4.3 并行 vs 串行

选择逻辑（L411-L426）：

```typescript
const hasSequentialToolCall = toolCalls.some(
	(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
	return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

**一个工具声明 sequential，整批降级为串行**。

并行版（L489-L554）的结构值得精读：

```typescript
async function executeToolCallsParallel(...): Promise<ExecutedToolCallBatch> {
	const finalizedCalls: FinalizedToolCallEntry[] = [];

	// 阶段一：顺序 preflight（prepare）所有调用
	for (const toolCall of toolCalls) {
		await emit({ type: "tool_execution_start", ... });
		const preparation = await prepareToolCall(...);
		if (preparation.kind === "immediate") {
			// 立即结果的直接收尾
			const finalized = { toolCall, result: preparation.result, isError: preparation.isError };
			await emitToolExecutionEnd(finalized, emit);
			finalizedCalls.push(finalized);
			continue;
		}
		// 延迟执行的闭包：execute → finalize → emit end
		finalizedCalls.push(async () => {
			const executed = await executePreparedToolCall(preparation, signal, emit);
			const finalized = await finalizeExecutedToolCall(...);
			await emitToolExecutionEnd(finalized, emit);
			return finalized;
		});
	}

	// 阶段二：并发执行全部闭包
	const orderedFinalizedCalls = await Promise.all(
		finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
	);

	// 阶段三：按 assistant 源序生成 toolResult 消息
	const messages: ToolResultMessage[] = [];
	for (const finalized of orderedFinalizedCalls) {
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit);
		messages.push(toolResultMessage);
	}
	return { messages, terminate: shouldTerminateToolBatch(orderedFinalizedCalls) };
}
```

`FinalizedToolCallEntry = FinalizedToolCallOutcome | (() => Promise<FinalizedToolCallOutcome>)`（L580）——**联合类型存"已完成的值"或"待执行的闭包"**，`Promise.all` 时统一展开。

**教学要点（关键设计）**：
- `tool_execution_end` 按**完成顺序**发出（闭包内即时 emit）——UI 实时显示先完成的工具；
- toolResult **消息** 按 assistant 源序发出（阶段三统一生成）——因为多数 provider 要求 toolResult 顺序与 toolCall 顺序一致。
- **事件顺序与消息顺序解耦**：这就是 types.ts L39-L41 注释所说的 "`tool_execution_end` is emitted in tool completion order ... while tool-result message artifacts are emitted later in assistant source order"。

### 4.4 executePreparedToolCall：流式部分更新

`packages/agent/src/agent-loop.ts`（课程快照 L670-L711）：

```typescript
async function executePreparedToolCall(prepared, signal, emit): Promise<ExecutedToolCallOutcome> {
	const updateEvents: Promise<void>[] = [];
	let acceptingUpdates = true;
	try {
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args as never,
			signal,
			(partialResult) => {              // onUpdate 回调
				if (!acceptingUpdates) return; // settle 后的调用直接忽略
				updateEvents.push(Promise.resolve(emit({ type: "tool_execution_update", ... })));
			},
		);
		acceptingUpdates = false;
		await Promise.all(updateEvents);      // 等待所有 in-flight update 事件
		return { result, isError: false };
	} catch (error) {
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return { result: createErrorToolResult(String(error)), isError: true };
	} finally {
		acceptingUpdates = false;
	}
}
```

**教学要点**：
1. `acceptingUpdates` 标志 + finally 三重置——工具 promise settle 后 onUpdate 回调静默忽略（types.ts L380-L383 注释的兑现），防止晚到的部分结果事件乱序混入。
2. **工具 throw → isError 结果**（catch 分支）——第 1 课"Throw on failure"契约在此落地。
3. update 事件 fire-and-forget 收集、settle 后 `Promise.all`——不阻塞工具执行主线。

### 4.5 finalizeExecutedToolCall：字段级覆盖

`packages/agent/src/agent-loop.ts`（课程快照 L713-L758）：

```typescript
if (afterResult) {
	result = {
		...result,
		content: afterResult.content ?? result.content,    // 提供才覆盖
		details: afterResult.details ?? result.details,
		usage: afterResult.usage ?? result.usage,
		terminate: afterResult.terminate ?? result.terminate,
	};
	isError = afterResult.isError ?? isError;
}
```

`??` 链实现"省略字段保持原值"——与 types.ts L74-L83 注释的合并语义严格一致，**无深合并**。

---

## 第 5 课 内置工具四件套

**目录**：`packages/agent/src/harness/tools`

### 5.1 bash 工具：节流流式输出

`packages/agent/src/harness/tools/bash.ts`（161 行）

Schema（L11-L14）：

```typescript
const bashSchema = Type.Object({
	command: Type.String({ description: "Bash command to execute" }),
	timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" })),
});
```

**流式更新节流**（L9、L69-L105）：

```typescript
const BASH_UPDATE_THROTTLE_MS = 100;

const emitOutputUpdate = (): void => {
	if (!onUpdate || !updateDirty || !getLatestProgress) return;
	updateDirty = false;
	lastUpdateAt = Date.now();
	const progress = getLatestProgress();
	onUpdate({
		content: [{ type: "text", text: progress.output }],
		details: { truncation: ..., fullOutputPath: ... },
	});
};

const scheduleOutputUpdate = (): void => {
	if (!onUpdate) return;
	updateDirty = true;
	const delay = BASH_UPDATE_THROTTLE_MS - (Date.now() - lastUpdateAt);
	if (delay <= 0) { clearUpdateTimer(); emitOutputUpdate(); return; }
	updateTimer ??= setTimeout(() => { updateTimer = undefined; emitOutputUpdate(); }, delay);
};
```

每个输出 chunk 到来时只调 `scheduleOutputUpdate()`（标脏 + 定时），实际 emit 最多 100ms 一次——**长输出命令不会淹没事件流**。

**截断与全文落盘**（L128-L142）：输出超过上限（DEFAULT_MAX_LINES 行 / DEFAULT_MAX_BYTES 字节）时截断显示，全文写入临时文件，并在文本尾部追加提示：

```
[Showing lines 81-580 of 1200. Full output: /tmp/xxx]
```

**错误统一 throw**（L144-L154）：`cancelled` → throw "Command aborted"；timeout → throw 带原因；非零退出码 → throw `Command exited with code N`（错误消息包含已有输出，模型能看到部分输出+失败原因）。

**教学要点**：`options.prepare`（L30-L39、L68）是执行前钩子——宿主可以改写 command（如加 commandPrefix）、cwd、env。Gondolin 微 VM 沙箱就是在这里把命令路由进 VM 的。

### 5.2 read 工具：文本与图片双模

`packages/agent/src/harness/tools/read.ts`（144 行）

关键分支：先嗅探 magic bytes（`detectSupportedImageMimeType`，L56），图片走 ImageContent 分支（可注入 imageProcessor 缩放，L58-L95），文本走行切片分支。

**文本续读设计**（L97-L141）：

```typescript
const allLines = textContent.split("\n");
const startLine = offset ? Math.max(0, offset - 1) : 0;
// ...
const truncation = truncateHead(selectedContent);   // 保留尾部（最新内容）
if (truncation.truncated) {
	const endLineDisplay = startLineDisplay + truncation.outputLines - 1;
	const nextOffset = endLineDisplay + 1;
	outputText += `\n\n[Showing lines ${startLineDisplay}-${endLineDisplay} of ${totalFileLines}. Use offset=${nextOffset} to continue.]`;
}
```

**教学要点**：
1. 截断策略是 `truncateHead`——**保留尾部**。文件头部是 import/声明，尾部是近期修改，对代码任务尾部信息密度更高。
2. 截断提示语直接告诉模型**下一跳参数**（`Use offset=N to continue`）——把续读协议写进模型可见文本，模型无需猜。
3. 超长单行（`firstLineExceedsLimit`，L119-L122）时给出 bash 替代命令：`sed -n 'Np' file | head -c LIMIT`——工具间的降级路径。

### 5.3 edit 工具：精确替换 + 兼容垫片

`packages/agent/src/harness/tools/edit.ts`（127 行）

新 schema（L17-L37）：`{ path, edits: [{ oldText, newText }, ...] }`——一次调用多处编辑，description 明确要求"non-overlapping、同块修改合并为一个 edit"。

**prepareArguments 兼容垫片**（L48-L64）——第 4 课 promised 的实战用例：

```typescript
function prepareEditArguments(input: unknown): EditToolInput {
	if (!input || typeof input !== "object") return input as EditToolInput;
	const args = input as Record<string, unknown>;
	if (typeof args.edits === "string") {
		try {
			const parsed: unknown = JSON.parse(args.edits);
			if (Array.isArray(parsed)) args.edits = parsed;   // 修复模型把数组编码成字符串
		} catch {}
	}
	const legacy = args as LegacyEditToolInput;
	if (typeof legacy.oldText !== "string" || typeof legacy.newText !== "string") return args as EditToolInput;
	// 旧版顶层 oldText/newText 迁移到 edits 数组
	const edits = Array.isArray(legacy.edits) ? [...legacy.edits] : [];
	edits.push({ oldText: legacy.oldText, newText: legacy.newText });
	const { oldText: _oldText, newText: _newText, ...rest } = legacy;
	return { ...rest, edits } as EditToolInput;
}
```

处理两种真实模型行为：① 把 edits 数组 JSON 编码成字符串；② 旧格式顶层 oldText/newText。**在校验前静默修复**，减少无效失败。

**执行主体**（L89-L125）：

```typescript
return withFileMutationQueue(env, absolutePath, async () => {
	// ① 读文件
	const readResult = await env.readTextFile(absolutePath, signal);
	// ② strip BOM + 归一化换行为 LF
	const { bom, text: content } = stripBom(readResult.value);
	const originalEnding = detectLineEnding(content);
	const normalizedContent = normalizeToLF(content);
	// ③ 应用全部 edits（基于原文匹配，非增量）
	const { baseContent, newContent } = applyEditsToNormalizedContent(normalizedContent, edits, path);
	// ④ 还原换行风格 + BOM 写回
	const finalContent = bom + restoreLineEndings(newContent, originalEnding);
	const writeResult = await env.writeFile(absolutePath, finalContent, signal);
	// ⑤ 生成 diff 与 unified patch 作为 details
	const diffResult = generateDiffString(baseContent, newContent);
	return {
		content: [{ type: "text", text: `Successfully replaced ${edits.length} block(s) in ${path}.` }],
		details: { diff: diffResult.diff, patch: generateUnifiedPatch(...), firstChangedLine: ... },
	};
});
```

**教学要点**：
1. **换行归一化三步曲**（strip BOM → LF 归一 → 编辑 → 还原）：编辑匹配在 LF 空间做，写回时还原 CRLF/BOM——跨平台文件不被破坏。
2. `withFileMutationQueue`：**同一文件的变更操作串行化**（write 工具同样使用）——并行工具批中若同时有 write+edit 同一文件，靠这个队列避免交错写坏。
3. 模型可见 content 只有一句成功摘要；diff/patch 进 details 给 UI——content/details 分离的又一次实践。

---

## 第 6 课 Session 持久化：Mutation / JSONL / torn-tail

**目录**：`packages/agent/src/harness/session`

### 6.1 数据模型：Entry 树 + LaneRecord + Fact

`packages/agent/src/harness/session/types.ts`（课程快照 L14-L74）：

```typescript
export interface EntryBase {
	type: string;
	id: string;
	seq: number;          // 全局递增序列（存储侧分配）
	parentId: string | null;  // 父节点 → 构成会话树
	timestamp: number;
}

export type Entry =
	| MessageEntry            // 对话消息
	| ModelChangeEntry        // 模型切换
	| ThinkingLevelEntry      // 思考等级切换
	| ActiveToolsEntry        // 工具集切换
	| CompactionEntry         // 压缩点（summary + retainedTail）
	| BranchSummaryEntry      // 分支摘要
	| CustomEntry;            // 应用自定义
```

**entries 构成一棵树**：`parentId` 指向前一个条目，"分支"即从某节点另开子链。lane（默认 "main"）是指向树中某 leaf 的**指针**——移动 lane 指针 = 切换分支，旧分支数据不删。

LaneRecord（L87-L212）是操作流水：`operation_started` / `abort_requested` / `operation_finished` / `step_attempt` / `tool_started` / `queue_enqueued` / `queue_cancelled` / `write_deferred` / `usage`。注意 `ToolStartedRecord`（L150-L160）带有 `replay: "never" | "safe"` 字段——这正是 harness.md 规格中 effect sandwich 的 replay policy 在存储层的前置定义。

### 6.2 SessionMutation：四类原子的写操作

`packages/agent/src/harness/session/state.ts`（课程快照 L17-L22）：

```typescript
export type SessionMutation =
	| { kind: "entry"; lane?: string; entry: Entry }   // 追加树条目
	| { kind: "record"; record: LaneRecord }           // 追加操作流水
	| { kind: "lane"; seq: number; lane: string; leafId: string | null }  // 移动 lane 指针
	| { kind: "fact"; seq: number; fact: "name"; name: string | undefined }        // 全局名字
	| { kind: "fact"; seq: number; fact: "label"; targetId: string; label: string | undefined };  // 节点标签
```

**教学要点**：持久化的最小单位是 mutation（不是 entry）——日志文件的每一行就是一个 mutation 的 JSON 编码。这使得"树写 + 指针移动 + 元数据"共享同一套追加/重放机制。

### 6.3 SessionState.applyMutation：重放即重建

`packages/agent/src/harness/session/state.ts`（课程快照 L97-L180）。核心校验+应用：

```typescript
applyMutation(mutation: SessionMutation, invalid: InvalidMutation = invalidMutation): void {
	const seq = /* entry.seq | record.seq | mutation.seq */;
	if (seq !== this.sequence + 1) invalid(`has non-consecutive seq ${seq}`);   // ① 严格连续

	switch (mutation.kind) {
		case "entry": {
			if (this.usedIds.has(mutation.entry.id)) invalid(`contains duplicate id ...`);   // ② id 唯一
			if (mutation.lane !== undefined) {
				const leafId = this.lanes.get(mutation.lane);
				if (leafId === undefined) invalid(`references missing lane ...`);            // ③ lane 存在
				if (mutation.entry.parentId !== leafId) invalid("does not chain to the lane leaf");  // ④ 链到 leaf
			}
			if (mutation.entry.parentId !== null && !this.entriesById.has(mutation.entry.parentId)) {
				invalid(`references missing parent ...`);                                    // ⑤ 父存在
			}
			this.sequence = seq;
			this.usedIds.add(mutation.entry.id);
			this.entries.push(mutation.entry);
			this.entriesById.set(mutation.entry.id, mutation.entry);
			if (mutation.lane !== undefined) this.lanes.set(mutation.lane, mutation.entry.id);  // ⑥ leaf 前移
			this.log.push({ kind: "entry", seq, entry: mutation.entry });
			if (mutation.entry.type === "message") this.stats.messageCount += 1;
			break;
		}
		// record / lane / fact 分支同理……
	}
}
```

**教学要点**：
1. **seq 严格连续**（`seq !== this.sequence + 1` 即 invalid）——日志的任何缺口都被视为损坏。这是"原子事务"在 JSONL 上的体现：一个事务的多条 mutation 必须连续写入，中间崩溃 → seq 断裂 → 加载时报损（或整批被 torn-tail 截掉）。
2. **entry 必须链到 lane 当前 leaf**（校验 ④）+ 应用后 leaf 前移（⑥）——保证 lane 永远指向"最新追加的条目"，写入即构建主链。
3. **usage record 顺带累计 stats**（L143-L148）——`getStats()` 免遍历。

### 6.4 JsonlSessionStorage：单写队列 + torn-tail 恢复

`packages/agent/src/harness/session/jsonl/storage.ts`（277 行）

**单写队列**（L258-L265）：

```typescript
private enqueue<T>(operation: () => Promise<T>): Promise<T> {
	const result = this.tail.then(operation);   // 排在上一个写后面
	this.tail = result.then(() => undefined, () => undefined);  // tail 不承接错误
	return result;
}

private async appendMutation(mutation: SessionMutation): Promise<void> {
	fileResult(
		await this.fs.appendFile(this.metadata.path, encodeMutation(mutation)),
		`Failed to append session ${this.metadata.path}`,
	);
}
```

所有写操作（appendEntry/appendRecord/createLane/setName…）都经 `enqueue` 串行化——**并发的 appendEntry 在 promise 链上排队**，文件中永不交错。`drain()`（L122-L124）等待队列清空。

**写入流程**（appendEntry，L154-L169）：

```typescript
appendEntry<TEntry extends Entry>(newEntry: lane: string): Promise<TEntry> {
	return this.enqueue(async () => {
		const parentId = this.state.requireLane(lane);        // ① 取 lane leaf 为 parent
		this.state.validateUnusedId(newEntry.id);             // ② id 查重
		const entry = {
			...structuredClone(newEntry),                     // ③ 深拷贝入参（防外部引用）
			parentId,
			seq: this.state.nextSequence,                     // ④ 存储侧分配 seq
			timestamp: Date.now(),
		} as unknown as TEntry;
		const mutation: SessionMutation = { kind: "entry", lane, entry };
		await this.appendMutation(mutation);                  // ⑤ 先落盘
		this.applyMutation(mutation);                         // ⑥ 再改内存状态
		return structuredClone(entry);                        // ⑦ 返回拷贝
	});
}
```

**教学要点**：⑤⑥ 的顺序是 **write-ahead**——先追加文件再更新内存。若 ⑤ 成功 ⑥ 前崩溃，重启重放时会重新 apply 该 mutation（幂等，因为重放从零构建 state）；若 ⑤ 失败则抛错、内存不变。**先持久化后提交内存**是事件溯源的标准姿势。

**torn-tail 恢复**（load 静态方法，L69-L108）：

```typescript
static async load(fs, path): Promise<JsonlSessionStorage> {
	const content = fileResult(await fs.readTextFile(path), ...);
	const physicalLines = content.split("\n");
	if (physicalLines.at(-1) === "") physicalLines.pop();     // 去掉末尾空行
	// ...解析 header（第一行）...
	for (let index = 1; index < physicalLines.length; index++) {
		const line = physicalLines[index]!;
		const mutationResult = parseMutation(line);
		if (!mutationResult.ok) {
			const isTornTail = index === physicalLines.length - 1 && mutationResult.error.kind === "syntax";
			if (isTornTail) {
				// 丢弃未确认的部分写入：原子发布有效前缀
				const validPrefix = `${physicalLines.slice(0, index).join("\n")}\n`;
				await publishFileAtomically(fs, path, async (tempPath) => {
					fileResult(await fs.writeFile(tempPath, validPrefix), ...);
				});
				return storage;
			}
			throw invalidFile(path, index + 1, mutationResult.error);  // 中间行损坏 = 报错
		}
		storage.applyMutation(mutationResult.value);
	}
	if (!content.endsWith("\n")) {
		// 修复未换行结尾
		fileResult(await fs.appendFile(path, "\n"), ...);
	}
	return storage;
}
```

**教学要点**：
1. **只有最后一行且是语法错误才判 torn-tail**——追加写崩溃只可能损坏最后一行；中间行损坏说明文件被外部改坏，直接报错不静默修复。
2. 修复用 `publishFileAtomically`（L33-L46）：写完整临时文件 → `rename` 原子替换。临时文件策略的注释（L26-L31）说明："a process crash while populating can leave only the ignored `.tmp` file behind"——崩溃只留垃圾 tmp，不损坏原文件。

### 6.5 codec：一行一 mutation 的编解码

`packages/agent/src/harness/session/jsonl/codec.ts`（课程快照 L229-L239）：

```typescript
export function encodeMutation(mutation: SessionMutation): string {
	switch (mutation.kind) {
		case "entry":
			return `${JSON.stringify({ kind: "entry", lane: mutation.lane, ...mutation.entry })}\n`;
		case "record":
			return `${JSON.stringify({ kind: "record", ...mutation.record })}\n`;
		case "lane":
			return `${JSON.stringify(mutation)}\n`;
		case "fact":
			return `${JSON.stringify(mutation)}\n`;
	}
}
```

文件格式：第一行 header（`{kind:"header", version:4, id, createdAt, cwd, ...}`，L70-L100），之后每行一个 mutation JSON。decode 侧（`parseMutation` L220-L227）用 Result 类型返回 `ok/err` 而非 throw——语法错误与 schema 错误分类（`error.kind === "syntax" | "schema"`），供 torn-tail 判定。

### 6.6 崩溃恢复的边界（重要！）

当前 JSONL 后端**不持久化操作执行进度**。`ToolStartedRecord` 有 `replay` 字段，`operation_started/finished` record 也存在，但生产路径（Agent 类）**并不写这些 record**——它们是 durable harness（第 12 课）的词汇。因此当前版本崩溃恢复语义是：

- 消息级：完整恢复（transcript 全在）；
- 操作级：不知道"删文件删到一半"之类的外部副作用状态——重启后无法判断某工具是否执行过。

这正是 harness.md 规格要补的缺口（见第 12 课）。

---

## 第 7 课 Compaction：估算、切点、摘要生成

**文件**：`packages/agent/src/harness/compaction/compaction.ts`（848 行）

### 7.1 token 估算：优先用量、降级启发式

```typescript
// compaction.ts L247-L250
export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
}
```

默认阈值（L158-L162）：`reserveTokens: 16384`（给摘要 prompt+输出留量）、`keepRecentTokens: 20000`（保留近期上下文）。

估算策略（L216-L244）：

```typescript
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
	const usageInfo = getLastAssistantUsageInfo(messages);   // 找最近一条有效 assistant usage
	if (!usageInfo) {
		/* 无用量：全量字符启发式估算 */
	}
	const usageTokens = calculateContextTokens(usageInfo.usage);   // provider 报告的真值
	let trailingTokens = 0;
	for (let i = usageInfo.index + 1; i < messages.length; i++) {
		trailingTokens += estimateTokens(messages[i]);             // 之后的增量启发式估算
	}
	return { tokens: usageTokens + trailingTokens, ... };
}
```

单消息启发式（L271-L311）：`Math.ceil(chars / 4)`（4 字符≈1 token），图片固定 4800 字符（L252）。

**教学要点**：**真值锚点 + 增量估算**——最近一次 provider 报告的 usage 是精确的，其后新消息用启发式。比全量启发式准，比全量 tokenizer 快。

### 7.2 切点选择：只在合法边界切

`findValidCutPoints`（L312-L344）：**user/bashExecution/branchSummary/compactionSummary 消息是合法切点；toolResult 永不是**（切在 toolResult 会把它与对应的 assistant toolCall 分离，破坏上下文结构）。

`findCutPoint`（L374-L422）：

```typescript
export function findCutPoint(entries, startIndex, endIndex, keepRecentTokens): CutPointResult {
	const cutPoints = findValidCutPoints(entries, startIndex, endIndex);
	// 从尾部累加 token，直到达到 keepRecentTokens
	let accumulatedTokens = 0;
	for (let i = endIndex - 1; i >= startIndex; i--) {
		/* accumulate messageTokens */
		if (accumulatedTokens >= keepRecentTokens) {
			// 找到第一个 >= i 的合法切点
			for (let c = 0; c < cutPoints.length; c++) {
				if (cutPoints[c] >= i) { cutIndex = cutPoints[c]; break; }
			}
			break;
		}
	}
	// 回退穿过非消息条目到消息/compaction 边界
	// ...
	const isUserMessage = cutEntry.type === "message" && cutEntry.message.role === "user";
	const turnStartIndex = isUserMessage ? -1 : findTurnStartIndex(entries, cutIndex, startIndex);
	return {
		firstKeptEntryIndex: cutIndex,
		turnStartIndex,
		isSplitTurn: !isUserMessage && turnStartIndex !== -1,   // 切点落在一个 turn 中间
	};
}
```

**isSplitTurn**：如果保留下界切在一个 turn 的中间（比如 toolResult 后的 assistant 消息），该 turn 的前缀（user 提问 + 早期工具调用）会被摘要，后缀保留——需要**双段摘要**（见 7.3）。

### 7.3 prepareCompaction 与 compact：两段式执行

`prepareCompaction`（L616-L687）纯函数准备（不调 LLM）：

```typescript
export function prepareCompaction(pathEntries: Entry[], settings): Result<CompactionPreparation | undefined, CompactionError> {
	// ① 上次 compaction 的 retainedTail 展开为"虚拟条目"参与本轮切分
	if (prevCompactionIndex >= 0) {
		const prevCompaction = pathEntries[prevCompactionIndex] as CompactionEntry;
		previousSummary = prevCompaction.summary;
		const virtualRetainedEntries: Entry[] = prevCompaction.retainedTail.map((message, index) => ({
			type: "message",
			id: `${prevCompaction.id}:retained:${index}`,   // 合成 id，不与真实条目冲突
			parentId: index === 0 ? prevCompaction.id : `${prevCompaction.id}:retained:${index - 1}`,
			/* ... */
		}));
		compactableEntries = [...virtualRetainedEntries, ...pathEntries.slice(prevCompactionIndex + 1)];
	}
	// ② 找切点，切成三段
	const cutPoint = findCutPoint(compactableEntries, 0, boundaryEnd, settings.keepRecentTokens);
	// → messagesToSummarize（将被摘要的历史）
	// → turnPrefixMessages（split turn 的前缀，单独摘要）
	// → retainedTail（保留的近期消息，将存进 CompactionEntry）
	// ③ 提取文件操作清单（read/edited 集合，含上次的累计）
	const fileOps = extractFileOperations(messagesToSummarize, pathEntries, prevCompactionIndex);
	return ok({ messagesToSummarize, turnPrefixMessages, retainedTail, isSplitTurn, tokensBefore, previousSummary, fileOps, settings });
}
```

**教学要点**：
1. **retainedTail 的虚拟展开**——上一轮 compaction 保留的尾部消息在本轮参与切点计算，否则它们永远"卡"在切点右侧之外。
2. **fileOps 跨轮累计**（L44-L67）：read/edited 文件集合从上次 compaction 的 details 继承再累加——模型"看过/改过哪些文件"的记忆跨压缩持续。

`compact`（L707-L794）调 LLM 生成摘要：

```typescript
export async function compact(preparation, models, model, customInstructions?, signal?, ...): Promise<Result<CompactResult, CompactionError>> {
	if (isSplitTurn && turnPrefixMessages.length > 0) {
		// 双段：先摘要历史（可与 previousSummary 增量合并），再单独摘要 turn 前缀
		const historyResult = await generateSummaryWithUsage(messagesToSummarize, ..., previousSummary, ...);
		const turnPrefixResult = await generateTurnPrefixSummary(turnPrefixMessages, ...);
		summary = `${historyText}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult.value.text}`;
		summaryUsage = combineUsage(historyUsage, turnPrefixResult.value.usage);
	} else {
		// 单段：正常摘要（有 previousSummary 则走"增量更新"prompt）
		const summaryResult = await generateSummaryWithUsage(messagesToSummarize, ..., previousSummary, ...);
	}
	// 摘要尾部追加文件操作清单
	const { readFiles, modifiedFiles } = computeFileLists(fileOps);
	summary += formatFileOperations(readFiles, modifiedFiles);
	return ok({ summary, tokensBefore, usage: summaryUsage, retainedTail, details: { readFiles, modifiedFiles } });
}
```

### 7.4 摘要 prompt 工程

两套结构化 prompt（L424-L498）：

- **SUMMARIZATION_PROMPT**（首次）：固定格式 `## Goal / ## Constraints & Preferences / ## Progress (Done|In Progress|Blocked) / ## Key Decisions / ## Next Steps / ## Critical Context`，强调"Preserve exact file paths, function names, and error messages"。
- **UPDATE_SUMMARIZATION_PROMPT**（增量）：基于 `<previous-summary>` 更新——规则是 PRESERVE 全部旧信息、ADD 新信息、UPDATE 进度、可移除不再相关项。

System prompt（L424-L426）强调角色隔离："Do NOT continue the conversation. Do NOT respond to any questions... ONLY output the structured summary."——摘要模型不得代答原对话。

**请求隔离**（L102-L122）：

```typescript
export async function completeSimpleWithRetries(models, model, context, options, ...): Promise<AssistantMessage> {
	// 摘要是独立请求：隔离路由、禁用不可复用的缓存写入
	const requestOptions: SimpleStreamOptions = {
		...options,
		cacheRetention: "none",
		sessionId: uuidv7(),   // 新 sessionId，不污染主会话的 provider 缓存
	};
	return retryAssistantCall(() => models.completeSimple(model, context, requestOptions), ...);
}
```

**教学要点**：`cacheRetention: "none"` + 全新 sessionId——摘要请求**不写 provider 缓存**（写了也无法被主会话复用，纯浪费钱），也不与主会话共享缓存路由。

### 7.5 触发点：coding-agent 侧的 _checkCompaction

生产环境压缩由 `packages/coding-agent/src/core/agent-session.ts`（课程快照 L2040-L2073） 在 turn 结束后驱动（非 agent-core 的 shouldStopAfterTurn）：

```typescript
// Case 3: threshold compaction without retry.
let contextTokens: number;
const directContextTokens = assistantMessage.usage ? calculateContextTokens(assistantMessage.usage) : 0;
if (assistantMessage.stopReason === "error" || directContextTokens === 0) {
	// 错误消息/零用量消息：从最近有效响应估算
	const estimate = estimateContextTokens(messages);
	if (estimate.lastUsageIndex === null) return false;
	// 校验用量来源是压缩后的（防旧用量误触发）
	const usageMsg = messages[estimate.lastUsageIndex];
	if (compactionEntry && usageMsg.role === "assistant"
		&& (usageMsg as AssistantMessage).timestamp <= new Date(compactionEntry.timestamp).getTime()) {
		return false;   // 保留的压缩前消息带旧（偏大）用量，不能作为触发依据
	}
	contextTokens = estimate.tokens;
} else {
	contextTokens = directContextTokens;
}
if (shouldCompact(contextTokens, contextWindow, settings)) {
	return await this._runAutoCompaction("threshold", false);
}
```

**教学要点**（L2054-L2064 注释）：压缩后 retainedTail 里保留的**压缩前** assistant 消息携带旧的（更大的）usage——若拿它当依据会"刚压完又触发压缩"的死循环。代码显式校验用量消息时间戳必须晚于 compaction entry。

`_runAutoCompaction`（L2085-L2170+）流程：`prepareCompaction` → 发 `session_before_compact` 事件让 extension 可 cancel/接管 →（extension 未接管时）`compact()` 生成摘要 → 写 CompactionEntry。**extension 可以完全接管压缩**（L2132-L2135 `extensionResult?.compaction`）——又一个"核心不内置、外包给扩展"的实例。

---

## 第 8 课 pi-ai 层：Provider / Models / streamSimple

**文件**：`packages/ai/src/models.ts`

### 8.1 Provider 接口：运行时单元

`packages/ai/src/models.ts`（课程快照 L97-L149）：

```typescript
export interface Provider<TApi extends Api = Api> {
	readonly id: string;
	readonly name: string;
	readonly baseUrl?: string;
	readonly headers?: ProviderHeaders;
	/** 必须至少有 apiKey/oauth 之一。无凭据 provider（env var、AWS profile）也提供 apiKey 型 auth，
	    其 resolve() 报告 provider 是否已配置 */
	readonly auth: ProviderAuth;

	/** 当前已知模型（同步）。静态 provider 返回目录；动态 provider 返回上次 refresh 结果（首次为空）。
	    不得 throw——throw 视为无模型 */
	getModels(): readonly Model<TApi>[];

	/** 仅动态 provider：恢复 context.stored 并可选拉新列表。
	    失败保留旧列表；持久化与同步状态变更经 context.publish()；遵守共享 abort signal */
	refreshModels?(context: RefreshModelsContext): Promise<void>;

	/** 可选：按凭据过滤模型可用性 */
	filterModels?(models: readonly Model<TApi>[], credential: Credential | undefined): readonly Model<TApi>[];

	stream<T extends TApi>(model: Model<T>, context: Context, options?: ApiStreamOptions<T>): AssistantMessageEventStream;
	streamSimple(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
	fetchDeferred?(model, handle: DeferredHandle, options?): AssistantMessageEventStream;
	cancelDeferred?(model, handle: DeferredHandle, options?): Promise<void>;
}
```

**教学要点**：
1. **静态 vs 动态 provider**：静态（模型清单硬编码）只实现 `getModels()`；动态（如 OpenRouter，模型目录在线变）实现 `refreshModels()`，由 `Models.refresh()` 并发刷新（注释 L172-L177："Provider errors and cancellation are returned without rejecting"——刷新错误不抛出而是收集进 result.errors）。
2. **auth 契约**（注释 L104-L111）：每个 provider 都必须有 auth 语义——连"环境变量/本地无鉴权服务"也建模为 apiKey 型 auth，`resolve()` 报告配置状态。这让 `Models.getAuth()` 能统一回答"这家配置好了吗"。
3. `TApi` 泛型让 provider 工厂能声明自己支持的 API 协议（如 `openaiProvider(): Provider<"openai-responses" | "openai-completions">`），直接用户拿到类型化模型列表。

### 8.2 Models：集合 + 鉴权解析 + 委派

```typescript
// models.ts L152-L155 注释
/**
 * Runtime collection of providers plus auth application and stream
 * convenience. Providers own stream behavior; `Models` resolves auth and
 * delegates each request to the provider that owns the model.
 */
export interface Models {
	getProviders(): readonly Provider[];
	getProvider(id: string): Provider | undefined;
	getModels(provider?: string): readonly Model<Api>[];
	getModel(provider: string, id: string): Model<Api> | undefined;
	refresh(options?: ModelsRefreshOptions): Promise<ModelsRefreshResult>;
	checkAuth(providerId: string, options?): Promise<AuthCheck | undefined>;
	// streamSimple(...) 等
}
```

职责一句话：**Models 解析 auth，把请求委派给拥有该 model 的 provider**。stream 行为属于 provider，Models 只做"鉴权 + 找到 owner + 转发"。

`RefreshModelsContext.publish`（L46-L62）是**代际检查的发布机制**——refresh 期间若发生了新的 refresh（generation 变化），旧的 publish 返回 false 不落盘，防止旧数据覆盖新数据。

### 8.3 Agent 如何用 pi-ai

agent-core 只需要 `streamSimple` 满足 `StreamFn` 形状（types.ts L19-L21 注释："Models.streamSimple satisfies this shape"）。组装示例（概念代码）：

```typescript
import { Models } from "@earendil-works/pi-ai";
import { Agent } from "@earendil-works/pi-agent-core";

const agent = new Agent({
	streamFn: (model, context, options) => models.streamSimple(model, context, options),
	// ...
});
```

**教学要点**：agent 与 pi-ai 的耦合仅一个函数签名——测试时可以塞一个假 StreamFn（同步 yield 假事件），完全离线测 agent 循环。这是本 repo 测试策略的基础。

---

## 第 9 课 扩展系统：jiti 加载 / 虚拟模块 / 事件

**目录**：`packages/coding-agent/src/core/extensions`（loader.ts / runner.ts / wrapper.ts / types.ts / index.ts）

### 9.1 loader：jiti 即时编译 + 虚拟模块

`packages/coding-agent/src/core/extensions/loader.ts`（课程快照 L1-L74）：

```typescript
import { createJiti } from "jiti/static";
// Static imports of packages that extensions may use.
// These MUST be static so Bun bundles them into the compiled binary.
// The virtualModules option then makes them available to extensions.
import * as _bundledTypebox from "typebox";
import * as _bundledPiAgentCore from "@earendil-works/pi-agent-core";
// ...更多静态导入

/** Modules available to extensions via virtualModules (for compiled Bun binary) */
const VIRTUAL_MODULES: Record<string, unknown> = {
	typebox: _bundledTypebox,
	"typebox/compile": _bundledTypeboxCompile,
	"@sinclair/typebox": _bundledTypebox,          // 旧包名别名
	"@earendil-works/pi-agent-core": _bundledPiAgentCore,
	"@earendil-works/pi-tui": _bundledPiTui,
	// Extensions resolve the pi-ai root to the compat entrypoint (a strict
	// superset of the core entrypoint): existing extensions using the old
	// global API keep working at runtime until compat is removed.
	"@earendil-works/pi-ai": _bundledPiAiCompat,
	"@earendil-works/pi-coding-agent": _bundledPiCodingAgent,
	"@mariozechner/pi-agent-core": _bundledPiAgentCore,   // 旧 scope 别名
	// ...
};
```

**教学要点（三层兼容策略）**：
1. **virtualModules（Bun 二进制模式）**：pi 的编译产物是单文件二进制，extension 的 `import "typebox"` 无法走 node_modules——pi 把可用的模块**静态打进二进制**，jiti 的 virtualModules 选项把 import 说明符映射到这些内置模块。
2. **aliases（Node 构建模式，L84-L142）**：普通 Node 安装时用 alias 把 extension 的 import 重定向到 workspace 内的 dist 入口（如 `"@earendil-works/pi-ai": piAiCompatEntry`）。
3. **新旧 scope 双注册**：`@earendil-works/*` 与 `@mariozechner/*` 同时映射——旧扩展（pi 改名前写的）零修改继续跑。

### 9.2 加载缓存与代际失效

```typescript
// loader.ts L146-L148
let extensionCacheCwd: string | undefined;
let extensionCacheGeneration = 0;
const extensionCache = new Map<string, ExtensionFactory>();
```

扩展工厂按路径缓存；generation 递增使旧缓存失效（供 `/reload` 热重载）。

### 9.3 runner：事件分发与保留键位

`packages/coding-agent/src/core/extensions/runner.ts`（课程快照 L69-L90）：

```typescript
// Extension shortcuts compete with canonical keybinding ids from keybindings.json.
// Only editor-global shortcuts are reserved here.
const RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS = [
	"app.interrupt",
	"app.clear",
	"app.exit",
	"app.suspend",
	/* ... */
	"tui.input.submit",
	"tui.select.confirm",
	/* ... */
] as const;
```

`buildBuiltinKeybindings`（L94-L113）：把 keybindings.json 解析成 key→action 映射，保留键（reserved）在冲突时**始终赢**——"If multiple actions bind the same key, the reserved action wins so extensions remain blocked by reserved shortcuts regardless of iteration order"（L102-L104）。扩展不能劫持 Ctrl+C（interrupt）等核心键。

事件分发的类型分层（L125-L138）：有专用 emitXxx 方法的事件（tool_call、project_trust、tool_result、user_bash、context、before_provider_request、before_provider_headers、before_agent_start、message_end、resources_discover、input——都是**有返回值、需要拦截语义**的事件）从通用 `emit()` 的类型中排除，走更强的类型安全通道。

### 9.4 扩展的编写模型（回顾 + 代码）

```typescript
export default function (pi: ExtensionAPI) {
  // 拦截 bash 危险命令
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("危险操作", `允许执行？\n${event.input.command}`);
      if (!ok) return { block: true, reason: "用户拒绝" };
    }
  });

  // 注册工具
  pi.registerTool({
    name: "greet",
    description: "向某人问好",
    parameters: Type.Object({ name: Type.String() }),
    async execute(toolCallId, params) {
      return { content: [{ type: "text", text: `你好，${params.name}!` }] };
    },
  });

  // 注册命令 / 快捷键 / provider
  pi.registerCommand("hello", { handler: async (args, ctx) => ctx.ui.notify(`Hello!`, "info") });
  pi.registerShortcut("ctrl+g", { handler: async (ctx) => { /* ... */ } });
  pi.registerProvider("my-proxy", { baseUrl: "...", apiKey: "...", models: [...], api: "openai-completions" });
}
```

生命周期约定（extensions.md）：async factory 完成后才开始 `session_start`；长生命周期资源在 `session_start` 起、`session_shutdown` 关（factory 可能在不启 session 的调用里跑）。

---

## 第 10 课 Skills 与 System Prompt

### 10.1 渐进式披露的实现

`packages/agent/src/harness/system-prompt.ts`（34 行，全文核心）：

```typescript
export function formatSkillsForSystemPrompt(skills: Skill[]): string {
	const visibleSkills = skills.filter((skill) => !skill.disableModelInvocation);
	if (visibleSkills.length === 0) return "";

	const lines = [
		"The following skills provide specialized instructions for specific tasks.",
		"Read the full skill file when the task matches its description.",
		"When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.",
		"",
		"<available_skills>",
	];

	for (const skill of visibleSkills) {
		lines.push("  <skill>");
		lines.push(`    <name>${escapeXml(skill.name)}</name>`);
		lines.push(`    <description>${escapeXml(skill.description)}</description>`);
		lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
		lines.push("  </skill>");
	}

	lines.push("</available_skills>");
	return lines.join("\n");
}
```

**教学要点**：
1. system prompt 只注入 **name + description + location** 三元组——模型匹配到任务后用现成的 `read` 工具加载 SKILL.md 全文。100 个 skill 也只占几百 token 索引。
2. 第三行指令显式教模型**相对路径解析规则**（相对 skill 目录）——skill 里引用 `./search.js` 时模型能算出绝对路径去 bash 执行。
3. `disableModelInvocation` 的 skill 不进列表（仅供人类 `/skill:name` 调用）。

### 10.2 skill 发现与校验

`packages/coding-agent/src/core/skills.ts`：

- 规范校验（L92-L112）：name ≤64 字符、`/^[a-z0-9-]+$/`、不以连字符开头/结尾、无连续连字符；description ≤1024 字符（Agent Skills 规范）。
- 发现路径：`~/.pi/agent/skills/`、`~/.agents/skills/`（全局）、`.pi/skills/`、`.agents/skills/`（项目，trust 后）、`package.json` 的 `pi.skills`、settings `skills` 数组、`--skill` flag。
- gitignore 感知（L16、L24-L65）：`addIgnoreRules` 读取 `.gitignore/.ignore/.fdignore` 并**按目录前缀改写模式**（`prefixIgnorePattern`）——扫描项目时尊重仓库的忽略规则，不把 node_modules 里的伪 skill 扫出来。

---

## 第 11 课 AgentSession：模式共享骨架

**文件**：`packages/coding-agent/src/core/agent-session.ts`（2200+ 行，coding-agent 最大文件）

头注释（L1-L14）定义其定位：

```
AgentSession - Core abstraction for agent lifecycle and session management.

This class is shared between all run modes (interactive, print, rpc).
It encapsulates:
- Agent state access
- Event subscription with automatic session persistence
- Model and thinking level management
- Compaction (manual and auto)
- Bash execution
- Session switching and branching

Modes use this class and add their own I/O layer on top.
```

**架构角色**：

```
interactive TUI ─┐
print mode ──────┤
JSON mode ───────┼──> AgentSession ──> Agent（agent-core）──> agentLoop ──> StreamFn（pi-ai）
RPC mode ────────┤        │
SDK 嵌入 ────────┘        └──> SessionManager（JSONL 持久化）
                          └──> ExtensionRunner（扩展事件）
                          └──> ModelRegistry（模型管理）
                          └──> SettingsManager（设置/压缩阈值）
```

AgentSession 是**所有模式的共享中间层**——把"agent 生命周期 + 持久化 + 压缩 + 扩展接线"做一次，模式只加 I/O。它订阅 Agent 事件并把消息自动写入 session（"Event subscription with automatic session persistence"），第 7 课的 `_checkCompaction`/`_runAutoCompaction` 也在这里。

**教学要点**：对比 dsh 用 Cordis 插件组合出 web/headless 两种 profile，pi 用"共享类 + 模式薄壳"达到同样效果——不需要框架，一个抽象层就够。

---

## 第 12 课 durable harness：规格 vs 占位实现

### 12.1 AgentHarness 类的现状

`packages/agent/src/harness/agent-harness.ts`：类型完整（RunOutcome / CompactionOutcome / NavigationOutcome、HookName 枚举、AgentHarnessOptions），`constructor`/`create` 已实现，但 **所有 mutating 方法（prompt/compact/navigateTree/resume…）抛 `HarnessNotImplemented`**。

### 12.2 规格的三个核心（docs/harness.md）

**① 三 stores 一不变量**：

```
entries        会话树 — write-once，append-only
registers      可变状态 — namespaced typed cells，overwrite/delete
usage ledger   成本历史 — append-only rows
```

**② durable 程序计数器**：每步之后 overwrite register `op.state/{operationId}`，存**完整**当前 operation state（total state，不依赖前态）。恢复时不 replay journal、不推断——读 register、switch on it。

**③ effect sandwich**：

```
TX: "about to do X; output will use ids R and U"     ← intent（预登记输出 id）
    do X                                              ← 不确定窗口（provider 请求/工具执行）
TX: output + usage + next state                       ← settlement（回填）
```

崩溃在 effect 中间 → 读 op.state 见 `effect_pending` + replay policy：`replay: "never"`（删文件等）注入合成 "interrupted" 结果、不重跑；`replay: "safe"`（读/查询）用持久化参数重跑。

### 12.3 存储层已就位、执行层未接线

第 6 课已展示：`ToolStartedRecord.replay` 字段、`operation_started/finished` record、seq 严格连续的原子性——**存储词汇表已经为 effect sandwich 备好**。缺的是 AgentHarness 执行层把这些 record 真正写起来、崩溃恢复逻辑真正跑起来。

**教学要点**：这是"规格先行、渐进实现"的工程样本——JSONL 后端（已生产）与 harness 规格（在制品）共享同一套 mutation 模型，未来接线不需要迁移存储格式。

---

## 第 13 课 总复习：设计模式清单 + 练习

### 13.1 本课程提取的设计模式

| 模式 | 出处 | 一句话 |
|---|---|---|
| **错误即数据** | StreamFn 契约、工具 throw、handleRunFailure | 失败编码进合法事件流，循环层零 try/catch |
| **双层消息管道** | transformContext → convertToLlm | transcript 语义与 LLM 语义分离，自定义消息类型靠 declaration merging |
| **单飞行 + 双队列** | Agent.activeRun / steering / followUp | 并发 prompt 拒绝，插话走队列，drain 点注入 |
| **批次 terminate AND 语义** | shouldTerminateToolBatch | 早停是全批共识，单个工具说了不算 |
| **事件顺序与消息顺序解耦** | executeToolCallsParallel | end 事件按完成序、消息按源序，两头都对 |
| **write-ahead 内存提交** | appendEntry 先落盘再 applyMutation | 崩溃后重放幂等重建 |
| **单写队列** | JsonlSessionStorage.enqueue | promise 链串行化所有写，文件永不交错 |
| **torn-tail 判定** | load() 最后一行+语法错 | 追加写崩溃只损坏末行，中间损坏即报错 |
| **原子发布** | publishFileAtomically | 临时文件 + rename，崩溃只留 .tmp |
| **真值锚点+增量估算** | estimateContextTokens | provider usage 为锚，其后启发式 |
| **虚拟切点回退** | prepareCompaction 的 retainedTail 展开 | 上一轮保留尾部参与本轮切分 |
| **请求隔离** | 摘要的 cacheRetention:none + 新 sessionId | 一次性请求不污染可复用缓存 |
| **函数注入反转** | Agent 只认 StreamFn 签名 | agent 与 pi-ai 解耦，可离线测试 |
| **虚拟模块 + 双 scope 别名** | extension loader | 单文件二进制内给扩展提供 import，旧名兼容 |
| **保留键位优先** | runner 的 reserved keybindings | 扩展不能劫持核心快捷键 |

### 13.2 练习

**练习 1（事件流）**：不运行代码，手写以下场景的完整事件序列：prompt("hi") → 模型回复一次工具调用 bash("ls") → 工具成功 → 模型纯文本回复。标注每个事件的关键字段值。

**练习 2（ terminate 语义）**：一个 assistant 消息含三个工具调用，结果分别是 A(terminate:true)、B(terminate:true)、C(terminate:false)。问：内层循环是否继续？依据是哪行代码？

**练习 3（seq 连续性）**：一个 JSONL 文件的 mutation seq 为 1,2,3,5。load 时发生什么？如果是 1,2,3,4 且第 4 行 JSON 语法损坏呢？两种情况行为为何不同？

**练习 4（切点））**：keepRecentTokens=20000，从尾部累加到 18000 时遇到全是 toolResult 的 5000 token 段。描述 findCutPoint 的行为与最终切点位置。

**练习 5（实现题）**：给 Agent 加一个 `beforeProviderRequest` 钩子（在 streamFn 调用前触发、可修改即将发送的 llmContext）。指出需要改动的最小文件集合与插入点（提示：streamAssistantResponse 内、第 5 步之前）。

**练习 6（对比题）**：用第 6 课的知识解释：为什么 pi 当前版本崩溃恢复无法回答"那个删文件的 bash 到底跑没跑"？harness.md 的哪三个机制合起来解决这个问题？

### 13.3 延伸阅读

- `packages/agent/docs/harness.md`——durable runtime 完整规格（224KB，9 part + 3 appendix）
- `packages/agent/README.md`——agent 包 API 文档，钩子语义的权威描述
- `packages/coding-agent/docs/extensions.md`——ExtensionAPI 全量事件/方法参考

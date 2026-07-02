# Chapter 9: Fork Agents and the Prompt Cache

# 第 9 章：Fork Agent 与 Prompt 缓存

## The Ninety-Five Percent Insight

## 九成五的洞察

When a parent agent spawns five child agents in parallel, the overwhelming majority of each child's API request is identical. The system prompt is the same. The tool definitions are the same. The conversation history is the same. The assistant message that triggered the spawns is the same. The only thing that differs is the final directive: "you handle the database migration," "you write the tests," "you update the docs."

当一个父 agent 并行派生出五个子 agent 时，每个子 agent 的 API 请求中绝大部分内容都是完全相同的。系统提示词相同。工具定义相同。对话历史相同。触发派生的那条 assistant 消息也相同。唯一不同的，是最后那条指令："你来处理数据库迁移"、"你来写测试"、"你来更新文档"。

On a typical fork with a warm conversation, the shared prefix might be 80,000 tokens. The per-child directive might be 200 tokens. That is 99.75% overlap. Anthropic's prompt cache gives a 90% discount on cached input tokens. If you can make those 80,000 tokens hit the cache for children 2 through 5, you just cut the input cost of those four requests by 90%. For the parent, this is the difference between spending $4 and spending $0.50 on the same parallel dispatch.

在一次典型的 fork 中，如果对话已经"热"起来了，共享前缀可能有 80,000 个 token，而每个子 agent 各自的指令可能只有 200 个 token。这意味着 99.75% 的重叠。Anthropic 的 prompt 缓存对命中缓存的输入 token 给予 90% 的折扣。如果你能让那 80,000 个 token 在第 2 到第 5 个子 agent 上命中缓存，你就把这四个请求的输入成本砍掉了 90%。对父 agent 来说，这就是同一次并行派发花掉 4 美元还是 0.50 美元的区别。

The catch is that prompt caching is byte-exact. Not "similar enough." Not "semantically equivalent." The bytes must match, character for character, from the first byte of the system prompt through to the last byte before the per-child content diverges. One extra space, one reordered tool definition, one stale feature flag changing a system prompt fragment -- and the cache misses. The entire prefix is reprocessed at full price.

问题在于，prompt 缓存是逐字节精确匹配的。不是"足够相似"，也不是"语义等价"。从系统提示词的第一个字节，一直到各子 agent 内容开始分叉前的最后一个字节，必须逐字符完全一致。多一个空格、调换一处工具定义的顺序、一个过时的 feature flag 改变了系统提示词的某个片段——缓存就会失效。整个前缀将以全价被重新处理。

Fork agents are Claude Code's answer to this constraint. They are not just a convenience for "spawn a child with context" -- they are a prompt cache exploitation mechanism disguised as an orchestration feature. Every design decision in the fork system traces back to one question: how do we guarantee byte-identical prefixes across parallel children?

Fork agent 就是 Claude Code 对这一约束给出的答案。它们不只是"派生一个带上下文的子 agent"这样的便利功能——而是一种伪装成编排特性的 prompt 缓存利用机制。fork 系统中的每一个设计决策，都可以追溯到同一个问题：我们如何保证并行子 agent 之间的前缀逐字节完全一致？

---

## What a Fork Child Inherits

## Fork 子 agent 继承了什么

A fork agent inherits four things from its parent, and it inherits them by reference or byte-exact copy, not by recomputation.

一个 fork agent 从父 agent 那里继承四样东西，而且是通过引用或逐字节精确拷贝来继承的，绝不重新计算。

**1. The system prompt.** Not regenerated -- threaded. The parent's already-rendered system prompt bytes are passed via `override.systemPrompt`, pulled from `toolUseContext.renderedSystemPrompt`. This is the exact string that was sent in the parent's most recent API call.

**1. 系统提示词。** 不是重新生成——而是直接传递（threaded）。父 agent 已经渲染好的系统提示词字节，通过 `override.systemPrompt` 传入，取自 `toolUseContext.renderedSystemPrompt`。这正是父 agent 在最近一次 API 调用中发送的那个字符串。

**2. The tool definitions.** The fork agent definition declares `tools: ['*']`, but with the `useExactTools` flag set to true, the child receives the parent's assembled tool array directly. No filtering, no reordering, no re-serialization.

**2. 工具定义。** fork agent 的定义声明了 `tools: ['*']`，但由于 `useExactTools` 标志被置为 true，子 agent 会直接接收父 agent 已组装好的工具数组。不做过滤，不做重排序，不做重新序列化。

**3. The conversation history.** Every message the parent has exchanged with the API -- user turns, assistant turns, tool calls, tool results -- is cloned into the child's context via `forkContextMessages`.

**3. 对话历史。** 父 agent 与 API 交换过的每一条消息——user 回合、assistant 回合、工具调用、工具结果——都通过 `forkContextMessages` 克隆进子 agent 的上下文。

**4. The thinking configuration and model.** The fork definition specifies `model: 'inherit'`, which resolves to the parent's exact model. Same model means same tokenizer, same context window, same cache namespace.

**4. thinking 配置与模型。** fork 定义指定了 `model: 'inherit'`，它会解析为父 agent 完全相同的模型。相同的模型意味着相同的 tokenizer、相同的上下文窗口、相同的缓存命名空间。

The fork agent definition itself is minimal -- almost a no-op:

fork agent 的定义本身极其精简——几乎是个空操作：

The fork agent definition is deliberately minimal -- it inherits everything from the parent. It specifies all tools (`'*'`), inherits the parent's model, uses bubble mode for permissions (so prompts surface in the parent's terminal), and provides a no-op system prompt function that is never actually called -- the real prompt arrives via the override channel, already rendered and byte-stable.

fork agent 的定义被刻意设计得极简——它从父 agent 那里继承一切。它指定使用全部工具（`'*'`），继承父 agent 的模型，权限上使用 bubble 模式（这样提示会浮现在父 agent 的终端里），并提供一个永远不会真正被调用的空操作（no-op）系统提示词函数——真正的提示词通过 override 通道送达，早已渲染完成且逐字节稳定。

---

## The Byte-Identical Prefix Trick

## 逐字节相同的前缀技巧

The API request to Claude has a specific structure: system prompt, then tools, then messages. For the prompt cache to hit, every byte from the start of the request through to some prefix boundary must be identical across requests.

发给 Claude 的 API 请求有一个固定的结构：先是系统提示词，然后是工具，再然后是消息。要让 prompt 缓存命中，从请求开头到某个前缀边界之间的每一个字节，在多个请求之间都必须完全一致。

Fork agents achieve this by ensuring three layers are frozen:

fork agent 通过冻结三个层次来达成这一点：

**Layer 1: System prompt via threading, not recomputation.**

**第 1 层：系统提示词通过传递获得，而非重新计算。**

When the parent agent's system prompt was rendered for its last API call, the result was captured in `toolUseContext.renderedSystemPrompt`. This is the string after all dynamic interpolation -- GrowthBook feature flags, environment details, MCP server descriptions, skill content, CLAUDE.md files. The fork child receives this exact string.

当父 agent 为它最近一次 API 调用渲染系统提示词时，结果被捕获进了 `toolUseContext.renderedSystemPrompt`。这是经过所有动态插值之后的字符串——GrowthBook feature flag、环境细节、MCP 服务器描述、skill 内容、CLAUDE.md 文件等等。fork 子 agent 收到的正是这个一字不差的字符串。

Why not just call `getSystemPrompt()` again? Because system prompt generation is not pure. GrowthBook flags transition from cold to warm state as the SDK fetches remote config. A flag that returned `false` during the parent's first turn might return `true` by the time the fork child spins up. If the system prompt includes a conditional block gated by that flag, the re-rendered prompt diverges by even a single character. Cache busted. Full-price reprocessing of 80,000 tokens, times five children.

为什么不干脆再调用一次 `getSystemPrompt()`？因为系统提示词的生成不是纯函数。随着 SDK 拉取远程配置，GrowthBook flag 会从冷状态过渡到热状态。在父 agent 第一个回合中返回 `false` 的某个 flag，到 fork 子 agent 启动时可能就返回 `true` 了。如果系统提示词里包含一个由该 flag 控制的条件块，那么重新渲染出来的提示词哪怕只差一个字符——缓存就废了。80,000 个 token 全价重新处理，再乘以五个子 agent。

Threading the rendered bytes eliminates this entire class of divergence.

直接传递已渲染的字节，就彻底消除了这一整类分叉问题。

**Layer 2: Tool definitions via exact passthrough.**

**第 2 层：工具定义通过精确透传获得。**

Normal sub-agents go through `resolveAgentTools()`, which filters the tool pool based on the agent definition's `tools` and `disallowedTools` arrays, applies permission mode differences, and potentially reorders tools. The resulting serialized tool array would differ from the parent's -- different subset, different order, different permission annotations.

普通的子 agent 会走 `resolveAgentTools()`，它会根据 agent 定义中的 `tools` 和 `disallowedTools` 数组来过滤工具池，应用权限模式上的差异，还可能对工具重新排序。最终序列化出来的工具数组会与父 agent 的不同——子集不同、顺序不同、权限注解不同。

Fork agents skip this entirely:

fork agent 完全跳过这一步：

```typescript
const resolvedTools = useExactTools
  ? availableTools  // parent's exact array
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```

The `useExactTools` flag is set to true only on the fork path. The child gets the parent's tool pool as-is. Same tools, same order, same serialization. This includes keeping the Agent tool itself in the child's pool, even though the child is forbidden from using it -- removing it would change the tool array and bust the cache.

`useExactTools` 标志只有在 fork 路径上才会被置为 true。子 agent 原封不动地拿到父 agent 的工具池。相同的工具、相同的顺序、相同的序列化结果。这甚至包括把 Agent 工具本身保留在子 agent 的工具池中——尽管子 agent 是被禁止使用它的——因为移除它会改变工具数组，从而破坏缓存。

**Layer 3: Message array construction.**

**第 3 层：消息数组的构建。**

This is where `buildForkedMessages()` does its careful work. The function constructs the final two messages that sit between the shared history and the per-child directive:

这正是 `buildForkedMessages()` 精心操作的地方。该函数构造出夹在共享历史与各子 agent 指令之间的最后两条消息：

The `buildForkedMessages()` function constructs the final two messages that sit between the shared history and the per-child directive. The algorithm:

`buildForkedMessages()` 函数构造出夹在共享历史与各子 agent 指令之间的最后两条消息。算法如下：

1. Clone the parent's assistant message (preserving all `tool_use` blocks with their original IDs).

1. 克隆父 agent 的 assistant 消息（保留所有 `tool_use` 块及其原始 ID）。

2. For each `tool_use` block, create a `tool_result` with a constant placeholder string (identical across all children).

2. 对每一个 `tool_use` 块，创建一个带有常量占位字符串的 `tool_result`（在所有子 agent 之间完全一致）。

3. Build a single user message containing all the placeholder results followed by the per-child directive wrapped in the boilerplate tag.

3. 构建一条 user 消息，其中包含所有占位结果，紧随其后的是被包裹在样板标签（boilerplate tag）中的各子 agent 指令。

4. Return `[clonedAssistantMessage, userMessageWithPlaceholdersAndDirective]`.

4. 返回 `[clonedAssistantMessage, userMessageWithPlaceholdersAndDirective]`。

```typescript
// Pseudocode — illustrates the message construction
function buildChildMessages(directive, parentAssistant) {
  const cloned = cloneMessage(parentAssistant)
  const placeholders = parentAssistant.toolUseBlocks.map(b =>
    toolResult(b.id, CONSTANT_PLACEHOLDER)  // Byte-identical across children
  )
  const userMsg = createUserMessage([...placeholders, wrapDirective(directive)])
  return [cloned, userMsg]
}
```

The resulting message array for each child looks like:

每个子 agent 最终得到的消息数组形如：

```
[...shared_history, assistant(all_tool_uses), user(placeholder_results..., directive)]
```

Every element before the directive is identical across children. The `FORK_PLACEHOLDER_RESULT` -- a constant string `'Fork started -- processing in background'` -- ensures even the tool result blocks are byte-identical. The `tool_use_id` values are identical because they reference the same assistant message. Only the final text block, containing the per-child directive, varies.

指令之前的每一个元素，在所有子 agent 之间都完全相同。`FORK_PLACEHOLDER_RESULT`——一个常量字符串 `'Fork started -- processing in background'`——确保了连工具结果块也是逐字节一致的。`tool_use_id` 的值也都相同，因为它们引用的是同一条 assistant 消息。唯一变化的，是包含各子 agent 指令的那个最后的文本块。

The cache boundary falls right before that final text block. Everything above it -- potentially tens of thousands of tokens of system prompt, tool definitions, conversation history, and placeholder results -- hits the cache at a 90% discount for every child after the first.

缓存边界恰好落在那个最后的文本块之前。它上方的一切——可能多达数万个 token 的系统提示词、工具定义、对话历史与占位结果——对第一个子 agent 之后的每一个子 agent 来说，都以 90% 的折扣命中缓存。

---

## The Fork Boilerplate Tag

## Fork 样板标签

Each child's directive is wrapped in a boilerplate XML tag that serves two purposes: it instructs the child on how to behave, and it acts as a marker for recursive fork detection.

每个子 agent 的指令都被包裹在一个样板 XML 标签中，这个标签有两个用途：一是指导子 agent 该如何行动，二是充当递归 fork 检测的标记。

The boilerplate contains approximately 10 rules. The key ones:

样板中包含大约 10 条规则。其中关键的几条：

- **Override the parent's forking instruction.** The parent's system prompt says "default to forking" -- the boilerplate explicitly tells the child: "that instruction is for the parent. You ARE the fork. Do NOT spawn sub-agents."

- **覆盖父 agent 的 fork 指令。** 父 agent 的系统提示词写着"默认进行 fork"——而样板明确告诉子 agent："那条指令是给父 agent 的。你本身就是那个 fork。不要再派生子 agent。"

- **Execute silently, report once.** No conversational text between tool calls. Use tools directly, then produce a structured summary.

- **静默执行，只汇报一次。** 工具调用之间不要有任何对话性文字。直接使用工具，然后产出一份结构化的总结。

- **Stay within scope.** The child must not expand beyond its directive.

- **不越界。** 子 agent 不得超出其指令的范围行事。

- **Structured output format.** The response must follow a Scope/Result/Key files/Files changed/Issues template that makes results easy for the parent to parse when multiple children report back simultaneously.

- **结构化输出格式。** 响应必须遵循 Scope/Result/Key files/Files changed/Issues 模板，这样当多个子 agent 同时回报时，父 agent 能轻松解析结果。

Rule 1 is particularly interesting. The parent's system prompt -- which the fork child inherits verbatim for cache reasons -- contains instructions like "default to forking when you have parallel work." If the child followed that instruction, it would try to fork its own children, creating an infinite recursion of agents. The boilerplate explicitly overrides: "that instruction is for the parent. You ARE the fork."

第 1 条规则尤其有意思。父 agent 的系统提示词——出于缓存原因，fork 子 agent 会一字不差地继承它——里面含有诸如"当你有并行工作时默认进行 fork"这样的指令。如果子 agent 真照做了，它就会试图 fork 出自己的子 agent，从而造成 agent 的无限递归。样板对此进行了显式覆盖："那条指令是给父 agent 的。你本身就是那个 fork。"

The structured output format (Scope/Result/Key files/Files changed/Issues) is not decorative. It constrains the child's output to factual reporting, which makes the results easier for the parent to parse and aggregate when five children report back simultaneously.

结构化输出格式（Scope/Result/Key files/Files changed/Issues）并非装饰。它把子 agent 的输出约束为事实性的汇报，这样当五个子 agent 同时回报时，父 agent 能更容易地解析并汇总结果。

---

## Recursive Fork Prevention

## 递归 Fork 的防范

The fork child keeps the Agent tool in its tool pool. It has to -- removing it would change the serialized tool array and bust the prompt cache. But if the child actually invokes the Agent tool without `subagent_type`, the fork path would trigger again, creating a grandchild fork. This grandchild would inherit an even larger context (parent + child conversation), spawn its own forks, and so on.

fork 子 agent 的工具池里保留着 Agent 工具。它不得不这么做——移除它会改变序列化后的工具数组，从而破坏 prompt 缓存。但如果子 agent 真的在不带 `subagent_type` 的情况下调用了 Agent 工具，fork 路径就会再次触发，产生一个"孙辈" fork。这个孙辈会继承一个更大的上下文（父 agent + 子 agent 的对话），再派生出它自己的 fork，如此往复。

Two guards prevent this:

有两道防线阻止这种情况：

**Primary guard: querySource check.** When a fork child is spawned, its `context.options.querySource` is set to `'agent:builtin:fork'`. The `call()` method checks this before allowing the fork path:

**主防线：querySource 检查。** 当一个 fork 子 agent 被派生时，它的 `context.options.querySource` 会被设为 `'agent:builtin:fork'`。`call()` 方法在允许走 fork 路径之前会检查这一点：

```typescript
// In AgentTool.call():
if (effectiveType === undefined) {
  // Fork path -- but are we already in a fork?
  if (querySource === 'agent:builtin:fork') {
    // Reject: already a fork child
  }
}
```

This is the fast path. It checks a single string in the options object.

这是快速路径。它只检查 options 对象中的一个字符串。

**Fallback guard: message scanning.** Fork prevention uses two guards: the `querySource` tag set at spawn time (the fast path -- a single string comparison), and a fallback that scans message history for the boilerplate XML tag. The fallback exists because the `querySource` survives autocompact, but in edge cases where it was not properly threaded, the message-scanning fallback catches the recursion. It is a belt-and-suspenders approach where the cost of the check (scanning messages) is trivial compared to the cost of accidental recursive forking (runaway API spend).

**后备防线：消息扫描。** fork 防范使用两道防线：派生时设置的 `querySource` 标签（快速路径——一次字符串比较），以及一个在消息历史中扫描样板 XML 标签的后备方案。之所以需要这个后备方案，是因为 `querySource` 虽然能在 autocompact 后幸存下来，但在某些它没有被正确传递的边界情况下，消息扫描这个后备方案就能捕获到递归。这是一种"双保险"（belt-and-suspenders）的做法——检查的代价（扫描消息）相比意外递归 fork 的代价（失控的 API 花销）微不足道。

Why the fallback? Because Claude Code has an autocompact feature that rewrites the message array when context gets too long. Autocompact can rewrite message content but preserves the `querySource` in options. In theory, `querySource` alone is sufficient. In practice, the message-scanning fallback catches edge cases where `querySource` was not properly threaded -- a belt-and-suspenders approach where the cost of the check (scanning messages) is trivial compared to the cost of accidental recursive forking (runaway API spend).

为什么要有后备方案？因为 Claude Code 有一个 autocompact 特性，会在上下文过长时重写消息数组。autocompact 可以重写消息内容，但会保留 options 中的 `querySource`。理论上，单靠 `querySource` 就够了。但在实践中，消息扫描这个后备方案能捕获 `querySource` 未被正确传递的边界情况——这是一种双保险的做法，其中检查的代价（扫描消息）相比意外递归 fork 的代价（失控的 API 花销）微不足道。

---

## The Sync-to-Async Transition

## 从同步到异步的切换

A fork child starts running in the foreground: its messages stream to the parent's terminal, and the parent blocks waiting for completion. But what if the child is taking too long? Claude Code allows mid-execution backgrounding -- the user (or an auto-timeout) can push a running foreground agent into the background without losing any work.

fork 子 agent 一开始是在前台运行的：它的消息流式输出到父 agent 的终端，父 agent 则阻塞着等待它完成。但如果子 agent 耗时太久怎么办？Claude Code 允许在执行过程中转入后台——用户（或自动超时）可以把一个正在运行的前台 agent 推入后台，且不丢失任何已完成的工作。

The mechanism is surprisingly clean:

这个机制出人意料地干净利落：

1. When a foreground agent is registered via `registerAgentForeground()`, a background signal promise is created.

1. 当一个前台 agent 通过 `registerAgentForeground()` 注册时，会创建一个后台信号 promise。

2. The parent's sync loop races between the agent's message stream and the background signal:

2. 父 agent 的同步循环在 agent 的消息流与后台信号之间进行竞速（race）：

```
while (true) {
  const result = await Promise.race([
    iterator.next(),         // next message from agent
    backgroundSignal,        // "move to background" trigger
  ])
  if (result === BACKGROUND_SIGNAL) break
  // ... process message
}
```

3. When the background signal fires, the foreground iterator is gracefully terminated via `iterator.return()`. This triggers the generator's `finally` block, which handles cleanup.

3. 当后台信号触发时，前台迭代器会通过 `iterator.return()` 被优雅地终止。这会触发 generator 的 `finally` 块，由它来处理清理工作。

4. A new `runAgent()` instance is spawned with `isAsync: true`, using the same agent ID and the message history accumulated so far. The agent continues from where it left off, now running in the background.

4. 一个新的 `runAgent()` 实例以 `isAsync: true` 被派生出来，沿用相同的 agent ID 以及目前为止累积的消息历史。该 agent 从它中断的地方继续，此时已在后台运行。

5. The original synchronous `call()` returns `{ status: 'async_launched' }`, and the parent continues its conversation.

5. 最初那个同步的 `call()` 返回 `{ status: 'async_launched' }`，父 agent 随即继续它的对话。

No work is lost because the message history is the agent's state. The sidechain transcript on disk has every message the agent has produced. The new async instance replays from this transcript and picks up where the sync instance stopped.

没有任何工作会丢失，因为消息历史就是 agent 的状态。磁盘上的旁路（sidechain）记录保存了该 agent 产出的每一条消息。新的异步实例从这份记录回放，并从同步实例停下的地方接着往下走。

---

## Auto-Backgrounding

## 自动转入后台

When the `CLAUDE_AUTO_BACKGROUND_TASKS` environment variable or the `tengu_auto_background_agents` GrowthBook flag is enabled, foreground agents are automatically backgrounded after 120 seconds:

当 `CLAUDE_AUTO_BACKGROUND_TASKS` 环境变量或 `tengu_auto_background_agents` GrowthBook flag 被启用时，前台 agent 会在 120 秒后自动转入后台：

When enabled via environment variable or feature flag, foreground agents are automatically backgrounded after 120 seconds. When disabled, the function returns 0 (no auto-backgrounding).

当通过环境变量或 feature flag 启用时，前台 agent 会在 120 秒后自动转入后台。当禁用时，该函数返回 0（不自动转入后台）。

This is a UX decision with cost implications. A foreground agent blocks the parent terminal -- the user cannot type, cannot issue new instructions, cannot spawn other agents. Two minutes is long enough for the agent to complete most quick tasks synchronously (where the streaming output is useful feedback), but short enough that long-running tasks do not hold the terminal hostage.

这是一个带有成本含义的 UX 决策。一个前台 agent 会阻塞父 agent 的终端——用户无法输入、无法下达新指令、无法派生其他 agent。两分钟足够让 agent 同步完成大多数快速任务（此时流式输出是有用的反馈），又短到不至于让长时间运行的任务把终端"扣为人质"。

Under the fork experiment, the auto-backgrounding question is moot: all fork spawns are forced async from the start. The `run_in_background` parameter is hidden from the schema entirely. Every fork child runs in the background, reports back via a `<task-notification>` when done, and the parent never blocks.

在 fork 实验下，自动转入后台这个问题就无关紧要了：所有 fork 派生从一开始就被强制为异步。`run_in_background` 参数被完全从 schema 中隐藏掉。每个 fork 子 agent 都在后台运行，完成后通过一条 `<task-notification>` 回报，父 agent 从不阻塞。

---

## When Fork Is NOT Used

## 何时不使用 Fork

Fork is one of several orchestration modes, and it is deliberately excluded in three cases:

Fork 是数种编排模式之一，在以下三种情况下它会被刻意排除：

**Coordinator mode.** Coordinator mode and fork mode are mutually exclusive. A coordinator has a structured delegation model: it maintains a plan, assigns tasks to workers with explicit prompts, and tracks progress. Fork's "inherit everything" approach would undermine this. A forked coordinator would inherit the parent coordinator's system prompt (which says "you are the coordinator, delegate work"), and the child would try to orchestrate instead of execute. The `isForkSubagentEnabled()` function checks `isCoordinatorMode()` first and returns false if active.

**协调者（Coordinator）模式。** 协调者模式与 fork 模式互斥。协调者拥有一套结构化的委派模型：它维护一份计划，用明确的提示词把任务分配给各个 worker，并跟踪进度。fork 那套"继承一切"的做法会破坏这一点。一个被 fork 出来的协调者会继承父协调者的系统提示词（其中写着"你是协调者，去委派工作"），于是这个子 agent 会试图去编排而非执行。`isForkSubagentEnabled()` 函数会先检查 `isCoordinatorMode()`，若处于激活状态则返回 false。

**Non-interactive sessions.** SDK and API consumers (`--print` mode, Claude Agent SDK) operate without a terminal. Fork's `permissionMode: 'bubble'` surfaces permission prompts to the parent terminal -- which does not exist in non-interactive mode. Rather than building a separate permission flow, the fork path is simply disabled. SDK consumers use explicit `subagent_type` selection instead.

**非交互式会话。** SDK 和 API 的使用方（`--print` 模式、Claude Agent SDK）在没有终端的情况下运行。fork 的 `permissionMode: 'bubble'` 会把权限提示浮现到父 agent 的终端——而在非交互模式下这个终端根本不存在。与其去单独搭建一套权限流程，不如干脆禁用 fork 路径。SDK 使用方转而通过显式选择 `subagent_type` 来实现。

**Explicit subagent_type.** When the model specifies a `subagent_type` (e.g., `"Explore"`, `"Plan"`, `"general-purpose"`), the fork path is not triggered. Fork only fires when `subagent_type` is omitted. This lets the model choose between "I want a specialized agent with its own system prompt and tool set" (explicit type) and "I want a context-inheriting clone of myself to handle this in parallel" (omitted type).

**显式指定 subagent_type。** 当模型指定了 `subagent_type`（例如 `"Explore"`、`"Plan"`、`"general-purpose"`）时，fork 路径不会被触发。只有在省略 `subagent_type` 时 fork 才会触发。这让模型可以在两者之间做选择："我想要一个拥有自己系统提示词和工具集的专用 agent"（显式类型）与"我想要一个继承上下文、能并行处理此事的我自己的克隆体"（省略类型）。

---

## The Economics

## 经济账

Consider a concrete scenario. A developer asks Claude Code to refactor a module. The parent agent analyzes the codebase, forms a plan, and dispatches five fork children in parallel: one to update the database schema, one to rewrite the service layer, one to update the router, one to fix the tests, and one to update the types.

来看一个具体的场景。一位开发者让 Claude Code 重构一个模块。父 agent 分析代码库，形成一份计划，然后并行派发五个 fork 子 agent：一个更新数据库 schema，一个重写服务层，一个更新路由，一个修复测试，一个更新类型。

At this point in the conversation, the shared context is substantial:
- System prompt: ~4,000 tokens
- Tool definitions (40+ tools): ~12,000 tokens
- Conversation history (analysis + planning): ~30,000 tokens
- Assistant message with five tool_use blocks: ~2,000 tokens
- Placeholder tool results: ~500 tokens

到对话进行到这一步时，共享的上下文已相当可观：
- 系统提示词：约 4,000 个 token
- 工具定义（40 多个工具）：约 12,000 个 token
- 对话历史（分析 + 规划）：约 30,000 个 token
- 含五个 tool_use 块的 assistant 消息：约 2,000 个 token
- 占位的工具结果：约 500 个 token

Total shared prefix: ~48,500 tokens. Per-child directive: ~200 tokens.

共享前缀总计：约 48,500 个 token。每个子 agent 各自的指令：约 200 个 token。

Without fork (five independent agents, each with fresh context and their own system prompt):
- Each child processes its own system prompt + tools + task prompt
- No cache sharing (different system prompts, different tool sets)
- Cost: 5 x full input processing

不使用 fork 的情况下（五个独立的 agent，各自拥有全新的上下文和自己的系统提示词）：
- 每个子 agent 都处理它自己的系统提示词 + 工具 + 任务提示词
- 没有缓存共享（系统提示词不同，工具集不同）
- 成本：5 倍的全价输入处理

With fork (byte-identical prefixes):
- Child 1: 48,700 tokens at full price (cache miss on first request)
- Children 2-5: 48,500 tokens at 10% price (cache hit) + 200 tokens at full price each
- Effective cost for children 2-5: ~4,850 + 200 = ~5,050 tokens equivalent each

使用 fork 的情况下（逐字节相同的前缀）：
- 第 1 个子 agent：48,700 个 token 按全价计（首次请求缓存未命中）
- 第 2 到第 5 个子 agent：48,500 个 token 按 10% 价格计（缓存命中）+ 各自 200 个 token 按全价计
- 第 2 到第 5 个子 agent 的有效成本：约 4,850 + 200 = 每个约等于 5,050 个 token

The savings scale with context size and child count. For a warm session with 100K tokens of history spawning 8 parallel forks, the cache savings can exceed 90% of what the input tokens would have cost without sharing.

节省的成本会随上下文大小和子 agent 数量而放大。对于一个拥有 10 万 token 历史、并派生 8 个并行 fork 的"热"会话，缓存所节省的部分可以超过——若不共享时——这些输入 token 本应花费金额的 90%。

This is why every design decision in the fork system -- the threading instead of recomputation, the exact tool passthrough, the placeholder results, even keeping the Agent tool in the child's pool despite it being forbidden -- optimizes for one thing: byte-identical prefixes. Each decision trades a small amount of elegance or safety for a measurable reduction in API cost.

这就是为什么 fork 系统中的每一个设计决策——用传递取代重新计算、工具的精确透传、占位结果、乃至明明禁用却仍把 Agent 工具保留在子 agent 工具池中——都在为同一件事做优化：逐字节相同的前缀。每个决策都是用少许的优雅或安全性，去换取 API 成本上可度量的下降。

---

## Design Tensions

## 设计上的张力

The fork system makes explicit trade-offs that are worth understanding:

fork 系统做出了一些显式的权衡，值得理解：

**Isolation vs. cache efficiency.** Fork children inherit everything, including conversation history that may be irrelevant to their task. A child rewriting tests does not need the 15 messages where the parent discussed database schema design. But including those messages is what makes the prefix identical. Stripping irrelevant history would save context window space at the cost of busting the cache. The design bet is that cache savings outweigh the context overhead.

**隔离性 vs. 缓存效率。** fork 子 agent 继承一切，包括可能与其任务无关的对话历史。一个重写测试的子 agent 并不需要父 agent 讨论数据库 schema 设计的那 15 条消息。但正是把这些消息包含进来，才使前缀保持一致。剥离掉无关的历史固然能省下上下文窗口空间，代价却是破坏缓存。这个设计赌的是：缓存所省下的，超过了上下文带来的额外开销。

**Safety vs. cache efficiency.** The Agent tool stays in the fork child's tool pool even though the child must not use it. Removing it would be safer (the child cannot even attempt to fork), but would change the tool array serialization. The boilerplate tag and recursive fork guards are the compensating controls -- runtime prevention instead of static removal.

**安全性 vs. 缓存效率。** Agent 工具留在 fork 子 agent 的工具池里，尽管子 agent 绝不能使用它。移除它会更安全（子 agent 连尝试 fork 都做不到），但会改变工具数组的序列化结果。样板标签和递归 fork 防线就是补偿性的控制手段——用运行时防范取代静态移除。

**Simplicity vs. cache efficiency.** The placeholder tool results are a lie. The child sees `'Fork started -- processing in background'` for every tool_use block in the parent's assistant message, regardless of what those tool calls actually did. This is fine because the child's directive tells it what to do -- it does not need accurate tool results from the parent's dispatching turn. But it means the child's conversation history is technically incoherent. The placeholder is chosen for brevity and uniformity, not accuracy.

**简洁性 vs. 缓存效率。** 占位的工具结果是一个谎言。对于父 agent assistant 消息中的每一个 tool_use 块，子 agent 看到的都是 `'Fork started -- processing in background'`，而不管那些工具调用实际做了什么。这没关系，因为子 agent 的指令已经告诉了它该做什么——它并不需要来自父 agent 派发回合的准确工具结果。但这也意味着子 agent 的对话历史在技术上是不自洽的。选择这个占位符是为了简短和统一，而非准确。

Each of these trade-offs reflects the same priority: when you are paying per-token for API calls at scale, byte-identical prefixes are worth contorting the architecture around.

这些权衡中的每一个都反映了同一个优先级：当你在大规模地为 API 调用按 token 付费时，逐字节相同的前缀值得让整个架构为之扭曲变形。

---

## Apply This: Designing for Prompt Cache Efficiency

## 实践应用：面向 Prompt 缓存效率来做设计

The fork agent pattern generalizes beyond Claude Code. Any system that dispatches multiple parallel LLM calls from the same context can benefit from cache-aware request construction. The principles:

fork agent 这一模式可以推广到 Claude Code 之外。任何从同一上下文派发多个并行 LLM 调用的系统，都能从"缓存感知"的请求构造中获益。原则如下：

**1. Thread rendered prompts, do not recompute.** If your system prompt includes any dynamic content -- feature flags, timestamps, user preferences, A/B test variants -- capture the rendered result and pass it to children by value. Recomputing risks divergence.

**1. 传递已渲染的提示词，不要重新计算。** 如果你的系统提示词包含任何动态内容——feature flag、时间戳、用户偏好、A/B 测试变体——就把渲染后的结果捕获下来，按值传给子 agent。重新计算会带来分叉的风险。

**2. Freeze the tool array.** If your children need different tool sets, you are giving up cache sharing on the tools block. Consider keeping the full tool set and using runtime guards (like the fork boilerplate's "do not use Agent") instead of compile-time removal.

**2. 冻结工具数组。** 如果你的子 agent 需要不同的工具集，那你就放弃了工具块上的缓存共享。可以考虑保留完整的工具集，用运行时防线（就像 fork 样板里的"不要使用 Agent"那样）来替代编译期的移除。

**3. Maximize the shared prefix, minimize the per-child suffix.** Structure your message array so that everything shared comes first and per-child content is appended at the end. Interleaving shared and per-child content fragments the cache boundary.

**3. 最大化共享前缀，最小化各子 agent 的后缀。** 这样组织你的消息数组：所有共享的内容排在前面，各子 agent 各自的内容追加在末尾。把共享内容与各子 agent 内容交错排列，会让缓存边界变得支离破碎。

**4. Use constant placeholders for variable content.** When the message structure requires responses to previous tool calls, use identical placeholder strings across all children rather than actual (divergent) results.

**4. 对可变内容使用常量占位符。** 当消息结构要求对此前的工具调用给出响应时，在所有子 agent 之间使用完全相同的占位字符串，而不是真实的（会分叉的）结果。

**5. Measure the break-even.** Cache sharing has overhead: larger context windows per child (they carry irrelevant history), runtime guards instead of static safety, architectural complexity. Calculate whether your parallelism pattern (how many children, how large the shared prefix) actually saves money after accounting for the extra context tokens.

**5. 测算盈亏平衡点。** 缓存共享是有开销的：每个子 agent 的上下文窗口更大（它们携带着无关的历史）、用运行时防线取代静态安全、架构上的复杂度。算一算，在把额外的上下文 token 计入之后，你的并行模式（多少个子 agent、共享前缀有多大）究竟是不是真的省了钱。

The fork agent system is, at its core, a prompt cache exploitation engine. It answers a question that every multi-agent system builder eventually faces: when the cache gives you a 90% discount on repeated prefixes, how far will you restructure your architecture to claim that discount? Claude Code's answer is: very far.

fork agent 系统，本质上就是一台 prompt 缓存利用引擎。它回答了每一个多 agent 系统构建者最终都会面对的问题：当缓存对重复前缀给你 90% 的折扣时，你愿意为拿到这份折扣而把架构重构到何种程度？Claude Code 的答案是：非常之深。

# Chapter 18: What We Learned

# 第 18 章：我们学到了什么

## Five Architectural Bets

## 五个架构赌注

Claude Code is not the only agentic system. It is not the first. But it made five architectural bets that distinguish it from the landscape of agent frameworks, and after nearly two thousand files and seventeen chapters, those bets deserve examination.

Claude Code 并不是唯一的智能体（agentic）系统，也不是第一个。但它下了五个架构赌注，正是这些赌注让它在众多 agent 框架中显得与众不同。在读完近两千个文件、十七章内容之后，这些赌注值得我们仔细审视。

### Bet 1: The Generator Loop Over Callbacks

### 赌注一：用 generator 循环取代回调

Most agent frameworks give you a pipeline: define tools, register handlers, let the framework orchestrate. The developer writes callbacks. The framework decides when to call them.

大多数 agent 框架给你的是一条流水线：定义工具、注册处理器，然后让框架来编排。开发者编写回调，框架决定何时调用它们。

Claude Code does the opposite. The `query()` function is an async generator -- the developer owns the loop. The model streams a response, the generator yields tool calls, the caller executes them, appends results, and the generator loops. There is one function, one data flow, one place where every interaction passes through. The 10 terminal states and 7 continuation states of the generator's return type encode every possible outcome. The loop is the system.

Claude Code 反其道而行之。`query()` 函数是一个 async generator——循环由开发者掌控。模型流式返回响应，generator 产出（yield）工具调用，调用方执行这些调用、追加结果，然后 generator 继续循环。只有一个函数、一条数据流、一个让所有交互都经过的地方。该 generator 返回类型中的 10 个终止状态和 7 个延续状态编码了所有可能的结果。循环就是整个系统。

The bet was that a single generator function, even one that grew to 1,700 lines, would be more comprehensible than a distributed callback graph. After studying the source, the bet paid off. When you want to understand why a session ended, you look at one function. When you want to add a new terminal state, you add one variant to one discriminated union. The type system enforces exhaustive handling. A callback architecture would scatter this logic across dozens of files, and the interactions between callbacks would be implicit rather than visible in the control flow.

这个赌注的赌的是：单个 generator 函数——哪怕它增长到 1,700 行——也会比一张分散的回调图更易于理解。研读源码之后，可以说这个赌注赢了。当你想搞清楚一次会话为何结束时，你只需查看一个函数。当你想新增一个终止状态时，你只需往一个可辨识联合（discriminated union）里添加一个变体。类型系统会强制你做穷尽处理。而回调式架构会把这套逻辑散落到数十个文件中，回调之间的交互是隐式的，无法在控制流中直观看到。

### Bet 2: File-Based Memory Over Databases

### 赌注二：用基于文件的记忆取代数据库

Chapter 11 made the case in detail, but the architectural significance extends beyond memory. The decision to use plain Markdown files instead of SQLite, a vector database, or a cloud service was a bet on transparency over capability. A database would support richer queries, faster lookups, and transactional guarantees. Files provide none of that. What files provide is trust.

第 11 章已经详细论述过这一点，但其架构意义远不止于记忆系统本身。选择用纯 Markdown 文件而非 SQLite、向量数据库或云服务，是一个押注透明性优先于能力的决定。数据库能支持更丰富的查询、更快的检索和事务保证，而文件这些都不提供。文件提供的是信任。

A user who opens `~/.claude/projects/myapp/memory/MEMORY.md` in vim and sees exactly what the agent remembers about them has a fundamentally different relationship with the system than a user who must ask the agent "what do you remember?" and hope the answer is complete. The file-based design makes the agent's knowledge state externally observable, not just self-reported. This matters more than query performance. The LLM-powered recall system compensates for the storage simplicity with retrieval intelligence -- a Sonnet side-query selecting five relevant memories from a manifest is more precise than embedding similarity and requires zero infrastructure.

一个能在 vim 中打开 `~/.claude/projects/myapp/memory/MEMORY.md` 并直接看到 agent 究竟记住了关于自己的哪些内容的用户，与一个必须去问 agent“你都记得些什么？”并寄望于答案完整的用户，他们和系统之间的关系有着本质区别。基于文件的设计让 agent 的知识状态从外部可观测，而不仅仅是自我报告。这比查询性能更重要。由 LLM 驱动的回忆系统用检索智能弥补了存储的简单——让 Sonnet 通过一次旁路查询（side-query）从清单（manifest）中挑选五条相关记忆，比 embedding 相似度更精确，且不需要任何基础设施。

### Bet 3: Self-Describing Tools Over Central Orchestrators

### 赌注三：用自描述工具取代中心化编排器

Agent frameworks typically provide a tool registry: you describe your tools in a central configuration, and the framework presents them to the model. Claude Code's tools describe themselves. Each `Tool` object carries its own name, description, input schema, prompt contribution, concurrency safety flag, and execution logic. The tool system's job is not to describe tools to the model -- it is to let tools describe themselves.

agent 框架通常会提供一个工具注册表（tool registry）：你在某个中心化配置中描述你的工具，框架再把它们呈现给模型。而 Claude Code 的工具是自我描述的。每个 `Tool` 对象都携带自己的名称、描述、输入 schema、对 prompt 的贡献、并发安全标志和执行逻辑。工具系统的职责不是向模型描述工具，而是让工具自己描述自己。

This bet pays off in extensibility. MCP tools (Chapter 15) become first-class citizens by implementing the same interface. A tool from an MCP server and a built-in tool are indistinguishable to the model. The system does not need a separate "MCP tool adapter" layer -- the wrapping produces a standard `Tool` object, and from that point forward, the existing tool pipeline handles it: permission checking, concurrent execution, result budgeting, hook interception.

这个赌注在可扩展性上得到了回报。MCP 工具（第 15 章）只需实现同一套接口，就能成为一等公民。对模型来说，来自 MCP 服务器的工具与内建工具毫无区别。系统不需要单独的“MCP 工具适配器”层——封装之后产出的就是一个标准 `Tool` 对象，从那一刻起，现有的工具流水线便会接管它：权限检查、并发执行、结果预算（result budgeting）、hook 拦截。

### Bet 4: Fork Agents for Cache Sharing

### 赌注四：用 fork agent 实现缓存共享

Chapter 9 covered the fork mechanism: a sub-agent that starts with the parent's full conversation in its context window, sharing the parent's prompt cache. This is not a convenience optimization -- it is an architectural bet that the cache sharing model is worth the complexity of fork lifecycle management.

第 9 章介绍过 fork 机制：一个子 agent 在启动时，其上下文窗口里已经包含了父级的完整对话，并共享父级的 prompt 缓存。这不是一项图方便的优化——它是一个架构赌注，赌的是这种缓存共享模型值得为 fork 生命周期管理所带来的复杂度买单。

The alternative -- spawning a fresh agent with a summary of the conversation -- is simpler but expensive. Every fresh agent pays the full cost of processing its context from scratch. A forked agent gets the parent's cached prefix for free (a 90% discount on input tokens), making it economical to spawn agents for small tasks: memory extraction, code review, verification passes. The background memory extraction agent (Chapter 11) runs after every query loop turn, and its cost is marginal precisely because it shares the parent's cache. Without fork-based cache sharing, that agent would be prohibitively expensive.

另一种做法——带着一份对话摘要启动一个全新 agent——更简单，但代价高昂。每个全新 agent 都要从零开始处理上下文，付出全额成本。而 fork 出来的 agent 能免费获得父级的缓存前缀（输入 token 享受 90% 折扣），这使得为小任务派生 agent 变得经济实惠：记忆提取、代码审查、验证回合。后台记忆提取 agent（第 11 章）在每个 query 循环回合之后运行，其成本之所以微不足道，正是因为它共享了父级的缓存。如果没有基于 fork 的缓存共享，这个 agent 的开销将高得令人望而却步。

### Bet 5: Hooks Over Plugins

### 赌注五：用 hook 取代插件

Most extensibility systems use plugins -- code that registers capabilities and runs within the host process. Claude Code uses hooks -- external processes that run at lifecycle points and communicate through exit codes and JSON on stdin/stdout.

大多数可扩展性系统使用插件——即注册能力并运行在宿主进程内部的代码。Claude Code 使用的是 hook——在生命周期节点上运行、通过退出码以及 stdin/stdout 上的 JSON 进行通信的外部进程。

The bet is that process isolation is worth the overhead. A plugin can crash the host. A hook crashes its own process. A plugin can leak memory into the host's heap. A hook's memory dies with its process. A plugin requires an API surface that must be versioned and maintained. A hook requires stdin, stdout, and an exit code -- a protocol that has been stable since 1971.

这个赌注赌的是：进程隔离值得为其开销付出代价。插件可能让宿主崩溃，而 hook 只会让自己的进程崩溃。插件可能把内存泄漏到宿主的堆里，而 hook 的内存随其进程一同消亡。插件需要一套必须做版本管理和维护的 API 表面，而 hook 只需要 stdin、stdout 和一个退出码——这套协议自 1971 年以来一直稳定。

The overhead is real: spawning a process per hook invocation costs milliseconds that an in-process callback would not. The -70% fast path for internal callbacks (Chapter 12) shows that the system knows this cost matters. But for external hooks -- user scripts, team linters, enterprise policy servers -- the isolation guarantee makes the system safer to extend. An enterprise can deploy hook-based policy enforcement without worrying that a malformed hook script will crash their developers' sessions.

这种开销是实实在在的：每次 hook 调用都要派生一个进程，会花费进程内回调本不需要付出的数毫秒。针对内部回调的 -70% 快速路径（第 12 章）表明系统清楚这项成本是要紧的。但对于外部 hook——用户脚本、团队 linter、企业策略服务器——隔离保证让系统的扩展更加安全。企业可以部署基于 hook 的策略强制执行，而不必担心某个格式错误的 hook 脚本会让开发者的会话崩溃。

---

## What Transfers, What Does Not

## 哪些可迁移，哪些不可

Not every pattern in Claude Code generalizes. Some are consequences of scale, resources, or specific constraints that other agent builders may not share.

并非 Claude Code 中的每一种模式都具有普适性。有些只是规模、资源或特定约束的产物，而其他 agent 构建者未必面临这些条件。

### Patterns That Transfer to Any Agent

### 可迁移到任何 agent 的模式

**The generator loop pattern.** Any agent that needs to stream responses, handle tool calls, and manage multiple terminal states benefits from making the loop explicit rather than hiding it behind callbacks. The discriminated union return type -- encoding exactly why the loop stopped -- is a pattern that eliminates an entire class of "why did the agent stop?" debugging sessions.

**generator 循环模式。** 任何需要流式返回响应、处理工具调用并管理多种终止状态的 agent，都能从“让循环显式化、而非藏在回调背后”中获益。用可辨识联合作为返回类型——精确编码循环为何停止——这一模式能消除一整类“agent 为什么停了？”式的调试场景。

**File-based memory with LLM recall.** The specific implementation details are Claude Code's, but the principle -- simple storage combined with intelligent retrieval -- applies to any agent that needs to persist knowledge across sessions. The four-type taxonomy (user, feedback, project, reference) and the derivability test ("can this be re-derived from the current project state?") are reusable design heuristics.

**配合 LLM 回忆的基于文件的记忆。** 具体的实现细节是 Claude Code 特有的，但其原则——简单存储结合智能检索——适用于任何需要跨会话持久化知识的 agent。四类分类法（用户、反馈、项目、参考）以及可推导性测试（“这条信息能否从当前项目状态中重新推导出来？”）都是可复用的设计经验法则。

**Asymmetric read/write channels for remote execution.** When reads are high-frequency streams and writes are low-frequency RPCs, separating them is correct regardless of the specific transport protocol.

**面向远程执行的非对称读/写通道。** 当读取是高频流、写入是低频 RPC 时，无论具体采用哪种传输协议，将二者分离都是正确的。

**Bitmap pre-filters for search.** Any agent searching a large file index benefits from a 26-bit letter bitmap as a pre-filter. Four bytes per entry, one integer comparison per candidate -- the cost-to-benefit ratio is remarkable.

**用于搜索的位图预过滤器。** 任何要在大型文件索引中搜索的 agent，都能从用 26 位字母位图作为预过滤器中获益。每条目四字节、每个候选项一次整数比较——其性价比相当惊人。

**Prompt cache stability as an architectural concern.** If your agent uses an API with prompt caching, structuring the prompt with stable content first and volatile content last is not an optimization -- it is an architectural decision that determines your cost structure.

**把 prompt 缓存的稳定性当作一项架构关切。** 如果你的 agent 使用带 prompt 缓存的 API，那么把稳定内容放在前、易变内容放在后地组织 prompt，就不是一项优化——它是一个决定你成本结构的架构决策。

### Patterns Specific to Claude Code's Scale

### 仅适用于 Claude Code 规模的模式

**The forked terminal renderer.** Claude Code forked Ink and reimplemented the rendering pipeline with packed typed arrays, pool-based interning, and cell-level diffing because it needed 60fps streaming in a terminal. Most agents render to a web interface or a simple log output. The engineering investment only makes sense when terminal rendering is your primary UI and you are streaming at high frequency.

**fork 出来的终端渲染器。** Claude Code fork 了 Ink，并用打包的类型化数组（packed typed arrays）、基于对象池的字符串驻留（pool-based interning）以及单元格级别的 diff 重新实现了渲染流水线，因为它需要在终端中实现 60fps 的流式渲染。大多数 agent 渲染到的是 Web 界面或简单的日志输出。只有当终端渲染是你的主要 UI、且你在高频流式输出时，这样的工程投入才有意义。

**The 50+ startup profiling checkpoints.** Meaningful when you have hundreds of thousands of users and 0.5% sampling produces statistically significant data. For a smaller agent, a simpler timing system suffices.

**50 多个启动性能剖析检查点。** 只有当你拥有数十万用户、0.5% 的采样就能产生具有统计显著性的数据时，这才有意义。对于规模较小的 agent，一套更简单的计时系统就足够了。

**Eight MCP transport types.** Claude Code supports stdio, SSE, HTTP, WebSocket, SDK, two IDE variants, and a Claude.ai proxy because it must integrate with every deployment topology. Most agents need stdio and HTTP.

**八种 MCP 传输类型。** Claude Code 支持 stdio、SSE、HTTP、WebSocket、SDK、两种 IDE 变体以及一个 Claude.ai 代理，因为它必须与每一种部署拓扑集成。大多数 agent 只需要 stdio 和 HTTP。

**The hooks snapshot security model.** Freezing hook configuration at startup and never re-reading it implicitly is a defense against a specific threat: malicious repository code modifying hooks after the user accepts the trust dialog. This matters when your agent runs in arbitrary repositories with untrusted `.claude/` configurations. An agent that only runs in trusted environments can use simpler hook management.

**hook 快照安全模型。** 在启动时冻结 hook 配置、之后绝不隐式地重新读取，是针对一种特定威胁的防御：在用户接受信任对话框之后，恶意的仓库代码篡改 hook。当你的 agent 在带有不可信 `.claude/` 配置的任意仓库中运行时，这一点至关重要。而只在可信环境中运行的 agent 可以采用更简单的 hook 管理。

---

## The Cost of Complexity

## 复杂度的代价

Nearly two thousand files. What does that buy, and what does it cost?

将近两千个文件。这换来了什么，又付出了什么？

The file count is misleading as a complexity metric. Much of it is test infrastructure, type definitions, configuration schemas, and the forked Ink renderer. The actual behavioral complexity concentrates in a small number of high-density files: `query.ts` (1,700 lines, the agent loop), `hooks.ts` (4,900 lines, the lifecycle interception system), `REPL.tsx` (5,000 lines, the interactive orchestrator), and the memory system's prompt building functions.

把文件数量当作复杂度指标是有误导性的。其中很大一部分是测试基础设施、类型定义、配置 schema 以及 fork 出来的 Ink 渲染器。真正的行为复杂度集中在少数几个高密度文件中：`query.ts`（1,700 行，agent 循环）、`hooks.ts`（4,900 行，生命周期拦截系统）、`REPL.tsx`（5,000 行，交互式编排器），以及记忆系统的 prompt 构建函数。

The complexity comes from three sources, each with a different character:

复杂度来自三个来源，每一个都有不同的性质：

**Protocol diversity.** Supporting five terminal keyboard protocols, eight MCP transport types, four remote execution topologies, and seven configuration scopes is inherently complex. Each additional protocol is a linear addition to the codebase, not an exponential one -- but the sum is large. This complexity is accidental in the Brooksian sense: it comes from the environment (terminal fragmentation, MCP transport evolution, remote deployment topologies), not from the problem being solved.

**协议多样性。** 支持五种终端键盘协议、八种 MCP 传输类型、四种远程执行拓扑以及七种配置作用域，本质上就是复杂的。每多支持一种协议，对代码库都是线性增加，而非指数增加——但其总和依然庞大。从布鲁克斯（Brooks）意义上说，这种复杂度是偶发的（accidental）：它来自环境（终端的碎片化、MCP 传输的演进、远程部署拓扑），而非来自所要解决的问题本身。

**Performance optimization.** The pool-based rendering, bitmap search pre-filters, sticky cache latches, and speculative tool execution each add complexity in exchange for measurable performance gains. This complexity is justified by measurement -- every optimization was preceded by profiling data that identified the bottleneck. The risk is that optimizations accumulate and interact in ways that make the hot paths harder to modify.

**性能优化。** 基于对象池的渲染、位图搜索预过滤器、粘性缓存锁存（sticky cache latches）以及推测式工具执行（speculative tool execution），每一项都以增加复杂度为代价换取可度量的性能收益。这种复杂度是由测量来证明其合理性的——每一项优化之前都有指明瓶颈所在的性能剖析数据。风险在于：这些优化会不断累积、相互交织，使热路径（hot paths）越来越难以修改。

**Behavioral tuning.** The memory system's prompt instructions, the staleness warnings, the verification protocol, the "ignore memory" anti-pattern instruction -- these are not code complexity. They are prompt complexity, and they carry a different maintenance burden. When the model's behavior changes between versions, prompt instructions that were carefully tuned through evals may need re-tuning. The eval infrastructure (referenced throughout the codebase as case numbers and eval scores) is the defense against regression, but it requires ongoing investment.

**行为调优。** 记忆系统的 prompt 指令、过期（staleness）警告、验证协议、“忽略记忆”这条反模式指令——这些都不是代码复杂度。它们是 prompt 复杂度，背负着另一种维护负担。当模型在不同版本之间行为发生变化时，那些经过 eval 精心调优过的 prompt 指令可能需要重新调优。eval 基础设施（在整个代码库中以 case 编号和 eval 分数的形式被引用）是防止回归的防线，但它需要持续投入。

The maintenance burden of this system is significant. A new engineer reading the codebase must understand not just the code paths but the eval outcomes that motivated specific prompt phrasings, the production incidents that motivated specific security checks, and the performance profiles that motivated specific optimizations. The code comments are thorough -- many include eval case numbers and before/after measurements -- but thorough comments in nearly two thousand files are themselves a reading burden.

这套系统的维护负担相当可观。一名阅读代码库的新工程师，不仅要理解代码路径，还要理解那些促成特定 prompt 措辞的 eval 结果、那些促成特定安全检查的生产事故，以及那些促成特定优化的性能剖析数据。代码注释很详尽——许多注释都包含 eval case 编号和前后对比的度量数据——但分布在近两千个文件里的详尽注释，本身也是一种阅读负担。

---

## Where Agentic Systems Are Heading

## 智能体系统的走向

Four trends are visible from the patterns in Claude Code, and they point toward where the field is going.

从 Claude Code 的诸多模式中可以看出四个趋势，它们指向了这一领域的未来走向。

### MCP as the Universal Protocol

### MCP 成为通用协议

Chapter 15 described Claude Code as one of the most complete MCP clients. The significance is not Claude Code's implementation -- it is that MCP exists at all. A standardized protocol for tool discovery and invocation means that tools built for one agent work with any agent. The ecosystem effects are obvious: an MCP server for Postgres, once built, serves every agent that speaks MCP. The developer's investment in tool integration is portable.

第 15 章把 Claude Code 描述为最完整的 MCP 客户端之一。其意义不在于 Claude Code 的实现，而在于 MCP 的存在本身。一套用于工具发现与调用的标准化协议，意味着为某一个 agent 构建的工具可以与任何 agent 协同工作。生态效应显而易见：一个为 Postgres 构建的 MCP 服务器，一旦建成，就能服务于每一个说 MCP 的 agent。开发者在工具集成上的投入是可移植的。

The implication for agent builders: if you are defining a custom tool protocol, you are probably making a mistake. MCP is good enough, it is getting better, and the ecosystem advantages of a standard protocol compound over time. Build an MCP client, contribute to the spec, and let the protocol evolve through community feedback.

对 agent 构建者的启示是：如果你正在定义一套自定义的工具协议，那你很可能犯了错误。MCP 已经足够好，并且还在变得更好，而标准协议带来的生态优势会随着时间复利式增长。构建一个 MCP 客户端，为规范做贡献，让协议在社区反馈中演进。

### Multi-Agent Coordination

### 多 agent 协同

Claude Code's sub-agent system (Chapter 8), task coordination (Chapter 10), and fork mechanism (Chapter 9) are early implementations of multi-agent patterns. They solve specific problems -- cache sharing, parallel exploration, structured verification -- but they also reveal the fundamental challenge: coordination overhead.

Claude Code 的子 agent 系统（第 8 章）、任务协调（第 10 章）和 fork 机制（第 9 章）都是多 agent 模式的早期实现。它们解决了一些具体问题——缓存共享、并行探索、结构化验证——但同时也暴露出根本性的挑战：协调开销。

Every message between agents consumes tokens. Every fork shares a cache but adds a conversation branch that the parent must eventually reconcile. The Task system's state machine (queued, running, completed, failed, cancelled) is coordination machinery that adds complexity without adding capability. As agents become more capable, the pressure will shift from "how do we coordinate multiple agents?" to "how do we make one agent capable enough that coordination is unnecessary?"

agent 之间的每一条消息都要消耗 token。每一次 fork 都共享缓存，但同时也新增了一条父级最终必须调和的对话分支。Task 系统的状态机（queued、running、completed、failed、cancelled）是一套增加复杂度却不增加能力的协调机制。随着 agent 能力越来越强，压力将从“我们如何协调多个 agent？”转向“我们如何让单个 agent 强大到根本不需要协调？”

The current evidence suggests both approaches will coexist. Simple tasks will use single agents. Complex tasks will use coordinated multi-agent systems. The engineering challenge is making the coordination overhead low enough that the crossover point favors multi-agent for genuinely parallel work, not just for tasks that are complex.

目前的证据表明两种方式将会并存。简单任务会使用单个 agent，复杂任务则会使用协调式的多 agent 系统。工程上的挑战在于：把协调开销压得足够低，使得那个交叉点（crossover point）只在面对真正可并行的工作时才偏向多 agent，而不是只要任务复杂就偏向多 agent。

### Persistent Memory

### 持久化记忆

Claude Code's memory system is version 1 of persistent agent memory. The file-based design, the four-type taxonomy, the LLM-powered recall, the staleness system, and the KAIROS mode for long-running sessions are all first-generation solutions to a problem that will evolve significantly.

Claude Code 的记忆系统是 agent 持久化记忆的 1.0 版本。基于文件的设计、四类分类法、由 LLM 驱动的回忆、过期系统，以及面向长期运行会话的 KAIROS 模式，都是针对一个将会大幅演进的问题的第一代解决方案。

Future memory systems will likely add structured retrieval (the current system retrieves whole files; future systems might retrieve specific facts), cross-project transfer learning (user preferences that apply everywhere, project conventions that do not), and collaborative memory (Chapter 11's team memory is a first step, but the sync, conflict resolution, and access control are minimal).

未来的记忆系统很可能会增加结构化检索（当前系统检索的是整个文件，未来的系统可能会检索具体的事实）、跨项目的迁移学习（处处适用的用户偏好，以及并不通用的项目约定），以及协作式记忆（第 11 章的团队记忆只是第一步，但其同步、冲突解决和访问控制都还很初级）。

The open question is whether the file-based approach scales. At 200 memories per project, it works. At 2,000 memories per project, the Sonnet side-query's manifest becomes too large, the consolidation becomes too expensive, and the index exceeds its caps. The architectural bet on files-over-databases will face its hardest test as usage grows.

悬而未决的问题是：基于文件的方法能否随规模扩展。在每个项目 200 条记忆时，它运转良好。在每个项目 2,000 条记忆时，Sonnet 旁路查询的清单会变得太大，整合（consolidation）会变得太昂贵，索引也会超出其上限。随着使用量增长，押注“文件优于数据库”的这一架构赌注将面临它最严峻的考验。

### Autonomous Operation

### 自主运行

The KAIROS mode, the background memory extraction agent, the auto-dream consolidation, the speculative tool execution -- these are all steps toward autonomous operation. The agent does useful work without being asked: it remembers what you forgot to tell it to remember, it consolidates its own knowledge while you sleep, it starts executing the next tool before the current response is complete.

KAIROS 模式、后台记忆提取 agent、auto-dream 整合、推测式工具执行——这些都是迈向自主运行的步伐。agent 在没有被要求的情况下做有用的工作：它记住了你忘记告诉它去记住的东西，它在你睡觉时整合自己的知识，它在当前响应尚未完成时就开始执行下一个工具。

The trajectory is clear. Future agents will be less reactive and more proactive. They will notice patterns the user has not described, suggest corrections the user has not requested, and maintain their own knowledge without explicit `/remember` commands. Claude Code's memory system, with its background extraction safety net and its prompt-engineered "what to save" heuristics, is the prototype for this future.

走向是清晰的。未来的 agent 会更少被动反应、更多主动作为。它们会注意到用户没有描述过的模式，提出用户没有请求过的修正，并在没有显式 `/remember` 命令的情况下维护自己的知识。Claude Code 的记忆系统，凭借其后台提取的安全网和经过 prompt 工程打磨的“该保存什么”的启发式规则，正是这一未来的原型。

The constraint is trust. Autonomous operation requires the user to trust that the agent will do the right thing when unattended. The file-based memory, the observable hook system, the staleness warnings, the permission dialogs -- all of these exist because trust must be earned, not assumed. The path to more autonomous agents runs through more transparent agents.

约束在于信任。自主运行要求用户相信 agent 在无人看管时会做正确的事。基于文件的记忆、可观测的 hook 系统、过期警告、权限对话框——所有这些之所以存在，是因为信任必须靠赢得，而非假定。通往更自主 agent 的道路，要穿过更透明的 agent。

---

## Closing

## 结语

Seventeen chapters. Six core abstractions. A generator loop at the center, tools extending outward, memory reaching backward through time, hooks guarding the perimeter, a rendering engine translating it all into characters on a screen, and MCP connecting it to the world beyond the codebase.

十七章。六个核心抽象。一个 generator 循环居于中心，工具向外延展，记忆穿越时间向后回溯，hook 守护着边界，一个渲染引擎将这一切翻译成屏幕上的字符，而 MCP 则把它连接到代码库之外的世界。

The deepest pattern in Claude Code is not any single technique. It is the recurring decision to push complexity to the boundaries. The rendering system pushes complexity to the pools and the diff -- inside the pipeline, everything is integer comparisons. The input system pushes complexity to the tokenizer and the keybinding resolver -- inside the handlers, everything is typed actions. The memory system pushes complexity to the write protocol and the recall selector -- inside the conversation, everything is context. The agent loop pushes complexity to the terminal states and the tool system -- inside the loop, it is just: stream, collect, execute, append, repeat.

Claude Code 中最深层的模式不是任何单一的技术。而是那个反复出现的决策：把复杂度推向边界。渲染系统把复杂度推给了对象池和 diff——在流水线内部，一切都是整数比较。输入系统把复杂度推给了 tokenizer 和键位解析器——在处理器内部，一切都是带类型的动作。记忆系统把复杂度推给了写入协议和回忆选择器——在对话内部，一切都是上下文。agent 循环把复杂度推给了终止状态和工具系统——在循环内部，它仅仅是：流式接收、收集、执行、追加、重复。

Each boundary absorbs chaos and exports order. Raw bytes become `ParsedKey`. Markdown files become recalled memories. MCP JSON-RPC becomes `Tool` objects. Hook exit codes become permission decisions. On one side of each boundary, the world is messy -- five keyboard protocols, fragile OAuth servers, stale memories, untrusted repository hooks. On the other side, the world is typed, bounded, and exhaustively handled.

每一道边界都吸收混沌、输出秩序。原始字节变成 `ParsedKey`。Markdown 文件变成被回忆起的记忆。MCP JSON-RPC 变成 `Tool` 对象。hook 退出码变成权限决策。在每道边界的一侧，世界是混乱的——五种键盘协议、脆弱的 OAuth 服务器、过期的记忆、不可信的仓库 hook。在另一侧，世界则是带类型的、有界的、被穷尽处理过的。

If you are building an agentic system, this is the transferable lesson. Not the specific techniques -- you may not need pool-based rendering or KAIROS mode or eight MCP transports. But the principle: define your boundaries, absorb complexity there, and keep everything between them clean. The boundaries are where the engineering is hard. The interior is where the engineering is pleasant. Design for pleasant interiors, and invest your complexity budget at the edges.

如果你正在构建一个智能体系统，这就是那条可迁移的经验。不是那些具体的技术——你也许并不需要基于对象池的渲染、KAIROS 模式或者八种 MCP 传输。而是这条原则：定义你的边界，在那里吸收复杂度，并让边界之间的一切保持整洁。边界是工程艰难之处，内部是工程愉悦之处。为愉悦的内部而设计，把你的复杂度预算投在边缘。

The source code is open. The crab has the map in its claw. Go read it.

源码是开放的。螃蟹的钳子里夹着地图。去读它吧。

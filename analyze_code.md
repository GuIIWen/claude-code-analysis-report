# Claude Code Source 全量工程分析报告

生成时间：2026-04-02  
最近同步时间：2026-04-02 19:58:11 +0800  
分析范围：`/root/claude-code-source-code`  
分析方式：主线程通读关键源码 + 2 个 `gpt-5.4` 子分析线程交叉审阅  
约束说明：本轮只做静态分析与文档沉淀，不修改业务代码，不做 debug/修复

## 1. 执行摘要

这不是一个“普通 CLI 工具源码仓”，而是一个把 CLI/TUI、headless SDK、Prompt 编排、Tool 执行、权限治理、沙箱、MCP 生态、Subagent/Team、上下文压缩、远程控制、遥测与远程策略全部装进同一运行时内核的 agent 平台。

从工程结构看，真正的系统控制面主要有六条：

1. `feature()` / `MACRO.*` / Bun 编译期裁剪。
2. `main.tsx` + `entrypoints/*` 的启动分流与模式装配。
3. `query.ts` + `QueryEngine.ts` 的统一会话执行内核。
4. `prompts.ts` + `context.ts` + `utils/api.ts` 的 prompt/cache 前缀构建。
5. `Tool.ts` + `tools.ts` + `services/tools/*` 的工具协议与调度。
6. `permissions/*` + `sandbox/*` + `services/mcp/*` 的安全边界与外部能力接入。

这套工程最强的地方不是“功能多”，而是已经形成了比较明确的平台级设计意图：

- Prompt cache 被当成一级性能指标来设计，不是 API 附带优化。
- Subagent 不是 prompt 模板，而是二级 runtime。
- MCP 不是插件边角，而是一等外部能力接入层。
- 权限不是单层 allow/deny，而是规则、classifier、hook、沙箱、运行模式叠加的决策链。
- headless/SDK 不是简化版，而是和 REPL 并列的一等运行面。

同时，它的主要风险也很明确：

- `src/main.tsx`、`src/screens/REPL.tsx`、`src/query.ts`、`src/tools/AgentTool/*`、`src/services/mcp/*` 已经出现超级枢纽化。
- 许多“行为正确性”依赖跨文件的隐式契约，尤其是 prompt cache、fork subagent、权限路径语义、MCP 去重、session restore。
- 这是一个“用于研究的反编译/解包源码视图”，不是官方原始研发仓；构建与运行行为不能无条件等同于 Anthropic 内部真实构建链。

## 2. 仓库画像与可信边界

### 2.1 基本画像

- 包名：`@anthropic-ai/claude-code-source`
- 版本：`2.1.88`
- `src` 文件数：约 `1902`
- 主要子目录文件数：
  - `utils`: 564
  - `components`: 389
  - `commands`: 207
  - `tools`: 184
  - `services`: 130
- 关键超大文件：
  - `src/cli/print.ts`: 5594 LOC
  - `src/main.tsx`: 4683 LOC
  - `src/services/mcp/auth.ts`: 2465 LOC
  - `src/query.ts`: 1729 LOC
  - `src/QueryEngine.ts`: 1295 LOC
  - `src/constants/prompts.ts`: 914 LOC
  - `src/Tool.ts`: 792 LOC

### 2.2 这是“研究用源码视图”，不是官方原始仓

`package.json` 直接把自己定义为研究用途的反编译源码：

```json
{
  "name": "@anthropic-ai/claude-code-source",
  "version": "2.1.88",
  "description": "Claude Code v2.1.88 — decompiled source for research"
}
```

文件：`package.json`

`README.md` 与 `QUICKSTART.md` 也明确说明：

- npm 发布物原本是单一 bunded `cli.js`
- 当前 `src/` 是“从 npm 包中提取/解包的 TypeScript 源码”
- 完整重建依赖 Bun 的编译期 intrinsic，如 `feature()`、`MACRO.*`、`bun:bundle`

### 2.3 构建链不是等价重建

`scripts/build.mjs` 说得非常直白：这是 best-effort build，不是官方原生构建。

```js
/**
 * ⚠️ IMPORTANT: A complete rebuild requires the Bun runtime's compile-time
 * intrinsics (feature(), MACRO, bun:bundle). This script provides a
 * best-effort build using esbuild.
 */
```

更关键的是，它会直接把 `feature('X')` 替换为 `false`，并自动补 stub：

```js
// 2a. feature('X') → false
if (/\bfeature\s*\(\s*['"][A-Z_]+['"]\s*\)/.test(src)) {
  src = src.replace(/\bfeature\s*\(\s*['"][A-Z_]+['"]\s*\)/g, 'false')
  changed = true
}
```

文件：`scripts/build.mjs`

这意味着：

- 外部分析仓中看到的功能树，和 Anthropic 内部真实 build 里保留下来的功能树不一定一致。
- 很多 `feature()` 分支对应的模块根本不在发布包里，而是发布时已经 DCE 掉了。
- 不能把“这个仓库里缺少某个实现”直接理解为产品没有这个能力。

### 2.4 `prepare-src.mjs` 直接 patch `src/`

这是一个很容易被忽略的点。

`scripts/prepare-src.mjs` 不是在副本上变换，而是直接遍历并 patch `src/`：

```js
const ROOT = path.resolve(__dirname, '..')
const SRC = path.join(ROOT, 'src')
...
for (const file of files) {
  if (patchFile(file)) {
    patched++
  }
}
```

文件：`scripts/prepare-src.mjs`

这至少带来两个结论：

- 如果执行过 `prepare-src.mjs`，当前仓库中的 `src/` 就可能偏离“最初解包出的原始快照”。
- 本节已经可以确认的，不是“它现在一定改过”，而是“这个仓库具备直接原地改写 `src/` 的机制”；因此它同时承载“分析视图”和“重建试验视图”，边界并不绝对干净。

## 3. 总体架构判断

### 3.1 真正的拓扑不是“CLI -> API”

更准确的描述应该是：

`入口分流 -> 初始化/策略装配 -> prompt/cache 前缀构建 -> query 内核 -> tool/permission/sandbox -> MCP/agent/skills -> transcript/memory/compaction -> 输出面(REPL/headless/bridge)`

这里不是作者主观喜欢把一条链写长，而是源码里这些节点确实各自形成了一层“一级边界”：

- `prompt/cache 前缀构建` 要单列，因为它在进入 `query()` 之前就已经影响 system blocks、tool schema 和 cache-safe 字节前缀。
- `query 内核` 要单列，因为上下文治理、流式采样、tool loop、follow-up query 都在这里统一发生。
- `tool/permission/sandbox` 要单列，因为模型一旦发出 `tool_use`，执行路径立刻进入另一套策略与安全栈，已经不是“继续跑 prompt”。
- `输出面` 要单列，因为 REPL、headless、bridge 共享 query 内核，但宿主协议、UI 能力、恢复/交互方式并不相同。

所以更短地说：这套系统当然可以被压扁成“CLI 调 API”，但那样会把真正决定 cache、权限、工具和运行面差异的结构全部压没。

### 3.2 可以把系统分成 8 个一级子系统

这里的 8 层更适合作为“分析框架”来理解，而不是把它误读成仓库内部明示的一张官方架构图。源码里的边界并不完全干净，尤其 `Query / Tool / Permission / Agent` 之间存在明显交叠。

1. 启动与运行面分流
2. Prompt / Context / API 前缀构建
3. Query 执行内核
4. Tool 协议与工具调度
5. Agent / Skill / Fork / Team
6. Permission / Classifier / Sandbox
7. MCP 配置、连接、鉴权与模型暴露面
8. Persistence / Session / Memory / Telemetry / Remote policy

### 3.3 几个“文件名低估职责”的例子

- `src/constants/prompts.ts` 不只是常量文件；它同时承载 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`、prompt section registry 和 system prompt 主体组装。
- `src/screens/REPL.tsx` 不只是 UI 文件；它同时承载 permission 交互、session restore、MCP/tool 面增量管理等交互式 runtime 外壳职责。
- `src/QueryEngine.ts` 不只是“另一个引擎”；它主要是 headless 会话宿主，用来把程序化调用形态接到共享 query 内核上。
- `src/tools.ts` 不只是工具列表；它负责 built-in 工具全集、session 裁剪、MCP 合并、稳定排序和最终模型可见工具池装配。
- `src/utils/config.ts` 不只是 config parser；它同时负责多层设置读写、来源追踪、优先级合并和一部分全局状态持久化。

## 4. 启动链路与运行面架构

## 4.1 `src/entrypoints/cli.tsx`：轻量外层分发器

`cli.tsx` 的职责很克制：它优先处理极少数 fast path，尽量减少冷启动模块加载。

典型例子是 `--version`：

```ts
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v' || args[0] === '-V')) {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;
}
```

文件：`src/entrypoints/cli.tsx`

但这个文件并不只是轻量路由，它还承担了一个非常关键的编译期职责：有些环境变量必须在 import 前决定，因为下游模块会在模块初始化时捕获它们。

```ts
// Harness-science L0 ablation baseline. Inlined here (not init.ts) because
// BashTool/AgentTool/PowerShellTool capture DISABLE_BACKGROUND_TASKS into
// module-level consts at import time — init() runs too late.
```

文件：`src/entrypoints/cli.tsx`

这说明它不是普通的 `main()`，而是一个“带编译期/模块初始化约束的启动入口”。

### 4.2 `src/entrypoints/init.ts`：安全初始化层

`init.ts` 的作用不是起业务，而是先把“可以安全做、必须尽早做”的事情完成：

- `enableConfigs()`
- `applySafeConfigEnvironmentVariables()`
- CA cert / mTLS / proxy
- remote managed settings / policy promise 初始化
- graceful shutdown
- OAuth account info 预热
- JetBrains / Git 仓库检测
- upstream proxy（CCR 场景）
- scratchpad 目录初始化

关键片段：

```ts
applySafeConfigEnvironmentVariables()
applyExtraCACertsFromConfig()
setupGracefulShutdown()
...
if (isEligibleForRemoteManagedSettings()) {
  initializeRemoteManagedSettingsLoadingPromise()
}
if (isPolicyLimitsEligible()) {
  initializePolicyLimitsLoadingPromise()
}
```

文件：`src/entrypoints/init.ts`

这里体现的是“信任建立前后的分层初始化”。

### 4.3 `src/main.tsx`：真正的系统编排根

`main.tsx` 才是整套系统的实控中心。它要做的事包括：

- argv 改写与模式识别
- interactive / non-interactive 分流
- commander 命令树构建
- `preAction` 统一初始化
- migration
- inline plugin 目录绑定
- remote managed settings / policy limits 热加载
- 默认 action 对 REPL 或 headless 的跳转

`preAction` 是这个文件里最关键的隐藏节点：

```ts
program.hook('preAction', async thisCommand => {
  await Promise.all([ensureMdmSettingsLoaded(), ensureKeychainPrefetchCompleted()]);
  await init();
  ...
  const { initSinks } = await import('./utils/sinks.js');
  initSinks();
  ...
  runMigrations();
  void loadRemoteManagedSettings();
  void loadPolicyLimits();
})
```

文件：`src/main.tsx`

这一层的设计意图很明确：避免子命令绕开关键初始化。

### 4.4 `--settings` 的 content-hash 路径是 cache 设计的一部分

这是一个特别典型的“跨模块隐式契约”。

```ts
// Use a content-hash-based path instead of random UUID to avoid
// busting the Anthropic API prompt cache.
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
});
```

文件：`src/main.tsx`

原因不是“文件路径美观”，而是：

- settings 路径会进入 Bash sandbox deny list
- deny list 又会进入 tool description
- tool description 会被送进 API prompt 前缀
- 如果路径每次都随机，prompt cache prefix 就会被持续打爆

这里的 `content hash` 不是维护一张映射表，而是做了一个确定性函数映射。`src/utils/tempfile.ts` 的实现是：

```ts
const id = options?.contentHash
  ? createHash('sha256')
      .update(options.contentHash)
      .digest('hex')
      .slice(0, 16)
  : randomUUID()
return join(tmpdir(), `${prefix}-${id}${extension}`)
```

文件：`src/utils/tempfile.ts`

这意味着：

- 输入是 `trimmedSettings` 这段 settings 原文
- 系统会对它做 `SHA-256`
- 取十六进制结果前 `16` 位作为稳定标识
- 最终路径形态类似：`/tmp/claude-settings-<16位hash>.json`

相同内容永远映射到相同路径，不同内容才会映射到不同路径。这个路径不是为了“可读性”，而是为了“模型可见字节稳定性”。

这条链路在源码里是闭环的：

1. `main.tsx` 把 `--settings` 的 JSON 内容写到 content-hash 命名的临时文件。
2. sandbox 配置会把相关路径带进 Bash 的文件系统限制描述。
3. `src/tools/BashTool/prompt.ts` 会把 `denyWithinAllow` 等限制序列化进 BashTool 的 prompt 文本。
4. `src/utils/api.ts` 在生成 API tool schema 时，使用的是 `await tool.prompt(...)` 的结果，而不是短说明。
5. 于是 BashTool prompt 的字节变化，会直接影响送给模型的 tool description，进而影响 API prompt cache 前缀。

其中第 3、4 步的关键代码分别在：

```ts
const filesystemConfig = {
  read: {
    denyOnly: dedup(fsReadConfig.denyOnly),
    ...(fsReadConfig.allowWithinDeny && {
      allowWithinDeny: dedup(fsReadConfig.allowWithinDeny),
    }),
  },
  write: {
    allowOnly: normalizeAllowOnly(fsWriteConfig.allowOnly),
    denyWithinAllow: dedup(fsWriteConfig.denyWithinAllow),
  },
}
```

文件：`src/tools/BashTool/prompt.ts`

```ts
base = {
  name: tool.name,
  description: await tool.prompt({
    getToolPermissionContext: options.getToolPermissionContext,
    tools: options.tools,
    agents: options.agents,
    allowedAgentTypes: options.allowedAgentTypes,
  }),
  input_schema,
}
```

文件：`src/utils/api.ts`

为什么说这件事最体现“工程味道”：

- 它优化的不是本地代码写法，而是最终 API 成本和 cache 命中率这个系统级指标。
- 它定位的是“真正进入模型前缀的那一层字节”，不是停留在 CLI 参数处理表面。
- 它考虑的是跨进程稳定性。注释已经写明，SDK 的每次 `query()` 可能都会起新进程，所以随机 UUID 会让完全相同的 settings 在每次调用里都变成不同 prompt 前缀。
- 它没有为了性能破坏现有安全/功能路径，而是在保留“真实临时文件 + 现有 sandbox 机制”的前提下，把随机名改成稳定名，属于低侵入但高收益的修正。

记忆锚点可以压缩成一句话：

> 这里优化的不是“临时文件名”，而是“模型看到的工具描述字节稳定性”；真正被保护的是 prompt cache，而不是文件系统美观。

## 5. 多运行面：CLI、REPL、Headless、Bridge

### 5.1 这套系统至少有 4 类运行面

1. CLI fast path
2. Interactive REPL / Ink TUI
3. Headless / SDK / structured IO
4. Bridge / Remote Control / background / daemon 类运行面

### 5.2 Interactive REPL 不是“聊天界面”

`src/screens/REPL.tsx` 虽然名字叫 screen，但实际承担了大量 runtime orchestration：

- AppState 与消息流
- 权限弹窗
- MCP 连接与增量工具面
- slash command
- IDE 集成
- background tasks
- session restore
- notifications
- input/output flow

因此它本质上更像“交互 runtime shell”，不是纯展示组件。

### 5.3 Headless 不是降级模式

`src/QueryEngine.ts` 不是另一套模型引擎，而是 headless host。

它负责：

- 会话级 mutable messages
- permission denial 跟踪
- systemPrompt/userContext/systemContext 构建
- structured output tool 跟踪
- transcript persistence
- replay 与 SDK 输出标准化

核心关系是：

- REPL 路径与 headless 路径共享底层 `query()`
- 区别主要在宿主层状态管理、交互能力与输出协议

## 6. 端到端请求生命周期

从“用户输入一句话”到“系统返回结果”，主链路可以概括为：

1. 用户从 REPL 或 headless 入口提交 prompt。
2. 入口层装配 `ToolUseContext`、tools、commands、MCP state、settings。
3. `fetchSystemPromptParts()` 取 `defaultSystemPrompt + userContext + systemContext`。
4. `buildEffectiveSystemPrompt()` 决定最终使用默认 prompt、custom prompt、agent prompt 还是 coordinator prompt。
5. 进入 `query()` 主循环。
6. query 对历史消息做 `tool result budget -> snip -> microcompact -> context collapse -> autocompact`。
7. 调用模型流式采样。
8. 流式收集 assistant message 和 `tool_use` blocks。
9. 通过 `StreamingToolExecutor` 或 `toolOrchestration.runTools()` 执行工具。
10. 工具执行通常先进入统一执行壳；其中输入校验、pre-tool hooks、permission resolution 是主干，classifier / sandbox 则按工具类型和 mode 命中子路径。
11. 工具结果被归并回消息链，必要时继续 follow-up query。
12. transcript、session memory、compact state、usage、notifications 分别落地。
13. 按 REPL/headless/bridge 各自协议输出。

这条链路的关键是：它不是“请求一次模型，跑一下工具”。它是一个持续治理上下文、权限、缓存、持久化和执行反馈的 agent loop。

## 7. Prompt / Context / Cache 架构

## 7.1 `prompts.ts` 的真实定位

`src/constants/prompts.ts` 最大的误导是名字。它不是 prompt 常量表，而是 prompt 编排器。

最关键常量：

```ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

文件：`src/constants/prompts.ts`

它后面跟着明确警告：

- 不要移动或删除这个 marker
- `src/utils/api.ts`
- `src/services/api/claude.ts`

都依赖它来切分缓存边界

### 7.2 `getSystemPrompt()` 的结构是“静态 prefix + 动态 section registry”

主 prompt 的构造模式：

```ts
return [
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  ...,
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

文件：`src/constants/prompts.ts`

这里的架构意义非常强：

- 静态部分用于跨用户/跨会话缓存
- 动态部分包含 session-specific 内容，避免污染全局 cache prefix

### 7.3 `systemPrompt` 不是唯一来源，`userContext`/`systemContext` 也是 cache prefix 一部分

`src/utils/queryContext.ts` 写得非常明确：

```ts
/**
 * Shared helpers for building the API cache-key prefix
 * (systemPrompt, userContext, systemContext) for query() calls.
 */
```

文件：`src/utils/queryContext.ts`

这意味着：分析 prompt 不能只看 `prompts.ts`，还必须看：

- `src/context.ts`
- `src/utils/queryContext.ts`
- `src/utils/systemPrompt.ts`

### 7.4 `getSystemContext()` 的 git status 是会话快照，不是实时观察

`context.ts` 明写：

```ts
return [
  `This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.`,
  ...
].join('\n\n')
```

文件：`src/context.ts`

这点非常关键。很多人会误以为 Claude 始终看到最新 git 状态，但这里明确只是会话启动时的快照。

### 7.5 effective system prompt 还有多级优先级

`src/utils/systemPrompt.ts` 定义了真正的 prompt 选择逻辑：

源码注释里的优先级可以直接读成：

1. `overrideSystemPrompt`：最高优先级，直接替换全部其他 prompt。
2. `Coordinator system prompt`：协调器模式开启且主线程没有 agent 时生效。
3. `Agent system prompt`：如果主线程绑定了 agent，则优先使用 agent prompt。
4. `customSystemPrompt`：用户显式传入的自定义 prompt。
5. `defaultSystemPrompt`：默认 Claude Code system prompt。

这里真正重要的不是“有 5 档优先级”，而是它说明 effective system prompt 并不是简单在 `prompts.ts` 里一次拼完，而是会在运行时根据 coordinator / agent / custom / override 等状态再做一次选择。

而且 proactive 模式下 agent prompt 是 append，不是 replace：

```ts
if (agentSystemPrompt && ... isProactiveActive_SAFE_TO_CALL_ANYWHERE()) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

文件：`src/utils/systemPrompt.ts`

记忆锚点：

> `prompts.ts` 负责“生产默认 prompt”，`systemPrompt.ts` 负责“决定这次最终到底用哪份 prompt”。

### 7.6 API 层会再次把 system prompt 转成 cache-control block

`src/utils/api.ts` 的 `splitSysPromptPrefix()` 再次把 prompt 拆成不同 cache scope：

```ts
if (staticJoined)
  result.push({ text: staticJoined, cacheScope: 'global' })
if (dynamicJoined)
  result.push({ text: dynamicJoined, cacheScope: null })
```

文件：`src/utils/api.ts`

`src/services/api/claude.ts` 则最终转成 API block：

```ts
return splitSysPromptPrefix(systemPrompt, {
  skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
}).map(block => {
  return {
    type: 'text' as const,
    text: block.text,
    ...(enablePromptCaching &&
      block.cacheScope !== null && {
        cache_control: getCacheControl({
          scope: block.cacheScope,
          querySource: options?.querySource,
        }),
      }),
  }
})
```

文件：`src/services/api/claude.ts`

这进一步证明：Prompt 架构和 API cache 架构是一套东西，不是两个独立子系统。

### 7.7 组合后的 prompt 模板

把 `prompts.ts`、`systemPrompt.ts`、`queryContext.ts`、`context.ts` 和 API 层拼在一起，可以把一次请求实际经历的 prompt 组合过程理解成下面这个模板：

```text
[第 1 层：默认 prompt 生产层]
defaultSystemPrompt =
  静态前缀：
    Intro
    System
    Doing tasks
    Actions
    Using your tools
    Tone and style
    Output efficiency
  + （可选）SYSTEM_PROMPT_DYNAMIC_BOUNDARY
  + 动态后缀：
    session_guidance
    memory
    ant_model_override
    env_info
    language
    output_style
    mcp_instructions
    scratchpad
    frc
    summarize_tool_results
    ...

[第 2 层：effective prompt 选择层]
effectiveSystemPrompt =
  overrideSystemPrompt
  or coordinatorSystemPrompt
  or agentSystemPrompt
  or customSystemPrompt
  or defaultSystemPrompt
  + appendSystemPrompt

[第 3 层：cache-key prefix 组合层]
queryPrefix =
  effectiveSystemPrompt
  + userContext
  + systemContext

[第 4 层：API block 转换层]
apiSystemBlocks =
  attribution header
  + CLI system prompt prefix
  + static blocks (global/org cache)
  + dynamic blocks (uncached)
```

如果再压缩成一句更好记的话：

- `prompts.ts` 负责“生产默认 prompt 原料”
- `systemPrompt.ts` 负责“决定这次最终采用哪份 prompt”
- `queryContext.ts` / `context.ts` 负责“把 user/system context 一起拼进 cache-key prefix”
- `utils/api.ts` / `services/api/claude.ts` 负责“把这份 prefix 再切成 API 可缓存的 block”

这个模板的意义是：以后再看 prompt 相关问题时，不要只盯着 `prompts.ts`。真正决定模型最终看到什么，要至少沿着这 4 层一起看。

再给一个“组合完成后的示意结果”，帮助把抽象模板落地。下面不是仓库里的真实 prompt 全文，而是一个结构上等价、便于记忆的示意例子：

```text
场景假设：
- 当前是 proactive 模式
- 主线程绑定了一个 agent
- 用户额外传了 appendSystemPrompt
- userContext 里有 CLAUDE.md 摘要
- systemContext 里有 gitStatus 快照

[A. 默认 prompt 原料：来自 prompts.ts]
defaultSystemPrompt = [
  "# Intro ...",
  "# System ...",
  "# Doing tasks ...",
  "# Actions ...",
  "# Using your tools ...",
  "# Tone and style ...",
  "# Output efficiency ...",
  "__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__",
  "# Session-specific guidance ...",
  "# Memory ...",
  "# Environment info ...",
  "# MCP Server Instructions ...",
]

[B. effective prompt：来自 systemPrompt.ts]
effectiveSystemPrompt = [
  ...defaultSystemPrompt,
  "# Custom Agent Instructions\nYou are the review agent for ...",
  "Please keep the final answer under 200 words unless I ask for more.",
]

[C. 进入 query() 之后的最终请求骨架]
systemPromptForModel = [
  ...effectiveSystemPrompt,
  "gitStatus: This is the git status at the start of the conversation ...",
]

messagesForModel = [
  user(meta):
    <system-reminder>
    As you answer the user's questions, you can use the following context:
    # claudeMd
    ...
    IMPORTANT: this context may or may not be relevant ...
    </system-reminder>,
  user(actual): "请帮我分析这个模块",
  assistant/history/tool_result ...
]

[D. API 层真正发送的 system blocks：来自 utils/api.ts + services/api/claude.ts]
apiSystemBlocks = [
  { text: "x-anthropic-billing-header: ...", cacheScope: null },
  { text: "CLI system prompt prefix", cacheScope: null 或 org },
  { text: "静态前缀合并块", cacheScope: global 或 org },
  { text: "动态后缀 + systemContext", cacheScope: null 或 org },
]
```

这个例子里最容易记错的点有两个：

1. `userContext` 不是 append 到 system prompt 尾部，而是作为一个额外的 meta user message 插进 messages 最前面。
2. `systemContext` 才是 append 到 system prompt 尾部的那部分，会和 prompt 本体一起进入 system blocks 的切分流程。

#### 7.7.1 按阅读视角看到的 system prompt 主体模板

```text
# 简介
你是一个交互式代理，旨在协助用户处理软件工程任务。请使用下述指令及可用工具为用户提供帮助。你必须永远不要为用户生成或猜测 URL，除非你确信这些 URL 是为了帮助用户进行编程。

# 系统规则
交互方式：你在工具调用之外输出的所有文本都会显示给用户。你可以使用 GitHub 风格的 Markdown 进行格式化。

权限模式：工具在用户选择的权限模式下执行。如果用户拒绝了某个工具调用，请不要尝试重复完全相同的调用，而应思考原因并调整方法。

自动压缩：系统会在对话接近上下文限制时自动压缩之前的消息。这意味着对话不受单一上下文窗口的限制。

# 执行任务规范
拒绝过度开发：不要添加未要求的特性、重构代码或进行“改进”。简单功能的实现不需要额外的可配置性。

注释准则：不要为你没有更改的代码添加文档字符串、注释或类型注解。仅在逻辑不直观时添加注释。

复杂度控制：不要为不可能发生的场景创建助手、工具函数或抽象。三行相似的代码好过一个不成熟的抽象。

安全第一：注意不要引入命令注入、XSS、SQL 注入等安全漏洞。如果你发现编写了不安全的代码，请立即修复。

忠实报告：诚实地报告结果。如果测试失败，请结合相关输出说明情况；如果你没有运行验证步骤，请如实告知，而非暗示其已成功。

# 操作风险控制
在执行操作前，请仔细考虑其可逆性和影响范围。对于难以逆转、影响本地环境之外的共享系统或具有破坏性的操作（如 rm -rf、强制推送、删除分支、修改 CI/CD 管道等），必须先征得用户确认。

# 工具使用规范
专用工具优先：严禁在有专用工具的情况下使用 bash 运行命令。

使用 read_file 代替 cat 或 sed。

使用 edit_file 代替 sed 或 awk。

使用 write_file 代替 echo 重定向。

并行调用：尽可能并行调用互不依赖的工具以提高效率。如果存在依赖关系，则必须按顺序调用。

# 语气与风格
禁止表情：除非用户明确要求，否则不要在所有沟通过程中使用表情符号。

代码引用：引用特定函数或代码片段时，请包含 file_path:line_number 模式，以便用户导航。

简洁明了：不要在工具调用前使用冒号。例如，应使用“让我读取文件。”而非“让我读取文件：”。

# 输出效率
直奔主题。尝试用最简单的方法，不要绕圈子。文本输出应保持简短直接，优先提供答案或行动，而非推理过程。如果你能用一句话说明，就不要用三句。

__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__

# 环境信息
你已被调用至以下环境：

当前工作目录: /home/user/project

Git 仓库状态: 是

平台: Linux

Shell: bash

操作系统版本: Linux 6.6.4

模型信息: 你由 Claude Opus 4.6 模型驱动。该模型的 ID 为 claude-opus-4-6。

知识截止日期: 2025 年 5 月。

# 语言设置
始终使用 中文 (Chinese) 进行回复。对所有解释、注释和与用户的沟通使用中文，但技术术语和代码标识符应保持其原始形式。
```

注意：这一段仍然只对应“最终请求里的 system prompt 主体”。真正发给模型的完整输入，还会像 `7.7` 开头那样继续叠加 `systemContext`，并把 `userContext` 作为一个额外的 meta user message 插到 messages 最前面。

#### 7.7.2 按 API 传输层看到的最终结果

如果继续往下一层，看 Anthropic API 真正收到的“线上的请求形态”，它又不会是上面那种单一长字符串，而会近似变成：

```text
system blocks:
1. attribution / billing header
2. CLI 固定 system 前缀
3. 可缓存静态 system 块
   = defaultSystemPrompt + agent/custom/append 等稳定部分
4. 动态 system 块
   = systemContext，例如 gitStatus

messages:
1. meta user message
   = userContext，例如 CLAUDE.md 摘要
2. actual user message
   = "请帮我分析这个模块"
3. assistant/history/tool_result ...
```

所以，为什么你“新看到的还是这样”？

- 因为这个系统客观上就同时存在“语义组合形态”和“API 传输形态”两种最终结果。
- 我之前只把“分层过程”写出来了，没有把“人眼可读的最终摊平版”单独拎出来。
- 你现在看到的 A/B/C/D 结构并不是错，而是它仍然停留在“内部装配视角”，不是“最终阅读视角”。

这里最值得记住的一句话是：

> 从阅读角度看，最终是一个摊平后的组合输入；从 API 角度看，最终又一定还是 `system blocks + messages` 的二段结构。

### 7.8 本模块的几个明确结论

- `prompts.ts` 不能按“文案常量”理解。
- prompt cache 是全仓一条贯穿主线。
- subagent、MCP、settings path、tool ordering 都在影响 cache key。
- 这是整仓最值得单独深挖的主题之一。

## 8. Query 内核：真正的 agent runtime

## 8.1 `query.ts` 是最核心的执行文件之一

`query.ts` 不是“发请求给模型”的函数，而是整套 agent runtime 的主循环。

它维护的不是单次调用，而是连续状态：

- messages
- toolUseContext
- compact / recovery 状态
- task budget
- tool use summary
- transition reason

### 8.2 `QueryConfig` 的意义是快照 runtime gates

`src/query/config.ts`：

```ts
// Immutable values snapshotted once at query() entry.
// Intentionally excludes feature() gates
export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(...),
      emitToolUseSummaries: isEnvTruthy(...),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
    },
  }
}
```

文件：`src/query/config.ts`

这里的 `runtime gates`，可以理解成“这次 query 期间要不要走某条运行时分支”的开关集合。所谓“快照化”，指的是在 `query()` 入口先把这一小组会影响行为的值读出来、锁进 `QueryConfig`，然后整轮 query 都按这份快照执行，避免中途重新读全局状态导致行为漂移。

- 会被快照的，主要是 immutable env / statsig / session state，例如 `streamingToolExecution`、`emitToolUseSummaries`、`fastModeEnabled`。
- 不在这里快照的，是 `feature()` 树上的另一类 gates；源码注释已经明确写了 `Intentionally excludes feature() gates`。

所以更准确地说，`query()` 不是“冻结所有运行时开关”，而是冻结“这一轮必须保持稳定”的那一小组 gates。

### 8.3 上下文治理流水线不是互斥关系，而是层叠关系

在主循环里，执行顺序是：

1. `applyToolResultBudget`
2. `snipCompactIfNeeded`
3. `microcompact`
4. `contextCollapse.applyCollapsesIfNeeded`
5. `autocompact`

这几个机制不是替代关系，而是处理对象不同、触发层级不同的层叠关系：

1. `applyToolResultBudget` 先限制工具结果体积，处理对象是最新回注回来的高噪声 `tool_result`。
2. `snipCompactIfNeeded` 再裁历史，目标是先用较轻的结构性裁剪把 transcript 拉回预算内。
3. `microcompact` 继续压缩局部高体积块，尽量保留更多细粒度上下文，而不是立刻做整段摘要。
4. `contextCollapse.applyCollapsesIfNeeded` 是 read-time projection，先看能否通过折叠前文把当前 query 拉回阈值。
5. `autocompact` 才是更重的兜底手段；前面几层若已经把上下文压回安全范围，它就可以不触发。

源码注释明确强调了顺序原因，例如 collapse 必须在 autocompact 前：

```ts
// Runs BEFORE autocompact so that if collapse gets us under the
// autocompact threshold, autocompact is a no-op and we keep granular
// context instead of a single summary.
```

文件：`src/query.ts`

所以这里真正该记住的不是“有五步”，而是“每一层都在处理不同对象，前一层不能简单替代后一层”。

### 8.4 `stop_reason === 'tool_use'` 被作者明确判定为不可靠

```ts
// Note: stop_reason === 'tool_use' is unreliable -- it's not always set correctly.
```

文件：`src/query.ts`

这类注释很重要，因为它揭示了系统行为不是照着 API 文档“理想路径”写的，而是经过线上经验修正。

### 8.5 `dumpPromptsFetch` 是一个典型的内存防御优化

```ts
// Creating it once means only the latest request body is retained (~700KB),
// instead of all request bodies from the session (~500MB for long sessions).
```

文件：`src/query.ts`

这是另一个高信号细节：这个团队不是只关注 correctness，也在做长会话下的 memory retention 工程。

### 8.6 `QueryEngine` 是 headless 会话宿主，不是另一套 loop

`QueryEngine.submitMessage()` 主要负责：

- 构造 prompt parts
- 叠加 user/system context
- 初始化 thinking config / model
- 维护 mutable message store
- 记录 transcript
- 将输出标准化为 SDK 事件
- 最终还是进入 `query()`

这一点很重要：REPL 与 headless 共享同一 agent 内核，只是宿主层不同。

## 9. Tool 协议与工具执行架构

## 9.1 `ToolUseContext` 很重，这不是普通函数调用

`src/Tool.ts` 里的 `ToolUseContext` 已经足以说明整个系统设计：

```ts
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(...)
  addNotification?: ...
  ...
  renderedSystemPrompt?: SystemPrompt
}
```

文件：`src/Tool.ts`

也就是说，Tool 执行不是“拿到入参，返回结果”，而是运行在一整个 session runtime 上下文里。

更准确地说，`ToolUseContext` 把几类原本很容易被误以为“外围配套”的状态，全部拉进了工具执行核心：

- 当前可见工具池
- thinking config / mainLoopModel
- MCP client 与资源视图
- 整个消息历史 `messages`
- `getAppState()` / `setAppState()` 这种全局状态读写能力
- `setInProgressToolUseIDs` / `setHasInterruptibleToolInProgress` 这种 UI 与执行器联动状态
- `renderedSystemPrompt` 这种直接影响 fork cache 共享的 prompt 字节快照

所以这里体现出的不是“工具函数参数很多”，而是一个更强的判断：

- Tool call 在这个系统里更像“进入 runtime 内核执行一个受控动作”
- 而不是“普通业务函数调用”

这套设计的优点是，工具可以天然参与：

- session 状态推进
- 权限判断
- UI 反馈
- transcript 记录
- subagent / fork cache 共享

代价也很明确：

- `ToolUseContext` 一旦继续膨胀，工具和 runtime 的耦合面会越来越大
- fork / resume / background agent 之类场景，都会更依赖这份 context 的正确克隆与冻结

### 9.2 `src/tools.ts` 是模型可见工具面的总装厂

这个文件最关键的一句话是：

```ts
/**
 * NOTE: This MUST stay in sync with ... claude_code_global_system_caching,
 * in order to cache the system prompt across users.
 */
```

文件：`src/tools.ts`

这说明工具列表和全局 system prompt cache 之间有直接耦合。

`getAllBaseTools()` 是内建工具真源；`getTools()` 再叠加：

- mode / env gating
- deny rules
- MCP 动态工具
- REPL/agent/headless 上下文

如果把职责拆开看，这里其实有三层：

1. `getAllBaseTools()` 定义“这次构建里理论上可能存在的内建工具全集”
2. `getTools()` 按当前 session 的 mode / deny rules / REPL 状态，把内建工具裁成“当前主线程能看到的 built-in tool 集”
3. `assembleToolPool()` 再把 built-in tools 与 MCP tools 合并、排序、去重，形成真正送去模型的完整工具池

这就是为什么这里不是“工具注册表”，而是“模型可见工具面的总装厂”：

- REPL、主线程、worker agent 并不是各自随手拼工具
- 它们都尽量复用同一套装配函数，避免“显示给模型的工具面”和“实际执行的工具面”发生分叉

这件事的工程收益有两层：

- 行为一致性：不同运行面尽量看到同一套工具定义与权限过滤结果
- cache 一致性：工具池本身就是 cache-safe 参数的一部分，装配路径一旦分叉，就很容易把 prompt cache 打散

### 9.3 工具池的排序是有意为之

```ts
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

文件：`src/tools.ts`

这不是“排序美观”，而是为了稳定 prompt 字节序，提升 cache 命中。

这一句如果不展开，读者其实很难知道“为什么排序会影响 cache”。真正的机制链是：

1. 这个系统发给模型的不只有 system prompt 文本，还有 tool schema 列表。
2. `src/utils/api.ts` 明确把 `serialized tool array bytes` 当成需要稳定保护的对象，而 `toolToAPISchema()` 会把工具池真正转换成发送给模型的 tool schema。
3. 多处 fork / compact / summary 路径又反复强调：要想复用 prompt cache，`system、tools、model、messages prefix、thinking config` 必须保持 cache-safe 参数一致。
4. 所以，一旦工具列表顺序变化，哪怕工具集合没怎么变，最终序列化出来的 tool schema 字节序也会变化，cache key 就会跟着变化。

这里要再区分一下“主证据”和“旁证”：

- 主证据是 `src/tools.ts` 和 `src/utils/api.ts` 自己都在显式保护 tool schema 的序列化字节稳定性。
- `recordPromptState(system + toolSchemas + model)` 这类逻辑更适合当旁证：它证明系统把这组字段视为 cache break 的观测对象，但不该直接等同于服务端真实 cache key 定义。

`assembleToolPool()` 这里真正高明的点不是“排序”，而是“分区排序”：

- built-in tools 先按名字排序
- MCP tools 再按名字排序
- 然后不是混排，而是 `built-in prefix + MCP tail`

源码注释已经把理由讲得很直白了：

- 服务端的 `claude_code_system_cache_policy` 会把“最后一个前缀命中的 built-in tool”之后当成 cache breakpoint
- 如果做 flat sort，让 MCP tool 插进 built-in 中间，那么只要某个 MCP tool 名字排进了这个中段，就会把它后面的 built-in 序列整体往后推
- 结果就是：原本全局可共享的 built-in 前缀不再字节一致，后续整段 cache key 都会失效

也就是说，它提升 cache 命中的方式不是“让整个工具列表永远不变”，而是：

- 把最稳定、最值得全局共享的 built-in tools 固定成一个连续前缀
- 把最容易按用户环境变化的 MCP tools 收敛到动态尾部
- 这样即使 MCP server 增删、重连、工具变化，也优先只污染尾部，而不是把整段 built-in 前缀都打碎

`uniqBy(...)` 这里也不是细枝末节。它保留插入顺序，所以：

- built-in tools 先进入数组，天然拥有优先级
- 如果 MCP tool 与 built-in tool 同名，最终保留下来的仍然是 built-in 版本
- 这进一步保证了“稳定前缀”不会被动态工具覆盖

所以更准确的总结应该是：

> `9.3` 的重点不是“排序让输出更整齐”，而是“通过 built-in 前缀稳定化，把 cache 失效面从整段工具面收缩到 MCP 动态尾部”。

### 9.4 并发不是“所有只读一起跑”，而是“连续 safe block 才一起跑”

`src/services/tools/toolOrchestration.ts`：

```ts
function partitionToolCalls(...) {
  ...
  if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
    acc[acc.length - 1]!.blocks.push(toolUse)
  } else {
    acc.push({ isConcurrencySafe, blocks: [toolUse] })
  }
}
```

文件：`src/services/tools/toolOrchestration.ts`

这意味着：

- 并发度受模型输出顺序影响
- 不是所有 safe tool 自动组成一个大批次
- 只会合并相邻 safe blocks

这里的 `safe` 不是工具类型常量，而是对“这次具体 tool input”做出的运行时判断。源码调用的是 `tool.isConcurrencySafe(parsedInput.data)`，所以同一个工具在不同输入下，也可能落入不同批次策略。

但这还没把“为什么这样设计”讲透。

这里真正保住的是“工具调用的顺序语义”。

`partitionToolCalls()` 不是先把所有 concurrency-safe tools 抽出来再统一并发，而是严格按模型输出顺序线性扫一遍：

- 遇到 safe tool，就尝试并入当前 safe batch
- 遇到非 safe tool，就立刻切断 batch
- 后面的 safe tool 只能重新开一个新 batch，不能回头和前面的 safe batch 合并

这样做的直接结果是：

- 模型输出顺序本身就成了并发边界的一部分
- 一个 non-safe tool 只要夹在中间，就会把前后 safe tools 分成两段

为什么这不是“保守过头”，而是合理设计？

因为工具执行不仅有 message 输出，还有 context 语义。

在 `runTools()` 里，并发 batch 的 `contextModifier` 不是边跑边乱写，而是：

- 先按 `toolUseID` 暂存
- 等这一整个 safe block 跑完后
- 再按原始 block 顺序回放到 `currentContext`

这说明系统宁可少吃一点并发，也要守住两件事：

- transcript / tool_result 的顺序语义
- context 更新的确定性

所以这节更准确的解释应该是：

- 这套并发模型不是“全局收集所有只读工具一起跑”
- 而是“在不打乱模型原始动作顺序的前提下，尽量把相邻 safe 工具并发化”

它的副作用也很有意思：

- 并发上限不只取决于工具本身是否 safe
- 还取决于模型会不会把 safe 调用连续吐出来

换句话说，模型输出质量本身也在影响 runtime 并发度。

### 9.5 `StreamingToolExecutor` 的错误传播是非对称的

```ts
// Only Bash errors cancel siblings.
// Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

文件：`src/services/tools/StreamingToolExecutor.ts`

这很重要，因为它直接影响 agent loop 对并发工具失败的恢复语义。

这里讨论的是 `StreamingToolExecutor` 这条流式执行路径里的 sibling cancellation 语义，不应直接外推为所有工具执行器的统一规则。

如果只写到这里，读者还是不知道“为什么要故意非对称”。

真正的设计意图是：

- 并发读工具之间，失败通常彼此独立
- Bash 工具失败则经常带有隐式依赖链和状态含义

比如：

- `Read` 读某个文件失败，不代表另一个 `Read`、`WebFetch`、`Grep` 就没意义
- 但 `Bash` 里的失败，往往意味着工作目录、前置命令、生成物、shell 状态已经不符合预期

所以这里不是“Bash 更危险”这么简单，而是：

- Bash 更像在执行一段命令式过程
- 一旦这段过程中的某个关键步骤报错，继续让 sibling 并发工具跑完，很多时候只是在制造更多噪音结果

`StreamingToolExecutor` 的处理是：

- 当前 Bash tool 产出 error result 时，标记 `hasErrored = true`
- 设置 `erroredToolDescription`
- 直接对 sibling controller 发 `abort('sibling_error')`
- 其他并发工具再收到 abort 时，会被合成为 synthetic error message

这带来的优点是：

- 对 shell 类失败快速止损
- 对 read/web 类失败保持独立性
- agent loop 后续看到的失败语义更贴近“哪种错误意味着整批动作应该停”

所以这里的“非对称”不是补丁式例外，而是把工具语义分层了：

- Bash 失败 = 可能破坏整批命令执行前提
- 非 Bash 失败 = 默认局部失败，不默认波及兄弟任务

### 9.6 streaming concurrent tools 目前不支持完整 context modifier

```ts
// NOTE: we currently don't support context modifiers for concurrent tools.
```

文件：`src/services/tools/StreamingToolExecutor.ts`

这里最容易被写虚。真正要讲清楚的是“不支持的到底是哪种组合”。

答案是：

- 在 streaming execution 路径里
- 如果某个工具既想被标记为 concurrency-safe
- 又想通过 `contextModifier` 去修改 `ToolUseContext`
- 这一组合目前并没有被完整支持

源码里已经把行为写死了：

- `contextModifiers` 会被收集
- 但真正应用时，只对 `!tool.isConcurrencySafe` 的工具执行
- 也就是说，concurrent tool 的 context modifier 现在基本只被“记住”，不会像串行工具那样真正推进 live context

这一点为什么重要？

因为它决定了未来工具演进的边界：

- 一个工具如果只是“只读 + 出文本结果”，它可以很自然地做成 concurrent-safe
- 但一个工具如果将来既想并发执行，又想顺手改 session state、file history、budget、UI state 或别的 runtime context，就会撞到这里

更关键的一层是：这不是整个工具系统都不支持，而是**streaming concurrent path** 不支持完整语义。

在非 streaming 的 `runTools()` 路径里，并发 block 的 modifier 还能被收集后按顺序回放；但 `StreamingToolExecutor` 这里为了边流式边执行，把这层能力收缩掉了。

所以这一节真正应该让读者记住的是：

> 这不是“有个小 TODO 没做完”，而是“当前 runtime 明确不支持『并发执行 + 上下文变更』同时成立”，这是未来扩展工具类型时必须先补的约束点。

## 10. Agent / Subagent / Fork / Skill 架构

## 10.1 Agent 不是 prompt 片段，而是二级 runtime

更准确地说，`AgentTool` + `runAgent` 不是“把 prompt 发给另一个 agent”这么简单，而是在进入 `query()` 之前，把子 agent 运行时最容易漂移的几件事统一收口。

| 维度 | 要解决的问题 | 主要机制 | 结果 |
| --- | --- | --- | --- |
| agent type 选择 | 调用方不应该自己硬编码 prompt、tools、model、hooks | `loadAgentsDir.ts` 先把 built-in / plugin / user / project agent 统一成 `AgentDefinition`，`AgentTool.tsx` 再根据 `subagent_type` 解析到具体 agent，`runAgent.ts` 按 definition 装配真实运行参数 | 新增 agent 不需要改执行框架，所有 spawn 路径共享同一套分发规则 |
| sync vs async | 子 agent 到底阻塞当前回合，还是后台跑；后台能不能弹权限框；会不会跟父线程一起被取消 | `AgentTool.tsx` 统一计算 `shouldRunAsync`；`runAgent.ts` 再按 `isAsync` 切换 `abortController`、`isNonInteractiveSession` 和 permission prompt 策略 | 长任务不会意外卡死主线程，后台 agent 也不会错误走前台交互语义 |
| fork vs specialized subagent | 有时要找“专家 agent”，有时要“继承我当前上下文继续做” | `subagent_type` 有值时走 specialized agent；省略时在 fork gate 打开下走 `FORK_AGENT`。fork 路径不重新拼父 prompt，而是复用已渲染的 system prompt，并通过 `buildForkedMessages()` 构造共享前缀 | 同一套入口同时覆盖“分工式委派”和“上下文复制式并行”两种需求 |
| worktree / remote / background | 子 agent 可能需要隔离仓库、副本环境或远程环境，还要能脱离前台继续跑 | `AgentTool.tsx` 支持 `isolation`，可落到 `worktree` 或 `remote`；后台执行则注册独立任务、输出文件和恢复元数据 | 解决主仓污染、本地环境不适配、长任务必须脱离前台三类运行环境问题 |
| agent-specific MCP | 不同 agent 需要不同外部能力，不能把所有 MCP 能力全局灌给每个 agent | `runAgent.ts` 会按 agent definition 启动 agent 专属 MCP，再和已有工具池 additive merge、按名称去重 | 外部能力跟 agent 绑定，而不是污染整个 session |
| transcript sidechain | 子 agent 要能单独恢复、单独通知、单独审计，不能把历史全揉进父对话 | `runAgent.ts` 在 query 前记录初始 sidechain transcript 和 metadata，后续每条可记录消息继续增量写入 | 恢复时知道它是谁、原任务是什么、是否绑定 worktree，以及该怎么续跑 |
| permission mode 缩域 | 父线程已经放开的权限，不该自动泄漏给子 agent；后台 agent 也不该卡在权限弹窗上 | `runAgent.ts` 允许 agent definition 覆盖 permission mode，`allowedTools` 重写 session allow，异步 agent 会自动启用 `shouldAvoidPermissionPrompts` | 解决最小权限运行和“后台无 UI 宿主”的权限死锁问题 |
| tool pool 定制 | 子 agent 不能盲继承父 agent 当前工具集，也不能被父线程临时限制误伤 | `AgentTool.tsx` 先独立装配 worker tool pool，`runAgent.ts` 再按 agent definition 过滤；fork 路径则直接继承父 tools 以保持 cache-safe 前缀一致 | 普通 subagent 拿到收敛后的能力，fork child 拿到 cache 友好的精确能力集 |
| thinking config 控制 | 普通 subagent 默认开 thinking 成本太高；fork child 如果关掉又会破坏 prompt cache 前缀一致性 | `runAgent.ts` 对普通 subagent 直接 `thinking: disabled`，但对 fork child 继承父 `thinkingConfig` | 在成本控制和 prompt cache 命中之间做了运行时分流 |

一句话概括：`AgentTool` 负责把一次 agent 调用描述成结构化意图，`runAgent` 负责把这次调用装成一个真正可执行的子 runtime。

这里的“二级 runtime”不是修辞，而是指主线程内部再启动一个拥有独立 `ToolUseContext`、消息链和 `query()` 循环的次级执行宿主。这里的 runtime 也不只是静态 agent 定义文件，而是模型、system prompt、tools、permission、上下文、transcript 和生命周期策略的组合。也正因为如此，agent 才不是简单的 prompt 片段展开。

### 10.2 agent 覆盖优先级是显式的

`src/tools/AgentTool/loadAgentsDir.ts`：

```ts
const agentGroups = [
  builtInAgents,
  pluginAgents,
  userAgents,
  projectAgents,
  flagAgents,
  managedAgents,
]
```

更精确地说，这是构建 `activeAgents` 时，同名 `agentType` 冲突下的覆盖顺序；`allAgents` 仍然保留所有定义。其最短因果链就是：先按 source 分组，再通过 `Map.set(agentType, agent)` 用后者覆盖前者。

这意味着优先级是：

`built-in < plugin < userSettings < projectSettings < flagSettings < policySettings`

### 10.3 `allowedTools` 并不等于完全切断父权限

`runAgent.ts` 明确保留 SDK `cliArg` 级别授权：

```ts
if (allowedTools !== undefined) {
  toolPermissionContext = {
    ...toolPermissionContext,
    alwaysAllowRules: {
      cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,
      session: [...allowedTools],
    },
  }
}
```

文件：`src/tools/AgentTool/runAgent.ts`

这属于非常容易误判的边界。更准确地说：

- `allowedTools` 会重写子 agent 的 session allow。
- `cliArg` 级 allow 会被保留，因为它被视为更高信任来源的“调用方显式授权”。
- 但这不等于“父权限整体泄漏”；deny rules、permission prompt 行为和其他 permission context 仍按子上下文继续生效。

### 10.4 fork child 和普通 subagent 不是一回事

`runAgent.ts` 里，fork child 为了 prompt cache 要继承 exact tools 与 thinking config：

```ts
thinkingConfig: useExactTools
  ? toolUseContext.options.thinkingConfig
  : { type: 'disabled' as const },
...
// Fork children ... need querySource on context.options
...(useExactTools && { querySource }),
```

文件：`src/tools/AgentTool/runAgent.ts`

这说明 fork 的设计目标之一不是“整个请求与父请求完全一致”，而是继承 cache-critical 参数，并构造尽可能长的共享前缀来提高 cache 命中。这里的 `querySource` 则另有职责，它主要服务于递归 fork 防护，不应和 cache 机制混为一谈。

### 10.5 Skill 也不是本地 prompt 宏

这里也要写得条件化一些。更准确的说法是：Skill 是统一的 prompt-command / skill 分发层；默认路径更接近 prompt 展开与 command materialization，而 `context: fork` 的 skill 才会借 subagent runtime 执行。所以它不能被简单理解成“纯文本本地 prompt 宏”，但也不能反过来把所有 skill 都当成 agent runtime。

## 11. 权限与沙箱：真正的安全边界

## 11.1 权限不是一层，而是多层决策叠加

更准确地说，它不是“所有请求都会完整经过的线性七层栈”，而是“核心判定层 + 场景分支层”：

1. 静态 allow/deny rules 与工具级规则检查
2. mode-specific restrictions 与 coordinator/swarm/worker 特判
3. classifier / permission hooks / user prompt 等分支化判定
4. OS sandbox 作为部分路径上的运行时兜底

不同场景再走不同分流：

- interactive 路径里，hooks、classifier 和本地确认对话框并不是全局固定顺序。
- coordinator / swarm 路径里，会先等自动检查结果，再决定是否落到 dialog。
- async / forked 路径里，则更强调“无法本地弹 UI 时如何交给 hooks 或 auto-deny”。

### 11.2 auto mode 会主动识别危险规则

`permissionSetup.ts`：

```ts
// Dangerous patterns:
// 1. Tool-level allow (Bash with no ruleContent)
// 2. Prefix rules for script interpreters (python:*, node:*, etc.)
// 3. Wildcard rules matching interpreters (python*, node*, etc.)
```

文件：`src/utils/permissions/permissionSetup.ts`

这里也要把结论收窄。更准确地说：进入 auto / plan-auto 相关路径时，系统会主动剥离那些可能绕过 classifier 的危险 allow 规则；退出这些路径时，再把配置恢复回来。它保护的重点不是“识别”本身，而是既不让危险规则绕过 classifier，又不永久改坏用户配置。风险面也不只 Bash，还包括 PowerShell、Agent，部分环境下还会覆盖 Tmux。

### 11.3 无本地 UI 的权限交互会走分流路径

`permissions.ts`：

```ts
if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
  ...
  return {
    behavior: 'deny',
    decisionReason: {
      type: 'asyncAgent',
      reason: 'Permission prompts are not available in this context'
    },
  }
}
```

文件：`src/utils/permissions/permissions.ts`

这里也不能把 `headless` 和“无 UI async agent”混成一类。更准确的说法是：

- 无 UI async / forked agent 路径里，系统会先跑 hooks；如果当前宿主确实无法弹 permission prompt，再 fallback 到 auto-deny。
- headless SDK / print 这类路径，并不等于“没有 prompt”，而更接近把审批委托给 `structuredIO` 或外部 `permissionPromptTool` 消费者。

所以这条逻辑真正定义的是：不同宿主在“谁来承接权限交互”这件事上，边界并不一样。

### 11.4 sandbox 的路径语义和 permission rule 的路径语义并不相同

这是整个系统里最容易误读的点之一。

`sandbox-adapter.ts` 不只是“解释 sandbox settings”，还会把 `permissions.allow/deny` 里的 WebFetch / Edit / Read 规则折叠进 sandbox runtime config。但它自己的路径语义并不完全等同于 permission rules。

最短的可操作心智模型是：

- permission rule 里的 `/path` 更接近 settings-relative 规则。
- `sandbox.filesystem.*` 里的 `/path` 则更接近 absolute path；`//path` 还承担兼容旧语义的 escape 角色。
- 例如同样写 `/tmp/demo.txt`，放在 permission rule 里和放在 sandbox filesystem 配置里，解释基准并不相同。

如果不把这个差别说出来，读者很容易把“规则层允许/禁止的路径”和“OS sandbox 真实看到的路径”误以为是一回事。

### 11.5 sandbox 会对 settings / skills 路径做定向写保护

```ts
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())
...
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

文件：`src/utils/sandbox/sandbox-adapter.ts`

这不是“全局文件系统级 deny 同名路径”，而是一个有明确作用域的自保护策略：

- 已知 settings source 对应路径会被加入 deny。
- `originalCwd/current cwd` 下额外的 `.claude/settings.json`、`.claude/settings.local.json`、`.claude/skills` 也会被加入 deny。

这不是通用 shell sandbox 默认会做的事，而是 Claude Code 特有的自保护策略：防止模型通过改 settings / skills 来实现持久化提权，同时补齐 `.claude/commands` / `.claude/agents` 之外的同等级特权面。

## 12. MCP：一等外部能力子系统

## 12.1 MCP 不是插件边角，而是一整层平台

这里先把三个来源名词说清楚：

- `manual server`：用户或项目显式写进 Claude Code 配置里的 MCP server。
- `plugin`：插件包带进来的 MCP server 定义。
- `connector`：来自 claude.ai / 外部平台接入面的 MCP server。

之所以说它是一整层平台，而不是“插件边角”，不是因为事情多，而是因为源码里已经能看到完整的平台链路：

- `config.ts` 负责多源配置加载、合并、去重和优先级策略。
- `auth.ts` 负责 server 级 OAuth、token refresh、session expiry 与持久化状态。
- `useManageMCPConnections.ts` / `client.ts` 负责 transport 建立、连接状态管理，以及 tool / command / resource 暴露。

MCP 子系统至少覆盖：

- 多源配置加载与合并
- connector / plugin / manual server 去重
- auth / token refresh / session expiry
- transport 建立
- tool / command / resource 暴露
- 认证/发现状态持久化与相关缓存失效
- UI 管理与动态接入

### 12.2 plugin / connector 的 MCP 去重采用 `command/url` 身份签名

`src/services/mcp/config.ts`：

```ts
/**
 * Ignores env ... and headers (same URL = same server regardless of auth).
 */
export function getMcpServerSignature(config: McpServerConfig): string | null {
  const cmd = getServerCommandArray(config)
  if (cmd) return `stdio:${jsonStringify(cmd)}`
  const url = getServerUrl(config)
  if (url) return `url:${unwrapCcrProxyUrl(url)}`
  return null
}
```

文件：`src/services/mcp/config.ts`

这里需要把结论收窄一些。更准确的说法是：

- plugin / claude.ai connector 这两条自动来源的去重，采用的是 `command array / url` 身份签名，而不是名字级比较。
- `env` 和 `headers` 明确不参与这一路签名；`sdk` 类型也不会进入这套签名去重。
- manual ↔ manual 的合并并不是“统一按内容签名折叠”，它仍然受 key、来源和作用域优先级控制。

这不一定是 bug，但一定是值得在架构报告中标记的设计假设。

### 12.3 plugin / connector 都可能被手工配置压制

`dedupPluginMcpServers()` 与 `dedupClaudeAiMcpServers()` 都体现出“手工配置优先于自动来源”的原则，但这个判断也不能写成无条件绝对规则。

更准确地说：

- 只有 enabled 且 policy-allowed 的 Claude Code 配置，才会真正压制 plugin / connector 自动来源。
- plugin 路径里还存在 `first-loaded wins` 的内部优先级；被禁用的 plugin server 也不是简单消失，而是会影响 `/mcp` 等可见状态。

这说明 MCP 配置层表达的是一个更细的产品优先级：用户显式声明 > 平台自动接入，但前提是这份显式声明本身真的处于可连接、可生效状态。

### 12.4 Bridge / Remote Control 的 gate 不是单纯 feature flag

`bridgeEnabled.ts` 表明 Remote Control 不是单纯 feature flag：

```ts
return feature('BRIDGE_MODE')
  ? isClaudeAISubscriber() &&
      getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
  : false
```

同时，阻塞检查和缓存检查是分开的：

```ts
export async function isBridgeEnabledBlocking(): Promise<boolean> {
  return feature('BRIDGE_MODE')
    ? isClaudeAISubscriber() &&
        (await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge'))
    : false
}
```

文件：`src/bridge/bridgeEnabled.ts`

单靠这段代码，还不足以直接推出“它在 MCP 之外”，但已经足以说明两件事：

- UI 可见性判断可以用 cached 语义。
- 真正进入 bridge 控制路径时，需要更强的 blocking 检查，而不是只看一个前端可见性 flag。

如果把源码链路继续接上去，它更像一套独立远程控制 runtime：有自己的命令入口、`initReplBridge()` 初始化、OAuth / policy / version 检查，以及 bridge session/handle 生命周期。因此更稳妥的结论是：Remote Control 至少不是“挂在 MCP 配置层上的一个小开关”。

## 13. Session / Transcript / Memory / Compaction

## 13.1 transcript 不是“所有消息都存”

`sessionStorage.ts` 明确说 progress 不应进入 transcript chain：

```ts
/**
 * Progress messages are NOT transcript messages.
 * They are ephemeral UI state and must not be persisted ...
 */
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
```

文件：`src/utils/sessionStorage.ts`

这是 session restore correctness 的核心约束之一。

### 13.2 session memory 是节流触发的，不是每回合都跑

`SessionMemory` 的触发条件设计得很克制：

```ts
const shouldExtract =
  (hasMetTokenThreshold && hasMetToolCallThreshold) ||
  (hasMetTokenThreshold && !hasToolCallsInLastTurn)
```

文件：`src/services/SessionMemory/sessionMemory.ts`

它强调的是“自然停顿点”和“足够上下文增量”。

### 13.3 compact 不是简单摘要

更准确地说，compact 在这个仓库里不是单一机制，而是一组共同服务于上下文治理的路径：

- `session memory compaction`：从长对话里提炼长期记忆，减少后续 query 的重复上下文负担。
- `reactive compact`：遇到 `prompt too long` 等即时压力时做自救裁剪，目标是先把执行拉回可继续的范围。
- `microcompact + traditional compact`：压缩 transcript 中高体积、高噪声块，必要时结合 cached microcompact / cache edit。
- attachment/image/document 处理：图像/文档剥离、attachment 再注入去重，避免大对象长期污染主上下文。

因此“压缩”在这里更像上下文治理，而不是一个单一 summarization 功能。也不是每次 compact 都会同时走完上述所有路径；某些路径还会被 reactive compact 或 context collapse 的结果抑制，避免过度压缩。

## 14. 配置、远程托管设置、策略限制、遥测

## 14.1 这是一个强运行时配置驱动系统

运行时行为确实受很多层共同影响，但它们不是“平权并列开关”，而是一套带 precedence 和观察时点差异的配置系统：

- settings.json
- project settings
- policy settings
- managed drop-in settings
- CLI flags
- env vars
- GrowthBook gates/configs
- policy limits

更应该记住的是下面几条：

- managed settings 内部本身也有顺序：`managed-settings.json -> managed-settings.d/*.json`。
- policy source 还存在更高一层优先级：`remote > plist/hklm > file > hkcu`。
- `getSettingsWithSources()` 和 `getSettingsWithErrors()` 又分别代表 fresh-read 视角与 session-cache 视角，排查问题时不能混着看。
- CLI flags、env vars、GrowthBook、policy limits 常常不是“替代 settings”，而是在更靠后的位置追加 runtime gating 或硬上限。

这使系统高度灵活，但也意味着排查时必须回答一个更精确的问题：某个行为究竟是“被哪层打开的”、还是“被更高层覆盖/封顶的”。

## 14.2 analytics 采用 queue-first 设计

`services/analytics/index.ts` 非常值得注意，因为它刻意做成“无依赖入口 API”：

```ts
if (sink === null) {
  eventQueue.push({ eventName, metadata, async: false })
  return
}
sink.logEvent(eventName, metadata)
```

文件：`src/services/analytics/index.ts`

这保证了：

- 启动早期事件不会因为 sink 还没挂上而丢失
- analytics API 不会引入 import cycle

### 14.3 GrowthBook 是高频运行时 gate/config 平面之一

把 GrowthBook 写成“运行时行为基底”会有点过满。更准确的说法是：它是这套系统里高频出现的一层 gate/config 平面，很多核心行为会被它 gated、tuned 或 latched，但并不是所有行为都由它单点决定。

最典型的三个例子是：

- dynamic config 在实现上就是 feature object，不只是外围实验开关。
- 某些 `1h TTL` eligibility 会被 session latch，避免 mid-request / mid-session 漂移。
- bridge poll interval、fast mode 等行为也可以被它做 fleet-wide 调参。

所以在这个仓库里，feature gate / dynamic config 不是“小优化开关”，而是系统的一部分。

## 15. 关键设计模式与优点

### 15.1 以 cache 稳定性为中心的跨层设计

这是全仓最强的一条主线。更准确地说，这些例子共享的是同一个工程目标：稳定 prompt / tool / schema / TTL 相关字节，冻结会话期高波动 gating，把动态污染尽量推到尾部或延后注入，从而扩大 prompt cache 的可复用范围。

典型点包括：

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- tools 排序与 Statsig 配置同步
- settings JSON content-hash path
- fork child 继承 rendered prompt / exact tools
- MCP instruction delta
- tool schema cache
- 1h TTL eligibility/session latching

如果把这些点映射到策略层，会更容易记住：

- `dynamic boundary`、tools 排序、settings hash path：在做“前缀稳定化”。
- MCP instruction delta：在做“动态尾部污染隔离”。
- fork child 继承 rendered prompt / exact tools：在做“继承已渲染结果，避免重建字节前缀”。
- `1h TTL eligibility/session latching`：在做“把高波动 eligibility 锁定到 session/query 边界内”。

### 15.2 多运行面共享同一 query 内核

这让系统：

- 不至于在 REPL/headless/agent 各写一套模型循环
- 但也让宿主层与内核层的边界变得很重要

### 15.3 权限安全链设计较成熟

这个系统已经不是“用户点 Yes/No”的权限模型了，但它也不是每次调用都完整经过同一条线性流水线。更准确地说，它是一套分支化的安全栈：

- 先看静态规则、mode、工作区/路径限制。
- 再看这次调用是否会命中 classifier。
- 然后进入 permission hooks、interactive prompt 或 async auto-deny。
- 最后在需要的路径上，再由 sandbox 形成 OS 级兜底。

因此它成熟的点，不是“层数多”，而是把不同风险面拆开了。以 `Edit/Bash` 这类调用为例，真正决定结果的往往是“规则/模式 -> 是否命中 classifier -> 当前场景能否弹 permission -> sandbox 是否仍然兜底”的组合，而不是所有层都等权参与。

### 15.4 MCP 与 Agent 已平台化

很多工程做到一定规模才会把“工具扩展”和“子代理”当平台能力，这个仓库已经走到了这一步。更落地地看，至少有三个可观察信号：

- 有独立的 MCP 管理 UI 和命令面，而不是把外部 server 接入当成零散配置。
- MCP skills / commands 可以进入 model-invocable command 面，不只是外围配套。
- Agent runtime 已经支持 background、worktree、remote、agent-specific MCP 等差异化执行形态。

## 16. 风险、隐式耦合与技术债

## 16.1 超级枢纽文件

风险最高的几个文件：

- `src/main.tsx`
- `src/screens/REPL.tsx`
- `src/query.ts`
- `src/tools/AgentTool/AgentTool.tsx`
- `src/tools/AgentTool/runAgent.ts`
- `src/services/mcp/client.ts`

问题不是“它们很大”，而是它们同时承载：

- 编排
- 策略
- 状态
- feature gate
- 性能优化
- 正确性约束

### 16.2 关键隐式契约过多

更适合按“违反后会坏成什么样”来分组理解：

- cache 稳定性契约：  
  `prompt dynamic boundary` 必须和 API split 逻辑一致；tool 列表与全局 system cache 配置必须同步；fork child 必须继承 exact tools / rendered prompt。  
  违反后的直接后果，是 prompt cache 命中骤降，fork/summary 继承失效，原本可共享的前缀被打散。
- 递归与恢复契约：  
  `querySource` 必须在线程间保留；progress message 不能进入 transcript chain。  
  违反后的直接后果，是 autocompact / restore 对调用来源和消息主链判断失真，进而出现递归保护错判或 resume 漂移。
- MCP 身份契约：  
  MCP dedup 会忽略 `env/headers`。  
  违反预期时的直接后果，是合法多实例可能被折叠，或连接身份被过度共享。
- 权限语义契约：  
  sandbox 路径语义与 permission rule 路径语义并不完全一致。  
  违反后的直接后果，是“规则上看似允许/禁止”和“OS 级实际行为”出现偏差，调试和审计都会变难。

这些都不是单文件能看懂的约束。

### 16.3 反编译/研究仓边界问题

- 不能简单把“当前仓库文本”当成官方权威实现
- `prepare-src.mjs` 会改 `src/`
- build 链是 best-effort
- feature-gated 模块不完整

### 16.4 一处显眼的静态可疑点

以当前仓库文本看，`src/constants/prompts.ts` 中 `getAntModelOverrideSection()` 调用了 `getAntModelOverrideConfig()`，但顶部导入区未见对应导入。  
这类问题有两种可能：

1. 反编译/预处理导致文本不完整。
2. 这里确实存在构建层面依赖某种额外注入/变换。

因为本轮不做 debug，不把它定性成 bug，但它值得在后续专项核对时优先检查。

## 17. 哪些机制最值得继续深挖

### 17.1 Prompt cache 全链路隐式契约表

这一节值得单独拉专题，不是因为名词多，而是这些点共同决定三件事：

- prompt 前缀字节是否稳定
- 请求是否仍具备 cache eligibility
- 复用范围能停在 built-in 前缀，还是被动态尾部污染打断

建议单独拉一个专题，串起这些点：

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- `splitSysPromptPrefix`
- `buildSystemPromptBlocks`
- tool 排序
- settings hash path
- MCP instruction delta
- fork exact-tools / rendered prompt
- cache_control / 1h TTL latching

一个很好记的反例是：如果 settings 路径每次随机、fork child 又不继承 rendered prompt，那么语义没变也会让字节前缀漂移，prompt cache 前缀就会被持续打爆。

### 17.2 Agent 权限边界图

建议专门拆，而且要按“决策输入 / 中间 gate / 最终表现”来画：

- 权限继承来源：`sync vs async agent`、`allowedTools`、inherited `cliArg` rules。
- 是否可弹确认：`bubble mode`、`canShowPermissionPrompts`（当前宿主是否具备权限确认 UI 承接能力）、background/async agent 在什么场景下直接避免 permission prompt。
- 最终执行表现：background agent 自动检查/等待、哪些路径会直接 deny、哪些路径会把决策冒泡回父线程。
- 能力作用域：`agent-specific MCP` 如何与父线程已有能力叠加或缩域。

### 17.3 Session restore correctness

所谓 restore correctness，不是“历史能读回来”这么简单，而是恢复后的 query 还必须保持与中断前同样的消息主链、compact 边界和 sidechain 关联，不产生行为漂移。

建议继续串：

- `transcript chain`：主消息链顺序不能断，否则 restore 后继续 query 读到的历史就变了。
- `progress exclusion`：非 transcript 进度事件必须排除在主链外，否则 replay 会引入重复噪音。
- `tombstone rewrite`：被删除/被替换过的内容要保留稳定 tombstone 标记，否则 ID 和引用关系会漂移。
- `sidechain transcript`：subagent / fork / background 的侧链 transcript 必须能重新挂回主链上下文。
- `compact boundary`：restore 后必须知道 compacted prefix 从哪里开始，否则会双重 compact 或丢失细节。
- `context collapse replay`：被 collapse 过的区段要能确定性重放，否则恢复后的可见上下文会和中断前不同。

### 17.4 MCP 多实例假设

重点核实：

- 忽略 env/headers 的 dedup 是否会压掉合法多实例
- manual > plugin > connector 的优先级是否在所有 UI/agent/headless 路径都一致

### 17.5 `main.tsx` / `REPL.tsx` 的拆层机会

这是未来维护性的最大杠杆之一。

## 18. 附录：高信号源码摘录

### A. 系统 prompt 动静边界

```ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

文件：`src/constants/prompts.ts`

信号点：这里存在显式的 prompt 动静边界 marker。边界：它证明“边界被一等对待”，但单看这一段还不能直接推出 API 层具体如何切块。

### B. settings hash path 避免 cache 爆炸

```ts
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
});
```

文件：`src/main.tsx`

信号点：这里证明 settings 临时路径是按内容 hash 稳定映射，而不是为了“路径好看”。边界：它能证明稳定前缀设计意图，但不能单独推出真实 cache 命中率数字。

### C. Query runtime gate 快照

```ts
export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(...),
      emitToolUseSummaries: isEnvTruthy(...),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
    },
  }
}
```

文件：`src/query/config.ts`

信号点：这里证明 `query()` 会在入口快照一部分 runtime gates。边界：源码同时明确写了 `feature()` gates 不在这份快照里。

### D. ToolUseContext 体现工具执行不是普通函数调用

```ts
export type ToolUseContext = {
  options: {
    tools: Tools
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(...)
  addNotification?: ...
  renderedSystemPrompt?: SystemPrompt
}
```

文件：`src/Tool.ts`

信号点：这里证明工具执行发生在完整 runtime context 上，而不是普通函数调用。边界：它能说明耦合面很大，但不能单靠类型定义推断每个字段每次都会被用到。

### E. tool 并发批次规则

```ts
if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
  acc[acc.length - 1]!.blocks.push(toolUse)
} else {
  acc.push({ isConcurrencySafe, blocks: [toolUse] })
}
```

文件：`src/services/tools/toolOrchestration.ts`

信号点：这里证明并发批次只会合并连续 safe block。边界：它证明的是批次边界规则，不等于“所有 safe 工具都会被全局并发”。

### F. Bash error 会中止 sibling 并发工具

```ts
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

文件：`src/services/tools/StreamingToolExecutor.ts`

信号点：这里证明 Bash error 会在 streaming 路径中止 sibling。边界：它描述的是这条执行器路径的语义，不应直接外推为所有工具执行器的统一规则。

### G. Agent 保留 SDK `cliArg` 级授权

```ts
alwaysAllowRules: {
  cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,
  session: [...allowedTools],
},
```

文件：`src/tools/AgentTool/runAgent.ts`

信号点：这里证明 agent 即使收窄到 `allowedTools`，也仍会保留父线程的 `cliArg` 级授权。边界：这说明边界复杂，不等于 agent 完全继承父权限。

### H. sandbox 主动保护 `.claude/skills`

```ts
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

文件：`src/utils/sandbox/sandbox-adapter.ts`

信号点：这里证明 sandbox 会主动保护 `.claude/skills` 等敏感路径。边界：这是 Claude Code 自己加的自保护策略，不是 OS sandbox 的默认常识。

### I. MCP server 内容签名去重

```ts
export function getMcpServerSignature(config: McpServerConfig): string | null {
  const cmd = getServerCommandArray(config)
  if (cmd) return `stdio:${jsonStringify(cmd)}`
  const url = getServerUrl(config)
  if (url) return `url:${unwrapCcrProxyUrl(url)}`
  return null
}
```

文件：`src/services/mcp/config.ts`

信号点：这里证明 plugin / connector 自动来源的 server dedup 采用 `command/url` 身份签名。边界：它不会把 `env/headers` 算进签名，`sdk` 也不走这一路。

### J. analytics queue-first

```ts
if (sink === null) {
  eventQueue.push({ eventName, metadata, async: false })
  return
}
sink.logEvent(eventName, metadata)
```

文件：`src/services/analytics/index.ts`

信号点：这里证明 analytics 入口是 queue-first。边界：它能证明启动早期不丢本地事件队列，但不能单靠这一段推断远端 sink 一定成功送达。

### K. session memory 的双阈值触发

```ts
const shouldExtract =
  (hasMetTokenThreshold && hasMetToolCallThreshold) ||
  (hasMetTokenThreshold && !hasToolCallsInLastTurn)
```

文件：`src/services/SessionMemory/sessionMemory.ts`

信号点：这里证明 session memory 提取不是每回合都跑，而是受双阈值控制。边界：它说明触发条件，并不能单独推出提取质量。

## 19. 最终结论

如果只给一句判断：

从本轮源码证据看，这个仓库更像一个以 prompt cache 稳定性、权限安全链、MCP 平台化、subagent 生命周期管理为核心的 agent 平台；CLI 更接近它最外层的交互壳，而不是系统本体。

这个判断主要由三条证据链支撑：

- `prompt / tools / settings path / fork inheritance` 多处都在保护 cache-safe 前缀，而不是把 prompt 当成一次性文本。
- `permissions + classifier + hooks + sandbox` 形成的是分支化安全链，而不是单层弹窗模型。
- `MCP + Agent + Query runtime` 共同构成平台层；REPL / headless / bridge 更像宿主壳和运行面。

如果从工程管理角度给结论：

- 当前最需要保护的不是“某个单功能模块”，而是跨层隐式契约。
- 当前最值得投入的不是“再加一个功能”，而是把超级枢纽周边的边界画清楚。
- 当前最该深入研究的技术主线，是 `Prompt cache / Query runtime / Agent permission / MCP dedup & lifecycle / Session restore correctness`。

## 附录：名词解释

这一节按“第一次接触 agent 系统”的阅读门槛来写，只解释最常见、最容易混淆的词。

- `Agent`：可以理解为一个“会读指令、会决定下一步、必要时会调用工具”的智能执行体。
- `主线程 / Main agent`：直接和用户对话、维护全局上下文、负责最终交付结果的那个 agent。
- `Subagent`：被主线程派出去处理子任务的 agent。它通常只负责一小块工作，做完后把结果回传。
- `Coordinator`：偏调度和拆解的角色，重点是分配任务、安排顺序、汇总结果，不一定亲自做具体实现。
- `Prompt`：喂给模型的指令文本。可以把它理解成“模型当前这次工作的任务说明书”。
- `System prompt`：优先级最高的一层 prompt，用来规定身份、边界、行为规则和输出风格。
- `User prompt`：用户这一轮真正提出的问题或要求。
- `Context`：模型当前能看到的背景信息，比如历史消息、文件内容、仓库状态、`CLAUDE.md` 摘要。
- `Context window`：模型一次最多能读进去多少文本。超出以后，就必须裁剪、压缩或做摘要。
- `Tool`：模型自己不能直接完成、必须借助外部能力做的动作，比如读文件、执行命令、联网、起子 agent。
- `Tool call`：agent 决定调用某个工具的那一次具体执行。
- `MCP`：`Model Context Protocol`。它的作用是把外部工具、服务、数据源，用统一协议接进 agent 系统。
- `Sandbox`：受限制的执行环境。它决定 agent 能读哪些文件、能写哪些路径、能不能联网、能不能做危险操作。
- `Permission`：权限规则。它决定某个工具调用是不是允许执行，以及是否需要用户确认。
- `Prompt cache`：把稳定不变的 prompt 前缀缓存起来，减少重复请求成本和延迟。这个仓库非常重视这一点。
- `Query`：一次完整的 agent 执行过程，不只是一次模型 API 调用，还包括消息组装、工具循环、状态推进和结果返回。
- `Tool result`：工具执行结束后回注给模型的结果消息。模型下一步怎么判断，很大程度取决于这些结果有没有被正确组织回 transcript。
- `Tool schema`：发给模型的工具定义，通常包含工具名、描述、输入参数 schema。它不是工具实现本体，而是模型看到的工具接口描述。
- `User context`：额外注入给模型的用户侧背景信息。这个仓库里它通常以一个 meta user message 的形式出现在正式用户问题前面。
- `System context`：额外拼进 system 侧的运行时背景信息，比如 git 状态、环境快照等。它和 system prompt 本体一起影响 cache-safe 前缀。
- `System reminder`：系统自动附加的提醒性上下文，通常包在 `<system-reminder>` 标签里。它不是用户原话，但模型必须把它当成高优先级辅助信息。
- `Tool pool`：某一轮 query 真正对模型可见、且实际允许执行的工具集合。它不是“仓库里所有工具”，而是 built-in、MCP、权限、mode、env 共同裁出来的结果。
- `Cache key / Cache-safe params`：为了复用 prompt cache，必须保持字节一致的那组请求参数。这个仓库里常见的关键项包括 `system`、`tools`、`model`、消息前缀和 thinking config。
- `Cache breakpoint`：服务端在 prompt 前缀里切分缓存边界的位置。边界前后是否稳定，直接影响到底能复用多少 cache。
- `Fork`：从主线程派生出一个继承现有上下文的子执行分支。它的核心目标之一是尽量复用父线程已有的 prompt cache，而不是从零开始重新建上下文。
- `Worktree`：Git 提供的隔离工作目录。并行 agent 可以各自在自己的 worktree 改代码，减少互相覆盖文件的风险。
- `Headless`：没有 REPL/TUI 交互壳的运行形态，只保留 agent 内核和程序化接口，常见于 SDK、后台任务或自动化执行路径。
- `Compaction`：当对话太长时，不是硬截断，而是把长历史压缩成更短、还能继续工作的表示形式，让 query 能在上下文预算内继续跑下去。
- `Resume / Restore`：把之前保存过的会话状态重新还原出来继续执行。难点不只是“把消息读回来”，而是要把 transcript、工具状态、上下文边界一起恢复正确。
- `Context modifier`：某个工具执行后，对 `ToolUseContext` 施加的内部状态修改。它和普通 `tool_result` 不同，后者是给模型看的，前者是给 runtime 自己用的。
- `Concurrency-safe`：某次具体工具调用是否允许与其他调用并发执行的判定结果。它关注的是这次输入和副作用，不只是工具名字。
- `Interrupt behavior`：某个工具在运行时，遇到用户打断时应该被取消，还是必须阻塞到完成的策略。
- `Transcript`：一次 query 到当前为止的完整对话记录，通常包含 user、assistant、tool result 等消息。它是模型继续往下工作的直接输入。
- `Collapse / Transcript collapse`：当 transcript 太长、快超出上下文上限时，系统把前面内容压缩成摘要的过程。目的不是“删除历史”，而是“用更短的形式保住关键信息”。
- `Task budget`：这次任务允许消耗的资源上限，可以理解成给 agent 设的“油量表”。常见约束包括轮数、token、工具调用次数或执行时间。
- `Proactive`：一种更主动的工作模式。agent 不只是等用户一步一步下指令，而会围绕当前目标自己继续拆解、推进，并补上它判断为必要的步骤。
- `Turn`：对话中的一轮发言。一次用户发言算一轮，一次 assistant 回复也算一轮。
- `Runtime`：运行时系统。它负责把 prompt、工具、权限、状态、模型请求这些东西真正组织起来跑。
- `Session memory`：从长对话里提炼出来的长期记忆，用来减少“对话一长就忘前文”的问题。
- `GrowthBook`：远程 feature gate / dynamic config 平面。它负责把一部分运行时开关和调参值下发给客户端。
- `TTL`：`time to live`，缓存或资格判断的有效期窗口。过了这个时间，系统就需要重新判定或刷新。
- `Latching / latch`：在一次 session 或 query 边界把某个值锁住，不让它在执行中途漂移。
- `querySource`：标记这次 query 从哪个调用来源或递归路径发起。它会影响 autocompact、restore 和递归保护这类判断。
- `async auto-deny`：在当前场景无法弹出 permission prompt 时，runtime 自动拒绝本次高风险调用，并返回原因。
- `cache_control`：请求/服务端使用的缓存策略标记，用来决定哪些 prompt 段可被复用、TTL 怎么算、cache breakpoint 在哪里。
- `Bubble mode`：一种权限处理策略。当前 agent 自己不直接拍板，而是把决策“冒泡”回父线程或上层宿主处理。
- `allowedTools`：显式给某个 agent/session 开的一组工具白名单。它会收窄这次会话能用的工具面，但不一定抹掉更高信任来源的授权。
- `cliArg`：来自 CLI 显式参数的授权来源。系统通常把它视为一类更高信任的“用户明确声明”权限。
- `Transcript chain`：会话恢复时必须重建正确的那条主消息链。顺序一旦错位，后续 query 看到的历史就会变。
- `Tombstone`：给已删除、已替换或已折叠内容保留的占位标记，用来维持引用关系和重放结构。
- `Sidechain transcript`：subagent / fork / background 路径使用的侧链 transcript。它不是主链，但必须能和主链重新挂接。

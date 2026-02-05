# OpenClaw Agentic/LLM Flow Architecture

This document provides a comprehensive analysis of how the agentic loop and LLM interactions work in the OpenClaw codebase.

## Overview

OpenClaw implements a sophisticated agentic architecture that:
1. Receives user prompts via CLI or messaging channels
2. Routes them through a model-agnostic agent layer
3. Streams responses and tool calls back to users
4. Persists conversation history for multi-turn interactions

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLI / Messaging Channels                        │
│        (Telegram, Discord, Signal, Slack, WhatsApp, Web, etc.)          │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      src/commands/agent.ts                               │
│                         agentCommand()                                   │
│  • Validates session parameters (--to, --session-id, --agent-id)        │
│  • Resolves configuration and model selection                            │
│  • Handles thinking level and verbose overrides                          │
│  • Manages auth profiles and model fallbacks                             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
              ┌──────────────────────┴──────────────────────┐
              ▼                                              ▼
┌─────────────────────────────┐              ┌─────────────────────────────┐
│     runCliAgent()           │              │   runEmbeddedPiAgent()      │
│  (CLI provider sessions)    │              │  (Embedded agent sessions)  │
│  src/agents/cli-runner.ts   │              │  src/agents/pi-embedded.ts  │
└─────────────────────────────┘              └──────────────┬──────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              src/agents/pi-embedded-runner/run/attempt.ts                │
│                         runEmbeddedAttempt()                             │
│  • Resolves workspace and sandbox context                                │
│  • Loads skills and bootstrap files                                      │
│  • Creates tools via createOpenClawCodingTools()                         │
│  • Builds system prompt with runtime context                             │
│  • Acquires session write lock                                           │
│  • Prepares SessionManager for conversation history                      │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        createAgentSession()                              │
│              (@mariozechner/pi-coding-agent)                             │
│  • Initializes agent with model, tools, system prompt                   │
│  • Sets up SessionManager for history persistence                        │
│  • Configures SettingsManager for agent preferences                      │
│  • Creates the core agent loop orchestrator                              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    ▼                                  ▼
┌────────────────────────────────┐    ┌────────────────────────────────────┐
│  subscribeEmbeddedPiSession()  │    │     activeSession.prompt()         │
│  (Event subscription setup)    │    │   (Initiates LLM interaction)      │
│  src/agents/pi-embedded-       │    │                                    │
│       subscribe.ts             │    │  Uses streamSimple from            │
└────────────────────────────────┘    │  @mariozechner/pi-ai               │
                                      └──────────────┬─────────────────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         LLM Provider APIs                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │Anthropic │ │  OpenAI  │ │  Google  │ │ Bedrock  │ │Custom/Proxy  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Streaming Event Handler                            │
│            createEmbeddedPiSessionEventHandler()                         │
│  src/agents/pi-embedded-subscribe.handlers.ts                            │
│                                                                          │
│  Event Types:                                                            │
│  • message_start  → handleMessageStart()  - Assistant begins             │
│  • message_update → handleMessageUpdate() - Text deltas stream           │
│  • message_end    → handleMessageEnd()    - Assistant completes          │
│  • tool_execution_start  → handleToolExecutionStart()                    │
│  • tool_execution_update → handleToolExecutionUpdate()                   │
│  • tool_execution_end    → handleToolExecutionEnd()                      │
│  • agent_start/agent_end → Lifecycle events                              │
│  • auto_compaction_start/end → History compaction                        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Entry Point: `agentCommand()`
**File:** `src/commands/agent.ts:63-527`

The main CLI entry point that orchestrates the agent execution:

```typescript
export async function agentCommand(
  opts: AgentCommandOpts,
  runtime: RuntimeEnv = defaultRuntime,
  deps: CliDeps = createDefaultDeps(),
) {
  // 1. Validate inputs
  // 2. Load config and resolve agent/session
  // 3. Build skills snapshot
  // 4. Resolve model and thinking level
  // 5. Execute via runWithModelFallback()
  // 6. Deliver results and update session store
}
```

Key responsibilities:
- Validates `--to`, `--session-id`, `--session-key`, `--agent-id` parameters
- Resolves workspace directory and agent configuration
- Handles model selection with allowlists and overrides
- Manages auth profiles for multi-account scenarios
- Implements model fallback chain on failures

### 2. Agent Session Runner: `runEmbeddedAttempt()`
**File:** `src/agents/pi-embedded-runner/run/attempt.ts:134-895`

The core agent loop implementation:

```typescript
export async function runEmbeddedAttempt(
  params: EmbeddedRunAttemptParams,
): Promise<EmbeddedRunAttemptResult> {
  // 1. Setup workspace and sandbox context
  // 2. Load skills and bootstrap files
  // 3. Create tools with policy filtering
  // 4. Build system prompt
  // 5. Create agent session
  // 6. Subscribe to events
  // 7. Execute prompt and handle streaming
  // 8. Return results
}
```

Key operations:
- **Sandbox resolution** (line 148-157): Determines if running in sandboxed mode
- **Tool creation** (line 203-236): Builds tool array via `createOpenClawCodingTools()`
- **System prompt** (line 340-388): Constructs context-aware prompt with runtime info
- **Session creation** (line 453-468): Initializes agent via external SDK
- **Event subscription** (line 602-620): Sets up streaming handlers
- **Prompt execution** (line 789-792): Sends user message to LLM

### 3. Tool System: `createOpenClawCodingTools()`
**File:** `src/agents/pi-tools.ts:113-434`

Creates the complete tool array available to the agent:

```typescript
export function createOpenClawCodingTools(options?: {
  exec?: ExecToolDefaults & ProcessToolDefaults;
  sandbox?: SandboxContext | null;
  sessionKey?: string;
  // ... many more options
}): AnyAgentTool[] {
  // 1. Resolve tool policies (global, agent, group, sandbox)
  // 2. Create base coding tools (read, write, edit)
  // 3. Create exec/process tools
  // 4. Add channel-specific tools
  // 5. Add OpenClaw custom tools
  // 6. Filter by policies
  // 7. Normalize schemas
}
```

Tool categories:
- **File operations**: `read`, `write`, `edit` (from `@mariozechner/pi-coding-agent`)
- **Execution**: `exec` (bash), `process` (background processes)
- **Messaging**: `message`, `sessions-send` (cross-session/channel)
- **Memory**: `memory` (session persistence)
- **Browser**: `browser` (web automation)
- **Scheduling**: `cron` (scheduled tasks)
- **Vision**: `image` (when model supports vision)
- **Web**: `web-search`, `web-fetch`

Tool policy resolution chain:
1. Profile policy (`tools.profile`)
2. Provider profile policy (`tools.byProvider.profile`)
3. Global policy (`tools.allow`)
4. Agent policy (`agents.<id>.tools.allow`)
5. Group policy (channel/group-level restrictions)
6. Sandbox policy (container restrictions)
7. Subagent policy (spawned session restrictions)

### 4. Event Subscription: `subscribeEmbeddedPiSession()`
**File:** `src/agents/pi-embedded-subscribe.ts:30-564`

Sets up the streaming event pipeline:

```typescript
export function subscribeEmbeddedPiSession(params: SubscribeEmbeddedPiSessionParams) {
  const state: EmbeddedPiSubscribeState = {
    assistantTexts: [],
    toolMetas: [],
    deltaBuffer: "",
    blockBuffer: "",
    // ... state management
  };

  // Create event handler and subscribe
  const unsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );

  return {
    assistantTexts,
    toolMetas,
    unsubscribe,
    // ... helper methods
  };
}
```

State tracking:
- `assistantTexts`: Accumulated assistant response chunks
- `toolMetas`: Tool execution metadata
- `deltaBuffer`: Raw streaming text accumulator
- `blockBuffer`: Chunked output buffer
- `blockState`: Tracks `<think>` and `<final>` tag parsing
- `messagingToolSentTexts`: Deduplication for messaging tools

### 5. Message Event Handlers
**File:** `src/agents/pi-embedded-subscribe.handlers.messages.ts`

#### `handleMessageStart()` (line 32-49)
- Resets state for new assistant message
- Triggers typing indicator via `onAssistantMessageStart()`

#### `handleMessageUpdate()` (line 51-188)
- Processes `text_delta`, `text_start`, `text_end` events
- Accumulates streaming chunks in `deltaBuffer`
- Strips `<think>` blocks for clean output
- Emits partial replies via `onPartialReply()`
- Handles block chunking for natural message breaks

#### `handleMessageEnd()` (line 190-332)
- Finalizes assistant message
- Promotes inline `<think>` tags to structured blocks
- Emits final block reply
- Handles reasoning output for `includeReasoning` mode

### 6. Tool Event Handlers
**File:** `src/agents/pi-embedded-subscribe.handlers.tools.ts`

#### `handleToolExecutionStart()` (line 39-114)
- Flushes pending block replies before tool
- Tracks tool metadata
- Emits tool start event
- Tracks pending messaging tool sends

#### `handleToolExecutionUpdate()` (line 116-146)
- Emits partial tool results
- Used for streaming tool output

#### `handleToolExecutionEnd()` (line 148-200+)
- Records tool result/error
- Commits or discards pending messaging texts
- Emits verbose tool output if enabled

## Data Flow

### User Message → LLM

```
User Input
    │
    ▼
agentCommand() validates and resolves session
    │
    ▼
runEmbeddedAttempt() sets up context
    │
    ▼
createAgentSession() initializes agent
    │
    ▼
before_agent_start hooks (optional context injection)
    │
    ▼
activeSession.prompt(text, { images? })
    │
    ▼
streamSimple() → LLM Provider API
```

### LLM Response → User

```
LLM Provider API (streaming)
    │
    ▼
Event Handler (message_update events)
    │
    ├─► deltaBuffer accumulation
    ├─► Block tag stripping (<think>, <final>)
    ├─► onPartialReply() (streaming UI)
    │
    ▼
Tool Call Detection (tool_execution_start)
    │
    ├─► Tool execution
    ├─► onToolResult() (verbose output)
    │
    ▼
message_end event
    │
    ├─► Final text assembly
    ├─► onBlockReply() (channel delivery)
    │
    ▼
SessionManager.append() (history persistence)
```

## Session Management

### Session Identification
- **Session ID**: Unique identifier for conversation thread
- **Session Key**: Compound key `<agentId>:<sessionId>` for routing
- **Session File**: JSONL file at `~/.openclaw/sessions/<sessionId>.jsonl`

### History Management
**File:** `src/agents/pi-embedded-runner/session-manager-init.ts`

- `SessionManager.open()`: Loads existing session or creates new
- `getLeafEntry()`: Gets most recent message
- `branch()`: Creates conversation fork
- `resetLeaf()`: Resets to clean state
- History limiting: Configurable max turns via `getDmHistoryLimitFromSessionKey()`

### Compaction
- Auto-compaction triggered when context exceeds limits
- `auto_compaction_start/end` events signal compaction
- `waitForCompactionRetry()` handles retry logic

## Provider Support

### Supported Providers
1. **Anthropic** (Claude models)
2. **OpenAI** (GPT models, including Codex)
3. **Google** (Gemini models)
4. **AWS Bedrock** (various models)
5. **Custom/Proxy** (configurable endpoints)

### Provider-Specific Handling
- **Google/Gemini**: `sanitizeToolsForGoogle()`, `validateGeminiTurns()`
- **Anthropic**: `validateAnthropicTurns()`, OAuth tool name remapping
- **OpenAI Codex**: `apply-patch` tool enabled for patch-based edits

### Model Resolution
**File:** `src/agents/pi-embedded-runner/model.ts`

```typescript
export function resolveModel(provider, modelId, agentDir, cfg) {
  const modelRegistry = discoverModels(authStorage, agentDir);
  const model = modelRegistry.find(provider, modelId);
  return model;
}
```

## Hooks System

### `before_agent_start` Hook
- Runs before prompt sent to LLM
- Can inject context via `prependContext`
- Access to prompt and existing messages

### `agent_end` Hook
- Runs after agent completes (fire-and-forget)
- Receives final messages, success status, duration
- Used for analytics/logging plugins

## Key External Dependencies

| Package | Purpose |
|---------|---------|
| `@mariozechner/pi-ai` | Core streaming (`streamSimple`) |
| `@mariozechner/pi-coding-agent` | Agent session, tools, SessionManager |
| `@mariozechner/pi-agent-core` | Type definitions (AgentMessage, AgentEvent) |
| `@aws-sdk/client-bedrock` | AWS Bedrock integration |

## Error Handling

### Timeout Handling
- Configurable via `--timeout` flag
- `runAbortController` manages cancellation
- `isTimeoutError()` detection for fallback decisions

### Model Fallback
**File:** `src/agents/model-fallback.ts`

```typescript
runWithModelFallback({
  cfg,
  provider,
  model,
  fallbacksOverride,
  run: (provider, model) => { ... }
})
```

Falls back to alternative models on:
- Rate limits
- Context length exceeded
- Provider errors

### Tool Errors
- `isToolResultError()` detects error responses
- `lastToolError` tracked in subscription state
- Errors don't abort agent loop (agent sees error and can retry)

## Streaming Modes

### Block Reply Break Modes
- `text_end`: Emit on each `text_end` event (chunked)
- `message_end`: Emit only on `message_end` (complete message)

### Reasoning Modes
- `off`: No reasoning output
- `on`: Include reasoning in final response
- `stream`: Stream reasoning via `onReasoningStream()`

### Verbose Levels
- `off`: Minimal output
- `on`: Include tool summaries
- `full`: Include tool output details

## Security Considerations

### Sandbox Mode
- Container isolation via Docker
- Restricted workspace access (`rw` or `ro`)
- Tool filtering based on sandbox policy
- Browser bridge URL for isolated web access

### Tool Policy Chain
Multiple layers of tool filtering ensure defense in depth:
1. Profile-level restrictions
2. Provider-specific restrictions
3. Global allow/deny lists
4. Agent-specific policies
5. Group/channel policies
6. Sandbox restrictions
7. Subagent restrictions

### Path Validation
- `sandboxRoot` enforcement for file operations
- Workspace boundary checks
- Symlink resolution safety

# Agent Go Reimplementation Plan (Resume-Oriented)

## Goal

Reimplement a simplified but high-signal version of the Loom agent in Go while preserving the same
architecture highlights discussed earlier:

- Event-driven explicit state machine
- Trait/interface-based LLM abstraction
- SSE streaming normalization
- Tool registry with JSON schema
- Workspace security boundary and retry reliability

This plan is intentionally scoped so it can be completed in a few days and explained clearly in a
resume or interview.

---

## Scope and Non-Goals

### In Scope

1. CLI-based agent loop (REPL style)
2. Single provider integration (Anthropic first)
3. Streaming and non-streaming completion
4. Tool execution with 2-3 core tools (`read_file`, `list_files`, `edit_file`)
5. Path validation to prevent traversal outside workspace
6. Structured logging and bounded retry
7. MCP client-side server management (Claude Code-like) with dynamic remote tool import
8. ACP mode over stdio JSON-RPC for IDE-driven sessions

### Out of Scope (for v1)

1. Multi-provider production routing/proxy
2. Full auth/ABAC system
3. Kubernetes/weaver/deployment platform work
4. Large observability platform (crash analytics, product analytics UI)
5. Implementing our own MCP server endpoint (this design focuses on MCP client behavior)
6. ACP authentication methods and non-text multimodal content blocks

---

## Recommended Go Libraries

### Core Runtime

- `cobra` for CLI commands and entrypoint
- Go stdlib `context`, `goroutine`, `channel` for async orchestration

### HTTP and Reliability

- `go-resty/resty/v2` for provider HTTP calls
- `cenkalti/backoff/v4` for exponential backoff retries

### Serialization and Validation

- stdlib `encoding/json`
- `go-playground/validator/v10` for input argument validation
- `invopop/jsonschema` to generate tool JSON schema definitions

### Logging and Testing

- `uber-go/zap` for structured logs
- `stretchr/testify` for unit tests
- `leanovate/gopter` (optional) for property-based tests

### Utility

- `google/uuid` for call IDs and correlation IDs

### MCP Client Integration

- `go.bug.st/serial` (optional) only if a transport requires low-level process/pipe handling
- stdlib `os/exec`, `bufio`, `io`, `sync` for stdio-based MCP transport
- stdlib `net/http` (or `resty`) for streamable HTTP transport

---

## Architecture Mapping (Rust Design -> Go Design)

- `loom-core` -> `internal/core`
  - `LlmClient` interface
  - `AgentState`, `AgentEvent`, `AgentAction`
  - `HandleEvent(event) -> action`
- `loom-llm-anthropic` -> `internal/llm/anthropic`
  - `Complete()` and `CompleteStreaming()`
  - SSE parser and event normalization
- `loom-tools` -> `internal/tools`
  - `Tool` interface + `Registry`
  - `read_file`, `list_files`, `edit_file`
  - workspace root path guard
- `loom-cli` -> `cmd/agent` + `internal/runtime`
  - REPL, action dispatcher, event loop
- `mcp-system` (conceptual client role) -> `internal/mcp`
  - MCP server lifecycle manager (add/remove/list/reconnect)
  - JSON-RPC request/response client
  - transport adapters (stdio first, streamable HTTP optional)
  - remote tool -> local tool bridge with namespacing
- `acp-system` -> `internal/acp`
  - ACP agent implementation over stdio JSON-RPC
  - session lifecycle (`initialize`, `session/new`, `session/load`, `session/prompt`, `session/cancel`)
  - protocol bridge between ACP content blocks and internal message/tool events

Layer rule: lower-level packages do not depend on upper-level packages.

---

## v1 Repository Layout

```text
agent-go/
  cmd/agent/main.go
  internal/core/
    agent.go
    state.go
    event.go
    action.go
    llm.go
    error.go
  internal/llm/anthropic/
    client.go
    stream.go
    types.go
  internal/tools/
    tool.go
    registry.go
    context.go
    path_guard.go
    read_file.go
    list_files.go
    edit_file.go
  internal/runtime/
    loop.go
  internal/mcp/
    manager.go
    client.go
    registry_bridge.go
    types.go
    transport_stdio.go
    transport_http.go
    config.go
  internal/acp/
    agent.go
    session.go
    bridge.go
    notify.go
    error.go
    transport_stdio.go
  go.mod
```

---

## MCP Client Manager Design (Claude Code-like)

### Design Intent

This project adds MCP in the same role as Claude Code:

1. The Go agent acts as an MCP client
2. It manages connections to external MCP servers (for example, Figma MCP)
3. It imports remote MCP tools and exposes them to the LLM as callable tools
4. It forwards tool calls to the corresponding MCP server and returns normalized outputs

This is intentionally different from implementing an MCP server endpoint.

### Capabilities

1. Register multiple MCP servers from config
2. Connect and initialize each server on startup
3. List available remote tools and map them into local registry
4. Route `tools/call` requests to correct MCP backend
5. Health-check, retry, and reconnect failed MCP sessions
6. Enable/disable servers without changing core agent logic

### Configuration Model

Use a user-level configuration file (for example, `~/.agent/config.yaml`):

```yaml
mcpServers:
  figma:
    enabled: true
    transport: stdio
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-figma"]
    env:
      FIGMA_ACCESS_TOKEN: "${FIGMA_ACCESS_TOKEN}"
    startup_timeout_secs: 20
    call_timeout_secs: 30
  internaldocs:
    enabled: false
    transport: http
    url: "http://127.0.0.1:8765/mcp"
    headers:
      Authorization: "Bearer ${MCP_TOKEN}"
```

Server entry fields:

- `name` (map key)
- `enabled`
- `transport` (`stdio` or `http`)
- transport-specific connection fields
- per-server timeouts and retry policy
- optional allowlist for tools

### Tool Namespacing and Registry Bridge

Remote MCP tools must be namespaced to avoid collisions:

- Local native tool: `read_file`
- MCP imported tool: `mcp.figma.get_file`

Bridge behavior:

1. `tools/list` from each MCP server
2. Convert MCP input schema into local `ToolDefinition`
3. Prefix tool name with `mcp.<server>.`
4. Register as a delegating proxy tool

When invoked, proxy tool extracts server+tool name and dispatches `tools/call`.

### MCP Lifecycle Flow

1. Agent startup loads config
2. Manager starts all `enabled` servers
3. For each server: `initialize` -> `notifications/initialized` -> `tools/list`
4. Bridge registers remote tools into main ToolRegistry
5. Runtime loop can now call MCP tools exactly like native tools

### Runtime Call Flow

1. LLM emits tool call for `mcp.figma.some_tool`
2. ToolRegistry resolves to MCP proxy tool
3. MCP manager sends JSON-RPC `tools/call` to `figma` session
4. Result normalized into existing tool output format
5. Agent continues normal state transition

### Reliability and Recovery

1. Startup retry with bounded exponential backoff
2. Per-call timeout and cancellation via context
3. Automatic reconnect on broken transport
4. Circuit-breaker style temporary disable after repeated failures
5. Surface health in CLI via command (for example, `/mcp status`)

### Security Model

1. Env var interpolation occurs at runtime and values are redacted in logs
2. Optional command allowlist for stdio servers
3. Optional tool allowlist per server
4. No automatic trust of server-declared tool schemas beyond validation checks
5. Audit events for connect/disconnect/tool-call operations

### CLI UX (Claude Code-like)

Provide lightweight management commands:

- `/mcp list` - list configured servers and status
- `/mcp add <name> ...` - add a server config entry
- `/mcp remove <name>` - remove server config
- `/mcp enable <name>` / `/mcp disable <name>`
- `/mcp reconnect <name>`
- `/mcp tools [name]` - inspect imported tools

These commands manage client-side connections only; they do not require changes to core state
machine design.

### Minimal MCP Protocol Surface

For v1 MCP client support, implement only:

1. `initialize`
2. `notifications/initialized`
3. `tools/list`
4. `tools/call`

Skip resources/prompts/sampling and server-initiated notifications for the first iteration.

### Core Interfaces (Go)

```go
type MCPManager interface {
    StartAll(ctx context.Context) error
    StopAll(ctx context.Context) error
    ListServers() []ServerStatus
    CallTool(ctx context.Context, server string, tool string, args map[string]any) (ToolResult, error)
    ListImportedTools() []ImportedTool
}

type MCPTransport interface {
    Send(ctx context.Context, req JsonRpcRequest) (JsonRpcResponse, error)
    Close() error
}
```

### Testing Plan for MCP Module

1. Unit tests for JSON-RPC serialization/parsing
2. Unit tests for namespacing and registry bridge mapping
3. Unit tests for reconnect and timeout behavior with fake transport
4. Integration test with a mock stdio MCP server process
5. End-to-end test: imported MCP tool appears in tool list and executes successfully

---

## Local Credential Configuration and LLM Client Resolution

### Design Intent

`agent-go` is a single-process REPL/TUI application, not a split CLI/server deployment. The user
configures provider credentials locally from the TUI, typically via `/model` or an initial `llm
init` flow. The runtime then resolves the selected model profile into a concrete provider client,
while the agent layer continues to depend only on the `LlmClient` interface.

This keeps the same architectural boundary as Loom's server-side `LlmClient` design, but moves
credential management and provider client construction into the local application.

### User Experience in TUI

The primary entrypoint is `/model`:

1. choose provider (`openai`, `anthropic`, `zai`, future providers)
2. choose backend/profile type supported by that provider
3. configure or select a local credential
4. choose default model
5. persist the profile and make it active for the current session

Example `/model` interactions:

1. `OpenAI Platform`
2. `Auth Method: API Key`
3. `Credential: paste API key or select saved key`
4. `Default Model: gpt-4o`

The same flow should be available in a first-run `llm init` wizard for non-TUI onboarding.

### Core Architectural Rule

The agent loop must not know whether a request is powered by:

1. a raw API key
2. an OAuth token set
3. an externally-managed session token
4. an alternate backend adapter that translates requests before calling a provider

The agent only holds:

```go
type LlmClient interface {
    Complete(ctx context.Context, req LlmRequest) (LlmResponse, error)
    CompleteStreaming(ctx context.Context, req LlmRequest) (LlmStream, error)
}
```

Everything below that boundary is runtime configuration and provider adapter work.

### Local Configuration Model

Store two kinds of local state:

1. credentials
2. model profiles

Credential store example:

```json
{
  "openai-default-key": {
    "provider": "openai",
    "type": "api_key",
    "api_key": "sk-..."
  },
  "anthropic-max-1": {
    "provider": "anthropic",
    "type": "oauth",
    "access": "at_...",
    "refresh": "rt_...",
    "expires": 1735500000000
  }
}
```

Model profile example:

```json
{
  "default": {
    "provider": "openai",
    "backend": "openai_platform",
    "credential_id": "openai-default-key",
    "model": "gpt-4o"
  },
  "claude-max": {
    "provider": "anthropic",
    "backend": "anthropic_subscription",
    "credential_id": "anthropic-max-1",
    "model": "claude-opus-4-20250514"
  }
}
```

`/model` edits the profile layer, not the agent loop.

### Provider Capability Model

Providers should advertise supported local configuration modes instead of assuming OAuth:

```go
type AuthMethod string

const (
    AuthMethodAPIKey AuthMethod = "api_key"
    AuthMethodOAuth  AuthMethod = "oauth"
    AuthMethodSession AuthMethod = "session_token"
)

type ProviderCapability struct {
    Provider        string
    Backends        []string
    SupportedAuth   []AuthMethod
    SupportsPooling bool
}
```

Initial expected capabilities:

1. `openai`
   - backend: `openai_platform`
   - auth: `api_key`
2. `anthropic`
   - backend: `anthropic_api`
   - auth: `api_key`
   - optional backend: `anthropic_subscription`
   - auth: provider-specific OAuth/session flow
3. experimental adapters
   - can expose alternate backend names without changing the agent layer

This is the right extension point for any future OpenAI-compatible or Codex-style adapter. The
backend is what changes request construction and transport behavior; `LlmClient` does not.

### Resolution Sequence

```mermaid
sequenceDiagram
    participant U as User
    participant T as TUI (/model)
    participant P as Profile Store
    participant C as Credential Store
    participant R as Client Resolver
    participant B as Provider Backend
    participant L as LlmClient
    participant A as Agent Loop

    U->>T: /model
    T->>C: create/update credential
    T->>P: save active model profile
    U->>A: send prompt
    A->>R: resolve active profile
    R->>P: load profile
    R->>C: load credential by credential_id
    R->>B: build concrete provider client
    B-->>R: configured LlmClient
    R-->>A: LlmClient
    A->>L: Complete / CompleteStreaming
```

### Backend Adapter Pattern

Provider integrations should be split into three pieces:

1. credential parser/validator
2. backend adapter
3. optional auth lifecycle manager

Suggested interfaces:

```go
type ResolvedCredential struct {
    Provider string
    Type     AuthMethod
    Fields   map[string]string
}

type ProviderBackend interface {
    Name() string
    Provider() string
    BuildClient(ctx context.Context, cred ResolvedCredential, model string) (LlmClient, error)
}

type CredentialValidator interface {
    Validate(cred ResolvedCredential) error
}
```

Examples:

1. `openai_platform` backend
   - validates API key
   - creates the normal OpenAI client
2. `anthropic_api` backend
   - validates API key
   - creates normal Anthropic HTTP client
3. `anthropic_subscription` backend
   - reads token/session data
   - injects provider-specific headers or prompt constraints
   - refreshes or reconnects when the credential expires
4. future experimental backend
   - transforms OpenAI-style requests into another upstream protocol
   - still returns `LlmClient`

### Request Path After Profile Selection

1. agent runtime asks the resolver for the active profile's `LlmClient`
2. resolver loads the referenced credential
3. backend adapter validates credential material
4. backend adapter creates the concrete provider client
5. agent sends `complete` or `complete_streaming`
6. provider-specific auth/header/session behavior happens below `LlmClient`

This preserves dependency direction:

1. agent depends on `internal/core`
2. runtime depends on `internal/config` and `internal/llm/...`
3. provider packages never depend on the agent loop

### Local Pooling

Pooling should be designed as a profile/backend concern, not an agent concern.

If needed later, allow a profile to reference multiple credentials:

```json
{
  "openai-team": {
    "provider": "openai",
    "backend": "openai_platform",
    "credential_ids": ["openai-key-a", "openai-key-b"],
    "model": "gpt-4o"
  }
}
```

Activation behavior:

1. one credential -> single client mode
2. multiple credentials -> pooled client with retry/cooldown/failover
3. pooled client still implements `LlmClient`

This keeps the plan compatible with future pool management without centering the whole design on
OAuth.

### Security and UX Requirements

1. never log API keys, access tokens, refresh tokens, or session tokens
2. redact secrets in debug output and panic paths
3. store credentials in OS keychain when possible, file fallback otherwise
4. make `/model` idempotent: editing a profile should not require restarting the app
5. allow quick switching between saved profiles in the TUI
6. provide explicit validation feedback when a saved credential is missing or malformed
7. keep auth lifecycle code below the `LlmClient` boundary

---

## Prompt, Context, and Planning Architecture

### Design Intent

Loom's core design remains transcript-first: conversation messages are appended to a canonical
thread history and then projected into each `LlmRequest`.

For `agent-go`, use a single-model convergence path with three additions:

1. layered context projection
2. recoverable summary memory
3. budget-based selection with explicit drop reasons

This gives better quality than naive truncation while keeping v1 implementation bounded.

### Canonical Data vs Request-Time Projection

The core boundary is unchanged:

1. `thread` is the canonical persisted record of what actually happened
2. `request context` is a temporary projection built for a single LLM turn

This means:

1. raw user messages, assistant responses, and tool results are written to the thread transcript
2. the runtime may choose to send only a subset of that history to the LLM
3. summaries, plans, pinned instructions, and provider-specific prompt material are request-time
   inputs, not replacements for the canonical transcript

Do not overwrite the original thread transcript with trimmed or summarized context.

### Thread Scope Guarantee

To keep changes localized:

1. `thread` subsystem continues to handle full-fidelity persistence, resume, and rollback
2. summary and planning artifacts live outside canonical transcript storage
3. request-time context decisions are recomputed each turn from thread + sidecar memory

The thread design does not need a structural rewrite for this architecture.

### Memory Layers (Single-Model)

Use four layers when building each request:

1. `Canonical Transcript`
   - full persisted history in thread store
2. `Working Set`
   - messages guaranteed to be sent this turn (recent goal, recent tool chain, latest user turn)
3. `Summary Memory`
   - compressed history slices with source anchors back to transcript ranges
4. `Plan Snapshot`
   - compact execution intent (`goal`, `current_step`, `todo`, `open_questions`)

Only layers 2-4 are projected into `LlmRequest`; layer 1 remains the source of truth.

### Provider Boundary

Provider-specific prompt requirements stay below the generic prompt/context layer:

1. generic coding-agent policy belongs in prompt composition
2. provider request mapping belongs in provider backend
3. provider-specific mandatory prefixes or hacks belong in provider adapters

This keeps state machine and runtime provider-agnostic.

### Planning Consistency Rule

To avoid drift between plan and transcript:

1. plan snapshots are anchored to thread head (`thread_id`, `thread_version`, `last_message_id`)
2. plan updates use optimistic checks against current thread head
3. on mismatch, reload and rebase delta instead of blind overwrite

### Detailed Implementation Checklist

Implementation tasks, sequence diagrams, pseudocode, module split, and acceptance criteria are
documented in:

- [`docs/agent-go-context-plan-prompt-implementation-checklist.md`](./agent-go-context-plan-prompt-implementation-checklist.md)

This keeps the current document focused on architecture while moving implementation design to a
dedicated playbook.

---

## ACP IDE Integration Design

### Design Intent

ACP support allows the Go agent to run as an IDE-driven backend process, similar to editor
integration patterns used by Zed/VSCode ACP clients:

1. IDE acts as ACP client
2. `agent-go` runs `acp-agent` mode over stdio
3. IDE sends prompts/sessions via JSON-RPC
4. Agent streams updates (text chunks and tool lifecycle events) back to IDE

This keeps core agent logic shared between REPL mode and IDE mode.

### ACP Capabilities to Implement (v1)

1. `initialize`
2. `session/new`
3. `session/load`
4. `session/prompt`
5. `session/cancel`

Authentication can remain no-op in v1 if agent is launched locally by the IDE.

### ACP Session Model

Use the same mapping pattern as Loom:

- `SessionID` maps directly to persisted `ThreadID`

Per-session state should include:

1. `threadID`
2. `workspaceRoot`
3. `messages` (conversation context)
4. `cancelled` flag

Session lifecycle:

1. `session/new` creates thread and initial state
2. `session/load` restores thread and conversation history
3. `session/prompt` runs prompt loop and streams updates
4. `session/cancel` marks cancellation and aborts in-flight operations

### Prompt Loop in ACP Mode

The ACP prompt loop reuses existing runtime/state-machine behavior:

1. Convert ACP content blocks to internal user message
2. Call LLM in streaming mode
3. Stream text deltas as ACP `session/update`
4. Execute tool calls and stream tool start/finish notifications
5. Append tool results to conversation
6. Continue until end-turn/no-more-tools
7. Persist thread/session state
8. Return ACP prompt response with mapped stop reason

### Protocol Mapping

1. ACP `ContentBlock::Text` -> internal `Message::User`
2. Internal `TextDelta` -> ACP `AgentMessageChunk`
3. Internal tool start/finish -> ACP tool update notifications
4. Internal stop reason -> ACP stop reason:
   - normal completion -> `EndTurn`
   - cancelled -> `Cancelled`
   - processing failure -> `Error`

### ACP Transport and Runtime

Implement stdio JSON-RPC transport for `acp-agent` subcommand:

1. Start ACP connection over stdin/stdout
2. Run a notification sender task for asynchronous `session/update`
3. Keep session state in an in-memory map keyed by session ID
4. Persist threads after each completed turn

### CLI Integration

Add new command mode:

- `agent-go acp-agent`

Behavior:

1. Initialize LLM client, ToolRegistry, ThreadStore, MCP manager
2. Start ACP stdio loop
3. Process ACP session methods until shutdown

### ACP + MCP Integration

ACP and MCP connect at the tool layer:

1. On session startup, initialize MCP manager (global or per-session policy)
2. Import remote MCP tools into ToolRegistry as `mcp.<server>.<tool>`
3. During ACP prompt handling, tool calls can target local or MCP-backed tools transparently
4. IDE remains protocol-focused (ACP); MCP details are hidden behind tool invocation

This gives an IDE experience similar to Claude Code: editor drives conversation while external MCP
servers extend available capabilities.

### ACP Security and Reliability Notes

1. Enforce per-request timeout and cancellation checks in prompt/tool loops
2. Redact sensitive config/env values in logs
3. Validate ACP request payloads before conversion
4. Avoid holding mutable session borrows across blocking/async boundaries
5. Persist session after each completed turn to support crash recovery

### ACP Testing Plan

1. Unit tests for content-block conversion and stop-reason mapping
2. Unit tests for session ID <-> thread ID mapping
3. Integration test for `new_session -> prompt -> streaming updates -> prompt response`
4. Integration test for cancellation behavior
5. Integration test for ACP prompt using MCP-imported tool

---

## Thread Persistence Design (Loom-aligned)

### Design Intent

Adopt Loom's thread persistence philosophy directly:

1. Thread is the source-of-truth conversation document
2. Persist locally first on every completed inferencing turn
3. Sync remotely in best-effort background mode
4. Keep private threads strictly local

This design supports reliable resume, ACP session continuity, and optional multi-device sync.

### Thread Data Model

Use a full snapshot model close to Loom:

1. `id` (prefixed, time-sortable thread ID)
2. `version` (monotonic optimistic concurrency value)
3. timestamps (`created_at`, `updated_at`, `last_activity_at`)
4. execution context (`workspace_root`, `cwd`, `agent_version`)
5. provider context (`provider`, `model`)
6. `conversation.messages`
7. `agent_state` snapshot
8. privacy/sharing flags (`visibility`, `is_private`, `is_shared_with_support`)
9. metadata (`title`, `tags`, `is_pinned`, `extra`)

Suggested ID format:

- `T-<uuid7>`

This preserves chronological sorting and global uniqueness.

### Persistence Triggers

Persist thread at two exact points:

1. **Turn Complete**: when runtime returns to waiting-for-input after user prompt + LLM + tool
   execution chain
2. **Graceful Shutdown**: on explicit exit, EOF, or signal-driven shutdown path

On each persist:

1. update thread from latest runtime state
2. increment `version`
3. refresh `updated_at` and `last_activity_at`
4. save via `ThreadStore`

### Local Storage (Offline-First)

Store thread files under XDG-compatible paths:

- `$XDG_DATA_HOME/agent-go/threads/<thread_id>.json`
- pending sync queue: `$XDG_STATE_HOME/agent-go/sync/pending.json`

Local write requirements:

1. atomic write (`.tmp` then rename)
2. JSON serialization with strict schema compatibility
3. list ordered by `last_activity_at` descending

### Thread Store Layers

Use layered store abstraction:

1. `LocalThreadStore` (load/save/list/delete/search)
2. `SyncingThreadStore` (wraps local store + optional remote sync client)

Save behavior in `SyncingThreadStore`:

1. always save locally first
2. if `is_private == true`, skip sync
3. otherwise sync asynchronously in background
4. on sync failure, enqueue pending operation for retry

### Pending Sync Queue

Persist failed sync operations so they survive process restarts.

Entry fields:

1. `thread_id`
2. `operation` (`upsert` or `delete`)
3. `failed_at`
4. `retry_count`
5. `last_error`

Queue behavior:

1. deduplicate by `(thread_id, operation)`
2. increment retry count on repeated failures
3. remove entry on successful retry

### Remote Sync API Contract

Keep REST contract compatible with Loom-style thread sync:

1. `PUT /v1/threads/{id}` (upsert)
2. `GET /v1/threads/{id}`
3. `GET /v1/threads` (paged list)
4. `DELETE /v1/threads/{id}` (soft delete)

Concurrency:

- use `If-Match` with local `version`
- return conflict on stale write

v1 conflict handling can be conservative:

1. surface conflict in logs/status
2. keep local copy intact
3. optionally provide manual resolution command later

### Privacy and Visibility Rules

Hard invariant:

- if `thread.is_private == true`, never send this thread to remote server

This must be enforced in store logic (not only in CLI command validation).

Visibility semantics for synced threads:

1. `organization`
2. `private`
3. `public`

Support-share flag:

- `is_shared_with_support` independent from visibility

### CLI Commands for Threads

Expose thread management commands:

1. `agent-go` (new thread)
2. `agent-go list`
3. `agent-go resume [thread_id]`
4. `agent-go search <query>`
5. `agent-go private`
6. `agent-go share [thread_id] --visibility ...`
7. `agent-go share [thread_id] --support`

### ACP Integration with Threads

ACP session mapping should follow Loom pattern:

- `SessionID == ThreadID`

Effects:

1. `session/new` creates a thread-backed session
2. `session/load` restores thread and conversation state
3. `session/prompt` appends messages and persists at turn completion
4. `session/cancel` does not discard persisted history

### MCP Integration with Threads

Record MCP context in thread metadata for reproducibility:

1. configured MCP server names
2. optional tool namespace used in the turn
3. optional per-turn MCP call summary (non-secret)

Do not persist secrets/tokens in thread documents.

### Go Module Layout (Thread Subsystem)

```text
internal/thread/
  model.go
  store.go
  local_store.go
  sync_store.go
  sync_client.go
  pending_queue.go
  error.go
```

Core interfaces:

```go
type ThreadStore interface {
    Load(ctx context.Context, id ThreadID) (*Thread, error)
    Save(ctx context.Context, thread *Thread) error
    List(ctx context.Context, limit int) ([]ThreadSummary, error)
    Delete(ctx context.Context, id ThreadID) error
    Search(ctx context.Context, query string, limit int) ([]ThreadSummary, error)
}
```

### Testing Plan for Thread System

1. roundtrip serialization tests for `Thread`
2. thread ID format/property tests (`T-` prefix + UUID7)
3. local store save/load/list/delete tests
4. private thread never-sync invariant tests
5. pending queue dedup/retry tests
6. sync conflict handling tests
7. integration: runtime turn completion triggers save

---

## Core Functional Design

### 1. Explicit State Machine

Primary states (same intent as previous design):

1. `WaitingForUserInput`
2. `CallingLlm`
3. `ProcessingLlmResponse`
4. `ExecutingTools`
5. `Error`
6. `ShuttingDown`

Primary event/action contract:

- Input: `HandleEvent(event)`
- Output: `AgentAction`
- State machine itself does not perform I/O
- Runtime executes action, then feeds follow-up events back

Benefits:

- Deterministic and testable transitions
- Clear error and retry lifecycle
- Easier interview explanation (problem -> transition table -> invariants)

### 2. LLM Interface and Provider Strategy

Define one interface:

- `Complete(ctx, request) (response, error)`
- `CompleteStreaming(ctx, request) (<-chan LlmEvent, error)`

Start with Anthropic implementation only.

Design still supports future providers by implementing the same interface.

### 3. SSE Event Normalization

Normalize provider stream into one event union:

1. `TextDelta`
2. `ToolCallDelta`
3. `Completed`
4. `Error`

Tool call JSON fragments are accumulated in memory and parsed only when complete.

### 4. Tool System

Tool interface includes:

- `Name()`
- `Description()`
- `InputSchema()`
- `Invoke(ctx, args, toolCtx)`

Registry behavior:

- Register tool by name
- List definitions for model tool registration
- Dispatch invocation by tool name

v1 tools:

1. `read_file`
2. `list_files`
3. `edit_file`

### 5. Security Boundary

All file paths must be validated against `workspaceRoot`:

1. Resolve absolute path
2. Canonicalize path
3. Ensure canonical path has workspace canonical prefix
4. Reject otherwise

This prevents traversal and symlink escape attacks.

### 6. Reliability

- Bounded retries with exponential backoff for transient LLM/API errors
- Context-driven cancellation for long calls and stream operations
- Structured logging with request ID, state, and tool name fields

---

## Minimal Delivery Milestones

### Milestone 1 - Core and Contracts

- Build core types and transition skeleton
- Add transition table tests

### Milestone 2 - Anthropic Client

- Implement `Complete` and `CompleteStreaming`
- Normalize SSE events and verify accumulation behavior

### Milestone 3 - Tools and Runtime

- Implement registry and three tools
- Wire REPL runtime: user input -> state machine -> action -> event loop

### Milestone 4 - Hardening

- Add retry policies and path guard tests
- Add integration test with mocked LLM client

### Milestone 5 - MCP Client Manager

- Implement MCP config loading and manager lifecycle
- Support stdio transport and basic JSON-RPC methods
- Import remote tools with `mcp.<server>.` namespacing
- Add `/mcp` CLI management commands
- Add integration tests using mock MCP server

### Milestone 6 - ACP IDE Mode

- Add `acp-agent` CLI subcommand and stdio JSON-RPC transport
- Implement ACP session lifecycle handlers (`initialize/new/load/prompt/cancel`)
- Stream message/tool updates back to IDE clients
- Persist/reload session context through thread store
- Validate ACP + MCP tool routing in integration tests

### Milestone 7 - Thread Persistence and Sync

- Implement `internal/thread` model and store interfaces
- Add XDG-based local thread storage with atomic writes
- Trigger saves on turn completion and graceful shutdown
- Add optional sync client and pending queue retry path
- Add private-thread sync guard and visibility/share metadata handling

---

## Resume-Focused Highlights

Use wording similar to the following:

1. Designed an event-driven agent state machine in Go, separating deterministic transition logic
   from asynchronous I/O execution for improved testability and reliability.
2. Implemented Anthropic SSE streaming with normalized event types and incremental tool-argument
   reconstruction.
3. Built an interface-based LLM provider abstraction and dynamic tool registry with JSON schema
   contracts.
4. Enforced workspace filesystem boundaries and added retry/backoff handling for resilient tool and
   LLM orchestration.
5. Implemented a Claude Code-like MCP client manager that connects to external MCP servers and
   dynamically imports remote tools into the local agent tool registry.

---

## Interview Narrative (Quick)

1. Start with the problem: agent systems mix conversational state, streaming, and side effects.
2. Explain architecture choice: explicit state machine + action dispatcher.
3. Show reliability/security: backoff retries + workspace path guard.
4. Show extensibility: provider interface + tool registry.
5. Show ecosystem integration: MCP client manager with namespaced remote tools.
6. End with outcomes: simple codebase, clear tests, and easy feature growth.

---

## Suggested Timeline

- Day 1: core state machine and tests
- Day 2: Anthropic complete + streaming parser
- Day 3: tool registry + tools + runtime wiring
- Day 4: reliability hardening and integration tests
- Day 5: MCP manager, stdio transport, tool import bridge, and `/mcp` commands

This timeline keeps scope realistic while preserving architecture depth for resume and interviews.

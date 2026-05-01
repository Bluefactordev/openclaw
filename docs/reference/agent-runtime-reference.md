---
summary: "Code-grounded map of the embedded OpenClaw agent runtime"
read_when:
  - You need the real runtime call graph for agent runs, retries, transcript persistence, and recovery
  - You want an architecture reference before building OpenClaw-parity agent loops elsewhere
title: "OpenClaw Agent Runtime Reference"
---

# OpenClaw Agent Runtime Reference

This page is the code-grounded reference for OpenClaw's embedded agent runtime.
It is intentionally about the real execution path, not the product surface.

## Scope

The authoritative embedded runtime path is:

1. `src/agents/agent-command.ts`
2. `src/agents/pi-embedded.ts`
3. `src/agents/pi-embedded-runner.ts`
4. `src/agents/pi-embedded-runner/run.ts`
5. `src/agents/pi-embedded-runner/run/attempt.ts`
6. `src/agents/pi-embedded-runner/compact.ts`
7. `src/agents/pi-embedded-runner/runs.ts`
8. `src/agents/pi-embedded-runner/tool-result-truncation.ts`
9. `src/agents/session-write-lock.ts`
10. `src/agents/session-tool-result-guard-wrapper.ts`
11. `src/agents/session-transcript-repair.ts`
12. `src/agents/context-window-guard.ts`
13. `src/agents/failover-error.ts`
14. `src/agents/model-fallback.ts`
15. `src/agents/timeout.ts`
16. `src/agents/workspace.ts`
17. `src/config/sessions.ts`
18. `src/config/sessions/transcript.ts`
19. `src/infra/agent-events.ts`

Related overview docs:

- [/concepts/agent-loop](/concepts/agent-loop)
- [/reference/session-management-compaction](/reference/session-management-compaction)
- [/reference/transcript-hygiene](/reference/transcript-hygiene)

## Runtime call graph

At a high level, one embedded run looks like this:

1. `agentCommand()` in `src/agents/agent-command.ts` validates the request, resolves the session, chooses the workspace, computes timeouts, and picks the initial provider and model.
2. `runWithModelFallback()` in `src/agents/model-fallback.ts` wraps the actual run so the runtime can try alternate models when the failure is failover-worthy.
3. `runAgentAttempt()` in `src/agents/agent-command.ts` calls `runEmbeddedPiAgent()`.
4. `runEmbeddedPiAgent()` in `src/agents/pi-embedded-runner/run.ts` serializes work onto the session lane and global lane, resolves the effective workspace, resolves the model, applies the context window guard, orders auth profiles, and drives the retry loop.
5. `runEmbeddedAttempt()` in `src/agents/pi-embedded-runner/run/attempt.ts` does the actual turn: it opens the session transcript, installs transcript guards, builds the prompt and tools, streams model output, executes tools, waits for compaction to settle, and returns payloads plus run metadata.
6. `agentCommand()` persists session-store metadata, emits lifecycle events if needed, and hands the final payloads to delivery.

The important architectural point is that OpenClaw splits the run into a thin command entrypoint, an outer retry and failover loop, and an inner attempt that owns transcript integrity.

## 1. Command entrypoint and session preparation

`prepareAgentCommandExecution()` in `src/agents/agent-command.ts` is the main intake phase.
It does all of the following before the model is called:

- loads config and resolves command secret references
- resolves the target session via `resolveSession()`
- determines `sessionId`, `sessionKey`, session store path, and current `SessionEntry`
- resolves the agent workspace with `resolveAgentWorkspaceDir()` plus `ensureAgentWorkspace()`
- resolves the effective timeout with `resolveAgentTimeoutMs()` from `src/agents/timeout.ts`
- chooses the `runId`
- resolves ACP takeover state when applicable

Later in `agentCommandInternal()`, OpenClaw also:

- restores or persists per-session thinking and verbose defaults
- creates or refreshes a workspace skill snapshot
- resolves the transcript path through `resolveSessionTranscriptFile()` from `src/config/sessions/transcript.ts`
- registers the run context for streaming via `registerAgentRunContext()` from `src/infra/agent-events.ts`

## 2. Workspace resolution

Workspace handling is layered on purpose.

### Base workspace

`src/agents/workspace.ts` is responsible for the durable workspace root.
It provides:

- the default workspace location
- bootstrap file seeding (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, memory files)
- guarded reads inside the workspace boundary
- workspace setup state persistence under `.openclaw/workspace-state.json`

### Per-run workspace

`runEmbeddedPiAgent()` calls `resolveRunWorkspaceDir()` from `src/agents/workspace-run.ts` before doing real work.
That allows the runtime to:

- honor explicit workspace inheritance from spawned runs
- fall back safely when the caller workspace is unusable
- log workspace fallback without crashing the whole run

### Sandbox-adjusted workspace

`runEmbeddedAttempt()` then calls `resolveSandboxContext()` and may switch from the durable workspace to a sandbox workspace.
The model turn still keeps the original workspace identity, but tool execution may happen in the sandbox copy depending on the sandbox mode.

## 3. Session identity, transcript persistence, and intermediate state

OpenClaw uses two persistence layers.

### Session store

`src/config/sessions.ts` re-exports the session store helpers.
The mutable store is `sessions.json`, keyed by `sessionKey`.
It tracks run metadata such as:

- current `sessionId`
- model and provider overrides
- auth profile override
- token counters
- compaction count
- skills snapshot
- timestamps and routing metadata

After the run completes, `updateSessionStoreAfterAgentRun()` in `src/agents/command/session-store.ts` writes the latest model, token, and compaction metadata back into the store.

### Transcript file

`resolveSessionTranscriptFile()` in `src/config/sessions/transcript.ts` resolves the exact JSONL file for the session and persists that path into the store when needed.
`ensureSessionHeader()` creates the session header lazily.

The transcript is the durable source of conversation history.
`runEmbeddedAttempt()` opens it through `SessionManager.open()` and keeps using that live manager during the run.

### Intermediate state that is persisted during the run

OpenClaw persists more than final assistant text.
During a run it may write:

- user, assistant, tool-call, and tool-result transcript entries through `SessionManager`
- compaction entries through the context engine and `compactEmbeddedPiSession()`
- hidden yield context messages through `persistSessionsYieldContextMessage()` in `src/agents/pi-embedded-runner/run/attempt.ts`
- mirrored assistant transcript entries for ACP turns through `appendAssistantMessageToSessionTranscript()` in `src/config/sessions/transcript.ts`

This is why OpenClaw can recover from retries, compaction, and resume-like follow-ups without rebuilding the entire run from scratch.

## 4. Queueing, active run state, abort, and wait

`runEmbeddedPiAgent()` serializes work on two lanes:

- a per-session lane from `resolveSessionLane()`
- an optional global lane from `resolveGlobalLane()`

Both are driven through `enqueueCommandInLane()`.
That makes transcript writes deterministic and avoids cross-turn races.

`src/agents/pi-embedded-runner/runs.ts` keeps the active run registry.
It provides:

- `setActiveEmbeddedRun()` and `clearActiveEmbeddedRun()`
- `queueEmbeddedPiMessage()` for follow-up steering while a run is streaming
- `abortEmbeddedPiRun()`
- `isEmbeddedPiRunActive()` and `isEmbeddedPiRunStreaming()`
- `waitForEmbeddedPiRunEnd()` and `waitForActiveEmbeddedRuns()`
- in-memory snapshots of the active transcript leaf and in-flight prompt

The runtime therefore has an explicit run lifecycle instead of inferring activity from loose logs or client state.

## 5. Model selection, model fallback, and auth-profile fallback

OpenClaw handles model choice in layers.

### Initial choice

`agentCommandInternal()` resolves the initial model from agent defaults, stored session overrides, and explicit command overrides.

### Model fallback

`runWithModelFallback()` in `src/agents/model-fallback.ts` builds a candidate list and retries only when the error is classified as failover-worthy.
The fallback loop deliberately preserves structured reasons such as:

- auth
- billing
- rate limit
- overloaded
- timeout
- model not found

`FailoverError` in `src/agents/failover-error.ts` is the normalized envelope for that behavior.
It carries the reason, provider, model, profile, status, and cause.

### Auth-profile fallback

Inside `runEmbeddedPiAgent()`, OpenClaw separately rotates auth profiles before it gives up on the model.
The runtime:

- resolves ordered profile candidates with `resolveAuthProfileOrder()`
- honors explicit profile locks when the override came from the user
- skips or probes cooldowned profiles depending on policy
- marks profiles good, failed, or used
- refreshes runtime auth when the provider plugin exposes a short-lived exchanged credential

This is a real distinction in the runtime: profile failover happens inside the same model path, while model failover is the outer loop.

## 6. Context window guard and prompt-budget safety

Context-window enforcement happens before the attempt starts.

`resolveContextWindowInfo()` and `evaluateContextWindowGuard()` in `src/agents/context-window-guard.ts` combine:

- model-declared context window
- models-config overrides
- agent-level `contextTokens` caps
- repo defaults

The result can warn or block.
If the context window is too small, `runEmbeddedPiAgent()` throws before building the turn.
If it is merely low, OpenClaw keeps running but logs the warning.

`runEmbeddedAttempt()` also enforces bootstrap-prompt budgets and records prompt truncation warnings, so the runtime protects both the hard context window and the large static prompt layer.

## 7. Session write lock and transcript integrity

OpenClaw treats transcript corruption as a runtime problem, not a caller problem.

### Write lock

`acquireSessionWriteLock()` in `src/agents/session-write-lock.ts` is taken before the live session manager is used.
It includes:

- per-session lock files
- stale-lock detection
- watchdog cleanup for over-held locks
- timeout-derived max hold computation via `resolveSessionLockMaxHoldFromTimeout()`

### Tool-result guard wrapper

`guardSessionManager()` in `src/agents/session-tool-result-guard-wrapper.ts` wraps `SessionManager` so pending tool calls do not silently leave the transcript in an invalid state.
It also wires in persistence-time hooks and input provenance.

### Transcript repair

`src/agents/session-transcript-repair.ts` sanitizes malformed transcript state before it hurts later turns.
That includes:

- dropping invalid tool-call blocks
- normalizing tool names and ids
- inserting synthetic missing tool results when required
- stripping unsafe tool-result details
- preserving valid tool-use and tool-result pairing

`runEmbeddedAttempt()` uses these repairs both on session history and on provider stream output.

## 8. Attempt execution and event emission

The inner attempt in `src/agents/pi-embedded-runner/run/attempt.ts` is where the actual turn happens.
It is responsible for:

- opening and prewarming the session file
- repairing the session file if needed
- loading history and applying transcript policy validation
- building the system prompt and prompt report
- creating tools, tool allowlists, MCP and LSP runtime shims, and sandbox info
- subscribing to pi-agent-core streaming events
- registering the active run handle in `runs.ts`
- scheduling timeout aborts
- collecting assistant text, tool metadata, usage totals, and side effects

At the OpenClaw layer, streaming is surfaced through `emitAgentEvent()` in `src/infra/agent-events.ts`.
That emits strictly increasing per-run sequence numbers on streams such as:

- `lifecycle`
- `assistant`
- `tool`
- `error`

The run context registry lets the stream carry the right `sessionKey` and visibility flags without every inner callback having to recalculate them.

## 9. Compaction, tool-result truncation, and long-session safety

OpenClaw has several independent safety nets for long-running sessions.

### Compaction

`compactEmbeddedPiSession()` in `src/agents/pi-embedded-runner/compact.ts` owns explicit compaction behavior.
`runEmbeddedPiAgent()` also invokes overflow-recovery compaction through the context engine when a turn fails because the prompt is too large.

### Oversized tool-result recovery

`src/agents/pi-embedded-runner/tool-result-truncation.ts` handles the case where a single tool result is too large to fit even after compaction.
It can:

- estimate safe size as a share of the context window
- preserve both head and important tail content when truncating
- rewrite the transcript branch from before the first oversized tool result

This is a separate safety mechanism from compaction because the failure mode is different.

### Retry limits

The outer loop in `runEmbeddedPiAgent()` keeps a bounded retry counter for:

- auth-profile rotation
- context-overflow recovery
- tool-result truncation retry
- thinking-level fallback
- overload backoff

OpenClaw does not treat one failed turn as fatal to the session, but it also does not allow unbounded silent retry loops.

## 10. Yield and resume-like behavior

The embedded runtime has explicit yield semantics.

`runEmbeddedAttempt()` tracks the `sessions_yield` tool path with hidden custom transcript messages.
When yield is detected, it:

- aborts the live provider stream with a synthetic reason
- strips the synthetic abort artifacts back out of the visible transcript
- persists a hidden context message explaining why the previous turn ended intentionally

That means a later follow-up turn can resume with the correct state in the transcript rather than relying on client memory.

## 11. Failure handling and usage accounting

OpenClaw keeps failures structured all the way back to the caller.

### Failure envelope

`FailoverError` is the normalized envelope for retryable failures.
For non-failover failures, `runEmbeddedPiAgent()` still returns structured `meta.error` kinds such as:

- `retry_limit`
- `context_overflow`
- `compaction_failure`
- `role_ordering`
- `image_size`

Timeouts, auth errors, billing errors, and rate limits are all handled distinctly.

### Usage accounting

`runEmbeddedPiAgent()` keeps a `usageAccumulator` across retries and tool loops, but it separately preserves the last-call prompt usage so session-level context accounting does not become inflated by retry history.

The returned `agentMeta` includes:

- `provider`
- `model`
- normalized usage totals
- last-call usage snapshot
- prompt token estimate
- compaction count

That data is then written back into the session store for later status and diagnostics.

## 12. What another runtime has to match for real OpenClaw parity

If you want behavior that is genuinely comparable to OpenClaw, the runtime needs all of these concerns at the runtime layer, not only in the model prompt:

- stable session identity plus durable transcript persistence
- explicit run lifecycle with active, wait, abort, and queued states
- deterministic per-session serialization
- workspace resolution plus sandbox-aware execution
- explicit context-window guard before model execution
- persistent compaction, not only ephemeral summarization
- transcript repair and tool-result integrity guards
- separate auth-profile fallback and model fallback
- structured failure envelopes
- usage accounting that survives retries without inflating prompt-size reporting
- explicit yield and follow-up context persistence
- streamed runtime events that are tied to a concrete run id

Those pieces are why OpenClaw can keep long-running sessions alive without reducing the runtime to a single brittle loop.

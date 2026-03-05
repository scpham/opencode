# Implementation Plan: Isolated Multi-Session Support

## Problem Statement

OpenCode cannot run multiple sessions concurrently — even across different directories. The architecture assumes a single active session per process, with several process-wide shared resources that cause cross-session interference.

## Root Causes

| # | Issue | File(s) | Severity |
|---|-------|---------|----------|
| 1 | Worker event stream hardcoded to single directory | `cli/cmd/tui/worker.ts:97` | **Critical** |
| 2 | SSE `/event` endpoint scoped to one Instance | `server/server.ts:513` | **Critical** |
| 3 | `process.env` mutations overwrite auth tokens globally | `provider/provider.ts:235,440`, `config/config.ts:90` | **High** |
| 4 | `BusyError` blocks concurrent execution per-directory | `session/prompt.ts:86-88` | **High** |
| 5 | TUI single-session view with no background session awareness | `cli/cmd/tui/app.tsx`, `context/route.tsx` | **Medium** |
| 6 | `GlobalBus` events forwarded as wrong RPC event type | `cli/cmd/tui/worker.ts:36-38` | **Medium** |

---

## Phase 1: Fix Process-Wide State Leaks

**Goal:** Eliminate shared mutable state that corrupts concurrent sessions, even across different directories.

### 1.1 — Replace `process.env` mutations with context-carried config

**Files to modify:**
- `src/provider/provider.ts` (lines 231-239, 436-444)
- `src/config/config.ts` (line 90)
- `src/index.ts` (lines 82-84)
- `src/cli/cmd/acp.ts` (line 23)

**Approach:**
- Create a new `ProviderEnv` context that carries auth tokens per-Instance instead of writing to `process.env`
- For provider SDKs that read from `process.env` (AWS Bedrock, SAP AI Core), pass credentials via SDK constructor options instead of env vars
- For `config.ts` line 90 (`process.env[value.key] = value.token`), store tokens in `Instance.state()` and inject them into tool/provider calls via context
- The one-time process identity vars (`AGENT`, `OPENCODE`, `OPENCODE_PID`) in `index.ts` are safe — they're set once at startup and never change. Leave as-is.
- `OPENCODE_CLIENT` in `acp.ts` is also set once before any Instance exists. Leave as-is.

**Key constraint:** AWS SDK's `fromNodeProviderChain()` reads `process.env` internally. We need to either:
- (a) Pass `credentials` directly to the Bedrock provider, bypassing env-based resolution, OR
- (b) Use a scoped env object passed to `spawn()` calls only (already done in `bash.ts:174`)

**Testing:**
- Unit test: two providers with different auth tokens resolve correctly in parallel
- Integration test: run two sessions with different AWS profiles concurrently

### 1.2 — Scope shell environment per Instance

**Files to modify:**
- `src/tool/bash.ts` (line 174 — already passes `env` to spawn, verify it doesn't leak)
- `src/shell/shell.ts` — audit for process.env reads
- `src/pty/index.ts` (line 146)

**Approach:**
- `BashTool` already passes `{ ...process.env, ...shellEnv.env }` to spawn — this is fine for isolation
- Ensure `shellEnv` is built from Instance-scoped config, not process.env directly
- For PTY, same pattern: inject Instance-scoped env into spawn

**No-op if:** shell env is already properly scoped (it appears to be, but needs audit)

---

## Phase 2: Fix Event System for Multi-Directory Support

**Goal:** Events from any active Instance reach the TUI regardless of which directory it was started from.

### 2.1 — Worker: subscribe to GlobalBus instead of single-directory SSE

**Files to modify:**
- `src/cli/cmd/tui/worker.ts` (lines 42-97)

**Current behavior:**
```
worker.ts:97  → startEventStream(process.cwd())
              → subscribes to /event SSE for ONE directory
              → Rpc.emit("event", event)

worker.ts:36  → GlobalBus.on("event")
              → Rpc.emit("global.event", event)  // TUI ignores this!
```

**New behavior:**
- Remove `startEventStream()` entirely
- Use the existing `GlobalBus.on("event")` listener (line 36) but emit as `"event"` instead of `"global.event"`
- Extract `event.payload` before emitting so the TUI receives the same shape it expects
- This means the TUI receives events from ALL directories automatically

**Change:**
```ts
// worker.ts line 36-38, change from:
GlobalBus.on("event", (event) => {
  Rpc.emit("global.event", event)
})

// to:
GlobalBus.on("event", (event) => {
  Rpc.emit("event", event.payload)
})
```

Remove the `startEventStream` function and its invocation (lines 42-97).

### 2.2 — SSE `/event` endpoint: support multi-directory subscriptions

**Files to modify:**
- `src/server/server.ts` (lines 502-541)

**Current behavior:** `/event` calls `Bus.subscribeAll()` which is Instance-scoped — only gets events for the directory in the request header.

**New behavior for HTTP clients (web UI, external tools):**
- Add optional query param `?global=true` to `/event`
- When `global=true`, subscribe to `GlobalBus` instead of Instance-scoped `Bus`
- Default behavior unchanged (backwards compatible)

### 2.3 — TUI sync store: handle events from multiple directories

**Files to modify:**
- `src/cli/cmd/tui/context/sync.tsx`

**Current behavior:** The sync store processes all events as if they belong to the current directory. Session events are keyed by sessionID (not directory), so they already work cross-directory — but some events like `lsp.updated`, `vcs.branch.updated` are directory-specific.

**Approach:**
- For session-scoped events (`session.*`, `message.*`, `part.*`, `todo.*`, `permission.*`, `question.*`): no change needed — already keyed by sessionID
- For directory-scoped events (`lsp.updated`, `vcs.branch.updated`): filter to only apply events matching the current directory
- Add `directory` field to event payloads if not already present (it's on `GlobalBus` events but stripped in Phase 2.1)

**Revised approach for 2.1:** Emit both `directory` and `payload` so the TUI can filter:
```ts
GlobalBus.on("event", (event) => {
  Rpc.emit("event", { ...event.payload, _directory: event.directory })
})
```

The sync store can check `_directory` for directory-scoped events and ignore if mismatched.

---

## Phase 3: Allow Concurrent Session Execution

**Goal:** Multiple sessions can run simultaneously within the same directory.

### 3.1 — Remove single-session BusyError enforcement

**Files to modify:**
- `src/session/prompt.ts` (lines 65-89)

**Current behavior:** `state()` is a `Record<sessionID, { abort, callbacks }>` per-Instance. `assertNotBusy(sessionID)` throws if that sessionID already has an entry. But the real issue is that only ONE session per Instance can call `loop()` — subsequent calls queue via callbacks.

**New behavior:**
- Remove `assertNotBusy()` — allow multiple sessions to have active loops concurrently
- The `state()` record already supports multiple sessionIDs as keys — the data structure is fine
- The `start()` function (line 238) already checks per-sessionID, not globally — so concurrent sessions with *different* IDs already work at the data structure level
- The issue is really that callers (API routes) may call `assertNotBusy` before prompting. Remove those guard calls.

**Files to also check:**
- `src/server/routes/session.ts` — check if it calls `assertNotBusy` before dispatching prompts
- `src/session/processor.ts` — check for serial execution assumptions

**Risk:** Concurrent sessions in the same directory may produce file conflicts (two sessions editing the same file). This is an inherent user-responsibility issue, not something we need to solve architecturally. The snapshot/revert system can help recover.

### 3.2 — Add session concurrency limit (optional safeguard)

**Files to modify:**
- `src/config/config.ts` — add `maxConcurrentSessions` config option
- `src/session/prompt.ts` — enforce limit

**Approach:**
- Default limit: 4 concurrent sessions per Instance
- When limit reached, queue new session prompts (existing callback mechanism)
- Configurable via `opencode.json`

---

## Phase 4: TUI Multi-Session Awareness

**Goal:** The TUI shows status of background sessions and lets users manage them.

### 4.1 — Session status indicators in session list

**Files to modify:**
- `src/cli/cmd/tui/component/dialog-session-list.tsx`
- `src/cli/cmd/tui/context/sync.tsx` (already tracks `session_status`)

**Approach:**
- Show working/idle/error badge next to each session in the list
- `session_status` in the sync store already receives `session.status` events — just render them

### 4.2 — Background session notification toasts

**Files to modify:**
- `src/cli/cmd/tui/app.tsx`

**Approach:**
- When a background session completes (status changes from "working" to "idle"), show a toast notification
- Only for sessions that are NOT currently displayed
- Use existing toast infrastructure

### 4.3 — (Future) Split-pane or tab-based multi-session view

**Out of scope for initial implementation.** The TUI framework (opentui/solid) would need significant layout work. Defer to a later phase.

---

## Phase 5: SDK Client Multi-Directory Support

**Goal:** The SDK client and TUI can interact with multiple directories simultaneously.

### 5.1 — SDK: per-request directory override

**Files to modify:**
- `packages/sdk/js/src/v2/client.ts`

**Approach:**
- Already supported — the SDK sends `x-opencode-directory` header
- The server middleware at `server.ts:195-212` already wraps each request in `Instance.provide(directory)`
- No changes needed if the TUI sends the correct directory per API call

### 5.2 — TUI: track directory per session

**Files to modify:**
- `src/cli/cmd/tui/context/sync.tsx`

**Approach:**
- Sessions already have a `directory` field in the DB schema
- When making API calls for a specific session, send that session's directory in the `x-opencode-directory` header
- This ensures the server resolves the correct Instance for each request

---

## Implementation Order

```
Phase 1 (Process safety)     ← Do first, unblocks everything
  1.1 Provider env scoping
  1.2 Shell env audit

Phase 2 (Event plumbing)     ← Needed for any multi-session UX
  2.1 Worker GlobalBus fix
  2.2 SSE global mode
  2.3 Sync store filtering

Phase 3 (Concurrent execution) ← Core feature
  3.1 Remove BusyError
  3.2 Concurrency limit

Phase 4 (TUI polish)          ← User-facing improvements
  4.1 Status indicators
  4.2 Background notifications

Phase 5 (SDK/directory)       ← Mostly already works
  5.1 Verify SDK headers
  5.2 Per-session directory tracking
```

## Files Changed Summary

| File | Phase | Change Type |
|------|-------|-------------|
| `src/provider/provider.ts` | 1.1 | Pass credentials via SDK options, not process.env |
| `src/config/config.ts` | 1.1 | Store tokens in Instance.state, not process.env |
| `src/cli/cmd/tui/worker.ts` | 2.1 | Use GlobalBus → emit as "event" with directory |
| `src/server/server.ts` | 2.2 | Add `?global=true` SSE mode |
| `src/cli/cmd/tui/context/sync.tsx` | 2.3 | Filter directory-scoped events |
| `src/session/prompt.ts` | 3.1 | Remove assertNotBusy, allow concurrent loops |
| `src/cli/cmd/tui/component/dialog-session-list.tsx` | 4.1 | Show session status badges |
| `src/cli/cmd/tui/app.tsx` | 4.2 | Background session completion toasts |

## Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Concurrent file edits from parallel sessions | Snapshot/revert system; warn user |
| AWS SDK reading process.env internally | Pass explicit credentials to SDK constructor |
| Event flooding from many active sessions | Batch events in SDK context (already done at 16ms intervals) |
| SQLite contention under concurrent writes | WAL mode already enabled; busy_timeout=5000ms |
| Breaking existing single-session workflows | All changes are additive; default behavior preserved |

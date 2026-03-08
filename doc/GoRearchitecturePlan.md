# SeedSync Rearchitecture Plan (Go + Vite/React + gRPC-web + Protobuf)

## 1) Why re-architect

Current implementation risk points:

- **Fragile lftp status parsing**: parsing depends on many regex patterns tied to specific `lftp` output formatting and raises hard errors when unexpected lines appear. One parse error can break status reporting instead of degrading gracefully.
- **Weak runtime recovery around `lftp`**: the backend wraps a long-running `pexpect` process with prompt matching; timeouts or output drift can leave the process in a bad state with only delayed/pending error propagation.
- **Polling-based streaming**: web streaming loops over handlers and sleeps every 100ms, causing unnecessary CPU wakeups and coarse-grained event delivery.
- **Monolithic process shape**: control plane, transfer orchestration, state management, and web delivery are tightly coupled in one Python process, making reliability isolation difficult.
- **Legacy UI stack constraints**: Angular-era architecture and route-level pages are functional but poor for modern real-time UX patterns and composable state updates.

## 2) Target architecture (proposed)

A **single Go backend** with internal modules + **Vite/React frontend** + **protobuf contracts** + **gRPC-web for commands/query** + **SSE (or WS) for high-rate events**.

### 2.1 Component map

1. **API Gateway (Go)**
   - Exposes gRPC services (command/query domain APIs).
   - Exposes gRPC-web (envoy or grpcwebproxy in front, or Connect-compatible transport).
   - Exposes SSE endpoint for event streams (`/events`) and optional websocket endpoint for bidirectional UI sessions.

2. **Transfer Engine (Go)**
   - Job scheduler + worker pool.
   - Driver abstraction:
     - `LftpDriver` (initial compatibility)
     - future `SftpDriver`/`RcloneDriver` for reduced parser coupling.
   - Supervised process model (circuit breaker, restart policy, jittered retries).

3. **State Store (Go)**
   - In-memory read model (job/progress/session state).
   - Durable event + snapshot persistence (BoltDB/SQLite/Postgres depending on deployment).
   - Idempotent command handling.

4. **Event Bus (Go internal)**
   - Typed domain events (`JobQueued`, `TransferProgress`, `JobFailed`, etc.)
   - Fan-out to SSE/WS clients with per-client backpressure handling.

5. **React Frontend (Vite)**
   - gRPC-web clients generated from protobufs.
   - Real-time event reducer consuming SSE/WS.
   - Virtualized file list, optimistic actions, and resilient reconnection UX.

## 3) Protocol strategy

Use protobuf for all canonical domain models and RPC contracts.

- **gRPC(-web)** for:
  - configuration CRUD
  - queue operations (enqueue/cancel/retry)
  - file actions (extract/delete)
  - health/admin endpoints
- **SSE (default) or websocket (optional)** for:
  - high-frequency status updates
  - logs
  - notifications

Rationale:

- gRPC-web is ideal for typed request/response APIs.
- SSE is operationally simpler than WS through proxies for server→client streaming and fits status/log pipelines.
- WS can remain optional if future bidirectional control/remote terminal use-cases are needed.

## 4) Solving brittle parsing + lftp recovery

### 4.1 Replace regex-centric parser with a tolerant parser pipeline

Introduce parse stages:

1. **Framing**: split raw `lftp` output into command-response frames and async status chunks.
2. **Tokenization**: stable token extraction with minimal assumptions.
3. **Interpretation**: map tokens to domain states with confidence flags.
4. **Validation**: soft validation (warn + partial update), not fail-fast.

Rules:

- Unknown lines become `UnparsedLine` diagnostics; do not crash stream.
- Keep last-known-good job state and apply partial deltas.
- Emit parser health metrics (`parse_success_ratio`, `unknown_line_rate`).

### 4.2 Supervised process lifecycle

- Wrap each `lftp` session in a supervisor with explicit states:
  `Starting -> Ready -> Degraded -> Restarting -> Failed`.
- On timeout/prompt mismatch:
  - kill session,
  - recreate session,
  - reconcile queue from durable command log,
  - publish `SessionRestarted` event.
- Retry policy:
  - exponential backoff + jitter,
  - retry budgets per job,
  - classify errors (auth, network, remote path, quota).

### 4.3 Decouple job truth from process output

- `lftp` output should be treated as **telemetry**, not source-of-truth.
- Source-of-truth job state should come from command/event log transitions.
- Progress updates are eventually consistent overlays.

## 5) UI redesign (React + real-time)

### 5.1 UI architecture

- Vite + React + TypeScript.
- TanStack Query for command/query cache.
- Zustand/Redux Toolkit reducer for event stream state.
- Route layout:
  - Dashboard (live transfer KPIs)
  - Queue/Jobs (bulk actions + retry)
  - Files (filter/sort/virtualized tree)
  - Settings + Diagnostics (session health, parser anomalies)

### 5.2 Real-time transport

- Default SSE channel with event types and sequence IDs.
- Client auto-reconnect with `Last-Event-ID` for resumable stream.
- Backoff reconnect and stale-data banners.
- Optional WS path for advanced interactive features.

### 5.3 UX fixes

- Explicit transfer states: queued/running/retrying/degraded/failed/completed.
- Inline error reasons and actionable recovery buttons.
- Progressive disclosure for logs and diagnostics.
- Accessibility + keyboard shortcuts + responsive layout.

## 6) Protobuf package sketch

- `seedsync.v1.common.proto`
  - `JobId`, `FileRef`, `ErrorInfo`, `TransferStats`
- `seedsync.v1.jobs.proto`
  - `JobsService` (List, Get, Enqueue, Cancel, Retry)
- `seedsync.v1.config.proto`
  - `ConfigService` (Get/Update/TestConnection)
- `seedsync.v1.files.proto`
  - `FilesService` (List/Delete/Extract)
- `seedsync.v1.events.proto`
  - `EventEnvelope { sequence, timestamp, oneof event }`

Versioning rules:

- additive-only fields,
- reserve removed field numbers/names,
- explicit deprecation windows.

## 7) Additional major issues to address

1. **Security hardening**
   - secrets currently likely pass through process args/env and logs; move to secure secret provider + redaction.
   - adopt mTLS/internal auth and CSRF-safe browser flow.

2. **Observability gaps**
   - add OpenTelemetry traces, structured logs, Prometheus metrics, and per-job correlation IDs.

3. **Config drift + migrations**
   - formalize config schema versioning + migration engine.

4. **Concurrency correctness**
   - isolate mutable state behind actor/event-loop or lock-disciplined store.

5. **Test strategy modernization**
   - property tests for parser/tokenizer,
   - contract tests for protobuf APIs,
   - deterministic replay tests from captured `lftp` transcripts,
   - browser e2e for reconnect/resume behavior.

6. **Deployment topology**
   - one binary with embedded static assets for simple installs,
   - optional split mode (API + UI) for larger deployments.

## 8) Migration plan (phased)

### Phase 0: Contracts first

- Define protobuf schemas from current domain.
- Generate Go server + TS gRPC-web clients.
- Build compatibility mapping layer from existing Python semantics.

### Phase 1: New control plane in Go

- Implement config/job command APIs.
- Keep existing Python transfer path as temporary adapter (strangler pattern).

### Phase 2: New transfer engine

- Implement Go transfer supervisor + resilient `lftp` driver.
- Mirror existing job outcomes; run shadow mode for comparison.

### Phase 3: React UI rollout

- Ship React app reading from new APIs/events.
- Keep old UI behind feature flag until parity.

### Phase 4: Hardening + decommission

- Cut over default runtime.
- Remove legacy polling stream and Python control path.
- Keep transcript replayer and compatibility fixtures for regression testing.

## 9) Minimal MVP definition

- Reliable enqueue/cancel/retry commands via gRPC-web.
- Robust lftp session restart and queue reconciliation.
- SSE live dashboard with resumable stream.
- Parser anomaly reporting without crashing status updates.
- End-to-end tests proving recovery from forced lftp disconnect.

## 10) Recommended tech choices

- Go: `connect-go` or `grpc-go` + grpc-web proxy.
- Event streaming: SSE first, WS optional.
- Persistence: SQLite for local installs; Postgres option for scale.
- Frontend: Vite, React, TypeScript, TanStack Query, Zustand/RTK.
- Infra: OpenTelemetry + Prometheus + Grafana.

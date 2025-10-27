# Go-Based SeedSync Migration Repository Setup

This guide walks you through bootstrapping a brand-new repository for the Go-based rearchitecture that replaces the existing Python/LFTP stack while preserving performance and delivering typed gRPC APIs and a React frontend.

## 1. Create the repository scaffold
1. Initialize an empty git repository and root workspace:
   ```bash
   mkdir seedsync-go
   cd seedsync-go
   git init
   ```
2. Add a top-level README that captures the project vision (Go backend, gRPC, React) so future contributors understand the direction from the first commit.
3. Create the baseline directory layout:
   ```text
   .
   ├── bazel/                 # Bazel module files (bzlmod, toolchains)
   ├── docs/
   ├── proto/                 # Shared protobuf definitions (Buf-managed)
   ├── services/
   │   ├── gateway/           # Go API gateway (gRPC + REST)
   │   ├── transferd/         # Transfer orchestrator daemon (LFTP wrapper first)
   │   └── scanners/          # Local/remote scanner service(s)
   ├── webapp/                # React + Vite frontend
   └── tools/
       └── scripts/           # Helper scripts (lint, dev env)
   ```

## 2. Configure Bazel with bzlmod
1. Enable Bazel’s module system in `MODULE.bazel` at the repo root:
   ```starlark
   bazel_dep(name = "rules_go", version = "0.47.0")
   bazel_dep(name = "rules_proto", version = "5.3.0")
   bazel_dep(name = "rules_js", version = "0.12.0")

   go_sdk = use_extension("@rules_go//go:extensions.bzl", "go_sdk")
   go_sdk.download(version = "1.22.4")

   go = use_extension("@rules_go//go:extensions.bzl", "go")
   go.toolchain()

   use_repo(go, "go_sdk")
   ```
2. Add `WORKSPACE.bzlmod` (required placeholder when using bzlmod) and commit it empty.
3. Set up `bazel/.bazelrc` for consistent builds:
   ```
   build --enable_bzlmod
   test --test_output=errors
   ```

## 3. Initialize Go tooling under Bazel
1. Inside `services/transferd`, create `BUILD.bazel` with a `go_binary` target and enable `gazelle` to keep BUILD files in sync:
   ```starlark
   load("@io_bazel_rules_go//go:def.bzl", "go_binary")

   go_binary(
       name = "transferd",
       embed = ["//services/transferd/cmd:transferd"],
   )
   ```
2. Add `tools/gazelle/BUILD.bazel` using `@bazel_gazelle//:def.bzl` to auto-generate Go targets.
3. Run `bazel run //:gazelle -- update-repos -from_file=go.mod` once Go modules are in place to sync dependencies.

## 4. Set up Buf + protobuf toolchain
1. Create `proto/buf.yaml` for lint/build settings:
   ```yaml
   version: v1
   lint:
     use:
       - DEFAULT
   breaking:
     use:
       - FILE
   ```
2. Add `buf.gen.yaml` to emit Go, gRPC, Connect-Web, and OpenAPI artifacts:
   ```yaml
   version: v1
   plugins:
     - plugin: buf.build/protocolbuffers/go
       out: gen/go
     - plugin: buf.build/grpc/go
       out: gen/go
     - plugin: buf.build/connectrpc/go
       out: gen/go
     - plugin: buf.build/grpc/web
       out: gen/ts
     - plugin: buf.build/grpc-gateway/protoc-gen-openapiv2
       out: gen/openapi
   ```
3. Wire Buf into Bazel by adding a `proto/BUILD.bazel` with `proto_library` and `go_proto_library` targets, and a `buf_generate` rule (from `rules_buf`) to keep generated code hermetic.

## 5. Bootstrap Go modules
1. For each service (`gateway`, `transferd`, `scanners`), initialize a Go module referencing the shared root module:
   ```bash
   cd services/transferd
   go mod init github.com/your-org/seedsync/transferd
   ```
2. Reference the shared protobuf package (`github.com/your-org/seedsync/proto/gen/go/seedsync/v1`) and add Bazel targets pointing to the generated code.
3. Install dev tooling via `tools/go.work` or Bazel external repos for `golangci-lint`, `buf`, and `mockgen`.

## 6. Hardened LFTP supervisor (phase 1 engine)
1. Under `services/transferd`, add packages:
   ```text
   pkg/lftp/          # Supervisor wrapping the binary via os/exec
   pkg/jobs/          # In-memory job registry + persistence interface
   internal/grpc/     # Server wiring generated stubs to business logic
   cmd/transferd/     # main.go launching gRPC server
   ```
2. Define Bazel `go_library` targets for each package and a `go_binary` for the daemon.
3. Add integration tests using Bazel’s `go_test` to exercise LFTP invocations (guard real-binary tests behind a tag like `manual` for CI control).

## 7. API gateway & realtime layer
1. Mirror the structure for `services/gateway`:
   ```text
   pkg/server/
   pkg/streams/      # Fan-out to gRPC-Web/WebSocket
   internal/http/
   cmd/gateway/
   ```
2. Use `rules_proto_grpc` or Connect’s Bazel rules to generate gateway bindings and register them with Bazel targets consumed by the Go binary.
3. Add Bazel `webapp` targets (via `rules_js` + Vite) later to build the React bundle alongside Go binaries.

## 8. Frontend project (Vite + TypeScript)
1. Initialize the app under `webapp/`:
   ```bash
   npm create vite@latest webapp -- --template react-ts
   ```
2. Add `BUILD.bazel` using `rules_js` with a `js_binary`/`js_library` target that runs `npm install` and `npm run build`. Configure Bazel outputs to land in `webapp/dist` for the gateway to serve.
3. Generate TypeScript clients from `proto/gen/ts` and wire them into React Query for realtime stream subscriptions.

## 9. Continuous integration
1. Add a `.bazelci/presubmit.yml` (if using Buildkite/BazelCI) or GitHub Actions workflow running:
   ```bash
   bazel test //...
   bazel build //services/gateway/cmd:gateway
   bazel build //services/transferd/cmd:transferd
   ```
2. Include lint steps:
   ```bash
   buf lint
   golangci-lint run ./...
   npm run lint --prefix webapp
   ```
3. Cache Bazel, Go, npm, and Buf artifacts between runs for faster CI.

## 10. First commit checklist
- [ ] README describing the Go/gRPC/Bazel vision.
- [ ] `MODULE.bazel`, `.bazelrc`, and initial Bazel scaffolding.
- [ ] `proto/` with placeholder `health.proto` and generated stubs committed.
- [ ] Skeleton Go service (`cmd/transferd/main.go` with health check).
- [ ] Vite React scaffold (optionally empty until services are ready).
- [ ] CI workflow ensuring Bazel + Buf + Go builds pass.

Once this skeleton is in place, you can iterate through the earlier migration tasks—implementing the LFTP supervisor, building the realtime communication layer, and expanding the React UI—knowing Bazel, protobuf, and gRPC are wired up from the outset.

## 11. Migration strategy: greenfield vs. incremental

| Approach | Advantages | Risks / Trade-offs | When to choose |
| --- | --- | --- | --- |
| **Incremental migration** (reuse parts of the Python/LFTP stack while gradually porting services) | *Lower upfront cost:* you can peel off subsystems (e.g., transfer orchestration) while the legacy UI/backend continue serving users.<br>*Operational safety:* fallback to the existing Python pipeline if the Go service misbehaves.<br>*Real-world validation:* compare throughput, error rates, and DX improvements under production load before cutting over. | *Complexity drag:* you must maintain gRPC bridges/adapters between Python and Go during the transition.<br>*Longer timeline:* dual-running services until parity is reached.<br>*Bazel + Python integration:* more glue to share protobufs and config files between stacks. | You need uninterrupted service for existing users, have limited engineering capacity, or must prove Go/LFTP parity before committing fully. |
| **Greenfield rewrite** (stand up a fresh Go/Bazel repo and migrate data/users in one cutover) | *Clean architecture:* design services around protobuf contracts and Bazel from day one.<br>*Developer experience:* no legacy Python constraints—Go modules, static typing, Bazel hermetic builds.<br>*Faster iteration on new UX:* React app can target only the new APIs. | *Higher initial investment:* you must rebuild minimum viable features (queueing, scanning, config, UI) before shipping.<br>*Operational risk:* big-bang cutover requires exhaustive testing and migration scripts.<br>*Feature drift:* legacy bug fixes/enhancements may need to be duplicated during the rewrite. | You can run the legacy Python app in parallel until the Go version is ready, have resources to deliver a complete replacement, and want maximum freedom to change protocols/UX. |

**Recommendation:** Start with a *greenfield repo* (this document’s focus) so you can establish Bazel, protobuf, and Go-first patterns without legacy coupling. Once the skeleton services are testable, plan a *controlled switchover* that synchronizes data (configs, queues) and retires the Python stack. If you must keep parts of the old system running, design gRPC adapters early so you can dual-home traffic temporarily.

## 12. Program-level milestones & deliverables

Break the migration into explicit milestones that map to the previously defined tasks. Each milestone should culminate in demonstrable artifacts, automated tests, and retrospective checkpoints.

### Milestone 0 — Project bootstrap (Weeks 0–1)
- ✅ Repository created with Bazel/bzlmod, Buf, and Go toolchains wired in.
- ✅ Protobuf health/checkpoint service defined and generated for Go + TypeScript.
- ✅ CI pipeline (GitHub Actions or Buildkite) running `bazel test //...`, `buf lint`, and `golangci-lint` on every push.
- Exit criteria: `bazel build //services/transferd/cmd:transferd` and `bazel build //services/gateway/cmd:gateway` succeed with placeholder main packages; React scaffold builds via Bazel.

### Milestone 1 — Typed contracts & LFTP parity foundation (Weeks 2–4)

This milestone teaches the core stack—protobuf authoring, Bazel-driven codegen, Go service wiring, and observable LFTP control—while delivering a functioning transfer daemon that matches the legacy system’s throughput.

#### 1. Design and generate protobuf contracts
1. Sketch entity relationships on paper/whiteboard (Jobs, Transfers, ScannerEvents, Config). Mark required/optional fields and enumerations that mirror the legacy Python models.
2. Author `proto/seedsync/v1/transfer.proto`, `scanner.proto`, and `config.proto`, reusing shared messages in a `common.proto` file (e.g., `FileRef`, `Endpoint`, `Error`).
3. Run `buf lint` locally to catch style issues, then `bazel run //proto:buf.generate` (or the equivalent target) to emit Go + TypeScript stubs into `proto/gen/...`.
4. Inspect the generated Go packages to understand the struct shapes; write a short `examples/contracts/main.go` that marshals/unmarshals sample messages to cement familiarity.

#### 2. Stand up the LFTP-backed transfer daemon
1. Create the Bazel targets for `services/transferd/pkg/lftp`, `pkg/jobs`, and `cmd/transferd`. Use Gazelle to keep BUILD files aligned with package imports.
2. Implement `pkg/lftp/supervisor.go` that:
   - Reads job requests from an in-memory queue (initially a simple channel) and spawns `exec.CommandContext` processes for `lftp -e "pget ..."`.
   - Pipes stdout/stderr into a parser that tails `.lftp-pget-status` files for progress instead of prompt scraping.
   - Emits structured telemetry (job state transitions, byte counts, ETA) via Go channels for later fan-out.
3. Add configuration loading (`pkg/jobs/config.go`) that maps legacy knobs (parallel downloads, connection caps) to LFTP command flags. Store defaults in a Bazel-managed YAML file so tests can tweak them.
4. Expose a gRPC server in `internal/grpc/server.go` implementing `QueueJob`, `CancelJob`, and `WatchJobs` using the generated protobuf interfaces.

#### 3. Introduce an internal event broker and CLI smoke client
1. Pick a lightweight broker (start with in-memory pub/sub using Go channels; later swap for NATS/Redis). Build `pkg/events` to fan out job progress updates to multiple subscribers.
2. Create `tools/transferctl/` (Go module + Bazel `go_binary`) that exercises the gRPC API: queue a job, stream progress, cancel it. This is both a learning tool and an executable exit-criterion check.
3. Document broker assumptions and add metrics/log hooks (structured logging via `zap` or `zerolog`) so you can observe state transitions during manual runs.

#### 4. Establish automated validation and performance parity
1. Write unit tests with `go test`/Bazel for the protobuf mappers (e.g., ensure LFTP status file parsing produces the correct `JobProgress` message).
2. Create integration tests tagged `manual` that spin up a disposable LFTP-enabled container (Docker or Podman) and execute a real download to verify resume and throughput settings. Document how to run them locally.
3. Add a Bazel `sh_test` or `go_test` that benchmarks the supervisor against a fixture file, capturing MB/s metrics. Compare results with the legacy Python tool and log any deltas.
4. Update CI to run the hermetic unit tests and contract linting; gate manual integration tests behind a separate workflow you can trigger on demand.

**Exit criteria:** The CLI smoke tool can queue jobs through the Go gRPC API, stream realtime updates without polling, and measured throughput matches or exceeds the baseline LFTP performance captured from the Python stack.

### Milestone 2 — Realtime gateway & client integration (Weeks 4–6)
- Build the Go API gateway exposing gRPC, gRPC-Web, and REST (via grpc-gateway or Connect) with resumable stream support.
- Implement authentication/session middleware (JWT or static token) and enforce TLS termination.
- Generate TypeScript clients and integrate them into the React app with realtime hooks (Connect-Web/WebSocket wrapper, reconnection policies).
- Replace polling-based status refreshes with streamed updates driving progress bars and job lists.
- Exit criteria: Running `bazel run //services/gateway/cmd:gateway` plus `bazel run //webapp:devserver` yields a browser experience with live-updating transfers sourced entirely from the Go stack.

### Milestone 3 — Scanner, extractor, and config parity (Weeks 6–9)
- Port local and remote scanner logic to Go, matching path filters, ignore rules, and `.lftp-pget-status` semantics.
- Implement extractor/post-processing service (initially wrapping existing scripts; later reimplemented in Go) and expose status via protobuf events.
- Build configuration management APIs (CRUD + validation) and connect the React settings UI.
- Migrate persistence to the chosen datastore (SQLite/PostgreSQL) for job metadata and configuration snapshots.
- Exit criteria: React UI offers full parity for browsing queues, configuring servers, and monitoring scanners without falling back to Python components.

### Milestone 4 — Native Go transfer engine (Weeks 9–12)
- Develop the pure-Go SFTP/FTP transfer engine with multi-chunk downloads, resume support, and bandwidth controls.
- Feature flag between LFTP and the native engine for A/B benchmarking; collect throughput metrics under realistic workloads.
- Harden telemetry (metrics, tracing, structured logs) and expose them via Prometheus/OpenTelemetry endpoints.
- Update documentation for switching the default engine and retiring LFTP when ready.
- Exit criteria: Benchmarks demonstrate equal or better throughput compared to LFTP, and feature flag defaults to the native Go engine after validation.

### Milestone 5 — Cutover & decommission (Weeks 12–14)
- Prepare migration scripts to import existing configurations, logs, and job histories from the Python deployment.
- Run coordinated user acceptance testing on staging, covering queue management, realtime updates, scanner accuracy, and failure recovery.
- Roll out the Go stack to production with monitoring dashboards and rollback plans; freeze changes on the legacy repo.
- Archive or sunset the Python services, keeping only necessary compatibility shims for long-term maintenance.
- Exit criteria: All production traffic served by the Go/Bazel stack, legacy infrastructure decommissioned, post-mortem confirms objectives met.

### Ongoing maintenance tracks
- **DX & observability:** Expand linting (golangci, eslint), add pre-commit hooks, integrate OpenTelemetry tracing, and publish developer onboarding docs.
- **Performance tuning:** Periodically benchmark transfer throughput and latency, adjusting goroutine pools and broker settings.
- **Security hardening:** Implement role-based access, secret management, and secure defaults for cloud deployment (Kubernetes/containers).

## 13. Next steps checklist

- [ ] Decide on the migration mode (pure greenfield vs. limited dual-run) and document rollback paths.
- [ ] Stand up milestone tracking (e.g., GitHub Projects or Jira) with deliverables aligned to the schedule above.
- [ ] Allocate resources/owners per milestone and schedule regular demo checkpoints.
- [ ] Begin with Milestone 0 tasks to ensure Bazel, protobuf, and CI foundations are solid before tackling transfer orchestration.

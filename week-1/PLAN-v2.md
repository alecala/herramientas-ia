## Plan: V3 HTTP Microservice Migration

Convert the current CLI extractor into the V2 microservice defined in technical-brief-v2.md by reusing existing parsing/extraction components, introducing HTTP/job orchestration adapters, and enforcing deterministic processing, strict validation, and cleanup guarantees. Scope is planning only: no implementation in this step.

**Steps**
1. Baseline and gap map
- Review current flow in [cmd/extractor/main.go](cmd/extractor/main.go), [internal/service/extractor.go](internal/service/extractor.go), [internal/service/ports.go](internal/service/ports.go), [internal/csv/writer.go](internal/csv/writer.go), [internal/failures/logger.go](internal/failures/logger.go), [internal/parser/table.go](internal/parser/table.go), [internal/pdf/reader.go](internal/pdf/reader.go).
- Confirm behavior gaps versus V3: HTTP endpoints, request envelope rules, job lifecycle, startup validation, background cadence, timeout/shutdown cleanup, UUID correlation model, and FIFO job history.

2. Solidify domain contracts first (blocks all adapter work)
- Redefine service ports in [internal/service/ports.go](internal/service/ports.go) to typed method signatures (no placeholder interface{} args and no empty methods).
- Extend extraction orchestration in [internal/service/extractor.go](internal/service/extractor.go) with deterministic merge semantics: file snapshot sorted alphabetically, workers process concurrently, final write order stable by file-index and row-index.
- Add service-level job/domain types in a new internal service file so API and infrastructure share one lifecycle vocabulary.

3. Add runtime configuration model and strict startup validation
- Expand config package around [internal/config/regex.go](internal/config/regex.go) with env-backed runtime config loader/validator files.
- Validate at startup: INPUT_DIR exists, PORT valid, LOG_LEVEL valid, WORKERS_DEFAULT within [1,16], JOB_HISTORY_MAX >= 1, parseable durations.
- Implement precedence env over defaults exactly as brief requires.

4. Introduce infrastructure adapters for V3 runtime behavior (parallel after Step 2)
- Implement in-memory rate limiter (single active job fast-fail 409).
- Implement in-memory job tracker with states accepted -> running -> finished|failed and FIFO eviction for completed history only.
- Add temp artifact sweeper utility for startup and cancellation/failure paths.
- Add mutex-safe writer strategy to guarantee thread-safe writes and atomic finalization via temp then rename.

5. Build HTTP API adapter layer (depends on Steps 2-4)
- Create server composition root with route prefix handling from config.
- Implement POST read-all request validation: Content-Type application/json only, max body 2KB, strict boolean overwrite, workers bounds, path validation.
- Run synchronous pre-accept checks before creating job_id: no_input and duplicated_file.
- On accept, create UUIDv4 once and use it as both API job_id and logging correlation_id.
- Implement GET jobs status endpoint with expected terminal/non-terminal responses and not_found envelope.

6. Background orchestration and scheduler behavior
- Execute accepted jobs asynchronously with panic recovery and timeout via JOB_MAX_DURATION.
- Add 5-minute scanner ticker for INPUT_DIR that auto-enqueues only when files exist and no active job exists.
- Preserve same tracker/rate-limit path for auto and manual jobs.

7. Logging, envelopes, and failure semantics
- Migrate logging toward slog-based structured events with required keys correlation_id and layer.
- Ensure every non-2xx API response uses the standard envelope fields and codes.
- Keep failure log format contract, including row/file error levels, RFC3339 timestamp, and RawContent truncation to 200 chars.

8. Post-processing and artifact lifecycle guarantees
- Rename each successfully processed source file from PlanDePag*.pdf to Done-PlanDePag*.pdf.
- Treat rename failures as FILE_ERROR but keep already written rows and continue remaining files.
- Enforce cleanup matrix: success commit and remove leftovers, fail/panic/timeout/shutdown delete job temp artifacts, startup sweep stale temp artifacts.

9. Entrypoints and migration strategy
- Keep current CLI entrypoint for backward compatibility while adding a new HTTP command entrypoint.
- Move shared orchestration out of CLI-only assumptions to service/infrastructure composition.

10. Testing and CI gating updates
- Add unit, contract, race, stress, and integration coverage required by V3.
- Verify deterministic row ordering with multi-file fixtures under concurrency.
- Add failure-injection tests for timeout, panic, CSV write errors, rename failures, and cleanup behavior.
- Ensure CI includes build, vet, race tests, gosec, and internal coverage >=95%.

11. Documentation completion
- Update README with microservice endpoints, env vars, build targets, architecture diagram, regex modification guide, ledongthuc/pdf rationale, and gosec suppression policy.

**Relevant files**
- [technical-brief-v2.md](technical-brief-v2.md) — authoritative V3 acceptance criteria to map into implementation tasks.
- [cmd/extractor/main.go](cmd/extractor/main.go) — existing CLI orchestration to be decomposed and reused.
- [internal/service/extractor.go](internal/service/extractor.go) — service orchestration currently placeholder; central refactor anchor.
- [internal/service/ports.go](internal/service/ports.go) — current placeholders to replace with strict contracts.
- [internal/config/regex.go](internal/config/regex.go) — keep regex ownership centralized while adding runtime config files in same package.
- [internal/csv/writer.go](internal/csv/writer.go) — writer adapter to support temp-file atomic replace and compatibility with service contracts.
- [internal/failures/logger.go](internal/failures/logger.go) — failure log adapter to preserve V3 entry format and truncation rules.
- [internal/pdf/reader.go](internal/pdf/reader.go) — extraction adapter backing PDFExtractor port.
- [internal/parser/table.go](internal/parser/table.go) — row parsing and validation behavior reused by service.
- [README.md](README.md) — V3 docs and operational guidance.

**Files to create**
- internal/config/runtime.go — RuntimeConfig struct with PORT, PREFIX_PATH, INPUT_DIR, SHUTDOWN_TIMEOUT, CSV_DEFAULT_OUTPUT, JOB_MAX_DURATION, WORKERS_DEFAULT, LOG_LEVEL, JOB_HISTORY_MAX.
- internal/config/runtime_loader.go — env/default loader and normalization helpers.
- internal/config/runtime_validate.go — startup validation and actionable error aggregation.
- internal/service/jobs.go — shared job state model and status payload mapping.
- internal/service/errors.go — typed domain/application errors mapped to API envelope codes.
- internal/infrastructure/ratelimiter/inmemory.go — goroutine-safe single active job guard.
- internal/infrastructure/jobtracker/inmemory.go — job storage with FIFO eviction for completed jobs.
- internal/infrastructure/tempfiles/cleanup.go — startup and per-job tmp cleanup orchestration.
- internal/infrastructure/csv/atomic_writer.go — temp write then atomic rename helper.
- internal/api/server.go — HTTP server wiring, routes, middleware chain, graceful shutdown hooks.
- internal/api/handlers/read_all.go — POST read-all handler with synchronous checks and async launch.
- internal/api/handlers/job_status.go — GET job status handler.
- internal/api/middleware/content_type.go — enforce application/json on POST read-all.
- internal/api/middleware/body_limit.go — enforce 2KB request body cap.
- internal/api/middleware/correlation.go — correlation id lifecycle for non-job errors and request logs.
- internal/api/middleware/recover.go — panic recovery into internal error envelope.
- internal/api/envelope/errors.go — canonical non-2xx envelope builders.
- internal/api/scheduler/scanner.go — 5-minute background scanner and auto-enqueue trigger.
- cmd/server/main.go — HTTP service entrypoint and dependency composition.
- internal/infrastructure/ratelimiter/inmemory_test.go — concurrency and fast-fail behavior.
- internal/infrastructure/jobtracker/inmemory_test.go — state transitions and eviction coverage.
- internal/infrastructure/tempfiles/cleanup_test.go — cleanup matrix validation.
- internal/api/handlers/read_all_test.go — endpoint validation and acceptance/failure scenarios.
- internal/api/handlers/job_status_test.go — status responses and not_found handling.
- internal/api/scheduler/scanner_test.go — cadence and enqueue conditions.
- internal/service/extractor_v3_test.go — deterministic ordering, cancellation, and error path tests.

**Files to modify**
- [cmd/extractor/main.go](cmd/extractor/main.go) — reduce CLI-specific orchestration and delegate to service contracts, keeping backward compatibility while avoiding duplicated core logic.
- [internal/service/ports.go](internal/service/ports.go) — replace placeholder method signatures with compile-time-safe contracts used by both CLI and HTTP adapters.
- [internal/service/extractor.go](internal/service/extractor.go) — implement orchestration rules from V3: snapshot discovery, concurrent processing, deterministic merge/write order, context timeout awareness.
- [internal/csv/writer.go](internal/csv/writer.go) — adapt writer behavior for atomic overwrite strategy and compatibility with new service interfaces.
- [internal/failures/logger.go](internal/failures/logger.go) — ensure exact V3 failure line format, truncation, and safe concurrent writes under background jobs.
- [internal/model/types.go](internal/model/types.go) — add/adjust job status and envelope-friendly model types only if needed for shared contract consistency.
- [internal/parser/table.go](internal/parser/table.go) — confirm row-level failure taxonomy aligns with V3 reasons (numeric parse, date parse, empty estado, schema mismatch).
- [internal/parser/date.go](internal/parser/date.go) — align accepted date formats and canonical output if any V3 mismatch remains.
- [internal/parser/numeric.go](internal/parser/numeric.go) — confirm normalization and optional default semantics exactly match V3 numeric rules.
- [internal/pdf/reader.go](internal/pdf/reader.go) — ensure adapter contract compatibility with service PDFExtractor and proper error propagation for FILE_ERROR mapping.
- [README.md](README.md) — document V3 API contracts, architecture, build matrix, regex guidance, dependency rationale, and gosec policy.
- [go.mod](go.mod) — add only required dependencies for UUID generation, HTTP routing/middleware, and slog helpers if standard library alone is insufficient.

**Verification**
1. Build and static checks: go build ./..., go vet ./...
2. Concurrency safety: go test -race ./...
3. Security check: gosec ./... with justified suppressions file.
4. Coverage gate: internal packages >=95%.
5. Endpoint behavior tests: 202, 409 rate_limit, 409 duplicated_file, 404 no_input, 400 validation, 413 body too large, 415 wrong content-type, 404 not_found for status.
6. Deterministic ordering test with multiple PDFs and concurrent workers.
7. Cleanup matrix tests for success/failure/panic/timeout/shutdown/startup sweep.
8. Integration fixture run: output has exact header and >=72 rows, parseable with encoding/csv.

**Decisions**
- Include both CLI and HTTP entrypoints in V3 migration; shared service core prevents logic fork.
- Keep regex ownership centralized in config package; do not scatter inline regex.
- Use in-memory job/rate state only; no distributed lock or persistence, per V3 scope.

**Further Considerations**
1. Route implementation choice: standard net/http mux plus lightweight middleware pattern is recommended unless the team already standardizes another router.
2. Temp-file naming convention should include job_id to simplify scoped cleanup and observability.
3. If module path rename is pending, apply it before V3 implementation to avoid import churn during refactor.

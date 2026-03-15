# Plan: Add HTTP Microservice to PDF Payment Plan Extractor

**TL;DR**: Transform the existing CLI-only PDF extractor into a dual-interface application (CLI + HTTP API) using hexagonal architecture. Extract current business logic from `cmd/extractor/main.go` into a reusable `service` layer, create interface-based ports for all infrastructure dependencies, implement HTTP handlers with rate limiting and background job processing, and ensure both CLI and API share identical business logic with zero duplication.

**Current State**: Working CLI with concurrent PDF processing, thread-safe CSV/failure writers, one interface (`PDFTextReader`), business logic embedded in main.go.

**Target State**: Transform CLI into a Hexagonal architecture with API/Service/Infrastructure layers, only HTTP would be driving the service core, rate-limited background jobs, structured error envelopes, graceful shutdown.

---

## Steps — 36 Steps Organized in 11 Phases

### PHASE 1: Create Service Layer Foundation (Steps 1-3)
*Steps can run in parallel; no external dependencies*

1. **Create service layer interfaces/ports** — Define all port interfaces (PDFExtractor, CSVWriter, FailureLogger, RateLimiter, JobTracker) in `internal/service/ports.go` as pure contracts with zero implementation

2. **Extract business orchestration into service** — Move worker pool orchestration from CLI main.go into `internal/service/extractor.go` with `ExtractionService` struct accepting ports via dependency injection

3. **Create job lifecycle types** — Define `Job`, `JobState` enum (Accepted→Running→Finished/Failed), `JobResult` types in `internal/service/job.go` with correlation_id and timestamps

### PHASE 2: Infrastructure Adapters (Steps 4-7, depends on Phase 1)
*Steps 4-7 can run in parallel after Phase 1*

4. **Extract CSVWriter interface** — Refactor `internal/csv/writer.go` to implement `service.CSVWriter` port; maintain existing mutex protection; add interface-returning constructor

5. **Extract FailureLogger interface** — Refactor `internal/failures/logger.go` to implement `service.FailureLogger` port; keep thread-safety

6. **Create RateLimiter infrastructure** — New `internal/infrastructure/ratelimit/limiter.go` with atomic state checks; methods: `TryAcquire()` (bool + job_id), `Release(job_id)`

7. **Create JobTracker infrastructure** — New `internal/infrastructure/job/tracker.go` with map[string]*Job + RWMutex; methods: `Create()`, `Get()`, `UpdateState()`, `List()`

### PHASE 3: Configuration Layer (Steps 8-10, depends on Phase 1)

8. **Create centralized config structure** — New `internal/config/config.go` with master `Config` struct containing all env vars (PORT, PREFIX_PATH, ALLOWED_PDF_DIRS, etc.) plus CLI/JSON parameters

9. **Implement config loading with precedence** — New `internal/config/loader.go` with `Load()` function implementing ENV > YAML > defaults precedence

10. **Add config validation** — New `internal/config/validator.go` with `Validate()` function checking all constraints and returning aggregate errors

### PHASE 4: Refactor CLI (Steps 11-12, depends on Phases 1-3)

11. **Refactor cmd/extractor/main.go** — Remove all business logic; parse flags into Config struct; instantiate adapters; create ExtractionService; call single `service.Extract(ctx, params)` method

12. **Add CLI context with cancellation** — Wrap Extract() call with context.WithCancel; register SIGINT/SIGTERM handlers; cancel on signal

### PHASE 5: HTTP API Layer (Steps 13-19, depends on Phases 1-4)
*Steps 14-19 can run in parallel after Step 13*

13. **Create API foundation** — New `internal/api/server.go` with `Server` struct embedding http.Server; inject ExtractionService, RateLimiter, JobTracker; methods: `Start()`, `Shutdown(ctx)`

14. **Implement POST /read-all handler** — New `internal/api/handlers/extract.go` parsing JSON request, validating payload, checking rate limiter (409 if busy), generating UUIDv4, responding 202, launching background goroutine

15. **Implement GET /health handler** — New `internal/api/handlers/health.go` returning `{"status":"ok", "version":"...", "build_time":"..."}` always 200 OK

16. **Implement GET /ready handler** — New `internal/api/handlers/ready.go` checking jobTracker.HasActiveJob(); 200 if idle, 503 if busy

17. **Create error envelope middleware** — New `internal/api/middleware/errors.go` wrapping handlers to catch panics/errors and map to standardized HTTP responses with correlation_id

18. **Add request validation middleware** — New `internal/api/middleware/validation.go` checking Content-Type, enforcing 64KB body limit, validating paths against traversal and whitelist

19. **Add security headers middleware** — New `internal/api/middleware/security.go` setting X-Content-Type-Options, X-Frame-Options, X-Request-ID, X-Response-Time

### PHASE 6: Background Job Processing (Steps 20-21, depends on Phase 5 Steps 13-14)

20. **Implement background job runner** — New `internal/api/jobs/runner.go` with `RunExtraction()` function wrapping execution in defer+recover; setting job states; calling service.Extract(); handling success/error/panic

21. **Add context timeout per job** — Wrap job context with timeout from JOB_MAX_DURATION; service respects cancellation at worker pool; log partial state on timeout

### PHASE 7: Resource Cleanup (Steps 22-24, depends on Phase 6)

22. **Implement cleanup sweep** — New `internal/infrastructure/cleanup/sweep.go` with `SweepOrphans()` finding and deleting temp files matching `brief-tmp-*` pattern; run on startup

23. **Add cleanup to job lifecycle** — Modify `jobs/runner.go` to track temp resources and defer cleanup on all exit paths (success, error, panic, timeout)

24. **Implement graceful shutdown cleanup** — Modify `api/server.go` Shutdown() to stop accepting requests, wait for active job up to SHUTDOWN_TIMEOUT, cancel and cleanup if timeout

### PHASE 8: Observability (Steps 25-26, depends on Phase 5)
*Steps can run in parallel*

25. **Enhance structured logging** — Modify `internal/logger/logger.go` to accept context, extract correlation_id, add layer field; format: `[TIMESTAMP] [LEVEL] [LAYER] correlation_id=<uuid> message key=value`

26. **Add request logging middleware** — New `internal/api/middleware/logging.go` logging every request with method, path, status, duration, correlation_id

### PHASE 9: HTTP Server Entry Point (Steps 27-28, depends on Phases 5-8)

27. **Create HTTP server command** — New `cmd/server/main.go` loading config, initializing adapters, creating ExtractionService and API Server, starting server, handling graceful shutdown

28. **Add server startup validation** — In cmd/server/main.go validate config immediately, check env vars, run cleanup sweep before accepting traffic, log configuration

### PHASE 10: Testing (Steps 29-33, depends on all phases)
*Steps can run in parallel*

29. **Create service layer unit tests** — New `internal/service/extractor_test.go` mocking all ports, testing success/failure scenarios, worker pool concurrency, context cancellation; target ≥95% coverage

30. **Create infrastructure adapter tests** — New test files for ratelimit, job tracker, cleanup testing concurrent access and state transitions with `-race` flag

31. **Create API handler tests** — New test files for extract, health, ready handlers testing all HTTP response scenarios (202, 400, 409, 500) and error envelope format

32. **Create CLI vs API parity test** — New `test/integration/parity_test.go` running identical input through CLI and API, comparing CSV output byte-for-byte; assert ≥72 rows

33. **Create stress tests** — New `test/stress/concurrent_test.go` sending 10 concurrent POST requests, asserting exactly 1 gets 202 and 9 get 409

### PHASE 11: Documentation (Steps 34-36, depends on all phases)

34. **Update README with microservice section** — Add HTTP Microservice section documenting env vars, curl examples, error codes, build instructions for darwin/arm64 and linux/amd64

35. **Document PDF library justification** — Add Architecture Decisions section explaining why ledongthuc/pdf was chosen (text extraction, X-coordinates, maintenance, license, limitations)

36. **Create regex modification guide** — Add Configuration section documenting how to use cmd/inspect to find X-coordinates and adjust ColumnXRanges in `internal/config/regex.go`

---

## Relevant Files

### Files to CREATE (32 new files)

**Service Layer:**
- `internal/service/ports.go` — All port interfaces
- `internal/service/extractor.go` — ExtractionService with worker pool
- `internal/service/job.go` — Job lifecycle types
- `internal/service/extractor_test.go` — Service unit tests

**Infrastructure Layer:**
- `internal/infrastructure/ratelimit/limiter.go` — Single-job rate limiter
- `internal/infrastructure/ratelimit/limiter_test.go` — Rate limiter tests
- `internal/infrastructure/job/tracker.go` — In-memory job tracker
- `internal/infrastructure/job/tracker_test.go` — Job tracker tests
- `internal/infrastructure/cleanup/sweep.go` — Orphaned resource cleanup
- `internal/infrastructure/cleanup/sweep_test.go` — Cleanup tests

**Configuration:**
- `internal/config/config.go` — Master Config struct
- `internal/config/loader.go` — Config loading with precedence
- `internal/config/validator.go` — Config validation
- `internal/config/loader_test.go` — Loader tests
- `internal/config/validator_test.go` — Validator tests

**API Layer:**
- `internal/api/server.go` — HTTP server with routing
- `internal/api/handlers/extract.go` — POST /read-all endpoint
- `internal/api/handlers/health.go` — GET /health endpoint
- `internal/api/handlers/ready.go` — GET /ready endpoint
- `internal/api/handlers/extract_test.go` — Handler tests
- `internal/api/handlers/health_test.go` — Health tests
- `internal/api/handlers/ready_test.go` — Ready tests
- `internal/api/middleware/errors.go` — Error envelope
- `internal/api/middleware/validation.go` — Request validation
- `internal/api/middleware/security.go` — Security headers
- `internal/api/middleware/logging.go` — Request logging
- `internal/api/middleware/errors_test.go` — Error middleware tests
- `internal/api/middleware/validation_test.go` — Validation tests
- `internal/api/jobs/runner.go` — Background job execution

**Commands:**
- `cmd/server/main.go` — HTTP server entry point

**Testing:**
- `test/integration/parity_test.go` — CLI vs API integration test
- `test/stress/concurrent_test.go` — Concurrent request stress test

### Files to MODIFY (6 existing files)

- `cmd/extractor/main.go` — **Major refactor**: Remove all business logic (worker pool, processFile loop); parse flags into Config; instantiate adapters; create ExtractionService with DI; call single Extract() method; handle exit codes

- `internal/csv/writer.go` — **Minor**: Implement `service.CSVWriter` port interface; add interface-returning constructor; no logic changes; existing mutex protection unchanged

- `internal/failures/logger.go` — **Minor**: Implement `service.FailureLogger` port interface; add interface-returning constructor; existing mutex unchanged

- `internal/logger/logger.go` — **Moderate**: Add context-aware logging functions extracting correlation_id from context; add layer field to log format; maintain existing level system

- `internal/config/regex.go` — **Minor**: Move regex constants into Config struct as fields; keep existing compiled regex; add env-var-related constants

- `internal/pdf/reader.go` — **Minor**: Ensure LedongthucReader implements `service.PDFExtractor` port (rename interface if needed); no logic changes

---

## Detailed Logic per File

### Service Layer

**internal/service/ports.go**
Define interface contracts for all infrastructure dependencies:
- `PDFExtractor` interface with `ReadRows(path string) ([]Row, error)` method
- `CSVWriter` interface with `WriteHeader()`, `WriteRow(row PaymentRow)`, `Flush()`, `Close()` methods
- `FailureLogger` interface with `Log(failure FailureRecord, level string)`, `Close()` methods
- `RateLimiter` interface with `TryAcquire() (acquired bool, existingJobID string)`, `Release(jobID string)` methods
- `JobTracker` interface with `Create(jobID, state string)`, `Get(jobID)`, `UpdateState(jobID, state string)`, `HasActiveJob() bool` methods

**internal/service/extractor.go**
Business orchestration logic:
- `ExtractionService` struct holds references to all port interfaces (injected via constructor)
- `Extract(ctx context.Context, params ExtractionParams) (*ExtractionResult, error)` method:
  1. Glob PDF files matching pattern in params.InputDir
  2. Return error if no PDFs found (corresponds to exit code 2)
  3. Create buffered channel for file paths
  4. Spawn N worker goroutines (params.Workers count) each calling extractFile()
  5. Each worker: calls PDFExtractor.ReadRows(), parser.ParseRows(), writes valid rows to CSVWriter, logs failures to FailureLogger
  6. Use sync.WaitGroup to wait for all workers to complete
  7. Check context cancellation between operations
  8. Return ExtractionResult with counts: filesProcessed, rowsWritten, rowsFailed
  9. If all files failed, return error (exit code 3)

**internal/service/job.go**
Job lifecycle data structures:
- `JobState` type alias for string with constants: StateAccepted, StateRunning, StateFinished, StateFailed
- `Job` struct with fields: ID (string), CorrelationID (string), State (JobState), StartTime (time.Time), EndTime (*time.Time), Error (string), Result (*ExtractionResult)
- `ExtractionParams` struct with fields matching CLI/API schema: InputDir, OutputPath, FailuresPath, Overwrite, Workers
- `ExtractionResult` struct with fields: FilesProcessed, RowsWritten, RowsFailed

### Infrastructure Layer

**internal/infrastructure/ratelimit/limiter.go**
Single-job concurrency enforcer:
- `Limiter` struct with mutex and currentJobID field
- `TryAcquire()` method: lock mutex, check if currentJobID is empty; if yes set new jobID and return (true, ""); if no return (false, currentJobID)
- `Release(jobID)` method: lock mutex, verify jobID matches currentJobID, clear currentJobID field
- Thread-safe using mutex for all state access

**internal/infrastructure/job/tracker.go**
In-memory job state storage:
- `Tracker` struct with RWMutex and map[string]*Job
- `Create(job *Job)` method: write-lock, add to map
- `Get(jobID string) (*Job, error)` method: read-lock, return job or ErrNotFound
- `UpdateState(jobID, state string, optionalError string)` method: write-lock, find job, update State and EndTime if terminal
- `HasActiveJob() bool` method: read-lock, iterate map checking for StateRunning; return true if any found
- `List()` method: read-lock, return slice of all jobs (for observability)

**internal/infrastructure/cleanup/sweep.go**
Orphaned resource cleanup:
- `SweepOrphans(workDir string) (deleted int, err error)` function:
  1. Walk workDir looking for files/dirs matching pattern `brief-tmp-*`
  2. For each match, attempt os.RemoveAll()
  3. Count successful deletions, collect errors
  4. Log metrics at INFO level
  5. Return count and aggregate error

### Configuration Layer

**internal/config/config.go**
Master configuration structure:
- `Config` struct with fields:
  - Server: Port (int), PrefixPath (string)
  - Processing: AllowedPDFDirs ([]string), RateLimitTimeout (duration), ShutdownTimeout (duration), JobMaxDuration (duration), WorkersDefault (int)
  - Defaults: CSVDefaultOutput (string), FailuresDefaultOutput (string)
  - Logging: LogLevel (string)
  - Request: InputDir (string), OutputPath (string), FailuresPath (string), Overwrite (bool), Workers (int)
- Use struct tags: `env:"PORT"`, `json:"input_dir"`, `flag:"input-dir"`

**internal/config/loader.go**
Configuration precedence implementation:
- `Load(configFilePath string) (*Config, error)` function:
  1. Create Config struct with hardcoded defaults
  2. If configFilePath provided, parse YAML and overlay values
  3. Parse all environment variables and overlay values
  4. Return populated Config
- `LoadFromEnv(cfg *Config)` helper: iterate env vars, set matching struct fields using reflection or manual mapping
- `LoadFromYAML(path string, cfg *Config)` helper: read file, unmarshal YAML into Config struct

**internal/config/validator.go**
Configuration validation:
- `Validate(cfg *Config) error` function:
  1. Check Port is in range [1, 65535]
  2. Check LogLevel is one of: debug, info, warn, error
  3. Check Workers is in range [1, 32]
  4. If InputDir contains "..", return error (path traversal)
  5. If AllowedPDFDirs is set and InputDir not in whitelist, return error
  6. Check output file ends with .csv
  7. Collect all violations, return aggregate error with actionable messages

### API Layer

**internal/api/server.go**
HTTP server with routing:
- `Server` struct holds http.Server, ExtractionService, RateLimiter, JobTracker, Config
- `New()` constructor: inject dependencies, configure http.Server with timeouts
- `Start()` method:
  1. Register routes with PREFIX_PATH prefix
  2. Apply middleware chain: logging → security → validation → error handling → handler
  3. Call http.Server.ListenAndServe()
- `Shutdown(ctx context.Context)` method:
  1. Call http.Server.Shutdown() to stop accepting new requests
  2. Wait for active job using context with SHUTDOWN_TIMEOUT
  3. If timeout, cancel job context and force cleanup
  4. Return when all connections closed

**internal/api/handlers/extract.go**
POST /read-all implementation:
- Handler function signature: `func HandleExtract(svc *service.ExtractionService, limiter service.RateLimiter, tracker service.JobTracker) http.HandlerFunc`
- Logic:
  1. Parse JSON body into ExtractionParams struct
  2. Validate params using config.Validator
  3. If validation fails: return 400 with error envelope
  4. Call limiter.TryAcquire()
  5. If busy (false returned): return 409 with error envelope including hint
  6. Generate UUIDv4 as correlationID
  7. Create Job with StateAccepted, add to tracker
  8. Start background goroutine calling jobs.RunExtraction()
  9. Respond 202 with JSON: `{"correlation_id":"...", "status":"accepted", "message":"..."}`

**internal/api/handlers/health.go**
GET /health implementation:
- Always return 200 OK
- JSON body: `{"status":"ok", "version":"<git-sha>", "build_time":"<timestamp>"}`
- Version/build_time injected via ldflags during build

**internal/api/handlers/ready.go**
GET /ready implementation:
- Call tracker.HasActiveJob()
- If false (idle): return 200 with `{"status":"ready"}`
- If true (busy): return 503 with `{"status":"busy", "reason":"job_in_progress"}`

**internal/api/middleware/errors.go**
Error envelope wrapper:
- Wrap http.HandlerFunc with defer+recover
- Catch panics: log at ERROR with stack trace, return 500 with correlation_id
- Define custom error types: ValidationError, RateLimitError, AuthError
- Map error types to HTTP status codes and error codes:
  - ValidationError → 400, code="validation"
  - RateLimitError → 409, code="rate_limit", include hint field
  - AuthError → 401/403, code="auth"
  - Other → 500, code="internal"
- Extract correlation_id from context, include in all responses
- Standard response format: `{"code":"...", "message":"...", "correlation_id":"...", "hint":"..."}`

**internal/api/middleware/validation.go**
Request validation:
- Check Content-Type header is "application/json"
- Wrap request body with http.MaxBytesReader(64KB limit)
- If body too large: return 400
- Extract and validate all path fields from request params
- Check for ".." in paths (traversal detection)
- If AllowedPDFDirs configured, check InputDir against whitelist
- Store validated params in context for handler access

**internal/api/middleware/security.go**
Security headers:
- Set response headers:
  - X-Content-Type-Options: nosniff
  - X-Frame-Options: DENY
  - X-Request-ID: <correlation_id from context>
  - X-Response-Time: <duration since request start>
- Use negroni-like pattern: call next handler, then set headers

**internal/api/middleware/logging.go**
Request/response logging:
- Before handler: record start time, extract/generate correlation_id, add to context
- After handler: calculate duration, extract status code
- Log line format: `[TIMESTAMP] [LEVEL] method=GET path=/health status=200 duration=2ms correlation_id=... ip=... user_agent=...`
- Log level based on status: INFO for 2xx/3xx, WARN for 4xx, ERROR for 5xx
- Sanitize IP address (mask last octet for privacy)

**internal/api/jobs/runner.go**
Background job execution:
- `RunExtraction(ctx context.Context, jobID string, params ExtractionParams, svc *service.ExtractionService, tracker service.JobTracker, limiter service.RateLimiter)` function:
  1. Wrap context with timeout from JOB_MAX_DURATION
  2. Define cleanup function to release limiter and log completion
  3. Use defer to ensure cleanup always runs
  4. Use defer+recover to catch panics
  5. Update job state to Running via tracker
  6. Call svc.Extract(ctx, params)
  7. If success: update state to Finished with result
  8. If error: update state to Failed with error message
  9. If panic: recover, log at ERROR, update state to Failed
  10. In defer: call limiter.Release(jobID)

### Commands

**cmd/server/main.go**
HTTP server entry point:
- Main function logic:
  1. Read CONFIG_FILE env var (optional)
  2. Call config.Load(configFilePath)
  3. Call config.Validate(cfg)
  4. If validation fails: log errors at ERROR, exit code 1
  5. Call logger.SetLevel(cfg.LogLevel)
  6. Run cleanup.SweepOrphans(cfg.WorkingDir) — clean any orphaned files from crashed runs
  7. Instantiate infrastructure adapters: LedongthucReader, CSVWriter, FailureLogger, RateLimiter, JobTracker
  8. Create service.ExtractionService with injected ports
  9. Create api.Server with injected service, limiter, tracker, config
  10. Set up signal handler for SIGINT/SIGTERM
  11. Start server in goroutine
  12. Block on signal channel
  13. On signal: create context with SHUTDOWN_TIMEOUT, call server.Shutdown(ctx)
  14. Exit with code 0

**cmd/extractor/main.go (refactored)**
CLI entry point:
- Main function logic (refactored):
  1. Parse CLI flags using flag package
  2. Populate config.Config struct from flags
  3. Call config.Validate(cfg)
  4. If validation fails: print errors, exit code 1
  5. Call logger.SetLevel(cfg.LogLevel)
  6. Instantiate infrastructure adapters: LedongthucReader, CSVWriter, FailureLogger (no RateLimiter/JobTracker for CLI)
  7. Create service.ExtractionService with injected ports
  8. Create context with cancellation
  9. Set up signal handler for SIGINT/SIGTERM calling cancelFunc
  10. Call svc.Extract(ctx, params) — single method call
  11. Handle result:
      - If error contains "no PDFs found": exit code 2
      - If error contains "all files failed": exit code 3
      - If other error: log and exit code 1
      - If success: exit code 0
  12. Print summary: files processed, rows written, rows failed

### Modifications to Existing Files

**internal/csv/writer.go**
Changes:
- Add interface check: `var _ service.CSVWriter = (*CSVWriter)(nil)`
- Add constructor: `func NewCSVWriter(path string, overwrite bool) (service.CSVWriter, error)` returning interface type
- No changes to existing methods (already mutex-protected)

**internal/failures/logger.go**
Changes:
- Add interface check: `var _ service.FailureLogger = (*FailureLogger)(nil)`
- Add constructor: `func NewFailureLogger(path string) (service.FailureLogger, error)` returning interface type
- No changes to existing methods (already mutex-protected)

**internal/logger/logger.go**
Changes:
- Add context-aware functions:
  - `InfoCtx(ctx context.Context, msg string, keyvals ...interface{})`
  - `WarnCtx(ctx context.Context, msg string, keyvals ...interface{})`
  - `ErrorCtx(ctx context.Context, msg string, keyvals ...interface{})`
  - `DebugCtx(ctx context.Context, msg string, keyvals ...interface{})`
- Each extracts correlation_id from context using context.Value()
- Update emit() to include layer field: format becomes `[LEVEL] [layer] correlation_id=... message key=value`
- Maintain existing non-context functions for backward compatibility

**internal/config/regex.go**
Changes:
- Keep all existing regex constants and compiled patterns
- Move constants into Config struct as exported fields (for runtime modification if needed in future)
- Add new constants for env var names: EnvPort, EnvPrefixPath, etc.

**internal/pdf/reader.go**
Changes:
- Ensure PDFTextReader interface matches service.PDFExtractor port name (rename if needed)
- Add interface check: `var _ service.PDFExtractor = (*LedongthucReader)(nil)`
- No logic changes to ReadRows() implementation

---

## Verification

**Automated Tests:**
1. `go test -race ./...` — must pass with zero data races
2. `go test -coverprofile=coverage.out ./internal/...` — coverage ≥95% on internal packages
3. Integration test — CLI and API must produce byte-identical CSV output
4. Stress test — verify rate limiter with 10 concurrent requests

**Manual Tests:**
1. Build binaries: `go build -o extractor ./cmd/extractor` and `go build -o server ./cmd/server`
2. CLI: `./extractor --input-dir testdata/fixtures --output test.csv` — verify ≥72 rows
3. Server: `PORT=8080 ./server` — verify starts clean
4. Health: `curl http://localhost:8080/api/pdf-converter/health` — verify 200 OK
5. Readiness (idle): `curl http://localhost:8080/api/pdf-converter/ready` — verify 200
6. Submit job: `curl -X POST http://localhost:8080/api/pdf-converter/read-all -d '{"input_dir":"testdata/fixtures"}' -H "Content-Type: application/json"` — verify 202
7. Readiness (busy): Retry immediately — verify 503
8. Concurrent: Send 5 simultaneous POST — verify 1×202, 4×409
9. Shutdown: SIGTERM during job — verify graceful completion
10. Parity: Compare CLI vs API CSV outputs — must be identical

**CI Pipeline:**
```bash
go build ./...
go vet ./...
go test -race ./...
gosec ./...
go test -coverprofile=coverage.out ./internal/...
go tool cover -func=coverage.out | grep total | awk '{print $3}' # ≥95%
```

---

## Decisions

**Architecture:**
- Hexagonal architecture with service layer defining ports; API and CLI as adapters
- Single job concurrency via RateLimiter (simplifies v1 state management)
- In-memory job state (no persistence across restarts in v1)
- Background processing with 202 response, panic recovery, cleanup guarantees
- CLI backward compatibility (exit codes, flags, output unchanged)

**Technology:**
- stdlib `net/http` with custom middleware (no framework)
- Manual env var + optional YAML config (no external config lib)
- Extend custom logger with context support (defer slog to v2)
- `github.com/google/uuid` for correlation_id (UUIDv4)

**Scope Included:**
- Both CLI and HTTP API with shared business logic
- Rate-limited single-job API processing
- Graceful shutdown with cleanup
- Structured logging with correlation IDs
- Standardized error envelopes with retry guidance

**Scope Excluded (v1):**
- Bearer token auth (stub only)
- Multiple concurrent jobs or queuing
- Job persistence across restarts
- Polling endpoint for job status
- Recursive directory search
- Webhook notifications
- Distributed locking for replicas

---

## Next Steps

The plan is complete and ready for implementation. All architectural exploration is done, no blocking ambiguities remain.

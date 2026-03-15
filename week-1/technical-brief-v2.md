# Technical Brief: PDF Payment Plan Extractor → Microservice

**Version:** 2.0  
**Language:** Go 1.24 | **Targets:** `darwin/arm64` (primary), `linux/amd64` (CI) | **Architecture:** Hexagonal (Ports & Adapters)

---

## 1. Context & Goal

### What exists today

A CLI application that:

1. Scans a configurable directory for files matching the pattern `PlanDePag*.pdf`.
2. Parses each PDF to extract a structured payment plan table — each file represents a personal credit plan of **up to 72 installments** (`cuotas`), generated at different cut-off dates.
3. Consolidates all extracted rows into a **single CSV file**, ordered alphabetically by filename then by row position within each file.
4. Records every failed row or failed file in a structured **failures log**.

Read `technical-brief-v1.md` for the full original specification of the CLI tool.

### What we are building

An HTTP microservice that exposes the same extraction flow via REST endpoints, Transforming CLI into a Hexagonal architecture with API/Service/Infrastructure layers. Where only HTTP would be driving the service core, rate-limited background jobs, structured error envelopes, graceful shutdown.

> **Key Principle:** Only the API would expose the business logic. The microservice is an HTTP adapter, not a rewrite.

---

## 2. Architecture — Hexagonal (Ports & Adapters)

| Package | Responsibility | Allowed Dependencies |
|---|---|---|
| `api` | HTTP handlers, auth middleware, request validation, rate limiting, background job orchestration, error envelope mapping | `service` via interfaces only |
| `service` | Business use cases: scan for PDFs, parse tables, transform rows, orchestrate CSV persistence. **Zero framework dependencies.** | `infrastructure` via interfaces/ports only |
| `infrastructure` | Concrete adapters: CSV writer (mutex-protected), PDF extractor, rate limiter, job tracker. Each implements a port defined in `service`. | stdlib + approved third-party libs |
| `config` | Central config struct, env var + file parsing, startup validation, precedence logic. | stdlib only |

### Interface-Driven Contracts (Ports)

All cross-layer dependencies must go through interfaces. Concrete types must never be imported across layer boundaries. Minimum required ports:

- `PDFExtractor` — implemented by `infrastructure`; used by `service`. Enables mock-based unit testing without real PDF files.
- `CSVWriter` — implemented by `infrastructure`; used by `service`. Must be mutex-protected for safe concurrent use by CLI and API.
- `RateLimiter` — implemented by `infrastructure`; used by `api`.
- `JobTracker` — implemented by `infrastructure`; used by `api` and `service`.

---

## 3. Input: PDF File Discovery

- **Pattern:** Only files matching `PlanDePag*.pdf` are processed. All others are silently ignored.
- **Source directory:** Configurable via environment `INPUT_DIR`. Default: current working directory (`.`).
- **Ordering:** Files are processed in **alphabetical order by filename**. Row order in the CSV follows this file order, then row position within each file.
- **Atomicity:** File processes should the rename to `Done-PlanDePag*.pdf`
- **Upload:** Each 5 min, a job should scan for new files.

---

## 4. Parameter Schema (Authoritative Contract)

This table is the single source of truth the `POST /read-all` JSON request body.

| Field | JSON Key | Type | Required | Default | Constraints |
|---|---|---|---|---|---|
| Output CSV path | `output` | string | No | `"output.csv"` | Must end in `.csv`; parent directory must exist |
| Failures log path | `failures` | string | No | `"failures.log"` | Parent directory must exist |
| Overwrite if exists | `overwrite` | boolean | No | `false` | If `false` and output file exists → abort with exit code `1` |
| Worker goroutines | `workers` | integer | No | `NumCPU` | Min: `1`, Max: `16` |
| Log level | — | string | No | `"info"` | Enum: `debug`, `info`, `warn`, `error`. CLI only; API uses `LOG_LEVEL` env var. |

---

## 5. Concurrency: PDF Processing

- Multiple PDFs are processed in **parallel** using a worker pool of `N` goroutines, where `N` is defined in a ENV var (`WORKERS_DEFAULT`), but could be overwritten by the workers parameter in the post endpoint.
- The CSV writer must be **mutex-protected** in the infrastructure adapter to keep writes atomic and safe for concurrent use.
- The worker pool is managed within the `service` layer; the infrastructure `CSVWriter` adapter handles thread safety transparently.

---

## 6. Output: CSV Specification

### Column Schema

All columns must appear in this **exact order**:

| Column | Type | Rules |
|---|---|---|
| `Filename` | string | PDF filename only — no path. RFC 4180 quoting if value contains a comma or double quote. |
| `Fecha Fin` | string | Normalized to `YYYY-MM-DD`. Source may be `DD/MM/YYYY` or `DD-MM-YYYY`. |
| `Capital` | decimal | Required. See numeric format rules below. |
| `Interes Corriente` | decimal | Required. See numeric format rules below. |
| `Interes Mora` | decimal | Default `0.00` if absent or empty in the PDF. |
| `Seguros/FNG` | decimal | Required. See numeric format rules below. |
| `Otros Conceptos` | decimal | Default `0.00` if absent or empty in the PDF. |
| `Total Exigible` | decimal | Required. See numeric format rules below. |
| `Total Pagado` | decimal | Required. See numeric format rules below. |
| `Estado` | string | Raw value as it appears in the PDF. Expected values: `Pagada`, `Pendiente`, `Vencida`, `Condonada`. If an unexpected value is found, it must be accepted and **documented in the README**. |
| `Pendiente por Pagar` | decimal | Required. See numeric format rules below. |
| `Condonado` | decimal | See numeric format rules below. |
| `Saldo de Capital` | decimal | Required. See numeric format rules below. |

### Numeric Format Rules

All decimal values in the CSV must follow these transformation rules:

| Rule | Example input (from PDF) | Example output (CSV) |
|---|---|---|
| Remove currency symbol | `$ 1.234,56` | `1234.56` |
| Remove thousands separator | `1.234.567` | `1234567` |
| Replace comma decimal separator with dot | `1.234,56` | `1234.56` |
| Preserve negative sign | `-$ 150,00` | `-150.00` |
| Always 2 decimal places | `100` | `100.00` |

Format string: `%.2f`

### Date Format Rules

- Accepted source formats: `DD/MM/YYYY` and `DD-MM-YYYY`.
- Output format: `YYYY-MM-DD`.
- If the date does not match either pattern → the row is a **failure** (see Section 8).

### File Encoding & Format

- Encoding: **UTF-8 without BOM**.
- Delimiter: **comma** (`,`).
- Fields containing commas or double quotes: **wrapped in double quotes** per RFC 4180.
- Overwrite behavior: controlled by the `overwrite` parameter. If `false` and the output file already exists → abort immediately with exit code `1`.

---

## 7. PDF Library

Use `ledongthuc/pdfreader`. **The chosen library must be justified in the README** with rationale covering: text extraction fidelity, table detection approach, maintenance status, and license. This decision is left to the implementer but must be documented before the first PR is merged.

---

## 8. Failures Log Specification

### Row-Level Failure Conditions (`ROW_ERROR`)

A row is invalid and must be **omitted from the CSV** if any of the following are true:

1. Any required numeric field (`Capital`, `Interes Corriente`, `Seguros/FNG`, `Total Exigible`, `Total Pagado`, `Pendiente por Pagar`, `Saldo de Capital`) is not parseable as a number.
2. `Fecha Fin` does not match `DD/MM/YYYY` or `DD-MM-YYYY`.
3. `Estado` is empty.
4. The number of columns extracted from the row does not match the expected schema.

### File-Level Failure Conditions (`FILE_ERROR`)

A file is considered completely failed if:

- It cannot be opened or read.
- No table matching the expected pattern is found within the file.
- Every row in the file fails individually.

### Log Entry Format

```
[TIMESTAMP] [LEVEL] Filename=<n> Row=<number|N/A> Reason=<specific description> RawContent=<raw extracted text, truncated to 200 chars>
```

- `TIMESTAMP`: RFC3339 format — `2006-01-02T15:04:05Z07:00`
- `LEVEL`: `ROW_ERROR` for individual row failures | `FILE_ERROR` for complete file failures
- `Reason` must be **specific**, not generic. e.g., `"Capital field '1.234' not parseable: missing decimal"` not `"parse error"`.
- `RawContent`: the raw text extracted from the row before any parsing attempt, truncated to 200 characters.

---

## 9. Regex Constants

All regular expressions must be defined as **named constants** in a single `config.go` or `regex.go` file. They must **not** appear inline anywhere in business logic. Each constant requires an explanatory comment.

```go
// ReNumeric matches currency values with optional symbol, thousands sep and comma decimal
// e.g. "$ 1.234,56" or "1.234.567,00"
const ReNumeric = `\$?\s*([\d\.]+),([\d]{2})`

// ReDate matches dates in DD/MM/YYYY or DD-MM-YYYY format
const ReDate = `(\d{2})[/\-](\d{2})[/\-](\d{4})`
```

---

## 10. HTTP API

### Prefix Path

Configurable via `PREFIX_PATH` env var. Default: `/api/pdf-converter`. Must be applied before any route is registered.

### Endpoints

| Method | Path                                     | Auth Required | Description                                                                                                                                            |
|---|------------------------------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `POST` | `$PREFIX_PATH/read-all`                  | No            | Validate payload, enqueue job, return `202` within ≤3s. Processing continues in background. Back to be available when finishes. Should return an JOBID |
| `GET` | `$PREFIX_PATH/health`                    | No            | Liveness probe. Returns `{"status":"ok"}` plus optional build metadata.                                                                                |
| `GET` | `$PREFIX_PATH/readyNoReadiness/{job_id}` | No |  probe. 200 if service is ready to accept new jobs (no job in progress); 503 if a job is currently running. |

### Response Contract

| Scenario                   | HTTP Status                 | Response Body                                                                                                                           |
|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Job accepted               | `202 Accepted`              | `{"correlation_id":"<uuidv4>","status":"accepted","message":"Job enqueued, processing in background"}`                                  |
| Job already running        | `409 Conflict`              | `{"code":"rate_limit","message":"A job is already in progress","correlation_id":"<id>","hint":"Retry after the current job completes"}` |
| Job finished               | `200 OK`                     | `{"status":"ready", "job_id": <job_id>}`                                                                                                |
| Validation failure         | `400 Bad Request`           | `{"code":"validation","message":"<field-level detail>","correlation_id":"<id>"}`                                                        |
| Missing / invalid token    | `401 Unauthorized`          | `{"code":"auth","message":"Missing or invalid bearer token","correlation_id":"<id>"}`                                                   |
| Valid token, no permission | `403 Forbidden`             | `{"code":"auth","message":"Caller not authorized","correlation_id":"<id>","hint":"<auditable reason>"}`                                 |
| Unexpected error           | `500 Internal Server Error` | `{"code":"internal","message":"Unexpected error","correlation_id":"<id>"}`                                                              |
| Service idle (readiness)   | `200 OK`                    | `{"status":"ready"}`                                                                                                                    |
| Job in-flight (readiness)  | `503 Service Unavailable`   | `{"status":"busy","reason":"job_in_progress"}`                                                                                          |

### Standard Error Envelope

Every non-2xx response must use this envelope:

```json
{
  "code":           "rate_limit | validation | auth | csv_write | internal",
  "message":        "<human-readable description>",
  "correlation_id": "<uuidv4>",
  "hint":           "<optional — required for 409 and 403>"
}
```

Retry guidance per code: `rate_limit` → retry after job completes; `validation` → fix payload and resubmit; `auth` → verify token; `internal` → contact ops with `correlation_id`.

---

## 11. Concurrency & Rate Limiting (API)

- Only **one active job** is allowed at any time. Enforced in the API layer via the `RateLimiter` port.
- When `POST /read-all` arrives: check job state atomically.
  - Job running → respond immediately with `409`. Do **not** queue.
  - No job running → set state `Accepted`, respond `202` within ≤3s, continue processing in background goroutine.
- The rate limiter must **not block the main goroutine**. All state checks and updates must be goroutine-safe (atomic ops or mutex-protected).
- The CSV writer mutex (Section 5) applies equally to API-triggered jobs.

---

## 12. Background Job Lifecycle

| State | Meaning | Transitions To |
|---|---|---|
| `Accepted` | Request validated, `correlation_id` assigned, processing not yet started | `Running` |
| `Running` | PDF extraction and CSV write in progress | `Finished` \| `Failed` |
| `Finished` | All PDFs processed; CSV written successfully | — (terminal) |
| `Failed` | Unrecoverable error; partial output may exist | — (terminal) |

- Job state is persisted **in memory** (minimum) for the lifetime of the process.
- Each job has a **UUIDv4** `correlation_id` generated at acceptance time and propagated through all layers via context.
- Terminal state transitions must be emitted as a structured log entry (`INFO`) and increment the relevant metric.
- Background goroutines must use `defer` and context cancellation to guarantee file handles, mutexes, and temp resources are always released — including on panic.
- Panics inside background goroutines must be **recovered**, logged at `ERROR`, and set job state to `Failed`.

### Graceful Shutdown

On `SIGTERM` or `SIGINT`:

1. Stop accepting new requests immediately.
2. Wait up to `SHUTDOWN_TIMEOUT` (default: `30s`, configurable) for the active job to complete.
3. If the job exceeds the timeout: cancel via context, log the partial state at `WARN`, run the cleanup sweep.
4. Exit with the appropriate exit code (see Section 17).
5. Store a checkpoint for a recoverable job state in a file `checkpoint_<yyyymmddhhmiss> for allowing resumption on restart (v2 enhancement).

---

## 13. Configuration

### Source & Precedence

```
Environment variables  >  YAML config file (path from CONFIG_FILE)  >  hardcoded defaults
```

| Env Var | Default | Description |
|---|---|---|
| `PORT` | `8080` | HTTP server port |
| `PREFIX_PATH` | `/api/pdf-converter` | Route prefix for all API endpoints |
| `INPUT_DIR` | . | Input dir  |
| `RATE_LIMIT_TIMEOUT` | `3s` | Max time to hold connection before responding while job starts in background |
| `SHUTDOWN_TIMEOUT` | `30s` | Max drain time on `SIGTERM` before force-cancelling the active job |
| `CSV_DEFAULT_OUTPUT` | `output.csv` | Default output CSV path when not specified in the request |
| `JOB_MAX_DURATION` | `1m` | Max wall-clock time for a background job before context cancellation |
| `WORKERS_DEFAULT` | `NumCPU` | Default worker goroutine count when not specified in the request |
| `LOG_LEVEL` | `info` | Enum: `debug`, `info`, `warn`, `error` |
| `CONFIG_FILE` | *(unset)* | Optional path to YAML config file for overrides |

Startup validation must **reject the process** with a clear, actionable error if any required variable is missing or any value fails its constraint (e.g., `PORT` not a valid integer, `LOG_LEVEL` not in enum, `WORKERS_DEFAULT` out of range).

---

## 14. Security

### Input Sanitization

- `INPUT_DIR` validated against `..` traversal patterns and validate that is a correct expression for a path.
- `workers` validated as integer within `[1, 16]`.
- `overwrite` validated strictly as boolean — non-boolean strings must return `400`.
- Request body size limit: **2KB** maximum to prevent payload abuse.

---

## 15. Resource Cleanup

- Track every temporary directory and file created during PDF parsing.
- Delete temp resources on: job completion, job failure, and shutdown hooks.
- On startup, **before accepting traffic**, run a cleanup sweep to remove orphaned artifacts from any previous crashed run.
- Use `defer` + context cancellation to guarantee cleanup even on panic.
- Emit metrics `temp_files_deleted_total` and `cleanup_errors_total` to detect leaks early.
- Failure-injection tests must confirm cleanup executes on panic and timeout paths.

---

## 16. Observability

### Structured Logging

- Use `log/slog` (stdlib). Level configurable via `LOG_LEVEL` env var.
- Every log entry must carry: `correlation_id`, `layer` (`api` / `service` / `infra`), level.
- Security events → `WARN`. Job state transitions → `INFO`. Cleanup events → `DEBUG`.

---

## 17. Testing & Quality Gates

| Test Type | Scope | Requirement |
|---|---|---|
| Unit tests | `service`, `infrastructure` parsing & transform packages | ≥95% coverage via `go test -coverprofile`.|
| Contract tests | Each adapter (`CSVWriter`, `RateLimiter`, `JobTracker`, `PDFExtractor`) | Verify each adapter satisfies its port interface and returns correct types/errors. |
| Race condition tests | All packages | `go test -race ./...` must pass with zero data race errors. |
| Stress tests | Rate limiter | Send N concurrent `POST /read-all` requests; assert exactly 1 receives `202` and all others receive `409`. |
| Failure injection | CSV write errors, job timeout, panic recovery, cleanup | Inject failures via mock adapters; assert correct error envelope is returned and all temp resources are cleaned. |

### Test Fixtures

- Tests in the parsing and transformation packages must use **real anonymized or synthetic PDF files**, or mocks of the `PDFExtractor` interface — **never hardcoded strings extracted from production PDFs**.
- No production data of any kind may exist in the repository.
- The integration test suite uses the **reference PDF fixture set**; the DoD assertion is `len(rows) >= 72` after a successful run.

### CI Gates (all must pass before merge)

```
go build ./...
go vet ./...
go test -race ./...
gosec ./...
coverage threshold ≥95% on internal packages
```

---

## 19. README Requirements

The README must document:

1. **Compilation instructions** for both `darwin/arm64` and `linux/amd64`.
2. **Environment variable reference** for the microservice (full table from Section 13).
3. **PDF library justification** — which library was chosen and why (see Section 7).
4. **How to modify regex patterns** — where the constants live, how to test a change, what to watch out for.

---

## 20. Out of Scope (v1)

- Password-protected PDFs
- Bearer token management/rotation strategy (beyond validation in the API layer)
- Recursive directory search (non-default; candidate for `--recursive` flag in v2)
- Extraction of data outside the payment plan table (contract header, debtor details, etc.)
- Export formats other than CSV
- Multiple replicas / distributed locking
- Automatic retry of failed PDFs
- Role-based access control (any valid token = full access in v1)
- mTLS (bearer token sufficient for v1)
- Webhook or polling endpoint for job completion status

---

## 22. Definition of Done

1. `POST $PREFIX_PATH/read-all` accepts the schema in Section 4, enforces auth and validation, returns `202` within ≤3s.
2. CSV writing is thread-safe (mutex-protected adapter). Rate limiter enforces single-job concurrency with fast-fail `409`.
3. All response cases return the standardized error envelope (Section 10).
4. Integration test against the reference PDF fixture set produces a CSV with header + **≥72 data rows**.
5. Every failure cause surfaced via standardized envelope or `failures.log` entry with a **specific, non-generic reason**.
6. Temp resources cleaned on success, failure, panic, and shutdown — confirmed by failure-injection tests.
7. All CI gates pass: `build`, `vet`, `race`, `gosec`, ≥95% coverage on internal packages.
8. README includes: build instructions, CLI examples, env var reference, PDF library justification, and regex modification guide.
10. In the failure file should not be an invalid row by format. It should be analized in order to sanitize and include it

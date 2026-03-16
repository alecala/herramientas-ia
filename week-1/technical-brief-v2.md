# Technical Brief Name: PDF Payment Plan Extractor Microservice (V3)

> **Legend:** Sections marked with `*` are optional. Skip or remove them if not applicable.

| Field   | Value                        |
|---------|------------------------------|
| Author  | Alexis Cala                  |
| Date    | 2026-03-14                   |
| Version | 3.1                          |
| Status  | Draft                        |

---

## 1. Context & Goal

### What exists today

A Go CLI tool currently:

1. Scans a configurable directory for files matching `PlanDePag*.pdf`.
2. Extracts payment-plan rows from each PDF (up to 72 installments per file).
3. Consolidates all valid rows into one CSV ordered by filename, then row position.
4. Logs row-level and file-level extraction failures in a structured failures log.

### What we are building

A Go 1.24 HTTP microservice (hexagonal architecture) that exposes the same extraction flow via REST endpoints and background jobs, with:

- API-driven execution (`POST /read-all`) returning `202 Accepted` quickly.
- Single active job policy (fast-fail `409` when a job is in progress).
- In-memory job lifecycle tracking (`accepted`, `running`, `finished`, `failed`).
- Structured error envelope and correlation IDs.
- Graceful shutdown with timeout and cancellation.

> **Key Principle:** The business extraction behavior remains the same as v2; HTTP is an adapter over service use cases, not a rewrite of core logic.

---

## 2. Technical Approach

### * Architecture

Hexagonal (Ports & Adapters), with strict interface boundaries:

- `api`: HTTP handlers, middleware, payload validation, request size limits, error envelope mapping, rate-limit gate, background orchestration.
- `service`: framework-agnostic use cases (discover PDFs, parse rows, transform values, orchestrate writer and failure logger).
- `infrastructure`: concrete adapters (`PDFExtractor`, `CSVWriter`, `RateLimiter`, `JobTracker`).
- `config`: env-driven configuration loading, defaults, validation, precedence.

Port contracts required:

- `PDFExtractor` used by `service`, implemented in `infrastructure`.
- `CSVWriter` used by `service`, implemented in `infrastructure` (must be mutex-protected).
- `RateLimiter` used by `api`, implemented in `infrastructure`.
- `JobTracker` used by `api` and `service`, implemented in `infrastructure`.

### * Input

#### Parameter Schema

Authoritative request schema for `POST $PREFIX_PATH/read-all`:

| Field               | JSON Key    | Type    | Required | Default                                    | Constraints                                                                                       |
|---------------------|-------------|---------|----------|--------------------------------------------|---------------------------------------------------------------------------------------------------|
| Output CSV path     | `output`    | string  | No       | `CSV_DEFAULT_OUTPUT` (fallback `output.csv`) | Must end with `.csv`; parent dir must exist                                                     |
| Failures log path   | `failures`  | string  | No       | `failures.log`                             | Parent dir must exist                                                                             |
| Overwrite if exists | `overwrite` | boolean | No       | `false`                                    | If `false` and output exists, it is checked synchronously before returning `202`; return `409`   |
| Worker goroutines   | `workers`   | integer | No       | `WORKERS_DEFAULT`                          | Min `1`, Max `16`                                                                                 |

Input/file discovery and scheduling:

- Source directory: `INPUT_DIR` (default `.`).
- Process only files matching `PlanDePag*.pdf`; ignore all others.
- `INPUT_DIR` must exist at startup; if not, the service must refuse to start with a fatal structured log and non-zero exit code.
- File discovery takes a snapshot of `INPUT_DIR` at the moment the job transitions to `running`; files added or renamed after that snapshot are not included in the current job.
- Process files in alphabetical order.
- Row ordering in CSV: file order (alphabetical) then row order within file. Workers process files concurrently but results are merged into a pre-allocated slot-indexed structure keyed by file position, guaranteeing deterministic output order regardless of which worker finishes first.
- Background scan cadence: every 5 minutes. The scanner checks for files matching `PlanDePag*.pdf` in `INPUT_DIR`; if at least one is found and no job is active, it enqueues a new job using env-var defaults (`CSV_DEFAULT_OUTPUT`, `WORKERS_DEFAULT`, `overwrite=false`). The auto-enqueued job is tracked identically to a manually submitted job; its `job_id` is logged at `INFO` level. No HTTP response is produced.

Input sanitization and limits:

- Reject `INPUT_DIR` traversal patterns like `..`.
- Validate `workers` range `[1,16]`.
- Enforce strict JSON boolean for `overwrite` (string `"true"` is rejected).
- Enforce `Content-Type: application/json` on `POST /read-all`; reject with `415 Unsupported Media Type` and `{"code":"unsupported_media_type","message":"Content-Type must be application/json","correlation_id":"<uuidv4>"}` if absent or different.
- Request body max size: 2KB. Requests exceeding this limit must be rejected with `413 Request Entity Too Large` and `{"code":"request_too_large","message":"Request body exceeds 2KB limit","correlation_id":"<uuidv4>"}`.

### * Output

CSV output requirements:

- UTF-8 without BOM.
- Delimiter `,`.
- RFC 4180 quoting for fields with comma or quotes.
- Default output file: `CSV_DEFAULT_OUTPUT` (default `output.csv`).
- If output exists and `overwrite=false`, return `409 duplicated_file` (checked synchronously before returning `202`).
- If output exists and `overwrite=true`, replace it atomically (write to a temp file, then rename).

#### Temp Artifacts

A temp artifact is any file written by the service that is not the final committed output:

- The in-progress partial CSV written during extraction (e.g. `output.csv.tmp`).
- The in-progress partial failures log written during extraction (e.g. `failures.log.tmp`).

Cleanup rules:

| Event | Action |
|-------|--------|
| Job finishes successfully | Rename `.tmp` files to final names; delete any leftover `.tmp` from previous runs. |
| Job fails, panics, or times out | Delete all `.tmp` files produced by that job. Final output file is not written. |
| Graceful shutdown during active job | Cancel job, delete `.tmp` files, log partial state at `WARN`. |
| Startup | Sweep and delete any `.tmp` files left from a previous crashed process. |

The final committed `output.csv` is never deleted by the service under any circumstance; only `.tmp` intermediates are cleaned.

CSV columns in exact order:

1. `Filename`
2. `Fecha Fin`
3. `Capital`
4. `Interes Corriente`
5. `Interes Mora`
6. `Seguros/FNG`
7. `Otros Conceptos`
8. `Total Exigible`
9. `Total Pagado`
10. `Estado`
11. `Pendiente por Pagar`
12. `Condonado`
13. `Saldo de Capital`

Column and transformation rules:

- `Fecha Fin`: accept `DD/MM/YYYY` or `DD-MM-YYYY`; store `YYYY-MM-DD`.
- Required numeric fields: `Capital`, `Interes Corriente`, `Seguros/FNG`, `Total Exigible`, `Total Pagado`, `Pendiente por Pagar`, `Saldo de Capital`.
- Default `0.00` when empty/missing: `Interes Mora`, `Otros Conceptos`, `Condonado`.
- Numeric normalization:
  - Remove currency symbols/spaces.
  - Remove thousands separator `.`.
  - Convert decimal separator `,` to `.`.
  - Preserve negative sign.
  - Always output with `%.2f`.
- `Estado` cannot be empty; expected values include `Pagada`, `Pendiente`, `Vencida`, `Condonada`; unknown but non-empty values are accepted and documented.

Post-processing requirement:

- After a file's rows are successfully extracted and written, that source file must be renamed from `PlanDePag*.pdf` to `Done-PlanDePag*.pdf`.
- If `os.Rename` fails (e.g. permissions error, cross-device link), the failure is logged as a `FILE_ERROR` with `Reason=rename failed: <os error>`. The rows already written to the CSV remain in the output; the file is not re-processed in the same job. The job continues processing remaining files and finishes normally unless all files fail.

### * Log Specification

#### Structured Logging

- Use `log/slog` with configurable `LOG_LEVEL` (`debug`, `info`, `warn`, `error`).
- Every log entry must carry: `correlation_id`, `layer` (`api`/`service`/`infra`), level.
- Security events: `WARN`.
- Job state transitions: `INFO`.
- Cleanup events: `DEBUG`.
- Startup validation failures: `ERROR` to stderr, then `os.Exit(1)`.

#### Failures Log Entry Format

```
[TIMESTAMP] [LEVEL] Filename=<n> Row=<number|N/A> Reason=<specific description> RawContent=<raw extracted text, truncated to 200 chars>
```

- `TIMESTAMP`: RFC3339 (`2006-01-02T15:04:05Z07:00`).
- `LEVEL`: `ROW_ERROR` for row failures, `FILE_ERROR` for complete file failures.
- `Reason` must be specific and actionable.
- `RawContent` must be truncated to 200 chars.

Row-level failure conditions (`ROW_ERROR`):

- Required numeric field is not parseable.
- Date is not in allowed formats.
- `Estado` is empty.
- Extracted column count does not match expected schema.

File-level failure conditions (`FILE_ERROR`):

- File cannot be opened/read.
- Expected table pattern not found.
- All rows in file fail.
- Rename to `Done-PlanDePag*.pdf` fails after successful extraction.

### * Libraries & Dependencies

- PDF extraction library: `ledongthuc/pdfreader`.
- Rationale (must be documented in README before first PR merge):
  - Extraction fidelity with payment plan table text.
  - Table-detection strategy compatibility.
  - Maintenance status/activity.
  - License compatibility.
- External libraries are allowed only when required for service operation and must satisfy security and maintainability criteria.

Regex standard:

- All regex constants live in a single config file (for example `internal/config/regex.go`).
- No inline regex in business logic.
- Every regex constant must include explanatory comments.

Example:

```go
// ReNumeric matches currency values with optional symbol, thousands sep and comma decimal.
const ReNumeric = `\$?\s*([\d\.]+),([\d]{2})`

// ReDate matches dates in DD/MM/YYYY or DD-MM-YYYY format.
const ReDate = `(\d{2})[/\-](\d{2})[/\-](\d{4})`
```

### * API Definition

#### Endpoint Table

Assume route prefix from `PREFIX_PATH` (default `/api/pdf-converter`) is applied before registration.

| Method | Path                                | Rate Limit            | Auth Required | Description                                                         |
|--------|-------------------------------------|-----------------------|---------------|---------------------------------------------------------------------|
| POST   | `$PREFIX_PATH/read-all`             | 1 active job globally | No            | Validates payload, enqueues background processing, returns `job_id` |
| GET    | `$PREFIX_PATH/jobs/{job_id}/status` | None                  | No            | Returns job state (`accepted/running/finished/failed`)              |

> **Auth note:** Authentication is fully out of scope for V3. All endpoints are unauthenticated. Bearer-token and RBAC are deferred to a future version.

#### Identity: `job_id` and `correlation_id`

The `job_id` returned in the `202` response body and the `correlation_id` propagated through logs and error envelopes are **the same UUIDv4**, generated once at acceptance time. All log entries and error envelopes for a given job must carry this single identifier under the `correlation_id` key; the HTTP response surfaces it as `job_id` for client-facing clarity.

#### Response Contract

| Scenario                  | HTTP Status                 | Response Body                                                                                                                                              |
|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Job accepted              | `202 Accepted`              | `{"job_id":"<uuidv4>","status":"accepted","message":"Job enqueued, processing in background"}`                                                             |
| Job already running       | `409 Conflict`              | `{"code":"rate_limit","message":"A job is already in progress","correlation_id":"<uuidv4>","job_id":"<active_job_id>","hint":"Retry after the current job completes"}` |
| Validation failure        | `400 Bad Request`           | `{"code":"validation","message":"<field-level detail>","correlation_id":"<uuidv4>"}`                                                                       |
| Unexpected error          | `500 Internal Server Error` | `{"code":"internal","message":"Unexpected error","correlation_id":"<uuidv4>"}`                                                                             |
| Output file exists        | `409 Conflict`              | `{"code":"duplicated_file","message":"<output_file> already exists, include overwrite: true","correlation_id":"<uuidv4>"}`                                 |
| No matching input files   | `404 Not Found`             | `{"code":"no_input","message":"No PlanDePag*.pdf files found in INPUT_DIR","correlation_id":"<uuidv4>"}`                                                   |
| Body exceeds 2KB          | `413 Request Entity Too Large` | `{"code":"request_too_large","message":"Request body exceeds 2KB limit","correlation_id":"<uuidv4>"}`                                                   |
| Wrong Content-Type        | `415 Unsupported Media Type`| `{"code":"unsupported_media_type","message":"Content-Type must be application/json","correlation_id":"<uuidv4>"}`                                          |
| Job running               | `200 OK`                    | `{"job_id":"<uuidv4>","status":"running"}`                                                                                                                 |
| Job finished              | `200 OK`                    | `{"job_id":"<uuidv4>","status":"finished"}`                                                                                                                |
| Job failed                | `200 OK`                    | `{"job_id":"<uuidv4>","status":"failed","reason":"<description>"}`                                                                                         |
| Job not found             | `404 Not Found`             | `{"code":"not_found","message":"No job with id <id>","correlation_id":"<uuidv4>"}`                                                                         |

> **`no_input` and `duplicated_file` checks are synchronous.** Both are evaluated before the job is accepted and before `202` is returned. If either condition is met, no `job_id` is created.

Standard non-2xx envelope:

```json
{
  "code": "rate_limit | validation | duplicated_file | no_input | request_too_large | unsupported_media_type | csv_write | internal | not_found",
  "message": "<human-readable description>",
  "correlation_id": "<uuidv4>",
  "hint": "<optional; required for 409 rate_limit, recommended for validation>"
}
```

Concurrency/rate-limit behavior:

- Only one active job at a time (active = `accepted` or `running`).
- If active job exists: fail fast with `409`; do not queue.
- If no active job: mark accepted, return quickly (target <=3s), process in background.
- Rate limiter and tracker must be goroutine-safe and non-blocking for server main loop.

Job lifecycle and runtime controls:

- States: `accepted -> running -> finished|failed`.
- Job metadata persisted in memory for the last `JOB_HISTORY_MAX` completed jobs (default `100`); older entries are evicted FIFO. Active and accepted jobs are never evicted.
- `correlation_id` / `job_id` generated as UUIDv4 at acceptance and propagated via context.
- Recover panic in background workers; log at `ERROR` and mark job `failed`.
- Apply `JOB_MAX_DURATION` timeout (default `1m`) to cancel long-running jobs.

Graceful shutdown:

1. Stop accepting new requests on `SIGTERM`/`SIGINT`.
2. Drain active job for up to `SHUTDOWN_TIMEOUT` (default `30s`).
3. Cancel active job after timeout, log partial state at `WARN`, run cleanup sweep (delete `.tmp` files).

Configuration precedence and validation:

- Precedence: environment variables override defaults.
- Required startup validation rejects invalid values (for example invalid `PORT`, invalid `LOG_LEVEL`, out-of-range workers, non-existent `INPUT_DIR`). Any validation failure logs to stderr at `ERROR` and exits with code `1`.

Primary env vars:

- `PORT` (default `8080`)
- `PREFIX_PATH` (default `/api/pdf-converter`)
- `INPUT_DIR` (default `.`; must exist at startup)
- `SHUTDOWN_TIMEOUT` (default `30s`)
- `CSV_DEFAULT_OUTPUT` (default `output.csv`)
- `JOB_MAX_DURATION` (default `1m`)
- `WORKERS_DEFAULT` (default `min(NumCPU, 16)`, bounded by `[1,16]`)
- `LOG_LEVEL` (default `info`)
- `JOB_HISTORY_MAX` (default `100`; minimum `1`)

---

## 3. Restrictions

### * Testing & Quality Gates

| Test Type            | Scope                                                    | Requirement                                                                  |
|----------------------|----------------------------------------------------------|------------------------------------------------------------------------------|
| Unit tests           | `service`, parsing, transforms, infrastructure adapters  | Coverage >=95% on internal packages                                          |
| Contract tests       | `CSVWriter`, `RateLimiter`, `JobTracker`, `PDFExtractor` | Must verify interface behavior and expected errors                           |
| Race tests           | Full repository                                          | `go test -race ./...` with zero races                                        |
| Stress tests         | API rate limiter                                         | N concurrent `POST /read-all`: exactly one `202`, rest `409`                 |
| Failure injection    | CSV errors, timeout, panic recovery, cleanup, rename fail | Verify envelope/logging correctness and `.tmp` cleanup execution            |
| Integration tests    | Fixture PDF set                                          | Successful run yields CSV with header + >=72 data rows                       |
| CSV parse validation | Integration output                                       | Parse output via Go `encoding/csv` with zero parse errors                    |
| Row ordering test    | Integration output with multiple fixture PDFs            | CSV rows must match expected alphabetical file order and intra-file row order |

Mandatory CI gates:

- `go build ./...`
- `go vet ./...`
- `go test -race ./...`
- `gosec ./...` (project-level `.gosec` suppression file required; all suppressions must include a justification comment)
- Internal package coverage threshold >=95%

> **Testing Coverage:** The minimum accepted coverage is 95% on internal packages, with mandatory race and failure-path validation.

### * Test Fixtures

- Use synthetic/anonymized PDFs or mocks for `PDFExtractor`; never include production data.
- Integration tests must use the agreed reference fixture set.
- Manual verification against Google Sheets is required on the reference fixture set before first release.

### * Out of Scope

- Password-protected PDFs.
- Download endpoint (files are supplied through input directory only).
- Authentication, bearer-token validation, RBAC, and mTLS (fully deferred to a future version).
- Recursive directory search.
- Data extraction outside payment-plan table.
- Export formats other than CSV.
- Multi-replica/distributed locking.
- Automatic retry of failed PDFs.
- Completion webhook/polling beyond explicit job status endpoint.
- Reintroduction of deprecated `Row Number` column.
- Per-file size limits (deferred; no `MAX_PDF_SIZE_MB` in V3).

Undefined behavior remains out of scope until explicitly approved through change management.

---

## 4. Definition of Done

- `POST $PREFIX_PATH/read-all` validates schema and returns `202` in <=3 seconds when accepted.
- Exactly one active job is enforced with fast-fail `409` for concurrent submissions.
- `overwrite` and `no_input` checks happen synchronously before `202` is returned.
- CSV writing is thread-safe via mutex-protected adapter.
- CSV row order is deterministic (alphabetical file order, then intra-file row order) regardless of worker concurrency.
- All non-2xx responses follow the standard error envelope with `correlation_id`.
- `job_id` and `correlation_id` are the same UUIDv4, consistently used across response body and logs.
- Integration run on reference fixtures generates output with header + >=72 rows.
- Failures are always surfaced with specific reasons in envelope and/or failures log.
- `.tmp` artifacts are cleaned on success, failure, panic, timeout, and shutdown; startup performs a cleanup sweep.
- Existing output handling follows overwrite policy (`409` when `overwrite=false`, atomic replace when `true`).
- Successfully parsed `PlanDePag*.pdf` files are renamed to `Done-PlanDePag*.pdf`; rename failures are logged as `FILE_ERROR` without removing already-written rows.
- Job history is capped at `JOB_HISTORY_MAX` completed entries with FIFO eviction.
- README documents build instructions (`darwin/arm64`, `linux/amd64`), library justification, architecture Mermaid diagram, regex modification guide, and `gosec` suppression policy.
- CI gates (`build`, `vet`, `race`, `gosec`, coverage threshold) pass.

### Checklist

- [ ] All acceptance criteria are met and verified.
- [ ] Unit and integration tests pass with required coverage.
- [ ] Out-of-scope items have not been implemented.
- [ ] Documentation updated.

---
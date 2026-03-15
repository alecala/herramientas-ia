# Plan: PDF Payment Plan Table Extractor (Go 1.24)

## Context

The user needs a Go 1.24 CLI tool that extracts payment plan tables from PDF files
matching `PlanDePagos-*.pdf` and consolidates them into a single CSV file.
The tool must handle multiple PDFs in parallel, log failures per-row with specific
causes, and achieve ≥95% test coverage on parsing/transformation packages.

**Project root:** `/opt/RAPPIDEV/brief/`
**PDF fixtures (5 files):** `/Users/alexiscala/Documents/PlanDePagos-*.pdf`
**Brief reference:** `/opt/RAPPIDEV/payouts-ai-tools/Brief.md`

---

## Library Decision: `ledongthuc/pdf`

Use `github.com/ledongthuc/pdf` (BSD-3-Clause, pure Go, no CGo).

**Reason:** Only pure-Go library that returns X/Y coordinates per text span via
`GetTextByRow()`. This allows column reconstruction by X-range bucketing.
`pdfcpu` only dumps raw PDF operator streams (unusable for text extraction).
`unipdf` is commercial. `go-fitz` requires CGo + AGPL license.

---

## Project Structure

```
/opt/RAPPIDEV/brief/
├── go.mod                          # module: github.com/rappidev/brief, go 1.24
├── go.sum
├── PLAN.md                         # this file
├── README.md
├── cmd/
│   └── extractor/
│       └── main.go                 # CLI flags, worker pool, orchestration
├── internal/
│   ├── config/
│   │   └── regex.go                # ALL regex constants + X-column boundaries
│   ├── model/
│   │   └── types.go                # PaymentRow struct, FailureRecord struct
│   ├── pdf/
│   │   ├── reader.go               # PDFTextReader interface + ledongthuc impl
│   │   └── reader_test.go
│   ├── parser/
│   │   ├── table.go                # Row assembly from pdf.Rows via X-bucketing
│   │   ├── table_test.go
│   │   ├── numeric.go              # Number normalization ($, thousands, comma→dot)
│   │   ├── numeric_test.go
│   │   ├── date.go                 # Date parsing + normalization to YYYY-MM-DD
│   │   └── date_test.go
│   ├── csv/
│   │   ├── writer.go               # Thread-safe CSV writer (mutex-protected)
│   │   └── writer_test.go
│   └── failures/
│       ├── logger.go               # Thread-safe slog-based failure logger
│       └── logger_test.go
└── testdata/
    └── fixtures/                   # 5 PDFs copied from ~/Documents/
```

---

## Implementation Steps

### Step 0 — Write PLAN.md

Write this plan document to `/opt/RAPPIDEV/brief/PLAN.md` as the first action,
so it lives alongside the code and serves as a durable reference.

---

### Step 1 — Scaffold Go module

```bash
mkdir -p /opt/RAPPIDEV/brief
go mod init github.com/rappidev/brief
go get github.com/ledongthuc/pdf
```

Create all directories in the structure above.

---

### Step 2 — `internal/model/types.go`

Define shared structs used across all packages:

```go
type PaymentRow struct {
    Filename          string
    RowNumber         int
    FechaFin          string   // YYYY-MM-DD
    Capital           float64
    InteresCorriente  float64
    InteresMora       float64
    SegurosFNG        float64
    OtrosConceptos    float64
    TotalExigible     float64
    TotalPagado       float64
    Estado            string
    PendientePorPagar float64
    Condonado         float64
    SaldoDeCapital    float64
}

type FailureRecord struct {
    Filename   string
    Row        string    // row number or "N/A"
    Reason     string    // one of the defined FAILURE_* constants
    RawContent string    // truncated to 200 chars
}

// Failure reason constants
const (
    FailureColumnCount   = "COLUMN_COUNT_MISMATCH"
    FailureNumericParse  = "NUMERIC_PARSE_ERROR"
    FailureDateParse     = "DATE_PARSE_ERROR"
    FailureEmptyEstado   = "EMPTY_ESTADO"
    FailureFileOpen      = "FILE_OPEN_ERROR"
    FailureTableNotFound = "TABLE_NOT_FOUND"
)
```

---

### Step 3 — `internal/config/regex.go`

Centralize ALL regex patterns and column X-boundary configuration:

```go
package config

import "regexp"

// ReNumeric matches currency values: optional $, thousands dots, comma decimal.
// Example: "$ 1.234,56" → groups: ("1234", "56")
const ReNumericRaw = `\$?\s*([\d\.]+),(\d{2})`

// ReDate matches dates in DD/MM/YYYY or DD-MM-YYYY format.
const ReDateRaw = `(\d{2})[/\-](\d{2})[/\-](\d{4})`

// ReTableHeader matches the header row of the payment table to detect table start.
// Adjust this pattern if the PDF changes its header wording.
const ReTableHeaderRaw = `(?i)fecha\s+fin`

// Compiled versions (package-level, compiled once)
var (
    ReNumeric     = regexp.MustCompile(ReNumericRaw)
    ReDate        = regexp.MustCompile(ReDateRaw)
    ReTableHeader = regexp.MustCompile(ReTableHeaderRaw)
)

// ColumnXRanges defines the X-coordinate boundaries for each of the 13 data columns
// (excludes Filename and RowNumber which are not in the PDF).
// Values are in PDF user space units. Adjust if PDFs have different layout.
// Format: [minX, maxX]
var ColumnXRanges = map[string][2]float64{
    "FechaFin":          {0, 80},
    "Capital":           {80, 145},
    "InteresCorriente":  {145, 210},
    "InteresMora":       {210, 270},
    "SegurosFNG":        {270, 330},
    "OtrosConceptos":    {330, 390},
    "TotalExigible":     {390, 450},
    "TotalPagado":       {450, 510},
    "Estado":            {510, 570},
    "PendientePorPagar": {570, 630},
    "Condonado":         {630, 690},
    "SaldoDeCapital":    {690, 760},
}
```

> **Note:** X-range values will be calibrated after inspecting the first real PDF
> during implementation. They are the primary knob to tune if column extraction fails.

---

### Step 4 — `internal/pdf/reader.go`

Define an interface to allow mocking in tests:

```go
package pdf

type TextSpan struct {
    X    float64
    Text string
}
type Row []TextSpan

// PDFTextReader abstracts PDF text extraction; mock in tests.
type PDFTextReader interface {
    ReadRows(path string) ([]Row, error)
}

// LedongthucReader implements PDFTextReader using github.com/ledongthuc/pdf.
type LedongthucReader struct{}

func (r LedongthucReader) ReadRows(path string) ([]Row, error) {
    // open with ledongthuc/pdf, iterate pages, call page.GetTextByRow()
    // convert each pdf.Row into []TextSpan preserving X and string
}
```

Tests use a `MockPDFReader` that returns hardcoded `[]Row` slices.

---

### Step 5 — `internal/parser/`

**`numeric.go`** — `ParseNumeric(s string) (float64, error)`
- Strip `$`, spaces, dots (thousands sep), replace `,` → `.`
- Parse with `strconv.ParseFloat`
- Table-driven tests: normal, negative, empty, missing comma, non-numeric

**`date.go`** — `ParseDate(s string) (string, error)`
- Match `ReDate` from config → return `YYYY-MM-DD`
- Tests: DD/MM/YYYY, DD-MM-YYYY, invalid

**`table.go`** — `ParseRows(rows []Row, filename string) ([]model.PaymentRow, []model.FailureRecord)`
- Skip rows until `ReTableHeader` matches
- Skip header row itself
- For each data row:
  1. Bucket spans into columns by `config.ColumnXRanges`
  2. Column count check → `FailureColumnCount`
  3. Parse `FechaFin` → `FailureDateParse`
  4. Validate `Estado` not empty → `FailureEmptyEstado`
  5. Parse numeric fields → `FailureNumericParse`

---

### Step 6 — `internal/csv/writer.go`

Thread-safe writer: `New`, `WriteHeader`, `WriteRow`, `Flush`, `Close`.
Floats formatted with `%.2f`. String fields via `encoding/csv` (RFC 4180).

---

### Step 7 — `internal/failures/logger.go`

Thread-safe logger writing entries in format:
```
[RFC3339] [ROW_ERROR|FILE_ERROR] Filename=... Row=... Reason=... RawContent=...
```

---

### Step 8 — `cmd/extractor/main.go`

Flags: `--input-dir`, `--output`, `--failures`, `--overwrite`, `--workers`, `--log-level`.

Flow: glob PDFs → worker pool (N goroutines via channel) → each worker reads+parses →
thread-safe write to CSV and failures → summary + exit codes (0/1/2/3).

---

### Step 9 — Copy PDF fixtures

```bash
cp /Users/alexiscala/Documents/PlanDePagos-*.pdf /opt/RAPPIDEV/brief/testdata/fixtures/
```

---

### Step 10 — README.md

Build instructions, CLI usage examples, regex modification guide, library justification.

---

## Coverage Strategy (≥95% on internal packages)

| Package            | Test approach                                      |
|--------------------|----------------------------------------------------|
| `internal/parser`  | Table-driven unit tests with mock `[]Row` inputs   |
| `internal/csv`     | Write to `os.CreateTemp`, assert file contents     |
| `internal/failures`| Write to temp file, assert log format              |
| `internal/pdf`     | Integration test using real fixture PDFs           |
| `internal/config`  | Tested indirectly via parser tests                 |
| `cmd/extractor`    | **Excluded** from 95% requirement (CLI/I/O layer)  |

```bash
go test -coverprofile=cover.out ./internal/...
go tool cover -func=cover.out
```

---

## Verification

```bash
# Build
go build ./...
go vet ./...

# Tests + race detector + coverage
go test -race -coverprofile=cover.out ./internal/...
go tool cover -func=cover.out

# Run against real PDFs
./extractor --input-dir=testdata/fixtures --output=output.csv --failures=failures.log --log-level=debug

# DoD: ≥72 data rows
wc -l output.csv    # expect ≥73

# Spot-check
head -3 output.csv
cat failures.log
```

---

## Critical Unknowns / Risks

1. **X-range calibration**: `config.ColumnXRanges` values must match the actual PDF
   layout. Debug logging will emit raw X values on first run to allow tuning.

2. **Table header text**: `ReTableHeaderRaw` must match the actual header row wording
   in the PDFs. Confirmed via debug log on first run.

3. **Row ordering**: `ledongthuc/pdf` groups by Y coordinate (bottom-to-top in PDF
   space). If rows appear reversed, apply `sort.Slice` by Y descending.

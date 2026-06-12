# AGENTS.md — fbc-update-planner

## What This Is

`plcc2fbc` fetches operator lifecycle data from Red Hat's Product Life Cycle Center (PLCC) API and converts it to File-Based Catalog (FBC) YAML blobs for OpenShift. Each PLCC product becomes an FBC package with versioned lifecycle phases (General Availability, End of Life, etc.).

## Tech Stack

- **Language:** Go (version in `go.mod`)
- **Dependencies:** `sigs.k8s.io/yaml` for YAML marshaling, `spf13/pflag` for CLI flag parsing
- **CI:** GitHub Actions on PRs to `main` — runs `make test` + `golangci-lint` (see `.github/workflows/tests.yaml`)
- **License:** Apache 2.0 (all `.go` files carry the header)

## Layout

```
cmd/plcc2fbc/main.go       CLI entry point — flag parsing, orchestration
cmd/plcc2fbc/main_test.go   Tests for CLI (run function)
pkg/plcc/plcc.go            PLCC API client, data types, filtering, sorting
pkg/plcc/plcc_test.go       Tests for PLCC package
pkg/fbc/fbc.go              FBC schema, PLCC→FBC translation, GenerateFBC()
pkg/fbc/fbc_test.go         Tests for FBC translation
pkg/fbc/filter.go           Validation pipeline — 6 ordered Filter callbacks
pkg/fbc/filter_test.go      Tests for individual filters
pkg/fbc/writer.go           PackageWriter interface + JSON/YAML serializers
pkg/fbc/writer_test.go      Tests for writers
pkg/fbc/pipeline_test.go    Integration test — full pipeline vs reference output
pkg/fbc/testdata/           Test fixtures (plcc.json, reference-fbc.yaml)
docs/VALIDATION_RULES.md    Filter pipeline spec (read before touching filters)
docs/FBC_SCHEMA.md          FBC output schema reference
schema-examples/            Example PLCC + FBC schemas for reference
fbc-samples/                Generated FBC snapshots (YAML, logs, validation logs)
.github/workflows/tests.yaml  CI workflow definition
```

## Commands

```sh
make build          # → bin/plcc2fbc
make test           # go test -v ./...
make generate-fbc   # build + run against live PLCC API, write YAML + logs to fbc-samples/
```

No separate lint command — CI runs `golangci-lint` with defaults (no `.golangci.yaml`).

### CLI Flags

```
plcc2fbc [flags] <output-file>

-o, --output   Output format: json, json-pretty, or yaml (default: json)
-l, --log      Write operational logs to a file (default: stdout)
-p, --package  Comma-separated package names to process (default: all)
-i, --input    Read PLCC JSON from a file instead of fetching from API
    --dump-plcc  Dump filtered PLCC JSON instead of generating FBC
```

## Architecture

### Data Flow

```
PLCC API (or -i file) → plcc.Fetch()/Load()
  → plcc.FilterByPackageNames()  # if -p flag set
  → plcc.FilterPackages()        # otherwise: drop products without package names
  → plcc.SortByPackage()         # alphabetical
  → fbc.GenerateFBC()            # translate + validate + emit via PackageWriter

With --dump-plcc:
  → catalog.Dump()               # write filtered PLCC JSON directly, skip FBC generation
```

### Validation Pipeline (`pkg/fbc/filter.go`)

Filters run in order via `DefaultFilters()`. Each has signature `func(*Package) []string`. Non-empty return rejects the package. Pipeline **short-circuits** on first rejection.

| Order | Function                    | Kind     |
|-------|-----------------------------|----------|
| 1     | FilterPointInTimePhases     | validate |
| 2     | FilterIncompletePhases      | mutate   |
| 3     | ValidateHasVersions         | validate |
| 4     | ValidateVersionNames        | validate |
| 5     | ValidatePhases              | validate |
| 6     | ValidateOCPCompatibility    | validate |

Order matters: mutating filters prepare data for validators downstream.

### Key Types

- `plcc.Catalog` / `plcc.Product` / `plcc.Version` / `plcc.Phase` — API-side types
- `fbc.Package` / `fbc.VersionLifecycle` / `fbc.Phase` — output-side types
- `fbc.ValidationResult` — structured JSON logged to stderr for rejected packages
- `fbc.Filter` — `func(*Package) []string` pipeline callback
- `fbc.PackageWriter` — interface for serializing packages (JSON, JSON-pretty, YAML)

### FBC Schema

Output blobs use schema `io.openshift.operators.lifecycles.v1alpha1`. See `docs/FBC_SCHEMA.md` for field details.

## Patterns to Follow

### Adding a validation filter

1. Write `func ValidateMyRule(p *Package) []string` in `pkg/fbc/filter.go`
2. Add it to `DefaultFilters()` at the correct position (mutators before validators)
3. Add test in `pkg/fbc/filter_test.go` — table-driven, cover accept + reject paths
4. Read `docs/VALIDATION_RULES.md` first — it explains ordering constraints

### Writing tests

- Test data lives in `pkg/fbc/testdata/` (plcc.json, reference-fbc.yaml)
- `pipeline_test.go` is the integration test — compares full pipeline output against `reference-fbc.yaml`
- If your change alters valid output, update `reference-fbc.yaml` to match
- Standard library test assertions — no external assertion libraries

### Version format

Versions must match `^\d+\.\d+$` (MAJOR.MINOR only). This regex is `plcc.MajorMinorRegex`. Patch versions, pre-release suffixes, etc. are rejected.

### Timestamps

- PLCC API uses ISO8601 with milliseconds: `2025-11-11T00:00:00.000Z`
- FBC output uses `YYYY-MM-DD`
- `"N/A"` or empty timestamps translate to empty strings

## Gotchas

- The CLI exits with code 2 if no valid FBC blobs are produced, and code 1 for other fatal errors — both are intentional
- `FilterIncompletePhases` mutates the package in place (drops phases) — it never rejects
- Phase continuity requires exactly +1 day gap between consecutive phases (no overlap, no gap)
- All `.go` files must have the Apache 2.0 license header
- `fbc-samples/` contains committed generated files — update via `make generate-fbc`, not by hand
- No `.golangci.yaml` — linter uses upstream defaults
- Duplicate package names across products cause rejection — `TranslateAndValidate` logs one failure and skips all affected products

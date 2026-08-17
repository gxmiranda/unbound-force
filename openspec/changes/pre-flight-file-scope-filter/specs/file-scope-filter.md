## ADDED Requirements

### FR-020: File-Scope Mapping Table

The pre-flight skill MUST define a declarative mapping from
each detected tool to the set of file patterns that determine
whether the tool is in scope for a given branch diff.

The mapping MUST include:

| Tool | In-scope file patterns |
|------|----------------------|
| `go test` | `*.go`, `go.mod`, `go.sum` |
| `golangci-lint` | `*.go`, `go.mod`, `go.sum`, `.golangci.yml`, `.golangci.yaml` |
| `ruff` | `*.py`, `ruff.toml`, `pyproject.toml` |
| `pytest` | `*.py`, `pyproject.toml`, `setup.py`, `conftest.py` |
| `yamllint` | `*.yml`, `*.yaml`, `.yamllint.yml`, `.yamllint.yaml` |
| `make check` | _always in scope_ |
| `pre-commit` | _always in scope_ |

Tools marked "always in scope" MUST NOT be skipped by the
file-scope filter. These are aggregate tools whose scope
cannot be reliably determined from file extensions alone.

The mapping SHOULD be extensible — additional tools added
to Phase 2's tool-to-command mapping in the future MUST
have a corresponding file-scope entry.

Note: `go vet` and `go build` are CI commands discovered in
Phase 1 (CI Workflow Parsing), not independently detected
tools in Phase 2 (Tool Detection). They do not need scope
entries because they are not individually executed by the
pre-flight skill. They are covered by the `make check`
aggregate tool entry (always in scope) and, in the case of
`go vet`, by `golangci-lint`'s inclusion of vet rules.

File patterns use suffix matching against the full path
returned by `git diff --name-only`. A pattern `*.go` matches
any file whose path ends in `.go`, regardless of directory
depth (e.g., `internal/scaffold/foo.go` matches `*.go`).

#### Scenario: Go-only tool mapping

- **GIVEN** the pre-flight skill is executing
- **WHEN** the branch diff contains only `.yaml` files
- **THEN** `go test` and `golangci-lint` are marked as
  out-of-scope because no diff files match `*.go`,
  `go.mod`, `go.sum`, `.golangci.yml`, or `.golangci.yaml`

#### Scenario: Config file triggers tool

- **GIVEN** the branch diff contains only `.golangci.yml`
- **WHEN** the file-scope filter runs
- **THEN** `golangci-lint` is marked as in-scope because
  `.golangci.yml` is in its file-scope pattern list

### FR-021: Branch Diff Computation

The pre-flight skill MUST compute the branch diff file list
using `git diff --name-only` against the merge base of the
current branch and the default branch.

The diff command MUST be:

```bash
git diff --name-only $(git merge-base HEAD origin/${DEFAULT_BRANCH})...HEAD
```

Where `${DEFAULT_BRANCH}` is detected using the same logic
as Phase 4a (lines 218-249 of the current SKILL.md).

If the default branch cannot be detected, or if the diff
command fails, the file-scope filter MUST be skipped and
all tools MUST run (conservative fallback).

If the diff is empty (no changed files) and the current
branch is NOT the default branch, the file-scope filter
MUST be skipped and all tools MUST run (conservative
fallback), since an empty diff on a feature branch may
indicate a git state anomaly. An informational note MUST
explain: "Empty diff detected — running all tools as
conservative fallback." Always-in-scope tools (FR-020)
run regardless.

If the repository is a shallow clone (detected via
`git rev-parse --is-shallow-repository`), the file-scope
filter MUST be skipped and all tools MUST run, since the
merge-base computation may produce incomplete results in
shallow clones.

#### Scenario: Empty diff on feature branch

- **GIVEN** the branch diff is empty (no files changed)
  and the current branch is not the default branch
- **WHEN** Phase 2a runs
- **THEN** the file-scope filter is bypassed, all tools
  run, and an informational note explains the fallback

#### Scenario: Shallow clone

- **GIVEN** the repository is a shallow clone
- **WHEN** the branch diff computation is attempted
- **THEN** the file-scope filter is skipped, all tools run,
  and an informational note explains the shallow clone
  limitation

#### Scenario: Diff against merge base

- **GIVEN** a feature branch with 3 commits ahead of `main`
- **WHEN** the branch diff is computed
- **THEN** the diff includes all files changed across all 3
  commits relative to the merge base, not just the latest
  commit

#### Scenario: Default branch unavailable

- **GIVEN** the default branch cannot be detected
- **WHEN** the branch diff computation is attempted
- **THEN** the file-scope filter is skipped, all tools run,
  and an informational note explains the fallback

### FR-022: File-Scope Filter Phase

The pre-flight skill MUST add a file-scope filtering step
between Phase 2 (Tool Detection) and Phase 3 (CI Coverage
Matrix). This step is designated Phase 2a.

Phase 2a MUST:

1. Compute the branch diff file list (per FR-021).
2. For each detected and available tool from Phase 2,
   intersect the diff file list with the tool's file-scope
   patterns (per FR-020).
3. If zero diff files match a tool's patterns, mark the tool
   as "SKIP — no in-scope files."
4. If one or more diff files match, the tool proceeds to
   Phase 3 as normal.

Phase 2a MUST apply in all three execution modes:
`hard-gate`, `ci-aware`, and `soft-gate`.

#### Scenario: YAML-only branch with Go tools detected

- **GIVEN** the branch diff contains only `release.yml`
  and `.goreleaser.yaml`
- **AND** Phase 2 detected `go test`, `golangci-lint`,
  `yamllint`, and `make check`
- **WHEN** Phase 2a runs
- **THEN** `go test` is marked "SKIP — no in-scope files"
- **AND** `golangci-lint` is marked "SKIP — no in-scope files"
- **AND** `yamllint` proceeds to Phase 3 (`.yaml` matches)
- **AND** `make check` proceeds to Phase 3 (always in scope)

#### Scenario: Mixed diff with Go and YAML files

- **GIVEN** the branch diff contains `main.go` and
  `config.yaml`
- **WHEN** Phase 2a runs
- **THEN** all detected tools proceed to Phase 3 (both Go
  and YAML tools have matching files)

### FR-023: Skip Reporting

Skipped tools MUST be reported with a distinct status that
differentiates them from tools that ran and passed.

The coverage matrix (Phase 3) MUST include skipped tools
with status "SKIP — no in-scope files" in the "Run locally?"
column.

The execution results (Phase 5) MUST include skipped tools
with:
- Exit code: `—` (not applicable)
- Status: `SKIP — no in-scope files`

Skipped tools MUST be counted as PASS for the purposes of
the overall verdict. A tool with zero applicable files in
the diff cannot produce findings, so skipping it is
semantically equivalent to a clean exit.

The verdict section (Phase 5) SHOULD include a count of
skipped tools, e.g., "2 tools skipped — no in-scope files."

Note: The canonical skip status string is
`SKIP — no in-scope files` (em dash). This string MUST be
used consistently in the coverage matrix "Run locally?"
column, the execution results Status column, and the
verdict summary.

#### Scenario: Skip reporting in coverage matrix

- **GIVEN** `golangci-lint` has no in-scope files
- **WHEN** the coverage matrix is displayed
- **THEN** the matrix row shows:
  `| golangci-lint | ... | ... | SKIP — no in-scope files |`

#### Scenario: Skip reporting in verdict

- **GIVEN** 4 tools were detected, 2 were skipped, 2 ran
  and passed
- **WHEN** the verdict is displayed
- **THEN** the result is PASS with a note: "2 tools skipped
  (no in-scope files)"

## MODIFIED Requirements

### FR-005: Hard-Gate Execution (lines 145-150)

In hard-gate mode, all detected and available tools that
have in-scope files in the branch diff are marked "Run
locally = Yes." Tools with zero in-scope files are marked
"SKIP — no in-scope files" and are not executed. The CI
status column in the matrix shows the actual status if
available. Skip decisions based on file scope are applied
before CI coverage matrix evaluation.

Previously: "In hard-gate mode, ALL detected and available
tools are marked 'Run locally = Yes' regardless of CI
status."

### FR-006: CI-Aware Execution (lines 128-141)

In ci-aware mode, the CI coverage matrix is built from
detected and available tools that have in-scope files in
the branch diff. Tools with zero in-scope files are marked
"SKIP — no in-scope files" and are excluded from CI
coverage evaluation. The file-scope filter (Phase 2a) is
applied before CI coverage matrix construction (Phase 3),
so tools removed by the filter never appear in the matrix's
"Run locally?" decision.

Previously: "For each detected and available tool, determine
which CI check covers the same verification" (no file-scope
filtering before matrix construction).

### FR-007: Soft-Gate Execution (lines 198-200)

Execute all detected and available tools that have in-scope
files in the branch diff (same scope filtering as hard-gate).
Do NOT stop on first failure — record all exit codes and
output for every tool.

Previously: "Execute ALL detected and available tools (same
as hard-gate)."

## REMOVED Requirements

None.

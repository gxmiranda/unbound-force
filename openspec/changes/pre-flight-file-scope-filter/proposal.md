<!-- spec-review: passed -->

## Why

The pre-flight skill runs ALL detected tools regardless of whether
the branch diff contains files relevant to each tool's scope. On
branches that only modify YAML, Markdown, or agent files, Go tools
(`go vet`, `golangci-lint`, `go test -race`, `go build`) still
execute, wasting ~2 minutes of wall time and significant LLM token
budget on tool output that cannot produce findings.

This was identified in issue #434, observed on PR #433 (branch
`428-adopt-org-infra-release-workflows`) which modified only
`.github/workflows/release.yml` and `.goreleaser.yaml`. All five
triage panelists assessed the issue as VALID.

The current behavior is by design — SKILL.md lines 145-150 state
hard-gate mode runs "ALL detected and available tools...regardless
of CI status." The spec lacks a file-scope awareness concept. This
change adds that concept.

Fixes #434.

## What Changes

Add a file-scope filtering phase to the pre-flight skill that
intersects the branch diff file list with each tool's applicable
file extensions. Tools with zero in-scope files are skipped with
an informational note. Skipped tools are counted as PASS (no
findings possible = no failures).

The change applies to all three execution modes: `hard-gate`,
`ci-aware`, and `soft-gate`.

## Capabilities

### New Capabilities

- `file-scope-filter`: A tool-to-file-extension mapping that
  determines which tools are relevant to the current branch diff.
  Tools with no applicable files in the diff are skipped.
- `scope-skip-reporting`: Skipped tools are reported with a
  distinct status ("SKIP — no in-scope files") in both the CI
  coverage matrix and the execution results, distinguishing them
  from tools that ran and passed.

### Modified Capabilities

- `hard-gate execution`: Now skips tools with zero in-scope files
  instead of running all tools unconditionally.
- `ci-aware execution`: File-scope filter applied before the CI
  coverage matrix decision, providing an additional skip reason.
- `soft-gate execution`: File-scope filter applied before
  execution, preventing unnecessary baseline establishment for
  tools that have no applicable files.
- `coverage-matrix display`: Adds a "Scope" column or status to
  show whether each tool has in-scope files.

### Removed Capabilities

- None.

## Impact

### Files affected

- `~/.config/opencode/skills/pre-flight/SKILL.md` — primary
  skill definition (user-level)
- `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
  — scaffold-deployed copy (must stay in sync)

### Behavioral changes

- Branches with no `.go` files in the diff will skip Go tools.
- Branches with no `.py` files in the diff will skip Python tools.
- Branches with no `.yml`/`.yaml` files in the diff will skip
  yamllint.
- `make check` and `pre-commit` always run (scope-ambiguous
  aggregate tools).
- Tool config file changes (`.golangci.yml`, `ruff.toml`,
  `go.mod`, `go.sum`) trigger their respective tools even when
  no source files changed.

### Complementary work

- Issue #390 (GitHub Actions security linters) adds scope-
  appropriate tooling for workflow files. These issues are
  complementary, not conflicting.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change modifies a shared skill consumed by AI agents. It
does not alter inter-hero artifact formats, communication
protocols, or metadata structures. The skill remains a
self-contained instruction document.

### II. Composability First

**Assessment**: PASS

The file-scope filter is additive — it does not introduce
dependencies on other heroes or tools. The pre-flight skill
remains independently consumable. The diff computation uses
only `git`, which is already a prerequisite for the skill's
operation (Phase 4a already uses git worktrees).

### III. Observable Quality

**Assessment**: PASS

The change maintains and improves observability. Skipped tools
are explicitly reported with a distinct status ("SKIP — no
in-scope files") rather than silently omitted. The coverage
matrix remains visible and auditable. The file-scope mapping
table itself is declarative and inspectable.

### IV. Testability

**Assessment**: PASS

The file-scope filter is a deterministic mapping (file
extensions → tool set) intersected with a deterministic input
(branch diff file list). Both inputs are observable and
reproducible. The filter logic can be verified by inspecting
the coverage matrix output — testable through the existing
Phase 5 result format.

### V. Security by Default

**Assessment**: PASS

The file-scope filter is a gate-narrowing mechanism, which
requires care. However, skipping a tool that has zero applicable
files in the diff is semantically equivalent to running it and
getting a clean exit — no false negatives are introduced. The
scope mapping includes tool config files (`.golangci.yml`,
`ruff.toml`, `go.mod`, `go.sum`) to ensure that configuration
changes trigger their respective tools. Aggregate tools
(`make check`, `pre-commit`) always run as a conservative
default. No gatekeeping integrity is violated.

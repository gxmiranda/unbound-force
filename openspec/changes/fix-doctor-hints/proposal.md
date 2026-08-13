# Proposal: Fix Doctor Install Hints to Be Tool-Agnostic

Fixes: #460 | Split from: #212

## Why

Five `InstallHint` strings in `internal/doctor/checks.go` hardcode
`"Run: /uf.agent-brief in OpenCode"`. This couples `uf doctor`
output to a specific tool (OpenCode), producing unhelpful guidance
for users who run `uf doctor` standalone or with a different editor
integration.

Other hints in the same file already follow a tool-agnostic pattern
(e.g., `"Run: uf init"`, `"Add fenced code blocks with build/test
commands"`, `"Add a Branch Protection section to AGENTS.md"`). The
five `/uf.agent-brief` hints are inconsistent outliers.

## What Changes

Replace five `InstallHint` string literals in
`internal/doctor/checks.go` with tool-agnostic plain-language
instructions that tell the user what to do, not which tool to
invoke:

| Line | Current hint | Proposed replacement |
|------|-------------|---------------------|
| 1668 | `"Run: /uf.agent-brief in OpenCode"` | `"Create an AGENTS.md file in your project root describing the project for AI agents"` |
| 1694 | `"Run: /uf.agent-brief in OpenCode"` | `"Add a '<section>' section to AGENTS.md"` (dynamic per Tier 1 section) |
| 1723 | `"Run: /uf.agent-brief in OpenCode"` | `"Condense AGENTS.md — current line count exceeds threshold"` |
| 1749 | `"Run: /uf.agent-brief in OpenCode"` | `"Add a constitution reference to AGENTS.md (e.g., Instructions from: path/to/constitution.md)"` |
| 1773 | `"Run: /uf.agent-brief in OpenCode"` | `"Add a Specification Workflow section to AGENTS.md describing your spec framework"` |

Update the corresponding test assertion in
`internal/doctor/doctor_test.go` (line ~3255) to match the new hint
text.

Optionally add a drift-detection test that reads the project's own
`AGENTS.md` and verifies all five Tier 1 sections are present.

## Capabilities

- `uf doctor` output becomes actionable regardless of editor or tool
  environment
- Hints describe the remediation action, not the mechanism to invoke
  it
- Existing JSON output schema (`install_hint` field in `CheckResult`)
  is unchanged

## Impact

- **Files changed**: 2 (`internal/doctor/checks.go`,
  `internal/doctor/doctor_test.go`)
- **API changes**: None — `InstallHint` is a display-only string
  field
- **Breaking changes**: None — the `CheckResult` struct and JSON
  schema are unmodified
- **Dependencies**: None added or removed
- **Risk**: Low — string literal replacements with no behavioral
  change

## Constitution Alignment

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | Autonomous Collaboration | N/A | No inter-hero communication affected |
| II | Composability First | PASS | This fix directly serves Principle II — `uf doctor` will produce actionable output when deployed alone, without requiring OpenCode |
| III | Observable Quality | N/A | JSON output format and schema unchanged |
| IV | Testability | PASS | Test assertion updated to match; optional drift-detection test strengthens regression protection |
| V | Security by Default | N/A | No new inputs, dependencies, or privilege changes |

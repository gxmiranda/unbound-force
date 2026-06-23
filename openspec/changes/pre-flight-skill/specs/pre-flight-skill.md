## ADDED Requirements

### Requirement: FR-001 Pre-flight Skill File

The system MUST provide a shared skill file at
`.opencode/skills/pre-flight/SKILL.md` that encapsulates
CI workflow detection, local tool detection, CI coverage
matrix generation, and local tool execution.

#### Scenario: Skill file exists and is loadable

- **GIVEN** the repository contains
  `.opencode/skills/pre-flight/SKILL.md`
- **WHEN** a command invokes the `skill` tool with
  name `pre-flight`
- **THEN** the skill instructions are injected into
  the agent's context

### Requirement: FR-002 CI Workflow Parsing

The pre-flight skill MUST read all files in
`.github/workflows/` to identify the exact commands
CI runs. The skill MUST NOT hardcode language-specific
commands — the workflow files are the source of truth.

#### Scenario: Workflow-driven command discovery

- **GIVEN** a repository with `.github/workflows/ci_local.yml`
  containing `go test -race -count=1 ./...`
- **WHEN** the pre-flight skill parses CI workflows
- **THEN** `go test -race -count=1 ./...` appears in the
  discovered CI commands list

#### Scenario: No workflow files present

- **GIVEN** a repository with no `.github/workflows/` directory
- **WHEN** the pre-flight skill parses CI workflows
- **THEN** the CI commands list is empty and the skill
  proceeds to config-file detection only

The skill MUST extract only `run:` step commands from
workflow YAML. Action references (`uses:` steps) MUST
be ignored — they are GitHub-hosted and not locally
executable. Commands containing unresolvable CI
expressions (`${{ secrets.* }}`, `${{ github.* }}`)
MUST be skipped with a warning.

#### Scenario: Workflow with unresolvable expressions

- **GIVEN** a workflow step containing
  `curl -H "Authorization: ${{ secrets.TOKEN }}" ...`
- **WHEN** the pre-flight skill parses CI workflows
- **THEN** the command is skipped with a warning noting
  unresolvable CI expressions
- **AND** the command does not appear in the execution list

### Requirement: FR-003 Local Tool Detection

The pre-flight skill MUST detect available local tools by
checking for their configuration files. The following
mappings MUST be supported:

| Config file | Tool | Command |
|-------------|------|---------|
| `Makefile` | Make | `make check` (preferred), else `make lint` |
| `.golangci.yml` | golangci-lint | `golangci-lint run ./...` |
| `ruff.toml` or `pyproject.toml` | ruff | `ruff check .` |
| `.yamllint.yml` | yamllint | `yamllint .` |
| `.pre-commit-config.yaml` | pre-commit | `pre-commit run --all-files` |
| `go.mod` | go test | `go test ./...` |
| `pyproject.toml` or `setup.py` | pytest | `pytest` or `python -m pytest` |

#### Scenario: Multiple tools detected

- **GIVEN** a repository containing `Makefile`,
  `.golangci.yml`, and `go.mod`
- **WHEN** the pre-flight skill detects local tools
- **THEN** three tools are reported: Make, golangci-lint,
  and go test

When `pyproject.toml` is present, both ruff and pytest
SHOULD be detected as separate tools.

#### Scenario: No tool configuration files present

- **GIVEN** a repository with no recognized config files
- **WHEN** the pre-flight skill detects local tools
- **THEN** the detected tools list is empty and the skill
  reports "no tools detected" before proceeding

#### Scenario: Config file present but tool not installed

- **GIVEN** a repository containing `.golangci.yml`
- **AND** `golangci-lint` is not in PATH
- **WHEN** the pre-flight skill detects local tools
- **THEN** golangci-lint is reported as "detected but not
  available" and skipped with a warning
- **AND** the tool does not appear in the execution list

### Requirement: FR-004 CI Coverage Matrix

The pre-flight skill MUST build a CI coverage matrix
mapping each detected local tool to its corresponding
CI check. The matrix MUST display the CI status and
the skip/run decision for each tool.

#### Scenario: CI-aware mode with passing CI checks

- **GIVEN** a PR where CI check "Local CI / test" has
  status PASS
- **AND** `go.mod` is detected (maps to `go test`)
- **WHEN** the pre-flight skill builds the coverage
  matrix in `ci-aware` mode
- **THEN** the matrix shows `go test` with CI status
  PASS and Run locally = No

#### Scenario: CI-aware mode with no CI checks

- **GIVEN** a PR with no CI check results available
- **WHEN** the pre-flight skill builds the coverage
  matrix in `ci-aware` mode
- **THEN** all detected tools show Run locally = Yes

#### Scenario: Hard-gate mode ignores CI status

- **GIVEN** any CI check status (PASS, FAIL, or NONE)
- **WHEN** the pre-flight skill builds the coverage
  matrix in `hard-gate` mode
- **THEN** all detected tools show Run locally = Yes
  (hard-gate always runs everything)

### Requirement: FR-005 Hard-Gate Execution Policy

The pre-flight skill MUST support a `hard-gate` execution
policy that runs all detected tools and stops on the first
failure. This policy MUST NOT skip tools based on CI status.

#### Scenario: All tools pass in hard-gate mode

- **GIVEN** the pre-flight skill is invoked in
  `hard-gate` mode
- **AND** all detected tools exit with code 0
- **WHEN** execution completes
- **THEN** the verdict is PASS and the consuming command
  proceeds

#### Scenario: Tool failure in hard-gate mode

- **GIVEN** the pre-flight skill is invoked in
  `hard-gate` mode
- **AND** `golangci-lint run ./...` exits with code 1
- **WHEN** the failure is detected
- **THEN** execution stops immediately with verdict FAIL
- **AND** the full error output is included in the result
- **AND** the consuming command MUST NOT proceed to
  AI review or implementation

### Requirement: FR-006 CI-Aware Execution Policy

The pre-flight skill MUST support a `ci-aware` execution
policy that consults the CI coverage matrix and skips tools
already verified by passing CI checks.

#### Scenario: Some tools skipped in ci-aware mode

- **GIVEN** the pre-flight skill is invoked in
  `ci-aware` mode
- **AND** CI check "CI Checks / lint" has status PASS
  (covers `golangci-lint`)
- **AND** no CI check covers `yamllint`
- **WHEN** the coverage matrix is evaluated
- **THEN** `golangci-lint` is skipped (CI verified)
- **AND** `yamllint` is run locally

#### Scenario: CI failure in ci-aware mode

- **GIVEN** the pre-flight skill is invoked in
  `ci-aware` mode
- **AND** CI check "Local CI / test" has status FAIL
- **WHEN** the coverage matrix is evaluated
- **THEN** `go test` is skipped locally (failure already
  captured from CI)
- **AND** the CI failure is included in the result
  context for AI review

### Requirement: FR-007 Standardized Result Format

The pre-flight skill MUST produce results in a standardized
format containing: CI coverage matrix, execution results
table, and verdict (mode, result, failures list).

#### Scenario: Result format is parseable by consumer

- **GIVEN** the pre-flight skill completes execution
- **WHEN** the result is returned to the consuming command
- **THEN** the result contains a CI Coverage Matrix table,
  an Execution Results table, and a Verdict section with
  mode, result (PASS/FAIL), and failures list

## MODIFIED Requirements

### Requirement: FR-008 /review-council Phase 1a

`/review-council` Phase 1a MUST be replaced with a
reference to the pre-flight skill in `hard-gate` mode.
Phase 1b (Gaze quality analysis) MUST remain in the
command unchanged.

Previously: Phase 1a contained inline logic to read
`.github/workflows/`, execute CI commands, and stop on
failure (lines 130-151 of `review-council.md`).

#### Scenario: review-council uses pre-flight skill

- **GIVEN** the `/review-council` command is invoked in
  Code Review Mode
- **WHEN** Phase 1a executes
- **THEN** the agent loads the `pre-flight` skill
- **AND** runs pre-flight checks in `hard-gate` mode
- **AND** stops on failure (same behavior as before)
- **AND** proceeds to Phase 1b on success

Note: The unified detection approach (D2) means
`/review-council` may discover additional tools via
config-file detection that it previously missed (e.g.,
yamllint). This is an intentional improvement, not a
regression. The tool set MAY be a superset of what
the previous inline implementation detected.

### Requirement: FR-009 /review-pr Step 4

`/review-pr` Step 4 MUST be replaced with a reference to
the pre-flight skill in `ci-aware` mode. The Step 4
decision rules (CI PASS = skip, CI FAIL = skip, CI NONE =
run, No CI = run all) MUST be preserved.

Previously: Step 4 contained inline logic for config-file
detection, CI coverage matrix construction, and conditional
tool execution (lines 142-198 of `review-pr.md`).

#### Scenario: review-pr uses pre-flight skill

- **GIVEN** the `/review-pr` command is invoked on PR #42
- **AND** Step 3 has fetched CI check results
- **WHEN** Step 4 executes
- **THEN** the agent loads the `pre-flight` skill
- **AND** runs pre-flight checks in `ci-aware` mode with
  the CI check results from Step 3
- **AND** skips tools already verified by CI
- **AND** records results for use in Step 8 (AI review)

### Requirement: FR-010 /unleash Step 5 and Phase Checkpoint

`/unleash` Step 5 CI command derivation and phase checkpoint
execution MUST be replaced with references to the pre-flight
skill in `hard-gate` mode.

Previously: Step 5 contained inline logic to read
`.github/workflows/` and derive build/test commands
(lines 330-335), and the phase checkpoint ran those commands
(lines 449-452 of `unleash.md`).

#### Scenario: unleash uses pre-flight skill

- **GIVEN** the `/unleash` command reaches Step 5
- **WHEN** it derives build/test commands
- **THEN** the agent loads the `pre-flight` skill
- **AND** uses workflow parsing to discover CI commands
- **AND** runs the phase checkpoint in `hard-gate` mode
- **AND** stops on failure (same behavior as before)

### Requirement: FR-011 Scaffold Sync

All modified command files and the new skill file MUST be
synced to their corresponding copies in
`internal/scaffold/assets/`. The new skill file MUST be
added to `expectedAssetPaths` in `scaffold_test.go`.
Existing drift detection tests MUST pass after sync.

#### Scenario: Command scaffold copies match source

- **GIVEN** `review-council.md`, `review-pr.md`, and
  `unleash.md` have been updated in `.opencode/commands/`
- **WHEN** the scaffold copies are synced
- **THEN** `internal/scaffold/assets/opencode/commands/review-council.md`
  matches `.opencode/commands/review-council.md`
- **AND** `internal/scaffold/assets/opencode/commands/review-pr.md`
  matches `.opencode/commands/review-pr.md`
- **AND** `internal/scaffold/assets/opencode/commands/unleash.md`
  matches `.opencode/commands/unleash.md`
- **AND** drift detection tests pass

#### Scenario: Skill scaffold copy matches source

- **GIVEN** `.opencode/skills/pre-flight/SKILL.md` has been
  created
- **WHEN** the scaffold copy is synced
- **THEN** `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
  matches `.opencode/skills/pre-flight/SKILL.md`
- **AND** `expectedAssetPaths` in `scaffold_test.go` includes
  the new skill path
- **AND** drift detection tests pass

## REMOVED Requirements

None — this change consolidates existing behavior without
removing any capabilities. The unified detection approach
may discover additional tools compared to individual
command implementations; this is expected and intentional
(see design decision D2).

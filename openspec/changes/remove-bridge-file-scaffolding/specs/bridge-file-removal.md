# Delta Spec: Remove Bridge File Scaffolding

## ADDED Requirements

### Requirement: Scaffold No-Bridge-File Guarantee

After this change is applied, `uf init` MUST NOT create `CLAUDE.md` or
`.cursorrules` in the target directory under any invocation. This guarantee
MUST be enforced by an automated regression test that verifies absence of
both files after a full `Run()` call.

#### Scenario: uf init on a clean directory

- **GIVEN** an empty target directory with no pre-existing files
- **WHEN** `uf init` is executed against that directory
- **THEN** `CLAUDE.md` MUST NOT exist in the target directory
- **THEN** `.cursorrules` MUST NOT exist in the target directory

#### Scenario: uf init on a directory with pre-existing bridge files

- **GIVEN** a target directory containing `CLAUDE.md` and `.cursorrules`
  (created by a prior invocation or manually)
- **WHEN** `uf init` is executed against that directory
- **THEN** the existing `CLAUDE.md` and `.cursorrules` MUST NOT be modified
  or deleted by `uf init`
- **THEN** `uf init` MUST NOT report any result referencing bridge files in
  its summary output

### Requirement: Doctor No-Bridge-File-Check Guarantee

After this change is applied, `uf doctor` MUST NOT produce any
`CheckResult` with a `Name` field containing "Bridge", "CLAUDE.md", or
".cursorrules" from the Agent Context check group. This guarantee MUST
be enforced by an automated regression test using name-based assertions
(not count-based).

#### Scenario: uf doctor on a directory without bridge files

- **GIVEN** a project directory that does not contain `CLAUDE.md` or
  `.cursorrules`
- **WHEN** `uf doctor` runs the Agent Context check group
- **THEN** no `CheckResult` with a `Name` containing "Bridge" SHALL be
  returned
- **THEN** the `uf doctor` exit code and overall result MUST NOT be
  affected by the absence of bridge files

#### Scenario: uf doctor on a directory with bridge files present

- **GIVEN** a project directory that contains `CLAUDE.md` and
  `.cursorrules` (from a prior `uf init` or manually created)
- **WHEN** `uf doctor` runs the Agent Context check group
- **THEN** no `CheckResult` referencing those files SHALL be returned
- **THEN** the presence of bridge files MUST NOT affect any check result

## MODIFIED Requirements

### Requirement: Agent Context Check Count

Previously: The `checkAgentContext` function produced 13 `CheckResult`
values in the full-pass scenario (when `.specify/memory/constitution.md`,
a `specs/` directory, and an `openspec/config.yaml` are present in the
target directory).

After this change: `checkAgentContext` MUST produce 11 `CheckResult`
values in the same full-pass scenario. The reduction of 2 corresponds to
the removal of the `Bridge: CLAUDE.md` and `Bridge: .cursorrules` checks.

#### Scenario: Full-pass agent context check

- **GIVEN** a project directory with a valid `AGENTS.md`, all Tier 1
  sections present, a `.specify/memory/constitution.md`, and a `specs/`
  directory
- **WHEN** `checkAgentContext` is called
- **THEN** exactly 11 `CheckResult` values MUST be returned
- **THEN** none of the results SHALL have a `Name` containing "Bridge"

### Requirement: uf.agent-brief Command Scope

Previously: The `/uf.agent-brief` command described bridge file management
(creation of `CLAUDE.md` and `.cursorrules`) as part of its Step 5 workflow,
listed bridge file checks in its output table, and included bridge files in
its guardrails as files the agent is permitted to modify.

After this change: The `/uf.agent-brief` command MUST NOT reference bridge
file creation, bridge file checks, or bridge files in its guardrails. The
command description MUST NOT mention `CLAUDE.md` or `.cursorrules`. Both
the live command file (`.opencode/commands/uf.agent-brief.md`) and the
embedded scaffold asset (`internal/scaffold/assets/opencode/commands/
uf.agent-brief.md`) MUST be updated in the same change and MUST remain
identical in content.

#### Scenario: uf.agent-brief run after removal

- **GIVEN** a project that does not have `CLAUDE.md` or `.cursorrules`
- **WHEN** `/uf.agent-brief` is invoked
- **THEN** no step SHALL reference bridge file creation or verification
- **THEN** no output table row SHALL reference "Bridge: CLAUDE.md" or
  "Bridge: .cursorrules"
- **THEN** the guardrails MUST NOT list `CLAUDE.md` or `.cursorrules`
  as files the command manages

## REMOVED Requirements

### Requirement: Bridge File Scaffold (ensureCLAUDEmd)

**Removed.** `uf init` SHALL NOT create or update `CLAUDE.md`. The
`ensureCLAUDEmd` function, `buildCLAUDEmdBlock` helper, and
`claudemdMarker` constant are removed. No replacement mechanism is
provided — OpenCode's native `.opencode/` discovery makes bridge files
unnecessary for the supported platform.

### Requirement: Bridge File Scaffold (ensureCursorrules)

**Removed.** `uf init` SHALL NOT create or update `.cursorrules`. The
`ensureCursorrules` function, `buildCursorrulesBlock` helper, and
`cursorrulesMarker` constant are removed. The `replaceManagedBlock`
helper is also removed as it has no remaining callers after this change.

### Requirement: Bridge File Doctor Check

**Removed.** `uf doctor` SHALL NOT check for the presence or content of
`CLAUDE.md` or `.cursorrules`. The `checkBridgeFile` function is removed.
No replacement check is provided — the presence or absence of bridge files
is no longer a project health signal for supported (OpenCode) users.

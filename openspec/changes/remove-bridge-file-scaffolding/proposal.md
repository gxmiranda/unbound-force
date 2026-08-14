## Why

`uf init` currently scaffolds two bridge files — `CLAUDE.md` and
`.cursorrules` — to help users of Claude Code and Cursor IDE reference
the project's AGENTS.md and convention packs. These files were introduced
as an early experiment in cross-tool compatibility.

The team has decided that OpenCode is the only officially supported
platform (see ADR-001: `docs/decisions/001-opencode-over-claude-cli.md`).
Claude Code and Cursor are not officially supported, and the bridge file
experiment is being retired. The bridge scaffolding now creates three
problems:

1. **False doctor failures on clean setups.** `uf doctor` checks for
   `CLAUDE.md` and `.cursorrules` and emits `Warn` when they are absent.
   On a fresh clone of any repo that was not initialized with bridge file
   support enabled, these warnings appear immediately and confusingly.
   This was the root cause reported in issue #212.

2. **Dead code surface.** Five functions (`ensureCLAUDEmd`,
   `ensureCursorrules`, `buildCLAUDEmdBlock`, `buildCursorrulesBlock`,
   `replaceManagedBlock`) and two constants (`claudemdMarker`,
   `cursorrulesMarker`) exist solely to support a feature the team has
   decided to remove. The corresponding `checkBridgeFile` function in
   the doctor likewise has no remaining purpose.

3. **Stale documentation.** `QUICKSTART.md`, `docs/architecture.md`,
   and the `/uf.agent-brief` command asset all describe bridge files as
   first-class features, misleading new adopters.

Removing the bridge scaffolding eliminates ~270 lines of scaffold code,
~30 lines of doctor code, and ~20 tests, and cleans up all stale
documentation references. This is tracked as child issue #459 of #212.

## What Changes

- **`uf init`** no longer creates or updates `CLAUDE.md` or `.cursorrules`
  in the target project directory.
- **`uf doctor`** no longer checks for `CLAUDE.md` or `.cursorrules`
  (`checkBridgeFile` removed; agent context check count goes from 13 to 11).
- **Dead code removed**: `ensureCLAUDEmd`, `ensureCursorrules`,
  `buildCLAUDEmdBlock`, `buildCursorrulesBlock`, `replaceManagedBlock`,
  `claudemdMarker`, `cursorrulesMarker` in `internal/scaffold/scaffold.go`.
- **Dead code removed**: `checkBridgeFile` in `internal/doctor/checks.go`.
- **Tests deleted**: 18 scaffold tests (`TestEnsureCLAUDEmd_*`,
  `TestEnsureCursorrules_*`, `TestReplaceManagedBlock_*`) and 2 doctor
  tests (`TestCheckAgentContext_BridgeCLAUDEmd`,
  `TestCheckAgentContext_BridgeCursorrules`).
- **Tests added**: `TestRun_DoesNotCreateBridgeFiles` (scaffold regression
  guard) and `TestCheckAgentContext_NoBridgeFileResults` (doctor regression
  guard).
- **Tests updated**: `TestDoctorRun_AllPass` (remove bridge fixtures),
  `TestCheckAgentContext_FullPass` (update count 13→11, update comment).
- **Root files deleted**: `CLAUDE.md` and `.cursorrules` from the
  `unbound-force` meta-repo root (these were artifacts of a prior `uf init`
  run).
- **Docs cleaned up**: `QUICKSTART.md` (lines 123, 149–150),
  `docs/architecture.md` (lines 289–295), GoDoc and inline comments in
  `checks.go`.
- **`/uf.agent-brief` updated**: Bridge file references removed from Step 5
  description, output table, and guardrails section — in both the live copy
  (`.opencode/commands/uf.agent-brief.md`) and the embedded scaffold asset
  (`internal/scaffold/assets/opencode/commands/uf.agent-brief.md`).
- **GAZE baseline refreshed**: `gaze baseline update` run after function
  deletion to prevent false CRAP regression alerts.

## Capabilities

### New Capabilities

None. This is a removal-only change.

### Modified Capabilities

- `uf init`: No longer creates or updates `CLAUDE.md` or `.cursorrules`.
  All other scaffold outputs are unchanged.
- `uf doctor` (Agent Context group): Bridge file checks (#11, #12) removed.
  Check count is 11 in the full-pass scenario.
- `/uf.agent-brief`: Bridge File Verification step removed from output;
  command description updated to reflect OpenCode-only scope.

### Removed Capabilities

- **Bridge file scaffolding**: `uf init` will no longer create or manage
  `CLAUDE.md` or `.cursorrules`. Existing bridge files in user repos are
  not deleted — they simply stop being created or updated.
- **Bridge file doctor checks**: `uf doctor` will no longer warn about
  absent or misconfigured `CLAUDE.md` / `.cursorrules`.

## Impact

**Files modified:**

| File | Change |
|---|---|
| `internal/scaffold/scaffold.go` | Remove 7 functions/constants; simplify `Run()` |
| `internal/scaffold/scaffold_test.go` | Delete 18 tests, add 1 new test |
| `internal/doctor/checks.go` | Remove `checkBridgeFile` + 2 call sites + comments |
| `internal/doctor/doctor_test.go` | Delete 2, update 2, add 1 new test |
| `.opencode/commands/uf.agent-brief.md` | Remove bridge file sections |
| `internal/scaffold/assets/opencode/commands/uf.agent-brief.md` | Same (embedded asset copy) |
| `QUICKSTART.md` | Remove bridge file references |
| `docs/architecture.md` | Remove "Bridge files" section |
| `CLAUDE.md` (root) | Delete |
| `.cursorrules` (root) | Delete |
| `.gaze/baseline.json` | Refresh via `gaze baseline update` |

**Users with existing bridge files**: Unaffected. `uf init` will stop
creating/updating them, but will not delete them from existing projects.

**CI**: No CI workflow references bridge files. No pipeline changes needed.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change removes internal scaffold and doctor code. It does not affect
inter-hero communication, artifact envelope formats, or any artifact
produced for consumption by other heroes. No artifact-based communication
channel is altered.

### II. Composability First

**Assessment**: PASS

Bridge files were intended to make the scaffold useful for non-OpenCode
users, but the team has determined that OpenCode is the only supported
platform. Removing bridge file scaffolding does not reduce standalone
usability for the supported platform. `uf init` remains independently
deployable and useful without any other hero. No mandatory dependencies
are introduced or removed.

### III. Observable Quality

**Assessment**: PASS

The `uf doctor` check count changes from 13 to 11 in the full-pass
scenario. The check group remains machine-parseable JSON and the removal
is reflected in updated test assertions. The GAZE baseline is refreshed
to keep CRAP regression analysis accurate. All other observable outputs
are unchanged.

### IV. Testability

**Assessment**: PASS

Two new regression-guard tests are added:
- `TestRun_DoesNotCreateBridgeFiles`: asserts `uf init` does not create
  `CLAUDE.md` or `.cursorrules` after removal.
- `TestCheckAgentContext_NoBridgeFileResults`: asserts `uf doctor`'s agent
  context check group produces no result with a name containing "Bridge",
  "CLAUDE", or "cursorrules".

Both tests use `t.TempDir()` for isolation and are compatible with
`-race -count=1`. All deleted tests targeted functions being removed, so
no coverage regression occurs. The net test count decreases by 18 (20
deleted, 2 added), but coverage on the remaining scaffold and doctor logic
is unaffected.

### V. Security by Default

**Assessment**: PASS

No security-sensitive logic is removed. The functions being deleted use
only stdlib file I/O with `filepath.Join` (no path traversal risk), no
credentials or secrets, and no shell execution. No governance gate is
weakened (`checkBridgeFile` was `Warn`-severity only). The removal reduces
the attack surface by eliminating ~300 lines of code that manipulate files
in the user's project directory.

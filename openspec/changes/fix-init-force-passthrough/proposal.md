## Why

`uf init --force` does not re-initialize Group B sub-tools
(specify, replicator, openspec, gaze) or pass `--force`
through to their CLIs. This breaks the recovery path for
partial or interrupted initialization and prevents force-
refreshing sub-tool configuration after upgrades.

Two defects exist in `internal/scaffold/scaffold.go`:

1. **`initSimpleTool` ignores `opts.Force`** (line 1448-1452):
   Returns `nil` unconditionally when the sentinel exists,
   unlike `initDewey` which explicitly checks `opts.Force`
   and re-indexes when forced.

2. **`--force` not passed to sub-tool CLIs** (line 1336-1347):
   The `simpleTool` struct has no `forceFlag` field, so
   sub-tools like `specify init` fail in non-empty
   directories because they never receive `--force`.

This is tracked as GitHub issue #479.

## What Changes

The `simpleTool` struct and `initSimpleTool` function gain
force-mode awareness, matching the existing `initDewey`
behavior.

## Capabilities

### New Capabilities
- None

### Modified Capabilities
- `uf init --force`: Correctly re-initializes all sub-tools
  (Group A and Group B) and passes tool-specific force flags
  through to each sub-tool CLI

### Removed Capabilities
- None

## Impact

- **Files**: `internal/scaffold/scaffold.go`,
  `internal/scaffold/scaffold_test.go`
- **Behaviors**: `uf init --force` will re-run sub-tool init
  commands with their respective `--force` flags instead of
  silently skipping when sentinels exist
- **Spec alignment**: Closes the gap between spec 017 FR-007
  (`uf init --force` MUST overwrite stale config) and spec
  027 FR-011 (idempotency) — force mode is an explicit
  override of the idempotency guard

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change does not affect artifact-based communication
between heroes. It modifies internal initialization
plumbing within the `uf` CLI.

### II. Composability First

**Assessment**: PASS

This fix restores composability. Currently, `uf init --force`
fails to re-initialize independently-installable heroes
(specify, openspec, gaze, replicator), forcing users to
bypass `uf init` and invoke each tool's init command
directly. The fix ensures the unified orchestrator
correctly propagates force semantics to all composed tools.

### III. Observable Quality

**Assessment**: PASS

The current behavior produces a silent skip — no output,
no error, no indication that sub-tools were skipped during
force re-initialization. The fix ensures sub-tools are
re-run when `--force` is specified, producing visible
init results (success or failure) for each tool.

### IV. Testability

**Assessment**: PASS

The fix adds test coverage for the force path in
`initSimpleTool`, closing a parity gap with `initDewey`
which already has 4 dedicated force-related tests. All
new tests will use `t.TempDir()` for isolation and
standard library assertions only.

### V. Security by Default

**Assessment**: PASS

The fix appends static, developer-defined flag strings
(e.g., `"--force"`) to the args slice passed to
`exec.Command`. No untrusted input is involved. No new
dependencies are added.

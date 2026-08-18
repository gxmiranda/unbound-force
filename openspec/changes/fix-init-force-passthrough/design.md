## Context

`uf init --force` correctly re-initializes Dewey (Group A)
via `initDewey`, which explicitly checks `opts.Force` and
re-indexes when the workspace already exists. However,
Group B tools (specify, replicator, openspec, gaze) use
`initSimpleTool`, which ignores `opts.Force` entirely —
returning nil when the sentinel exists regardless of the
force flag.

The `simpleTool` struct also lacks a mechanism to declare
tool-specific force flags, so even if the sentinel guard
were bypassed, sub-tool CLIs like `specify init` would
fail in non-empty directories without receiving `--force`.

This is a bug fix, not a new feature. The design aligns
`initSimpleTool` with the existing `initDewey` pattern.

## Goals / Non-Goals

### Goals
- `initSimpleTool` respects `opts.Force` when sentinel
  exists, matching `initDewey` behavior
- Each `simpleTool` entry can declare its tool-specific
  force flag (or empty for idempotent tools)
- Force re-initialization produces a distinct action label
  (`"re-initialized"`) for observable output
- Test coverage parity with `initDewey` force tests

### Non-Goals
- Changing `initDewey` behavior (already correct)
- Adding new sub-tools to the `simpleTools` slice
- Modifying the `subToolResult` struct itself
- Adding verbose/debug logging beyond the existing `logf`
  pattern

## Decisions

### D1: Add `forceFlag` field to `simpleTool` struct

The `simpleTool` struct gains a `forceFlag string` field.
Each tool entry declares its own flag value. This keeps
force-flag knowledge local to the tool definition rather
than hardcoding flag names in `initSimpleTool`.

**Rationale**: Mirrors how `args` already captures
tool-specific extra arguments. Different tools may use
different flag names (though all current tools use
`--force` or none). The struct-field approach is
extensible without modifying the init function.

**Constitution alignment**: Composability First — each
tool's integration contract is self-contained in its
struct definition.

### D2: Conditional sentinel bypass in `initSimpleTool`

The sentinel-exists check at line 1450-1451 becomes:

```
if sentinel exists AND NOT opts.Force:
    return nil
```

When `opts.Force` is true and sentinel exists, execution
falls through to the init command. If `forceFlag` is
non-empty, it is appended to the args slice.

**Rationale**: Direct pattern match with `initDewey`
lines 1387-1441, which uses the same conditional
structure. Minimal change surface.

### D3: Pass `simpleTool` struct to `initSimpleTool`

Refactor `initSimpleTool` to accept the `simpleTool`
struct directly instead of individual parameters. This
avoids parameter proliferation when adding `forceFlag`
and improves readability.

Current signature:
```go
func initSimpleTool(opts *Options, name, sentinel,
    resultName, label string, extraArgs []string,
    logf func(string, ...interface{})) *subToolResult
```

New signature:
```go
func initSimpleTool(opts *Options, tool simpleTool,
    logf func(string, ...interface{})) *subToolResult
```

**Rationale**: The current function takes 7 parameters,
all of which are fields of `simpleTool` except `opts`
and `logf`. Passing the struct reduces parameter count
to 3 and makes the call site cleaner.

### D4: Distinct action label for force re-initialization

When `initSimpleTool` re-runs a tool due to `opts.Force`,
the result uses action `"re-initialized"` instead of
`"initialized"`. This matches the `initDewey` pattern
where force produces `"re-indexed"` (line 1438).

**Rationale**: Observable Quality — the output
distinguishes between first-time init and force
re-initialization, giving users and downstream
consumers clear signal about what happened.

## Risks / Trade-offs

### Risk: Sub-tool `--force` semantics vary

Each sub-tool may interpret `--force` differently. For
example, `specify init --force` overwrites existing files,
while `gaze init --force` may have different behavior.
This is acceptable because `uf init --force` already
implies the user wants to override existing state for
all tools. The per-tool `forceFlag` field allows each
tool to declare its own flag semantics.

**Mitigation**: The `forceFlag` field is optional (empty
string means no flag). Tools that are naturally idempotent
(like replicator, which overwrites without needing a
flag) do not declare a force flag.

### Risk: Signature change to `initSimpleTool`

Changing the function signature from individual parameters
to a struct parameter requires updating the call site in
the `for` loop (line 1361-1362). This is a single call
site, so the risk is minimal.

**Mitigation**: The call site change is mechanical and
covered by existing tests that exercise the init loop.

### Trade-off: No logging for force-skip-sentinel

When `opts.Force` is true and sentinel exists, we re-run
the tool without logging "Sentinel exists, forcing
re-initialization" — we simply run the init command with
the force flag. This keeps the log output clean and
matches the `initDewey` pattern, which also does not log
the force bypass explicitly.

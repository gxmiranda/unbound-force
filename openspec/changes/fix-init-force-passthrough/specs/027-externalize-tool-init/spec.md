## ADDED Requirements

### Requirement: FR-014 — Force flag passthrough for Group B tools

`initSimpleTool` MUST check `opts.Force` before applying
the sentinel-exists guard. When `opts.Force` is true and
the sentinel path exists, the function MUST re-run the
sub-tool's init command instead of returning nil.

When `opts.Force` is true, the sentinel path exists, and
the `simpleTool` entry declares a non-empty `forceFlag`,
`initSimpleTool` MUST append that flag to the args slice
passed to `ExecCmd`.

When `opts.Force` is true, the sentinel path exists, and
the `simpleTool` entry has an empty `forceFlag`,
`initSimpleTool` MUST re-run the init command without an
additional force flag (the tool is assumed idempotent).

On first-time initialization (sentinel does not exist),
the `forceFlag` MUST NOT be appended regardless of
`opts.Force`. The function MUST proceed with the standard
init args and report action `"initialized"`.

#### Scenario: Force re-initializes specify with --force flag

- **GIVEN** the `.specify` sentinel directory exists
- **WHEN** the user runs `uf init --force`
- **THEN** `initSimpleTool` MUST invoke
  `specify init --here --integration opencode --offline --force`
- **AND** the result MUST report action "re-initialized"

#### Scenario: Force re-initializes replicator without force flag

- **GIVEN** the `.uf/replicator` sentinel directory exists
- **WHEN** the user runs `uf init --force`
- **THEN** `initSimpleTool` MUST invoke `replicator init`
  (no additional force flag, since replicator declares no
  forceFlag)
- **AND** the result MUST report action "re-initialized"

#### Scenario: First-time init with force (no sentinel)

- **GIVEN** the `.specify` sentinel directory does NOT exist
- **WHEN** the user runs `uf init --force`
- **THEN** `initSimpleTool` MUST invoke
  `specify init --here --integration opencode --offline`
  (without `--force`, since sentinel did not exist)
- **AND** the result MUST report action "initialized"

#### Scenario: Force re-init failure reports error

- **GIVEN** the `.specify` sentinel directory exists
- **WHEN** the user runs `uf init --force` and `ExecCmd`
  returns an error
- **THEN** the result MUST report action "failed"
- **AND** the error detail MUST be preserved

#### Scenario: No-force skips existing sentinel (unchanged)

- **GIVEN** the `.specify` sentinel directory exists
- **WHEN** the user runs `uf init` (without `--force`)
- **THEN** `initSimpleTool` MUST return nil
- **AND** the tool MUST NOT be re-initialized

### Requirement: FR-015 — simpleTool struct forceFlag field

The `simpleTool` struct MUST include a `forceFlag` field
of type `string`. Each tool entry in the `simpleTools`
slice MUST declare its tool-specific force flag:

| Tool     | forceFlag |
|----------|-----------|
| specify  | `--force` |
| openspec | `--force` |
| gaze     | `--force` |
| replicator | (empty) |

An empty `forceFlag` means the tool does not require a
force flag for re-initialization.

### Requirement: FR-016 — Re-initialized action label

When `initSimpleTool` re-runs a tool due to `opts.Force`,
the returned `subToolResult` MUST use action
`"re-initialized"` to distinguish force re-initialization
from first-time initialization (action `"initialized"`).

#### Scenario: Action label distinguishes force from first-time

- **GIVEN** the sentinel path exists and `opts.Force` is true
- **WHEN** `initSimpleTool` completes successfully
- **THEN** the `subToolResult.action` MUST be
  `"re-initialized"`

## MODIFIED Requirements

### Requirement: FR-011 — Idempotency (clarification)

Each tool delegation MUST be idempotent — if the tool's
directory already exists **and** `opts.Force` is false,
the step is skipped. When `opts.Force` is true, the
idempotency guard is bypassed and the tool is
re-initialized.

Previously: "Each tool delegation MUST be idempotent —
if the tool's directory already exists, the step is
skipped." (No mention of force override.)

## REMOVED Requirements

None.

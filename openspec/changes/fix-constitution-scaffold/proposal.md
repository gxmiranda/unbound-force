## Why

`uf init` delegates constitution creation to `specify init`, which
requires interactive TTY input (arrow-key selection for AI
assistant type). When called programmatically via `opts.ExecCmd()`
in `initSimpleTool()` (scaffold.go:1616), it fails silently.
Even when `specify init` succeeds (run manually afterwards), it
produces a template full of `[PLACEHOLDER]` tokens that require a
separate `/speckit.constitution` session to fill in.

The result: new projects start without a working constitution.
`uf doctor` flags the gap. The Guard agent has no document to
check alignment against. The scaffolded project is not usable for
spec-driven development until the user manually runs an additional
interactive workflow.

Fixes #213.

## What Changes

Embed a starter constitution as a scaffolded asset, following the
same `embed.FS` pattern used for all other `uf init` files. The
starter constitution is placed at `.specify/memory/constitution.md`
during scaffold and contains the five org-level principles from
the Unbound Force constitution, adapted with generic language so
any project can use them immediately.

The `specify init` call remains — it scaffolds templates, config,
and scripts that are still needed. The constitution file is written
*before* `specify init` runs (during the embedded asset walk), so
`specify init` finds `.specify/` already partially populated and
skips its own constitution template. If `specify init` is not
available, the project still gets a working constitution.

Post-scaffold hints are updated to recommend `/speckit.constitution`
as a *refinement* step ("customize your constitution") rather than
a prerequisite ("create your constitution").

## Capabilities

### New Capabilities
- `starter-constitution`: `uf init` scaffolds a working starter
  constitution at `.specify/memory/constitution.md` with the five
  org principles (Autonomous Collaboration, Composability First,
  Observable Quality, Testability, Security by Default) using
  generic project language.

### Modified Capabilities
- `post-scaffold-hints`: Next-step guidance changes from "create
  your project constitution" to "customize your project
  constitution" since a working default now exists.
- `uf-doctor-constitution-check`: No code change needed — the
  existing check at `doctor/checks.go:1765` already looks for
  `.specify/memory/constitution.md` and will pass once the file
  is scaffolded.

### Removed Capabilities
- None.

## Impact

- **internal/scaffold/assets/**: New embedded file at
  `opencode/specify/memory/constitution.md` (mapped to
  `.specify/memory/constitution.md` in the target directory via
  the existing asset path prefix conventions, or via explicit
  path mapping in the scaffold walk).
- **internal/scaffold/scaffold.go**: Minor change to
  post-scaffold hint text (lines 1736, 1739). Possible addition
  of path mapping if the asset prefix convention does not
  naturally map to `.specify/`.
- **internal/scaffold/scaffold_test.go**: New test cases for
  constitution file scaffolding; updated expected file lists;
  updated hint text assertions.
- **internal/doctor/**: No changes — existing checks will pass
  automatically once the file is present.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

The starter constitution is a self-contained artifact written to
a well-known location (`.specify/memory/constitution.md`). It
requires no synchronous interaction with other tools. The Guard
agent, doctor checks, and spec workflows can all consume it
independently.

### II. Composability First

**Assessment**: PASS

This change strengthens composability. Currently, `uf init`
depends on `specify` being installed *and* interactive to produce
a constitution. After this change, the constitution is scaffolded
from embedded assets — no external tool dependency required. The
`specify init` integration remains optional and additive.

### III. Observable Quality

**Assessment**: PASS

The starter constitution is a versioned Markdown file with clear
principle numbering. It follows the same format as the org
constitution, making it parseable by existing tooling. The
scaffold result reports the file as created/skipped/updated in
the standard `Result` struct.

### IV. Testability

**Assessment**: PASS

The scaffold engine already has comprehensive test infrastructure
(`scaffold_test.go`) with stub I/O functions. New tests will
verify the constitution file is created, respects `Force` and
`DivisorOnly` flags, and is included in the expected file lists.
All tests run in isolation with `t.TempDir()`.

### V. Security by Default

**Assessment**: N/A

This change writes a Markdown document from embedded assets. No
external inputs, network access, shell execution, or privilege
changes are involved. File permissions follow the existing
scaffold defaults (0o644).

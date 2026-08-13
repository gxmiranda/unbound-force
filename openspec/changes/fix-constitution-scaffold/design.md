## Context

`uf init` delegates constitution creation to `specify init`, which
requires interactive TTY input. When called programmatically via
`opts.ExecCmd()` in `initSimpleTool()` (scaffold.go:1616), it fails
silently. Even when run manually, `specify init` produces a template
full of `[PLACEHOLDER]` tokens. New projects start without a working
constitution, breaking `uf doctor`, Guard agent alignment checks,
and spec-driven workflows. See proposal.md for full motivation.

Fixes #213.

## Goals / Non-Goals

### Goals
- `uf init` scaffolds a working constitution at
  `.specify/memory/constitution.md` from embedded assets
- No external tool dependency required for a usable constitution
- Existing user-customized constitutions are preserved on re-run
- Post-scaffold hints guide users to customize, not create
- `uf doctor` constitution check passes immediately after scaffold

### Non-Goals
- Removing the `specify` tool from `simpleTools` — it remains
  available for users who install it separately and want its
  full template/config/script scaffolding
- Customizing the starter constitution per-project at scaffold
  time (that is what `/speckit.constitution` does post-scaffold)
- Changing the `uf doctor` check logic — it already looks for
  `.specify/memory/constitution.md` and will pass automatically

## Decisions

### D1: New `specify/` asset prefix

Add a `specify/` directory to `internal/scaffold/assets/` with
the starter constitution at `specify/memory/constitution.md`.
Extend `mapAssetPath()` to map `specify/` to `.specify/` using
the same `"." + relPath` pattern as `opencode/` and
`devcontainer/`. Add `"specify/"` to `knownAssetPrefixes`.

**Rationale**: Follows the established embed.FS pattern. No new
mechanisms needed — the existing asset walk, version markers,
and file ownership checks apply automatically.

### D2: Sentinel interaction — `specify init` is correctly skipped

The `simpleTools` entry for `specify` uses sentinel `.specify`.
Scaffolding `.specify/memory/constitution.md` creates the
`.specify/` directory, so `os.Stat(".specify")` succeeds and
`initSimpleTool()` returns nil (skips `specify init`).

This is the **correct** behavior:
- `specify init` fails silently in non-interactive mode (the bug)
- The constitution file — the critical artifact — is now provided
  by the embedded asset
- Users who want the full Speckit framework can run `specify init`
  manually after `uf init`

**Alternative considered**: Change the sentinel to a more specific
path (e.g., `.specify/config.yaml`). Rejected because it would
cause `specify init` to run after our scaffold, potentially
overwriting the starter constitution with a `[PLACEHOLDER]`
template. The sentinel serving as a skip gate is correct.

### D3: Constitution is user-owned, not tool-owned

The `isToolOwned()` function does NOT return true for `specify/`
paths (only `openspec/schemas/`, `opencode/commands/`,
`opencode/skills/`, and canonical convention packs). This means:

- First run: file is created (reported as "created")
- Re-run without `--force`: file is preserved (reported as
  "skipped") — user customizations are safe
- Re-run with `--force`: file is overwritten (reported as
  "overwritten") — explicit user intent

This matches the constitution's nature as a project-specific
document that users are expected to customize.

### D4: Starter constitution content

The embedded starter constitution contains the five org-level
principles (Autonomous Collaboration, Composability First,
Observable Quality, Testability, Security by Default) adapted
with generic project language. It follows the same Markdown
format and principle numbering as the org constitution at
`.specify/memory/constitution.md` so existing tooling (Guard
agent, constitution-check command) can parse it without changes.

The file includes a header comment indicating it is a starter
template and directing users to `/speckit.constitution` for
customization.

### D5: DivisorOnly mode excludes constitution

The `isDivisorAsset()` function does not match `specify/` paths.
In `--divisor-only` mode, the constitution is not scaffolded.
This is correct — Divisor-only deployments target existing
projects that should already have their own constitution.

### D6: Post-scaffold hint text update

Lines 1736 and 1739 in `scaffold.go` change from:
- "Run /speckit.constitution to create your project constitution"

To:
- "Run /speckit.constitution to customize your project constitution"

The hint position in the step list remains unchanged.

## Risks / Trade-offs

### R1: `specify init` skipped on fresh projects

Since the embedded asset creates `.specify/`, the sentinel
check will skip `specify init` on fresh projects. Users who
want Speckit templates and scripts must run `specify init`
manually. This is acceptable because:
- `specify init` was already failing silently (the bug)
- The constitution was the only critical output
- Speckit templates/config are a separate concern

### R2: Starter constitution may not fit all projects

The five org principles are generic enough for most projects
but may not be relevant for all (e.g., a pure documentation
repo may not need Testability). Mitigation: the starter file
includes a clear header directing users to customize via
`/speckit.constitution`. The principles serve as a reasonable
starting point, not a final document.

### R3: Version marker drift

The constitution file receives a version marker (it is `.md`).
On `uf init` upgrades, because the file is user-owned, updated
starter content will NOT be pushed to existing projects (they
keep their customized version). This is intentional — the
constitution is a living document, not a tool-managed template.

### R4: Existing `[PLACEHOLDER]` constitutions preserved on upgrade

Users who previously ran `specify init` manually may have a
`.specify/memory/constitution.md` full of `[PLACEHOLDER]`
tokens. When they upgrade `uf` and re-run `uf init`, the
user-owned file logic preserves the broken file (reported as
"skipped"). This is correct behavior — we do not silently
overwrite user files — but the user retains a non-functional
constitution. Mitigation: users can run `uf init --force` to
replace it, or run `/speckit.constitution` to customize it.
This is a pre-existing concern, not introduced by this change.
<!-- scaffolded by uf vdev -->

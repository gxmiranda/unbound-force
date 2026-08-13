## Context

`uf init` calls `ensureCLAUDEmd()` and `ensureCursorrules()` unconditionally
in `internal/scaffold/scaffold.go:Run()` (lines 234–235). Each call reads
the target file, checks for the managed block marker, and writes the block if
missing or stale. `uf doctor` calls `checkBridgeFile()` twice (lines
1811–1812) from `checkAgentContext()`, producing `Warn`-severity results when
`CLAUDE.md` or `.cursorrules` are absent.

The team has decided OpenCode is the only supported platform. Claude Code and
Cursor bridge files were an early experiment that is being retired. The
removal eliminates ~300 lines of production code, ~20 test functions, and
several stale documentation sections.

## Goals / Non-Goals

### Goals

- Remove all production code for bridge file creation from `uf init`
- Remove all doctor checks for bridge files from `uf doctor`
- Delete root-level `CLAUDE.md` and `.cursorrules` from the meta-repo
- Update `/uf.agent-brief` (both live and embedded copies) to remove
  bridge file references
- Clean up stale documentation in `QUICKSTART.md` and `docs/architecture.md`
- Add regression-guard tests that enforce the removal cannot silently regress
- Refresh the GAZE CRAP baseline after function deletion

### Non-Goals

- Supporting Claude Code or Cursor via any alternative mechanism
- Adding a `--bridge-files` opt-in flag to `uf init`
- Deleting bridge files from existing user project repos (the tool stops
  creating them; it does not clean up existing ones)
- Changing any other scaffold outputs (`ensureGitignore`,
  `ensureAGENTSmdPackSection`, etc.)
- Updating `.specify/scripts/bash/update-agent-context.sh` — this speckit
  helper references `CLAUDE.md` (line 58) but is outside the `uf init` /
  `uf doctor` scope of this change. A separate follow-up chore should clean
  up the Claude Code agent type from this script.

## Decisions

### D-1: Delete, don't gate

Options considered:
- **Gate behind a flag**: Add `--bridge-files` to `uf init` to make
  creation opt-in rather than default.
- **Delete entirely**: Remove the code with no opt-in path.

**Decision**: Delete entirely. The team has stated these platforms are not
supported. A flag would imply ongoing support intent and add dead-weight
maintenance surface. The zero-waste mandate supports full deletion.

### D-2: Keep `collectDeployedPacks`, `filterEmptyCustomPacks`, `hasRuleContent`

These three helpers are called by `ensureCLAUDEmd` and `ensureCursorrules`,
but also by `ensureAGENTSmdPackSection` (line 1069). They have independent
test coverage (`TestCollectDeployedPacks_*`, `TestHasRuleContent_*`,
`TestFilterEmptyCustomPacks_*`). They MUST NOT be removed.

### D-3: Add name-based (not count-based) regression tests

The existing `TestCheckAgentContext_FullPass` count assertion (13→11) is a
weak proxy — it would pass even if the bridge checks were replaced by two
different checks. Two new tests are added with explicit name-based assertions:

- `TestRun_DoesNotCreateBridgeFiles` — asserts `CLAUDE.md` and `.cursorrules`
  do not exist in the target directory after `Run()`.
- `TestCheckAgentContext_NoBridgeFileResults` — asserts no `CheckResult`
  with a `Name` containing "Bridge", "CLAUDE", or "cursorrules" is returned
  from `checkAgentContext()`.

### D-4: Update both uf.agent-brief files atomically

`.opencode/commands/uf.agent-brief.md` (live, used in this repo) and
`internal/scaffold/assets/opencode/commands/uf.agent-brief.md` (embedded,
deployed by `uf init` to user projects) must be kept identical. They are
updated in the same commit. Drift between the two would cause
`uf init`-deployed projects to use stale bridge file instructions.

### D-5: Refresh GAZE baseline after deletion

`.gaze/baseline.json` contains entries for `buildCursorrulesBlock` and
`ensureCursorrules`. After deletion, the baseline will reference functions
that no longer exist, which will trigger false CRAP regression alerts in
`ci_crapload.yml`. Running `gaze baseline update` as a final step prevents
this. This is the last step in the task list.

### D-6: Delete root CLAUDE.md and .cursorrules

The `CLAUDE.md` and `.cursorrules` files at the repo root are managed
artifacts created by a prior `uf init` run. They should be deleted as
part of this change to be consistent with the removal of bridge file
scaffolding from `uf init`. Existing user repos that have these files
are unaffected — the tool stops creating them, but does not delete them.

## Risks / Trade-offs

| Risk | Likelihood | Mitigation |
|---|---|---|
| Developer re-adds bridge calls to `Run()` without noticing | Low | `TestRun_DoesNotCreateBridgeFiles` fails immediately |
| Developer re-adds `checkBridgeFile` without noticing | Low | `TestCheckAgentContext_NoBridgeFileResults` fails immediately |
| Stale GAZE baseline causes false CRAP alert in CI | Medium | `gaze baseline update` as final task step |
| User repos that relied on `uf init` to manage bridge files are broken | None | Removal is from scaffold only; existing files untouched |
| Live and embedded `uf.agent-brief.md` drift post-update | Low | Both updated atomically in the same commit; diff verified |

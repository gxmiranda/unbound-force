<!--
  [P] marks tasks eligible for parallel execution.
  Add [P] when a task: (a) touches different files from
  other [P] tasks in the group, (b) has no dependency
  on prior tasks in the group, (c) can safely execute
  without ordering constraints.
  Do NOT add [P] when tasks modify the same file —
  parallel workers will cause merge conflicts.
  Tasks without [P] run sequentially first, then [P]
  tasks run in parallel.
-->

## 1. Add Phase 2a: File-Scope Filter to SKILL.md

All tasks in this group modify the same file
(`~/.config/opencode/skills/pre-flight/SKILL.md`) and
MUST run sequentially.

- [x] 1.1 Add the file-scope mapping table after Phase 2
  (Tool Detection), before the Phase 3 heading. The new
  section is "Phase 2a: File-Scope Filter." Include the
  mapping table from FR-020:

  | Tool | In-scope file patterns |
  |------|----------------------|
  | `go test` | `*.go`, `go.mod`, `go.sum` |
  | `golangci-lint` | `*.go`, `go.mod`, `go.sum`, `.golangci.yml`, `.golangci.yaml` |
  | `ruff` | `*.py`, `ruff.toml`, `pyproject.toml` |
  | `pytest` | `*.py`, `pyproject.toml`, `setup.py`, `conftest.py` |
  | `yamllint` | `*.yml`, `*.yaml`, `.yamllint.yml`, `.yamllint.yaml` |
  | `make check` | _always in scope_ |
  | `pre-commit` | _always in scope_ |

  Include the branch diff computation instructions per
  FR-021:
  ```bash
  git diff --name-only \
    $(git merge-base HEAD origin/${DEFAULT_BRANCH})...HEAD
  ```
  Include the default branch detection reference (reuse
  Phase 4a logic, lines 218-249). Include the conservative
  fallback: if diff computation fails, skip the filter and
  run all tools.

  File: `~/.config/opencode/skills/pre-flight/SKILL.md`

- [x] 1.2 Modify the Phase 3 decision rules for hard-gate
  mode (lines 145-150) per modified FR-005. Change from
  "ALL detected and available tools" to "all detected and
  available tools that have in-scope files." Add a note
  that tools marked "SKIP — no in-scope files" in Phase 2a
  are excluded from the coverage matrix's "Run locally"
  decisions.

  File: `~/.config/opencode/skills/pre-flight/SKILL.md`

- [x] 1.3 Modify the ci-aware mode section (Phase 3
  decision rules, lines 128-141) per modified FR-006.
  Clarify that the CI coverage matrix is built only from
  tools that survived Phase 2a's file-scope filter. Tools
  removed by the filter are marked "SKIP — no in-scope
  files" and are excluded from CI coverage evaluation.

  File: `~/.config/opencode/skills/pre-flight/SKILL.md`

- [x] 1.4 Modify the Phase 4 soft-gate mode section
  (lines 198-200) per modified FR-007. Change from
  "Execute ALL detected and available tools" to "Execute
  all detected and available tools that have in-scope
  files in the branch diff."

  File: `~/.config/opencode/skills/pre-flight/SKILL.md`

- [x] 1.5 Update Phase 5 result format to include skip
  reporting per FR-023. Add "SKIP — no in-scope files"
  as a valid status in the Execution Results tables for
  all three modes. Add a skip count line to the Verdict
  sections: "- **Skipped — no in-scope files**: N tools".
  Include the distinct status in the coverage matrix
  "Run locally?" column.

  File: `~/.config/opencode/skills/pre-flight/SKILL.md`

## 2. Sync Scaffold Copy

This task depends on Group 1 being complete.

- [x] 2.1 Copy the updated SKILL.md to the scaffold
  assets directory. The user-level copy is the
  authoritative source — copy from user-level to
  scaffold, not the reverse. The scaffold copy MUST be
  identical to the user-level copy. Note: the scaffold
  copy currently has an older description; this sync
  will resolve that pre-existing drift.

  Source: `~/.config/opencode/skills/pre-flight/SKILL.md`
  Target: `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`

## 3. Verification

- [x] 3.1 Verify scaffold sync: diff the two SKILL.md
  copies and confirm they are identical.
  ```bash
  diff ~/.config/opencode/skills/pre-flight/SKILL.md \
    internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md
  ```

- [x] 3.2 [P] Verify constitution alignment:
  - **Observable Quality (III)**: Confirm skip status is
    distinct from pass status in both coverage matrix and
    execution results tables.
  - **Security by Default (V)**: Confirm tool config files
    are included in scope patterns. Confirm aggregate tools
    are always in scope. Confirm conservative fallback when
    diff is unavailable.
  - **Testability (IV)**: Confirm the mapping table is
    declarative and the diff command is deterministic.

- [x] 3.3 [P] Run existing tests to verify no regressions:
  ```bash
  go test -race -count=1 ./internal/scaffold/...
  ```
  Scaffold drift detection tests must pass with the
  updated asset.

- [x] 3.4 [P] Smoke test: run the pre-flight skill in
  hard-gate mode against the current branch (which
  modifies only Markdown files under `openspec/`) and
  verify:
  - Go tools (`go test`, `golangci-lint`) are reported
    as "SKIP — no in-scope files"
  - `make check` runs (always in scope)
  - The verdict section includes a skip count
  - The coverage matrix shows distinct SKIP status

## 4. Documentation Assessment

- [x] 4.1 Assess CHANGELOG.md impact: add entry under
  the next release section noting the file-scope filter
  addition. Category: "Changed" (pre-flight skill behavior).

- [x] 4.2 [P] Assess AGENTS.md impact: no structural
  changes needed (no new packages, commands, or build
  steps). No update required.

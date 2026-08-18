## Context

The pre-flight skill is a Markdown instruction file consumed by
AI agents through three commands: `/unleash` (hard-gate),
`/review-pr` (ci-aware), and `/review-council` (soft-gate). It
defines a 5-phase pipeline: CI workflow parsing, tool detection,
CI coverage matrix, execution, and result formatting.

Currently, Phase 4 executes all tools detected in Phase 2
regardless of whether the branch diff contains files relevant
to each tool. This wastes time and tokens on branches that only
modify non-code files (YAML, Markdown, agent definitions).

The skill exists in two locations that must stay in sync:
- `~/.config/opencode/skills/pre-flight/SKILL.md` (user-level)
- `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
  (scaffold-deployed copy)

## Goals / Non-Goals

### Goals

- Skip tools that have zero applicable files in the branch diff
- Apply consistently across all three execution modes
- Maintain clear auditability (distinguish "skipped" from
  "passed")
- Preserve conservative defaults (aggregate tools always run,
  config file changes trigger their tools)
- Keep the skill as a declarative Markdown document (no code)

### Non-Goals

- Adding new tools to the detection phase (that's #390's scope)
- Changing how `ci-aware` mode interacts with the GitHub API
- Making the file-scope mapping user-configurable (the mapping
  is hardcoded in the skill; extensibility comes from editing
  the skill file)
- Implementing the filter as Go code — this is purely a skill
  instruction change

## Decisions

### D1: Insert as Phase 2a (between detection and coverage matrix)

The file-scope filter runs after tool detection (Phase 2) and
before the CI coverage matrix (Phase 3). This position is
optimal because:

1. Tools filtered out in Phase 2a never appear as "Run locally
   = Yes" in the matrix, reducing noise.
2. The filter is orthogonal to CI coverage — a tool can be
   skipped by file-scope (no applicable files) independently
   of being skipped by CI coverage (CI already verified).
3. Phase 2a uses only `git` (already a prerequisite) and the
   static mapping table — no new dependencies.

The alternative of filtering during Phase 4 (execution) was
rejected because it would still show skipped tools as "Run
locally = Yes" in the matrix, creating a confusing display.

### D2: Aggregate tools always run

`make check` and `pre-commit` are marked "always in scope"
because they aggregate multiple underlying tools whose file
scope cannot be reliably determined from the aggregate's
config file alone. A Makefile's `check` target may proxy to
`go vet`, `golangci-lint`, `shellcheck`, and other tools not
individually detected.

This is the conservative default. If `make check` becomes too
noisy on non-code branches, the mapping can be refined in a
future change.

### D3: Include tool config files in scope patterns

Tool configuration files (`.golangci.yml`, `ruff.toml`,
`go.mod`, `go.sum`, `pyproject.toml`, `.yamllint.yml`) are
included in their respective tool's scope patterns. A change
to a linter's configuration can alter its behavior (enable/
disable rules, change thresholds) even without source file
changes. Omitting these would create a blind spot where
config changes bypass validation.

This aligns with Constitution Principle V (Security by Default):
the scope mapping must not create bypass opportunities.

### D4: Diff uses merge-base, not HEAD~1

The branch diff is computed against the merge base with the
default branch (`git merge-base HEAD origin/${DEFAULT_BRANCH}`),
not against the previous commit. This captures all files changed
across the entire branch, preventing a scenario where:

1. Commit 1 adds `foo.go`
2. Commit 2 modifies only `config.yaml`
3. A HEAD~1 diff would miss `foo.go` and skip Go tools

The merge-base approach matches how CI evaluates PRs and is
consistent with how `/review-pr` computes diffs.

### D5: Conservative fallback when diff unavailable

If the default branch cannot be detected or the diff command
fails, the file-scope filter is bypassed entirely and all tools
run. This preserves the current behavior as a safe fallback and
prevents the filter from silently suppressing tools due to a
transient git issue.

### D6: Scaffold sync required

Both copies of SKILL.md must be updated atomically in the same
commit. The scaffold copy at `internal/scaffold/assets/opencode/
skills/pre-flight/SKILL.md` must match the user-level copy.
Drift detection tests (per existing scaffold conventions) will
catch any desynchronization.

### D7: Skip status is distinct from pass status

Skipped tools use the status string "SKIP — no in-scope files"
rather than "PASS" in both the coverage matrix and execution
results. This distinction serves Observable Quality (Constitution
Principle III) by making the filter's decisions auditable. A
reviewer can see at a glance which tools were actually exercised
versus which were determined to be irrelevant.

## Risks / Trade-offs

### Risk: Incomplete scope mapping

If a tool can be affected by a file type not listed in its
scope patterns, the filter could skip the tool when it should
run. Mitigated by:
- Including config files in scope patterns (D3)
- Conservative fallback for aggregate tools (D2)
- The mapping is visible in the skill file and easy to extend

### Risk: False sense of security

Skipping tools might give reviewers the impression that fewer
checks are needed. Mitigated by:
- Explicit "SKIP — no in-scope files" status (D7)
- Skip count in the verdict section
- The coverage matrix is always displayed

### Trade-off: Always running aggregate tools

`make check` runs even on YAML-only branches. This means some
waste remains, but it's the safe default. The trade-off is
acceptable because `make check` typically includes `go vet`
and `golangci-lint`, which exit quickly when no `.go` files
changed — the actual cost is the Go module initialization
overhead, not the tool execution itself.

### Trade-off: Git dependency for diff

The filter requires `git` to compute the branch diff. This is
not a new dependency — Phase 4a already requires `git` for
worktree operations. However, if the repo is in a detached
HEAD state or a shallow clone, the merge-base computation may
fail. The conservative fallback (D5) handles this.

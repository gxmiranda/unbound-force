## Why

The `/review-pr` command has 5 usability and effectiveness
gaps that reduce its value as a standardized PR review
tool:

1. **No structural overview** — The review jumps straight
   into findings. For PRs with many changed files,
   reviewers lack orientation before reading detailed
   analysis.
2. **Fix suggestions require manual effort** — Findings
   describe problems and suggest fixes in prose, but the
   reviewer must manually locate and apply each fix.
   GitHub natively supports suggestion blocks that enable
   one-click application.
3. **Uniform review depth** — All files receive identical
   review treatment regardless of their role. Test files,
   API handlers, CI workflows, and documentation have
   fundamentally different risk profiles that warrant
   different review focus.
4. **No issue validation** — PRs commonly reference issues
   via `Fixes #N`, but the review does not validate
   whether the PR actually addresses the issue's
   acceptance criteria. Misalignment between PR and issue
   goes undetected.
5. **Verdict not reflected in GitHub status** — The review
   computes a verdict (APPROVE / REQUEST CHANGES /
   COMMENT) but always posts as a comment. GitHub's
   branch protection and merge eligibility never see the
   actual verdict. A review that says "REQUEST CHANGES"
   in text but is posted as a comment does not block
   merge.

## What Changes

Modify the existing `/review-pr` command with 5
enhancements. No new files, no new dependencies, no Go
code changes.

1. **PR Walkthrough** — Add a `### Walkthrough` section
   to the output listing each changed file with a
   one-line change summary, placed before detailed
   findings.
2. **GitHub Suggestion Blocks** — Use GitHub's native
   suggestion syntax in in-line comments for concrete
   code fixes, enabling one-click application.
3. **Path-based Review Focus** — Built-in heuristics that
   apply targeted review instructions based on file path
   patterns (tests get test quality focus, handlers get
   security focus, CI config gets permissions focus).
4. **Issue Linking** — Parse `Fixes/Closes/Resolves #N`
   from the PR body, fetch issue details, validate
   acceptance criteria during alignment checking.
5. **Verdict-aligned Review Posting** — Map the review
   verdict to the correct `gh pr review` event type so
   GitHub status reflects the actual review outcome.
   Verdict and in-line comments are submitted as a single
   review event.

## Capabilities

### New Capabilities

- `Walkthrough section`: file-by-file change summary for
  structural orientation before findings
- `Suggestion blocks`: one-click fixable code suggestions
  in PR comments
- `Path-based focus`: directory-aware review heuristics
  (no config file required)
- `Issue linking`: automatic validation against linked
  GitHub issue acceptance criteria
- `Verdict-aligned posting`: single review event with
  correct APPROVE / REQUEST CHANGES / COMMENT status

### Modified Capabilities

- Step 9 output format: new `### Walkthrough` and
  `### Linked Issues` sections
- Step 11 comment posting: suggestion block format for
  code fixes, verdict-based review submission as a single
  API call
- Step 8 AI review: path-based focus instructions
  appended per file group, issue acceptance criteria
  added to alignment check

### Removed Capabilities

None. All changes are additive and backward-compatible.

## Impact

**Files modified:**
- `.opencode/command/review-pr.md` — all 5 enhancements
- `internal/scaffold/assets/opencode/command/review-pr.md`
  — scaffold sync (byte-identical copy)
- `AGENTS.md` — PR Review section updated, Recent Changes
  entry

No new files. No scaffold asset count changes. No Go code
changes.

## Constitution Alignment

### I. Autonomous Collaboration

**Assessment**: N/A

No inter-hero artifact changes. The command produces
terminal output and optional GitHub PR reviews — not hero
artifacts.

### II. Composability First

**Assessment**: PASS

All enhancements degrade gracefully. Path focus uses
built-in heuristics (no config file). Issue linking skips
silently when no references found. Suggestion blocks are
only used when concrete fixes exist. Verdict-aligned
posting uses the same `gh` CLI already required.

### III. Observable Quality

**Assessment**: PASS

Walkthrough, issue linking, and verdict alignment all
increase the structure and actionability of the review
output. Verdict-aligned posting makes the review outcome
machine-readable by GitHub's branch protection system.

### IV. Testability

**Assessment**: PASS

Scaffold test suite covers deployment (drift detection,
asset path validation). No new Go logic is introduced.
The command is a Markdown prompt file — verification is
through the existing scaffold test infrastructure and
manual invocation.

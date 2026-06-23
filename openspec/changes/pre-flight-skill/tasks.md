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

## 1. Create Pre-flight Skill

- [x] 1.1 Create `.opencode/skills/pre-flight/SKILL.md`
  with the unified pre-flight logic covering:
  - CI workflow parsing (read `.github/workflows/*.yml`)
  - Local tool detection (config file checks)
  - CI coverage matrix generation
  - `hard-gate` execution policy
  - `ci-aware` execution policy
  - Standardized result format
  Implements: FR-001, FR-002, FR-003, FR-004, FR-005,
  FR-006, FR-007

## 2. Update Consuming Commands

- [x] 2.1 [P] Update `/review-council` (`review-council.md`):
  replace Phase 1a inline CI logic (lines 130-151) with a
  reference to load the `pre-flight` skill in `hard-gate`
  mode. Keep Phase 1b (Gaze analysis) unchanged.
  Implements: FR-008

- [x] 2.2 [P] Update `/review-pr` (`review-pr.md`): replace
  Step 4 inline pre-flight logic (lines 142-198) with a
  reference to load the `pre-flight` skill in `ci-aware`
  mode. Ensure Step 3 CI check results are passed as input.
  Implements: FR-009

- [x] 2.3 [P] Update `/unleash` (`unleash.md`): replace
  Step 5 CI command derivation (lines 330-335) and phase
  checkpoint execution (lines 449-452) with references to
  load the `pre-flight` skill in `hard-gate` mode.
  Implements: FR-010

## 3. Scaffold Sync

- [x] 3.1 [P] Sync `review-council.md` to
  `internal/scaffold/assets/opencode/commands/review-council.md`
  Implements: FR-011

- [x] 3.2 [P] Sync `review-pr.md` to
  `internal/scaffold/assets/opencode/commands/review-pr.md`
  Implements: FR-011

- [x] 3.3 [P] Sync `unleash.md` to
  `internal/scaffold/assets/opencode/commands/unleash.md`
  Implements: FR-011

- [x] 3.4 [P] Sync `.opencode/skills/pre-flight/SKILL.md` to
  `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
  and add the path to `expectedAssetPaths` in
  `internal/scaffold/scaffold_test.go`
  Implements: FR-011

## 4. Verification

- [x] 4.1 Run `make check` to verify build, lint, and
  tests pass (CI parity gate)

- [x] 4.2 Run drift detection tests to confirm scaffold
  copies match command sources and skill sources

- [x] 4.3 Behavioral verification: run `/review-council`
  in code review mode on the current branch to confirm
  pre-flight skill produces expected behavior (tools
  detected, execution order, pass/fail verdict)

- [x] 4.4 Documentation assessment: check whether
  AGENTS.md project structure or CLAUDE.md available
  skills list need updates for the new pre-flight skill
<!-- spec-review: passed -->
<!-- code-review: passed -->

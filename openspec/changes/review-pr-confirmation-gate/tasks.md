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

## 1. Add confirmation gate hardening to review-pr.md

All tasks in this group modify the same file
(`.opencode/commands/review-pr.md`), so none are
parallel-eligible.

- [x] 1.1 Add session-resume guard block at the top
  of Step 11, before the existing item 1 ("Prepare
  comments"). The guard block MUST implement FR-001:
  instruct the agent to re-confirm via AskUserQuestion
  if the session has been resumed from compressed
  context or if prior confirmation cannot be verified
  in the current uncompressed conversation history.
  Use negative-default semantics per design decision
  D3.

- [x] 1.2 Add opening visual gate marker
  `>>> MANDATORY GATE: HUMAN CONFIRMATION REQUIRED <<<`
  immediately before item 2 ("Show all comments for
  human review"), per FR-002 and design decision D2.

- [x] 1.3 Add closing visual gate marker
  `>>> END MANDATORY GATE <<<`
  immediately after item 5 (the CRITICAL RULE), per
  FR-002. The marker MUST appear after the last line
  of the CRITICAL RULE text.

## 2. Verification

- [x] 2.1 Verify the `MANDATORY GATE` marker appears
  exactly twice in `review-pr.md` (opening and closing)
  using grep. Per FR-002 scenario.

- [x] 2.2 Verify the session-resume guard text is
  present and appears before the first `MANDATORY GATE`
  marker in `review-pr.md`.

- [x] 2.3 Verify the existing CRITICAL RULE text
  (item 5, "NEVER post reviews without explicit human
  confirmation") is preserved unchanged within the
  marker boundary.

- [x] 2.4 Verify constitution alignment: confirm that
  the change is additive guard language only, does not
  alter quality gates, does not introduce runtime
  coupling, and does not modify CI or coverage
  thresholds. Per proposal constitution assessment
  (all PASS).

<!-- spec-review: passed -->

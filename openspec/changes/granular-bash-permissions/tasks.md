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

## 1. Add granular bash permissions to opencode.json

- [x] 1.1 Add `"permission"` block to `opencode.json` with
  granular bash rules. Gate mutation commands (`gh issue
  create*`, `gh issue edit*`, `gh issue close*`,
  `gh issue comment*`, `gh pr create*`, `gh pr merge*`,
  `gh pr close*`, `gh pr comment*`, `gh pr edit*`,
  `gh api*`, `git push*`, `git commit*`, `rm *`) behind
  `"ask"`. Set catch-all `"*"` to `"allow"`.
  File: `opencode.json`

## 2. Harden Curator agent permissions

- [x] 2.1 [P] Replace `divisor-curator.md` prose-only bash
  restriction with granular frontmatter permissions.
  Set `bash` to object syntax: `"*": "deny"`,
  `"gh issue list*": "allow"`, `"gh issue view*": "allow"`,
  `"gh issue create*": "ask"`, `"gh repo view*": "allow"`.
  Remove or reduce the prose "Bash Access Restriction"
  section to reference the frontmatter.
  File: `.opencode/agents/divisor-curator.md`

- [x] 2.2 [P] Update scaffold copy of Curator agent with
  identical permission changes.
  File: `internal/scaffold/assets/opencode/agents/divisor-curator.md`

## 3. Verification

- [x] 3.1 Run drift detection tests to verify live and
  scaffold Curator copies match:
  `go test -race -count=1 ./internal/scaffold/...`

- [x] 3.2 Verify `opencode.json` is valid JSON and the
  `"permission"` block structure matches the OpenCode
  config schema.

- [x] 3.3 Verify constitution alignment: the change
  maintains Autonomous Collaboration (agents propose,
  humans approve), Composability First (project-level
  config, no cross-repo deps), Observable Quality
  (declarative JSON permissions), and Testability
  (structure is testable).

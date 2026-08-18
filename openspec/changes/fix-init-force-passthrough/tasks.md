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

  All tasks in this change modify the same two files
  (scaffold.go and scaffold_test.go), so no tasks are
  parallel-eligible.
-->

## 1. Struct and Function Changes

- [x] 1.1 Add `forceFlag string` field to the `simpleTool`
  struct in `internal/scaffold/scaffold.go` (after `args`
  field, line 1333). Update each entry in the
  `simpleTools` slice: specify `"--force"`,
  openspec `"--force"`, gaze `"--force"`,
  replicator `""`. (FR-015, D1)

- [x] 1.2 Refactor `initSimpleTool` signature to accept
  `tool simpleTool` instead of individual parameters
  `name, sentinel, resultName, label string,
  extraArgs []string`. Update the function body to
  reference `tool.name`, `tool.sentinel`, etc. Update the
  single call site in the `for` loop (line 1361-1362).
  Update the GoDoc comment to reflect force-aware
  behavior (the current comment states "Returns nil if
  the sentinel already exists" which will be inaccurate
  after the change). (D3, DR-001)

- [x] 1.3 Modify `initSimpleTool` sentinel guard: change
  the early return condition from
  `if !os.IsNotExist(statErr) { return nil }` to
  `if !os.IsNotExist(statErr) && !opts.Force { return nil }`.
  (FR-014, D2)

- [x] 1.4 Add force flag injection in `initSimpleTool`:
  after the args construction (`append([]string{"init"},
  tool.args...)`), if `opts.Force`, the sentinel existed,
  and `tool.forceFlag` is non-empty, append
  `tool.forceFlag` to args. On first-time init (sentinel
  did not exist), do NOT append forceFlag regardless of
  `opts.Force`. (FR-014)

- [x] 1.5 Add action label branching: when `opts.Force` is
  true and sentinel existed (sentinel was found at the
  guard check), the successful result MUST use action
  `"re-initialized"` instead of `"initialized"`. Track
  whether sentinel existed before the guard. (FR-016, D4)

## 2. Test Coverage

- [x] 2.1 Add `TestInitSimpleTool_SentinelExistsForceTrue`:
  Create sentinel via `t.TempDir()`, set `Force=true`,
  verify `ExecCmd` IS called with the tool's init command
  and result is non-nil with action `"re-initialized"`.

- [x] 2.2 Add `TestInitSimpleTool_SentinelExistsNoForce`:
  Create sentinel via `t.TempDir()`, set `Force=false`,
  verify `initSimpleTool` returns nil and `ExecCmd` is
  NOT called.

- [x] 2.3 Add `TestInitSimpleTool_ForcePassesForceFlag`:
  Set `Force=true`, tool declares `forceFlag: "--force"`,
  verify ExecCmd args include `"--force"`. Assert specific
  arg values.

- [x] 2.4 Add `TestInitSimpleTool_NoForceFlagDeclared`:
  Set `Force=true`, tool declares empty `forceFlag`,
  verify init is re-run without an extraneous force flag
  in args.

- [x] 2.5 Add `TestInitSubTools_SpecifyForcePassthrough`:
  Integration-level test with `Force=true`, specify
  sentinel existing, stubbed ExecCmd. Verify recorded
  command includes
  `specify init --here --integration opencode --offline --force`.

- [x] 2.6 Add `TestInitSimpleTool_ForceReinitFails`:
  Create sentinel via `t.TempDir()`, set `Force=true`,
  stub ExecCmd to return an error. Verify the result has
  `action: "failed"` and `err` is non-nil. Confirms
  error handling is preserved through the force path.

- [x] 2.7 Add `TestInitSimpleTool_NoSentinelForceTrue`:
  Set `Force=true`, do NOT create sentinel. Verify result
  has `action: "initialized"` (not `"re-initialized"`)
  and ExecCmd args do NOT include the `forceFlag`. This
  confirms first-time init behavior is unchanged when
  `Force` is true.

## 3. Verification

- [x] 3.1 Run `go vet ./internal/scaffold/...` and
  `golangci-lint run ./internal/scaffold/...` — no new
  lint warnings.

- [x] 3.2 Run `go test -race -count=1 ./internal/scaffold/...`
  — all existing and new tests pass.

- [x] 3.3 Verify constitution alignment: Composability
  (Principle II) — `uf init --force` re-initializes all
  tools. Observable Quality (Principle III) — force
  produces visible `"re-initialized"` output. Testability
  (Principle IV) — force path has test parity with
  `initDewey`.

<!-- spec-review: passed -->
<!-- code-review: passed -->

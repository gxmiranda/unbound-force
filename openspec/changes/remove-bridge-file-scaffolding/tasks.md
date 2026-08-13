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

## 1. Remove bridge file scaffold code

- [x] 1.1 Delete `ensureCLAUDEmd`, `buildCLAUDEmdBlock`, `claudemdMarker`,
  `ensureCursorrules`, `buildCursorrulesBlock`, `cursorrulesMarker`, and
  `replaceManagedBlock` from `internal/scaffold/scaffold.go`. Remove the
  two call sites in `Run()` (lines 234–235), the comment block at lines
  231–232, and the two variables (`claudeResult`, `cursorResult`) from
  the `subResults` append at line 238. Keep `hasRuleContent`,
  `collectDeployedPacks`, and `filterEmptyCustomPacks` — they are used by
  `ensureAGENTSmdPackSection`.

## 2. Remove bridge file doctor checks

- [x] 2.1 Delete `checkBridgeFile` from `internal/doctor/checks.go`
  (lines 1649–1678). Remove the two call sites at lines 1811–1812 and
  the wrapping append block at lines 1809–1813. Update the GoDoc comment
  at lines 1680–1684 (remove bridge file mention), the inline comment at
  line 1809 ("Checks #11-12: bridge files"), and the inline comment at
  line 1815 (renumber "Check #13" to "Check #11" for branch protection).

## 3. Update scaffold tests

- [x] 3.1 Delete the following test functions from
  `internal/scaffold/scaffold_test.go`:
  - `TestEnsureCLAUDEmd_FreshDir` (~lines 4712–4747)
  - `TestEnsureCLAUDEmd_ExistingWithoutMarker` (~lines 4749–4782)
  - `TestEnsureCLAUDEmd_AlreadyConfigured` (~lines 4839–4867)
  - `TestEnsureCLAUDEmd_StaleContent` (~lines 4869–4905)
  - `TestEnsureCLAUDEmd_StaleContentPreservesPrefix` (~lines 4907–4942)
  - `TestEnsureCLAUDEmd_Idempotent` (~lines 4944–4969)
  - `TestEnsureCLAUDEmd_EmptyCustomPacksOmitted` (~lines 5479–5521)
  - `TestEnsureCLAUDEmd_PopulatedCustomPackIncluded` (~lines 5523–5566)
  - `TestEnsureCursorrules_FreshDir` (~lines 4971–5007)
  - `TestEnsureCursorrules_ExistingWithoutMarker` (~lines 5009–5042)
  - `TestEnsureCursorrules_AlreadyConfigured` (~lines 5044–5072)
  - `TestEnsureCursorrules_StaleContent` (~lines 5074–5110)
  - `TestEnsureCursorrules_StaleContentPreservesPrefix` (~lines 5112–5146)
  - `TestEnsureCursorrules_Idempotent` (~lines 5148–5173)
  - `TestReplaceManagedBlock_NoMarker` (~lines 4784–4810)
  - `TestReplaceManagedBlock_Identical` (~lines 4812–4824)
  - `TestReplaceManagedBlock_Updated` (~lines 4826–4836)
  - `TestReplaceManagedBlock_PreservesPrefix` (~line 4821)

- [x] 3.2 Add `TestRun_DoesNotCreateBridgeFiles` to
  `internal/scaffold/scaffold_test.go`. The test MUST call `Run()` with
  a temp directory, then assert that neither `CLAUDE.md` nor `.cursorrules`
  exist in the target directory after the run. Also assert that the
  `Result` returned by `Run()` contains no `subToolResult` with a name
  or detail containing "CLAUDE" or "cursorrules". Use `t.TempDir()` for
  isolation. Must pass under `-race -count=1`.

- [x] 3.3 Add `TestRun_DoesNotModifyExistingBridgeFiles` to
  `internal/scaffold/scaffold_test.go`. The test MUST pre-create
  `CLAUDE.md` and `.cursorrules` with known content in the temp dir,
  call `Run()`, then assert both files still exist with their original
  content unchanged. Also assert the `Result` does not reference bridge
  files. Use `t.TempDir()` for isolation. Must pass under `-race -count=1`.

## 4. Update doctor tests

- [x] 4.1 Delete `TestCheckAgentContext_BridgeCLAUDEmd` (lines 3565–3618)
  and `TestCheckAgentContext_BridgeCursorrules` (lines 3620–3655) from
  `internal/doctor/doctor_test.go`.

- [x] 4.2 Update `TestDoctorRun_AllPass` in `internal/doctor/doctor_test.go`:
  remove the two `createFile` calls at lines 999–1000 that create
  `CLAUDE.md` and `.cursorrules` fixtures.

- [x] 4.3 Update `TestCheckAgentContext_FullPass` in
  `internal/doctor/doctor_test.go`:
  - Remove the two `createFile` calls at lines 3663–3664 (bridge file
    fixtures).
  - Change the count assertion from 13 to 11 (line 3689).
  - Update the count-enumeration comment at lines 3685–3688 to remove
    the "2 (bridges)" term and update the total.

- [x] 4.4 Add `TestCheckAgentContext_NoBridgeFileResults` to
  `internal/doctor/doctor_test.go`. The test MUST call `checkAgentContext()`
  with a temp directory that contains both `CLAUDE.md` and `.cursorrules`,
  then assert that no `CheckResult` in `group.Results` has a `Name`
  containing "Bridge", "CLAUDE", or "cursorrules". Use `t.TempDir()` for
  isolation. Must pass under `-race -count=1`.

## 5. Update /uf.agent-brief command files

- [x] 5.1 Update `.opencode/commands/uf.agent-brief.md` (live copy):
  - Remove lines 6–7 (description mentioning bridge files).
  - Remove table rows at lines 440–441 ("Bridge: CLAUDE.md" and
    "Bridge: .cursorrules").
  - Remove Step 5 "Bridge File Verification" section (~lines 465–491,
    ~30 lines).
  - Remove bridge file entries from the guardrails section (~lines
    525–526: "CLAUDE.md" and ".cursorrules").

- [x] 5.2 Update
  `internal/scaffold/assets/opencode/commands/uf.agent-brief.md`
  (embedded scaffold asset): apply identical changes as 5.1.
  Both files MUST be identical after this step. Verify with `diff`.

## 6. Clean up documentation

- [x] 6.1 [P] Update `QUICKSTART.md`:
  - Line 123: remove `CLAUDE.md` from the `git add` example command.
  - Lines 149–150: remove the "Bridge files" bullet point.

- [x] 6.2 [P] Update `docs/architecture.md`:
  - Lines 289–295: remove the "Bridge files" subsection entirely.

## 7. Delete root bridge files

- [x] 7.1 Delete `CLAUDE.md` and `.cursorrules` from the repository root.

## 8. Verify and close

- [x] 8.1 Build: `go build ./...` — must succeed with no errors.

- [x] 8.2 Test: `go test -race -count=1 ./internal/scaffold/...
  ./internal/doctor/...` — all tests must pass.

- [x] 8.3 Lint: `go vet ./... && golangci-lint run` — no new lint
  violations. Verify that no dead-code warnings reference the deleted
  functions.

- [x] 8.4 Refresh GAZE baseline: `gaze baseline update` — eliminates
  stale entries for `buildCursorrulesBlock` and `ensureCursorrules` that
  would otherwise cause false CRAP regression alerts in `ci_crapload.yml`.

- [x] 8.5 Constitution alignment: verify that the removal does not
  introduce any violation of the five org constitution principles.
  Confirm Principles I (N/A), II (PASS), III (PASS), IV (PASS),
  V (PASS) as documented in `proposal.md`.

- [x] 8.6 Update `CHANGELOG.md` under `## Unreleased` → `### Removed`
  with an entry documenting bridge file scaffolding removal, referencing
  issue #459. Note: (1) bridge file scaffolding removed from `uf init`,
  (2) bridge file checks removed from `uf doctor`, (3) existing bridge
  files in user repos are unaffected.

- [x] 8.7 Documentation gate: confirm `AGENTS.md` was assessed and
  requires no changes (bridge files are not referenced in the current
  AGENTS.md project structure or conventions sections).

<!-- spec-review: passed -->
<!-- code-review: passed -->

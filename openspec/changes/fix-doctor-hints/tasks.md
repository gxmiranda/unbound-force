# Tasks: Fix Doctor Install Hints

## 1. Replace InstallHint strings

- [x] 1.1 Replace `InstallHint` at line 1668 (AGENTS.md not
  found) with `"Create an AGENTS.md file in your project root
  describing the project for AI agents"`
  — File: `internal/doctor/checks.go`
- [x] 1.2 Replace `InstallHint` at line 1694 (Tier 1 section
  missing) with `fmt.Sprintf("Add a '%s' section to
  AGENTS.md", section.name)` — dynamic per section
  — File: `internal/doctor/checks.go`
- [x] 1.3 Replace `InstallHint` at line 1723 (line count
  exceeds threshold) with `"Condense AGENTS.md — current line
  count exceeds threshold"`
  — File: `internal/doctor/checks.go`
- [x] 1.4 Replace `InstallHint` at line 1749 (constitution
  reference missing) with `"Add a constitution reference to
  AGENTS.md (e.g., Instructions from:
  .specify/memory/constitution.md)"`
  — File: `internal/doctor/checks.go`
- [x] 1.5 Replace `InstallHint` at line 1773 (spec framework
  not described) with `"Add a Specification Workflow section
  to AGENTS.md describing your spec framework"`
  — File: `internal/doctor/checks.go`

## 2. Update tests

- [x] 2.1 Update `TestCheckAgentContext_NoFile` assertion at
  line ~3255 in `internal/doctor/doctor_test.go` to match the
  new hint text from task 1.1
  — File: `internal/doctor/doctor_test.go`
- [x] 2.2 Update `TestCheckAgentContext_MissingTier1Section`
  to assert the new dynamic `InstallHint` text contains the
  section name (e.g., `"Add a 'Project Overview' section to
  AGENTS.md"`) for each missing Tier 1 section
  — File: `internal/doctor/doctor_test.go`
- [x] 2.3 Add `InstallHint` assertions to
  `TestCheckAgentContext_LineCount` ("over threshold"
  subtest), `TestCheckAgentContext_ConstitutionReference`
  ("not referenced" subtest), and
  `TestCheckAgentContext_SpecFrameworkReference` ("not
  described" subtest) to verify the new tool-agnostic hint
  text for each scenario
  — File: `internal/doctor/doctor_test.go`
- [x] 2.4 Extend `TestDoctorHints_NoBareUnboundReferences` to
  also scan for `/uf.` slash command patterns and `OpenCode`
  references in `InstallHint` values — the existing test only
  checks for bare `"unbound "` branding
  — File: `internal/doctor/doctor_test.go`
- [x] 2.5 Add drift-detection test
  `TestCheckAgentContext_ProjectAGENTSmdHasAllTier1Sections`
  that reads the project's own `AGENTS.md` and asserts all
  five Tier 1 section patterns from
  `agentContextTier1Sections` match
  — File: `internal/doctor/doctor_test.go`

## 3. Verification

- [x] 3.1 Run `go test -race -count=1 ./internal/doctor/...`
  — all tests pass
- [x] 3.2 Run `go vet ./internal/doctor/...` and
  `golangci-lint run ./internal/doctor/...` — no warnings
- [x] 3.3 Verify Constitution Principle II alignment:
  `uf doctor` output contains no OpenCode-specific references
  in any `InstallHint` value
<!-- spec-review: passed -->

# Design: Tool-Agnostic Install Hints

## Context

The `uf doctor` command runs health checks on a project and
reports findings with remediation hints. Five `InstallHint`
strings in the agent context check group hardcode
`"Run: /uf.agent-brief in OpenCode"`, coupling `uf doctor`
output to OpenCode. Other hints in the same file already use
tool-agnostic phrasing (e.g., `"Run: uf init"`,
`"Add fenced code blocks with build/test commands"`).

This change aligns the five outliers with the established
pattern, serving Constitution Principle II (Composability
First) — `uf doctor` must produce actionable output when
deployed alone.

## Goals / Non-Goals

### Goals
- Replace five `InstallHint` string literals with
  tool-agnostic remediation instructions
- Update the corresponding test assertion to match
- Optionally add a drift-detection test for Tier 1 sections

### Non-Goals
- Changing the `CheckResult` struct or JSON output schema
- Modifying the check logic (severity, thresholds, pass/fail)
- Addressing other `uf doctor` issues from parent #212

## Decisions

### D-001: Descriptive hints over command references

Each replacement hint describes what to do rather than which
command to run. This makes hints useful regardless of the
user's editor or workflow tool.

Example: `"Add a '<section>' section to AGENTS.md"` instead of
`"Run: /uf.agent-brief in OpenCode"`.

Rationale: Consistent with the existing pattern in
`checks.go` (lines 1712, 1790) and with Principle II — users
should not need to know about OpenCode to act on doctor
output.

### D-002: Dynamic section name in Tier 1 hint

The Tier 1 section missing hint (line 1694) runs in a loop
over `agentContextTier1Sections`. The replacement hint
includes the section name via `fmt.Sprintf` so the user knows
exactly which section to add. This is already the pattern
used for the check message itself.

### D-003: Drift-detection test scope

The optional drift-detection test reads the project's own
`AGENTS.md` and asserts all five Tier 1 section headers are
present. This test lives in `internal/doctor/` alongside
existing drift tests (e.g.,
`TestDoctorHints_NoBareUnboundReferences`). It does not
validate content — only header presence.

## Risks / Trade-offs

- **Hint text changes are user-visible**: Users or downstream
  tooling that pattern-match on hint text may need to update.
  Risk is low because `InstallHint` is documented as a
  display-only field and the JSON schema does not constrain
  its value.
- **No docs guide exists yet**: The AGENTS.md creation hint
  references `docs/agents-md-guide.md` which may not exist.
  The hint should use a generic phrasing that remains useful
  even without the guide. Fallback: reference the project's
  own `AGENTS.md` as an example.

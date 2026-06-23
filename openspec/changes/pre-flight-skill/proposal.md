## Why

Three commands (`/review-council`, `/review-pr`, `/unleash`) each
independently implement logic for detecting CI workflows, discovering
local tools, and executing quality checks before AI review or
implementation proceeds. This duplication means:

- Changes to CI detection logic must be replicated in 3+ places
- `/review-council` lacks the CI coverage matrix optimization that
  `/review-pr` has (wastes time re-running tools CI already verified)
- `/review-pr` lacks the hard-gate option that `/review-council` has
- Bug fixes or improvements to one command don't propagate to others

Fixes: https://github.com/unbound-force/unbound-force/issues/175

## What Changes

Extract the duplicated CI detection and local tool execution logic
into a shared OpenCode skill at `.opencode/skills/pre-flight/SKILL.md`.
Update the three consuming commands to reference the skill instead of
inlining the logic.

## Capabilities

### New Capabilities
- `pre-flight skill`: Shared skill encapsulating CI workflow parsing,
  local tool detection, CI coverage matrix generation, and two
  execution policies (`hard-gate` and `ci-aware`)

### Modified Capabilities
- `/review-council`: Phase 1a replaced with skill reference in
  `hard-gate` mode; gains CI coverage matrix optimization
- `/review-pr`: Step 4 replaced with skill reference in `ci-aware`
  mode; gains hard-gate option for stricter local review
- `/unleash`: Step 5 CI derivation and phase checkpoint replaced
  with skill reference in `hard-gate` mode

### Removed Capabilities
- None — no capabilities removed. The unified detection
  approach may discover additional tools compared to
  individual command implementations; this is an intentional
  improvement (see design decision D2)

## Impact

### Files Changed
- `.opencode/skills/pre-flight/SKILL.md` (new)
- `.opencode/commands/review-council.md` (Phase 1a extraction)
- `.opencode/commands/review-pr.md` (Step 4 extraction)
- `.opencode/commands/unleash.md` (Step 5 + phase checkpoint)
- `internal/scaffold/assets/opencode/commands/review-council.md`
  (scaffold sync)
- `internal/scaffold/assets/opencode/commands/review-pr.md`
  (scaffold sync)
- `internal/scaffold/assets/opencode/commands/unleash.md`
  (scaffold sync)
- `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
  (scaffold sync for new skill)
- `internal/scaffold/scaffold_test.go` (add to
  expectedAssetPaths)

### Follow-up
`/agent-brief` (L61-70) reads `.github/workflows/` and config files
for discovery (not execution). A `detect-only` mode could serve this
use case but is a different lifecycle stage. Tracked separately per
issue #175.

## Constitution Alignment

Assessed against the Unbound Force org constitution (v1.2.0).

### I. Autonomous Collaboration

**Assessment**: N/A

This change affects agent instruction files (skills and commands),
not inter-hero artifact communication. No artifact formats, metadata,
or exchange patterns are modified.

### II. Composability First

**Assessment**: PASS

The skill is independently loadable — any command can reference it
without requiring other commands or skills to be present. Commands
that do not use the skill continue to function unchanged. The skill
adds value when composed with review/implementation commands but
creates no mandatory dependencies.

### III. Observable Quality

**Assessment**: PASS

The skill defines a standardized result format (CI coverage matrix)
that makes skip/run decisions visible and auditable. Both execution
policies produce structured output that consuming commands can
display and act on consistently.

### IV. Testability

**Assessment**: PASS

The skill is an instruction file (no compiled code), so traditional
unit testing does not apply. However, the behavioral parity
requirement in the acceptance criteria ensures that both commands
produce identical outcomes before and after the extraction — this
is verifiable by running `/review-council` and `/review-pr` against
a real branch and comparing results.

### V. Security by Default

**Assessment**: PASS

The skill reads workflow files and derives commands for local
execution, inheriting the same security posture as the existing
inline implementations. No new input vectors are introduced.
The design (D8) specifies extraction boundaries: only `run:`
steps are extracted, CI expressions are skipped, and action
references are ignored.

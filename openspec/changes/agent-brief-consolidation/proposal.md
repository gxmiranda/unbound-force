## Why

AGENTS.md is loaded into every agent conversation (~25K tokens at
983 lines). Analysis shows ~12% of the file is redundant or
historical content that agents never use for decisions. Two
separate commands (`/agent-brief` and `/uf-init` Step 9) modify
the file with overlapping scope but no mutual awareness, creating
a coordination gap where neither detects the other's missing
sections.

Additionally, the "Recent Changes" section grows monotonically
with every completed spec (~50 lines currently), is never
consulted for agent decision-making, and should live in a
dedicated CHANGELOG.md. The "Active Technologies" section
repeats "Go 1.24+" 20+ times with per-spec annotations that
duplicate go.mod content.

## What Changes

1. **Consolidate AGENTS.md ownership** into `/agent-brief` as
   the single command that creates, audits, and maintains the
   file. Remove Step 9 (AGENTS.md behavioral guidance injection)
   from `/uf-init`.

2. **Compress AGENTS.md** from ~983 lines to ~250-300 lines by:
   - Converting 8 governance rules from multi-paragraph prose
     into one-line RFC-style rules (~137 lines → ~12 lines)
   - Removing redundant sections (Constitution summary, Git &
     Workflow, Inter-Hero Artifact Types, Sibling Repositories)
   - Compressing Specification Framework from ~133 to ~20 lines
   - Compressing Knowledge Retrieval from ~60 to ~10 lines
   - Removing Active Technologies entirely (go.mod is the
     source of truth)

3. **Externalize Recent Changes** to CHANGELOG.md and redirect
   the decentralized writing chain (task templates, Divisor
   agents, `update-agent-context.sh`) to target the new file.

4. **Optimize `/agent-brief`** with template-based governance
   blocks (0 reasoning tokens vs ~2K), conditional file reads,
   and selective refresh in audit mode.

## Capabilities

### New Capabilities
- `CHANGELOG.md`: Dedicated changelog file for completed
  spec/change entries, replacing the AGENTS.md Recent Changes
  section
- `agent-brief` template governance: Governance blocks injected
  as verbatim templates with context-sensitive detection
  (constitution → full governance, Dewey → knowledge retrieval)
- `agent-brief` selective refresh: Audit mode re-derives Build
  Commands and Project Structure from filesystem and compares
  against existing content to detect staleness

### Modified Capabilities
- `/agent-brief`: Restructured section taxonomy, merged
  governance templates from `/uf-init`, optimized file reads
  (19 → ~8-10 conditional), added selective refresh
- `/uf-init`: Step 9 removed (AGENTS.md injection). Steps 2-8
  (OpenSpec/Speckit customizations) unchanged.
- `divisor-guard.md`: Documentation Completeness checklist
  redirected from "AGENTS.md Recent Changes" to "CHANGELOG.md"
- `divisor-curator.md`: Documentation Gap Detection redirected
  to CHANGELOG.md
- `divisor-scribe.md`: Change entry format guide targets
  CHANGELOG.md; AGENTS.md procedure covers structural sections
- `divisor-herald.md`: Source reference redirected to
  CHANGELOG.md for recent changes
- `divisor-envoy.md`: Source reference redirected to
  CHANGELOG.md
- `divisor-architect.md`: Spec review check references
  CHANGELOG.md for change currency
- `update-agent-context.sh`: Recent Changes logic targets
  CHANGELOG.md; Active Technologies logic removed entirely

### Removed Capabilities
- `/uf-init` Step 9: AGENTS.md behavioral guidance injection.
  Absorbed by `/agent-brief` governance templates.
- AGENTS.md `## Recent Changes`: Moved to CHANGELOG.md
- AGENTS.md `## Active Technologies`: Removed (redundant with
  go.mod and per-spec technology is changelog material)
- AGENTS.md `## Constitution`: Removed (already loaded
  verbatim via constitution.md in system prompt)
- AGENTS.md `## Git & Workflow`: Removed (7-line pointer to
  constitution, which is already loaded)

## Website Gate

Website issue exempt: changes are internal to the agent
toolchain (slash command prompt files, agent behavioral
instructions, scaffold asset sync). The `/agent-brief` and
`/uf-init` commands are not documented on the public website.

## Impact

**Files modified**: 20 files across commands (2), agents (6),
templates (1), scripts (1), documentation (2), scaffold copies
(8).

**Per-conversation savings**: ~17-19K tokens saved per agent
interaction (~70% reduction in AGENTS.md size).

**Governance coverage**: All 8 behavioral rules preserved in
compressed format. Zero rules dropped.

**Breaking changes**: None. The Recent Changes redirect is
transparent to implementing agents -- task templates generate
the right target based on the Documentation Validation Gate
wording, which is updated as part of this change.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

This change improves artifact quality by producing a more
compact, focused AGENTS.md that agents consume more
efficiently. CHANGELOG.md is a new self-describing artifact
with provenance (spec paths in each entry). No runtime
coupling is introduced -- the redirect from AGENTS.md to
CHANGELOG.md is purely artifact-based.

### II. Composability First

**Assessment**: PASS

`/agent-brief` remains independently usable. The Go binary's
`ensureAGENTSmdPackSection` continues to work independently
(deterministic, no LLM dependency). CHANGELOG.md is a
standard file that works regardless of which heroes or tools
are deployed.

### III. Observable Quality

**Assessment**: PASS

CHANGELOG.md entries follow the Scribe's format with spec
paths for provenance. The compressed AGENTS.md retains all
machine-actionable rules (MUST/SHOULD format). The
`/agent-brief` audit mode provides measurable quality scoring.

### IV. Testability

**Assessment**: PASS

Scaffold drift tests verify asset sync. A grep-based
regression test can verify no remaining "Recent Changes"
references point to AGENTS.md in agent/command files. The
Go binary's `ensureAGENTSmdPackSection` is already tested
with dependency injection. No Go source code is added or
removed; coverage ratchets are not expected to regress.

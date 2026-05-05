## Context

AGENTS.md serves as the primary behavioral context file loaded
into every AI agent conversation. At 983 lines (~25K tokens),
it has grown through append-only additions across 35+ specs.
Two commands (`/agent-brief` and `/uf-init` Step 9) modify the
file independently with no coordination, and the "Recent
Changes" section grows without bound.

The constitution alignment from the proposal confirms all four
principles are satisfied (PASS on I-IV). This design preserves
Autonomous Collaboration by keeping CHANGELOG.md as a
self-describing artifact, Composability by maintaining
independent tool operation, Observable Quality via the Scribe's
provenance format, and Testability via scaffold drift tests and
regression grep.

## Goals / Non-Goals

### Goals
- Reduce AGENTS.md from ~983 to ~250-300 lines (~70% reduction)
- Single command (`/agent-brief`) owns all AGENTS.md content
- Redirect change entries to CHANGELOG.md with zero dropped
  governance rules
- Reduce `/agent-brief` token cost by ~35% through template-
  based governance blocks and conditional file reads
- Update the full chain of agents and scripts that read/write
  Recent Changes

### Non-Goals
- Rewriting the Go binary's `ensureAGENTSmdPackSection` --
  it stays as-is (deterministic, zero LLM cost)
- Modifying historical `tasks.md` files in completed specs --
  they correctly targeted AGENTS.md at the time
- Changing the Speckit or OpenSpec task templates -- LLM-
  generated tasks follow the Documentation Validation Gate
  wording, which this change updates
- Upstream proposals to OpenSpec or Speckit -- those are
  separate initiatives identified during analysis but out of
  scope here

## Decisions

### D1: Governance blocks as templates, not LLM-generated

The 8 governance blocks from `/uf-init` Step 9 are static
text that don't change between runs. Rather than having the
LLM generate them (~2K reasoning tokens), `/agent-brief`
embeds them as verbatim template text and inserts them based
on context detection:

- Constitution exists → full governance rules
- Dewey configured → Knowledge Retrieval section
- `specs/` or `openspec/` present → Specification Workflow

This eliminates ~2K tokens of LLM reasoning per run while
producing identical output.

### D2: Compression strategy -- rules as rules, not prose

Each governance subsection is compressed from a multi-paragraph
explanation to a single RFC-style MUST/SHOULD rule line.
Example: CI Parity Gate goes from 10 lines to 1 line.

The full explanations remain available in the constitution
(loaded separately) and in individual agent files. AGENTS.md
only needs the actionable rule.

### D3: CHANGELOG.md as the change-entry target

Recent Changes entries follow the Scribe's format
(`- <name>: <summary>` with `Spec:` paths). This format
transfers directly to CHANGELOG.md with no structural change.

The redirect is achieved by updating the Documentation
Validation Gate in AGENTS.md -- since LLM-generated task lists
are driven by this gate's wording, changing it from "AGENTS.md"
to "CHANGELOG.md" automatically redirects all future task
generation without modifying task templates.

### D4: Go binary keeps Convention Packs section

The `ensureAGENTSmdPackSection` function in the scaffold engine
runs deterministically at zero LLM cost on every `uf init`.
`/agent-brief` detects and respects the existing section rather
than regenerating it. Two owners for this one section is
acceptable because the Go binary is deterministic and the
section is trivial (just a file listing).

### D5: Active Technologies removed, not relocated

The Active Technologies section duplicates go.mod content with
per-spec annotations. Unlike Recent Changes (which has value as
a changelog), Active Technologies has no standalone value -- the
source of truth is go.mod. It is removed entirely, not moved to
another file.

### D6: update-agent-context.sh dual update

The Speckit script `update-agent-context.sh` currently writes
both `## Active Technologies` and `## Recent Changes` in
AGENTS.md. Both code paths are modified:

- Active Technologies logic: removed entirely (section deleted)
- Recent Changes logic: redirected to write to CHANGELOG.md
  instead, with the same prepend-and-trim-to-2 behavior

## Test Strategy

Tasks 2.1 and 3.1 modify LLM prompt files (`.opencode/command/*.md`),
not Go source code. The test strategy is:

1. **Scaffold drift tests** (existing): verify asset sync between
   live copies and scaffold copies (tasks 6.1-6.8)
2. **Grep regression** (task 7.2): verify no stale "AGENTS.md
   Recent Changes" references remain in agent/command files
3. **`make check`** (task 7.1): verify Go code integrity

No Go source code is added or removed by this change; coverage
ratchets are not expected to regress. The bash script change
(task 5.1, `update-agent-context.sh`) is a Speckit automation
script without Go test coverage — its correctness is verified
by manual execution during task completion.

## Risks / Trade-offs

### R1: Compressed rules may be less discoverable

One-line rules are more token-efficient but less explanatory
for new contributors reading AGENTS.md directly. Mitigation:
the constitution (loaded separately in every conversation)
provides full explanations, and individual agent files contain
role-specific elaboration.

### R2: Two-owner Convention Packs section

Both the Go binary and `/agent-brief` can write to AGENTS.md.
The Convention Packs section is the overlap point. Mitigation:
`/agent-brief` checks for the existing section before writing,
and the Go binary only appends if the marker heading is absent
(both are idempotent).

### R3: Historical tasks.md files reference old target

Completed spec task files say "Update AGENTS.md Recent Changes"
-- these are not updated retroactively. Mitigation: these tasks
are already completed and checked off. No agent re-executes
completed tasks from historical files.

### R4: Hero repo propagation

Hero repos (Gaze, Replicator, etc.) will receive the new
AGENTS.md format on their next `/agent-brief` or `uf init`
run. This is the expected propagation path and requires no
manual intervention. Removed sections (Active Technologies,
Recent Changes, Constitution summary) will not be regenerated.

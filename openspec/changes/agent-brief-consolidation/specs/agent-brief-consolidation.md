## ADDED Requirements

### Requirement: CHANGELOG.md as change-entry target

The project MUST maintain a `CHANGELOG.md` file at the
repository root for recording completed spec and change
entries. Each entry MUST follow the Scribe format:
`- <change-name>: <summary>` with `Spec:` paths to
canonical specs.

#### Scenario: New spec completed

- **GIVEN** an agent completes a spec implementation
- **WHEN** it reaches the documentation task
- **THEN** it appends an entry to `CHANGELOG.md` (not
  AGENTS.md) with the change name, summary, and spec path

#### Scenario: CHANGELOG.md does not exist

- **GIVEN** a fresh repository after `uf init`
- **WHEN** `/agent-brief` runs in create mode
- **THEN** it creates `CHANGELOG.md` with a heading and
  no entries

### Requirement: Template-based governance in agent-brief

`/agent-brief` MUST embed governance blocks as verbatim
template text, not LLM-generated content. Blocks MUST be
inserted conditionally based on project context detection:

- `.specify/memory/constitution.md` exists →
  Behavioral Rules section with all 8 governance rules
- Dewey MCP configured in `opencode.json` →
  Knowledge Retrieval section
- `specs/` or `openspec/` directories exist →
  Specification Workflow section

#### Scenario: Constitution detected

- **GIVEN** `.specify/memory/constitution.md` exists
- **WHEN** `/agent-brief` runs in create mode
- **THEN** the generated AGENTS.md includes a
  "Behavioral Rules" section with all 8 compressed
  governance rules

#### Scenario: No constitution

- **GIVEN** `.specify/memory/constitution.md` does not exist
- **WHEN** `/agent-brief` runs in create mode
- **THEN** the generated AGENTS.md omits the
  "Behavioral Rules" section entirely

### Requirement: Selective refresh in audit mode

`/agent-brief` audit mode MUST re-derive Build Commands
and Project Structure from the current filesystem and
compare against what AGENTS.md contains. Stale sections
MUST be flagged specifically with the detected delta.

#### Scenario: Build command added

- **GIVEN** a new Makefile target exists that is not in
  AGENTS.md Build section
- **WHEN** `/agent-brief audit` runs
- **THEN** the audit report flags "Build & Test: stale --
  missing target `<name>`" and offers to update

## MODIFIED Requirements

### Requirement: Documentation Validation Gate

Previously: "AGENTS.md -- new specs, changed hero status,
updated conventions"

New: The Documentation Validation Gate MUST direct change
entries to `CHANGELOG.md` and structural updates to
`AGENTS.md`. Specifically:

- `CHANGELOG.md` -- completed spec/change entries
- `AGENTS.md` -- project structure, conventions, build
  commands (only when these actually changed)

#### Scenario: Agent assesses documentation impact

- **GIVEN** an agent completes an implementation task
- **WHEN** it reads the Documentation Validation Gate
- **THEN** it writes the change entry to `CHANGELOG.md`
  and updates AGENTS.md only if structure or conventions
  changed

### Requirement: Divisor Guard documentation checklist

Previously: "Was AGENTS.md updated (Recent Changes,
Project Structure, Active Technologies as applicable)?"

New: The Guard MUST check for `CHANGELOG.md` updates
for change entries and `AGENTS.md` updates for structural
changes. Active Technologies reference MUST be removed.

### Requirement: Divisor Curator documentation checklist

Previously: same as Guard.

New: same redirect as Guard -- check `CHANGELOG.md` for
change entries, `AGENTS.md` for structural changes.

### Requirement: Divisor Scribe format guide

Previously: "AGENTS.md Updates" section with Recent
Changes format targeting AGENTS.md.

New: The Scribe's change-entry format guide MUST target
`CHANGELOG.md`. The AGENTS.md procedure MUST cover
structural sections only (Project Structure, Conventions,
Build Commands).

### Requirement: Divisor Herald source documents

Previously: "AGENTS.md: Project overview, recent changes,
hero descriptions"

New: MUST reference "CHANGELOG.md: recent changes" and
"AGENTS.md: project overview, hero descriptions"

### Requirement: Divisor Envoy source documents

Previously: "AGENTS.md: Project overview, capabilities,
recent changes"

New: same redirect as Herald.

### Requirement: update-agent-context.sh

Previously: script writes to `## Recent Changes` and
`## Active Technologies` in AGENTS.md.

New: Active Technologies logic MUST be removed entirely.
Recent Changes logic MUST target `CHANGELOG.md` instead
of AGENTS.md, preserving the prepend-and-trim-to-2
behavior.

### Requirement: /uf-init Step 9

Previously: Step 9 injects 8 behavioral guidance blocks
into AGENTS.md.

New: Step 9 MUST be removed. The summary in Step 10 MUST
be updated to remove the "AGENTS.md Guidance" section.
The governance blocks are absorbed by `/agent-brief`.

### Requirement: agent-file-template.md

Previously: template includes `## Recent Changes` with
`[LAST 3 FEATURES AND WHAT THEY ADDED]` placeholder.

New: The `## Recent Changes` section MUST be removed
from the template. A pointer "See CHANGELOG.md for
recent changes" MAY be added.

## REMOVED Requirements

### Requirement: AGENTS.md Recent Changes section

The `## Recent Changes` section in AGENTS.md is removed.
Content is migrated to `CHANGELOG.md`. Reason: the
section grows without bound, is never used for agent
decision-making, and accounts for ~5% of the file's
token budget with zero actionable value.

### Requirement: AGENTS.md Active Technologies section

The `## Active Technologies` section is removed. Reason:
it duplicates go.mod content with per-spec annotations
that are changelog material, not runtime context.

### Requirement: AGENTS.md Constitution summary

The `## Constitution (Highest Authority)` section is
removed. Reason: the full constitution text is already
loaded verbatim via `.specify/memory/constitution.md`
in every conversation's system prompt.

### Requirement: AGENTS.md Git & Workflow section

The `## Git & Workflow` section is removed. Reason: it
is a 7-line pointer to the constitution's Development
Workflow section, which is already loaded.

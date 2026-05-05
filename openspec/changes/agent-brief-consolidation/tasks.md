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

## 1. Create CHANGELOG.md and restructure AGENTS.md

- [x] 1.1 Create `CHANGELOG.md` at repository root. Migrate the current `## Recent Changes` section content from `AGENTS.md` preserving the Scribe format (`- <name>: <summary>` with `Spec:` paths). Add a file heading `# Changelog`.
- [x] 1.2 Restructure `AGENTS.md` to the target ~250-300 line format. Remove: `## Recent Changes`, `## Active Technologies`, `## Constitution (Highest Authority)`, `## Git & Workflow`, `## Inter-Hero Artifact Types`, `## Sibling Repositories`. Compress: `## Behavioral Constraints` (6 subsections → ~12 one-line rules in `## Behavioral Rules`), `## Specification Framework` (~133 → ~20 lines in `## Specification Workflow`), `## Knowledge Retrieval` (~60 → ~10 lines), `## Spec Organization` (~65 → ~5 lines), `## The Heroes` (~19 → ~3 lines). Merge `## Core Mission` into `## Project Overview`. Merge `## Writing Style for Specs` into `## Coding Conventions`. Keep `## Convention Packs` as-is (Go binary owned). Update `## Documentation Validation Gate` to split targets: `CHANGELOG.md` for change entries, `AGENTS.md` for structural updates only.

## 2. Redesign /agent-brief command

- [x] 2.1 Restructure `.opencode/command/agent-brief.md` (live copy). Replace the 11-section Tier taxonomy with the target section list: Project Overview, Build & Test, Project Structure, Coding Conventions, Testing Conventions, Behavioral Rules, Specification Workflow, Knowledge Retrieval, Convention Packs. Embed governance blocks as verbatim templates with conditional insertion based on context detection (constitution → Behavioral Rules, Dewey → Knowledge Retrieval, specs/openspec → Specification Workflow). Optimize file reads: make CI workflow reads conditional on Build section generation (not in audit mode). Add selective refresh: in audit mode, re-derive Build Commands and Project Structure from filesystem, flag stale content with specific delta. Integrate Convention Packs awareness: detect `.opencode/uf/packs/` listing, respect existing section from Go binary. Add CHANGELOG.md handling: in create mode, create empty CHANGELOG.md if it doesn't exist.

## 3. Update /uf-init command

- [x] 3.1 Update `.opencode/command/uf-init.md` (live copy). Remove Step 9 (AGENTS.md Behavioral Guidance) entirely -- all 8 blocks, detection logic, insertion logic, and verification. Update Step 10 summary report to remove the "AGENTS.md Guidance" section. Renumber remaining steps if needed (Steps 1-8 become Steps 1-8, Step 10 becomes Step 9). Update the summary template to reflect the removed section.

## 4. Redirect Divisor agents to CHANGELOG.md

- [x] 4.1 [P] Update `.opencode/agents/divisor-guard.md`. In the "Documentation Completeness" checklist (~line 100-101): change "Was AGENTS.md updated (Recent Changes, Project Structure, Active Technologies as applicable)?" to "Was CHANGELOG.md updated with change entry? Was AGENTS.md updated (Project Structure, Conventions) if structure or conventions changed?". Remove "Do Recent Changes entries include `Spec:` paths" and replace with "Do CHANGELOG.md entries include `Spec:` paths to canonical specs?". Remove Active Technologies reference.
- [x] 4.2 [P] Update `.opencode/agents/divisor-curator.md`. In "Documentation Gap Detection" (~line 101-102): apply the same redirect as Guard -- CHANGELOG.md for change entries, AGENTS.md for structural changes. Remove Active Technologies reference.
- [x] 4.3 [P] Update `.opencode/agents/divisor-scribe.md`. In "AGENTS.md Updates" section (~lines 59-71): rename to "Documentation Updates". Split into two subsections: "CHANGELOG.md Entries" (change entries with the existing format guide: `- <name>: <summary>` with `Spec:` paths) and "AGENTS.md Updates" (structural sections only: Project Structure, Conventions, Build Commands). Update all references from "AGENTS.md Recent Changes" to "CHANGELOG.md".
- [x] 4.4 [P] Update `.opencode/agents/divisor-herald.md`. At ~line 39: change source document reference from "AGENTS.md: Project overview, recent changes, hero descriptions" to "CHANGELOG.md: recent changes; AGENTS.md: project overview, hero descriptions".
- [x] 4.5 [P] Update `.opencode/agents/divisor-envoy.md`. At ~line 40: change source document reference from "AGENTS.md: Project overview, capabilities, recent changes" to "CHANGELOG.md: recent changes; AGENTS.md: project overview, capabilities".
- [x] 4.6 [P] Update `.opencode/agents/divisor-architect.md`. At ~line 158: update spec review check from "Is AGENTS.md up to date" to "Is CHANGELOG.md up to date with change entries? Is AGENTS.md up to date with structural changes?".

## 5. Update Speckit automation and templates

- [x] 5.1 [P] Update `.specify/scripts/bash/update-agent-context.sh`. Remove the `## Active Technologies` logic entirely (lines ~400-442, ~476-487). Redirect the `## Recent Changes` logic (lines ~444-494) to write to `CHANGELOG.md` instead of AGENTS.md. Keep the prepend-and-trim-to-2 behavior. Update the section heading check from `"^## Recent Changes"` in AGENTS.md to the same heading in CHANGELOG.md. Handle the case where CHANGELOG.md doesn't exist (create with `# Changelog` heading).
- [x] 5.2 [P] Update `.specify/templates/agent-file-template.md`. At line 23: remove the `## Recent Changes` section with `[LAST 3 FEATURES AND WHAT THEY ADDED]` placeholder. Add a line: "See CHANGELOG.md for recent changes."

## 6. Sync scaffold assets

- [x] 6.1 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-guard.md` with the live copy updated in 4.1.
- [x] 6.2 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-curator.md` with the live copy updated in 4.2.
- [x] 6.3 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-scribe.md` with the live copy updated in 4.3.
- [x] 6.4 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-herald.md` with the live copy updated in 4.4.
- [x] 6.5 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-envoy.md` with the live copy updated in 4.5.
- [x] 6.6 [P] Sync `internal/scaffold/assets/opencode/agents/divisor-architect.md` with the live copy updated in 4.6.
- [x] 6.7 [P] Sync `internal/scaffold/assets/opencode/command/agent-brief.md` with the live copy updated in 2.1.
- [x] 6.8 [P] Sync `internal/scaffold/assets/opencode/command/uf-init.md` with the live copy updated in 3.1.

## 7. Verification and regression

- [x] 7.1 Run `make check` (build, test, vet, lint). Fix any failures.
- [x] 7.2 Grep all `.opencode/agents/*.md` and `.opencode/command/*.md` files for remaining references to "AGENTS.md" combined with "Recent Changes" -- none should remain (regression check). Verify no agent/command still directs change entries to AGENTS.md.
- [x] 7.3 Verify governance rule completeness: confirm all 8 rules from the old Behavioral Constraints subsections (Gatekeeping, Phase Boundaries, CI Parity, Review Council, Branch Protection, Documentation Gate, Website Gate, Zero-Waste) are present in the compressed Behavioral Rules section.
- [x] 7.4 Verify AGENTS.md line count is within target range (~250-300 lines). Verify CHANGELOG.md contains all entries from the old Recent Changes section.
- [x] 7.5 Update `CHANGELOG.md` with entry for this change: `- agent-brief-consolidation: <summary>`.
<!-- spec-review: passed -->
<!-- code-review: passed -->

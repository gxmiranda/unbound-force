# ADR-002: Durable source-of-truth layer for UF references in speckit command templates

**Status:** Accepted
**Date:** 2026-08-27
**Context:** The `speckit.*.md` command files require UF-specific
references — Dewey knowledge retrieval (`dewey_semantic_search` /
`dewey_search`), `.specify/memory/constitution.md` awareness, and
`/uf.review-council` (for `speckit.implement.md` only). Several
upstream commands are missing these references (tracked in #445,
#546). Before any edit can durably add them, the authoritative
edit layer must be pinned so the sibling edit issue can target it
without re-litigating durability.

## Problem

The five upstream speckit command files are canonical-but-not-embedded:

- `speckit.specify.md`
- `speckit.plan.md`
- `speckit.tasks.md`
- `speckit.implement.md`
- `speckit.constitution.md`

They are listed in `knownNonEmbeddedFiles`
(`internal/scaffold/scaffold_test.go:1242-1251`) and explicitly
excluded from the `TestCanonicalSources_AreEmbedded` drift test
(`scaffold_test.go:1337`). They are NOT embedded in the binary —
they are produced per-repo by upstream `specify init` and then
post-processed by `/uf.init`.

Note: `knownNonEmbeddedFiles` lists all nine speckit commands as
non-embedded. The remaining four (`speckit.analyze.md`,
`speckit.checklist.md`, `speckit.clarify.md`,
`speckit.taskstoissues.md`) are UF-custom commands created by
Step 5 and are out of scope for this decision. This ADR concerns
only the five *upstream* speckit files listed above, which
`specify init` regenerates.

Consequently, editing the installed `.opencode/commands/` copies
is **non-durable**: the next `specify init` + `/uf.init` cycle
regenerates them and any manual edit is silently lost. This is a
reliability / silent-data-loss failure mode, not a cosmetic one.

The current `/uf.init` layers relevant to this decision are:

| Step | Role | Mutating? |
|------|------|-----------|
| Step 5 (`uf.init.md:452`) | Creates the 4 UF-custom commands upstream does not provide (`analyze`, `checklist`, `clarify`, `taskstoissues`) | Yes (create) |
| Step 6 (`uf.init.md:546`) | Injects/corrects a `## Guardrails` section into all 9 speckit commands via an idempotent read → check → inject/correct pattern with correctness markers | Yes (inject/correct) |
| Step 7 (`uf.init.md:677`) | Verifies UF references are present and reports what is missing | **No** — read-only (`uf.init.md:704`) |

## Decision

**Extend Step 6 of `/uf.init` (Speckit Command Guardrails) to
also inject the UF references into the five upstream speckit
commands (Option B).**

Step 6 is the authoritative source-of-truth layer for durable
UF customization of speckit command files. The sibling edit issue
targets Step 6 in the canonical asset
`internal/scaffold/assets/opencode/commands/uf.init.md` (which IS
embedded and drift-tested), NOT the installed `.opencode/commands/`
copies.

Reference matrix the injection MUST enforce:

| Command | Dewey | Constitution awareness | `/uf.review-council` |
|---------|-------|------------------------|----------------------|
| `speckit.specify.md` | required | required | — |
| `speckit.plan.md` | required (exemplar) | required (exemplar) | — |
| `speckit.tasks.md` | required | required | — |
| `speckit.implement.md` | required | required | required |
| `speckit.constitution.md` | required | required | — |

`speckit.plan.md` is the compliant exemplar (Dewey + constitution
pattern already present) and defines the target reference shape.

The injection MUST reuse Step 6's existing discipline:

- Idempotent read → check → inject/correct with a per-reference
  presence check (not a naive append).
- A stable correctness marker per reference, consistent with the
  existing Guardrails correctness-marker table
  (`uf.init.md:591-595`), so re-running `/uf.init` neither
  duplicates nor clobbers references.
- The Post-Write Verification gate MUST confirm no existing
  content was removed by the injection.

## Rationale

| Factor | Option A (upgrade Step 7) | **Option B (extend Step 6)** | Option C (embed the 5 files) |
|--------|---------------------------|------------------------------|------------------------------|
| Operational risk | Converts a read-only step into a mutating one; needs new idempotency guards | **Lowest — reuses tested inject/correct machinery** | High — changes the upstream/embedded boundary; upstream drift conflicts |
| Mutating loci | Two (Step 6 + newly-mutating Step 7) | **One (Step 6 remains the single mutating locus for speckit commands)** | N/A |
| Idempotency | New markers required | **Existing marker pattern extended** | N/A |
| Test-observability | Achievable | **Achievable — same regenerate-cycle assertion path as guardrails** | Divergence from upstream |
| Upstream compatibility | Preserved | **Preserved** | Broken — we would own upstream files |

Option B keeps a single auditable mutating locus, reuses the
already-proven idempotent injection/marker/skip logic, and leaves
Step 7 as the read-only verifier (a useful independent check that
the injection succeeded). Option A was favored by some reviewers
for concentrating all UF-reference logic in one place, but it
broadens Step 7's surface and requires duplicating idempotency
machinery that already exists in Step 6. Option C was rejected
because embedding the upstream files would make us the owner of
files that upstream `specify init` regenerates, creating recurring
merge/drift conflicts.

The Divisor review panel triaged #546 as VALID / feature /
objective (5/5), with every premise verified against the working
tree.

## Consequences

**Positive:**
- Edits to UF references become durable across `specify init` +
  `/uf.init` regeneration.
- A single mutating locus (Step 6) governs all durable
  customization of speckit command files.
- Step 7 remains a read-only verifier, providing independent
  confirmation that the injection is present.

**Trade-offs:**
- Step 6 grows in scope (guardrails + UF references). This is
  acceptable because both are the same class of operation
  (idempotent, marker-based injection into speckit commands).

**Follow-up (sibling issues, NOT this ADR):**
- The reconciliation **edit** MUST target Step 6 in the canonical
  asset `internal/scaffold/assets/opencode/commands/uf.init.md`.
- A **regenerate-cycle regression test** MUST prove the three UF
  references survive a `specify init` → `/uf.init` cycle. The
  chosen mechanism MUST be test-observable; this requirement is a
  MEDIUM-severity acceptance criterion on the sibling edit issue.
- Per constitution Gatekeeping Integrity, the injection MUST NOT
  weaken any existing guardrail or verification gate.

## Alternatives Considered

| Option | Verdict | Why |
|--------|---------|-----|
| Option A — upgrade Step 7 to enforcing injection | Rejected | Converts a clean read-only verifier into a mutating step; duplicates idempotency machinery already in Step 6; broadens operational surface |
| Option B — extend Step 6 post-processing | **Accepted** | Lowest operational risk; single mutating locus; reuses tested idempotent inject/correct pattern; preserves upstream compatibility and Step 7's independent verification |
| Option C — embed the 5 upstream files | Rejected | Breaks the upstream/embedded boundary; we would own files upstream regenerates, causing recurring drift/merge conflicts |

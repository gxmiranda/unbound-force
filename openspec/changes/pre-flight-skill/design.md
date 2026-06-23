## Context

Three commands (`/review-council`, `/review-pr`, `/unleash`) each
independently implement CI workflow detection and local tool
execution logic. The duplication manifests as two distinct
approaches:

1. **Workflow-driven** (`/review-council` Phase 1a, `/unleash`
   Step 5): Parse `.github/workflows/*.yml` to discover the exact
   commands CI runs, then replay them locally.
2. **Config-file-driven** (`/review-pr` Step 4): Detect tools by
   checking for configuration files (`Makefile`, `.golangci.yml`,
   `ruff.toml`, etc.), then build a CI coverage matrix against
   GitHub PR check results to decide what to skip.

Both approaches share the same goal — run local quality checks
before proceeding — but neither command benefits from the other's
strengths.

See proposal.md for motivation and constitution alignment.

## Goals / Non-Goals

### Goals
- Single source of truth for CI detection and local tool execution
- Both execution policies (`hard-gate`, `ci-aware`) available to
  all consumers
- CI coverage matrix output available in both modes
- No behavioral regression for previously-detected tools; expanded
  detection for tools missed by single-strategy approaches is an
  intentional improvement
- Scaffold copies stay in sync with command and skill sources

### Non-Goals
- Executable code (Go package, script) — this is an agent
  instruction skill only
- `detect-only` mode for `/agent-brief` — tracked as follow-up
  per issue #175
- Gaze integration — `/review-council` Phase 1b (Gaze quality
  analysis) remains in the command; it is not CI detection logic
- Changing which tools are detected or which commands are run —
  the skill consolidates existing behavior, not new behavior

## Decisions

### D1: Skill file, not Go code

The pre-flight logic lives entirely in agent instruction files
(markdown). There is no compiled code to extract. The skill will
be a `.opencode/skills/pre-flight/SKILL.md` file that consuming
commands reference via the `skill` tool.

**Rationale**: Matches the existing pattern — all current skills
are SKILL.md instruction files. No new tooling or runtime
infrastructure needed. Aligns with Composability First: any
command can load the skill independently.

### D2: Unify both detection approaches

The skill will use **both** detection strategies in sequence:

1. **Workflow parsing** (primary): Read `.github/workflows/*.yml`
   to discover CI commands. This is the source of truth for what
   CI actually runs.
2. **Config-file detection** (supplementary): Check for tool
   configuration files to discover tools that may not appear in
   CI workflows (e.g., a `.yamllint.yml` with no CI job).

This unified approach gives every consumer the most complete
picture. Currently `/review-council` only does (1) and
`/review-pr` only does (2).

**Rationale**: Neither approach alone is sufficient. Workflow
parsing misses tools not in CI. Config-file detection misses
CI-only checks (e.g., coverage ratchets). Combining them
produces a superset.

### D3: Two execution policies via mode parameter

The skill defines two modes that consuming commands select:

| Mode | Behavior | Current consumer |
|------|----------|-----------------|
| `hard-gate` | Run all detected tools. Stop on first failure. | `/review-council`, `/unleash` |
| `ci-aware` | Build CI coverage matrix against PR check results. Skip tools CI already verified. Run the rest. | `/review-pr` |

The mode is selected by the consuming command's instructions,
not by the skill itself. The skill describes both policies; the
command tells the agent which to use.

**Rationale**: The two modes serve different lifecycle stages
(pre-PR vs. post-PR). Forcing a single mode would break
existing behavior. Making the mode a parameter preserves
behavioral parity.

### D4: Standardized result format

The skill defines a standard output structure for pre-flight
results:

```
## Pre-flight Results

### CI Coverage Matrix
| Local tool | CI check | CI status | Run locally? |
|------------|----------|-----------|--------------|
| ...        | ...      | ...       | ...          |

### Execution Results
| Tool | Command | Exit code | Status |
|------|---------|-----------|--------|
| ...  | ...     | ...       | ...    |

### Verdict
- **Mode**: hard-gate | ci-aware
- **Result**: PASS | FAIL
- **Failures**: [list if any]
```

This format is consumed by the calling command to decide whether
to proceed (hard-gate) or to include failure context in AI review
(ci-aware).

**Rationale**: Aligns with Observable Quality — standardized,
structured output that any consumer can parse and act on
consistently.

### D5: Consuming commands reference skill, keep policy selection

Each consuming command replaces its inline pre-flight logic with:

1. A reference to load the `pre-flight` skill
2. The mode selection (`hard-gate` or `ci-aware`)
3. Command-specific behavior on the result (stop vs. continue
   with context)

The skill does NOT replace command-specific logic that happens
after pre-flight (e.g., `/review-council` Phase 1b Gaze analysis,
`/review-pr` Step 5 diff fetching).

### D6: Scaffold sync as explicit task

Both `.opencode/commands/` and
`internal/scaffold/assets/opencode/commands/` contain identical
copies of each command file. Changes to command files MUST be
mirrored to the scaffold copies. This is enforced by existing
drift detection tests.

### D7: Pre-flight skill is scaffold-deployed

The pre-flight skill is distributed via `uf init` (like
`speckit-workflow`), not local-only (like the Replicator
skills). This means:

1. A scaffold asset copy MUST exist at
   `internal/scaffold/assets/opencode/skills/pre-flight/SKILL.md`
2. The path MUST be added to `expectedAssetPaths` in
   `scaffold_test.go`
3. Drift detection tests enforce sync between the live
   skill and the scaffold copy

**Rationale**: The pre-flight skill is useful in any project
that uses the review commands — it should be scaffolded into
new projects alongside the commands that consume it.

### D8: Workflow command extraction boundaries

When parsing `.github/workflows/*.yml`, the skill extracts
only `run:` step commands. It does NOT execute:

- Action references (`uses:` steps) — these are GitHub-hosted
  and not locally executable
- Commands containing unresolvable CI expressions
  (`${{ secrets.* }}`, `${{ github.* }}`) — these depend on
  CI runtime context
- Commands that reference CI-only tools not available locally
  — these are reported as "not locally available" rather than
  treated as failures

When a config file is detected but the corresponding tool
binary is not in PATH, the tool is reported as "detected but
not available" and skipped with a warning rather than treated
as a hard failure. This preserves the existing behavior where
developers may not have all tools installed locally.

## Risks / Trade-offs

### R1: Skill loading adds indirection

Commands that previously had self-contained pre-flight logic now
depend on loading a skill. If the skill file is missing or
malformed, the command will fail differently than before.

**Mitigation**: The skill is checked into the repository and
synced via scaffold. Drift detection tests catch missing or
out-of-sync files.

### R2: Unified detection may change behavior

Combining workflow parsing + config-file detection means
`/review-council` may discover tools it previously missed (e.g.,
yamllint), and `/review-pr` may discover CI commands it previously
missed (e.g., coverage ratchets).

**Mitigation**: This is a net positive — more complete detection
is better. However, the behavioral change should be noted in
testing. Consumers should verify that newly-discovered tools
do not cause unexpected failures.

### R3: Instruction-only skill cannot be unit tested

The skill is markdown instructions, not executable code. There
are no functions to test in isolation.

**Mitigation**: Verification is behavioral — run each consuming
command against a real branch before and after the change,
confirm identical outcomes for existing tools and improved
detection for previously-missed tools.

### R4: Single point of failure

A defect in the skill affects all three consumers
simultaneously. Previously, a bug in one command's pre-flight
logic only affected that command.

**Mitigation**: Git revert restores the previous inline logic.
The skill is instruction-only, so no binary rebuild is needed.
The risk is acceptable given the consolidation benefit.

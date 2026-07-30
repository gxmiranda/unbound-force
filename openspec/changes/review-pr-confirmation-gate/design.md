## Context

The `/review-pr` command (`.opencode/commands/review-pr.md`)
defines a multi-step workflow for AI-assisted PR reviews. Step
11 contains a CRITICAL RULE requiring explicit human
confirmation via AskUserQuestion before posting any review to
GitHub. This gate exists because the review is posted under
the human's GitHub identity and can unblock merges.

In practice, when the agent's session context is compressed
(e.g., after a long review), the confirmation gate language
can be summarized away by the compression algorithm. The agent
then proceeds to post the review without stopping to ask for
permission. This occurred on PR gaze#194 where an APPROVE
review was posted without consent.

The proposal (see `proposal.md`) defines two hardening
measures: a session-resume guard and a visual gate marker.
This design documents the technical approach.

## Goals / Non-Goals

### Goals
- Make the confirmation gate survive context compression
  by using structural markers that resist summarization.
- Add explicit re-confirmation semantics when the agent
  detects it is operating from compressed or resumed
  context.
- Preserve the existing confirmation flow for the normal
  (uncompressed) case — no new friction for the happy path.

### Non-Goals
- Changing how context compression works (that is an LLM
  platform concern, not a command spec concern).
- Adding programmatic enforcement (e.g., a pre-post hook
  that checks for confirmation). The agent spec is the
  enforcement mechanism for agent commands.
- Modifying any other `/review-*` or `/address-feedback`
  commands (those can be hardened separately if needed).

## Decisions

### D1: Guard placement — top of Step 11

The session-resume guard block is placed at the very top
of Step 11, before any confirmation flow logic. This
ensures the guard is the first thing the agent processes
when entering the posting step, regardless of what was
compressed from earlier steps.

**Rationale**: If the guard were embedded within the
existing confirmation flow (items 3-5), compression could
summarize the surrounding context and the guard together.
Placing it as a standalone block at the top gives it
maximum independence from surrounding prose.

### D2: Visual gate marker — `>>> MANDATORY GATE <<<`

The marker uses a distinctive ASCII delimiter pattern
rather than emoji or Unicode symbols:

```
>>> MANDATORY GATE: HUMAN CONFIRMATION REQUIRED <<<
```

This pattern is chosen because:
- ASCII delimiters are structurally distinct from prose
  and less likely to be compressed into a summary.
- The pattern is grep-able and lint-able.
- It does not depend on terminal emoji rendering.

The marker wraps the entire confirmation gate section
(items 2-5 of the current Step 11), creating a visually
bounded region.

### D3: Re-confirmation condition

The guard uses a negative-default approach: the agent
MUST re-confirm unless it can verify that explicit human
confirmation occurred in the current, uncompressed
conversation history. This means:

- If the agent is resuming from compressed context →
  MUST re-confirm.
- If the agent cannot determine whether prior
  confirmation occurred → MUST re-confirm.
- If the agent has uncompressed evidence of prior
  confirmation → MAY proceed (but the existing Step 11
  flow already handles this case).

**Rationale**: False negatives (re-confirming when not
strictly necessary) are harmless — the human clicks a
button. False positives (proceeding without confirmation)
are the actual harm case. Negative-default eliminates
false positives.

### D4: No structural changes to existing flow

The existing Step 11 items (1-5) remain unchanged in
their content and ordering. The guard and marker are
additive — they wrap the existing flow rather than
replacing it. This minimizes diff size and avoids
accidentally altering the well-tested confirmation
semantics.

**Constitution alignment**: Per the proposal, this change
is PASS on all five principles. The negative-default
approach (D3) specifically aligns with V. Security by
Default — the agent MUST NOT exercise the human's
authority without re-confirming permission when session
state is degraded.

## Risks / Trade-offs

### R1: Over-prompting

In edge cases where compression occurs mid-Step-11 (e.g.,
after comments are shown but before the user responds),
the guard may cause the agent to re-display all comments
and re-ask for confirmation. This is strictly preferable
to the alternative (posting without consent), but may be
mildly annoying.

**Mitigation**: The guard language is scoped to "posting"
specifically, not to the entire step. Re-displaying
comments is not harmful.

### R2: Compression resistance is heuristic

There is no guarantee that any marker pattern will survive
all possible compression algorithms. The `>>> MANDATORY
GATE <<<` pattern is designed to be hard to summarize but
a sufficiently aggressive compressor could still drop it.

**Mitigation**: The guard uses a negative-default (D3) —
if the marker is lost, the agent still cannot verify
prior confirmation and MUST re-confirm. The two
mechanisms (guard + marker) are defense-in-depth.

### R3: Other commands with similar gates

The `/review-council` and `/address-feedback` commands
may have similar confirmation gates that are vulnerable
to the same compression issue. This change only addresses
`/review-pr`.

**Mitigation**: Explicitly out of scope (see Non-Goals).
Separate changes can apply the same pattern to other
commands if needed.

## Why

The `/review-pr` command has a CRITICAL RULE (Step 11, item 5)
requiring explicit human confirmation via the AskUserQuestion
tool before posting any GitHub review. During a review of PR
gaze#194, the agent bypassed this gate when the session was
resumed from compressed context, posting an APPROVE review
under the human's account without consent.

This is a consent violation: the agent acted with the human's
GitHub identity (posting reviews, unblocking merges) without
the human explicitly authorizing the action. The root cause is
that the confirmation gate language is embedded in narrative
prose that context compression can summarize away, losing the
mandatory stop-and-ask semantics.

Fixes: #346

## What Changes

Two hardening measures are added to the `/review-pr` command
to make the confirmation gate survive context compression:

1. **Session-resume guard**: A standalone, prominently
   formatted block at the top of Step 11 that instructs the
   agent to re-confirm with the human if the session has been
   resumed from compressed context or if the agent cannot
   verify that the human explicitly confirmed in the current
   uncompressed conversation history.

2. **Visual gate marker**: A distinctive, non-prose marker
   (e.g., `🔒 HUMAN CONFIRMATION REQUIRED` or similar
   structural delimiter) that is harder for compression
   algorithms to summarize away than ordinary paragraph text.
   This marker wraps the existing CRITICAL RULE and
   AskUserQuestion requirement.

## Capabilities

### New Capabilities
- `session-resume-guard`: Explicit re-confirmation
  requirement when the agent detects compressed context
  or cannot verify prior human confirmation in the
  current conversation window.
- `visual-gate-marker`: Structural delimiter marking
  the confirmation gate as a non-summarizable mandatory
  stop point.

### Modified Capabilities
- `confirmation-gate` (Step 11, items 3-5): Existing
  confirmation flow is preserved but wrapped with the
  new guard and marker language.

### Removed Capabilities
- None.

## Impact

- **Files affected**: `.opencode/commands/review-pr.md`
  (lines ~879-956, Step 11 confirmation section)
- **Behavioral change**: Agents resuming from compressed
  context will be required to re-confirm before posting.
  No change to the happy path (uncompressed sessions
  already require confirmation).
- **Risk**: Low. This is additive guard language in a
  command spec. No Go code, no schema changes, no CI
  changes. The only risk is over-prompting the human in
  edge cases, which is strictly preferable to the
  alternative (posting without consent).

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

This change modifies a command spec artifact
(`review-pr.md`) — the standard medium through which
hero behavior is defined. The artifact remains
self-describing and does not introduce runtime coupling
between heroes.

### II. Composability First

**Assessment**: PASS

The `/review-pr` command is a standalone command within
this meta-repository. This change does not affect hero
composability or introduce cross-hero dependencies.

### III. Observable Quality

**Assessment**: PASS

The visual gate marker improves observability by making
the confirmation gate structurally visible rather than
buried in narrative prose. The guard language requires
the agent to surface its confirmation state to the
human, increasing transparency.

### IV. Testability

**Assessment**: PASS

The guard language defines a testable condition: "if
context is compressed and no prior confirmation is
verified, re-confirm." This is a verifiable behavioral
contract. The visual marker is a static artifact
property that can be verified by grep/lint.

### V. Security by Default

**Assessment**: PASS

This change directly addresses a security concern:
an agent acting with the human's GitHub identity
without explicit consent. The session-resume guard
enforces a least-privilege principle — the agent MUST
NOT exercise the human's authority (posting reviews)
without re-confirming permission, even when session
state is degraded by compression.

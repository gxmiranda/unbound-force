## ADDED Requirements

### Requirement: FR-001 Session-Resume Guard

The `/review-pr` command MUST include a session-resume
guard at the top of Step 11 (posting step). The guard
MUST instruct the agent to re-confirm with the human
via the AskUserQuestion tool before posting any review
if any of the following conditions are true:

1. The session has been resumed from compressed context.
2. The agent cannot verify that explicit human
   confirmation occurred in the current, uncompressed
   conversation history.

The guard MUST use a negative-default approach: the
agent MUST re-confirm unless it can positively verify
prior uncompressed confirmation.

#### Scenario: Compressed context resume

- **GIVEN** the agent is executing Step 11 of
  `/review-pr`
- **AND** the session has been resumed from compressed
  context (i.e., earlier conversation turns have been
  summarized)
- **WHEN** the agent reaches the confirmation gate
- **THEN** the agent MUST present the review content
  (verdict + comments) and use the AskUserQuestion
  tool to obtain explicit human confirmation before
  posting, regardless of any prior confirmation that
  may have been recorded in the compressed context.

#### Scenario: Uncertain confirmation state

- **GIVEN** the agent is executing Step 11 of
  `/review-pr`
- **AND** the agent cannot determine from the current
  uncompressed conversation whether the human
  previously confirmed the review
- **WHEN** the agent reaches the confirmation gate
- **THEN** the agent MUST re-confirm with the human
  via the AskUserQuestion tool.

#### Scenario: Normal uncompressed session

- **GIVEN** the agent is executing Step 11 of
  `/review-pr`
- **AND** the session has NOT been compressed
- **AND** no prior confirmation has been obtained
- **WHEN** the agent reaches the confirmation gate
- **THEN** the existing confirmation flow (items 2-5)
  applies unchanged.

### Requirement: FR-002 Visual Gate Marker

The confirmation gate section in Step 11 MUST be wrapped
with a distinctive structural delimiter that is resistant
to context compression. The marker MUST:

1. Use ASCII-only characters (no emoji or Unicode
   symbols that depend on terminal rendering).
2. Be visually distinct from surrounding prose.
3. Be grep-able for automated verification.

The opening marker format MUST be:

```
>>> MANDATORY GATE: HUMAN CONFIRMATION REQUIRED <<<
```

The closing marker format MUST be:

```
>>> END MANDATORY GATE <<<
```

The opening marker MUST appear before the confirmation
gate content (items 2-5 plus the CRITICAL RULE) and the
closing marker MUST appear after it.

#### Scenario: Marker present in spec

- **GIVEN** the `/review-pr` command spec
  (`.opencode/commands/review-pr.md`)
- **WHEN** a grep is run for the pattern
  `MANDATORY GATE`
- **THEN** exactly two matches MUST be found:
  one opening marker (`>>> MANDATORY GATE: HUMAN
  CONFIRMATION REQUIRED <<<`) and one closing marker
  (`>>> END MANDATORY GATE <<<`).

#### Scenario: Marker survives compression

- **GIVEN** the agent's session context is compressed
- **AND** the confirmation gate section was included
  in the compressed region
- **WHEN** the agent processes the compressed context
- **THEN** the marker SHOULD be preserved as a
  structural delimiter rather than summarized into
  prose (note: this is a best-effort heuristic, not
  a hard guarantee; FR-001's negative-default provides
  the safety net).

## MODIFIED Requirements

### Requirement: FR-003 CRITICAL RULE Scope

The existing CRITICAL RULE (Step 11, item 5) is
unchanged in its semantics. The rule text is preserved
verbatim within the new visual gate marker boundary.

Previously: The CRITICAL RULE appeared as item 5 in
Step 11 without any structural delimiter.

Now: The CRITICAL RULE appears as item 5 within the
`>>> MANDATORY GATE <<<` boundary, immediately preceded
by the session-resume guard (FR-001).

#### Scenario: Existing confirmation preserved

- **GIVEN** a normal (uncompressed) `/review-pr` session
- **WHEN** the agent reaches Step 11
- **THEN** the confirmation flow (show comments, ask
  via AskUserQuestion, post or skip) MUST behave
  identically to the pre-change behavior.

## REMOVED Requirements

None.

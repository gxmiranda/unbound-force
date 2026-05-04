## Design Decisions

### D1. Walkthrough placement

**Decision**: The `### Walkthrough` section appears after
`### Local Tool Results` and before `### Summary` in the
output format. The `### Summary` section shifts from
"overview" to "assessment summary" when a Walkthrough
is present — the Walkthrough provides structural
orientation, while the Summary provides the overall
assessment and verdict context.

**Rationale**: Reviewers need structural orientation
before reading findings. Placing the walkthrough early
gives them the "what changed" map before the "what's
wrong" analysis. The walkthrough lists every changed file
with a one-line description of the change, organized by
directory grouping.

For PRs with 30+ files, the walkthrough groups by
directory with file counts instead of individual file
listings to limit token consumption.

### D2. Suggestion blocks — scope of use

**Decision**: Use GitHub suggestion blocks only for
concrete, literal code replacements. Never for
architectural suggestions, design recommendations,
multi-file changes, or removal of security controls.

**Rationale**: GitHub's suggestion syntax replaces exact
lines. It works for "change this function call to use
the correct argument" but not for "redesign this module
to use dependency injection." Misuse produces broken
suggestions that frustrate rather than help. The command
MUST include the original code lines and the replacement
in the suggestion block for one-click application.

Security control removal (input validation, auth checks,
lint suppressions) MUST NOT be suggested via one-click
blocks because the reduced friction makes accidental
acceptance of security weakening too easy. These
recommendations use plain text with explicit reasoning.

Each suggestion block appears in the Step 11 human
review preview with its full before/after context, so
the human can evaluate the replacement before it is
posted.

### D3. Path focus — built-in heuristics

**Decision**: Path-based review focus uses built-in
heuristics derived from common directory conventions, not
a user-managed config file.

**Rationale**: Adding a new config file would introduce
a dependency and setup burden. Built-in heuristics follow
the "convention over configuration" principle — they
work immediately without setup. The existing convention
pack system (`go.md`, `typescript.md`) handles
language-specific rules; path focus adds the
directory-specific layer on top.

These heuristics are intentionally minimal and
language-agnostic. Language-specific path patterns
SHOULD be contributed via convention pack evolution
(e.g., `go.md` could define Go-specific path patterns)
rather than expanding the built-in heuristic table.

**Heuristics**:

| Path pattern | Focus |
|-------------|-------|
| `*_test.go`, `*_test.py`, `**/__tests__/**`, `**/*_spec.*` | Test quality: edge cases, assertion strength, mock isolation, test naming |
| `**/cmd/**`, `**/cli/**` | UX: error messages, flag validation, help text |
| `**/api/**`, `**/handler/**`, `**/middleware/**`, `**/routes/**` | Security: auth, input validation, injection |
| `*.md`, `docs/**` | Clarity, accuracy, broken links |
| `.github/workflows/**`, `Dockerfile*` | CI/CD: permissions, pinned versions, secrets exposure |
| `go.mod`, `package.json`, `requirements.txt` | Dependencies: maintenance status, license, scope |
| Everything else | Standard review: architecture, SOLID, coupling, baseline security |

These heuristics are applied as additive context during
AI review — they do not replace the standard review
categories (alignment, security, constitution). Step 8b
(Security Review) applies to ALL changed files
regardless of path heuristic. They add targeted emphasis
for files where specific concerns are most relevant.

The path classification for each file is visible in the
`### Walkthrough` table's Focus column, making the
heuristic assignment observable and auditable.

### D4. Issue linking — pattern matching

**Decision**: Parse `Fixes #N`, `Closes #N`, and
`Resolves #N` (case-insensitive) and their GitHub URL
variants (`Fixes https://github.com/.../issues/N`) from
the PR body. Limit to 5 issues. Same-repo only for URL
variants.

**Rationale**: These are the patterns GitHub recognizes
for automatic issue closing. Supporting them means the
command validates exactly what GitHub will act on when
the PR merges. Other patterns (e.g., "Related to #N")
are intentionally excluded — they indicate association,
not commitment to resolve.

URL-format references to other repositories are listed
but not fetched — the command cannot reliably access
cross-repo issues (permissions, different org settings)
and injecting external content increases the attack
surface.

**Issue body as untrusted input**: Issue bodies are
user-controlled content. They are truncated to 2000
characters before being incorporated into the review
context. This limits prompt injection risk and context
window consumption.

**Acceptance criteria extraction**: The command looks for
checkbox lines (`- [ ]` or `- [x]`) and content under
an `## Acceptance Criteria` heading in the issue body.
If neither exists, the issue title and body are used as
general intent context for the alignment check.

**Error handling**: If `gh issue view` returns 404, 403,
or a timeout, the issue is skipped with a "fetch failed"
note in the output. The review continues without
blocking.

### D5. Verdict-aligned posting — single API call

**Decision**: Submit verdict and in-line comments as a
single review event via the GitHub API, not as separate
`gh pr review` calls.

**Rationale**: A single API call creates one review event
in GitHub's timeline. Two separate calls (one for
verdict, one for comments) would create two events, which
is noisy. The `gh api repos/{owner}/{repo}/pulls/<N>/reviews`
endpoint accepts the event type, body text, and inline
comments array in one payload.

**Threat model change**: This enhancement changes the
command's role from passive observer (COMMENT only) to
active merge gate participant (can post APPROVE or
REQUEST_CHANGES). This is an intentional, risk-accepted
change mitigated by: (1) MUST-level human confirmation,
(2) explicit "approve" text required for APPROVE verdicts,
(3) AI-generated label in review body, (4) graceful
fallback to COMMENT on permission failures.

**Mapping**:

| Verdict | API event type | GitHub effect |
|---------|---------------|---------------|
| APPROVE | `"event": "APPROVE"` | Green checkmark, unblocks merge |
| REQUEST CHANGES | `"event": "REQUEST_CHANGES"` | Red X, blocks merge |
| COMMENT | `"event": "COMMENT"` | Neutral, no merge effect |

**Graceful degradation**: If the API returns 403 or 422
(insufficient permissions, non-collaborator, or
self-review prohibition), the command falls back to
posting as COMMENT with the original verdict noted in the
body. This preserves the existing behavior as a fallback.

### D6. Human confirmation gate — verdict awareness

**Decision**: The human confirmation gate is preserved
and enhanced to show the verdict type being posted.
APPROVE verdicts require explicit "approve" text input.

**Rationale**: Posting APPROVE or REQUEST_CHANGES has
stronger consequences than COMMENT — it changes the PR's
merge eligibility in repos with branch protection. The
user must be explicitly aware of and approve the verdict
type before it's posted. The confirmation prompt changes
from:

```
Post these comments? (yes/no/edit)
```

To:

```
Post review as REQUEST CHANGES with N comments?
(yes/no/edit/change-verdict)
```

For APPROVE verdicts specifically:

```
Post review as APPROVE with N comments?
⚠ This may unblock merge in repos with branch protection.
Type "approve" to confirm, or choose another option:
(approve/no/edit/change-verdict)
```

The `change-verdict` option lets the user downgrade
(e.g., from REQUEST CHANGES to COMMENT) if they
disagree with the computed verdict.

All posted reviews include an AI-generated label:
`_This review was generated by /review-pr
(AI-assisted)._` This allows other reviewers and merge
gatekeepers to distinguish AI reviews from human reviews.

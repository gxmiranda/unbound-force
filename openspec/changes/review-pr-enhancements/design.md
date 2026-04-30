## Design Decisions

### D1. Walkthrough placement

**Decision**: The `### Walkthrough` section appears after
`### Local Tool Results` and before `### Summary` in the
output format.

**Rationale**: Reviewers need structural orientation
before reading findings. Placing the walkthrough early
gives them the "what changed" map before the "what's
wrong" analysis. The walkthrough lists every changed file
with a one-line description of the change, organized by
directory grouping.

### D2. Suggestion blocks — scope of use

**Decision**: Use GitHub suggestion blocks only for
concrete, literal code replacements. Never for
architectural suggestions, design recommendations, or
multi-file changes.

**Rationale**: GitHub's suggestion syntax replaces exact
lines. It works for "change this function call to use
the correct argument" but not for "redesign this module
to use dependency injection." Misuse produces broken
suggestions that frustrate rather than help. The command
MUST include the original code lines and the replacement
in the suggestion block for one-click application.

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

**Heuristics**:

| Path pattern | Focus |
|-------------|-------|
| `*_test.go`, `*_test.py`, `**/__tests__/**`, `**/*_spec.*` | Test quality: edge cases, assertion strength, mock isolation, test naming |
| `**/cmd/**`, `**/cli/**` | UX: error messages, flag validation, help text |
| `**/api/**`, `**/handler/**`, `**/middleware/**`, `**/routes/**` | Security: auth, input validation, injection |
| `*.md`, `docs/**` | Clarity, accuracy, broken links |
| `.github/workflows/**`, `Dockerfile*` | CI/CD: permissions, pinned versions, secrets exposure |
| `go.mod`, `package.json`, `requirements.txt` | Dependencies: maintenance status, license, scope |
| Everything else | Standard review: architecture, SOLID, coupling |

These heuristics are applied as additive context during
AI review — they do not replace the standard review
categories (alignment, security, constitution). They
add targeted emphasis for files where specific concerns
are most relevant.

### D4. Issue linking — pattern matching

**Decision**: Parse `Fixes #N`, `Closes #N`, and
`Resolves #N` (case-insensitive) and their GitHub URL
variants (`Fixes https://github.com/.../issues/N`) from
the PR body.

**Rationale**: These are the patterns GitHub recognizes
for automatic issue closing. Supporting them means the
command validates exactly what GitHub will act on when
the PR merges. Other patterns (e.g., "Related to #N")
are intentionally excluded — they indicate association,
not commitment to resolve.

**Acceptance criteria extraction**: The command looks for
checkbox lines (`- [ ]` or `- [x]`) and content under
an `## Acceptance Criteria` heading in the issue body.
If neither exists, the issue title and body are used as
general intent context for the alignment check.

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

**Mapping**:

| Verdict | API event type | GitHub effect |
|---------|---------------|---------------|
| APPROVE | `"event": "APPROVE"` | Green checkmark, unblocks merge |
| REQUEST CHANGES | `"event": "REQUEST_CHANGES"` | Red X, blocks merge |
| COMMENT | `"event": "COMMENT"` | Neutral, no merge effect |

### D6. Human confirmation gate — verdict awareness

**Decision**: The human confirmation gate is preserved
and enhanced to show the verdict type being posted.

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

The `change-verdict` option lets the user downgrade
(e.g., from REQUEST CHANGES to COMMENT) if they
disagree with the computed verdict.

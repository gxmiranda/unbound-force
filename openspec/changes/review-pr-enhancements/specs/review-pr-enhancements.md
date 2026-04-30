## Overview

Enhance the `/review-pr` command with 5 improvements:
PR walkthrough, GitHub suggestion blocks, path-based
review focus, issue linking, and verdict-aligned review
posting.

## Functional Requirements

### Walkthrough

- **FR-016** [MUST] The output MUST include a
  `### Walkthrough` section listing each changed file
  with a one-line summary of what changed. This section
  MUST appear after `### Local Tool Results` and before
  `### Summary` in the output format. Files SHOULD be
  grouped by directory.

### Suggestion Blocks

- **FR-017** [SHOULD] When preparing in-line comments with
  concrete code fix suggestions, the comment body SHOULD
  use GitHub's suggestion block syntax (triple-backtick
  `suggestion`). Suggestion blocks MUST only be used for
  literal code replacements that can be applied as-is.
  Architectural, design, or multi-file suggestions MUST
  use plain text.

### Path-based Review Focus

- **FR-018** [SHOULD] During AI review (Step 8), the
  command SHOULD apply targeted review instructions based
  on file path patterns using built-in heuristics:
  - Test files (`*_test.go`, `*_test.py`,
    `**/__tests__/**`, `**/*_spec.*`) → test quality:
    edge cases, assertion strength, mock isolation
  - API/handler files (`**/api/**`, `**/handler/**`,
    `**/middleware/**`, `**/routes/**`) → security:
    auth, input validation, injection
  - CLI files (`**/cmd/**`, `**/cli/**`) → UX: error
    messages, flag validation, help text
  - Documentation (`*.md`, `docs/**`) → clarity,
    accuracy, broken links
  - CI/CD config (`.github/workflows/**`,
    `Dockerfile*`) → permissions, pinned versions,
    secrets exposure
  - Dependency files (`go.mod`, `package.json`,
    `requirements.txt`) → maintenance, license, scope
  - All other files → standard review (architecture,
    coupling, SOLID)

  No configuration file is required. Path focus is
  additive to the standard review categories, not a
  replacement.

### Issue Linking

- **FR-019** [SHOULD] The command SHOULD parse
  `Fixes #N`, `Closes #N`, and `Resolves #N` patterns
  (case-insensitive) from the PR description, including
  GitHub URL variants
  (`Fixes https://github.com/.../issues/N`). For each
  linked issue, fetch its title, body, and labels via
  `gh issue view <N> --json title,body,labels`.

- **FR-020** [SHOULD] For each linked issue, extract
  acceptance criteria: checkbox lines (`- [ ]` or
  `- [x]`) or content under an `## Acceptance Criteria`
  heading. During alignment checking (Step 8a), validate
  that the PR addresses each criterion. Report uncovered
  criteria as MEDIUM findings.

- **FR-021** [MUST] The output MUST include a
  `### Linked Issues` section (after `### Walkthrough`,
  before `### Summary`) listing each linked issue with
  its title and criteria coverage status. If no issues
  are linked, omit this section entirely.

### Verdict-aligned Review Posting

- **FR-022** [MUST] When posting the review to GitHub,
  the command MUST use the verdict-appropriate review
  event type: APPROVE for APPROVE verdict,
  REQUEST_CHANGES for REQUEST CHANGES verdict, COMMENT
  for COMMENT verdict.

- **FR-023** [MUST] Verdict and in-line comments MUST be
  submitted as a single review event via the GitHub API
  (`gh api repos/{owner}/{repo}/pulls/<N>/reviews`). The
  JSON payload MUST include the event type, body text
  (review summary), and inline comments array.

- **FR-024** [MUST] The human confirmation prompt MUST
  display the verdict type being posted (e.g., "Post
  review as REQUEST CHANGES with N comments?"). The
  prompt MUST offer a `change-verdict` option that lets
  the user override the computed verdict before posting.

- **FR-025** [SHOULD] The confirmation prompt SHOULD warn
  the user when posting APPROVE or REQUEST_CHANGES since
  these affect merge eligibility in repos with branch
  protection rules.

## Acceptance Scenarios

### SC-010: Walkthrough in output

Given a PR changing 5 files across 2 directories
When the review completes
Then the output includes a `### Walkthrough` section
  listing each file with a one-line change summary,
  grouped by directory, before the `### Summary` section.

### SC-011: Suggestion blocks in comments

Given a finding with a concrete single-file code fix
When the user approves posting in-line comments
Then the comment body uses GitHub's suggestion block
  syntax so the fix is one-click applicable in the
  GitHub UI.

### SC-012: Suggestion blocks not used for design issues

Given a finding that recommends an architectural change
  (e.g., "extract this into a separate package")
When the comment is prepared
Then the comment uses plain text, not a suggestion block.

### SC-013: Path-based focus on test files

Given a PR that modifies `internal/gateway/gateway_test.go`
When the AI review runs
Then the review applies test quality focus (edge cases,
  assertion strength, mock isolation) to that file in
  addition to the standard review categories.

### SC-014: Path-based focus — no special path

Given a PR that modifies `internal/config/config.go`
  (no matching path heuristic)
When the AI review runs
Then the file receives the standard review treatment
  (architecture, SOLID, coupling) with no additional
  path-based focus.

### SC-015: Issue linking with acceptance criteria

Given a PR with `Fixes #42` in the description and
  issue #42 has 4 acceptance criteria checkboxes
When the alignment check runs
Then each criterion is validated against the PR's changes
  and uncovered criteria are reported as MEDIUM findings
  in the `### Alignment` section.

### SC-016: Issue linking — no linked issues

Given a PR without `Fixes/Closes/Resolves` patterns
When the review runs
Then no `### Linked Issues` section appears and no issue
  linking errors are reported.

### SC-017: Verdict posted as REQUEST CHANGES

Given a review with HIGH findings and a REQUEST CHANGES
  verdict
When the user approves posting
Then the review is submitted via the GitHub API with
  `"event": "REQUEST_CHANGES"` and the PR shows
  "Changes requested" status in GitHub.

### SC-018: Verdict override

Given a review with a computed REQUEST CHANGES verdict
When the user selects `change-verdict` at the
  confirmation prompt and chooses COMMENT
Then the review is posted as COMMENT instead of
  REQUEST_CHANGES.

### SC-019: Verdict confirmation warns about merge impact

Given a review with an APPROVE verdict
When the confirmation prompt is displayed
Then it includes a note that posting APPROVE may
  unblock merge in repos with branch protection.

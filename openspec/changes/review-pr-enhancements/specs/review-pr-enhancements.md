## Overview

Enhance the `/review-pr` command with 5 improvements:
PR walkthrough, GitHub suggestion blocks, path-based
review focus, issue linking, and verdict-aligned review
posting. FR numbering continues from the original
`/review-pr` command spec (FR-001 through FR-015).

## Functional Requirements

### Walkthrough

- **FR-016** [MUST] The output MUST include a
  `### Walkthrough` section listing each changed file
  with a one-line summary of what changed. This section
  MUST appear after `### Local Tool Results` and before
  `### Summary` in the output format. Files SHOULD be
  grouped by directory. For PRs with more than 30
  changed files, the walkthrough MAY group files by
  directory with a directory-level summary instead of
  per-file summaries to limit token consumption.

### Suggestion Blocks

- **FR-017** [SHOULD] When preparing in-line comments with
  concrete code fix suggestions, the comment body SHOULD
  use GitHub's suggestion block syntax (triple-backtick
  `suggestion`). Suggestion blocks MUST only be used for
  literal code replacements that can be applied as-is.
  Architectural, design, or multi-file suggestions MUST
  use plain text. Suggestion blocks MUST NOT propose
  removal of security controls (input validation,
  authentication checks, error handling, lint
  suppressions). If a suggested fix involves removing a
  security-relevant line, use plain text with an
  explanation instead.

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
    coupling, SOLID, baseline security)

  No configuration file is required. Path focus is
  additive to the standard review categories (alignment,
  security, constitution), not a replacement. Step 8b
  (Security Review) applies to ALL changed files
  regardless of path heuristic — the path heuristic
  adds emphasis, it does not define the security review
  scope.

### Issue Linking

- **FR-019** [SHOULD] The command SHOULD parse
  `Fixes #N`, `Closes #N`, and `Resolves #N` patterns
  (case-insensitive) from the PR description, including
  GitHub URL variants
  (`Fixes https://github.com/.../issues/N`). Parsed
  issue numbers MUST be validated as positive integers
  before use in any `gh` command. URL-format issue
  references MUST be validated to belong to the same
  `{owner}/{repo}` as the PR; cross-repository
  references MUST be listed but not fetched. Issue
  linking MUST be limited to a maximum of 5 linked
  issues per PR; additional references beyond 5 MUST
  be listed but not fetched. For each in-scope linked
  issue, fetch its title, body, and labels via
  `gh issue view <N> --json title,body,labels`.

- **FR-019a** [MUST] Issue body content fetched via
  `gh issue view` MUST be treated as untrusted input.
  Before incorporating it into the review context,
  truncate to a maximum of 2000 characters. If
  `gh issue view` returns an error (404, 403, or
  timeout), log the error, skip that issue, and note
  in the `### Linked Issues` section that the issue
  could not be fetched. The review MUST continue
  without blocking.

- **FR-020** [SHOULD] For each linked issue, extract
  acceptance criteria: checkbox lines (`- [ ]` or
  `- [x]`) or content under an `## Acceptance Criteria`
  heading. If neither exists, the issue title and body
  are used as general intent context. During alignment
  checking (Step 8a), validate that the PR addresses
  each criterion using AI judgment. Report uncovered
  criteria as MEDIUM findings with a per-criterion
  coverage status (COVERED / NOT COVERED / PARTIAL).

- **FR-021** [MUST] The output MUST include a
  `### Linked Issues` section (after `### Walkthrough`,
  before `### Summary`) listing each linked issue with
  its title and criteria coverage status. Issues that
  could not be fetched MUST be listed with status
  "fetch failed". If no issues are linked, omit this
  section entirely.

### Verdict-aligned Review Posting

- **FR-022** [MUST] When posting the review to GitHub,
  the command MUST use the verdict-appropriate review
  event type: APPROVE for APPROVE verdict,
  REQUEST_CHANGES for REQUEST CHANGES verdict, COMMENT
  for COMMENT verdict. If the `gh api` call returns
  HTTP 403 or 422 (insufficient permissions or
  non-collaborator), fall back to posting as COMMENT
  event type with a note explaining why the verdict
  could not be applied.

- **FR-023** [MUST] Verdict and in-line comments MUST be
  submitted as a single review event via the GitHub API
  (`gh api repos/{owner}/{repo}/pulls/<N>/reviews`). The
  JSON payload MUST be constructed in a temporary file
  and passed via `--input <json-file>`, consistent with
  the existing shell injection prevention pattern. The
  existing 15-comment cap from Step 11 MUST be
  preserved. Error responses from the GitHub API (403,
  422, 404) MUST be surfaced to the user with actionable
  guidance.

- **FR-024** [MUST] The human confirmation prompt MUST
  display the verdict type being posted (e.g., "Post
  review as REQUEST CHANGES with N comments?"). The
  prompt MUST offer a `change-verdict` option that lets
  the user override the computed verdict before posting.

- **FR-025** [MUST] The confirmation prompt MUST warn
  the user when posting APPROVE or REQUEST_CHANGES since
  these affect merge eligibility in repos with branch
  protection rules. When the computed verdict is APPROVE,
  the confirmation MUST require the user to explicitly
  type "approve" (not just "yes") to prevent reflexive
  confirmation.

- **FR-026** [MUST] The review body posted to GitHub MUST
  include a line identifying it as AI-generated:
  `_This review was generated by /review-pr
  (AI-assisted)._` This allows other reviewers and
  merge gatekeepers to distinguish AI reviews from
  human reviews.

## Acceptance Scenarios

### SC-010: Walkthrough in output

Given a PR changing 5 files across 2 directories
When the review completes
Then the output includes a `### Walkthrough` section
  with a table containing File, Change, and Focus
  columns, listing each file with a one-line change
  summary grouped by directory, before the `### Summary`
  section.

### SC-011: Suggestion blocks in comments

Given a finding with a concrete single-file code fix
When the user approves posting in-line comments
Then the comment body contains a fenced code block with
  the `suggestion` language identifier (triple-backtick
  `suggestion`) wrapping the replacement code, and the
  original code lines are included for GitHub's diff
  rendering.

### SC-012: Suggestion blocks not used for design issues

Given a finding that recommends an architectural change
  (e.g., "extract this into a separate package")
When the comment is prepared
Then the comment uses plain text, not a suggestion block.

### SC-013: Path-based focus on test files

Given a PR that modifies `internal/gateway/gateway_test.go`
When the AI review runs
Then the `### Walkthrough` table shows `test-quality` in
  the Focus column for that file, and the review applies
  test quality focus (edge cases, assertion strength,
  mock isolation) to that file in addition to the
  standard review categories.

### SC-014: Path-based focus — no special path

Given a PR that modifies `internal/config/config.go`
  (no matching path heuristic beyond default)
When the AI review runs
Then the `### Walkthrough` table shows `standard` in
  the Focus column for that file, and it receives the
  standard review treatment (architecture, SOLID,
  coupling, baseline security).

### SC-015: Issue linking with acceptance criteria

Given a PR with `Fixes #42` in the description and
  issue #42 has 4 acceptance criteria checkboxes
When the alignment check runs
Then the `### Linked Issues` section lists issue #42
  with each criterion and a coverage status (COVERED /
  NOT COVERED / PARTIAL), and criteria marked NOT COVERED
  appear as MEDIUM findings in `### Alignment`.

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
  `"event": "REQUEST_CHANGES"` in the JSON payload,
  alongside the review body and inline comments array.

### SC-018: Verdict override

Given a review with a computed REQUEST CHANGES verdict
When the user selects `change-verdict` at the
  confirmation prompt and chooses COMMENT
Then the review is posted as COMMENT instead of
  REQUEST_CHANGES.

### SC-019: Verdict confirmation warns about merge impact

Given a review with an APPROVE verdict
When the confirmation prompt is displayed
Then it includes a warning that posting APPROVE may
  unblock merge in repos with branch protection, and
  requires the user to type "approve" explicitly.

### SC-020: Issue linking — fetch failure

Given a PR with `Fixes #999` in the description and
  issue #999 does not exist
When the review runs
Then the `### Linked Issues` section lists issue #999
  with status "fetch failed" and the review continues
  without blocking. No acceptance criteria findings are
  produced for unfetchable issues.

### SC-021: Issue linking — no acceptance criteria

Given a PR with `Fixes #42` and issue #42 has only
  prose description (no checkboxes, no
  `## Acceptance Criteria` heading)
When the alignment check runs
Then the issue title and description are used as general
  intent context for the alignment check, and no
  uncovered criteria findings are reported.

### SC-022: Verdict posting — permission fallback

Given a user whose `gh` token lacks collaborator access
When the command attempts to post with `APPROVE` event
Then the `gh api` call returns 422, the command falls
  back to posting as `COMMENT` with a note explaining
  the permission limitation, and the original verdict is
  preserved in the comment body.

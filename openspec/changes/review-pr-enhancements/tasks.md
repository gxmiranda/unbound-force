## 1. Walkthrough Section

- [ ] 1.1 Add `### Walkthrough` to Step 9 output format
  template, positioned after `### Local Tool Results`
  and before `### Summary`. Format as a table with File
  and Change columns, grouped by directory. (FR-016)
- [ ] 1.2 Add instructions in Step 8 to generate a
  one-line change summary per file during diff analysis.
  The summary describes what changed (e.g., "Add error
  handling for null inputs"), not how (no code snippets).

## 2. GitHub Suggestion Blocks

- [ ] 2.1 Modify Step 11 comment preparation to detect
  findings with concrete single-file code fixes and
  format them using GitHub suggestion block syntax
  (triple-backtick `suggestion` with the replacement
  code). (FR-017)
- [ ] 2.2 Add guidance distinguishing when to use
  suggestion blocks (literal code replacements) vs plain
  text (architectural, design, or multi-file
  suggestions).

## 3. Path-based Review Focus

- [ ] 3.1 Add built-in path heuristics table to Step 8
  with the 7 path categories (test, API/handler, CLI,
  docs, CI/CD, dependencies, default). (FR-018)
- [ ] 3.2 Add instructions to classify each changed file
  against the heuristics table and append the matching
  focus instruction to the review context for that file.
  Path focus is additive — it supplements the standard
  review categories, not replaces them.

## 4. Issue Linking

- [ ] 4.1 Add Step 6a after Step 6: parse issue
  references from PR body using case-insensitive regex
  for `Fixes #N`, `Closes #N`, `Resolves #N` and their
  GitHub URL variants. (FR-019)
- [ ] 4.2 For each linked issue, fetch details via
  `gh issue view <N> --json title,body,labels`.
  (FR-019)
- [ ] 4.3 Extract acceptance criteria: checkbox lines
  (`- [ ]` / `- [x]`) or content under an
  `## Acceptance Criteria` heading. (FR-020)
- [ ] 4.4 Modify Step 8a alignment check to include
  issue acceptance criteria validation. Report uncovered
  criteria as MEDIUM findings. (FR-020)
- [ ] 4.5 Add `### Linked Issues` section to Step 9
  output format, positioned after `### Walkthrough`
  and before `### Summary`. Omit the section entirely
  when no issues are linked. (FR-021)

## 5. Verdict-aligned Review Posting

- [ ] 5.1 Modify Step 11 to map the verdict from Step 9
  to the corresponding `gh api` event type: APPROVE →
  `"event": "APPROVE"`, REQUEST CHANGES →
  `"event": "REQUEST_CHANGES"`, COMMENT →
  `"event": "COMMENT"`. (FR-022)
- [ ] 5.2 Replace the current two-step posting (separate
  summary comment + inline comments) with a single
  `gh api repos/{owner}/{repo}/pulls/<N>/reviews` call
  that includes event type, body, and comments array
  in one JSON payload. (FR-023)
- [ ] 5.3 Update the human confirmation prompt to show
  the verdict type: "Post review as REQUEST CHANGES
  with N comments? (yes/no/edit/change-verdict)". The
  `change-verdict` option lets the user override the
  computed verdict. (FR-024)
- [ ] 5.4 Add a warning note when the verdict is APPROVE
  or REQUEST_CHANGES, informing the user that posting
  may affect merge eligibility in repos with branch
  protection. (FR-025)

## 6. Scaffold Sync and Documentation

- [ ] 6.1 Sync scaffold asset copy — ensure
  `internal/scaffold/assets/opencode/command/review-pr.md`
  is byte-identical to `.opencode/command/review-pr.md`.
- [ ] 6.2 Update AGENTS.md PR Review section to document
  new capabilities (walkthrough, suggestion blocks,
  path focus, issue linking, verdict-aligned posting).
- [ ] 6.3 Add Recent Changes entry to AGENTS.md.
- [ ] 6.4 Run `make check` — verify build, test, vet,
  and lint all pass.

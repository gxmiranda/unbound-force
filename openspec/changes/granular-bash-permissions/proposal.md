## Why

During triage of issue #465, the `/uf.triage-issue` command
autonomously filed 4 child issues (#469-472) and posted
comments without asking the user for confirmation. The root
cause: prose-based confirmation gates (`AskUserQuestion` tool
references) were lost under context compression, and no
runtime permission enforcement exists for mutation commands
like `gh issue create` or `gh issue comment`.

The `council-review-action` already uses OpenCode's
`permission` config via `OPENCODE_CONFIG_CONTENT` to deny
`bash` entirely in CI. But interactive local commands (triage,
review-council, finale, unleash) run under the default
`bash: allow` with zero granular rules. OpenCode supports
pattern-based bash permissions (object syntax with wildcards),
but the project has never used them.

This change adds granular bash permissions to `opencode.json`
so that GitHub-mutating and git-mutating commands require
runtime approval (`"ask"`) while read-only commands remain
allowed. This is a structural fix that survives context
compression because the OpenCode runtime enforces it
regardless of what the model remembers.

Related issues:
- #465 (parent: review-council correctness gaps)
- #473 (triage Phase 4 lacks mandatory gate)
- #474 (inconsistent mandatory gate markers)
- #475 (`AskUserQuestion` vs `question` tool name)

## What Changes

Add a `"permission"` block to the project-level
`opencode.json` with granular bash rules that gate
GitHub mutations and git write operations behind `"ask"`.

## Capabilities

### New Capabilities
- `granular-bash-gating`: Runtime-enforced approval
  prompts for `gh issue create`, `gh issue comment`,
  `gh issue edit`, `gh issue close`, `gh pr create`,
  `gh pr merge`, `gh pr comment`, `gh pr edit`,
  `gh pr close`, `gh api`, `git push`, `git commit`,
  and `rm` commands. Read-only commands (`gh issue view`,
  `gh issue list`, `gh pr view`, `gh pr list`,
  `git status`, `git log`, `git diff`, `go test`, etc.)
  remain auto-allowed.

### Modified Capabilities
- `opencode.json`: Gains a `"permission"` top-level key
  alongside existing `"mcp"` configuration.

### Removed Capabilities
- None.

## Impact

- `opencode.json`: New `"permission"` block added.
- All interactive commands (`/uf.triage-issue`,
  `/uf.review-council`, `/uf.finale`, `/uf.unleash`,
  `/uf.review-pr`, `/uf.address-feedback`): Mutations
  now require user approval at the OpenCode runtime level,
  regardless of whether prose gates survive context
  compression.
- `council-review-action`: Unaffected. CI sets
  `OPENCODE_CONFIG_CONTENT` which takes precedence over
  project config, and its blanket `bash: deny` is stricter
  than granular rules.
- Agent frontmatter: Unaffected. Agents with `bash: deny`
  (all Divisor reviewers except Curator) remain fully
  denied. The granular rules apply only to agents that
  currently inherit the default `bash: allow`.
- `divisor-curator.md`: Agent currently has prose-only
  bash restrictions. Should be updated to use granular
  permissions in frontmatter, replacing the soft boundary
  with a hard one.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

This change strengthens artifact-based communication by
ensuring agents cannot silently create GitHub issues or
comments without human approval. The permission layer is
a runtime enforcement mechanism that operates independently
of agent instructions, maintaining the separation between
agent capabilities and human oversight.

### II. Composability First

**Assessment**: PASS

The granular permissions are set at the project level in
`opencode.json`. Heroes scaffolded by `uf init` can adopt
or customize these permissions independently. No mandatory
cross-repo dependencies are introduced.

### III. Observable Quality

**Assessment**: PASS

OpenCode's `"ask"` action produces observable approval
prompts with `once`/`always`/`reject` outcomes. The
permission configuration is declarative JSON, machine-
parseable, and version-controlled.

### IV. Testability

**Assessment**: PASS

The permission configuration can be tested by verifying
the JSON structure in `opencode.json`. Runtime behavior
can be validated by running commands and confirming
approval prompts appear for gated operations.

### V. Security by Default

**Assessment**: PASS

This change directly implements Principle V's "least
privilege" requirement by gating mutation commands at the
OpenCode runtime level. The `"ask"` action ensures agents
cannot perform file deletions, git pushes, or GitHub
mutations without human approval. No new dependencies are
introduced (supply chain integrity N/A). No external input
paths are added (input validation N/A). File permissions
are not modified.

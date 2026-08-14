## Context

The project uses OpenCode's permission system in two ways:

1. **Agent frontmatter** — 9 Divisor agents use simple
   `bash: deny` to block all bash. The Curator has bash
   allowed with prose-only restrictions.
2. **CI runtime** — `council-review-action/scripts/run-review.sh`
   sets `OPENCODE_CONFIG_CONTENT` with blanket tool denials.

Neither uses OpenCode's granular bash object syntax, which
supports wildcard command patterns with `allow`/`ask`/`deny`
actions. The project-level `opencode.json` has zero permission
configuration, so all non-agent bash commands inherit the
default `"allow"`.

This design adds granular bash permissions to `opencode.json`
to gate GitHub-mutating and git-write commands behind `"ask"`
prompts while allowing read-only operations.

## Goals / Non-Goals

### Goals
- Gate `gh issue create/edit/close/comment` behind `"ask"`
- Gate `gh pr create/merge/close/comment/edit` behind `"ask"`
- Gate `gh api` behind `"ask"` (arbitrary API calls)
- Gate `git push`, `git commit` behind `"ask"`
- Gate `rm` behind `"ask"`
- Allow read-only commands: `gh issue view/list`,
  `gh pr view/list`, `git status/log/diff/branch`,
  `go test/build/vet`, `make`, `grep`, standard dev tools
- Harden `divisor-curator.md` with granular frontmatter
  permissions replacing its prose-only restrictions

### Non-Goals
- Modifying `council-review-action` CI permissions (blanket
  deny via `OPENCODE_CONFIG_CONTENT` takes precedence and
  is stricter)
- Fixing the `AskUserQuestion` → `question` tool name
  mismatch (separate issue #475)
- Adding `>>> MANDATORY GATE <<<` markers to commands
  (separate issues #473, #474)
- Modifying scaffold assets for `uf init` (follow-up change)

## Decisions

### D1: Use `"ask"` not `"deny"` for mutations

Interactive commands legitimately need to create issues,
post comments, and push branches. Blanket `"deny"` would
break `/uf.triage-issue`, `/uf.finale`, and `/uf.unleash`.
The `"ask"` action prompts the user with `once`/`always`/
`reject`, giving humans control without blocking workflows.

This aligns with Constitution Principle I (Autonomous
Collaboration) — agents can propose mutations but humans
approve them.

### D2: Last-match-wins rule ordering

OpenCode evaluates bash permission patterns with **last
matching rule wins**. The config places the permissive
catch-all `"*": "allow"` first, then specific mutation
patterns after. This means:

```json
{
  "bash": {
    "*": "allow",
    "gh issue create *": "ask",
    "git push *": "ask"
  }
}
```

`git status` matches only `"*"` → allowed.
`gh issue create --title foo` matches both `"*"` and
`"gh issue create *"` → last wins → ask.

### D3: Pattern coverage strategy

Patterns must cover commands both with and without
arguments. OpenCode wildcards: `*` matches zero or more
characters, `?` matches exactly one.

- `gh issue create*` covers `gh issue create` (no args)
  and `gh issue create --title "foo"` (with args)
- `git push*` covers `git push` and `git push origin main`
- `git commit*` covers `git commit` and `git commit -m "msg"`

Use trailing `*` without space to match both bare commands
and commands with arguments. Exception: `rm` uses `"rm *"`
(with space) to avoid false-matching `rmdir` and other
`rm`-prefixed commands.

### D4: Curator agent hardening

Replace the Curator's prose-only bash restriction section
with granular frontmatter permissions:

```yaml
permission:
  edit: deny
  webfetch: deny
  bash:
    "*": "deny"
    "gh issue list*": "allow"
    "gh issue view*": "allow"
    "gh issue create*": "ask"
    "gh repo view*": "allow"
```

This converts the soft boundary into a hard one enforced
by OpenCode's runtime. Constitution Principle III
(Observable Quality) — the restriction is now declarative
and machine-parseable rather than hidden in prose.

### D5: Scope to `opencode.json` only

The granular rules go in the project-level `opencode.json`,
not in individual agent frontmatter. This provides a
single, auditable location for the project's mutation
policy. Agent-level overrides (like the Curator) are
exceptions that tighten the policy for specific agents.

## Risks / Trade-offs

**[R1] Approval fatigue**: Developers running `/uf.unleash`
or `/uf.cobalt-crush` may get frequent `"ask"` prompts for
legitimate `git commit` and `git push` operations. Mitigated
by OpenCode's `always` approval option, which whitelists a
command pattern for the session. Also mitigated by `--auto`
flag which approves all non-denied commands.

**[R2] Pattern gaps**: A new `gh` subcommand or alias could
bypass the rules. The catch-all `"*": "allow"` means
unlisted commands are allowed by default. Periodic audit
of the permission block is needed. Accept this trade-off
over a restrictive `"*": "ask"` default which would cause
excessive prompts for `ls`, `cat`, `go test`, etc.
Additionally, `"gh api*": "ask"` gates all `gh api`
invocations including read-only GET requests (e.g.,
fetching PR state in `/uf.review-pr`). This is accepted
because `gh api` can construct arbitrary mutations; the
DX cost of extra prompts on reads is preferable to
leaving an unrestricted mutation path.

**[R3] CI override precedence**: `OPENCODE_CONFIG_CONTENT`
takes precedence over project `opencode.json`. The CI
pipeline's blanket `bash: deny` is unaffected. If a future
CI workflow needs granular bash, it must set its own rules
via `OPENCODE_CONFIG_CONTENT`.

**[R4] Agent frontmatter merge behavior**: Agent-level
`permission.bash` merges with project-level config, with
agent rules taking precedence. Agents with `bash: deny`
(all Divisor reviewers except Curator) are unaffected —
their blanket deny overrides the project's granular rules.
The Curator's new granular rules will merge correctly.

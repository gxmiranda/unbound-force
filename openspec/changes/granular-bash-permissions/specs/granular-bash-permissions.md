## ADDED Requirements

### FR-001: Project-level granular bash permissions

The project `opencode.json` MUST include a `"permission"`
block with granular bash rules that gate GitHub-mutating
and git-write commands behind `"ask"` approval prompts.

The following command patterns MUST be gated with `"ask"`:

- `gh issue create*`
- `gh issue edit*`
- `gh issue close*`
- `gh issue comment*`
- `gh pr create*`
- `gh pr merge*`
- `gh pr close*`
- `gh pr comment*`
- `gh pr edit*`
- `gh pr review*`
- `gh api*`
- `git push*`
- `git commit*`
- `rm *`

The catch-all `"*"` pattern MUST be set to `"allow"` to
avoid prompts on read-only and development commands.

#### Scenario: Agent attempts to create a GitHub issue

- **GIVEN** `opencode.json` contains granular bash rules
- **WHEN** an agent executes `gh issue create --title "foo"`
- **THEN** OpenCode prompts the user with
  `once`/`always`/`reject` before executing the command

#### Scenario: Agent runs a read-only git command

- **GIVEN** `opencode.json` contains granular bash rules
- **WHEN** an agent executes `git status --short`
- **THEN** the command executes without prompting

#### Scenario: Agent runs go test

- **GIVEN** `opencode.json` contains granular bash rules
- **WHEN** an agent executes `go test -race -count=1 ./...`
- **THEN** the command executes without prompting

### FR-002: Curator agent granular bash hardening

The `divisor-curator.md` agent MUST replace its prose-only
bash restriction section with granular bash permissions in
the YAML frontmatter `permission:` block.

The Curator's bash permissions MUST be:

- `"*": "deny"` (default deny)
- `"gh issue list*": "allow"`
- `"gh issue view*": "allow"`
- `"gh issue create*": "ask"`
- `"gh repo view*": "allow"`

The prose "Bash Access Restriction" section SHOULD be
removed or reduced to a reference to the frontmatter
permissions.

#### Scenario: Curator attempts to list issues

- **GIVEN** the Curator agent has granular bash permissions
- **WHEN** the Curator executes `gh issue list --repo foo`
- **THEN** the command executes without prompting

#### Scenario: Curator attempts to create an issue

- **GIVEN** the Curator agent has granular bash permissions
- **WHEN** the Curator executes `gh issue create --title "x"`
- **THEN** OpenCode prompts the user with
  `once`/`always`/`reject` before executing

#### Scenario: Curator attempts an unlisted command

- **GIVEN** the Curator agent has granular bash permissions
- **WHEN** the Curator executes `curl https://example.com`
- **THEN** the command is denied

### FR-003: CI permission precedence

The `council-review-action` CI pipeline MUST NOT be
affected by the project-level granular permissions.
`OPENCODE_CONFIG_CONTENT` takes precedence over
`opencode.json`, and the CI pipeline's blanket
`bash: deny` MUST remain the effective rule.

#### Scenario: CI review pipeline runs with project config

- **GIVEN** `opencode.json` has granular bash `"ask"` rules
- **AND** `OPENCODE_CONFIG_CONTENT` sets `bash: deny`
- **WHEN** the CI review pipeline executes
- **THEN** all bash commands are denied (CI config wins)

### FR-004: Scaffold asset parity

Both live agent files and their scaffold counterparts
MUST have identical permission blocks:

- `.opencode/agents/divisor-curator.md`
- `internal/scaffold/assets/opencode/agents/divisor-curator.md`

Drift detection tests MUST pass after the change.

#### Scenario: Scaffold drift detection

- **GIVEN** the Curator agent has updated permissions
- **WHEN** drift detection tests execute
- **THEN** live and scaffold copies match

## MODIFIED Requirements

None.

## REMOVED Requirements

None.

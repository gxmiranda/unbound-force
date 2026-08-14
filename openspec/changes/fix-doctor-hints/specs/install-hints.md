# Spec: Tool-Agnostic Install Hints

## MODIFIED Requirements

### Requirement: FR-001 Agent Context Install Hints

All `InstallHint` values produced by the agent context check
group in `internal/doctor/checks.go` MUST describe the
remediation action in tool-agnostic plain language. Hints
MUST NOT reference specific editor integrations, slash
commands, or third-party tool invocations.

Previously: Five `InstallHint` strings hardcoded
`"Run: /uf.agent-brief in OpenCode"`, coupling output to a
specific tool.

#### Scenario: AGENTS.md missing
- **GIVEN** a project without an `AGENTS.md` file
- **WHEN** `uf doctor` runs the agent context checks
- **THEN** the `InstallHint` MUST instruct the user to create
  an `AGENTS.md` file without referencing a specific tool

#### Scenario: Tier 1 section missing
- **GIVEN** a project with `AGENTS.md` that lacks one or more
  Tier 1 sections
- **WHEN** `uf doctor` runs the agent context checks
- **THEN** the `InstallHint` MUST name the missing section and
  instruct the user to add it, without referencing a specific
  tool

#### Scenario: AGENTS.md exceeds line threshold
- **GIVEN** a project with `AGENTS.md` that exceeds the
  configured line count threshold
- **WHEN** `uf doctor` runs the agent context checks
- **THEN** the `InstallHint` MUST instruct the user to
  condense the file, without referencing a specific tool

#### Scenario: Constitution reference missing
- **GIVEN** a project with `AGENTS.md` that lacks a
  constitution reference
- **WHEN** `uf doctor` runs the agent context checks
- **THEN** the `InstallHint` MUST instruct the user to add a
  constitution reference, without referencing a specific tool

#### Scenario: Spec framework not described
- **GIVEN** a project with `AGENTS.md` that does not describe
  the specification framework
- **WHEN** `uf doctor` runs the agent context checks
- **THEN** the `InstallHint` MUST instruct the user to add a
  specification workflow section, without referencing a
  specific tool

## ADDED Requirements

### Requirement: FR-002 No Bare Tool References in Hints

The existing regression test
`TestDoctorHints_NoBareUnboundReferences` MUST be extended
to also scan for `/uf.` slash command patterns in
`InstallHint` values. The current test only checks for bare
`"unbound "` branding references (FR-006) and does not
guard against tool-specific slash command reintroduction.

#### Scenario: Regression guard
- **GIVEN** the full doctor check suite
- **WHEN** all check groups are executed via `Run()`
- **THEN** no `InstallHint` value SHALL contain `/uf.` slash
  command references or `OpenCode` tool-specific references

### Requirement: FR-003 Drift Detection Test (Optional)

A drift-detection test SHOULD verify that the project's own
`AGENTS.md` contains all Tier 1 sections defined in
`agentContextTier1Sections`.

#### Scenario: Project AGENTS.md has all Tier 1 sections
- **GIVEN** the project's `AGENTS.md` at repository root
- **WHEN** the drift-detection test runs
- **THEN** all five Tier 1 section headers as defined by
  `agentContextTier1Sections` (Project Overview, Build
  Commands, Project Structure, Code Conventions, Technology
  Stack) MUST be matched by the section patterns in the file

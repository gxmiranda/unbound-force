## ADDED Requirements

### FR-001: Embedded Starter Constitution Asset

The scaffold engine MUST include a starter constitution file
at `internal/scaffold/assets/specify/memory/constitution.md`.
The file MUST contain the five Unbound Force org-level
principles (Autonomous Collaboration, Composability First,
Observable Quality, Testability, Security by Default) adapted
with generic project language. The file MUST NOT contain
`[PLACEHOLDER]` tokens. The constitution file MUST be created
with `0o644` permissions. Parent directories MUST be created
with `0o755` permissions.

#### Scenario: Fresh scaffold creates constitution
- **GIVEN** a target directory with no `.specify/` directory
- **WHEN** `uf init` runs
- **THEN** `.specify/memory/constitution.md` is created
- **AND** the file contains all five principle headings
- **AND** the file is reported as "created" in the Result

#### Scenario: Constitution file includes version marker
- **GIVEN** a target directory with no `.specify/` directory
- **WHEN** `uf init` runs with version "1.0.0"
- **THEN** `.specify/memory/constitution.md` contains a
  `<!-- scaffolded by uf v1.0.0 -->` version marker

### FR-002: `specify/` Asset Path Mapping

The `mapAssetPath()` function MUST map the `specify/` prefix
to `.specify/` in the target directory, following the same
`"." + relPath` pattern used by `opencode/` and
`devcontainer/`. The `knownAssetPrefixes` slice MUST include
`"specify/"`.

#### Scenario: Asset path mapping
- **GIVEN** an embedded asset at `specify/memory/constitution.md`
- **WHEN** `mapAssetPath("specify/memory/constitution.md")` is
  called
- **THEN** it returns `.specify/memory/constitution.md`

### FR-003: Constitution Is User-Owned

The constitution file MUST NOT be classified as tool-owned by
`isToolOwned()`. On re-runs without `--force`, existing
constitution files MUST be preserved and reported as "skipped".

#### Scenario: Re-run preserves user-customized constitution
- **GIVEN** a target directory with an existing customized
  `.specify/memory/constitution.md`
- **WHEN** `uf init` runs without `--force`
- **THEN** the existing file is not overwritten
- **AND** the file is reported as "skipped" in the Result

#### Scenario: Force mode overwrites constitution
- **GIVEN** a target directory with an existing customized
  `.specify/memory/constitution.md`
- **WHEN** `uf init` runs with `--force`
- **THEN** the file is overwritten with the embedded starter
- **AND** the file is reported as "overwritten" in the Result

### FR-005: Sentinel Interaction — `specify init` Skipped

When the embedded constitution asset creates the `.specify/`
directory during the asset walk, the `specify` simple tool
sentinel check (`os.Stat(".specify")`) MUST succeed, causing
`initSimpleTool()` to skip `specify init`. Users requiring
full Speckit framework scaffolding (templates, config, scripts)
MUST run `specify init` manually after `uf init`.

#### Scenario: specify init is skipped when constitution is scaffolded
- **GIVEN** `specify` is installed and available on PATH
- **WHEN** `uf init` runs on a fresh directory
- **THEN** `.specify/memory/constitution.md` is created from
  embedded assets
- **AND** `specify init` is NOT executed (sentinel `.specify/`
  already exists from the asset walk)

### FR-006: Constitution Is Not Tool-Owned

The `isToolOwned()` function MUST return `false` for
`specify/` paths (including `specify/memory/constitution.md`).
This ensures the constitution file is treated as user-owned
content that is preserved on re-runs.

#### Scenario: isToolOwned returns false for specify paths
- **GIVEN** a path `specify/memory/constitution.md`
- **WHEN** `isToolOwned("specify/memory/constitution.md")` is
  called
- **THEN** it returns `false`

### FR-004: DivisorOnly Mode Excludes Constitution

The constitution asset MUST NOT be classified as a Divisor
asset by `isDivisorAsset()`. In `--divisor-only` mode, the
constitution file MUST NOT be scaffolded.

#### Scenario: DivisorOnly skips constitution
- **GIVEN** `--divisor-only` mode is active
- **WHEN** `uf init` runs
- **THEN** `.specify/memory/constitution.md` is NOT created

## MODIFIED Requirements

### MR-001: Post-Scaffold Hint Text

The post-scaffold hints MUST change the `/speckit.constitution`
guidance from "create your project constitution" to "customize
your project constitution". This applies to both hint variants
(with and without Dewey).

Previously: "Run /speckit.constitution to create your project
constitution"

Now: "Run /speckit.constitution to customize your project
constitution"

#### Scenario: Hint text after scaffold
- **GIVEN** `uf init` has completed successfully
- **WHEN** the next-steps hints are printed
- **THEN** the `/speckit.constitution` hint reads "customize
  your project constitution"
- **AND** the hint does NOT contain "create your project
  constitution"

## REMOVED Requirements

None.

## Coverage Strategy

Unit tests: `TestMapAssetPath` (FR-002), `TestIsToolOwned`
(FR-006) — test classification functions in isolation.

Integration tests: `TestRun_*` variants (FR-001, FR-003,
FR-004, FR-005) — test `Run()` end-to-end with `t.TempDir()`
and stub I/O functions. All tests use the scaffold engine's
existing test infrastructure.

Drift detection: `TestAssetPaths`/`TestAssetManifest` — verify
embedded asset manifest includes the new constitution file.

Error paths: Covered by the existing scaffold engine error
handling tests (`TestRun_CreatesFiles` would fail if the
embedded asset were missing). No new error path tests needed.

## Known Limitations

Existing `[PLACEHOLDER]` constitutions from previous `specify
init` runs are preserved on upgrade (user-owned file logic).
Users must manually replace them. This is a pre-existing
`uf doctor` concern, not introduced by this change.
<!-- scaffolded by uf vdev -->

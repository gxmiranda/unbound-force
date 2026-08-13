<!-- spec-review: passed -->
<!-- code-review: passed -->
<!--
  [P] marks tasks eligible for parallel execution.
  Add [P] when a task: (a) touches different files from
  other [P] tasks in the group, (b) has no dependency
  on prior tasks in the group, (c) can safely execute
  without ordering constraints.
  Do NOT add [P] when tasks modify the same file —
  parallel workers will cause merge conflicts.
  Tasks without [P] run sequentially first, then [P]
  tasks run in parallel.
-->

## 1. Embedded Starter Constitution Asset

- [x] 1.1 Create `internal/scaffold/assets/specify/memory/constitution.md`
  with the five org principles adapted for generic project
  use. No `[PLACEHOLDER]` tokens. Include a header comment
  directing users to `/speckit.constitution` for customization.
  Files: `internal/scaffold/assets/specify/memory/constitution.md`
  Refs: FR-001 (D4 in design)

## 2. Scaffold Engine Updates

- [x] 2.1 Add `"specify/"` to `knownAssetPrefixes` and add a
  `specify/` case to `mapAssetPath()` that returns
  `"." + relPath` (maps to `.specify/`).
  Files: `internal/scaffold/scaffold.go`
  Refs: FR-002 (D1 in design)

- [x] 2.2 Update post-scaffold hint text on lines 1736 and 1739
  from "create your project constitution" to "customize your
  project constitution".
  Files: `internal/scaffold/scaffold.go`
  Refs: MR-001 (D6 in design)

## 3. Tests

All test tasks modify `internal/scaffold/scaffold_test.go`.
Run sequentially — same file, no parallel markers.

- [x] 3.1 Update expected asset manifest in
  `TestAssetPaths`/`TestAssetManifest` (or equivalent) to
  include `specify/memory/constitution.md`. This MUST be
  done before other test tasks to avoid manifest mismatch.
  Refs: FR-001

- [x] 3.2 Add `TestMapAssetPath` cases for `specify/` prefix
  mapping (input: `specify/memory/constitution.md`, expected:
  `.specify/memory/constitution.md`).
  Refs: FR-002

- [x] 3.3 Add `TestIsToolOwned` entry for
  `specify/memory/constitution.md` with expected: `false`.
  Refs: FR-006

- [x] 3.4 Add test: fresh scaffold creates constitution file,
  file contains all five principle headings, file is reported
  as "created" in Result, file does NOT contain `[PLACEHOLDER]`
  tokens, and file contains a version marker.
  Refs: FR-001 (Scenarios 1 and 2)

- [x] 3.5 Add test: re-run without `--force` preserves
  existing constitution (reported as "skipped"), does not
  overwrite user content.
  Refs: FR-003

- [x] 3.6 Add test: `--force` re-run overwrites existing
  constitution (reported as "overwritten").
  Refs: FR-003

- [x] 3.7 Add test: `--divisor-only` mode does NOT scaffold
  the constitution file. Verify `.specify/memory/constitution.md`
  is NOT in `result.Created` and the file does not exist on
  disk in the target directory.
  Refs: FR-004

- [x] 3.8 Add test: post-scaffold hints contain "customize
  your project constitution" AND do NOT contain "create your
  project constitution". Assert both positive and negative
  for both hint variants (with and without Dewey).
  Refs: MR-001

## 4. Verification

- [x] 4.1 Run `make check` (lint + test + build) and confirm
  all tests pass with `-race -count=1`.
  Refs: CI parity rule

- [x] 4.2 Verify constitution alignment: embedded starter
  constitution contains principles that match the org
  constitution format; scaffold produces a self-contained
  artifact (Principle I); no external tool dependency
  required (Principle II); file is machine-parseable
  Markdown with version marker (Principle III); all new
  code is tested in isolation (Principle IV); embedded
  asset contains no secrets or credentials (Principle V).
  Refs: Proposal constitution alignment

- [x] 4.3 Add CHANGELOG.md entry under bug fixes describing
  the embedded starter constitution and `specify init` skip
  behavior change. Example: `fix: uf init now scaffolds a
  working starter constitution instead of delegating to
  specify init (#213)`.
  Files: `CHANGELOG.md`
  Refs: Documentation gate (AGENTS.md)

- [x] 4.4 Verify documentation issue #478 exists in
  `unbound-force/unbound-force` tracking docs updates for
  `uf init` behavior change.
  Refs: Cross-repo documentation rule (AGENTS.md)
<!-- scaffolded by uf vdev -->

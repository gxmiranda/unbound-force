---
tag: pre-flight-skill
author: yvonne-devlin
category: gotcha
created_at: 2026-06-23T12:30:57Z
identity: pre-flight-skill-20260623T123057-yvonne-devlin
tier: draft
---

When extracting shared logic from multiple command files into a skill, the scaffold sync requirement extends beyond the consuming commands to the new skill file itself. If a skill is intended to be distributed via `uf init` (like speckit-workflow), it needs a scaffold asset copy at `internal/scaffold/assets/opencode/skills/<name>/SKILL.md` AND an entry in the `expectedAssetPaths` list in `scaffold_test.go`. All 5 Divisor reviewers independently flagged this gap during spec review — the spec originally only included scaffold sync for the three modified command files but missed the new skill file. The drift detection test `TestEmbeddedAssets_MatchSource` catches this at build time, but specifying it upfront in the tasks prevents rework.

---
tag: pre-flight-skill
author: yvonne-devlin
category: pattern
created_at: 2026-06-23T12:31:05Z
identity: pre-flight-skill-20260623T123105-yvonne-devlin
tier: draft
---

When consolidating duplicated logic from multiple commands into a shared skill, be explicit about whether the unified approach is "strict behavioral parity" or "intentional behavioral expansion." In the pre-flight skill extraction, combining workflow-driven detection (from review-council) with config-file detection (from review-pr) means each consumer gains the other's detection capabilities — review-council may discover tools like yamllint it previously missed, and review-pr may discover CI-only commands like coverage ratchets. The spec review caught that the original spec simultaneously claimed "no behavioral regression" while the design acknowledged this expansion. The fix was to explicitly state that expanded detection is an intentional improvement, not strict parity, and to update acceptance scenarios accordingly. This prevents implementers and future reviewers from treating newly-discovered tools as regressions.

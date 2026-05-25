# Handoff — 2026-05-25

**Head commit (project):** (scaffold — not yet pushed)
**Head commit (workspace):** (initial)

## What Changed This Session

Repo bootstrapped: casehubio/life created. Maven structure (api, app), CLAUDE.md, LAYER-LOG.md
(7 tutorial layers stubbed), .githooks, CI workflow, spec docs (life-automation.md,
life-actor-model.md), deep-dive in parent, all build infrastructure updated. BOM entries added
to casehub-parent pom.xml. PLATFORM.md and APPLICATIONS.md updated.

## Immediate Next Step

Create Epic 1 + Epic 2 issues in casehubio/life, then begin Layer 1 (Naive Java domain model).
Run `work-start` referencing Epic 2 (Layer 1) issue.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 1 | Layer 1: household domain model + REST API | M | Low | Hexagonal: api/ + app/ |
| 2 | Layer 2: + casehub-work SLA enforcement | S | Med | Flyway V100+, work WorkItem |
| 3 | Layer 3: + casehub-qhorus commitment lifecycle | M | Med | Oversight gates, contractor follow-up |

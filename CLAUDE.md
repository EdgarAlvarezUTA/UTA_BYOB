# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a planning and architecture project for migrating ~3TB of media assets from Cloudinary-managed S3 to UTA-owned AWS S3, while retaining Cloudinary as the metadata management and CDN layer (BYOB — Bring Your Own Bucket model). The project is currently in planning/pre-implementation phase.

**Key constraint:** Migration must NOT touch the live Cloudinary environment. All migration work uses a new, isolated Cloudinary environment as the target.

## BMAD Framework

This project uses the [BMAD Method](https://github.com/bmad-method/bmad-method) v6.0.2 for AI-assisted planning and development. Before activating any agent or workflow, load `_bmad/bmm/config.yaml` to set session variables:

- `{user_name}` = Edgar
- `{output_folder}` = `_bmad-output/`
- `{planning_artifacts}` = `_bmad-output/planning-artifacts/`
- `{implementation_artifacts}` = `_bmad-output/implementation-artifacts/`
- `{project_knowledge}` = `docs/`

**Workflow execution rules:**
- MD-based workflows: load and follow the `.md` file directly
- YAML-based workflows: load `_bmad/core/tasks/workflow.xml` first, then pass the `.yaml` config
- Execute workflows step-by-step (JIT loading); save outputs after each step
- Use `/bmad-` prefix to trigger workflows in Copilot Chat

**Available agents** (activate via agent dropdown or `/bmad-` commands):

| Agent | Persona | Primary Use |
|-------|---------|-------------|
| analyst (Mary) | Business Analyst | Research, requirements, domain expertise |
| architect (Winston) | Architect | Cloud infrastructure, distributed systems |
| dev (Amelia) | Developer | Story execution, code implementation |
| pm (John) | Product Manager | PRDs, stakeholder alignment |
| qa (Quinn) | QA Engineer | Test planning, coverage analysis |
| sm (Bob) | Scrum Master | Sprint planning, backlog management |
| tech-writer (Paige) | Technical Writer | Documentation, Mermaid diagrams |

## Architecture

### Target AWS Stack
- **Storage:** Amazon S3 (UTA-owned, migration destination)
- **Queue:** AWS SQS (main queue + DLQ for failed assets)
- **State:** DynamoDB (per-asset tracking: status, attempts, error, cloudinary_id, s3_etag)
- **Compute:** AWS Batch or ECS Fargate (auto-scaling worker fleet, 50–500 concurrent workers)
- **Monitoring:** CloudWatch + Datadog
- **IaC:** Terraform (`infrastructure/terraform/cloudinary_to_s3/`)

### Migration Flow (Option B — Recommended)
```
Controller → SQS Queue → Worker Fleet → UTA S3
                              ↕
                         DynamoDB (state)
                              ↕
                   New Cloudinary Environment (BYOB)
```

Workers are stateless: download from old Cloudinary env → verify → upload to UTA S3 → update metadata in new Cloudinary env.

### URL Continuity Strategy
CNAME switch + same `public_id` preservation → zero database remapping required across 7 UTA web applications.

## Key Project Artifacts

| File | Purpose |
|------|---------|
| `_bmad-output/planning-artifacts/migration-path-options-analysis.md` | Three migration strategies compared (A/B/C) |
| `_bmad-output/implementation-artifacts/prd-enterprise-external-storage.md` | Full PRD |
| `_bmad-output/planning-artifacts/assumption-change-log.md` | Rate limit discoveries and assumption revisions |
| `docs/Cloudinary Q&A 1.md` | Initial vendor technical clarifications |
| `docs/Cloudinary Q&A 2.md` | Rate limits, BYOB constraints, Admin API limits |

## Validated Decisions (as of Mar 3, 2026)

- Admin API rate limit: **5,000 req/hr** (Enterprise Plus & DAM Plan)
- CNAME cutover is viable for URL continuity
- `public_id` preservation confirmed
- GDPR not applicable; all assets are public
- Old Cloudinary environment kept as rollback safeguard post-migration

## Open Items (Pending Cloudinary PS Workshop)

1. Does `cld clone` copy binary assets + metadata into BYOB-backed environment?
2. Does `cld sync` support cloud-to-cloud (environment-to-environment)?
3. Can Admin API limit be temporarily raised to 20,000+ req/hr for migration burst?
4. Cross-account S3 copy permission feasibility (Option C evaluation)

## Dependencies

```bash
npm install   # Installs bmad-method v6.0.2 and dev tooling
```

No build or test commands exist yet — implementation phase has not begun.

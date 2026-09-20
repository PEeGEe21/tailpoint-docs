# Tailpoint Documentation

This repository is the source of truth for Tailpoint product requirements, delivery plans, engineering decisions, and operational policies that span multiple code repositories.

## Start here

- [Product feature roadmap](./product/PRODUCT_FEATURE_ROADMAP.md) — product phases, feature sequencing, and strategic direction.
- [Phase 3 implementation plan](./phases/PHASE_3_IMPLEMENTATION_PLAN.md) — completed core Phase 3 contracts, delivery record, and remaining operational verification.
- [Phase 6A GitHub implementation plan](./phases/PHASE_6A_GITHUB_IMPLEMENTATION_PLAN.md) — one-way GitHub webhook ingestion, task linking, security, and delivery tickets.
- [GitHub integration usage guide](./operations/GITHUB_INTEGRATION_GUIDE.md) — repository setup, task references, health, secret rotation, troubleshooting, and pilot validation.
- [Project ownership](./product/PROJECT_OWNERSHIP.md) — permanent creator, co-owner permissions, role lifecycle, and notification rules.
- [Mobile implementation plan](./phases/MOBILE_APP_IMPLEMENTATION_PLAN.md) — ordered delivery phases, engineering checklists, dependencies, gates, and release timeline.
- [Mobile app PRD](./product/mobile/MOBILE_APP_PRD.md) — mobile product scope, experience, screens, feature order, and release gates.
- [Mobile infrastructure](./product/mobile/MOBILE_APP_INFRASTRUCTURE.md) — mobile architecture, security, API contracts, delivery stack, and engineering backlog.
- [Task tracker](./engineering/TASK_TRACKER.md) — cross-repository implementation tracking.

## Repository structure

```text
tailpoint-docs/
├── README.md
├── product/       # Product strategy and feature requirements
│   └── mobile/    # Mobile product and architecture specifications
├── phases/        # Phase plans and completion audits
├── engineering/   # Cross-repository engineering plans and audits
└── operations/    # Delivery runbooks and operational policies
```

## Product

- [Tailpoint product feature roadmap](./product/PRODUCT_FEATURE_ROADMAP.md)
- [Pinned Projects PRD](./product/PINNED_PROJECTS_PRD.md)
- [Mobile app PRD](./product/mobile/MOBILE_APP_PRD.md)
- [Mobile infrastructure and architecture](./product/mobile/MOBILE_APP_INFRASTRUCTURE.md)

## Delivery phases

- [Phase 1 completion audit](./phases/PHASE_1_COMPLETION_AUDIT.md)
- [Phase 2 implementation plan](./phases/PHASE_2_IMPLEMENTATION_PLAN.md)
- [Phase 3 implementation plan](./phases/PHASE_3_IMPLEMENTATION_PLAN.md)
- [Mobile app implementation plan and checklist](./phases/MOBILE_APP_IMPLEMENTATION_PLAN.md)

## Engineering

- [Implementation audit](./engineering/IMPLEMENTATION_AUDIT.md)
- [Mobile API contract audit](./engineering/MOBILE_API_CONTRACT_AUDIT.md)
- [Task tracker](./engineering/TASK_TRACKER.md)
- [Universal intake](./engineering/UNIVERSAL_INTAKE_README.md)
- [Chat redesign PRD](./engineering/CHAT_REDESIGN.md)

## Operations

- [CI/CD runbook](./operations/CI_CD_RUNBOOK.md)
- [Data lifecycle policy](./operations/DATA_LIFECYCLE_POLICY.md)
- [GitHub integration usage guide](./operations/GITHUB_INTEGRATION_GUIDE.md)

## Documentation conventions

- Update the relevant document in the same pull request as a material product or architecture change.
- Put product behavior and user-facing requirements in `product/`.
- Put delivery scope, status, and acceptance matrices in `phases/`.
- Put cross-repository technical plans and audits in `engineering/`.
- Put deployment, support, security-operation, and lifecycle procedures in `operations/`.
- Use relative Markdown links so navigation works locally and on GitHub.
- Include a status, version, or last-updated field on documents that represent a changing contract.
- Preserve historical decisions. When a decision changes materially, record the replacement and rationale instead of silently rewriting history.

## Related code repositories

- [`PEeGEe21/track-a-project`](https://github.com/PEeGEe21/track-a-project) — workspace web application.
- [`PEeGEe21/track-a-project-backend`](https://github.com/PEeGEe21/track-a-project-backend) — API and background services.
- [`PEeGEe21/track-a-project-admin`](https://github.com/PEeGEe21/track-a-project-admin) — platform administration application.
- [`PEeGEe21/projecttrakr-sdk`](https://github.com/PEeGEe21/projecttrakr-sdk) — ingestion and monitoring SDK.

- [`PEeGEe21/tailpoint-mobile`](https://github.com/PEeGEe21/tailpoint-mobile) — Expo/React Native mobile application.

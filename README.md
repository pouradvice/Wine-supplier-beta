# Vine — Distribution Data Normalization Platform

> We make sure wineries always know exactly what they have, where it is, and how it's selling — no matter how many distributors they work with.

Vine is an interpretation infrastructure layer for the U.S. wine industry. It ingests distributor sales and inventory reports in any format, normalizes them against a growing registry of known distributor schemas, resolves product and account identity across inconsistent naming conventions, and produces a single confidence-aware system of record that improves with every file processed and every human decision made.

This is not a reporting tool. It is the layer that makes reporting possible.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Core Concepts](#core-concepts)
- [Getting Started](#getting-started)
- [Development Environment](#development-environment)
- [Track Ownership](#track-ownership)
- [Engineering Non-Negotiables](#engineering-non-negotiables)
- [Sprint Roadmap](#sprint-roadmap)
- [Contributing](#contributing)
- [Environment Variables](#environment-variables)

---

## Architecture Overview

The system is organized around three core components that work in sequence:

```
Distributor File
       │
       ▼
┌─────────────────────────────────────────────────────┐
│  INGESTION PIPELINE                                 │
│  File reception → Format fingerprinting →           │
│  Column classification → Row extraction →           │
│  Confidence scoring → State assignment →            │
│  Immutable event log write                          │
└────────────────────┬────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       ACCEPTED   PROVISIONAL  UNRESOLVED
      (auto-used) (human gate) (retry queue)
          │          │          │
          └──────────┴──────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  SYSTEM OF RECORD                                   │
│  Confidence-aware unified view across all           │
│  distributors. Every number traceable to source.   │
└─────────────────────────────────────────────────────┘
```

The **Distributor Format Registry** sits alongside the pipeline and is the primary competitive moat. Every distributor format template acquired makes the system more accurate for every winery. Every winery onboarded makes the registry more complete.

The **Human-in-the-Loop Learning System** means that every user action in the Provisional review interface is a labeled training example. Users generate training data as a byproduct of doing their jobs. This collapses ML readiness from year two to late year one.

---

## Repository Structure

```
vine/
├── apps/
│   ├── api/                  # Core backend — ingestion, state management, event log
│   └── web/                  # Frontend — dashboard, Provisional review, exception UI
├── packages/
│   ├── pipeline/             # Ingestion pipeline stages 1–7
│   ├── registry/             # Distributor format registry — templates, versioning, matching
│   ├── identity/             # Entity resolution — SKU, account, location
│   ├── exceptions/           # Exception pipeline — grouping, reason codes, impact scoring
│   ├── ml/                   # ML layer (introduced S11 after rules stability gate)
│   └── shared/               # Types, constants, canonical schema definitions
├── infra/
│   ├── terraform/            # Infrastructure as code
│   ├── docker/               # Container definitions
│   └── scripts/              # Deployment and migration scripts
├── docs/
│   ├── engineering-plan.md   # Full engineering specification
│   ├── ui-directive.md       # UI/UX behavioral specifications
│   ├── sprint-roadmap.md     # 12-sprint roadmap with track ownership
│   └── adr/                  # Architecture Decision Records
└── tests/
    ├── fixtures/             # Sample distributor files for pipeline testing
    ├── integration/          # Cross-package integration tests
    └── e2e/                  # End-to-end behavioral tests
```

---

## Core Concepts

### The Three-State Model

Every unit of processed data — row, SKU match, account mapping, inventory calculation — exists in exactly one of three states at all times. This is the architectural spine of the system.

| State | Meaning | Behavior |
|---|---|---|
| `ACCEPTED` | High confidence. Auto-applied. | Immediately queryable. No human review required. Threshold rises over time. |
| `PROVISIONAL` | Interpreted. Awaiting confirmation. | System has an interpretation but cannot confirm without human. Requires explicit accept/reject. **Never auto-accepts under any condition.** |
| `UNRESOLVED` | No viable interpretation. | Raw form preserved immutably. Retried on learning events, with monthly sweep as fallback. |

At launch, all data that fails the Accepted confidence threshold routes to Provisional. There is no direct path to Unresolved on first pass. This is intentional — Provisional generates labeled training data at the moment the system needs it most.

### The Distributor Format Registry

The registry is the primary competitive moat. It is a relationship asset encoded in a technical form — every distributor format acquired represents a permanent barrier to entry.

- Templates are **immutable once published**. New versions only; no overwrites.
- Multiple active versions are supported to handle distributor format transitions.
- Automatic fingerprint matching on every ingestion event.
- Schema drift detection runs on every file, compares against matched template version, and surfaces differences without blocking the file.

### Confidence Language

Raw confidence scores are never surfaced to end users. They map to four plain-language tiers:

| Internal | User-Facing | State |
|---|---|---|
| Above threshold | **Confirmed** | Accepted |
| High-confidence Provisional | **Likely** | Provisional |
| Low-confidence Provisional | **Uncertain** | Provisional |
| No viable interpretation | **Needs review** | Unresolved |

### Exception Reason Codes

Every exception carries a reason code from this fixed taxonomy. Unclassified exceptions are a system failure.

| Code | Meaning |
|---|---|
| `NO_TEMPLATE_MATCH` | File structure does not match any registered distributor format |
| `LOW_COLUMN_CONFIDENCE` | Column classification below threshold; mapping uncertain |
| `IDENTITY_AMBIGUOUS` | Multiple candidate matches for SKU or account; none dominant |
| `SCHEMA_DRIFT` | File matches a known template but structural differences detected |
| `MISSING_REQUIRED_FIELD` | Canonical schema requires a field absent from source file |
| `CROSS_SOURCE_CONFLICT` | Same entity with conflicting values across two distributor reports |
| `PARSE_FAILURE` | Field present but value cannot be parsed against expected type |

---

## Getting Started

### Prerequisites

- Node.js 20+
- Python 3.11+
- Docker and Docker Compose
- PostgreSQL 15+
- AWS CLI (for S3-backed raw storage in staging/production)

### Local Setup

```bash
# Clone the repository
git clone https://github.com/your-org/vine.git
cd vine

# Install dependencies
npm install

# Copy environment template
cp .env.example .env.local
# Edit .env.local with your local credentials (see Environment Variables below)

# Start dependent services (Postgres, Redis, local S3 via LocalStack)
docker compose up -d

# Run database migrations
npm run db:migrate

# Seed the registry with test distributor templates
npm run registry:seed

# Start all services in development mode
npm run dev
```

The web application will be available at `http://localhost:3000`.
The API will be available at `http://localhost:4000`.

### Running Tests

```bash
# All tests
npm run test

# Pipeline unit tests only
npm run test --filter=pipeline

# Integration tests (requires running Docker services)
npm run test:integration

# End-to-end behavioral tests
npm run test:e2e

# Test with a real distributor fixture file
npm run pipeline:test-file -- --fixture=fixtures/southern-glazers-sample.csv
```

---

## Development Environment

### GitHub Codespace

This repository is configured for GitHub Codespaces. Opening the repo in a Codespace will automatically:

- Install all Node and Python dependencies
- Start Postgres, Redis, and LocalStack via Docker Compose
- Run database migrations
- Seed the registry with test templates
- Start the API and web application in development mode

The Codespace is configured in `.devcontainer/devcontainer.json`.

### Tech Stack

| Layer | Technology |
|---|---|
| API | Node.js, TypeScript, Fastify |
| Pipeline processing | Python, Pandas, custom classification layer |
| Frontend | Next.js, TypeScript, Tailwind |
| Database | PostgreSQL (primary), Redis (job queue) |
| Raw storage | S3 (immutable event log and raw ingested files) |
| Job queue | BullMQ |
| Observability | Datadog (production), Grafana + Prometheus (local) |
| Infrastructure | Terraform, AWS |
| CI/CD | GitHub Actions |

### Branch Strategy

- `main` — production-ready. Protected. Requires PR review and passing CI.
- `staging` — integration branch. Deployed to staging environment on push.
- `feature/*` — feature branches. Branch from `main`, PR back to `main`.
- `hotfix/*` — emergency fixes. Branch from `main`, merge to `main` and `staging`.

All PRs require: passing tests, no lint errors, and at least one approving review.

---

## Track Ownership

| Track | Package | Sprint Start | Current Owner |
|---|---|---|---|
| Infrastructure | `infra/` | S1 | — |
| Distributor Registry | `packages/registry` | S1 | — |
| Ingestion Pipeline | `packages/pipeline`, `packages/exceptions` | S3 | — |
| Identity Resolution | `packages/identity` | S5 | — |
| UI / Product | `apps/web` | S3 | — |
| ML / Learning Layer | `packages/ml` | S11 (gated) | — |

The ML layer package exists in the repository from day one as a scaffold, but no production code is merged to it until the rules stability gate is passed. See [Engineering Non-Negotiables](#engineering-non-negotiables).

---

## Engineering Non-Negotiables

These constraints have no exceptions. Any proposal that requires violating one requires a full architecture review — not an engineering decision made in a PR.

**No dropped rows.** Every row that enters the system must be accounted for in a terminal state or active queue. Unaccounted rows are a data integrity failure. The pipeline must produce output from every uploaded file, regardless of content quality.

**No silent failures.** Every failure is logged, classified, and surfaced. A failure that cannot be observed cannot be fixed. Every exception carries a reason code from the defined taxonomy. Unclassified exceptions are a system failure.

**No auto-acceptance of Provisional.** Provisional items require human confirmation. No confidence threshold, volume threshold, or time condition overrides this. The system-of-record claim depends on this rule having no exceptions — ever.

**No cross-winery data sharing.** Business outcome data — sales figures, inventory levels, depletion rates, account relationships — is winery-private. Distributor format templates (structural information only) are shared. This boundary is architectural, not a permission flag or a policy setting.

**No opaque transformations.** Every output must be traceable to its source. Any number displayed in the product must be followable back to: source file → distributor → template match used → columns included → confidence score → Provisional items excluded.

**ML after rules stability.** The ML layer is introduced only after the rules-based layer has demonstrated stability across the seven required metrics. ML introduced to an unstable rules layer learns the wrong patterns permanently. The stability gate is a condition, not a calendar date.

**Immutable event log.** The event log is append-only. It is never modified, only extended. Every ingested row — in raw form, interpreted form, confidence score, state assignment, and timestamp — has a permanent record. This log is the audit trail and the replay source.

**Templates are append-only.** Distributor format templates are immutable once published. Updates create new versions. No overwrites. The database schema enforces this constraint — it is not a convention.

---

## Required Observability Metrics

The following metrics must be instrumented and visible at all times. They are the signals that determine when the ML gate opens, when thresholds should be adjusted, and when a distributor relationship needs attention.

| Metric | Purpose |
|---|---|
| `% data in each confidence tier` | System health. Drives threshold adjustment decisions. |
| `provisional_resolution_rate` | User engagement with the learning loop. Surfaces UX friction. |
| `exception_volume_by_distributor` | Distributor format quality. Drives registry maintenance priority. |
| `template_match_success_rate` | Registry completeness. Drives format acquisition priority. |
| `time_to_first_usable_output` | Onboarding experience. Target: under 5 minutes. |
| `accepted_rate_trend` | Learning system effectiveness. Primary ML introduction signal. |
| `rejection_correction_pairs_generated` | Training data accumulation. ML readiness assessment. |

---

## Sprint Roadmap

12 sprints across 4 phases. Full interactive roadmap available in `docs/sprint-roadmap.html`.

| Phase | Sprints | Focus |
|---|---|---|
| **Phase 0 — Foundation** | S1–S2 | Dev environment, CI/CD, event log schema, registry schema. Infra ready gate. |
| **Phase 1 — Core Pipeline** | S3–S6 | Tier 1 registry acquisition (100 formats in 90 days), ingestion pipeline stages 1–6, SKU matching v1, first output screen, Provisional review interface. Anchor pilot gate at S6. |
| **Phase 2 — Product Surface** | S7–S10 | Tier 2 registry acquisition, exception pipeline, account resolution, unified dashboard, exception and identity UI, canonical entity graph. Rules stability gate at S10. |
| **Phase 3 — ML Readiness** | S11–S12 | Schema drift system, location resolution, ML layer introduction (gated), vindication moment and provenance UI, trust architecture. Series A readiness gate at S12. |

The anchor client program launches at S3. Relationship capital converts to engineering asset. 100 distributor formats by end of S6 is the first major milestone — everything downstream is cheaper for every format in the registry.

---

## Definition of Success

New customer onboarding is a matching problem, not a mapping problem.

Exception volume per distributor decreases month-over-month without manual scaling.

The system improves measurably with every dataset processed.

The user stops checking the distributor report. They check this instead.

That last point is the only definition of success that matters. Everything else is a leading indicator.

---

## Contributing

This codebase is internal. Access is restricted to the engineering team.

For architecture decisions, open an ADR in `docs/adr/` before implementing. The template is at `docs/adr/000-template.md`. Decisions that touch any of the non-negotiables listed above require review before the ADR is accepted.

For bugs and feature work, open an issue and reference the relevant sprint and track. PRs without a linked issue will not be reviewed.

For questions about the distributor registry — specifically, new format acquisition or template versioning decisions — contact the registry track owner directly. Do not make template changes without review. Templates are permanent.

---

*Vine is confidential. Do not share this repository or its contents outside the organization.*

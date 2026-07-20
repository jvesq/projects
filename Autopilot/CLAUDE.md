# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository has a monorepo skeleton (uv workspace for Python, pnpm workspace for TypeScript) but **no implemented business logic yet** — every package under `services/`, `adapters/`, and `apps/` is currently a stub (`pyproject.toml`/`package.json` + an empty entry file). Treat this as a scaffolded-but-unimplemented repo: the directory structure and package boundaries are real and intended to be built into, but don't assume any function, class, or endpoint exists until you've checked.

## Repository layout

- `docs/spec/Autopilot_Platform_Specification_INTEGRATED_FINAL.md` — the full platform spec (Draft V2, ~4,700 lines). This is the source of truth for scope, architecture, data model, API contracts, and rollout plan. See its table of contents (top of file) to jump to a section; section numbers in prose below refer to it.
- `docs/diagrams/` — 12 architecture diagrams (`diagram1_platform_architecture.png`, `diagram2_shared_intelligence_context.png`, `diagram3_unified_workflow.png`, `platform_architecture_polished.png`, `shared_intelligence_polished.png`, `workflow_polished.png`, and the `Platform Architecture - *.png` set covering Integrations, Intelligence, Mapping/DQA, ML, and Overview views). **Note:** these filenames were renamed to drop "Enable"/"Edge" branding, but the pixel content of the images (labels drawn inside the diagrams) still shows the old branding and needs manual re-export before external use.
- `COMPONENTS.md` — the component breakdown (core framework vs. independently-buildable pieces vs. external dependencies) that this package layout was derived from. Check it before adding a new package or deciding where new code belongs.
- `services/` — the core framework (Python, uv workspace members): `core`, `console-api`, `agent-runtime`, `pipeline-orchestrator`, `stage-workers`, `ml-client-wrapper`, `audit-writer`, `adapter-pool`, `observability`.
- `adapters/` — the six source-system adapters (Python, uv workspace members, each depends on `autopilot-core` for the `SourceAdapter` protocol): `filesystem`, `sap`, `s3`, `sql`, `sharepoint`, `snowflake`.
- `apps/` — the three frontend surfaces (TypeScript/React, pnpm workspace members): `console`, `agent-chat`, `credential-entry`.
- Root `pyproject.toml` is a virtual uv workspace root (no `[project]` table); root `package.json` + `pnpm-workspace.yaml` are the pnpm workspace root.

The spec also contains inline Mermaid diagrams (flowcharts) for platform context, internal components, and pipeline data flow — these are more current than the standalone PNGs, which predate some of the spec's revisions, and have already been updated to use vendor-neutral labels.

## What Autopilot is

**Autopilot Agent Framework** is a **governed onboarding orchestration and operational intelligence platform**, designed to embed into an implementing organization's existing host platform, that automates enterprise customer data onboarding. Instead of customers manually finding, extracting, and sending target business data, Autopilot's agent and pipeline discover, classify, retrieve, map, validate, and land that data from customer source systems (SAP, filesystems, S3, SQL, SharePoint, Snowflake).

The spec's worked example is a pricing/incentive-management deployment (illustrated with a price-waterfall reference frame and a SAP table catalog), but the framework itself is vendor-neutral and general-purpose: any enterprise organization can use it to solve its own data-onboarding problem by supplying its own canonical model, target-schema table catalog, and downstream domain.

Positioning (spec §1, ADR-002/003 in §22): Autopilot is **governance-first**, not an "unconstrained AI onboarding assistant." Runtime LLM providers generate reasoning and conversation; Autopilot's own orchestration layer owns workflow execution, lifecycle state, approvals, replay, and audit. Replayability, auditability, and tenant isolation are treated as foundational architecture, not optional tooling (ADR-004).

## Architecture (spec §7)

- **Client tier**: the host platform's UI shell (React) embeds an agent chat surface (Claude Agent SDK) and an operational console (React + TS + REST).
- **Backend microservices**: `agent-runtime` (hosts the Claude Agent SDK session, translates tool calls into REST calls against `console-api` — there is deliberately no separate agent API/auth path), `console-api` (single FastAPI service; source of truth for REST + MCP tool surface, auth, RBAC, audit), `pipeline-orchestrator` (job lifecycle, gates stages on customer confirmation), `stage-workers` (one Kafka consumer per pipeline stage), `adapter-pool` (hosts the six source-system adapters), `ml-client-wrapper` (sidecar handling auth/retry/circuit-breaking for all ML calls — workers never call ML services directly), `audit-writer` (append-only signed audit records).
- **Stores/bus**: Kafka (Azure Event Hubs in production), PostgreSQL (operational state, job/audit records), Snowflake (canonical warehouse), Key Vault (credentials, owned by the Integrations domain — Autopilot never stores raw secrets).
- **Eight-stage event-driven pipeline** (§7.4): `discover → classify → retrieve → map → dqa → enrich → anomaly → serialize`. Stages communicate only via Kafka and durable stores — no HTTP between orchestrator and workers. `map`/`dqa`/`enrich`/`anomaly` each call one of four production host-platform Intelligence-domain ML services (Semantic Data Mapper, Data Quality Assurance, Data Enrichment, Anomaly Detection); Autopilot is a client of these, not their owner. `serialize` emits one event per master domain to `*-ingested` topics owned by the host platform's Master Data domain — that's where Autopilot's responsibility ends.
- **Source adapters** (spec §11): all six adapter types (Filesystem, SAP, S3, SQL, SharePoint, Snowflake) implement a common `SourceAdapter` protocol (async, strictly read-only against source systems) so pipeline workers and the agent's MCP tool surface treat them uniformly. Filesystem and SAP get deep-dive coverage; SAP is pilot-critical.
- **Constraints carried through the spec**: Azure deployment, Auth0 IAM for identity/RBAC (federated through the host platform's IAM tenant), 85% confidence threshold below which items route to human review, v1 scope limited to a configurable set of "essential" data elements (10 in the baseline catalog).

## Key architectural decisions (spec §22, ADR-001–007)

Read these before proposing structural changes — they represent resolved decisions, not open questions:
1. Provider-agnostic runtime (Claude, OpenAI, future providers behind one abstraction).
2. Autopilot owns orchestration/execution; LLM providers only generate reasoning.
3. Governance-first: replayability, auditability, tenant isolation prioritized over autonomy.
4. Replay is foundational platform architecture, not bolt-on tooling.
5. Enterprise operational UX over consumer chatbot UX.
6. Schema evolution is expected operational reality, not an edge case.
7. Architecture preserves compatibility with a future customer-hosted Agent-as-a-Service model.

## Working with the spec

- The spec is explicit about what's a locked decision vs. an open assumption. Sections carry inline `🟡 ASM-*` (assumption to validate at build time) and `REV-*` (item to revisit later) markers, plus per-section "Open assumptions to validate before build" and "Items to revisit / re-evaluate" tables — check these before treating a section's details as final, especially in §8 (data model) and §11 (source adapters), which carry explicit build-time validation warnings pointing at the host platform's teams as the source of truth for anything that has drifted.
- Requirement IDs (e.g. `AGNT-*`, `PIPE-*`, `ADPT-*`, `DISC-*`, `ML-*`, `DATA-*`, `AUTH-*`, `API-*`, `OBS-*`) are defined in §6 (Functional requirements) and referenced throughout the rest of the doc — search for the ID to find its defining requirement.
- §26 has a glossary and reference list if terminology is unclear.

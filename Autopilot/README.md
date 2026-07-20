# Autopilot Agent Framework

A governed onboarding orchestration and operational intelligence framework for enterprise data onboarding — vendor-neutral and general-purpose, configurable to any organization's canonical data model and source systems.

- **Spec**: [`docs/spec/Autopilot_Platform_Specification_INTEGRATED_FINAL.md`](docs/spec/Autopilot_Platform_Specification_INTEGRATED_FINAL.md) — the full platform specification.
- **Component breakdown**: [`COMPONENTS.md`](COMPONENTS.md) — what's core framework, what's independently buildable, what's an external dependency, and a suggested build order.
- **Diagrams**: [`docs/diagrams/`](docs/diagrams/)

## Status

Scaffolded, unimplemented. The monorepo layout below exists as stub packages; no business logic has been written yet.

## Layout

```
services/    Python (uv workspace) — core framework: core, console-api, agent-runtime,
             pipeline-orchestrator, stage-workers, ml-client-wrapper, audit-writer,
             adapter-pool, observability
adapters/    Python (uv workspace) — source-system adapters: filesystem, sap, s3, sql,
             sharepoint, snowflake
apps/        TypeScript/React (pnpm workspace) — console, agent-chat, credential-entry
docs/        Spec and architecture diagrams
```

## Setup

```sh
uv sync        # Python workspace (services/, adapters/)
pnpm install   # TypeScript workspace (apps/)
```

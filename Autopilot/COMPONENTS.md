# Autopilot Agent Framework — Component Breakdown

This is a build-planning inventory derived from `docs/spec/Autopilot_Platform_Specification_INTEGRATED_FINAL.md`. It splits everything the spec describes into three buckets:

1. **Core framework** — the engine you build once; not meaningfully separable from "Autopilot" itself.
2. **Framework-owned, independently buildable** — still part of Autopilot's codebase/scope, but each is a self-contained unit with a defined protocol boundary, buildable and shippable on its own timeline.
3. **External dependencies** — systems Autopilot integrates against but does not build. Some are infrastructure to provision, others are the host platform's existing services to get API access to.

Section references (`§n`) point back to the spec.

---

## 1. Core framework

The seven backend microservices plus the pipeline they run, per §7.3/§7.4. These are the load-bearing pieces — none of them is meaningfully shippable without the others, since they share the job-state/event contracts.

| Component | What it does | Package | Spec ref |
|---|---|---|---|
| **agent-runtime** | Hosts the Claude Agent SDK session per customer; translates agent tool calls into REST calls against `console-api` (no separate agent auth path); streams responses to the chat UI | `services/agent-runtime` | §7.3, §10 |
| **Agent Runtime Layer (provider abstraction)** | Provider-agnostic layer inside agent-runtime: prompt orchestration, tool-call normalization, provider routing, retry handling, eval hooks, trace propagation. Ships with a Claude adapter; OpenAI adapter and future-provider slots are part of the core abstraction (ADR-001) | `services/agent-runtime` | §10.1–10.3 |
| **console-api** | Single FastAPI service; source of truth for REST + MCP tool surface, auth, RBAC, validation, audit fan-out, idempotency, rate limiting | `services/console-api` | §7.3, §13 |
| **pipeline-orchestrator** | Owns job lifecycle (create/start/monitor/complete/fail); produces stage-input events to the bus; tracks per-job state in Postgres; gates `retrieve` on customer confirmation | `services/pipeline-orchestrator` | §7.3, §9 |
| **stage-workers (pipeline engine)** | The consumer framework running the eight stages — `discover → classify → retrieve → map → dqa → enrich → anomaly → serialize`. Each stage is an independent Kafka consumer with its own retry/DLQ | `services/stage-workers` | §7.3, §7.4, §9 |
| **ml-client-wrapper** | Internal library (sidecar/co-process) handling auth, retries, timeouts, circuit-breaking, structured error mapping for every ML service call. Pipeline stages never call ML services directly (ML-22) | `services/ml-client-wrapper` | §7.3, ML-22 |
| **audit-writer** | Async consumer writing append-only, signed audit records from every other service (DATA-11, AUTH-10) | `services/audit-writer` | §7.3, §14 |
| **File catalog / job-state store** | Postgres schema + data-access layer: discovered/retrieved/classified object inventory, per-job state, schema fingerprints — source of truth for incremental runs (DATA-07, PIPE-06) | `services/core` | §8.6, DATA-07 |
| **Replay & recovery engine** | DLQ replay, idempotency via `parent_event_id`, scheduled refresh — treated as foundational, not bolt-on (ADR-004) | `services/core` (shared logic), used by `stage-workers` / `pipeline-orchestrator` | §9.0.1, §9.6 |
| **Tenant isolation layer** | `tenant_id` as a partition key enforced across every store/topic/query path; agent context isolation guarantees (AUTH-09, DATA-09) | `services/core` | §14.5, DATA-09 |
| **Canonical-model configuration layer** | The mechanism that lets a deployment plug in its *own* canonical dimensions, target-schema table catalog, and downstream reference frame instead of the illustrative pricing/incentive example. This is what makes the framework vertical-agnostic — arguably the single most important core piece for the "general framework" repositioning | `services/core` | §8.4, DATA-01, DATA-02 |
| **Review queue** | Human-in-the-loop work queue for low-confidence classifications/mappings and flagged anomalies | `services/console-api` (API surface), `services/core` (domain model) | §9.7 |
| **Global-learnings / alias-registry contribution (framework side)** | The logic that decides what confirmed mappings get contributed back, anonymization gating before contribution | `services/ml-client-wrapper` | §11.8, AUTH-11 |

## 2. Framework-owned, independently buildable

Same repo/product, but each has a protocol boundary (`SourceAdapter`, MCP tool surface, etc.) that lets it be built, tested, and shipped on its own — good candidates for separate packages in the monorepo and separate build sequencing.

| Component | What it does | Package | Spec ref |
|---|---|---|---|
| **Filesystem adapter** | `SourceAdapter` implementation for local/network filesystems. Tier 1, deep-dive coverage — lowest-friction onboarding entry point, good first adapter to build | `adapters/filesystem` | §11.5 |
| **SAP adapter** | `SourceAdapter` implementation for SAP (pyrfc-based). Tier 1, deep-dive — most operationally complex, pilot-critical | `adapters/sap` | §11.6 |
| **S3 adapter** | Tier 1, standard coverage — cloud-storage equivalent of Filesystem | `adapters/s3` | §11.7 |
| **SQL adapter** | Tier 2 — generic RDBMS via SQLAlchemy | `adapters/sql` | §11.8 |
| **SharePoint adapter** | Tier 2 — Microsoft Graph SDK | `adapters/sharepoint` | §11.9 |
| **Snowflake adapter** | Tier 2 | `adapters/snowflake` | §11.10 |
| **adapter-pool (host/runtime)** | The shared runtime that discovers and hosts adapter implementations at startup and exposes them uniformly to workers, the credential service, and the agent's MCP tool generator (ADPT-01) — buildable ahead of any individual adapter, against the `SourceAdapter` protocol as a contract | `services/adapter-pool` | §7.3, §11.2 |
| **MCP tool-surface generator** | Expands the agent's tool surface automatically as adapters are added, per ADPT-01 — a defined generation step, separable from any one adapter | `services/agent-runtime` (generation), reads from `services/adapter-pool` | §11.1, §11.4 |
| **Operational console (web UI)** | React + TS console for internal users (onboarding lead, IA, admin): navigation, job monitoring, review-queue UI, replay/escalation controls | `apps/console` | §13.6–13.9 |
| **Agent chat surface (UI embed)** | The Claude Agent SDK chat-UI primitives embedded in the host platform's UI shell — a distinct frontend unit from the console | `apps/agent-chat` | §13.2–13.5 |
| **Tokenized credential-entry UI** | The primitive that lets a customer enter source-system credentials directly into the host platform's credential store without values passing through the agent runtime | `apps/credential-entry` | §13.5, §14.4 |
| **Observability instrumentation package** | Shared OpenTelemetry instrumentation (traces, the metrics catalog, structured logging conventions) wired into every service — buildable as one internal library and dropped into each service | `services/observability` | §15 |

## 3. External dependencies

Autopilot is a **client** of these — integrate against their APIs/contracts, do not build or own them. Per the spec's positioning (§7.0.1, §7.2), reusing these rather than re-implementing them is a deliberate architectural choice.

### Host platform services (the org's existing platform ecosystem)
| Dependency | Role | Spec ref |
|---|---|---|
| **Host platform IAM (Auth0 tenant)** | Identity, SSO, RBAC — Autopilot federates into it rather than running a parallel identity system | §14.2, §15.0 |
| **Key Vault** | Credential storage for customer source-system secrets; Autopilot never sees raw values (ADPT-12, AUTH-04) | §14.4 |
| **Semantic Data Mapper** | ML service — schema matching across integration axes (ML-01–07) | §7.2, ML-01–07 |
| **Data Quality Assurance (DQA)** | ML service — readiness scoring, hard-block gating (ML-08–11) | ML-08–16 |
| **Data Enrichment** | ML service — product classification, location enrichment (ML-12–13) | ML-12–14 |
| **Anomaly Detection** | ML service — Bayesian GMM anomaly scoring, Dagster-batched (ML-17–21) | ML-17–21 |
| **Data Dictionary** | Authoritative canonical type definitions and ERP-native pattern library that `discover`/`classify` consume | §11.4, §12.2 |
| **Master Data domain** | Owns the authoritative canonical stores; consumes Autopilot's `*-ingested` events — Autopilot's responsibility ends at emitting the event | §7.2, DATA-04 |
| **Host platform schema registry** | Owns `*-ingested` topic schemas and canonical-type versioning | §8.7 |
| **Host platform UI shell / design system** | What the agent chat and console visually embed into; not built by Autopilot | §13.3 |

### Infrastructure to provision (not code to write, but not optional either)
| Dependency | Role |
|---|---|
| **Event bus (Kafka / Azure Event Hubs)** | Inter-stage pipeline transport |
| **PostgreSQL** | Operational state (jobs, file catalog, audit) |
| **Snowflake** | Canonical-warehouse intermediate state |
| **Dagster** | Nightly anomaly-detection batch orchestration (owned/run by the Anomaly Detection service's team, not Autopilot) |

### Third-party platforms
| Dependency | Role |
|---|---|
| **Claude API / Anthropic** | Primary LLM runtime provider |
| **Claude Agent SDK** | The SDK the agent-runtime and chat UI are built on |
| **OpenAI API** | Secondary/future provider (ADR-001 provider-agnostic runtime) |
| **OpenTelemetry backend, Datadog, Grafana, PagerDuty** | Observability and alerting stack |
| **Braintrust** | LLM eval platform for prompt/runtime evaluation hooks |
| **MLflow** | Model-version tracking, owned by the host platform's Intelligence domain |

### Customer-side (per deployment, not part of the product at all)
- Customer source systems themselves (SAP, filesystems, S3, SQL databases, SharePoint, Snowflake instances) — what adapters connect *to*, never built or owned by Autopilot.

---

## Suggested build order

Roughly follows the spec's own priority tiers (§5, §11.1) and dependency direction — each phase depends only on what's above it:

1. **Foundations**: canonical-model configuration layer, file catalog/job-state store, tenant isolation layer, `console-api` skeleton (auth/RBAC/audit wiring).
2. **Pipeline skeleton**: pipeline-orchestrator + stage-workers framework, wired to a stub adapter and stub ML client so the eight-stage event flow runs end-to-end before any real integration exists.
3. **First adapter**: Filesystem (lowest integration cost, validates the `SourceAdapter` protocol and adapter-pool).
4. **ML integration**: ml-client-wrapper against the real four host-platform ML services (these are external dependencies you need API access to before this phase can complete).
5. **Agent surface**: agent-runtime + provider abstraction + chat UI embed, MCP tool-surface generator.
6. **Second adapter (SAP)** and **console UI** can proceed in parallel once the pipeline skeleton and agent surface are stable.
7. **Remaining adapters** (S3, SQL, SharePoint, Snowflake), replay/DLQ hardening, observability package, global-learnings contribution — GA-hardening phase per §25.

# AI Testcase Generator - Implementation Phases

This plan breaks the architecture in `ai-testcase-generator-architecture.md` into an incremental delivery sequence. Each phase produces a usable, testable capability and preserves the contracts needed by later phases.

## Delivery Strategy

Build the first production-capable path around one source type, one embedding provider, one LLM provider, human review, and export. Add connectors, providers, publishing targets, and execution only after the evidence-backed generation path is reliable.

### Initial production slice

```text
React Web App
  -> API Gateway / BFF
  -> Project and Access Service
  -> File Upload Connector
  -> Ingestion, Extraction, Normalization
  -> Mistral Embeddings
  -> MongoDB Hybrid Search
  -> Retrieval Service
  -> One LLM Provider
  -> Canonical Testcase Generation
  -> Review Workflow
  -> JSON / Excel Export
```

### Cross-cutting rules for every phase

- Use tenant and project context on every persisted resource and API request.
- Version source content, prompts, templates, models, generated testcases, and review decisions.
- Treat long-running ingestion, generation, export, and publishing as jobs with durable state.
- Propagate `eventId`, `correlationId`, `tenantId`, and `projectId` through APIs, workers, and events.
- Record enough evidence and metadata to reconstruct why a testcase was generated.
- Add unit, integration, contract, and security tests with each capability rather than postponing them.
- Keep provider-specific logic behind adapters and connector-specific logic behind connector contracts.

## Phase 0 - Product and Technical Foundations

### Goal

Turn the architecture into an executable product contract and a working development platform.

### Implement

- Confirm MVP personas, project workflow, approval policy, supported source format, and export format.
- Create the monorepo structure from the architecture document.
- Configure pnpm, TypeScript, build orchestration, formatting, linting, and test runners.
- Establish local Docker infrastructure for MongoDB, queueing, and event streaming as needed.
- Define OpenAPI conventions, error responses, pagination, job states, and idempotency behavior.
- Define shared packages for common types, API contracts, event contracts, logging, configuration, and MongoDB access.
- Add CI checks for build, lint, unit tests, dependency scanning, and container builds.
- Create development, staging, and production configuration conventions without committing secrets.

### Deliverables

- Buildable repository with one health-check service and one web shell.
- Initial OpenAPI document and event envelope schema.
- Local development and CI instructions.
- Architecture decision records for unresolved choices such as queue and event-stream products.

### Exit criteria

- A clean checkout can install dependencies, start local infrastructure, run all checks, and start the web app and API.
- API and event contract validation fails the build when incompatible changes are introduced.

## Phase 1 - Identity, Tenancy, Projects, and API Boundary

### Goal

Provide the secure control plane that all later capabilities use.

### Implement

- API Gateway / BFF with request validation, correlation IDs, rate limiting, and consistent errors.
- Identity mapping and authentication integration; begin with a development identity provider if necessary.
- Tenant, user, project, membership, and project-level RBAC models.
- Roles: Admin, Architect, QA Lead, Tester, Developer, and Viewer.
- Authorization middleware and resource-level checks for every project-scoped route.
- Project creation and settings APIs and the corresponding React screens.
- Audit events for sign-in, project changes, membership changes, and permission failures.
- Encrypted configuration storage for integration metadata; keep credential values in a secrets manager.

### Deliverables

- Project dashboard with member management.
- Working project-scoped API boundary.
- MongoDB indexes for tenant, project, membership, and timestamps.
- Authorization and tenant-isolation test fixtures.

### Exit criteria

- A user can create a project, assign a role, and access only resources permitted by that role.
- Tests prove that cross-tenant and unauthorized cross-project reads and writes are rejected.

## Phase 2 - Job, Messaging, and Operational Backbone

### Goal

Establish asynchronous processing and operational behavior before adding expensive AI work.

### Implement

- Queue abstraction and worker runtime for long-running jobs.
- Event-stream abstraction and versioned event envelope.
- Durable job state machine: queued, running, succeeded, failed, cancelled, and retrying.
- Idempotency keys, retry with exponential backoff, dead-letter handling, timeouts, and cancellation.
- Outbox or equivalent reliable event publication pattern.
- Structured logs, metrics, traces, health checks, readiness checks, and correlation propagation.
- Initial dashboards and alerts for job failures, queue depth, latency, and dependency health.

### Deliverables

- Shared queue, event-stream, observability, and job packages.
- A sample worker that can be retried and inspected through an API.
- Runbook for failed jobs, dead-letter messages, and dependency outages.

### Exit criteria

- A test job survives a worker restart without duplicate side effects.
- Failed jobs can be retried or cancelled and remain visible to the user.

## Phase 3 - Source Connectors and File Ingestion

### Goal

Accept source content and make ingestion observable.

### Implement

- Connector SDK with normalized source metadata, connection testing, fetch, webhook, and sync contracts.
- File-upload connector as the first implementation.
- Upload validation, size/type limits, malware scanning integration point, and object-storage abstraction.
- Source, source-document, source-version, and ingestion-job persistence.
- Ingestion APIs, job progress, retry, cancellation, and ingestion history screens.
- `source.connected`, `source.changed`, `ingestion.requested`, and `ingestion.started` events.
- Deduplication using source identifiers, content hashes, and version metadata.

### Deliverables

- User can upload supported text, PDF, spreadsheet, JSON, YAML, and OpenAPI files.
- User can see queued, running, completed, failed, and retried ingestion jobs.
- Connector contract tests and file-ingestion integration tests.

### Exit criteria

- Re-uploading the same source version is idempotent.
- Every ingested item has tenant, project, source, version, timestamps, and job lineage.

## Phase 4 - Extraction, Normalization, Chunking, and Knowledge Indexing

### Goal

Convert heterogeneous input into searchable, version-aware knowledge.

### Implement

- Extraction handlers for plain text, PDF, spreadsheet/tabular content, JSON/YAML, and OpenAPI.
- Canonical content model with document, section, and chunk hierarchy.
- Content classification and metadata normalization.
- Hierarchical chunking with stable identifiers and parent relationships.
- Mistral embedding adapter with batching, retries, model/version tracking, and re-embedding support.
- MongoDB persistence for documents, versions, chunks, embeddings, and indexing status.
- MongoDB text/BM25-capable indexes, vector indexes, and compound tenant/project filters.
- `content.extracted`, `content.normalized`, `embedding.requested`, and `content.indexed` events.
- Document, version, chunk, reindex, and re-embed APIs.

### Deliverables

- Searchable knowledge base populated from uploaded content.
- Admin-visible extraction/indexing errors and partial processing status.
- Data migration and re-indexing scripts.

### Exit criteria

- A source can be processed from upload through indexed document, section, and chunk records.
- Search results cannot cross project or tenant boundaries and include source/version metadata.

## Phase 5 - Hybrid Retrieval and Evidence Service

### Goal

Return relevant, explainable context for a user query.

### Implement

- Query normalization, abbreviation expansion, and configurable synonym expansion.
- Parallel BM25 and vector retrieval with candidate merging.
- Configurable rule and score-based reranking using vector similarity, BM25, source authority, status, freshness, and project metadata.
- Deduplication and context summarization with source references preserved.
- Retrieval result persistence for audit and evaluation.
- Search API and knowledge-search UI with filters, snippets, relevance scores, and source/version links.
- `retrieval.completed` event and retrieval latency/error metrics.

### Deliverables

- Retrieval service with a provider-neutral search interface.
- Evidence response containing chunks, scores, ranking explanation, and lineage.
- Golden query set for relevance evaluation.

### Exit criteria

- A reviewer can search a requirement and verify each returned evidence item in its source version.
- Retrieval quality meets an agreed baseline on the golden query set, with tenant isolation and deterministic reranking tests passing.

## Phase 6 - Testcase Generation and AI Guardrails

### Goal

Generate canonical, evidence-backed testcases from retrieved context.

### Implement

- Canonical testcase schema and schema-validation package.
- Testcase type framework and recommendation flow with user acceptance, removal, and addition of types.
- Prompt assembly from system instructions, user request, evidence, project template, and output schema.
- Versioned prompt and testcase templates.
- Provider-neutral `LLMProvider` interface and first provider adapter.
- Structured generation, timeout/retry behavior, response parsing, and validation.
- Duplicate testcase checks, requirement coverage checks, unsupported-assumption detection, and insufficient-evidence handling.
- Hybrid confidence engine with component scores and explanations.
- Generation jobs, generation traces, model/provider metadata, and `testcase.generation.requested`, `testcase.generated`, and `testcase.validation.failed` events.
- Generator UI for request entry, type selection, progress, generated results, evidence, and confidence.

### Deliverables

- Canonical testcases with requirement references, source references, confidence, and pending-review status.
- Generation trace showing prompt version, retrieved evidence, model, validation, and confidence components.
- Evaluation set covering supported testcase types and insufficient-evidence cases.

### Exit criteria

- A request produces schema-valid testcases or an explicit insufficient-evidence result.
- No testcase can be marked ready for review without evidence references and validation results.
- Provider failures are visible and do not leave jobs indefinitely stuck.

## Phase 7 - Human Review, Testcase Lifecycle, and Auditability

### Goal

Give QA users control over generated content and preserve a complete decision history.

### Implement

- Testcase and testcase-version APIs.
- Review workflow: generated, pending review, approved, rejected, modified, and repending review.
- Project policies for every testcase, low-confidence-only, or optional review.
- Approve, reject, modify, comment, and reassignment actions.
- Immutable review actions and audit events.
- Testcase list, detail, side-by-side evidence, confidence explanation, lineage, and review queue screens.
- Optimistic-concurrency/version checks to prevent overwriting another reviewer’s edits.
- `testcase.review.requested`, `testcase.approved`, `testcase.rejected`, and `testcase.modified` events.

### Deliverables

- End-to-end reviewable testcase lifecycle.
- Audit trail from requirement through source version, retrieval, prompt, model, testcase version, and review decision.
- Permission tests for reviewer and non-reviewer roles.

### Exit criteria

- Publishing/export is blocked when the configured approval gate is unmet.
- A user can explain what evidence and model configuration produced an approved testcase.

## Phase 8 - Export and First-Class QA Publishing

### Goal

Deliver approved testcases into existing QA workflows.

### Implement

- Export job framework for JSON, CSV, Excel, and PDF.
- Export templates and field-mapping configuration.
- Connector publishing contract with external ID mapping, retries, idempotency, and rate-limit handling.
- First publishing connector, selected according to MVP customer priority, such as TestRail or Jira/Xray.
- Publish preview, validation errors, partial-failure recovery, and external-link tracking.
- `testcase.published` event and publish audit records.

### Deliverables

- Downloadable exports and one production-capable publishing integration.
- Traceability from internal testcase to external system ID and URL.
- Runbooks for credential rotation, API limits, and failed publishing jobs.

### Exit criteria

- Only approved testcases can be published under the default policy.
- Repeating a publish request does not create unintended duplicates.
- External failures are retryable and clearly reported to the user.

## Phase 9 - Connector Expansion and Provider Portability

### Goal

Broaden source coverage and reduce dependence on individual vendors.

### Implement

- Jira and Azure DevOps source connectors.
- Confluence/wiki, Git repositories, Figma, defect databases, and additional file sources in customer-priority order.
- TestRail, Zephyr, ADO, Jira/Xray, and other delivery connectors in customer-priority order.
- OAuth/token lifecycle, webhook registration, scheduled fallback sync, and connector health status.
- OpenAI, Groq, and Anthropic adapters behind the existing LLM interface.
- Provider configuration, model selection, health checks, cost/usage metrics, and fallback policy.
- Incremental sync, changed-source events, deleted-content handling, and version-aware reindexing.

### Deliverables

- Connector catalog and connector certification checklist.
- Provider comparison and evaluation reports.
- Consistent source and delivery behavior across integrations.

### Exit criteria

- Adding a connector does not require changes to retrieval or generation business logic.
- Switching LLM providers does not change the canonical testcase contract or review workflow.

## Phase 10 - Execution Orchestration and Defect Traceability

### Goal

Connect generated testcases to automated execution and defect management.

### Implement

- Execution orchestration service with API/UI automation and CI/CD integration points.
- Execution job lifecycle, cancellation, retries, results, artifacts, and environment metadata.
- Associations between testcase, execution run, result, and defect.
- Defect creation and update connectors.
- `testcase.execution.started`, `testcase.execution.completed`, and `defect.created` events.
- Execution results and defect views in the React application.

### Deliverables

- Execution APIs and run history.
- Traceability from requirement to testcase to execution result to defect.
- CI/CD integration example and operational runbook.

### Exit criteria

- An approved and published testcase can be associated with an execution run and its result.
- Failed executions can be linked to an existing defect or create one with preserved lineage.

## Phase 11 - Production Hardening and Kubernetes Delivery

### Goal

Make the complete application deployable and operable across cloud and on-premises Kubernetes environments.

### Implement

- Container images and Helm/Kustomize manifests for all services and workers.
- Kubernetes base configuration plus dev, staging, and production overlays.
- Terraform modules for environment dependencies where appropriate.
- Network policies, service identities, least-privilege accounts, pod security, and resource limits.
- Encryption in transit and at rest, secret-manager integration, credential rotation, and configurable prompt/context redaction.
- Circuit breakers, bulkheads, graceful degradation, provider fallback, and dependency timeouts.
- Horizontal scaling policies for API, workers, ingestion, embeddings, retrieval, and generation workloads.
- Backup/restore, disaster recovery, retention, migration, and data-deletion procedures.
- SLOs, alert thresholds, dashboards, on-call runbooks, and incident exercises.
- Load, soak, chaos, security, privacy, and end-to-end tests.

### Deliverables

- Repeatable deployment to dev, staging, and production-like Kubernetes clusters.
- Security threat model and remediation record.
- Capacity, cost, performance, and recovery reports.
- Release checklist and rollback procedure.

### Exit criteria

- The platform meets agreed availability, latency, recovery, security, and data-retention targets.
- A deployment, rollback, backup restore, and dependency-outage drill completes successfully.

## Phase 12 - Optional Voice Interaction

### Goal

Add voice as an alternate input/output channel without coupling it to generation logic.

### Implement

- Browser microphone permissions and audio capture.
- Speech-to-text and text-to-speech provider adapters.
- Voice interaction layer calling the existing query orchestrator.
- Transcripts, consent, redaction, failure handling, and accessibility behavior.
- Usage and latency metrics.

### Exit criteria

- Voice can submit a query and read back a response while using the same retrieval, evidence, audit, and authorization rules as typed interaction.

## Recommended Work Allocation

Teams can work in parallel after the contracts are stable:

| Workstream | Primary phases |
| --- | --- |
| Platform and contracts | 0-2, 11 |
| Identity and web foundation | 1, 7-8 |
| Ingestion and knowledge | 3-5, 9 |
| AI generation and evaluation | 5-7, 9 |
| Integrations | 3, 8-10 |
| Reliability and security | 1-2, 6-11 |

Parallel work should merge only through versioned API, event, and package contracts. A feature is complete only when its backend, worker behavior, UI state, authorization, audit trail, tests, and operational signals are present.

## MVP Release Boundary

The first release should include Phases 0 through 8, with scope limited to:

- One authenticated tenant/project workflow.
- File upload and the highest-priority source connector.
- Text, PDF, spreadsheet, JSON/YAML, and OpenAPI extraction as practical for the target users.
- Mistral hierarchical embeddings and MongoDB hybrid retrieval.
- One LLM provider, canonical schema, configurable templates, and confidence scoring.
- Evidence-backed generation with insufficient-evidence handling.
- Configurable human review and complete audit/lineage.
- JSON/Excel export and one priority QA publishing connector.
- Basic Kubernetes deployment, observability, security controls, and recovery procedures.

Defer broader connector coverage, multiple LLM providers, execution orchestration, and voice until the MVP has measurable retrieval quality, generation quality, review adoption, publishing success, and operational stability.

## Release Gates and Measures

Track these measures throughout delivery:

- Ingestion success rate, processing latency, retry rate, and duplicate rate.
- Retrieval relevance on a maintained golden query set.
- Evidence coverage and unsupported-assumption rate.
- Schema-validation failure rate and testcase duplicate rate.
- Human edit, approval, rejection, and insufficient-evidence rates.
- Export and publishing success rate, retry rate, and external duplicate rate.
- API/job latency, queue depth, provider error rate, and cost per generation.
- Authorization-test coverage, audit completeness, recovery time, and data-restore success.

Promote a phase only when its exit criteria pass in an environment representative of the next phase. Keep architecture changes and contract changes versioned, reviewed, and backward compatible wherever possible.

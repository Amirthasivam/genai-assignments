# AI Testcase Generator --- Architecture

## 1. Executive Summary

The AI Testcase Generator is a microservices-based platform that ingests
heterogeneous engineering and QA sources, builds a version-aware
knowledge base, retrieves relevant context using hybrid search, and
generates structured test cases with configurable human review and
delivery into existing QA platforms.

### Primary goals

-   Ingest requirements, UI specifications, technical documentation,
    release notes, spreadsheets/PDFs, recordings, source code,
    Swagger/OpenAPI definitions, and defect data.
-   Normalize heterogeneous content into a common representation.
-   Create hierarchical embeddings using Mistral AI and store them in
    MongoDB.
-   Retrieve context through query preprocessing, BM25 + vector search,
    rule-based reranking, deduplication, and summarization.
-   Generate test cases through configurable LLM providers: OpenAI,
    Groq, or Anthropic.
-   Produce canonical structured test cases and transform them into
    organization-specific formats.
-   Support configurable human approval/rejection/modification
    workflows.
-   Calculate a hybrid confidence score.
-   Maintain source/version lineage and AI auditability.
-   Publish approved test cases to TestRail, Xray, Zephyr, ADO, Jira, or
    export them.
-   Keep the architecture Kubernetes-portable across AWS, Azure, GCP,
    and on-premises environments.

------------------------------------------------------------------------

## 2. Architecture Decisions From the 20 Questions

  -----------------------------------------------------------------------
  \#                      Decision                Selected
  ----------------------- ----------------------- -----------------------
  1                       Overall architecture    Microservices

  2                       Service communication   Direct
                                                  service-to-service
                                                  communication

  3                       Ingestion trigger       Event-driven

  4                       Source processing       Generic file/content
                                                  extraction

  5                       Knowledge               MongoDB hybrid search
                          storage/search          

  6                       Embedding strategy      Hierarchical embeddings

  7                       Reranking               Rule + score-based

  8                       LLM integration         Configurable provider

  9                       Testcase output         Canonical schema +
                                                  configurable templates

  10                      Testcase type selection AI recommendation +
                                                  user control

  11                      Human review            Configurable workflow

  12                      Confidence score        Hybrid confidence
                                                  engine

  13                      Async processing        Queue + event streaming

  14                      Authorization           Project-level RBAC

  15                      Tenant isolation        Configurable
                                                  multi-tenant isolation

  16                      Observability/audit     AI audit trail

  17                      Versioning              Incremental +
                                                  version-aware

  18                      Integrations            Connector framework +
                                                  API-first +
                                                  independently
                                                  deployable connectors

  19                      Deployment              Cloud-agnostic
                                                  Kubernetes

  20                      Delivery                Export + direct
                                                  publishing + optional
                                                  execution orchestration
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. High-Level Architecture

``` text
                    +-----------------------------+
                    |       React Web App         |
                    +--------------+--------------+
                                   |
                          REST / WebSocket
                                   |
                    +--------------v--------------+
                    |       API / BFF Layer       |
                    +--------------+--------------+
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
        v                          v                          v
+---------------+         +----------------+         +------------------+
| Project/Auth  |         | Testcase       |         | Integration      |
| Service       |         | Generation     |         | Service          |
+---------------+         | Service        |         +--------+---------+
                           +-------+--------+                  |
                                   |                           |
                                   v                           v
                          +----------------+          +-------------------+
                          | Retrieval      |          | Connector Workers |
                          | Service       |          +-------------------+
                          +-------+--------+                    |
                                  |                             |
                     +------------+------------+                |
                     |            |            |                |
                     v            v            v                v
                 Preprocess   Hybrid Search  Rerank       External Systems
                                  |            |
                                  +-----+------+
                                        |
                                        v
                                  +-----------+
                                  | MongoDB   |
                                  | Knowledge |
                                  +-----------+

External source events
        |
        v
+------------------+       +------------------+
| Event Streaming  | ----> | Ingestion       |
| Platform         |       | Service/Workers  |
+------------------+       +--------+---------+
                                    |
                                    v
                           +------------------+
                           | Extraction &     |
                           | Normalization    |
                           +--------+---------+
                                    |
                                    v
                           +------------------+
                           | Embedding        |
                           | Service          |
                           | Mistral AI       |
                           +--------+---------+
                                    |
                                    v
                               MongoDB

Generation path:
Query -> Preprocess -> Hybrid Search -> Rerank -> Deduplicate
-> Summarize -> Prompt Assembly -> Configured LLM
-> Schema Validation -> Confidence -> Review -> Publish/Export
```

------------------------------------------------------------------------

## 4. Major Microservices

### 4.1 API Gateway / BFF

Responsibilities:

-   Frontend-facing REST APIs.
-   Request authentication and project context.
-   Request validation.
-   Routing to backend services.
-   Correlation ID propagation.
-   WebSocket/SSE updates for long-running jobs where required.

> Although service-to-service communication is direct, the frontend
> should have a stable API boundary rather than calling internal
> microservices directly.

### 4.2 Identity & Project Service

Responsibilities:

-   User identity mapping.
-   Project management.
-   Project membership.
-   Project-level RBAC.
-   Tenant isolation configuration.
-   Integration credentials metadata.

Suggested roles:

-   Admin
-   Architect
-   QA Lead
-   Tester
-   Developer
-   Viewer

### 4.3 Connector Service

Responsibilities:

-   Source-specific connector lifecycle.
-   OAuth/token management.
-   Webhook registration.
-   Scheduled fallback synchronization where a source cannot provide
    events.
-   Normalized connector contract.
-   Source metadata and source-system identifiers.

Initial connectors:

-   Jira
-   Azure DevOps
-   TestRail
-   Xray
-   Zephyr
-   Confluence/wiki
-   Figma
-   Git repositories
-   Swagger/OpenAPI
-   Defect databases
-   File upload

### 4.4 Ingestion Service

Responsibilities:

1.  Receive ingestion event.
2.  Fetch source content.
3.  Extract content.
4.  Normalize content.
5.  Classify source/content.
6.  Split into hierarchical chunks.
7.  Generate document/section/chunk embeddings.
8.  Persist content, metadata, embeddings, lineage, and versions.
9.  Emit `content.indexed`.

Processing should be asynchronous.

### 4.5 Extraction & Normalization Service

Responsibilities:

-   PDF/text extraction.
-   Excel/tabular extraction.
-   JSON/YAML/OpenAPI extraction.
-   Code parsing.
-   Recording transcription integration.
-   HTML/wiki extraction.
-   Common canonical content model.

The initial architecture uses a generic extraction pipeline.
Source-specific semantic enrichment can be added later without changing
the downstream retrieval contract.

### 4.6 Embedding Service

Responsibilities:

-   Generate hierarchical embeddings.
-   Use Mistral AI embedding model.
-   Batch requests.
-   Retry transient failures.
-   Track embedding model/version.
-   Support re-embedding when model configuration changes.

Levels:

``` text
Document
  └── Section
       └── Chunk
```

### 4.7 Retrieval Service

Responsibilities:

1.  Query normalization.
2.  Abbreviation expansion.
3.  Synonym expansion.
4.  Hybrid BM25 + vector retrieval.
5.  Rule/score-based reranking.
6.  Deduplication.
7.  Context summarization.
8.  Return evidence with source/version metadata.

### 4.8 Testcase Generation Service

Responsibilities:

-   Determine relevant testcase types.
-   Use retrieved evidence.
-   Build prompt/context.
-   Invoke configured LLM provider.
-   Validate output against canonical testcase schema.
-   Calculate confidence.
-   Store generation trace.
-   Create reviewable testcase version.

### 4.9 LLM Provider Adapter

Provider-neutral interface:

``` text
LLMProvider
  - generate()
  - generateStructured()
  - healthCheck()
  - getModelMetadata()
```

Implementations:

-   OpenAI adapter
-   Groq adapter
-   Anthropic adapter

Provider selection should be configuration-driven rather than embedded
throughout business logic.

### 4.10 Review Service

Responsibilities:

-   Approve.
-   Reject.
-   Modify with comments.
-   Maintain review history.
-   Apply project-specific workflow.
-   Prevent publishing when a configured approval gate is not satisfied.

### 4.11 Delivery / Export Service

Responsibilities:

-   Generate JSON/CSV/Excel/PDF exports.
-   Publish to TestRail.
-   Publish to Xray.
-   Publish to Zephyr.
-   Publish to ADO.
-   Publish to Jira.
-   Maintain external-system IDs.
-   Maintain requirement-to-testcase-to-execution traceability.

### 4.12 Execution Orchestration Service

Optional/phase-2 capability:

-   Trigger API/UI automation.
-   Integrate with CI/CD.
-   Collect execution results.
-   Associate results with generated testcases.
-   Associate failures with defects.

------------------------------------------------------------------------

## 5. Ingestion Pipeline

``` text
External Source
      |
      v
Webhook / Event
      |
      v
Event Stream
      |
      v
Ingestion Job
      |
      v
Fetch Content
      |
      v
Extract
      |
      v
Normalize
      |
      v
Classify
      |
      v
Hierarchical Chunking
      |
      +---- Document embedding
      +---- Section embedding
      +---- Chunk embedding
      |
      v
MongoDB
      |
      v
content.indexed event
```

### Canonical content model

``` json
{
  "documentId": "doc-123",
  "projectId": "project-1",
  "tenantId": "tenant-1",
  "sourceType": "jira",
  "sourceId": "PROJ-123",
  "versionId": "v7",
  "parentVersionId": "v6",
  "title": "Checkout validation",
  "contentType": "requirement",
  "text": "...",
  "hierarchy": {
    "document": "doc-123",
    "section": "checkout",
    "chunk": "chunk-42"
  },
  "metadata": {
    "labels": ["checkout", "payment"],
    "status": "approved",
    "updatedAt": "..."
  },
  "embedding": {
    "model": "mistral-embedding",
    "level": "chunk",
    "vector": []
  }
}
```

------------------------------------------------------------------------

## 6. Retrieval Pipeline

``` text
User Query
   |
   v
Normalization
   |
   v
Abbreviation Expansion
   |
   v
Synonym Expansion
   |
   +-------------------+
   |                   |
   v                   v
BM25 Search       Vector Search
   |                   |
   +---------+---------+
             |
             v
       Candidate Merge
             |
             v
       Rule-Based Rerank
             |
             v
        Deduplication
             |
             v
        Summarization
             |
             v
     Evidence + Context
```

### Reranking signals

Suggested configurable weights:

-   Vector similarity: 30%
-   BM25 relevance: 25%
-   Source authority: 15%
-   Requirement/status relevance: 10%
-   Freshness: 10%
-   Project metadata match: 10%

Weights should be configuration, not hard-coded constants.

------------------------------------------------------------------------

## 7. Testcase Generation Flow

``` text
User Request
    |
    v
Testcase Type Recommendation
    |
    v
User Accepts/Modifies Types
    |
    v
Retrieval Service
    |
    v
Relevant Evidence
    |
    v
Prompt Assembly
    |
    +---- System instructions
    +---- User query
    +---- Retrieved context
    +---- Project testcase template
    +---- Output schema
    |
    v
Configured LLM
    |
    v
Structured JSON
    |
    v
Schema Validation
    |
    v
Duplicate/Quality Checks
    |
    v
Confidence Engine
    |
    v
Review Workflow
    |
    +---- Approved
    +---- Rejected
    +---- Modified
    |
    v
Export / Publish / Execute
```

------------------------------------------------------------------------

## 8. Canonical Testcase Schema

``` json
{
  "testCaseId": "TC-0001",
  "projectId": "project-1",
  "title": "Verify checkout rejects an expired card",
  "type": "negative",
  "priority": "high",
  "preconditions": [
    "User is authenticated",
    "Cart contains at least one item"
  ],
  "testData": [
    {
      "name": "card",
      "value": "expired test card"
    }
  ],
  "steps": [
    {
      "step": 1,
      "action": "Enter an expired card",
      "expectedResult": "Card details are accepted for validation"
    },
    {
      "step": 2,
      "action": "Submit payment",
      "expectedResult": "Payment is rejected with the expected validation message"
    }
  ],
  "requirementReferences": ["PROJ-123"],
  "sourceReferences": ["doc-123:v7:chunk-42"],
  "confidence": 0.91,
  "status": "pending_review"
}
```

------------------------------------------------------------------------

## 9. Testcase Type Framework

The system should support:

-   Functional
-   Positive
-   Negative
-   Boundary
-   Equivalence partitioning
-   Regression
-   Smoke
-   Sanity
-   Integration
-   API
-   UI
-   Security-oriented functional checks
-   Data validation
-   Error handling
-   Accessibility-oriented checks where UI evidence supports them

AI should recommend relevant types, but the user can accept, remove, or
add types.

------------------------------------------------------------------------

## 10. Confidence Engine

Confidence should not rely exclusively on the LLM.

Suggested model:

``` text
Confidence =
  Retrieval Quality
+ Source Quality
+ Evidence Coverage
+ Schema/Rule Validation
+ Requirement Coverage
+ LLM Assessment
```

Normalize to `0.0 - 1.0`.

Store both:

-   Overall confidence.
-   Component-level confidence/explanations.

Example:

``` json
{
  "overall": 0.91,
  "retrieval": 0.95,
  "sourceQuality": 0.94,
  "requirementCoverage": 0.90,
  "validation": 1.0,
  "llmAssessment": 0.82
}
```

------------------------------------------------------------------------

## 11. Human-in-the-Loop Workflow

``` text
Generated
   |
   v
Pending Review
   |
   +---- Approve ------> Approved
   |
   +---- Reject -------> Rejected
   |
   +---- Modify -------> Modified
                             |
                             v
                       Pending Review
```

Project configuration determines whether:

-   Every testcase requires approval.
-   Only low-confidence cases require approval.
-   Review is optional.

------------------------------------------------------------------------

## 12. Versioning & Lineage

Every generated testcase should be traceable to:

``` text
Requirement
   |
   v
Source Document Version
   |
   v
Section / Chunk
   |
   v
Retrieval Result
   |
   v
Prompt Version
   |
   v
LLM Provider + Model
   |
   v
Generated Testcase Version
   |
   v
Review Decision
   |
   v
Published Testcase
   |
   v
Execution Result
   |
   v
Defect
```

This is critical for auditability and explaining why a testcase was
generated.

------------------------------------------------------------------------

## 13. MongoDB Logical Collections

Recommended collections:

``` text
tenants
users
projects
project_members
integrations
integration_credentials
source_documents
source_versions
content_chunks
embeddings
retrieval_queries
retrieval_results
testcase_generations
testcases
testcase_versions
review_actions
export_jobs
execution_runs
defects
prompt_templates
generation_templates
audit_events
jobs
```

Indexes should include:

-   `tenantId`
-   `projectId`
-   `sourceType`
-   `sourceId`
-   `versionId`
-   `contentType`
-   timestamps
-   status fields
-   searchable text fields
-   vector indexes
-   compound metadata indexes

------------------------------------------------------------------------

## 14. API Endpoints

### Project & Access

``` text
POST   /api/v1/projects
GET    /api/v1/projects
GET    /api/v1/projects/:projectId
PATCH  /api/v1/projects/:projectId
GET    /api/v1/projects/:projectId/members
POST   /api/v1/projects/:projectId/members
PATCH  /api/v1/projects/:projectId/members/:userId
DELETE /api/v1/projects/:projectId/members/:userId
```

### Integrations

``` text
GET    /api/v1/projects/:projectId/integrations
POST   /api/v1/projects/:projectId/integrations
GET    /api/v1/integrations/:integrationId
PATCH  /api/v1/integrations/:integrationId
DELETE /api/v1/integrations/:integrationId
POST   /api/v1/integrations/:integrationId/test
POST   /api/v1/integrations/:integrationId/sync
POST   /api/v1/webhooks/:connectorType
```

### Ingestion

``` text
POST   /api/v1/projects/:projectId/sources/upload
POST   /api/v1/projects/:projectId/ingestion/jobs
GET    /api/v1/projects/:projectId/ingestion/jobs
GET    /api/v1/ingestion/jobs/:jobId
POST   /api/v1/ingestion/jobs/:jobId/retry
POST   /api/v1/ingestion/jobs/:jobId/cancel
GET    /api/v1/projects/:projectId/documents
GET    /api/v1/documents/:documentId
GET    /api/v1/documents/:documentId/versions
```

### Knowledge / Search

``` text
POST   /api/v1/projects/:projectId/search
GET    /api/v1/projects/:projectId/documents/:documentId/chunks
POST   /api/v1/projects/:projectId/reindex
POST   /api/v1/projects/:projectId/reembed
```

### Testcase Generation

``` text
POST   /api/v1/projects/:projectId/testcase-recommendations
POST   /api/v1/projects/:projectId/testcases/generate
GET    /api/v1/projects/:projectId/testcases
GET    /api/v1/testcases/:testCaseId
PATCH  /api/v1/testcases/:testCaseId
POST   /api/v1/testcases/:testCaseId/regenerate
POST   /api/v1/testcases/:testCaseId/duplicate-check
GET    /api/v1/testcases/:testCaseId/evidence
GET    /api/v1/testcases/:testCaseId/lineage
```

### Review

``` text
POST   /api/v1/testcases/:testCaseId/reviews
GET    /api/v1/testcases/:testCaseId/reviews
POST   /api/v1/testcases/:testCaseId/approve
POST   /api/v1/testcases/:testCaseId/reject
POST   /api/v1/testcases/:testCaseId/modify
```

### Templates & Configuration

``` text
GET    /api/v1/projects/:projectId/testcase-templates
POST   /api/v1/projects/:projectId/testcase-templates
PATCH  /api/v1/testcase-templates/:templateId
DELETE /api/v1/testcase-templates/:templateId

GET    /api/v1/projects/:projectId/prompt-templates
POST   /api/v1/projects/:projectId/prompt-templates
PATCH  /api/v1/prompt-templates/:templateId

GET    /api/v1/projects/:projectId/llm-config
PATCH  /api/v1/projects/:projectId/llm-config
```

### Export & Publishing

``` text
POST   /api/v1/projects/:projectId/exports
GET    /api/v1/exports/:exportId
GET    /api/v1/exports/:exportId/download

POST   /api/v1/testcases/:testCaseId/publish
POST   /api/v1/projects/:projectId/publish
GET    /api/v1/testcases/:testCaseId/external-links
```

### Execution

``` text
POST   /api/v1/testcases/:testCaseId/executions
GET    /api/v1/executions/:executionId
GET    /api/v1/testcases/:testCaseId/executions
POST   /api/v1/executions/:executionId/cancel
```

### Audit

``` text
GET    /api/v1/projects/:projectId/audit-events
GET    /api/v1/testcases/:testCaseId/audit-trail
GET    /api/v1/testcases/:testCaseId/generation-trace
```

------------------------------------------------------------------------

## 15. Event Contracts

Recommended events:

``` text
source.connected
source.changed
ingestion.requested
ingestion.started
content.extracted
content.normalized
embedding.requested
content.indexed
retrieval.completed
testcase.generation.requested
testcase.generated
testcase.validation.failed
testcase.review.requested
testcase.approved
testcase.rejected
testcase.modified
testcase.published
testcase.execution.started
testcase.execution.completed
defect.created
```

Events should include:

``` json
{
  "eventId": "evt-123",
  "eventType": "content.indexed",
  "tenantId": "tenant-1",
  "projectId": "project-1",
  "timestamp": "...",
  "correlationId": "corr-123",
  "payload": {}
}
```

------------------------------------------------------------------------

## 16. Recommended Project Folder Structure

``` text
ai-testcase-generator/
│
├── apps/
│   ├── web/
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   ├── features/
│   │   │   │   ├── projects/
│   │   │   │   ├── integrations/
│   │   │   │   ├── ingestion/
│   │   │   │   ├── search/
│   │   │   │   ├── testcases/
│   │   │   │   ├── reviews/
│   │   │   │   └── settings/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   └── types/
│   │   └── package.json
│   │
│   └── api-gateway/
│       └── src/
│
├── services/
│   ├── project-service/
│   ├── connector-service/
│   ├── ingestion-service/
│   ├── extraction-service/
│   ├── embedding-service/
│   ├── retrieval-service/
│   ├── testcase-generation-service/
│   ├── llm-provider-service/
│   ├── review-service/
│   ├── export-service/
│   ├── execution-service/
│   └── audit-service/
│
├── packages/
│   ├── common-types/
│   ├── api-contracts/
│   ├── event-contracts/
│   ├── auth/
│   ├── logger/
│   ├── observability/
│   ├── mongodb/
│   ├── queue/
│   ├── event-stream/
│   ├── testcase-schema/
│   ├── prompt-engine/
│   └── connector-sdk/
│
├── connectors/
│   ├── jira/
│   ├── azure-devops/
│   ├── testrail/
│   ├── xray/
│   ├── zephyr/
│   ├── confluence/
│   ├── figma/
│   ├── git/
│   ├── swagger/
│   ├── defect-db/
│   └── file-upload/
│
├── infrastructure/
│   ├── kubernetes/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── production/
│   ├── helm/
│   ├── terraform/
│   └── docker/
│
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── event-contracts/
│   ├── security/
│   └── runbooks/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   ├── e2e/
│   └── evaluation/
│
├── .github/
│   └── workflows/
│
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
└── README.md
```

------------------------------------------------------------------------

## 17. Backend Service Internal Structure

Each Node.js + TypeScript service should follow a consistent structure:

``` text
service-name/
└── src/
    ├── api/
    │   ├── controllers/
    │   ├── routes/
    │   ├── validators/
    │   └── dto/
    ├── application/
    │   ├── commands/
    │   ├── queries/
    │   └── use-cases/
    ├── domain/
    │   ├── entities/
    │   ├── value-objects/
    │   └── interfaces/
    ├── infrastructure/
    │   ├── mongodb/
    │   ├── messaging/
    │   ├── external/
    │   └── providers/
    ├── workers/
    ├── config/
    ├── observability/
    └── index.ts
```

Use Express for HTTP APIs and TypeScript throughout the backend.

------------------------------------------------------------------------

## 18. Frontend Architecture

React application areas:

``` text
Dashboard
  |
  +-- Projects
  +-- Sources & Integrations
  +-- Ingestion Jobs
  +-- Knowledge Search
  +-- Testcase Generator
  +-- Generated Testcases
  +-- Review Queue
  +-- Templates
  +-- Execution Results
  +-- Audit / Traceability
  +-- Project Settings
```

Important UI workflows:

1.  Connect source.
2.  Observe ingestion status.
3.  Search knowledge.
4.  Enter testcase-generation request.
5.  Review AI-recommended testcase types.
6.  Generate testcases.
7.  Inspect evidence and confidence.
8.  Approve/reject/modify.
9.  Export/publish.
10. Track execution and defects.

------------------------------------------------------------------------

## 19. Voice Chat --- Nice to Have

Add a separate voice interaction layer rather than coupling voice logic
to testcase generation.

``` text
Browser microphone
       |
       v
Speech-to-Text
       |
       v
Query Orchestrator
       |
       v
Retrieval + LLM
       |
       v
Text response
       |
       v
Text-to-Speech
```

Voice should initially be treated as an alternate UI input/output
channel.

------------------------------------------------------------------------

## 20. Security Architecture

Minimum controls:

-   Project-level authorization on every resource.
-   Encryption in transit.
-   Encryption at rest.
-   Secrets stored outside source code.
-   Connector credentials encrypted.
-   Prompt/context logging with configurable redaction.
-   PII/secrets filtering before sending content to external LLMs where
    required.
-   Audit events for sensitive operations.
-   Rate limiting.
-   Request validation.
-   Service identity for internal calls.
-   Network policies in Kubernetes.
-   Least-privilege service accounts.

------------------------------------------------------------------------

## 21. Reliability

Recommended patterns:

-   Idempotency keys for ingestion and publishing.
-   Retry with exponential backoff.
-   Dead-letter queues.
-   Circuit breakers for external APIs and LLM providers.
-   Timeouts.
-   Bulkheads between workloads.
-   Job state machine for long-running processes.
-   Graceful degradation when an external source or LLM provider is
    unavailable.

------------------------------------------------------------------------

## 22. AI-Specific Guardrails

The generator should not treat retrieved context as unquestionable
truth.

Generation should enforce:

1.  Evidence-backed generation.
2.  Source/version references.
3.  No unsupported requirement assumptions.
4.  Structured output validation.
5.  Duplicate testcase detection.
6.  Requirement coverage checks.
7.  Confidence scoring.
8.  Human review when configured.
9.  Prompt/template versioning.
10. Model/provider version tracking.

A testcase should be marked as **insufficient evidence** rather than
invented when the retrieval context does not support it.

------------------------------------------------------------------------

## 23. Recommended Initial MVP

Build the first production-capable slice around:

``` text
React
  |
API Gateway
  |
Project Service
  |
Connector/File Upload
  |
Ingestion Service
  |
Extraction + Normalization
  |
Mistral Embeddings
  |
MongoDB
  |
Retrieval Service
  |
Configured LLM Provider
  |
Testcase Generation
  |
Human Review
  |
JSON/Excel Export
```

Then add:

-   Jira/ADO connectors.
-   TestRail/Xray/Zephyr publishing.
-   Figma.
-   Confluence.
-   Git.
-   Swagger/OpenAPI.
-   Defect integrations.
-   Event streaming.
-   Execution orchestration.
-   Voice chat.

------------------------------------------------------------------------

## 24. Key Architectural Principles

### Separation of concerns

Ingestion, retrieval, generation, review, and delivery should remain
independently evolvable.

### Provider abstraction

LLM and embedding providers must be behind interfaces.

### Evidence first

Every generated testcase should be explainable through retrieved source
evidence.

### Version awareness

Never lose the source version that supported an AI-generated testcase.

### Async by default

Long-running ingestion and generation should be job-based rather than
blocking HTTP requests.

### API-first

All major capabilities should have stable APIs.

### Connector extensibility

Adding a new source should require implementing a connector contract,
not modifying the core generation engine.

### Human control

AI should accelerate testcase engineering, while project-configured
review workflows retain human authority.

------------------------------------------------------------------------

## 25. Target Technology Stack

  Layer               Technology
  ------------------- -------------------------------------------------
  Frontend            React + TypeScript
  Backend             Node.js + TypeScript
  HTTP                Express
  Knowledge DB        MongoDB
  Vector Search       MongoDB Vector Search
  Keyword Search      MongoDB/BM25-capable search
  Embeddings          Mistral AI
  LLM                 OpenAI / Groq / Anthropic
  Async Jobs          Queue system
  Event Backbone      Event streaming platform
  Deployment          Kubernetes
  Infrastructure      Cloud-agnostic
  API Contract        OpenAPI
  Authentication      Project-level RBAC; enterprise IdP can be added
  Observability       Centralized logs, metrics, traces
  Audit               AI generation and review trace
  Frontend realtime   WebSocket/SSE where required

------------------------------------------------------------------------

## 26. Architectural Outcome

The resulting architecture is a **cloud-agnostic, event-driven
microservices platform** with:

-   Generic ingestion and normalization.
-   Hierarchical Mistral embeddings.
-   MongoDB hybrid retrieval.
-   Rule-based reranking.
-   Configurable LLM providers.
-   Structured/template-driven testcase generation.
-   AI-assisted testcase type selection.
-   Configurable human-in-the-loop review.
-   Hybrid confidence scoring.
-   Version-aware knowledge management.
-   Extensible connector framework.
-   Direct publishing and optional execution orchestration.
-   End-to-end AI auditability and requirement traceability.

The architecture is intentionally modular so the MVP can be implemented
incrementally without locking the system into a single LLM provider,
cloud provider, or external QA platform.

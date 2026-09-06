# Resume RAG: Reranking, Deduplication, and Summarization Architecture

## 1. Purpose and Scope

This document defines the implementation plan for adding three capabilities to the current Resume RAG application:

1. Frontend integration for the existing backend reranking endpoint.
2. Frontend integration for the existing backend summarization endpoint.
3. New backend and frontend deduplication functionality for finding and removing duplicate resume records.

The plan follows the phase-wise structure of `Retreival Pipeline/02_Resume_Retrieval_Phasewise_Implementation.md` and the component, API-client, state, accessibility, and responsive UI conventions in `UI Integration/FRONTEND_WEB_INSTRUCTIONS_FINAL_V1 3.md`.

The implementation remains in the existing two-application workspace:

```text
backend/   Node.js + TypeScript + Express + MongoDB
frontend/  React 18 + TypeScript + Vite + Zustand + TailwindCSS
```

No second server, database, resume collection, embedding pipeline, or LLM integration should be introduced.

## 2. Current Implementation Baseline

### 2.1 Existing backend routes

`backend/src/app.ts` mounts both ingestion and retrieval routes under `/v1`. The current retrieval router exposes:

| Method | Endpoint | Current responsibility |
|---|---|---|
| GET | `/v1/search/readiness` | Reports whether stored resume embeddings are usable. |
| GET | `/v1/candidate/:id` | Returns one candidate profile without its embedding. |
| POST | `/v1/embeddings` | Generates a Mistral query embedding. |
| POST | `/v1/search/bm25` | Executes lexical Atlas Search. |
| POST | `/v1/search/vector` | Generates a query embedding and executes vector search. |
| POST | `/v1/search/hybrid` | Returns BM25 and vector result lists for debugging. |
| POST | `/v1/search/rerank` | Sends supplied candidates to the existing Groq reranker. |
| POST | `/v1/search/summarize` | Sends one supplied candidate to the existing Groq summarizer. |
| POST | `/v1/search` | Runs the end-to-end BM25/vector/merge/rerank/optional-summary workflow. |

### 2.2 Existing backend components to reuse

| Existing component | Reuse in this work |
|---|---|
| `retrievalRoutes.ts` | Keep rerank and summarize routes; add deduplication routes to the same versioned router or a dedicated router mounted under `/v1`. |
| `retrievalController.ts` | Preserve current rerank and summarize validation and response shapes. Add deduplication controllers. |
| `SearchService.ts` | Continue to own end-to-end retrieval, merge results, fallback behavior, and optional summaries. Do not duplicate reranking or summarization in the frontend. |
| `LLMService.ts` | Reuse `rerankCandidates` and `summarizeCandidateFit`; do not create another Groq client. |
| `ResumeRepository.ts` | Reuse the `resumes` collection and add explicitly named deduplication read/delete methods. |
| `retrieval/utils/deduplicate.ts` | Keep its current meaning: it merges the same resume appearing in BM25 and vector results by `resumeId`. It is not sufficient for record-level duplicate detection. |
| `EmbeddingService.ts` | Continue using the ingestion embedding service for query and resume embeddings. |
| request ID, logger, error handler | Use for all new deduplication requests and errors. |

### 2.3 Existing frontend structure to extend

The current frontend has:

```text
frontend/src/
├── components/layout/Sidebar.tsx
├── components/layout/MobileDrawer.tsx
├── components/features/sidebar/
├── components/features/results/
├── components/features/candidate/
├── hooks/use-search.ts
├── lib/api/client.ts
├── lib/api/search.api.ts
├── lib/stores/search.store.ts
├── pages/ChatPage.tsx
├── features/ingestion/
└── types/
```

`FrontendApp.tsx` currently routes `/` to `ChatPage` and `/ingestion` to `IngestionPage`. The new sections should be added without breaking the existing search and ingestion paths. The sidebar must remain usable in the mobile drawer as well as the desktop layout.

## 3. Overall Architecture and Data Flow

```text
                         Existing ingestion
PDF upload → parse → embedding → MongoDB `resumes`
                                      │
                                      ▼
                              Retrieval readiness
                                      │
Recruiter query ───────────────► POST /v1/search
                                      │
                       ┌──────────────┴──────────────┐
                       ▼                             ▼
                 BM25 search                   Query embedding
                       │                             │
                       └──────────────┬──────────────┘
                                      ▼
                         Merge by resumeId and limit
                                      ▼
                            Existing Groq reranker
                                      │
                                      ▼
                   Optional existing Groq summarization
                                      │
                                      ▼
                      Ranked candidate results in frontend

Direct frontend actions:
  Reranking       → POST /v1/search/rerank
  Summarization   → POST /v1/search/summarize
  Deduplication   → POST /v1/deduplication/scan
                    POST /v1/deduplication/remove

Deduplication backend:
  resumes → normalize identity fields → exact/fingerprint matches
          → duplicate groups → explicit keep/delete confirmation
```

### Architecture rules

- Reranking and summarization remain backend-owned LLM operations.
- The frontend sends only backend-supported fields and renders backend responses.
- The existing end-to-end search endpoint remains the preferred production path for normal candidate search.
- Direct reranking and summarization screens are validation, inspection, and focused-action surfaces; they must not reimplement BM25, vector search, prompt construction, or LLM parsing.
- Deduplication must default to a non-destructive scan. Deletion requires an explicit user action and a server-side confirmation payload.
- A duplicate group must never be removed solely because two resumes have similar names. At least one stronger identity or content signal is required.

## 4. Existing Reranking Contract

### 4.1 Endpoint

```http
POST /v1/search/rerank
Content-Type: application/json
```

### 4.2 Request contract

```json
{
  "query": "Need a senior QA architect experienced in RAG, DeepEval and MCP",
  "candidates": [
    {
      "resumeId": "691db80aa895776f97b6eca6",
      "name": "Rajesh Mohan Kumar",
      "role": "Test Architect",
      "company": "Example Corp",
      "skills": ["RAG", "DeepEval", "MCP"],
      "snippet": "Test Architect with 13+ years of experience..."
    }
  ],
  "topK": 10
}
```

Required fields:

- `query`: non-empty string; maximum length is the backend `env.maxSearchQueryLength`.
- `candidates`: non-empty array; no more than `env.maxRetrievalCandidates` entries.
- Each candidate requires a non-empty string `resumeId`.

Optional candidate fields accepted by the controller:

- `name`: string
- `role`: string
- `company`: string
- `skills`: array of strings
- `snippet`: string, maximum 2,000 characters

Optional `topK` is an integer from 1 through 100. The controller defaults it to `env.retrievalDefaultTopK`.

### 4.3 Response contract

```json
{
  "results": [
    {
      "resumeId": "691db80aa895776f97b6eca6",
      "rank": 1,
      "relevanceScore": 0.96,
      "reason": "Strong match for senior agentic QA and RAG requirements."
    }
  ],
  "model": "meta-llama/llama-4-scout-17b-16e-instruct"
}
```

The backend validates that returned IDs came from the submitted candidates, `rank` is a positive integer, and `relevanceScore` is between 0 and 1. The frontend must preserve result ordering and display `reason` as backend-generated grounded explanation text.

### 4.4 Error contract

| Status | `errorCode` | Meaning |
|---|---|---|
| 400 | `INVALID_SEARCH_QUERY` | Query is missing or empty. |
| 400 | `INVALID_CANDIDATES` | Candidate list or candidate ID is invalid. |
| 413 | `SEARCH_QUERY_TOO_LARGE` | Query exceeds configured limit. |
| 413 | `TOO_MANY_CANDIDATES` | Candidate list exceeds configured limit. |
| 413 | `CANDIDATE_SNIPPET_TOO_LARGE` | A snippet exceeds 2,000 characters. |
| 400 | `INVALID_TOP_K` | `topK` is outside 1 through 100. |
| 502 | `RERANK_FAILED` | Groq reranking failed or returned invalid data. |

All error responses also include `success: false`, `requestId`, and a user-safe `message`.

## 5. Existing Summarization Contract

### 5.1 Endpoint

```http
POST /v1/search/summarize
Content-Type: application/json
```

### 5.2 Request contract

```json
{
  "query": "Senior QA architect with GenAI RAG experience",
  "candidate": {
    "resumeId": "691db80aa895776f97b6eca6",
    "name": "Rajesh Mohan Kumar",
    "role": "Test Architect",
    "company": "Example Corp",
    "skills": ["RAG", "DeepEval", "MCP"],
    "snippet": "13+ years, RAG, DeepEval, MCP and agentic QA experience"
  },
  "style": "short",
  "maxTokens": 150
}
```

Required fields:

- `query`: non-empty string within `env.maxSearchQueryLength`.
- `candidate.resumeId`: non-empty string.
- `candidate.snippet`: non-empty string, maximum 2,000 characters.

Optional fields:

- `candidate.name`, `candidate.role`, `candidate.company`: strings.
- `candidate.skills`: array of strings.
- `style`: exactly `short` or `detailed`; defaults to `short`.
- `maxTokens`: integer from 1 through 2,000; defaults to 150.

### 5.3 Response contract

```json
{
  "resumeId": "691db80aa895776f97b6eca6",
  "summary": "Strong fit for a senior QA/GenAI architecture role."
}
```

### 5.4 Error contract

| Status | `errorCode` | Meaning |
|---|---|---|
| 400 | `INVALID_SEARCH_QUERY` | Query is missing or empty. |
| 400 | `INVALID_CANDIDATE` | ID or non-empty snippet is missing. |
| 400 | `INVALID_SUMMARY_STYLE` | Style is not `short` or `detailed`. |
| 400 | `INVALID_MAX_TOKENS` | Token count is outside 1 through 2,000. |
| 413 | `SEARCH_QUERY_TOO_LARGE` | Query exceeds configured limit. |
| 502 | `SUMMARIZATION_FAILED` | Groq summarization failed or returned invalid data. |

## 6. Deduplication Design

### 6.1 Objective

Add a backend capability that scans the existing `resumes` collection, identifies records representing the same resume, presents explainable duplicate groups, and removes only records explicitly selected by the user.

This is separate from `mergeAndDeduplicateCandidates`, which only collapses one resume appearing in two retrieval strategies during a search request.

### 6.2 Matching strategy

Use deterministic and conservative matching in this order:

1. Normalize available identity values:
   - email: trim, lowercase, remove surrounding whitespace;
   - phone: retain digits and normalize country-prefix representation where possible;
   - name: trim, lowercase, collapse whitespace, remove punctuation;
   - role/company: trim, lowercase, collapse whitespace;
   - raw text and structured fields: normalize line endings and whitespace.
2. Calculate a stable content fingerprint from normalized resume content. The fingerprint input should include `rawText` and stable structured fields, excluding `_id`, `embedding`, and volatile metadata.
3. Classify a pair as an automatic duplicate only when one of these strong rules is true:
   - identical normalized non-empty email;
   - identical normalized phone plus compatible normalized name;
   - identical content fingerprint.
4. For records without strong identity signals, return a `possible` match only when multiple weaker signals agree, such as normalized name plus company/role plus high content similarity. Possible matches require explicit review and must never be auto-deleted.
5. Group records transitively by a deterministic `groupId` derived from sorted resume IDs. Do not use LLM output as the deletion decision.

The first implementation may use exact fingerprints and normalized identity fields. Approximate text similarity, if introduced later, must be isolated behind a service interface and covered by precision/recall tests before enabling deletion.

### 6.3 Canonical record selection

The scan should recommend a record to keep, but the client must be able to override it. The default recommendation ranks records by:

1. Successful ingestion completeness: name, email, phone, role, skills, experience, and raw text.
2. Most recent ingestion metadata if such metadata exists.
3. Longest non-empty normalized resume content.
4. Stable lexical `_id` tie-breaker.

No field should be silently merged into the kept document in the first version. Removal deletes only the selected duplicate IDs; data merging is a separate future capability.

### 6.4 New versus reused components

**New implementation required:**

```text
backend/src/modules/deduplication/
├── controllers/deduplicationController.ts
├── routes/deduplicationRoutes.ts
├── services/DeduplicationService.ts
├── repositories/DeduplicationRepository.ts
├── utils/normalizeResume.ts
├── utils/resumeFingerprint.ts
└── types/deduplication.types.ts
```

The exact filenames may follow the retrieval module naming convention if the implementation keeps deduplication under `modules/retrieval`, but the ownership boundary must remain explicit.

**Reused:** MongoDB connection from `config/database.ts`, `ResumeDocument` fields where suitable, request ID middleware, logger, centralized error handler, and existing frontend API client/toast conventions.

**Do not reuse as the deduplication algorithm:** `retrieval/utils/deduplicate.ts`; its contract and tests are search-result merging only.

## 7. Deduplication API Contract

### 7.1 Scan endpoint

```http
POST /v1/deduplication/scan
Content-Type: application/json
```

Request:

```json
{
  "matchMode": "exact",
  "includePossible": true,
  "limit": 100
}
```

All request fields are optional:

- `matchMode`: currently `exact`; reject unsupported values rather than silently changing behavior.
- `includePossible`: boolean, default `false`; controls whether conservative possible matches are included.
- `limit`: integer from 1 through 500, default 100 groups or implementation-configured safe limit.

Response:

```json
{
  "scanId": "scan-2026-09-06T12:00:00.000Z",
  "matchMode": "exact",
  "scannedCount": 42,
  "duplicateGroupCount": 1,
  "groups": [
    {
      "groupId": "sha256-group-id",
      "confidence": "exact",
      "matchReasons": ["IDENTICAL_EMAIL", "IDENTICAL_FINGERPRINT"],
      "recommendedKeepId": "691db80aa895776f97b6eca6",
      "records": [
        {
          "resumeId": "691db80aa895776f97b6eca6",
          "name": "Rajesh Mohan Kumar",
          "email": "rajesh@example.com",
          "role": "Test Architect",
          "totalExperience": 13,
          "contentFingerprint": "sha256...",
          "isRecommendedKeep": true
        },
        {
          "resumeId": "691db80aa895776f97b6eca7",
          "name": "Rajesh Mohan Kumar",
          "email": "rajesh@example.com",
          "role": "Test Architect",
          "totalExperience": 13,
          "contentFingerprint": "sha256...",
          "isRecommendedKeep": false
        }
      ]
    }
  ]
}
```

The response must not expose embeddings or full raw resume text. `contentFingerprint` may be returned for audit/debugging, but never the source text used to calculate it.

### 7.2 Removal endpoint

```http
POST /v1/deduplication/remove
Content-Type: application/json
```

Request:

```json
{
  "scanId": "scan-2026-09-06T12:00:00.000Z",
  "groupId": "sha256-group-id",
  "keepResumeId": "691db80aa895776f97b6eca6",
  "removeResumeIds": ["691db80aa895776f97b6eca7"],
  "confirm": true
}
```

Validation rules:

- `scanId`, `groupId`, `keepResumeId`, and `removeResumeIds` are required.
- `confirm` must be exactly `true`.
- `removeResumeIds` must be non-empty, unique, and must not contain `keepResumeId`.
- Every selected ID must be a valid MongoDB ObjectId and belong to the scanned group.
- The server re-reads the records and revalidates the duplicate relationship immediately before deletion to avoid stale-scan deletion.
- The endpoint deletes only the selected duplicate records, never the recommended keep record implicitly.
- The operation should be atomic where practical, or return a partial-failure result with deleted and failed IDs.

Response:

```json
{
  "scanId": "scan-2026-09-06T12:00:00.000Z",
  "groupId": "sha256-group-id",
  "keptResumeId": "691db80aa895776f97b6eca6",
  "deletedResumeIds": ["691db80aa895776f97b6eca7"],
  "deletedCount": 1,
  "remainingGroupSize": 1
}
```

### 7.3 Deduplication errors

| Status | `errorCode` | Meaning |
|---|---|---|
| 400 | `INVALID_DEDUPLICATION_REQUEST` | Malformed scan or removal payload. |
| 400 | `CONFIRMATION_REQUIRED` | Removal was not explicitly confirmed. |
| 404 | `DEDUPLICATION_GROUP_NOT_FOUND` | Scan or group no longer exists. |
| 409 | `DEDUPLICATION_STATE_CHANGED` | Records changed or are no longer duplicates. |
| 422 | `INVALID_DUPLICATE_SELECTION` | Keep/remove IDs overlap or are not in the group. |
| 503 | `DEDUPLICATION_UNAVAILABLE` | MongoDB scan or delete operation is unavailable. |

All errors should include `success: false`, `requestId`, `errorCode`, and a user-safe `message`. Do not return database driver messages to the browser.

## 8. Phase-by-Phase Implementation Plan

# PHASE 1 — Contract and Readiness Verification

## Objective

Lock the implementation to the existing backend contracts and verify that the current retrieval flow is healthy before adding UI or destructive operations.

## Scope

- Confirm `/v1/search/rerank` and `/v1/search/summarize` behavior from controller tests.
- Confirm `/v1/search` already performs merge, rerank, fallback, and optional summary work.
- Confirm current frontend API client, sidebar, store, and result types.
- Add no duplicate LLM or search logic.

## Backend changes

- No behavioral change expected.
- Add or update contract tests only if a missing boundary is found.

## Frontend changes

- No production UI change in this phase.
- Record the existing component paths and current error/toast behavior.

## API changes

- None.

## Components/modules involved

- `retrievalController.ts`
- `LLMService.ts`
- `SearchService.ts`
- `retrievalRoutes.ts`
- `search.api.ts`
- `search.types.ts`
- `Sidebar.tsx`

## Data flow

Use a known candidate from existing search results as the input to direct rerank or summarize calls. The frontend must not reconstruct a candidate from arbitrary profile text when a backend result already contains the required `snippet`.

## Dependencies

- Existing backend and frontend dependencies only.
- Groq and MongoDB remain backend-only dependencies.

## Validation and error handling

- Run backend tests for rerank and summarize validation.
- Verify error payloads retain `errorCode` and `requestId`.

## Testing strategy

- Existing `rerank-endpoint.test.ts` and `summarize-endpoint.test.ts` remain green.
- Add frontend API type tests only if the project introduces a frontend test runner.

## Integration testing

- Manually submit one existing candidate from the search UI to each direct endpoint using a local backend.

## Acceptance criteria

- Existing search, ingestion, rerank, and summarize endpoints remain unchanged and passing.
- Every field in the frontend plan maps to a field accepted by the backend.

## Implementation checklist

- [ ] Verify current endpoint responses.
- [ ] Verify frontend route and sidebar extension points.
- [ ] Capture environment limits from `config/env.ts`.
- [ ] Confirm no direct Groq calls exist in frontend.

## Deployment considerations

- No new environment variables.
- Existing Groq and MongoDB credentials remain server-side.

# PHASE 2 — Reranking Frontend Integration

## Objective

Expose the existing reranking capability in the recruiter UI without duplicating backend retrieval or LLM logic.

## Scope

Add a sidebar section titled `Reranking` with the exact subtitle:

> Combines BM25 + Vector search for intelligent semantic reranking.

The section should allow a recruiter to inspect or rerank candidates already produced by the current search workflow.

## Backend changes

- No reranking algorithm changes.
- Do not add a second reranking route.
- Keep the current `LLMService.rerankCandidates` validation and Groq model configuration.

## Frontend changes

Recommended new files:

```text
frontend/src/features/reranking/
├── components/RerankingPanel.tsx
├── services/rerankingApi.ts
├── stores/reranking.store.ts
└── types/reranking.types.ts
```

Extend existing `Sidebar.tsx` and `MobileDrawer.tsx` navigation consistently. Reuse current `Button`, `Badge`, `Card`, `Textarea`, loading, empty-state, and toast patterns. Use Lucide icons already used by the application.

Required controls:

- Read-only or editable query field populated from the current search query.
- Candidate selection based on current `SearchResponse.results`.
- Optional candidate fields sent only when present: `name`, `role`, `company`, `skills`, `snippet`.
- `topK` control constrained to 1 through 100, with a practical UI default of the current result limit.
- Primary `Rerank candidates` action.
- Loading, empty, success, validation-error, and backend-error states.

Do not expose BM25/vector weights as rerank request fields; those are not accepted by `/v1/search/rerank`. Existing hybrid weight controls remain associated with the search experience only.

## API changes

Add a typed method in `frontend/src/features/reranking/services/rerankingApi.ts` or the existing API layer:

```ts
rerankCandidates(request: RerankRequest): Promise<RerankResponse>
```

The method calls `POST /v1/search/rerank` through `apiClient`. Define types matching the contract exactly; do not use `any` for response parsing.

## Components/modules involved

- Existing: `search.api.ts`, `search.types.ts`, `use-search.ts`, `search.store.ts`, `Sidebar.tsx`, `MobileDrawer.tsx`.
- New: reranking API method, request/response types, panel, and local/store state only if state must persist across navigation.
- Existing backend: `retrievalController.ts`, `LLMService.ts`.

## Data flow

```text
Current search query + selected result cards
        ↓
RerankingPanel validates query, candidate IDs, snippets, and topK
        ↓
apiClient → POST /v1/search/rerank
        ↓
retrievalController validation
        ↓
LLMService.rerankCandidates → Groq
        ↓
validated results + model
        ↓
panel orders candidates by rank and displays score/reason
```

## Request/response contracts

Use the exact contracts in section 4. The frontend should map `candidateId` to backend `resumeId` only at the API boundary if an existing UI type uses a different name. Candidate objects sent to the API must include `resumeId`, not `candidateId`.

## Dependencies

- Existing Axios client and request ID interceptor.
- Existing React, Zustand, TailwindCSS, Framer Motion, Lucide React, and toast dependencies.

## Validation and error handling

- Block submission when query is blank, no candidates are selected, candidate IDs are missing, or `topK` is outside 1 through 100.
- Display backend `message` for 400/413 errors where safe.
- Display a retryable generic message for 502/network errors.
- Keep previous search results visible while reranking is in progress.
- Never display an LLM result for an ID that was not submitted.

## Testing strategy

- Unit-test request mapping, especially `candidateId` to `resumeId`, optional fields, and `topK`.
- Component-test empty, loading, success, and error states.
- Assert the exact subtitle and sidebar navigation label.

## Integration testing

- Search for a known resume.
- Open Reranking.
- Submit the current query and at least one candidate.
- Verify returned `rank`, `relevanceScore`, `reason`, and `model` render without changing the search result contract.

## Acceptance criteria

- Reranking is reachable from desktop and mobile sidebars.
- The exact required subtitle is present.
- The browser calls only `/v1/search/rerank` for the direct action.
- No Groq key or prompt is shipped to the frontend.
- Validation and backend errors are understandable and non-destructive.

## Implementation checklist

- [ ] Add typed reranking API method.
- [ ] Add request/response types.
- [ ] Build panel and states.
- [ ] Add desktop/mobile sidebar entry.
- [ ] Reuse current search result candidate data.
- [ ] Add unit/component tests.
- [ ] Verify local end-to-end flow.

## Deployment considerations

- No backend deployment change beyond serving the existing route.
- Ensure frontend API base URL targets the same backend environment.
- Treat `model` as informational; do not hard-code it in the frontend.

# PHASE 3 — Summarization Frontend Integration

## Objective

Expose the existing candidate-fit summarization operation from the frontend using the existing backend contract.

## Scope

Add a sidebar section titled `Summarization` with the exact subtitle:

> Generate AI summaries.

The UI should summarize one selected search candidate in the context of the current query.

## Backend changes

- No new summarization service or prompt.
- Keep `LLMService.summarizeCandidateFit` and `/v1/search/summarize` as the source of truth.
- Preserve `style` and `maxTokens` validation.

## Frontend changes

Recommended new files:

```text
frontend/src/features/summarization/
├── components/SummarizationPanel.tsx
├── services/summarizationApi.ts
├── stores/summarization.store.ts
└── types/summarization.types.ts
```

Required fields and controls:

- Query text, defaulting to the current search query.
- Candidate selector from current results.
- Candidate data: `resumeId` and non-empty `snippet` are mandatory; `name`, `role`, `company`, and `skills` are optional.
- `style` select with exactly `short` and `detailed`.
- `maxTokens` numeric control constrained to 1 through 2,000, default 150.
- `Generate summary` action.
- Summary output associated with the selected `resumeId`.

Do not add fields such as temperature, model, system prompt, or arbitrary resume text limits to the frontend contract; the current backend does not accept them.

## API changes

Add a typed method that calls `POST /v1/search/summarize` through `apiClient` and returns `{ resumeId, summary }`.

## Components/modules involved

- Existing result cards and candidate modal can provide candidate context.
- New summarization panel, API method, and types.
- Existing `Sidebar.tsx`, `MobileDrawer.tsx`, `ChatPage.tsx`, `search.store.ts`, and `use-search.ts`.
- Existing backend `retrievalController.ts` and `LLMService.ts`.

## Data flow

```text
Current query + selected result with snippet
        ↓
SummarizationPanel validates query, resumeId, snippet, style, maxTokens
        ↓
apiClient → POST /v1/search/summarize
        ↓
retrievalController validation
        ↓
LLMService.summarizeCandidateFit → Groq
        ↓
{ resumeId, summary }
        ↓
summary displayed beside the matching candidate
```

## Request/response contracts

Use section 5 exactly. The summary must be associated by `resumeId`, not by array position or candidate name.

## Dependencies

- Existing API client, toast handling, form controls, and result state.

## Validation and error handling

- Disable submission without a selected candidate or non-empty snippet.
- Constrain style and token count in the UI and rely on backend validation as the final authority.
- Preserve the selected candidate and query if generation fails so retry is straightforward.
- Never treat an LLM summary as a replacement for the source resume profile.

## Testing strategy

- Unit-test payload mapping and defaults.
- Component-test style selection, max-token bounds, loading, empty, success, 400, 413, 502, and network states.
- Verify summaries remain attached to the correct resume ID when the result list changes.

## Integration testing

- Search for a known candidate with a non-empty `snippet`.
- Generate both short and detailed summaries.
- Verify the response appears on the selected candidate and no other card.

## Acceptance criteria

- Summarization is reachable from desktop and mobile sidebars.
- The exact required subtitle is present.
- Only `/v1/search/summarize` is called for the direct action.
- `style` and `maxTokens` match the backend contract.
- The UI handles unavailable Groq service without losing search results.

## Implementation checklist

- [ ] Add typed summarization API method.
- [ ] Add candidate, request, and response types.
- [ ] Build panel and result state.
- [ ] Add desktop/mobile sidebar entry.
- [ ] Add tests for style and token validation.
- [ ] Verify short and detailed flows.

## Deployment considerations

- No Groq credentials are added to frontend deployment.
- Confirm production frontend timeout is sufficient for the backend LLM request.
- Keep summary generation optional in end-to-end search to control latency and cost.

# PHASE 4 — Deduplication Backend Foundation

## Objective

Implement a conservative, explainable resume-level deduplication service using the existing MongoDB `resumes` collection.

## Scope

- Add normalization and fingerprint utilities.
- Add repository scan and delete methods.
- Add service-level grouping, confidence, recommendation, and revalidation.
- Keep search-result merge deduplication unchanged.

## Backend changes

Create the new components from section 6.4. `DeduplicationRepository` should:

- Read only the required fields for scanning.
- Fetch records by validated `ObjectId`.
- Delete only explicitly selected IDs after service validation.
- Avoid returning embeddings and raw text in API DTOs.

`DeduplicationService` should:

- Normalize fields deterministically.
- Build exact identity indexes and fingerprints.
- Produce stable groups and reasons.
- Recommend a keep record without mutating data.
- Revalidate a group immediately before deletion.
- Return structured counts and warnings.

## Frontend changes

- None in this phase.

## API changes

- Add `POST /v1/deduplication/scan`.
- Add `POST /v1/deduplication/remove`.
- Mount routes under `/v1` in `app.ts`.

## Components/modules involved

- New deduplication module.
- Existing `config/database.ts`, `middleware/requestId.ts`, `middleware/logger.ts`, `middleware/errorHandler.ts`.
- Existing `ResumeDocument` shape, extended only where actual ingestion metadata exists.

## Data flow

```text
scan request → controller validation → service reads resumes
             → normalize → fingerprint/index → group matches
             → response without raw text or embeddings

remove request → controller validation → service re-reads group
               → revalidates keep/remove selection and duplicate status
               → repository deletes selected IDs → audit response
```

## Request/response contracts

Use section 7. The scan response is read-only. The remove response must report exactly what was deleted.

## Dependencies

- Node `crypto` for a stable SHA-256 fingerprint.
- Existing MongoDB driver and ObjectId validation.
- No new external similarity or LLM dependency for the exact first version.

## Validation and error handling

- Reject malformed ObjectIds and unsupported modes with 400.
- Treat an empty collection as a successful scan with zero groups.
- Return 503 for database availability failures.
- Return 409 when the selected group changed between scan and removal.
- Log scan counts and deletion counts with request ID; do not log raw resume text or sensitive contact values.

## Testing strategy

- Unit-test whitespace/case/phone normalization.
- Unit-test identical fingerprint grouping.
- Unit-test email, phone-plus-name, and possible-match rules.
- Unit-test stable group IDs and keep recommendations.
- Unit-test that embeddings and raw text are excluded from DTOs.
- Unit-test invalid selection, confirmation, stale scan, and delete failure paths.

## Integration testing

- Use a test database or repository mock with two identical and one distinct resume.
- Scan, assert one group, remove one selected duplicate, and assert the keeper remains.
- Assert a stale scan cannot delete a changed record.

## Acceptance criteria

- Scan is independently executable before frontend work.
- Exact duplicate identification is deterministic and explainable.
- No record is removed without explicit confirmation and selected IDs.
- Existing ingestion and retrieval tests remain green.

## Implementation checklist

- [ ] Add types and DTOs.
- [ ] Add normalization and fingerprint functions.
- [ ] Add repository scan/delete operations.
- [ ] Add service grouping and stale-scan checks.
- [ ] Add controller validation and safe errors.
- [ ] Register routes.
- [ ] Add unit and integration tests.
- [ ] Exercise endpoints with curl or Supertest.

## Deployment considerations

- Take a MongoDB backup before enabling deletion in a shared environment.
- Initially deploy scan-only or keep removal behind an environment feature flag if operational review is required.
- Monitor scan duration and document-count limits; paginate or batch scans if the collection grows.
- Ensure application credentials have the minimum read/delete permissions required.

# PHASE 5 — Deduplication API Validation Gate

## Objective

Prove that deduplication is safe and independently testable before building its UI.

## Scope

- Contract tests for scan and remove endpoints.
- Repeatable fixture data.
- Negative and concurrency tests.

## Backend changes

- Add Supertest endpoint tests alongside existing backend tests.
- Add a repository test double or test database strategy that does not touch production data.

## Frontend changes

- None.

## API changes

- Freeze status codes, error codes, and response fields from section 7 before frontend implementation.

## Components/modules involved

- Deduplication route, controller, service, repository, and test fixtures.

## Data flow

```text
fixture resumes → POST scan → inspect groups → POST remove with confirm:true
                 → verify deletion result → GET/search verifies keeper remains
```

## Dependencies

- Existing Jest and Supertest setup.
- Isolated MongoDB test environment or mocked repository.

## Validation and error handling

Test at minimum:

- Missing request body.
- Unsupported `matchMode`.
- Empty `removeResumeIds`.
- `confirm: false`.
- Keep ID included in remove IDs.
- Unknown group and stale scan.
- Duplicate deletion request.
- Database failure.

## Testing strategy

- Run `npm test -- --runInBand` in `backend`.
- Add assertions for no accidental deletion on every rejected request.

## Integration testing

- Run scan and removal against a disposable test collection or an isolated database.
- Verify the retrieval readiness and search endpoints still see the remaining keeper.

## Acceptance criteria

- The API can be validated without the frontend.
- Removal is idempotently rejected or reports no-op behavior when the selected record is already gone.
- The contract is stable enough for typed frontend clients.

## Implementation checklist

- [ ] Add endpoint tests.
- [ ] Add fixture setup/cleanup.
- [ ] Verify success and all documented errors.
- [ ] Run the full backend test suite.

## Deployment considerations

- Do not run destructive tests against production MongoDB.
- Capture request IDs and deletion counts in deployment verification logs.

# PHASE 6 — Deduplication Frontend Integration

## Objective

Provide a safe recruiter-facing scan, review, and removal workflow.

## Scope

Add a sidebar section titled `Deduplication` with the exact subtitle:

> Search and remove the duplicate resumes.

The workflow must make scan results reviewable and deletion explicit.

## Backend changes

- No algorithm changes expected after the API gate.
- Add only fixes required by frontend contract tests.

## Frontend changes

Recommended new files:

```text
frontend/src/features/deduplication/
├── components/DeduplicationPanel.tsx
├── components/DuplicateGroup.tsx
├── components/DuplicateRecord.tsx
├── services/deduplicationApi.ts
├── stores/deduplication.store.ts
└── types/deduplication.types.ts
```

Add the sidebar section to `Sidebar.tsx` and its mobile equivalent. Use the existing dialog component for removal confirmation. Do not put a destructive action in a generic text button without a confirmation dialog and clear selected IDs.

Required controls and fields:

- `Scan duplicates` action.
- Optional exact-match mode control, disabled for unsupported modes.
- `includePossible` toggle, default off.
- Group list showing confidence, match reasons, recommended keeper, and records.
- Per-record selection control for records to remove.
- `keepResumeId` selection or recommendation override.
- Explicit `Remove selected duplicates` action.
- Confirmation dialog showing group ID, keeper, selected IDs/count, and irreversible-action warning.
- Refresh/re-scan action after removal.

Display rules:

- Show groups, not raw fingerprints as the primary UI.
- Show name, email, role, experience, and resume ID when present.
- Mark the recommended keeper visually but allow override.
- Display exact versus possible confidence.
- Never show embeddings or raw full resume text in the deduplication panel.
- Remove deleted records from local state only after a successful response.

## API changes

Add typed methods:

```ts
scanDuplicates(request: DeduplicationScanRequest): Promise<DeduplicationScanResponse>
removeDuplicates(request: DeduplicationRemoveRequest): Promise<DeduplicationRemoveResponse>
```

Both use the existing `apiClient`, inherit request IDs, and surface backend errors through the existing error handling convention.

## Components/modules involved

- Existing: `Sidebar.tsx`, `MobileDrawer.tsx`, `Dialog`, `Button`, `Badge`, `Card`, `EmptyState`, `StatusDot`, toast handling.
- New: deduplication API/types/store/panel/group/record components.
- Backend: deduplication routes and service.

## Data flow

```text
Deduplication sidebar section
        ↓ Scan duplicates
apiClient → POST /v1/deduplication/scan
        ↓
groups rendered and selected locally
        ↓ Remove selected duplicates
confirmation dialog → POST /v1/deduplication/remove
        ↓
backend revalidates group and deletes selected IDs
        ↓
success response → remove records/update group/announce result
```

## Request/response contracts

Use section 7 exactly. The frontend must send the `scanId` returned by the scan and the selected `keepResumeId`/`removeResumeIds`; it must not calculate fingerprints or group IDs itself.

## Dependencies

- Existing React, Zustand, Axios, TailwindCSS, Lucide, and dialog primitives.
- No frontend hashing or MongoDB dependency.

## Validation and error handling

- Disable removal until at least one record is selected and a keeper is selected.
- Prevent keeper/removal overlap in client state, while retaining server validation.
- Handle `409 DEDUPLICATION_STATE_CHANGED` by preserving the user’s choices, showing a stale-results message, and requiring a fresh scan.
- Handle 503/network errors without clearing scan results.
- After success, update the group using `deletedResumeIds`; do not infer deletion from the requested list.
- Announce scan and removal status accessibly and maintain keyboard focus in the confirmation dialog.

## Testing strategy

- Unit-test API payloads and selection invariants.
- Component-test empty scan, exact groups, possible groups, keeper override, confirmation, success, stale scan, and network errors.
- Test desktop and mobile sidebar visibility.
- Test that raw text and embeddings are never rendered.

## Integration testing

- Start backend and frontend.
- Scan a fixture dataset containing duplicates.
- Review one group, override the recommended keeper, remove one record, and re-scan.
- Verify the result disappears from subsequent search results while the keeper remains.

## Acceptance criteria

- The exact required subtitle is present.
- Users can scan without deleting anything.
- Users must confirm selected deletions.
- The UI displays match reasons and confidence.
- Successful deletion updates the UI only from the API response.
- Stale scans are blocked safely.

## Implementation checklist

- [ ] Add typed scan/remove API methods.
- [ ] Add deduplication types and store.
- [ ] Build group and record views.
- [ ] Add confirmation dialog.
- [ ] Add desktop/mobile sidebar entry.
- [ ] Add loading, empty, success, stale, and error states.
- [ ] Add component/API tests.
- [ ] Run a full local scan/remove/re-scan flow.

## Deployment considerations

- Use feature flags or role-based access if deletion should initially be restricted.
- Keep API and frontend versions compatible because removal depends on `scanId` and `groupId`.
- Provide an operational audit trail containing request ID, actor if available, group ID, keeper ID, and deleted IDs, without logging resume contents.

# PHASE 7 — End-to-End Retrieval Integration

## Objective

Verify that reranking, summarization, and deduplication fit the existing ingestion-to-retrieval workflow without degrading normal search.

## Scope

- Search after ingestion.
- Direct rerank and summary from search results.
- Deduplication scan/removal followed by search.
- Failure and fallback behavior.

## Backend changes

- Confirm `/v1/search` continues to use `SearchService.endToEndSearch`.
- Confirm BM25 and vector failures still degrade independently.
- Confirm rerank failure falls back to retrieval order and emits `LLM_RERANK_FAILED`.
- Confirm optional summary failure emits `SUMMARIZATION_FAILED` without discarding ranked results.
- Confirm deleting a duplicate does not delete the recommended keeper.

## Frontend changes

- Preserve existing result cards and candidate modal.
- Render rerank results as an additional ranked view or update an explicit reranking result state; do not silently overwrite the original search response.
- Render summaries per `resumeId`.
- Refresh or invalidate search results after successful duplicate removal.

## API changes

- No new search contract.
- Deduplication endpoints remain separate from search to keep responsibilities clear.

## Components/modules involved

- `SearchService`, `LLMService`, `ResumeRepository`, deduplication service.
- `use-search.ts`, search store, new feature stores and panels.
- Existing result and candidate components.

## Data flow

```text
ingest resume → readiness → search
    → BM25 + vector → merge by resumeId → rerank → optional summaries
    → display results

dedup scan/remove → search cache invalidation or fresh search
    → deleted duplicate absent, keeper still searchable
```

## Request/response contracts

- Existing `/v1/search` response remains `{ query, results, degraded, warnings, timings }`.
- Existing direct rerank and summarize contracts remain sections 4 and 5.
- New deduplication contracts remain section 7.

## Dependencies

- MongoDB Atlas Search and vector indexes.
- Mistral embeddings.
- Groq credentials for reranking/summarization.
- Running backend and frontend with matching base URLs.

## Validation and error handling

- Verify degraded search states are visible but usable.
- Verify direct LLM actions do not remove or mutate resumes.
- Verify removal is the only operation that mutates resume records.
- Verify request IDs appear in logs and are available for support diagnostics.

## Testing strategy

Backend:

- Run all existing Jest tests.
- Add deduplication unit, endpoint, repository, and service tests.
- Exercise search fallback and optional summary tests.

Frontend:

- Run the frontend TypeScript build.
- Add component/API tests when the frontend test runner is introduced.

## Integration testing

- Upload or use an existing PDF through `/ingestion`.
- Verify it becomes searchable.
- Rerank returned candidates.
- Summarize one candidate in both styles.
- Scan and remove a known duplicate.
- Search again and verify the expected record set.

## Acceptance criteria

- Existing ingestion and search workflows are unchanged for users who do not use the new sections.
- All three capabilities are reachable from desktop and mobile navigation.
- Backend failures are represented by stable error states rather than blank screens.
- No frontend code contains Groq prompts, MongoDB access, or duplicate matching logic.

## Implementation checklist

- [ ] Run backend build and tests.
- [ ] Run frontend build.
- [ ] Run API contract checks.
- [ ] Test desktop and mobile navigation.
- [ ] Test LLM failure fallbacks.
- [ ] Test stale deduplication removal.
- [ ] Verify post-delete search results.

## Deployment considerations

- Deploy backend contract changes before enabling frontend deduplication controls.
- Keep production deletion disabled until scan accuracy and audit logging are reviewed.
- Configure frontend timeout and backend request limits for LLM actions.
- Monitor total retrieval time, rerank time, summary time, scan time, and deletion failures.

# 9. Validation and Delivery Gates

The implementation is complete only when these gates pass in order:

1. **Existing behavior gate:** backend existing tests pass; frontend TypeScript build passes.
2. **Reranking gate:** direct frontend rerank request produces ordered results with rank, score, and reason.
3. **Summarization gate:** short and detailed summaries render against the correct `resumeId`.
4. **Deduplication scan gate:** fixture duplicates produce deterministic groups without mutation.
5. **Deduplication removal gate:** explicit confirmation deletes only selected records and rejects stale or unsafe selections.
6. **Workflow gate:** ingestion → readiness → search → rerank/summary → dedup scan/remove → search again works end to end.
7. **Responsive/accessibility gate:** sections work in the desktop sidebar and mobile drawer, controls are keyboard accessible, and destructive actions are clearly confirmed.

## Suggested commands

```bash
# Backend
cd backend
npm run build
npm test -- --runInBand

# Frontend
cd ../frontend
npm run build
```

The architecture is intentionally implementation-ready while preserving the current backend contracts. Any new field, endpoint, similarity rule, database index, or dependency introduced during coding must be added to this document and covered by a contract or integration test before release.
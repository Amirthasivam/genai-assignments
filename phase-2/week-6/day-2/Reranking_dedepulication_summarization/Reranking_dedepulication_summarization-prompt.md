Take `02_Resume_Retrieval_Phasewise_Implementation.md` and `FRONTEND_WEB_INSTRUCTIONS_FINAL_V1 3.md` as reference documents, review their structure and implementation guidelines, and create a new file named `Reranking-deduplication-summarization.architecture.md` in the project folder.

The architecture document must provide a clear, detailed, and phase-by-phase implementation plan for the current project.

### 1. Reranking — Frontend Integration

- Review the existing Backend Reranking implementation and API contract.
- Integrate the existing Reranking functionality into the Frontend without duplicating backend logic.
- Add a `Reranking` section to the Frontend sidebar.
- Use the following subtitle:
`Combines BM25 + Vector search for intelligent semantic reranking.`
- Define the required Frontend fields, inputs, controls, and actions based strictly on the existing Backend Reranking API request/response fields.
- Document the Frontend → API → Backend flow and expected response handling.

### 2. Summarization — Frontend Integration

- Review the existing Backend Summarization implementation and API contract.
- Integrate the existing Summarization functionality into the Frontend.
- Add a `Summarization` section to the Frontend sidebar.
- Use the following subtitle:
`Generate AI summaries.`
- Define the required Frontend fields, inputs, controls, and actions based strictly on the existing Backend Summarization API request/response fields.
- Document the Frontend → API → Backend flow and expected response handling.

### 3. Deduplication — Backend Implementation

- Design and implement a new Deduplication functionality in the Backend for identifying and removing duplicate resumes.
- Reuse existing project architecture, services, utilities, models, and patterns wherever applicable.
- Define the deduplication approach, matching logic, inputs, outputs, validation rules, and error handling.
- Clearly distinguish newly introduced components from existing reusable components.

### 4. Deduplication API

- Add a dedicated Backend API endpoint for Deduplication validation/testing.
- Define the endpoint, HTTP method, request schema, response schema, validation rules, error responses, and processing flow.
- Ensure the API can be independently tested before Frontend integration.
- Document example request/response flows where appropriate.

### 5. Deduplication — Frontend Integration

- Integrate the new Deduplication API and functionality into the Frontend.
- Add a `Deduplication` section to the Frontend sidebar.
- Use the following subtitle:
`Search and remove the duplicate resumes.`
- Define the required Frontend fields, inputs, controls, and actions based on the Deduplication API contract.
- Document how duplicate results are displayed, validated, and removed.
- Document the complete Frontend → API → Backend data flow.

### 6. Phase-by-Phase Architecture Plan
Follow the structure, organization, conventions, and level of detail of `02_Resume_Retrieval_Phasewise_Implementation.md` while adhering to the Frontend guidelines in `FRONTEND_WEB_INSTRUCTIONS_FINAL_V1 3.md`.

Break the implementation into logical phases. For each phase, include:

- Objective
- Scope
- Backend changes
- Frontend changes
- API changes
- Components/modules involved
- Data flow
- Request/response contracts
- Dependencies
- Validation and error handling
- Testing strategy
- Integration testing
- Acceptance criteria
- Implementation checklist
- Deployment considerations
Also include an overall architecture/data-flow overview showing how Reranking, Summarization, and Deduplication fit into the existing Resume Retrieval workflow.

Before finalizing the architecture file, inspect the existing project structure and Backend APIs so that all referenced fields, endpoints, components, and dependencies are based on the actual current implementation rather than assumptions. Clearly mark anything that requires a new implementation.
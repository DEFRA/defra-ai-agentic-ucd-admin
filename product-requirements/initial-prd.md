# **Research Analysis Backend – Product Requirements & User Stories**

_(Backend-only, API & persistence layer — **no authentication or paging**)_

---

## **Context**

A research-operations team needs an automated backend that ingests raw user-research interview transcripts, runs an asynchronous LangGraph agentic workflow, and exposes the generated affinity map and research-findings report through a REST API. The backend must (1) durably store all artefacts, (2) expose a minimal **unauthenticated** public surface, and (3) maintain clear coarse- and fine-grained status so that a frontend or other services can poll for progress.

---

## **Data Model (v2 — reference)**

### `research_analysis`

|Field|Type|Notes|
|---|---|---|
|`_id`|ObjectId|PK|
|`created_at`|Date|Session creation time|
|`status`|String|`INIT` \| `FILES_UPLOADED` \| `RUNNING` \| `COMPLETED` \| `ERROR`|
|`error_message`|String \| null|Coarse-level error description|
|`agent_state`|Object \| null|See sub-document|

#### `agent_state` sub-document

|Field|Type|Notes|
|---|---|---|
|`process_start_date`|Date|Filled by status update to `RUNNING`|
|`transcripts`|Array<String>|Raw markdown text|
|`transcripts_pii_cleaned`|Array<String>|PII-scrubbed text|
|`affinity_map`|String|Markdown|
|`findings_report`|String|Markdown|
|`status`|String|Fine-grained LangGraph status|
|`error_message`|String \| null|Fine-grained error|

---

### `analysis_file`

|Field|Type|Notes|
|---|---|---|
|`_id`|ObjectId|PK|
|`analysis_id`|ObjectId|FK → `research_analysis._id`|
|`s3_key`|String|Full S3 object key|
|`uploaded_at`|Date|Timestamp|

> **Indexes**  
> • `research_analysis.status`  
> • `analysis_file.analysis_id`

---

## **Feature & Story Catalogue**

### **Feature 1 – Session Lifecycle Management**

|#|Story Title|
|---|---|
|**1.1**|Create Analysis Session|
|**1.2**|List Analysis Sessions|
|**1.3**|Retrieve Analysis Session (with agent state)|
|**1.4**|Update Analysis Session Status|
|**1.5**|Delete Analysis Session|

---

#### **Story 1.1 – Create Analysis Session**

- **As a** client application
- **I want** to create an empty research-analysis session
- **so that** transcripts can later be attached and processed.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|a request with an empty JSON body|the client `POST`s **/api/v1/research-analyses**|• API responds **201** with new `_id`, `created_at`, `status = INIT`, `agent_state = null`.• A `research_analysis` doc is inserted.• `created_at` uses server UTC clock (ISO-8601).|

##### Architecture Design Notes

- REST controller validates empty body.
- Service layer inserts document with preset fields (`status = INIT`).
- Mongo driver returns the generated `_id`.

---

#### **Story 1.2 – List Analysis Sessions**

- **As a** client
- **I want** to see all sessions with coarse status
- **so that** I can display a dashboard of in-progress and completed analyses.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|any number of `research_analysis` docs exist|the client `GET`s **/api/v1/research-analyses**|• API responds **200** with an array.• Each element contains `_id`, `created_at`, `status` only (no `agent_state`).• Result is sorted by `created_at` **descending**.• Entire collection is returned (no paging, no limit).|

##### Architecture Design Notes

- Collection scan with sort by `created_at` descending; no pagination.
- Projection excludes large `agent_state` field.
- Add index `created_at` for efficient sort.

---

#### **Story 1.3 – Retrieve Analysis Session (Detailed)**

- **As a** client
- **I want** to retrieve a session with full agent state
- **so that** I can render progress or final artefacts.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|a valid session `_id`|the client `GET`s **/api/v1/research-analyses/{id}**|• If found, **200** with full document including `agent_state`.• If not found, **404** with standard error object (`code = NOT_FOUND`).|

##### Architecture Design Notes

- `findOne` by `_id`; no projection.
- Large markdown strings served directly.

---

#### **Story 1.4 – Update Analysis Session Status**

- **As a** client
- **I want** to update a session's status to trigger workflow execution
- **so that** the system can start processing uploaded transcripts.

##### Acceptance Criteria

| Given                                                                      | When                                                    | Then                                                                                                                                                                                                                                                                                                             |
| -------------------------------------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| session `status = FILES_UPLOADED` and request body `{"status": "RUNNING"}` | the client `PATCH`es **/api/v1/research-analyses/{id}** | • Server responds **200** with updated session object.• `status` field updated to `RUNNING`.• `process_start_date` set (UTC) in `agent_state`.• Background job queued.• Initial `agent_state.status = STARTING`.• Operation is **idempotent** - repeated PATCH requests with same status maintain current state. |
| session `status = RUNNING` and request body `{"status": "RUNNING"}`        | the client `PATCH`es **/api/v1/research-analyses/{id}** | • Server responds **200** with current session object.• No state changes occur (idempotent behavior).• No additional background jobs are queued.                                                                                                                                                                 |
| invalid `status` value provided                                            | the client `PATCH`es **/api/v1/research-analyses/{id}** | • Server responds **400** with error object (`code = INVALID_STATUS`).                                                                                                                                                                                                                                           |

##### Architecture Design Notes

- Validates status transition rules (e.g., can only move to `RUNNING` from `FILES_UPLOADED` or stay at `RUNNING`).
- Idempotent operation - duplicate requests with same status are safe.
- Only queues background job on actual state transition.

---

#### **Story 1.5 – Delete Analysis Session**

- **As a** client
- **I want** to permanently delete a session and its transcript files
- **so that** we can remove sensitive data on demand.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|a valid session `_id`|the client sends `DELETE /api/v1/research-analyses/{id}`|• All related `analysis_file` docs are removed.• Corresponding S3 objects are deleted.• `research_analysis` doc is deleted.• Returns **204 No Content**.|

##### Architecture Design Notes

- Two-phase delete: remove S3 objects, then delete Mongo documents (use transaction for docs).
- No role checks — endpoint is public (trade-off accepted for v1 prototype).

---

### **Feature 2 – Transcript File Handling**

|#|Story Title|
|---|---|
|**2.1**|Upload Transcripts to Session|
|**2.2**|List Transcripts for Session|

---

#### **Story 2.1 – Upload Transcripts to Session**

- **As a** client
- **I want** to upload one or more `.md` / `.txt` transcripts
- **so that** the analysis workflow can read them.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|session `status ∈ {INIT, FILES_UPLOADED}`|client `POST`s **multipart/form-data** to **/api/v1/research-analyses/{id}/transcripts**|• Each part has `Content-Type` `text/markdown` or `text/plain` and filename ends `.md`/`.txt`.• Invalid files → **400** (`code = UNSUPPORTED_FILE_TYPE`).• For each valid file:  – Streamed to S3 under `research/{analysis_id}/`.  – `analysis_file` doc written.• Response **201** with array of new `analysis_file` objects.• Parent `status` becomes `FILES_UPLOADED` if it was `INIT`.|

##### Architecture Design Notes

- Streaming upload to S3.
- S3 key pattern: `research/{analysis_id}/{uuid}-{safeFilename}`.
- Bulk insert `analysis_file` docs; single `$set` to update parent status.

---

#### **Story 2.2 – List Transcripts for Session**

- **As a** client
- **I want** to retrieve metadata for uploaded transcripts
- **so that** I can display what has been attached.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|a session exists|client `GET`s **/api/v1/research-analyses/{id}/transcripts**|• **200** with array of `analysis_file` docs (`_id`, `analysis_id`, `s3_key`, `uploaded_at`).• Empty array if none.|

##### Architecture Design Notes

- Query by indexed `analysis_id`.
- No file bodies are returned.

---

### **Feature 3 – Workflow Orchestration**

|#|Story Title|
|---|---|
|**3.1**|Status-based Workflow Triggering|
|**3.2**|Persist Fine-grained Agent State|
|**3.3**|Update Coarse Session Statuses|

---

#### **Story 3.1 – Status-based Workflow Triggering**

- **As a** client
- **I want** to trigger the agentic workflow by updating session status
- **so that** the system produces an affinity map and findings report.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|session `status = FILES_UPLOADED`|client `PATCH`es **/api/v1/research-analyses/{id}** with `{"status": "RUNNING"}`|• Server responds **200** with updated session object.• `status` field becomes `RUNNING`.• `process_start_date` set (UTC) in `agent_state`.• Background job queued.• Initial `agent_state.status = STARTING`.• Subsequent identical PATCH requests are idempotent and don't queue additional jobs.|

##### Architecture Design Notes

- Status update triggers async job (i.e. LangGraph).
- Idempotent operation prevents duplicate job queueing.
- Clear separation between coarse API status and fine-grained agent status.

---

### **Feature 4 – Error Response Standardisation**

Standardise error payloads across all endpoints.

|#|Story Title|
|---|---|
|**4.1**|Implement Consistent Error Object Shape|

---

#### **Story 4.1 – Implement Consistent Error Object Shape**

- **As an** API consumer
- **I want** every non-2xx response to follow the same JSON contract
- **so that** my client code can handle errors generically.

##### Acceptance Criteria

|Given|When|Then|
|---|---|---|
|an endpoint encounters validation or runtime error|response is returned|• HTTP status code reflects error class (400, 404, 409, 500).• Body matches:`{ "status": "ERROR", "error_message": "<text>", "code": "<MACHINE_CODE>" }`.• `code` is optional for 500s but present for deterministic errors (e.g., `UNSUPPORTED_FILE_TYPE`, `INVALID_STATUS`).|

##### Architecture Design Notes

- Centralised error-handling middleware converts thrown exceptions to standard object.
- Map common exceptions (`ValidationError`, `NotFound`, `Conflict`, etc.) to HTTP codes.
- Log full stack traces server-side; never expose them in `error_message`.

##### Dependencies

None; cross-cuts all features.

---

## Notes for Implementation Tools

- **Language & Framework**: Python (FastAPI) both fit; emphasise typed pydantic v2   schemas.
- **Authentication**: the api will not support authentication initially
- **Observability**: log coarse status transitions (`INIT` → …) to application logs

---

_End of product requirements & user stories._
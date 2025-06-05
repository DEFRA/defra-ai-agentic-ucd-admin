### Context

The **Research Transcript Analysis Workflow** provides an automated, agent-driven pipeline that ingests user-research interview transcripts, removes all personally-identifiable information (PII), and produces two Markdown outputs—an affinity map and a findings report.  
It is implemented in **LangGraph** and invoked by an existing **FastAPI** endpoint.  
All LLM calls are routed through **AWS Bedrock**.

---

### State Model (reference)

| Field                       | Type             | Notes                                           |
| --------------------------- | ---------------- | ----------------------------------------------- |
| **process_start_date**      | `Date`           | Set when status transitions to `RUNNING`.       |
| **transcripts**             | `Array\<String>` | Full raw transcript text (markdown).            |
| **transcripts_pii_cleaned** | `Array\<String>` | PII-scrubbed transcript text.                   |
| **affinity_map**            | `String`         | Markdown affinity-mapping output.               |
| **findings_report**         | `String`         | Markdown research-findings report.              |
| **status**                  | `String`         | Fine-grained LangGraph status (see list below). |
| **error_message**           | `String`         | Latest error, if any.                           |

**Allowed status values**

1. STARTING
2. LOADING_TRANSCRIPTS
3. REMOVING_PII
4. VALIDATING_PII
5. GENERATING_AFFINITY_MAP
6. GENERATING_FINDINGS
7. FINISHED

---

## Feature 1 – **Workflow Orchestration**

> Implements the sequential LangGraph flow that links all functional nodes and updates fine-grained status.

### 1.1 Story – Create LangGraph Graph

- **Story**: _As a backend developer, I want a LangGraph graph definition that wires together the five nodes in the specified order so that the workflow executes deterministically and surfaces status updates for the API layer._
    
- **Acceptance Criteria**
    
    - **Given** the workflow is instantiated
    - **When** the developer inspects the graph topology
    - **Then** exactly five nodes exist in order: `transcript-loader → remove-pii → validate-pii-removal → affinity-mapping → findings-report`.
    - **And** a single directed edge exists between each consecutive pair.
        
- **Architecture Notes**
    
    - Use `langgraph.Graph()`; register each node with its handler coroutine.
    - Configure a graph-level callback to persist `status` on node start and completion.
    - Hard-fail in `validate-pii-removal` returns `GraphExecutionError` that the driver converts to `status='VALIDATING_PII'` and records `error_message`.
        
- **Dependencies**: none.
    

### 1.2 Story – Status Lifecycle Management

- **Story**: _As a calling service, I want the workflow to expose granular status transitions so that I can display real-time progress to the requester._
    
- **Acceptance Criteria**
    
    - **Given** the graph is running
    - **When** each node begins, **Then** `status` is updated to its matching enum value (`LOADING_TRANSCRIPTS`, etc.).
    - **When** the last node completes without error, **Then** `status` is `FINISHED`.
    - **When** any node raises an unhandled exception, **Then** `status` is frozen at the in-progress value and `error_message` contains the exception string.
        
- **Architecture Notes**
    
    - Implement a small helper `set_status(state, enum)` mutating function.
    - Wrap each node in a try/except that sets `error_message` and re-raises to stop execution.

---

## Feature 2 – **transcript-loader Node**

### 2.1 Story – Load Transcripts From Session Storage

- **Story**: _As a workflow node, I want to iterate over the session’s uploaded transcript files, read their contents, and populate `state.transcripts` so that downstream tasks have full raw text._
    
- **Design Considerations**: None (pure backend).
    
- **Acceptance Criteria**
    
    - **Given** a research_analysis session ID containing one or more files in S3
    - **When** `transcript-loader` executes
    - **Then** each file’s full markdown contents are appended to `state.transcripts`.
    - **And** `state.transcripts` length equals the file count.
    - **And** `process_start_date` is set if not already present.
        
- **Architecture Notes**
    
    - Streaming read to avoid large memory spikes; store file key list in env / context.
    - Treat encoding as UTF-8; raise validation error on decode failure.
    - Node sets `status='LOADING_TRANSCRIPTS'` on start.
        

---

## Feature 3 – **remove-pii Node**

### 3.1 Story – PII Scrubbing Prompt

- **Story**: _As a workflow node, I want to send each transcript through an LLM prompt that removes names, emails, phone numbers, addresses, and any direct identifiers, so that the cleaned text can be shared safely._
    
- **Acceptance Criteria**
    
    - **Given** `state.transcripts` populated
    - **When** `remove-pii` runs
    - **Then** the Bedrock response for every transcript is stored in parallel index in `state.transcripts_pii_cleaned`.
    - **And** no identifier tokens from a configurable regex list exist in any cleaned transcript.
        
- **Architecture Notes**
    
    - Prompt template: “You are a redaction engine. Remove … Return only markdown text.”
    - Pass transcripts in batch or loop, depending on token length limits.
    - Node sets `status='REMOVING_PII'`.
        

---

## Feature 4 – **validate-pii-removal Node**

### 4.1 Story – Automatic PII Audit

- **Story**: _As a workflow node, I want to validate that the cleaned transcripts truly contain no PII, so that regulatory compliance is assured._
    
- **Acceptance Criteria**
    
    - **Given** `state.transcripts_pii_cleaned` populated
    - **When** validation passes
    - **Then** the workflow advances to the next node.
    - **When** validation fails on any transcript
    - **Then** an exception is raised, `status` remains `VALIDATING_PII`, and `error_message` mentions the offending transcript index.
        
- **Architecture Notes**
    
    - Implementation: deterministic regex + optional secondary Bedrock “PII detector” prompt.
    - Failure path stops graph (LangGraph hard fail).
    - Node sets `status='VALIDATING_PII'`.

---

## Feature 5 – **affinity-mapping Node**

### 5.1 Story – Generate Affinity Map Markdown

- **Story**: _As a workflow node, I want to create an affinity map summarising themes across cleaned transcripts so that researchers can quickly see clusters of insights._
    
- **Design/UX**: Output is Markdown; FastAPI already surfaces it.
    
- **Acceptance Criteria**
    
    - **Given** validated cleaned transcripts
    - **When** `affinity-mapping` runs
    - **Then** `state.affinity_map` contains non-empty Markdown with at least three top-level headings.
    
- **Architecture Notes**
    
    - Prompt instructs Bedrock to return heading-prefixed Markdown clusters.
    - Node sets `status='GENERATING_AFFINITY_MAP'`.


---

## Feature 6 – **findings-report Node**

### 6.1 Story – Compile Findings Report

- **Story**: _As a workflow node, I want to produce a concise findings report that references the affinity map and cleaned transcripts so that stakeholders receive actionable insights._
    
- **Acceptance Criteria**
    
    - **Given** `state.affinity_map` and cleaned transcripts
    - **When** `findings-report` runs
    - **Then** `state.findings_report` contains Markdown with an **Executive Summary**, **Key Findings**, and **Recommendations** sections.
    - **And** references at least 100% of the affinity-map clusters (string match on headings).
    - **Then** overall `status` is updated to `FINISHED` on success.
        
- **Architecture Notes**
    
    - Prompt instructs Bedrock to structure report with the three required sections.
    - Node sets `status='GENERATING_FINDINGS'` on start; final driver sets `FINISHED

---

## Feature 7 – **Error Handling & Propagation**

### 7.1 Story – Unified Error Surface

- **Story**: _As the API consumer, I want any node failure to bubble up a clear error message so that I can surface it in the UI without inspecting logs._
    
- **Acceptance Criteria**
    
    - **Given** any node raises
    - **When** the driver catches the exception
    - **Then** `state.error_message` stores the exception text.
    - **And** the FastAPI endpoint returns HTTP 500 with the same message in the body.
        
- **Architecture Notes**
    
    - FastAPI dependency injection: wrap `graph.invoke()` in `try/except LangGraphError`.
    - Ensure partial state is persisted for later forensic review.
        
- **Dependencies**: relies on Features 1–6.

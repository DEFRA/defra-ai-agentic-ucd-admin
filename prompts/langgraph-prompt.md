# CONTEXT

We're developing a backend API that processes user research transcripts and then runs an agentic workflow to analyze the transcripts and produce and affinity map and research findings report.

The LangGraph workflow will be called by an existing FastAPI endpoint, so we just need to remove the placeholder workflow with the correct LangGraph workflow as outlined below.

# ANALYSIS PHASE:

Look at the attached image of a LangGraph workflow.  Also look at the state model as defined below which will be used by LangGraph to maintain the state for the workflow

LangGraph State Object:

| Field                       | Type            | Notes                                                       |
| --------------------------- | --------------- | ----------------------------------------------------------- |
| **process_start_date**      | `Date`          | Set when status is updated to `RUNNING`.                    |
| **transcripts**             | `Array<String>` | **Full raw transcript text** (markdown).                    |
| **transcripts_pii_cleaned** | `Array<String>` | **PII‑scrubbed transcript text**.                           |
| **affinity_map**            | `String`        | Markdown content of the generated affinity map.             |
| **findings_report**         | `String`        | Markdown content of the generated research findings report. |
| **status**                  | `String`        | Fine‑grained LangGraph status.                              |
| **error_message**           | `String`        | Latest LangGraph error, if any.                             |

LangGraph Statuses:

1. **STARTING** - Initialize workflow state
2. **LOADING_TRANSCRIPTS** - Fetch files from S3
3. **REMOVING_PII** - Process transcripts to remove PII
4. **VALIDATING_PII** - Validate PII removal completion
5. **GENERATING_AFFINITY_MAP** - Create affinity map from transcripts
6. **GENERATING_FINDINGS** - Generate research findings report
7. **FINISHED** - Mark workflow as complete

Confirm you have read the content of the documents, then continue...

# IMPLEMENTATION PHASE:

We want to create product requirements for the LangGraph workflow.  The nodes should match the attached image, but is detailed below:

All LLM prompts below will use AWS Bedrock, as called by LangGraph.
## LangGraph Nodes (sequential):
- **transcript-loader** - this node will loop over all the files uploaded to the existing research_analysis session, extract the contents, then load them into the `transcripts` state object.
- **remove-pii** - this node will run an LLM prompt that will look at the contents of the `transcripts` state object and remove any identifiable information.  This will save the cleaned transcript contents into the `transcripts_pii_cleaned` state object.
- **validate-pii-removal** - this node will validate that all the identifiable data and PII has been removed from the `transcripts` state object.  If not, then the whole workflow will hard-fail and error out.  If the data was successfully removed, it will carry on to the next node
- **affinity-mapping** - this node will run an LLM prompt to generate a user research affinity map based on the data in the `transcripts_pii_cleaned` contents and save the affinity map text as markdown format in the `affinity_map` state object.
- **findings-report** - this node will run an LLM prompt to generate a user research findings report based on the data in the `transcripts_pii_cleaned` contents and  `affinity_map` state object, then save the finding report text as markdown format in the `findings_report` state object.

Upon completion of the findings report, then the overall state of the workflow will be marked as complete.

Create a detailed product requirements document that breaks down the functionality by feature. The end result should be a list of features, with both backend user stories detailed for the given feature. The stories should be discrete and detailed. There may be multiple stories per feature. The end result should be a hybrid of very good user stories, with the details found in a PRD. Please number the features and the stories so they can be easily referred to later.

Each story format should be in the following format:

- Story title
- Story written in As a, I want, so that story format
- Design / UX consideration (if applicable)
- Testable acceptance criteria in Given, When, Then BDD format
- Detailed Architecture Design Notes
- Include any other detail or relevant notes that would help an AI-powered coding tool understand and correctly implement the features.
- Include any information about stories that are dependencies.
- Include any information about related stories for context.

You should also give any overarching context in the feature description.

At the top of the document include the detail of the data model for reference.

Do NOT include any summary, timelines, or non-functional requirements, unless they are relevant to the specific feature implementations.

Do NOT add any functionality that isn't in the above requirements, only add the functionality already defined.

Include a short 'Context' part at the top of the document that details the purpose and background information that is relevant to the project overall.

# VERIFICATION AND COMPLETION PHASE:

Validate your work to check that all the features defined in the documents above meet all the necessary requirements and there is no ambiguity or overlap before writing the final output.
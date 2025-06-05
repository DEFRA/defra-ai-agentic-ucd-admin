## API Endpoints

All endpoints are versioned (`/api/v1/**`) and use **plural** resource names.  
Unless stated otherwise, the payload format is `application/json` and IDs are MongoDB `ObjectId` values (24-character hex strings).

---

### 1. Research-Analysis Sessions

| Verb       | Endpoint                         | Purpose                                       |
| ---------- | -------------------------------- | --------------------------------------------- |
| **POST**   | `/api/v1/research-analyses`      | Create a new session                          |
| **GET**    | `/api/v1/research-analyses`      | List all sessions                             |
| **GET**    | `/api/v1/research-analyses/{id}` | Retrieve one session (includes `agent_state`) |
| **PATCH**  | `/api/v1/research-analyses/{id}` | Update session status                         |
| **DELETE** | `/api/v1/research-analyses/{id}` | Hard-delete a session (admin-only)            |
|            |                                  |                                               |

#### 1.1 Create session

`POST /api/v1/research-analyses`

```http
Request Body   # empty for now; kept extensible
{}
```

```jsonc
Response 201
{
  "_id": "661c0a7b8f45cc46d0f9f2c1",
  "created_at": "2025-06-04T14:05:21.381Z",
  "status": "INIT",
  "error_message": null,
  "agent_state": null
}
```

#### 1.2 List sessions

`GET /api/v1/research-analyses`

```jsonc
[
  {
    "_id": "661c0a7b8f45cc46d0f9f2c1",
    "created_at": "2025-06-04T14:05:21.381Z",
    "status": "COMPLETED"
  },
  {
    "_id": "661c0a96b54eea8535d9b34a",
    "created_at": "2025-06-04T15:17:02.119Z",
    "status": "RUNNING"
  }
]
```

#### 1.3 Get single session

`GET /api/v1/research-analyses/{id}`

```jsonc
{
  "_id": "661c0a7b8f45cc46d0f9f2c1",
  "created_at": "2025-06-04T14:05:21.381Z",
  "status": "COMPLETED",
  "error_message": null,
  "agent_state": {
    "process_start_date": "2025-06-04T14:06:04.002Z",
    "transcripts": [
      "## Interview #1\n\nRaw markdown transcript…"
    ],
    "transcripts_pii_cleaned": [
      "## Interview #1\n\n(PII removed)…"
    ],
    "affinity_map": "Markdown affinity map content…",
    "findings_report": "# Key Findings\n\n* …",
    "status": "FINISHED",
    "error_message": null
  }
}
```

#### 1.4 Update session status

`PATCH /api/v1/research-analyses/{id}`

```jsonc
Request Body
{
  "status": "RUNNING"
}
```

```jsonc
Response 200
{
  "_id": "661c0a7b8f45cc46d0f9f2c1",
  "created_at": "2025-06-04T14:05:21.381Z",
  "status": "RUNNING",
  "error_message": null,
  "agent_state": {
    "process_start_date": "2025-06-04T14:06:04.002Z",
    "status": "STARTING",
    "error_message": null
  }
}
```

---

### 2. Transcript Uploads

|Verb|Endpoint|Purpose|
|---|---|---|
|**POST**|`/api/v1/research-analyses/{id}/transcripts`|Upload one or more transcript files|
|**GET**|`/api/v1/research-analyses/{id}/transcripts`|List registered transcripts|

#### 2.1 Upload transcripts

`POST /api/v1/research-analyses/{id}/transcripts`  
_Consumes_ `multipart/form-data` — each part must be a `.md` or `.txt` file (no `.docx`).

```bash
curl -X POST \
  -F "files=@interview1.md" \
  -F "files=@interview2.txt" \
  /api/v1/research-analyses/661c0a7b8f45cc46d0f9f2c1/transcripts
```

```jsonc
Response 201
[
  {
    "_id":        "661c0b32a18dff4b3c9bd5e0",
    "analysis_id": "661c0a7b8f45cc46d0f9f2c1",
    "s3_key":     "research/661c0a7b…/interview1.md",
    "uploaded_at": "2025-06-04T14:07:54.009Z"
  },
  {
    "_id":        "661c0b32a18dff4b3c9bd5e1",
    "analysis_id": "661c0a7b8f45cc46d0f9f2c1",
    "s3_key":     "research/661c0a7b…/interview2.txt",
    "uploaded_at": "2025-06-04T14:07:54.087Z"
  }
]
```

---

### 3. Error Object (shared shape)

```jsonc
{
  "status": "ERROR",
  "error_message": "Human-readable explanation",
  "code": "TRANSCRIPT_FORMAT_INVALID"   // optional machine code
}
```

---

## Example Sequence (Happy Path)

1. **Create session** → `POST /research-analyses`  
    → returns `_id = 661c0a7b…`, `status = INIT`
    
2. **Upload transcripts** → `POST /research-analyses/{id}/transcripts`  
    → server records files, may update `status` internally
    
3. **Start workflow** → `PATCH /research-analyses/{id}` with `{"status": "RUNNING"}`  
    → returns `status = RUNNING`
    
4. **Poll session** → `GET /research-analyses/{id}` until processing finishes (`status = COMPLETED` or `ERROR`).
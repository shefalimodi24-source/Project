Project 
<img width="1329" height="2415" alt="mermaid-diagram (2)" src="https://github.com/user-attachments/assets/bdfdce9e-0090-4ae7-86e3-539861f1984d" />

**Meeting Intelligence Prototype - Development Plan**

## 1. Objective

Build a single-user web prototype that turns an uploaded recording of an in-person sales meeting into:

- a timestamped, speaker-labelled transcript;
- user-confirmed speaker roles;
- evidence-based customer insights;
- an evidence-based sales-representative scorecard; and
- human-approved, versioned proposals to update a company knowledge pack.

The default deployment is a self-hosted GPU server. The architecture must also permit a managed-cloud deployment later.

## 2. Fixed Product Decisions

| Area | Decision |
| --- | --- |
| Meeting type | Recorded, in-person meetings uploaded after the meeting |
| Speakers | 2 to 8 expected speakers; count is optional input |
| Languages | English, Hindi, and Hinglish |
| User model | Single-user prototype; that user can access all meetings |
| Setup inputs | Optional meeting purpose, optional participant context, optional expected speaker count, required Markdown knowledge pack |
| Diarization | WhisperX generates anonymous speaker labels; it does not assign real identities |
| Role assignment | Gemma 4 infers roles; user must confirm, swap, or edit them before analysis |
| LLM | Self-hosted Gemma 4 by default; deployment-specific model/serving configuration |
| Retrieval | PostgreSQL with pgvector; retrieve relevant Markdown knowledge chunks |
| Persistence | PostgreSQL for application data; local filesystem for audio files |
| Async jobs | Redis plus Celery |
| Retention | Delete unsaved meetings after 30 days; saved meetings persist until manually deleted |
| Knowledge updates | Human approval only; apply approved updates as versioned Markdown diffs |

## 3. User Workflow

1. User uploads an audio recording.
2. User optionally supplies meeting purpose, participant context, and expected speaker count (2-8).
3. User uploads or selects the required Markdown knowledge pack.
4. The system stores the audio and creates a background job.
5. WhisperX transcribes, aligns words, and diarizes speakers.
6. The UI shows a progress state: `uploaded`, `transcribing`, `awaiting_role_confirmation`, `analysing`, `complete`, or `failed`.
7. Gemma 4 proposes roles for anonymous speakers. The user confirms, swaps, or edits these roles.
8. Background analysis retrieves relevant knowledge-pack chunks and produces structured results.
9. The results page shows the transcript, customer insights, sales scorecard, and proposed knowledge updates.
10. The user approves or rejects each proposed knowledge update. Approved updates become a new version of the Markdown knowledge pack.

## 4. Architecture

```text
React / Next.js UI
        |
        v
FastAPI API ----------------------> PostgreSQL + pgvector
        |                                  |
        |                                  +-- meetings, transcripts, roles, analyses,
        |                                      knowledge-pack versions, audit events
        v
Redis job broker <----------------> Celery workers
        |                                  |
        |                                  +-- audio preprocessing / WhisperX
        |                                  +-- Gemma 4 role and meeting analysis
        |                                  +-- knowledge-pack chunking / embeddings
        v
Local audio storage                    Self-hosted GPU inference services
```

### Services

- **Web UI:** Next.js/React. Uploads, progress, role confirmation, results, approvals, and deletion.
- **API:** FastAPI. Validates requests, exposes meeting and knowledge-pack APIs, and enqueues work.
- **Worker:** Celery. Runs CPU/GPU-heavy tasks outside HTTP requests.
- **Broker:** Redis. Holds Celery jobs and retry metadata.
- **Database:** PostgreSQL with `pgvector` extension.
- **Audio storage:** local mounted directory. Database stores path, checksum, duration, and deletion deadline.
- **Speech service:** WhisperX for VAD, transcription, alignment, and diarization; use its required Hugging Face token/model access.
- **LLM service:** Gemma 4 served locally through a deployment adapter. The application calls an internal, OpenAI-compatible or equivalent inference interface so it can be swapped for managed hosting later.
- **Embeddings:** a local embedding model or deployment adapter. Keep it independent from Gemma generation so it can be changed without affecting the data model.

## 5. Core Data Model

Create migrations for at least these entities:

- `knowledge_packs`: name, current version, source Markdown, created/updated timestamps.
- `knowledge_pack_versions`: pack ID, version number, Markdown content, change summary, approval metadata.
- `knowledge_chunks`: version ID, source heading, content, embedding vector, ordinal.
- `meetings`: title, audio path, checksum, duration, purpose, participant context, expected speaker count, knowledge-pack version, status, quality warning, retention deadline, saved flag.
- `speaker_labels`: meeting ID, WhisperX label, inferred role, confidence, confirmed role, confirmation timestamp.
- `transcript_segments`: meeting ID, speaker label, start/end time, text, word-level data, confidence if available.
- `analyses`: meeting ID, schema version, structured JSON result, model metadata, created timestamp.
- `knowledge_proposals`: meeting ID, target pack/version, proposed Markdown diff, evidence references, status, approval/rejection metadata.
- `job_events`: meeting ID, stage, status, message, timestamps, retry count.

Use a deletion transaction/job that removes the local audio file and all meeting-linked rows for expired, unsaved meetings.

## 6. Structured Analysis Contract

Gemma must return validated JSON rather than display-ready prose. Define Pydantic schemas for:

- role inferences: speaker label, inferred role, confidence, rationale, transcript evidence;
- customer insights: category (`pain_point`, `objection`, `question`, `interest_signal`, `confusion`, `follow_up_request`), summary, evidence timestamps/quotes, confidence;
- sales assessment: dimensions `product_knowledge`, `communication_clarity`, `discovery`, `objection_handling`, each with a 0-100 score, rationale, transcript evidence, knowledge-pack references where applicable, confidence, and coaching actions;
- knowledge proposals: title, proposed Markdown change, evidence, confidence, and target knowledge-pack section.

Never render an unsupported conclusion as a fact. A failed schema validation should retry with a repair prompt and then mark the analysis failed if still invalid.

## 7. Quality and Safety Behaviour

- Detect and surface poor audio, likely overlapping speech, or uncertain diarization.
- Include affected timestamps in the quality warning.
- Let the user retry transcription with a changed expected speaker count from 2 to 8.
- Require acknowledgement before analysis proceeds with a quality warning.
- Do not run sales/customer analysis until speaker roles are confirmed.
- Preserve transcript evidence and knowledge references for every result.
- Do not auto-merge knowledge proposals.

## 8. Delivery Phases

### Phase 0 - Repository and local platform

Deliver:

- monorepo layout: `web/`, `api/`, `worker/`, `infra/`, `docs/`;
- Docker Compose for PostgreSQL + pgvector, Redis, API, worker, and local development dependencies;
- environment-variable template for database, Redis, audio directory, Hugging Face token, and Gemma endpoint;
- database migrations, health checks, linting, formatting, and test commands.

Done when a new developer can start the stack locally and create a database migration.

### Phase 1 - Meeting ingestion and lifecycle

Deliver:

- upload UI and API for supported audio formats;
- setup form with the four fixed inputs;
- secure local file naming and checksum calculation;
- meeting list/detail APIs and visible progress state;
- Celery job creation and event log;
- manual delete and scheduled 30-day deletion for unsaved meetings.

Done when an audio file survives upload, has a persisted lifecycle, and is deleted correctly by both manual and retention paths.

### Phase 2 - WhisperX transcription and diarization

Deliver:

- worker adapter for WhisperX;
- expected-speaker-count mapping to diarization minimum/maximum bounds;
- persistence of word/segment timestamps and anonymous labels;
- quality-warning generation;
- retry with a new speaker count;
- transcript UI grouped by speaker and timestamp.

Done when representative 2-8 speaker recordings produce viewable, persisted transcripts or clear actionable failure/warning states.

### Phase 3 - Role inference and confirmation

Deliver:

- Gemma role-inference prompt using transcript plus optional meeting context;
- structured role-inference validation;
- confirmation UI with edit and swap controls;
- job gate: no downstream analysis until roles are confirmed.

Done when the confirmed role mapping is applied consistently to every transcript segment and downstream prompt.

### Phase 4 - Knowledge-pack retrieval

Deliver:

- Markdown upload/version creation;
- deterministic heading-aware chunking;
- embedding generation and pgvector retrieval;
- knowledge-pack source references in API responses;
- version pinning: every meeting uses the knowledge-pack version selected at upload.

Done when a known query retrieves the intended Markdown section and historical meeting analysis remains reproducible after a pack update.

### Phase 5 - Customer and sales analysis

Deliver:

- Gemma prompts and Pydantic contracts for customer insights and the four-part scorecard;
- evidence/timestamp links for every result;
- knowledge-pack citations for product-knowledge scoring;
- results page with transcript, customer insights, scorecard, and coaching actions;
- confidence display and graceful analysis failure handling.

Done when generated results are schema-valid, traceable, and rendered without unsupported claims.

### Phase 6 - Knowledge proposals and review

Deliver:

- proposal generation from approved analysis output;
- Markdown-diff preview with transcript evidence;
- approve/reject controls;
- immutable knowledge-pack versions and rollback to a prior version.

Done when only approved proposals alter the active knowledge pack and every change has a review trail.

### Phase 7 - Evaluation, deployment, and hardening

Deliver:

- curated English, Hindi, and Hinglish evaluation recordings with expected labels and expected insight categories;
- documented GPU/server requirements for selected Gemma and WhisperX configurations;
- self-hosted deployment guide as the default;
- managed-cloud deployment adapter guide;
- backup, retention-job, observability, and failure-recovery checks.

Done when the full upload-to-result workflow works against 10-20 representative recordings and meets the acceptance criteria below.

## 9. Acceptance Criteria

- The app accepts in-person meeting recordings and an optional 2-8 speaker expectation.
- It supports English, Hindi, and Hinglish test recordings, with documented alignment limitations.
- It never treats WhisperX anonymous labels as real roles without user confirmation.
- Every customer insight and score has timestamped transcript evidence.
- Every product-knowledge assessment references relevant knowledge-pack content.
- Unsaved meetings are automatically and completely deleted after 30 days; saved meetings are excluded.
- Users can retry poor diarization with a different expected speaker count.
- Knowledge pack changes require explicit approval and produce version history.
- The system completes end-to-end on 10-20 representative recordings without manual engineering intervention.

## 10. Out of Scope for V1

- Live meeting capture or meeting-platform bots.
- Multi-user accounts, permissions, and tenant isolation.
- Manual transcript editing.
- Automatic knowledge-base updates.
- Claims of perfect diarization, transcription, role identification, or sales evaluation.
- General-purpose meeting types beyond the sales-focused workflow.

## 11. Recommended First Agent Task

Create the monorepo and Phase 0 Docker Compose stack only. Do not begin UI or model integration until the environment, migrations, health checks, and local test commands are documented and working.


whisperX github repo link : https://github.com/m-bain/whisperX

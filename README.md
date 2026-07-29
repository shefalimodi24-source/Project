# Meeting Intelligence - Server-Hosted Development Plan

## Product Objective

Build a self-hosted company web application for analysing in-person sales meetings. An authenticated user starts and stops a browser microphone recording. The platform securely uploads the audio to the company server, identifies speakers, asks the user to confirm roles, and produces three evidence-backed result buckets:

1. Customer insights
2. Sales-representative assessment
3. Company knowledge proposals

The system is designed for a company-owned GPU server. A managed-cloud deployment can be added later without changing the core business workflow.

## Final Product Decisions

| Area | Decision |
| --- | --- |
| Recording | The browser records a user-selected microphone for an in-person meeting. System/call audio is out of scope. |
| Upload reliability | Audio is uploaded as chunks while recording; the recording is finalized when the user presses Stop. |
| Authentication | Email/password login with secure session cookies. The first account is created by an administrator. |
| User roles | Admin, Manager, and Sales Representative. |
| Access | Admin sees everything and manages users/settings; Manager sees team meetings and approves knowledge updates; Sales Representative sees only their own recordings and feedback. |
| Speaker range | User can provide an optional expected count from 2 to 8 speakers. |
| Speech pipeline | WhisperX performs VAD, transcription, alignment, and diarization, returning anonymous speaker labels. |
| Role mapping | Gemma 4 suggests roles. User confirms, swaps, or edits them before analysis. The recording creator is the default sales-rep account; Admin/Manager can reassign it. |
| LLM | Gemma 4 runs on the company GPU server through an internal inference service. |
| Knowledge pack | One shared, versioned Markdown knowledge pack per company. Managers/Admins manage it. |
| Retrieval | PostgreSQL with pgvector retrieves relevant knowledge-pack chunks for analysis. |
| Audio storage | Self-hosted MinIO. PostgreSQL stores metadata and object references, not raw audio. |
| Job execution | PostgreSQL-backed job table plus a dedicated worker service. No Redis or Celery. |
| Processing UX | In-app progress states and in-app notifications only. |
| Retention | Delete unsaved audio, transcript, analyses, and proposals after 30 days. Admin/Manager can save meetings to exempt them. |
| Languages | English, Hindi, and Hinglish; validate alignment with representative recordings. |

## User Journey

```text
Log in
  -> Start recording
  -> Select browser microphone
  -> Upload encrypted/reliable audio chunks to MinIO during recording
  -> Stop recording and finalize upload
  -> Server worker transcribes and diarizes with WhisperX
  -> User confirms inferred speaker roles
  -> Server worker retrieves relevant knowledge-pack sections and runs Gemma 4 analysis
  -> Results page displays three buckets
  -> Manager approves/rejects knowledge proposal
  -> Approved proposal creates a new Markdown knowledge-pack version
```

## Three Result Buckets

### 1. Customer insights

- pain points and feature requests;
- objections and competitor mentions;
- questions, confusion, and requested clarification;
- interest signals and recommended follow-ups.

Every item must include transcript timestamps, supporting quotes, and confidence.

### 2. Sales-representative assessment

Score each category from 0-100 with rationale, timestamped evidence, confidence, and 1-3 coaching actions:

- product knowledge, grounded in the active knowledge-pack version;
- communication clarity;
- discovery / needs understanding;
- objection handling.

### 3. Company knowledge proposals

Generate proposed Markdown changes based on repeated or useful customer learning. Every proposal includes a Markdown diff, transcript evidence, confidence, and the target knowledge-pack section. Only a Manager or Admin can approve a proposal.

## Server Architecture

```text
Browser UI (Next.js / React)
  |-- recording controls, chunked upload, roles, results, notifications
  v
FastAPI API
  |-- authentication, authorization, recording metadata, signed MinIO upload routes
  |-- PostgreSQL transaction creates/retries durable processing jobs
  +--> PostgreSQL + pgvector
  |      users, roles, recordings, chunks, jobs, transcripts, analyses,
  |      knowledge-pack versions, proposals, audit events
  +--> MinIO
  |      recording chunks and completed audio objects
  v
Dedicated worker service
  |-- claims jobs from PostgreSQL using row locks
  |-- updates progress and retry state in PostgreSQL
  +--> WhisperX service
  +--> Gemma 4 inference service
  +--> embedding service for knowledge retrieval
```

## Processing State Machine

```text
recording
  -> uploading
  -> upload_finalized
  -> transcribing
  -> quality_warning | awaiting_role_confirmation
  -> analysing
  -> complete | failed
```

- A quality warning identifies affected timestamps and allows a retry with a different expected speaker count.
- Analysis cannot begin until a user confirms the speaker-role mapping.
- The worker must use idempotent job keys and database row locking so only one worker processes a job at a time.
- Failed jobs have visible failure messages, bounded retries, and an authorized retry action.

## Core Data Model

- `users`: email, password hash, role, active state, timestamps.
- `recordings`: creator, assigned sales representative, status, meeting context, expected speaker count, retention deadline, saved flag, MinIO object key, active knowledge-pack version.
- `recording_chunks`: recording ID, sequence number, MinIO object key, checksum, upload state.
- `processing_jobs`: recording ID, job type, state, attempt count, lock fields, progress message, error data.
- `speaker_labels`: recording ID, WhisperX label, inferred role/confidence, confirmed role, account assignment, confirmation audit data.
- `transcript_segments`: recording ID, speaker label, start/end timestamps, text, word-level data, quality metadata.
- `knowledge_packs` and `knowledge_pack_versions`: company-scoped Markdown content, version, author, approval history.
- `knowledge_chunks`: version ID, source heading, chunk content, pgvector embedding, ordinal.
- `analyses`: recording ID, JSON schema version, structured bucket output, model configuration, timestamps.
- `knowledge_proposals`: recording ID, target pack/version, Markdown diff, evidence, status, reviewer metadata.
- `audit_events`: actor, action, entity, timestamp, and change details for sensitive actions.

## API and UI Scope

### Authentication and team administration

- login, logout, password management, and secure cookie sessions;
- Admin user creation/deactivation and role assignment;
- authorization checks on every recording, analysis, knowledge-pack, and deletion route.

### Recording workspace

- microphone selector and browser permission handling;
- Start, Pause/Resume if supported by browser implementation, and Stop controls;
- chunk-upload recovery and finalization;
- visible elapsed time and upload/recording error states;
- optional meeting purpose, participant context, and expected speaker count (2-8).

### Processing and role confirmation

- in-app status timeline;
- quality warning and retry control;
- role-inference view showing anonymous speaker labels, confidence, evidence, confirm/swap/edit controls;
- sales-rep account assignment confirmation.

### Results and review

- timestamp-linked speaker-labelled transcript;
- three clearly separated result buckets;
- score evidence and knowledge-pack citations;
- proposal diff review, approve/reject controls, and knowledge-pack version history;
- save and delete controls according to permissions.

## Delivery Phases

### Phase 0 - Platform foundation

Create a monorepo with `web/`, `api/`, `worker/`, `infra/`, and `docs/`. Provide Docker Compose or equivalent server configuration for PostgreSQL + pgvector, MinIO, API, worker, and local Gemma/WhisperX service endpoints. Add migrations, health checks, environment templates, linting, tests, and deployment documentation.

**Done when:** a clean company-server environment starts all services and passes health checks.

### Phase 1 - Authentication and authorization

Implement email/password authentication, cookie sessions, Admin/Manager/Sales Representative permissions, user administration, and an audit trail for account/role changes.

**Done when:** each role can log in and cannot access unauthorized records/routes.

### Phase 2 - Browser recording and MinIO upload

Implement microphone selection, browser recording, chunked MinIO upload, recording finalization, and recovery from interrupted chunk upload. Persist recording metadata and retention deadline in PostgreSQL.

**Done when:** an authenticated user can create a complete server-stored recording without keeping the audio solely in browser memory.

### Phase 3 - PostgreSQL-backed processing worker

Implement a durable `processing_jobs` table, claim/lock behavior, progress events, retries, failure states, and retention/deletion jobs. Do not introduce Redis or Celery.

**Done when:** a stopped recording reliably starts processing even after an API restart, and job progress appears in the UI.

### Phase 4 - WhisperX pipeline

Run WhisperX for final audio objects, persist timestamps and anonymous speaker labels, map expected speaker counts to diarization bounds, and surface quality warnings/retries.

**Done when:** representative 2-8 speaker recordings produce a persisted transcript or an actionable warning/failure.

### Phase 5 - Roles and account assignment

Use Gemma 4 to infer roles from transcript/context. Build role confirmation, manual edit/swap, and sales-representative account reassignment for Admin/Manager.

**Done when:** no analysis can run without role confirmation, and the assessment is attached to the intended user account.

### Phase 6 - Knowledge-pack retrieval and structured analysis

Implement shared Markdown pack versioning, heading-aware chunking, embeddings, pgvector retrieval, and Gemma 4 structured JSON analysis. Validate every output with schemas.

**Done when:** all three buckets render with timestamped evidence, confidence, and knowledge-pack references where required.

### Phase 7 - Review, retention, and evaluation

Implement proposal approval/version history, 30-day deletion for unsaved meetings, saved-meeting exemptions, audit events, and test suites using English, Hindi, and Hinglish recordings.

**Done when:** Managers can safely approve knowledge changes and server retention removes all linked objects/data for expired unsaved recordings.

## Acceptance Criteria

- Authenticated users can record browser microphone audio and upload chunks to MinIO while recording.
- The server processes completed recordings without Redis/Celery and continues durable jobs through restarts.
- 2-8 speaker recordings can be diarized, quality-flagged, and role-confirmed.
- Users see only data allowed by their role.
- All three result buckets are evidence-backed; product-knowledge claims cite the selected knowledge-pack version.
- Only Admin/Manager can approve knowledge-pack updates.
- Unsaved meetings are fully deleted after 30 days from PostgreSQL and MinIO; saved meetings remain.
- The end-to-end workflow succeeds on representative English, Hindi, and Hinglish recordings.

## Explicitly Out of Scope for V1

- System audio or video-call capture.
- Live transcription or live coaching during a meeting.
- External SSO, email notifications, or multi-company tenancy.
- Manual transcript editing.
- Automatic knowledge-pack updates without approval.
- Claims of perfect diarization, role inference, or scoring.

WhisperX repo link : https://github.com/m-bain/whisperX 

<img width="3015" height="1781" alt="mermaid-diagram (4)" src="https://github.com/user-attachments/assets/212c380a-e855-44fa-847c-5dabfd52c053" />

We are building a self-hosted Meeting Intelligence platform.

Before writing application code:
1. Create CLAUDE.md, DEVELOPMENT_PLAN.md, ARCHITECTURE.md, and docs/decisions.md.
2. I will paste the complete development plan and architecture after this message. Use them as the source of truth.
3. Set up only Phase 0: monorepo structure, Docker Compose, Next.js, FastAPI, PostgreSQL with pgvector, MinIO, a PostgreSQL-backed worker service, health checks, environment template, linting, and tests.
4. Do not use Redis or Celery.
5. Do not build recording, WhisperX, Gemma, login, or UI features yet.
6. Show me the implementation plan before changing files.

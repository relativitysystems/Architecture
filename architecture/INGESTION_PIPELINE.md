# Ingestion Pipeline

Source repositories: `relativitysystems/Relativity` (entry points, storage handoff) and `relativitysystems/AIKB` (processing, chunking, embedding). See [AIKB.md](AIKB.md) for the schema these steps write to, and [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) for how each source provider feeds this pipeline.

## Overview

There is **one** ingestion pipeline, not one per source. Every document — whatever its origin — is normalized to raw bytes in Relativity, uploaded to AIKB's Supabase Storage bucket, and handed to AIKB via a single `POST /api/knowledge/ingest` call that enqueues an Inngest event. All parsing, chunking, embedding, and indexing happens inside AIKB's `knowledge-document-ingest` Inngest function, regardless of whether the file arrived via a manual upload, a ZIP archive, or a Google Drive import.

## Current Implementation — Entry Points

All routes below live in `Relativity/routes/api.js`, require `clientAuth`, and ultimately call `aikbService.uploadAndIngest()`.

| Entry point | Route | Trigger | Constraints |
|---|---|---|---|
| Single file upload | `POST /api/knowledge/upload` | User selects one or more files in the portal | `multer` memory storage; extensions `.txt/.md/.pdf/.docx`; default 20MB/file; per-client document count cap (default 50); blocked for `viewer` role |
| ZIP / folder archive | `POST /api/knowledge/import-zip` | User uploads a `.zip`, or the portal's folder picker packages a folder client-side | `AdmZip` extraction with zip-slip/path-traversal guard, hidden-file skip, extension allow-list, per-entry and total-size caps (default 20MB/entry, 200MB total, 100 files), concurrency 2 during ingest, continues on individual entry failures |
| Google Drive import | `POST /api/google-drive/import` | User picks files via the Google Picker UI in the portal | Client-obtained Google OAuth token (`x-google-access-token` header); server validates `mimeType` against an allow-list (PDF, DOCX, `text/plain`, `text/markdown`), downloads bytes directly from the Drive API, then ingests sequentially |

Every ingestion route also writes a `document_import_log` row in Relativity's own database (`supabaseService.logImportBatch`) tagging `source_type` (`local` / `folder_upload` / `zip` / `google_drive`) — this is Relativity-local provenance/display metadata only; AIKB's schema is the source of truth for document state.

`POST /api/voice/transcribe` is **not** an ingestion path — it transcribes audio to text for the chat input box and never writes to `knowledge_documents`.

## Architecture — Full Flow

```mermaid
sequenceDiagram
    participant B as Browser (Portal)
    participant Rel as Relativity (routes/api.js)
    participant Store as AIKB Supabase Storage
    participant AIKB as AIKB (routes/knowledge.js)
    participant Inn as Inngest (in-process)
    participant DB as AIKB Supabase (Postgres)
    participant OAI as OpenAI

    B->>Rel: POST /api/knowledge/upload (or import-zip / google-drive/import)
    Rel->>Store: upload raw bytes to uploads/{clientId}/{sourceFileId}
    Rel->>AIKB: POST /api/knowledge/ingest {clientId, sourceProvider:'portal_upload', sourceFileId, storagePath}
    AIKB->>Inn: inngest.send('knowledge/document.ingest', {...})
    AIKB-->>Rel: 202 {queued:true, eventId}
    Rel->>DB: (Relativity-local) document_import_log row
    Rel-->>B: upload accepted; portal polls for status

    Inn->>Inn: step create-job -> knowledge_ingestion_jobs (queued)
    Inn->>DB: step check-existing (content-hash dedup lookup)
    Inn->>DB: step update-job-running
    Inn->>Store: download file (30s timeout)
    Inn->>Inn: parseDocument() (60s timeout)
    Inn->>Inn: SHA-256 hash parsed text
    alt unchanged hash or duplicate content elsewhere, not forceReindex
        Inn->>DB: mark job completed, point at existing document (skip below)
    else new or forced
        Inn->>DB: upsert knowledge_documents (status=indexing, collection assigned if first insert)
        Inn->>DB: delete old chunks for this document (15s timeout)
        Inn->>Inn: chunkText() per PDF page or whole document
        Inn->>OAI: generateEmbeddings() batched 100/call (60s/batch, 90s outer)
        Inn->>DB: insert knowledge_chunks rows
        Inn->>DB: mark-indexed (status=indexed, last_indexed_at)
        Inn->>DB: complete-job (status=completed)
    end
    Note over Inn,DB: on any thrown error: job -> failed, document -> error
```

## Background Jobs

Inngest runs **in-process** — `aikb/server.js` mounts `serve({ client: inngest, functions })` on `/api/inngest` inside the same Express app that serves the REST API. The Inngest platform calls back into this HTTP endpoint to drive step execution; there is no separate worker process or deployment in this repository.

| Inngest function | Event | Retries | Concurrency | Steps |
|---|---|---|---|---|
| `knowledge-document-ingest` | `knowledge/document.ingest` | 3 | limit 2, keyed by `clientId` | create-job → check-existing → update-job-running → index-document-core (download, parse, hash, dedup, upsert, delete-old-chunks, chunk, embed, insert) → mark-indexed → complete-job |
| `knowledge/document.reindex` | `knowledge/document.reindex` | 3 | — | thin wrapper: validates metadata, then re-sends `knowledge/document.ingest` with `forceReindex: true` |
| `knowledge/document.delete` | `knowledge/document.delete` | 3 | — | find-document → delete-chunks → mark-deleted → delete-storage-file (optional) |
| `knowledge/slack.question.requested` | (Slack question path, not ingestion) | 3 | limit 5, keyed by `clientId` | see [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) |

Job status (`knowledge_ingestion_jobs.status`): `queued` → `running` → `completed` \| `failed`.
Document status (`knowledge_documents.status`): `pending` → `indexing` → `indexed` \| `error` \| `deleted`.

## Processing (Parsing)

`services/documentParser.js` handles exactly four input types:

- `.txt`, `.md`, `.csv` — read as plain text.
- `.docx` — parsed via `mammoth`.
- `.pdf` — parsed via `pdf-parse`; this extracts the **embedded text layer only**. There is no OCR fallback, so scanned or image-only PDFs yield no usable text.

A generic-MIME-to-extension fallback maps ambiguous MIME types (e.g. from Google Drive) onto the correct parser branch so that local uploads and Drive imports resolve identically.

## Chunk Creation

`services/chunkService.js#chunkText(text, metadata, options)`:

- `DEFAULT_CHUNK_SIZE = 4000` characters, `DEFAULT_OVERLAP = 400` characters — no caller in the codebase overrides these defaults.
- Splits on paragraph boundaries (`\n\n+`), accumulating into chunks and carrying the last `overlap` characters forward as context for the next chunk.
- A single paragraph exceeding `chunkSize` is hard-sliced at a `chunkSize - overlap` stride.
- For PDFs with a detected page structure, chunking runs **per page** (each chunk's `metadata.pageNumber` preserved, `chunk_index` numbered globally across the whole document); other file types are chunked as one whole-document pass.
- Each resulting `knowledge_chunks` row carries `document_id` (FK, `ON DELETE CASCADE`), `client_id`, `chunk_index`, `content`, `embedding`, and `metadata` (`clientId`, `fileName`, `sourceProvider`, `sourceFileId`, `pageNumber?`).

## Embedding Generation

`services/openaiService.js#generateEmbeddings(texts)`:

- Model: `text-embedding-3-small` (`OPENAI_EMBEDDING_MODEL` env var, default), 1536-dimension output matching the `VECTOR(1536)` column.
- Batched at 100 texts per OpenAI call; results re-sorted by response index to preserve chunk ordering.
- 60-second `AbortController` timeout per batch, nested inside a 90-second outer step timeout in the Inngest function.
- A returned-embedding-count vs. chunk-count mismatch throws and fails the step (and therefore the job, up to the function's retry limit).

## Storage

- Single bucket: AIKB's own Supabase Storage project, name from `AIKB_STORAGE_BUCKET` (default `aikb-documents`).
- Path convention: `uploads/{clientId}/{sourceFileId}`, recorded on `knowledge_documents.storage_path`.
- **Writer**: Relativity uploads the raw bytes directly to AIKB's Storage bucket (using AIKB's own service-role credentials configured in Relativity's environment) — this is a direct cross-service Storage write, not proxied through an AIKB API route.
- **Reader/deleter**: AIKB downloads the object during ingestion and deletes it when a document is deleted (`deleteFromStorage`, called from the `delete-storage-file` step).

## Content-Hash Dedup / Reindexing Behavior

Detected inside the `index-document-core` step:

1. **Unchanged-hash skip**: if a document already exists for `(clientId, sourceProvider, sourceFileId)` and the freshly-parsed text's SHA-256 hash matches the stored `content_hash`, and `forceReindex` is not set, the entire chunk/embed/insert sequence is skipped — the job completes pointing at the existing document.
2. **Cross-file duplicate-content skip**: if not `forceReindex`, AIKB additionally checks whether *any other* already-indexed document for the same client has the same content hash; if so, ingestion is skipped and the job points at that other document instead.
3. **Forced reindex** (`forceReindex: true`, only set by the reindex Inngest event) bypasses both checks, always deleting old chunks and re-chunking/re-embedding/re-inserting.

Re-uploading identical content is therefore cheap: no OpenAI calls and no chunk writes occur on a hash match.

## Current Limitations

- **One `sourceProvider` value accepted end-to-end.** Every AIKB ingest/reindex/delete code path hard-rejects any `sourceProvider` other than `'portal_upload'`, even though the schema (`source_provider TEXT`, comment `-- e.g. 'google_drive'`) is provider-generic. Google Drive imports are already tagged `portal_upload` by the time they reach AIKB — the provider distinction exists only on the Relativity side, before the Storage upload.
- **No OCR / scanned-PDF support** — see Processing above.
- **No recurring or webhook-driven ingestion.** Every entry point is a one-shot, user-initiated action; Google Drive import is a synchronous pull, not an ongoing sync (no watcher/webhook/cron exists for it).
- **No Slack-triggered ingestion** — Slack only triggers questions against already-ingested content, never new document ingestion.
- **In-process Inngest** — background jobs share the same Express process as the REST API; there is no independently scaled worker.
- Job "dismissal" in the portal UI is a client-side `localStorage` flag only — it does not affect server-side job records.

## Future Extension Points

- `knowledge_documents.source_provider` and `document_import_log.source_type` are already free-text/enum fields designed to hold more values — `document_import_log.source_type`'s allow-list has already been widened once (3 → 4 values, migration `20260712_document_import_log_widen.sql`) when folder upload was added.
- Adding a new file type requires only a new `documentParser.js` branch plus an extension/MIME allow-list update in the relevant Relativity route(s) — the Inngest pipeline itself is format-agnostic beyond the parse step.
- A future "recurring sync" capability (e.g., a continuously-synced Google Drive folder) is a natural extension of the existing one-shot Import/job model, but **no such capability exists today** — it would require a new watermark/cursor concept not present anywhere in the current schema or code.

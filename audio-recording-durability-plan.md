# Audio Recording Durability + Base64 Removal Plan

Last updated: 2026-03-10

## Implementation Status (Live)
1. ✅ **Backend durability is implemented**:
- Transcription request now uses `recording_id` in [transcriptions schema](/Users/crro/dev/versive/versive_api/versive_api/app/schemas/transcriptions.py#L4).
- `message_id` and `should_transcribe_audio` were removed from the transcription schema/type contract.
- In [transcriptions service](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py#L47), we now:
  - support transcribe-by-`recording_id`,
  - mark status as `processing/succeeded/failed`,
  - return `{ id, transcript: null }` when transcription fails but recording exists.
  - return `transcription_status` and `transcription_error` in API response.
2. ✅ **Base64 transcription path is removed**:
- Backend transcription service no longer decodes/accepts base64 audio payloads.
- Transcription API contract in frontend/backend now uses `recording_id` for this flow.
3. ✅ **Shared audio multipart endpoints are implemented** in [shared endpoints](/Users/crro/dev/versive/versive_api/versive_api/app/api/v0/endpoints/shared.py):
- `POST /audio-recordings/initiate-multipart`
- `POST /audio-recordings/part-urls`
- `POST /audio-recordings/complete-multipart`
- `POST /audio-recordings/abort-multipart`
- plus retry/recovery:
  - `POST /audio-recordings/{recording_id}/transcribe`
  - `GET /audio-recordings/recover`
4. ✅ **DB migration created** for transcription durability columns:
- [20260309230624_add_transcription_status_to_user_audio_recordings.sql](/Users/crro/dev/versive/versive_db/supabase/migrations/20260309230624_add_transcription_status_to_user_audio_recordings.sql)
5. ✅ **Frontend upload-first migration implemented**:
- [Interview.tsx](/Users/crro/dev/versive/interviewer/components/pages/Interview.tsx) now uploads audio via multipart, transcribes by `recording_id`, and shows recovery UI with playback + retry.
- [PrototypeTask.tsx](/Users/crro/dev/versive/interviewer/components/questions-types/PrototypeTask.tsx) now does the same.
- Shared client logic added in [user_audio_recordings.ts](/Users/crro/dev/versive/interviewer/services/user_audio_recordings.ts).
6. ✅ **Recovery on load implemented**:
- Interview and prototype flows now fetch recoverable recordings on question load and surface replay/retry UX after refresh.
7. ✅ **Explicit user-triggered upload retry implemented**:
- If upload fails before a recording row is created, the UI now keeps the blob in memory for the current tab and shows a `Retry Upload` action (Interview + PrototypeTask).
- If upload succeeded but transcription failed, UI shows replay + `Retry Transcription` using saved recording.
8. ✅ **Recovery UI polished and unified**:
- Interview and PrototypeTask now use the same toast-only recovery pattern instead of rendering a second panel below the question.
- The toast owns the preview player + retry action for upload/transcription failure states, and the layout was tightened to avoid collisions with existing interviewer controls.
- Recovery messaging now uses `audio response` language and the new upload-failure toast copy is localized from the participant's selected interview language.
- The unused legacy recovery-card component was removed after the toast-only flow replaced it.
- PrototypeTask fallback recovery now guards the secondary `recover` request so a failed fallback lookup does not surface as an unhandled frontend error; the toast still appears with retry state when both submit and recover fail.
9. ✅ **Mux creation removed from request critical path**:
- `complete-multipart` no longer waits for Mux asset creation before responding; asset generation now runs fire-and-forget in the background.
10. ✅ **Single-endpoint parallel ingest is implemented**:
- New endpoint: `POST /api/v0/shared/audio-recordings/ingest-and-transcribe`
- Backend now:
  - creates `user_audio_recordings` immediately,
  - writes the incoming audio request to a temp file,
  - starts R2 save in the background,
  - transcribes from the local temp file in parallel,
  - returns as soon as transcription finishes.
- Frontend Interview + PrototypeTask now use one submit call instead of `upload -> transcribe` for the primary live path.
11. ✅ **Storage status tracking is implemented**:
- New migration: [20260309230625_add_storage_status_to_user_audio_recordings.sql](/Users/crro/dev/versive/versive_db/supabase/migrations/20260309230625_add_storage_status_to_user_audio_recordings.sql)
- New row state:
  - `storage_status`
  - `storage_error`
  - `storage_started_at`
  - `storage_completed_at`
- Current upload-first multipart flow now marks rows as `storage_status='succeeded'`.
12. ✅ **Explicit upload retry now also covers background-save failures**:
- New endpoint: `POST /api/v0/shared/audio-recordings/{recording_id}/retry-upload`
- Frontend keeps the local audio blob in the current tab while storage is still resolving.
- If transcription succeeds but background save later fails, the UI now surfaces `Retry Upload` without making the user re-record.
13. ✅ **Review hardening fixes are applied**:
- PrototypeTask now restores the retry toast state when a recoverable recording is rehydrated after remount.
- PrototypeTask storage-resolution polling now guards against stale question state and wraps detached waiters in `try/catch`.
- Frontend storage polling now treats temporary `status` fetch failures as transient instead of exiting early.
- Public saved-recording endpoints now require `interview_id` and verify that the recording belongs to the requested interview before returning status, retrying upload, or retrying transcription.
- Public audio persistence endpoints now validate that `question_id` belongs to the same study and organization as the interview before inserting `user_audio_recordings`.
- `user_audio_recordings` service queries now scope reads/writes by `organization_id`.
- Blocking S3 downloads in [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py) were replaced with the async R2 client pattern.
- The recovery-copy helper was renamed to [recording-recovery-copy.ts](/Users/crro/dev/versive/interviewer/utils/recording-recovery-copy.ts) to match repo naming conventions.
- The interview recovery load/storage-monitor catch paths now report to Sentry. I also verified the multipart upload path and kept presigned R2 chunk PUTs on native `fetch`, since that is the safer match for existing multipart behavior.
- The un-applied migrations were corrected so transcription/storage backfills and defaults match the intended durability model.
14. ⏳ **Still pending / verify before closing**:
- Apply migration in the target environments.
- Run full end-to-end QA across interview modes.
- Decide whether to keep the old multipart-first path long-term or retire it after enough production confidence in the new direct-ingest path.

## Current Gap To The Next Target Architecture
1. The durability work is done, but the low-latency architecture you described is **not** implemented yet.
2. The current live flow is still strictly sequential:
- Browser uploads audio to R2 via `initiate-multipart` / `part-urls` / `complete-multipart`.
- `complete-multipart` inserts the `user_audio_recordings` row and returns `recording_id`.
- Browser then makes a second request to `POST /audio-recordings/{recording_id}/transcribe`.
3. What we already have that is reusable for the next phase:
- durable `user_audio_recordings` rows with transcription status/error tracking,
- transcription fallback chain in [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py),
- recovery + replay + retry UI in Interview / PrototypeTask,
- background `fire_and_forget(...)` helper in [common/utils.py](/Users/crro/dev/versive/versive_api/versive_api/app/common/utils.py),
- R2 save + Mux creation primitives already used by audio uploads.
4. What is still missing for the single-endpoint design:
- a backend `multipart/form-data` ingest endpoint that accepts the audio blob directly,
- creation of the `user_audio_recordings` row **before** durable upload finishes,
- a background storage task that continues independently of the transcription request path,
- separate storage/upload state on the recording row,
- frontend migration from `upload -> transcribe` to `submit once -> receive transcript`.
5. Important implementation note:
- FastAPI `BackgroundTasks` is not the right primitive for this specific flow because it runs only after the response is returned.
- If we want to upload in the background while the request is still waiting on transcription, we need an independent async task (`fire_and_forget(...)` / `asyncio.create_task(...)`) or a real job queue.

## Phase 6 (Single-Endpoint Parallel Ingest + Background Save)
1. Add a new shared endpoint for direct audio ingest, for example:
- `POST /api/v0/shared/audio-recordings/ingest-and-transcribe`
- request type: `multipart/form-data`
- fields:
  - `file`
  - `interview_id`
  - `question_id`
  - `language`
  - `keywords` (JSON string)
  - `failed_models` (JSON string)
  - `study_mode`
  - optional `media_duration`
2. Keep the existing multipart endpoints for fallback / very large uploads / future resumable upload needs, but move the live interview path to the new single-ingest endpoint.
3. Add explicit upload/storage status tracking on `user_audio_recordings` via a new migration:
- `storage_status text not null default 'pending'`
- `storage_error text`
- `storage_started_at timestamptz`
- `storage_completed_at timestamptz`
- optional check constraint for `('pending','uploading','succeeded','failed')`
4. In the new ingest endpoint:
- validate interview + organization metadata exactly like current audio upload endpoints,
- generate the final R2 key immediately,
- create the `user_audio_recordings` row up front with:
  - `study_id`
  - `organization_id`
  - `interview_id`
  - `question_id`
  - `bucket_name`
  - `file_path`
  - `media_duration`
  - `storage_status='uploading'`
  - `transcription_status='processing'`
5. Materialize the incoming `UploadFile` to a temp file in `/tmp` first.
- This avoids base64 entirely.
- It also avoids tying background upload to the request object's lifetime.
- Use the temp file as the source of truth for the parallel work.
6. Start the durable save as an independent async task:
- use `fire_and_forget(...)` or `asyncio.create_task(...)`,
- upload the temp file to R2 using [async_upload_file](/Users/crro/dev/versive/versive_api/versive_api/app/common/utils.py#L65) or an equivalent helper,
- on success:
  - set `storage_status='succeeded'`
  - set `storage_completed_at`
  - trigger Mux asset creation fire-and-forget
- on failure:
  - set `storage_status='failed'`
  - set `storage_error`
7. Run transcription in the request path in parallel with that storage task:
- read bytes from the temp file,
- feed those bytes into the existing fallback chain in [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py),
- keep the current provider retry order behavior; this part already exists and should be reused, not rewritten,
- update `transcription_status`, `transcription_model`, `media_duration`, `transcription_error` exactly like the current service.
8. Return the response as soon as transcription finishes.
- Do **not** wait for R2 upload or Mux.
- response should include:
  - `id`
  - `transcript`
  - `transcription_status`
  - `transcription_error`
  - `storage_status`
  - `storage_error`
9. Add cleanup guarantees for temp files.
- The upload task should own final temp-file cleanup because it may outlive the request.
- Do not delete the temp file in the request `finally` block until the background upload is done.
10. Update recovery endpoints/UI to understand both states independently:
- transcription can succeed while storage is still `uploading`,
- transcription can fail while storage succeeds,
- storage can fail even if transcription succeeds.
11. Frontend migration:
- replace `uploadAudioRecordingMultipart(...)` + `transcribeAudioRecording(...)` in Interview / PrototypeTask with one `submitAudioResponse(...)` call using `FormData`,
- keep the saved-audio recovery card for:
  - transcription failures,
  - storage failures,
  - client disconnects / timeouts after the server created the recording row.
12. Keep the current explicit `Retry Upload` UI, but repurpose it for the new world:
- if `storage_status='failed'`, retry only the storage step for the existing `recording_id`,
- if transcription failed but storage succeeded, retry only transcription,
- if both failed, offer both actions.
13. Add a backend retry endpoint for storage only:
- `POST /api/v0/shared/audio-recordings/{recording_id}/retry-upload`
- This should read the last known temp/staging source only if it still exists, or require a new upload if the ingest request already lost local source data.
- If keeping server-side source for retry is too operationally risky, skip this endpoint for phase 1 of the parallel architecture and rely on client-side retry while the blob is still available.
14. Practical recommendation for first rollout:
- do **not** try to make the new single-ingest endpoint resumable,
- do **not** remove the current multipart path yet,
- ship the direct-ingest endpoint first for normal live interview recordings, measure latency improvement, then decide whether to fully replace multipart for that path.

## Phase 6 Implementation Notes
1. Implemented backend endpoint:
- [shared.py](/Users/crro/dev/versive/versive_api/versive_api/app/api/v0/endpoints/shared.py)
  - `POST /audio-recordings/ingest-and-transcribe`
  - `POST /audio-recordings/{recording_id}/retry-upload`
  - `GET /audio-recordings/{recording_id}/status`
2. Implemented service changes:
- [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py) now supports transcribing a file-like source directly, so transcription no longer depends on downloading the recording back out of R2 for the new path.
- [user_audio_recordings.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/user_audio_recordings.py) now owns storage state updates and only generates playable URLs when storage has actually succeeded.
3. Implemented frontend changes:
- [user_audio_recordings.ts](/Users/crro/dev/versive/interviewer/services/user_audio_recordings.ts) now exposes:
  - `submitAudioResponse(...)`
  - `retryAudioRecordingUpload(...)`
  - `getAudioRecordingById(...)`
  - `waitForAudioRecordingStorageResolution(...)`
- [Interview.tsx](/Users/crro/dev/versive/interviewer/components/pages/Interview.tsx) now:
  - submits audio with one request,
  - falls back to recoverable row lookup if the request fails after row creation,
  - monitors background storage resolution,
  - shows `Retry Upload` if background save fails after transcription already succeeded.
- [PrototypeTask.tsx](/Users/crro/dev/versive/interviewer/components/questions-types/PrototypeTask.tsx) now follows the same pattern.
4. Current limitation:
- Background-save retry after page refresh is still not possible if storage failed before the file made it to R2.
- Retry for that specific case currently depends on the browser tab still holding the original audio blob in memory.
- This is an acceptable first phase because it removes the R2 upload from the live interview critical path without regressing explicit retry in the active session.

## Historical Baseline (Before This Implementation)
1. Audio is currently sent as base64 to `/api/v0/shared/transcriptions` from two places: [Interview.tsx](/Users/crro/dev/versive/interviewer/components/pages/Interview.tsx#L482) and [PrototypeTask.tsx](/Users/crro/dev/versive/interviewer/components/questions-types/PrototypeTask.tsx#L80).
2. Backend currently saves audio only after transcription succeeds in [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py#L84).
3. Frontend timeout is 60s in interview flow and 30s in prototype flow, which can be shorter than fallback transcription path.
4. `should_transcribe_audio` and `message_id` are defined for transcription request but not used in service logic ([transcriptions schema](/Users/crro/dev/versive/versive_api/versive_api/app/schemas/transcriptions.py#L4)).
5. Message linking via `audioRecordingId -> user_audio_recordings.message_id` already works in message routes and optimized RPCs, and should be kept unchanged.

## Important Scope Note
1. This plan covers flows that use the `/shared/transcriptions` path (VOICE_ONLY, VOICE_WITH_PREVIEW, VIDEO, prototype task recordings).
2. `VOICE_TO_VOICE` realtime flow uses Pipecat mic streaming and does not use this base64 transcription endpoint; do not block this project on that path.

## Phase 1 (Immediate Durability Hotfix, No Frontend Migration Yet)
1. Add transcription state columns on `user_audio_recordings` via new migration in `/Users/crro/dev/versive/versive_db/supabase/migrations`:
```sql
alter table public.user_audio_recordings
add column transcription_status text not null default 'uploaded',
add column transcription_error text,
add column transcription_attempt_count integer not null default 0,
add column transcription_last_attempt_at timestamptz;

alter table public.user_audio_recordings
add constraint user_audio_recordings_transcription_status_check
check (transcription_status in ('uploaded','processing','succeeded','failed','abandoned'));
```
2. Backfill existing rows in same migration:
```sql
update public.user_audio_recordings
set transcription_status = case
  when transcription_model is not null then 'succeeded'
  when file_path is not null then 'uploaded'
  else 'failed'
end;
```
3. In [transcriptions.py](/Users/crro/dev/versive/versive_api/versive_api/app/services/transcriptions.py), change `create_transcription` so upload is not dependent on transcription success:
- Decode base64.
- Save recording first (or in parallel), set `transcription_status='processing'`.
- Run transcription.
- On success update recording row with `transcription_model`, `media_duration`, `transcription_status='succeeded'`, clear error.
- On failure update row with `transcription_status='failed'`, `transcription_error`.
- Return `{ id: recording_id, transcript: null }` on transcription failure if upload succeeded.
4. Keep current response shape for compatibility.
5. Raise frontend transcription timeout to at least 180s in both call sites as temporary mitigation.

## Phase 2 (Move Off Base64: Upload-First Multipart Audio)
1. Implement dedicated shared audio multipart endpoints by copying the successful pattern used by media upload in [shared.py](/Users/crro/dev/versive/versive_api/versive_api/app/api/v0/endpoints/shared.py#L1496):
- `POST /api/v0/shared/audio-recordings/initiate-multipart`
- `POST /api/v0/shared/audio-recordings/part-urls`
- `POST /api/v0/shared/audio-recordings/complete-multipart`
- `POST /api/v0/shared/audio-recordings/abort-multipart`
2. Reuse multipart helpers from [common/utils.py](/Users/crro/dev/versive/versive_api/versive_api/app/common/utils.py#L429).
3. On `complete-multipart`, insert row in `user_audio_recordings` with:
- `interview_id`, `question_id`, `study_id`, `organization_id`
- `bucket_name`, `file_path`
- `transcription_status='uploaded'`
- optional `media_duration` if client sends it
4. Trigger Mux asset creation non-blocking like existing audio upload path.
5. Add new schema file or extend existing schema module for audio multipart request/response models.

## Phase 3 (Transcribe By recording_id + Recovery APIs)
1. Extend transcription request schema in [transcriptions.py schema](/Users/crro/dev/versive/versive_api/versive_api/app/schemas/transcriptions.py):
- Add `recording_id: Optional[str]`
- Make `audio` optional for transition
- Validate that at least one of `recording_id` or `audio` is present
2. In transcription service:
- If `recording_id` is provided, fetch `bucket_name/file_path`, read bytes from R2, transcribe, update same row status/model/duration.
- If `audio` is provided, keep legacy path (temporary).
3. Add recovery/list endpoint in shared routes:
- `GET /api/v0/shared/audio-recordings/recover?interview_id=...&question_id=...`
- Return rows with `message_id is null` and status in `('uploaded','failed')`, newest first.
- Include playable URL using existing presign utility.
4. Add retry endpoint convenience:
- `POST /api/v0/shared/audio-recordings/{id}/transcribe` (wrapper around transcription by `recording_id`).

## Phase 4 (Frontend Migration To Upload-First + Recovery UX)
1. Create frontend audio upload service (parallel to [mediaUpload.ts](/Users/crro/dev/versive/interviewer/services/mediaUpload.ts)) that uses multipart endpoints.
2. Update [Interview.tsx](/Users/crro/dev/versive/interviewer/components/pages/Interview.tsx):
- Replace base64 `transcribeAudio(audioResult...)` call path with:
  - upload blob -> `recording_id`
  - transcribe by `recording_id`
- Persist `recording_id` even when transcript fails.
- Show recovery card when transcription fails/timeouts:
  - Play audio
  - Retry transcription
  - Continue later
3. On interview/question load, call recovery endpoint and surface pending recording for that question.
4. Update [PrototypeTask.tsx](/Users/crro/dev/versive/interviewer/components/questions-types/PrototypeTask.tsx) to same upload-first + transcribe-by-id flow.
5. Keep existing message submission behavior unchanged: pass `audioRecordingId` so backend links `message_id`.

## Phase 5 (Remove Unused Fields + Deprecate Base64)
1. Remove `should_transcribe_audio` and `message_id` from:
- [backend transcription schema](/Users/crro/dev/versive/versive_api/versive_api/app/schemas/transcriptions.py)
- [frontend APITranscriptRequest type](/Users/crro/dev/versive/interviewer/types/user_audio_recordings.d.ts)
- payload builders in [Interview.tsx](/Users/crro/dev/versive/interviewer/components/pages/Interview.tsx#L483)
2. Keep backend tolerant for 1 deploy window.
3. After confirming all clients use upload-first, remove legacy base64 branch from transcription service.

## API Contract To Implement (Target)
1. Upload complete returns:
```json
{ "id": "recording_uuid", "file_path": "...", "transcription_status": "uploaded" }
```
2. Transcribe by recording id request:
```json
{
  "recording_id": "recording_uuid",
  "language": "en",
  "keywords": ["Acme"],
  "failed_models": [0,2],
  "study_mode": "VOICE_ONLY"
}
```
3. Transcribe response:
```json
{
  "id": "recording_uuid",
  "transcript": { "text": "...", "duration": 42, "model": "versive-transcribe", "failures": [1] },
  "transcription_status": "succeeded",
  "transcription_error": null
}
```
4. Failure-but-saved response:
```json
{
  "id": "recording_uuid",
  "transcript": null,
  "transcription_status": "failed",
  "transcription_error": "provider timeout"
}
```

## Testing Checklist (Must Pass)
1. 5+ minute recording uploads and survives forced transcription timeout.
2. Retry transcription by `recording_id` succeeds without re-upload.
3. Page refresh during failed transcription shows recoverable audio entry.
4. VOICE_ONLY, VOICE_WITH_PREVIEW, VIDEO, and PrototypeTask paths all support recovery.
5. Message submission still links `audioRecordingId` to `message_id`.
6. Mux failure does not fail recording save/transcription flow.
7. Legacy base64 request still works during transition window.
8. New migration does not break results pages that read `audioRecordings`.

## Rollout Sequence
1. Deploy Phase 1 backend durability + migration first.
2. Deploy Phase 2/3 backend endpoints.
3. Deploy Phase 4 frontend migration.
4. Monitor errors/timeouts and orphaned recording counts.
5. Remove legacy fields and base64 branch in Phase 5 after verification.

## Why This Solves The Problem
1. User audio is persisted before transcription success is required.
2. Provider slowness/timeouts no longer cause irreversible audio loss.
3. Base64 payload-size risk is removed from primary path.
4. Users can recover, play, and resubmit recordings across interview flows.

# Prompt: Add Full Interview Audio Player to Transcript Page

Use this prompt to instruct a coding agent to add an audio player on the transcript page that plays the full merged interview audio. The asset is stored in the `interview_assets` table (type `INTERVIEW_GENERATED_MERGED_AUDIO`), the file is in R2, and the player should look similar to the podcast player card but use native audio (not Mux).

---

## Goal

1. **Backend**: Retrieve the full-interview merged audio asset by `interview_id` from `interview_assets` (type `INTERVIEW_GENERATED_MERGED_AUDIO`), generate a presigned URL for the R2 file, and expose it in the API response used by the transcript page.
2. **Frontend**: On the transcript page, show an audio player (similar visual layout to the podcast player card) that uses that presigned URL with native audio playback (no Mux) so the user can play the full audio of the interview.

---

## Reference: Where Things Live

### Database

- **Table**: `interview_assets`
- **Asset type**: `INTERVIEW_GENERATED_MERGED_AUDIO`
- **Lookup**: by `interview_id` (and filter by `type = 'INTERVIEW_GENERATED_MERGED_AUDIO'`). Order by `created_at desc` and take the latest if multiple exist.
- **Columns used for playback**: `id`, `bucket_name`, `file_path` (presigned URL is generated from these; **no Mux for this asset** - it uses native audio playback).

### Transcript page (where the player must appear)

- **Study results transcript pane** (fetches interview details and renders transcript):
  - **File**: `interviewer/components/pages/study/results/responses-transcript-pane.tsx`
  - It calls `interviewService.getInterviewWithDetails(activeId, supabase)` and passes the result into the transcript view.
- **Transcript / voice transcript components** (actual transcript UI):
  - **Files**: `interviewer/components/pages/study/results/responses-transcript.tsx` (wrapper), `interviewer/components/StudyResults/VoiceTranscript.tsx`, `interviewer/components/StudyResults/Transcript.tsx`
  - The player should be added in the transcript view (e.g. top of the transcript content in `VoiceTranscript.tsx` or the shared layout) so it appears when viewing a single interview's transcript.

### Podcast player (copy the visual layout, NOT the Mux player)

**IMPORTANT**: The podcast player uses **Mux** (with `playbackId` and `tokens`), but the merged interview audio is **NOT uploaded to Mux** yet. So copy the **visual card layout** from the podcast player, but use the **native audio player** pattern (like user message recordings without Mux).

- **Component**: `interviewer/components/StudyInsights/StudyInsightSummary.tsx`
- **Relevant snippet** (podcast section with player, ~lines 231–301):
  - **Visual layout to copy**: rounded card (`rounded-xl bg-zinc-950/5`), flex layout, icon (e.g. `AudioLines`), title (e.g. "Full interview audio"), short description, then the player inline.
  - **Player**: Use `AudioPlayer` from `@/components/AudioPlayer` with **ONLY `src`** (no `playbackId` or `tokens`). This makes it fall back to native `<audio>` element, which is what you want for non-Mux assets.

**AudioPlayer usage (podcast - uses Mux):**

```tsx
<AudioPlayer
  src={podcast.audio_url}
  playbackId={podcast.mux_playback_id}  // ❌ Don't include this for merged audio
  tokens={{ playback: podcast.mux_playback_token }}  // ❌ Don't include this for merged audio
  themeColor={study.config.primaryColor}
  className="max-w-none bg-transparent"
/>
```

**AudioPlayer usage (merged interview audio - NO Mux, native audio only):**

```tsx
<AudioPlayer
  src={mergedAudioUrl}  // ✅ Only pass src - this triggers native audio player
  themeColor={study.config.primaryColor}
  className="max-w-none bg-transparent"
/>
```

- **File**: `interviewer/components/AudioPlayer.tsx` — when you pass only `src` (no `playbackId`/`tokens`), it automatically falls back to native `<audio>` element. This is the same pattern used for user message audio recordings when they don't have Mux.

### Reference: User message audio recordings (native audio player pattern)

When user message audio recordings don't have Mux, they use the same `AudioPlayer` component with just `src`:

- **File**: `interviewer/components/StudyResults/Transcript.tsx` (around lines 966–974)
- **Pattern**: `<AudioPlayer src={recording} playbackId={audioRecording?.mux_playback_id} tokens={{...}} />` — when `mux_playback_id` is null/undefined, it falls back to native audio. For merged audio, simply don't pass `playbackId` or `tokens` at all.

---

## Reference: How presigned URLs are produced today

### Backend (presign for interview assets)

- **File**: `versive_api/versive_api/app/common/utils.py`
  - **`presign_audio_url(interview, supabase)`** (lines ~153–161): If `interview.get('audio_id')` is set, loads that row from `interview_assets` by `id`, then sets `interview['audio_url']` using `generate_presigned_url(bucket_name, file_path)`.
  - **`generate_presigned_url(bucket_name, file_path)`**: Builds the presigned URL for R2 (same pattern to reuse for merged audio).

So the pattern is: get one row from `interview_assets` (by id or by `interview_id` + type), then call `generate_presigned_url(record['bucket_name'], record['file_path'])` and attach the result to the payload.

**Suggested backend change**: In the same place interview details are built (e.g. where `presign_interview_with_urls` is used), add a step that:

1. Queries `interview_assets` for `interview_id = <id>` and `type = 'INTERVIEW_GENERATED_MERGED_AUDIO'`, order by `created_at desc`, limit 1.
2. If a row exists, generates a presigned URL and adds it to the response (e.g. `interview['merged_audio_url']` or a top-level `mergedAudioUrl` in the JSON). Use the same `generate_presigned_url` from `versive_api/versive_api/app/common/utils.py`.

**Interview details endpoint** (so the frontend gets the URL in one call):

- **File**: `versive_api/versive_api/app/services/interviews.py`
  - **Function**: `get_interview_with_details(interview_id, supabase)` (around lines 505–656). It already calls `presign_interview_with_urls(interview, supabase)` and returns a dict that includes `"interview": enhanced_interview`. Add the merged-audio presign logic (query by `interview_id` + type, then presign) and put the URL on `enhanced_interview` (e.g. `merged_audio_url`) or on the root of the returned dict (e.g. `mergedAudioUrl`). Keep the same API shape the frontend already uses (see `getInterviewWithDetails` and the transcript pane).

### Frontend (existing presign usage for interview assets)

- **File**: `interviewer/services/interview_assets.ts`
  - **`mapRecordingIdToPresignedUrl(recordingId, supabase)`**: Fetches an `interview_assets` row by `id`, then calls `generatePresignedUrl(bucket_name, file_path)` from `@/lib/aws`. So if you ever need to presign on the frontend (e.g. from a recording id), that's the pattern. Prefer returning the URL from the backend in `get_interview_with_details` so the transcript page just receives a ready-to-play URL.

---

## Reference: Study assets (podcast) retrieval pattern

For comparison with how "one asset per context" is retrieved and presigned:

- **Backend**:
  - **File**: `versive_api/versive_api/app/services/study_assets.py`
  - **Function**: `get_last_study_asset_by_study_id(study_id, supabase)` (lines ~106–130): selects from `study_assets` by `study_id`, `is_deleted = False`, order `created_at desc`, limit 1; then calls `presign_audio_insights_url(study_asset, supabase)` to set `audio_url` and Mux token. Same idea: query by parent id (+ type if needed), take latest, then presign.
- **Presign helper**: `versive_api/versive_api/app/common/utils.py` — `presign_audio_insights_url(study_asset, supabase)` (lines ~192–204) loads `file_path`/`bucket_name` from `study_assets` and sets `study_asset['audio_url']`.

For **interview_assets** you don't have a helper yet; implement the same idea: query by `interview_id` and `type`, then one presign call and attach the URL to the interview details response.

---

## Reference: User message audio retrieval (same R2/presign idea)

- **File**: `versive_api/versive_api/app/common/utils.py`
  - **`process_interview_user_media_urls(recording, column_name)`** (lines ~222–242): For a recording that has `bucket_name` and `file_path`, sets `recording[column_name] = generate_presigned_url(bucket_name, file_path)` and optionally adds Mux playback token. So "get bucket + path, then presign" is the pattern.
- **File**: `versive_api/versive_api/app/services/interviews.py`
  - **`get_interview_with_details`** uses that to add `audio_url` (and similar) to each user recording. The merged-audio URL can be added in the same function to the returned interview/details object.

---

## Implementation checklist for the agent

1. **Backend**
   - In `versive_api/versive_api/app/services/interviews.py`, inside `get_interview_with_details` (after building `interview` and before or after `presign_interview_with_urls`):
     - Query `interview_assets` for `interview_id`, `type = 'INTERVIEW_GENERATED_MERGED_AUDIO'`, order by `created_at desc`, limit 1.
     - If a row exists, call `generate_presigned_url(bucket_name, file_path)` from `versive_api/versive_api/app/common/utils.py` and set the result on the response (e.g. `enhanced_interview['merged_audio_url']` or a top-level key like `mergedAudioUrl` in the returned dict).
   - Ensure the interview details API response (used by `getInterviewWithDetails`) includes this new field so the frontend doesn't need an extra request.

2. **Frontend**
   - In the transcript view (e.g. `interviewer/components/StudyResults/VoiceTranscript.tsx` or the shared transcript layout used by both Transcript and VoiceTranscript), add a section at the top that:
     - Renders only when the interview has a merged-audio URL (e.g. `interview.merged_audio_url` or `mergedAudioUrl` from the API).
     - **Copy the visual card layout** from the podcast block in `StudyInsightSummary.tsx` (lines ~231–301): rounded card (`rounded-xl bg-zinc-950/5`), flex layout, icon (e.g. `AudioLines`), title (e.g. "Full interview audio"), short description.
     - **Use native audio player**: `<AudioPlayer src={mergedAudioUrl} themeColor={study.config.primaryColor} className="max-w-none bg-transparent" />` — **ONLY pass `src`**, do NOT pass `playbackId` or `tokens` (this makes it use native `<audio>` element, not Mux player). See `interviewer/components/StudyResults/Transcript.tsx` (lines ~966–974) for an example of user message audio without Mux.
     - Use `study.config.primaryColor` for theme if available.
     - Ensure the type for the interview/details response includes the new field (e.g. in `interviewer/types` or wherever `Interview` / interview-details type is defined).

3. **Optional**
   - Add a download link/button that opens the same presigned URL (or URL with a query param for download) so users can download the full interview audio, similar to the podcast download in `StudyInsightSummary.tsx`.

---

## File paths summary (for copy-paste)

| Purpose | File path |
|--------|-----------|
| Podcast player UI (visual layout reference - copy the card style) | `interviewer/components/StudyInsights/StudyInsightSummary.tsx` (lines ~231–301) |
| AudioPlayer component (use with only `src` for native audio) | `interviewer/components/AudioPlayer.tsx` |
| User message audio player example (native audio pattern) | `interviewer/components/StudyResults/Transcript.tsx` (lines ~966–974) |
| Transcript pane (loads interview details) | `interviewer/components/pages/study/results/responses-transcript-pane.tsx` |
| Voice transcript view (add player here or in shared layout) | `interviewer/components/StudyResults/VoiceTranscript.tsx` |
| Study transcript wrapper | `interviewer/components/pages/study/results/responses-transcript.tsx` |
| Interview details API (backend) | `versive_api/versive_api/app/services/interviews.py` → `get_interview_with_details` |
| Presign helpers (backend) | `versive_api/versive_api/app/common/utils.py` → `generate_presigned_url`, `presign_audio_url` |
| Interview assets presign (frontend, if needed) | `interviewer/services/interview_assets.ts` → `mapRecordingIdToPresignedUrl` |
| Study assets retrieval (backend pattern) | `versive_api/versive_api/app/services/study_assets.py` → `get_last_study_asset_by_study_id` |

---

## Constant to use

- Asset type: **`INTERVIEW_GENERATED_MERGED_AUDIO`** (string value `"INTERVIEW_GENERATED_MERGED_AUDIO"`). The voice interviewer (chandler) already uses this type when creating the asset. If the constant is not in the backend yet, add it in **`versive_api/versive_api/app/common/constants.py`** next to `INTERVIEW_GENERATED_AUDIO` (line ~17) and use it in the query filter.

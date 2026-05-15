# Video Content Retrieval Tool — Design Doc

## Goal

Given a video and a natural-language query, return the most relevant
screenshots, the surrounding video clip, and a textual answer grounded in
the audio transcript.

## Non-goals

- Multi-video corpora. One video at a time.
- Real-time ingestion. Preprocessing runs once per upload.
- Production deployment / multi-user auth.

## Current state

| Component        | Status   | Notes                                              |
| ---------------- | -------- | -------------------------------------------------- |
| Frame extraction | Done     | `backend.py::extract_frames`, 1 fps via cv2        |
| Audio extraction | Done     | `backend.py::extract_audio`, moviepy → mp3         |
| Transcription    | Done     | OpenAI Whisper → `response.txt`                    |
| CLIP encoding    | External | `test_clip.py` POSTs to `localhost:3000/encode` — service not in repo |
| Vector store     | Done     | Milvus Lite, `milvus_demo.db`, dim=768             |
| Image retrieval  | Done     | `test_milvus.py`, top-K cosine search              |
| Video clip cut   | Missing  | Only still frames today                            |
| Transcript search| Missing  | Transcript exists but isn't indexed                |
| Frontend wiring  | Missing  | `frontend.py` shows placeholder image only         |
| CLIP service     | Missing  | `service.py` is a summarizer, not CLIP             |

## Architecture

```
       ┌───────────────┐
upload │  Streamlit    │ query
──────►│  frontend.py  │◄──────── user
       └──┬─────────┬──┘
          │ ingest  │ query
          ▼         ▼
   ┌────────────┐  ┌──────────────┐
   │ preprocess │  │ retrieval    │
   │ (backend)  │  │ (backend)    │
   └──┬─────────┘  └──┬───────────┘
      │ frames+audio  │ text+img embed
      ▼               ▼
   ┌────────────┐  ┌──────────────┐
   │ Whisper    │  │ CLIP service │ (BentoML, :3000)
   │ (OpenAI)   │  └──┬───────────┘
   └──┬─────────┘     │
      │ transcript    │ vectors
      ▼               ▼
   ┌────────────────────────────┐
   │ Milvus Lite (local file)   │
   │  - frames collection       │
   │  - transcript collection   │
   └────────────────────────────┘
```

## Data model

**`frames` collection** (Milvus, dim=768)
- `id`: int, frame index
- `vector`: CLIP image embedding
- `image_loc`: path under `output_frames/`
- `timestamp_s`: float, derived from frame_index / fps

**`transcript` collection** (Milvus, dim=768)
- `id`: int, sentence index
- `vector`: CLIP text embedding (same space → enables text↔text and text↔image)
- `text`: sentence string
- `start_s`, `end_s`: word-level timestamps from Whisper

## Components to build

### 1. CLIP service (`service.py`)
Replace the summarization stub with a BentoML service exposing `/encode`.
Accepts either `img_blob` (base64) or `text`, returns a 768-dim vector.
Uses `openai/clip-vit-large-patch14` to match the dim=768 already in
`test_milvus.py`. Must handle both modes in one endpoint since
`test_clip.py` and `test_milvus.py` both call the same URL.

### 2. Preprocessing module (`preprocess.py`, refactored from `backend.py`)
- Stop running at import time.
- Functions: `preprocess(video_path) -> PreprocessResult`
- Switch Whisper call to `response_format="verbose_json"` to get
  word/segment timestamps; persist as `transcript.json` instead of plain
  `response.txt`.
- Persist `fps` so retrieval can map frame_index → timestamp.

### 3. Indexer (`indexer.py`)
- Batch-encode all frames via CLIP service → insert into `frames`.
- Split transcript into segments → encode → insert into `transcript`.
- Idempotent: drop+recreate collections per video.

### 4. Retrieval (`retrieval.py`)
- `search(query, k=5) -> {frames: [...], segments: [...], clips: [...]}`
- Encode query once, search both collections.
- For each top frame, cut a ±3 s clip with moviepy and cache under
  `output_clips/`.

### 5. Frontend (`frontend.py`)
- On upload: call `preprocess` + `index`. Show a spinner.
- On query: call `retrieval.search`. Render:
  - Top transcript snippet(s) as the textual answer.
  - Grid of matched frames.
  - Inline `st.video` of the top clip.

## Data flow

**Ingest** (once per upload)
1. Save video to working dir.
2. Extract frames @ 1 fps → `output_frames/`.
3. Extract audio → `output_audio/extracted_audio.mp3`.
4. Whisper transcribe (verbose) → `transcript.json`.
5. CLIP-encode frames → insert into `frames` collection.
6. CLIP-encode transcript segments → insert into `transcript` collection.

**Query**
1. CLIP-encode query text.
2. Search `frames` (top-K) and `transcript` (top-K) in parallel.
3. For top frame, map index → timestamp → cut clip.
4. Return frames + segments + clip path to frontend.

## Open questions

- **Sentence vs. segment granularity for transcript?** Whisper segments
  are ~5–10 s; sentence-splitting may give tighter answers but loses
  Whisper's timestamps. Start with Whisper segments.
- **Should the summarizer in `service.py` survive?** It's not on the
  critical path. Drop it unless we want a "summarize the video" feature
  later.
- **Frame sampling rate.** 1 fps is fine for talking-head footage; for
  fast cuts we'd want adaptive sampling (shot detection). Out of scope
  for v1.
- **Hosting the CLIP service.** Local BentoML on `:3000` is fine for
  hackathon-style use. No plan to deploy.

## Milestones

1. CLIP service stands up, `test_clip.py` and `test_milvus.py` keep working.
2. Preprocessing refactored to a function; transcript stored with timestamps.
3. Indexer builds both collections from one video.
4. Retrieval returns frames + segments for a text query.
5. Video-clip cutting wired in.
6. Frontend end-to-end: upload → query → text + images + clip.

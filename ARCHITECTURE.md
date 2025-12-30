```markdown
# truthy Architecture

truthy is a full-stack system for deepfake detection on **any online video**, built as:

- A **Chrome extension (Manifest V3)** that works on all major sites (YouTube, TikTok, Instagram, X, random players).
- A **FastAPI + Celery + Redis backend** that ingests videos, runs a multimodal detection pipeline (metadata, neural, behavioral), and returns structured JSON results. [web:52][web:56][web:83]

---

## 1. High-Level Flow

1. User opens a page with a video and clicks the truthy extension.
2. The extension:
   - Tries to read a **direct video URL** from the page.
   - If it cannot (DRM, blob URLs), it **records N seconds** of the tab (audio + video).
3. The extension sends either:
   - `{ "url": "…" }` to the backend, or
   - An uploaded video **file** (captured clip).
4. Backend:
   - Normalizes input into a local video file.
   - Runs three detection layers in parallel:
     - **Metadata**
     - **Neural (visual + audio)**
     - **Behavioral / temporal**
   - Fuses them into a final reliability-aware score + explanation. [web:56][web:84]
5. The extension polls the backend for results and renders:
   - Overall label (e.g. “Likely AI-Generated”)
   - Confidence score
   - 2–4 natural-language explanations. [web:53][web:68]

---

## 2. Backend Architecture

**Tech:** FastAPI, Celery, Redis, ffmpeg, PyTorch, MediaPipe. [web:80][web:82]

Root layout:

```
truthy-backend/
├── app/           # FastAPI entrypoint (HTTP + WS)
├── core/          # Shared schemas, Celery config, utils
├── ingestion/     # URL/file → local video file
├── layers/        # 3 detection layers (metadata, neural, behavioral)
├── ensemble/      # Fusion + final output
├── storage/       # Temp files + cleanup
└── tests/         # Sample videos + demo scripts
```

### 2.1 app/

**Responsibility:** Public API for the extension.

- `main.py`
  - `POST /analyze/video`
    - Accepts JSON `{ "url": "..." }` **or** `multipart/form-data` with `file`.
    - Enqueues a Celery job for ingestion + analysis.
    - Returns a `task_id` immediately.
  - `GET /tasks/{task_id}`: Returns task status and (when ready) the final JSON.
  - WebSocket endpoint (optional) for streaming progress.
- `config.py`
  - Central configuration (e.g. `REDIS_URL`, `MPS_ENABLED`, timeouts). [web:80]

FastAPI is configured with CORS to allow requests from the Chrome extension (`chrome-extension://…`). [web:61][web:68]

### 2.2 core/

**Responsibility:** Shared contracts and plumbing.

- `schemas.py`
  - `VideoInput`: input to the pipeline (URL or file reference).
  - `MetadataResult`, `NeuralResult`, `BehavioralResult`: typed outputs of each layer.
  - `FinalOutput`: final JSON response with:
    - `overall_score`, `label`
    - `layer_scores` (metadata / neural_visual / neural_audio / behavioral)
    - `conditions` and `explanations`. [web:56][web:84]
- `celery_app.py`
  - Celery app configuration with Redis as broker/result backend.
- `utils.py`
  - `ffmpeg_wrapper()`: convenience around ffmpeg for frame sampling, audio extraction.
  - `redis_client()`: simple Redis client factory.

### 2.3 ingestion/

**Responsibility:** Universal input normalization.

- `ingest.py`
  - **URL path:**
    - Uses `yt-dlp` for major platforms and `requests` for direct `.mp4` / `.m3u8`.
    - Applies size/time caps: may download only the first X seconds.
  - **File path:**
    - Accepts uploaded recording (e.g. tab capture) and moves it into temp storage.
  - Probes basic file info (duration, resolution, fps). [web:16][web:80]
- `tasks.py`
  - Celery task: `ingest_video()`
    - Produces a normalized local file path + basic metadata.
    - Triggers the three detection layer tasks using the normalized path.

### 2.4 layers/

**Responsibility:** Multimodal detection pipeline, three independent, parallel experts. [web:56][web:82]

#### layers/metadata/ – Layer 1

- `analyzer.py`
  - Uses `pymediainfo` / ExifTool to read:
    - Container, codec, bitrate, fps, resolution, encoder tags.
  - Heuristics:
    - Non-standard fps, multiple re-encodes, missing camera metadata, etc.
  - Outputs:
    - `metadata_score` and flags like `["no_camera_metadata", "odd_framerate"]`.
- `tasks.py`
  - Celery task writes result to Redis (e.g. `layer:{task_id}:metadata`).

#### layers/neural/ – Layer 2

- `preprocess.py`
  - Samples frames (e.g. 2–5 fps) with ffmpeg/OpenCV.
  - Detects/crops faces using MediaPipe.
  - Extracts audio and builds spectrograms / MFCCs. [web:82][web:84]
- `models.py`
  - Loads visual deepfake model (e.g. MesoNet/EfficientNet/ResNet).
  - Loads audio deepfake model (e.g. CNN over spectrograms).
  - Provides prediction functions for visual and audio.
- `tasks.py`
  - Runs preprocessing + models.
  - Aggregates frame and audio predictions into:
    - `visual_score`, `audio_score`, `nn_video_score`.
  - Stores neural results in Redis (e.g. `layer:{task_id}:neural`).

#### layers/behavioral/ – Layer 3

- `analyzer.py`
  - Tracks face landmarks over time.
  - Computes:
    - Blink rate and distribution.
    - Head pose dynamics and landmark coherence.
    - Audio–visual sync metrics (lip movement vs audio energy). [web:56][web:84]
  - Outputs:
    - `behavioral_score` and flags like `["lip_sync_mismatch", "blink_pattern_anomaly"]`.
- `tasks.py`
  - Writes behavioral layer result to Redis (`layer:{task_id}:behavioral`).

### 2.5 ensemble/

**Responsibility:** Reliability-aware fusion + final JSON. [web:57][web:60]

- `fuse.py`
  - Collects outputs from metadata, neural, behavioral layers.
  - Applies configurable base weights (e.g. 15% / 50% / 35%).
  - Adjusts weights based on reliability:
    - No audio → drop audio and renormalize.
    - No face → downweight behavioral.
    - Very short clip → lower confidence overall.
  - Produces `FinalOutput`:
    - `overall_score` and discrete `label`.
    - `layer_scores`.
    - `conditions` and user-facing `explanations`.
- `tasks.py`
  - Celery task waits for all needed layer keys or times out.
  - Runs `fuse()` and stores final result for `GET /tasks/{id}`.

### 2.6 storage/ and tests/

- `storage/cleanup.py`
  - Periodic cleanup of temp video files and related keys (e.g. after 1 hour) to avoid unbounded storage growth.
- `tests/`
  - Sample videos and unit tests for individual layers.
  - `demo_curl.sh` for quick end-to-end testing of `/analyze/video`. [web:26][web:80]

---

## 3. Frontend (Chrome Extension) Architecture

**Tech:** Chrome Extensions Manifest V3, background service worker, content script, popup UI. [web:62][web:83][web:85]

Frontend layout (under `frontend/` or similar):

```
frontend/
├── public/
│   ├── icon16.png
│   ├── icon48.png
│   └── icon128.png
├── src/
│   ├── background/
│   │   └── service_worker.ts
│   ├── content/
│   │   └── content.ts
│   ├── popup/
│   │   ├── index.html
│   │   ├── popup.tsx
│   │   └── popup.css
│   └── shared/
│       └── api.ts
├── manifest.json
└── package.json
```

### 3.1 manifest.json

- Declares **Manifest V3** extension with:
  - `background.service_worker` (main orchestrator).
  - `action.default_popup` (popup UI).
  - `content_scripts` injected on `<all_urls>`.
- Grants permissions:
  - `"activeTab"`, `"tabs"`, `"scripting"`, `"tabCapture"`, `"storage"`.
- `host_permissions: ["<all_urls>"]` for broad video support. [web:62][web:66]

### 3.2 content script (`src/content/content.ts`)

**Responsibility:** Understand the current page’s video situation.

- Finds the primary `<video>` element on the page (largest area).
- Returns one of:
  - `found: false` → no video; background will go to capture mode.
  - `found: true, hasDirectUrl: true, url: "…"`.
  - `found: true, hasDirectUrl: false` (blob/DRM) → capture mode. [web:52][web:83]

Communicates with background via `chrome.runtime.sendMessage`.

### 3.3 background service worker (`src/background/service_worker.ts`)

**Responsibility:** Decide between URL-based analysis and tab capture, then call backend. [web:66][web:69][web:85]

- Handles `START_ANALYSIS` messages from the popup.
- Asks content script for video info.
- Branches:
  - **Direct URL available:**
    - Calls backend `POST /analyze/video` with `{ url }` using shared API helper.
  - **No usable URL (or no `<video>`):**
    - Uses `chrome.tabCapture.getMediaStreamId` + `MediaRecorder` to record N seconds of tab audio+video.
    - Builds a `File` from the captured blob and posts it as `multipart/form-data` to `/analyze/video`. [web:66][web:69]
- Returns the backend’s initial response (`task_id` or final JSON) to the popup.

### 3.4 shared API helper (`src/shared/api.ts`)

**Responsibility:** Standardize backend HTTP calls.

- `analyzeByUrl(videoUrl: string)`
- `analyzeByFile(file: File)`
- `getTaskResult(taskId: string)`

Uses `fetch` to call the FastAPI backend, which exposes the same API across all platforms. [web:52][web:68]

### 3.5 popup UI (`src/popup/*`)

**Responsibility:** User-facing controls and results.

- `popup.tsx`
  - Button: “Analyze video”.
  - On click:
    - Sends `START_ANALYSIS` to background.
    - Handles initial response:
      - If `task_id`, polls `/tasks/{id}` via `getTaskResult`.
      - If final JSON, renders it directly.
  - Displays:
    - Status (`idle`, `running`, `done`, `error`).
    - Final label, score, and top explanations from `FinalOutput`. [web:53][web:68]
- `index.html` / `popup.css`
  - Minimal layout and styling for a modern, clean UI.

---

## 4. Key Design Principles

- **Platform-agnostic ingestion:** All site-specific quirks are isolated to:
  - Browser: URL discovery + tab capture.
  - Ingestion layer: URL/file → normalized video file. [web:52][web:69]
- **Multimodal detection:** Visual, audio, and behavioral signals combined for robustness against single-modality attacks. [web:56][web:82][web:84]
- **Reliability-aware ensemble:** Final decision downweights unreliable modalities (no face, no audio, short clip) for more stable scores. [web:57][web:60]
- **Clean contracts:** The system revolves around a single core API:
  - **Input:** URL or file.
  - **Output:** `FinalOutput` JSON (score, label, explanations) used by any frontend (extension, future web app, etc.).

```

[1](https://www.edenai.co/post/how-to-create-an-ai-content-detector-for-text-images-and-deepfakes-a-step-by-step-tutorial)
[2](https://github.com/Znis/deepfake-video-detection-project)
[3](https://www.linkedin.com/posts/mayur-thakre-a03857261_ai-deepfakedetection-machinelearning-activity-7337480583121391616-7KXI)
[4](https://chromewebstore.google.com/detail/deepfake-detector/apickkejnjmjmpmalonhpgjbkmhljgmc?hl=en-US)
[5](https://github.com/nithins7676/AIAD-Deepfake-project)
[6](https://www.ijraset.com/research-paper/multimodal-deepfake-detection)
[7](https://reintech.io/blog/understanding-chrome-extension-architecture)
[8](https://dev.to/aurigin/tutorial-building-an-ai-deepfake-detector-chrome-plugin-4p06)
[9](https://arxiv.org/html/2410.03487v1)
[10](https://sriniously.xyz/blog/chrome-extension)

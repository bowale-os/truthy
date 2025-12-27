```
# truthy Architecture

This document describes the end-to-end architecture of truthy: a browser extension + backend system that analyzes online images, audio, and video to estimate whether content is AI-generated and explains why.

---

## 1. System Overview

truthy consists of three major parts:

- **Browser Extension / Web UI**
  - Injected into websites (YouTube, TikTok, Instagram, X, generic HTML5 players).
  - Lets users trigger analysis on the currently playing media.
- **Backend API (FastAPI)**
  - Receives media URLs or uploaded clips.
  - Normalizes inputs and creates asynchronous detection tasks.
- **Detection Pipeline (Celery Workers + ML Models)**
  - Runs a 3-layer detection pipeline:
    - Layer 1 – Metadata Analysis
    - Layer 2 – Neural Network Classification
    - Layer 3 – Behavioral / Temporal Analysis
  - Aggregates scores and generates explanations.

### 1.1 Architecture Diagram

```
+------------------------+
|  Browser Extension     |
|  (Chrome MV3, TS/JS)   |
+-----------+------------+
            |
            |  HTTPS (URL or clip)
            v
+-----------+------------+
|  FastAPI Backend       |
|  /analyze/* endpoints  |
+-----------+------------+
            |
            |  Celery task (task_id)
            v
+-----------+------------+
|  Celery Workers        |
|  (Python, PyTorch)     |
+-----------+------------+
   |       |        |
   |       |        |
   v       v        v
Layer 1  Layer 2  Layer 3
Metadata Neural  Behavioral
   \       |       /
    \      |      /
     +-----+-----+
           |
           v
Score Aggregation
+ JSON Result in Redis
            |
            v
+-----------+------------+
|  FastAPI Result API    |
+-----------+------------+
            |
            v
+------------------------+
|  Extension Popup / UI  |
+------------------------+
```

---

## 2. Browser Extension

### 2.1 Manifest and Scripts

- **Manifest V3**
  - Declares:
    - `content_scripts` for `https://*/*` and `http://*/*` (or narrower).
    - `background.service_worker` for network calls and messaging.
    - `action` for popup UI.

- **Content Script**
  - Injected into pages to:
    - Detect `<video>` elements.
    - For platforms like YouTube, read the current video URL/ID.
    - Optionally show a floating “Scan with truthy” button.

- **Background / Popup**
  - Handles:
    - Calls to the FastAPI backend with `fetch`.
    - Optional `chrome.tabCapture` and `MediaRecorder` to record a few seconds when no direct URL is available (DRM, blob URLs).
  - Updates the popup UI with:
    - Progress (queued, running, completed).
    - Final confidence score and explanations.

### 2.2 Input Modes

- **URL mode (preferred)**  
  When a usable media URL or known platform URL is visible:
  - Sends JSON `{ "url": "https://.../video" }` to `POST /analyze/video`.

- **Capture mode (fallback)**  
  When no usable URL is visible:
  - Uses `chrome.tabCapture` to capture the tab.
  - Records 5–10 seconds via `MediaRecorder`.
  - Uploads the resulting WebM or MP4 as a file.

Both modes give the backend a video file to analyze.

---

## 3. Backend API (FastAPI)

### 3.1 Endpoints

- **`POST /analyze/video`**
  - Request:
    - JSON with `{ "url": string }` **or**
    - `multipart/form-data` with a `file` field.
  - Behavior:
    - Validates input.
    - If URL:
      - Uses `yt-dlp` or HTTP GET to download the video.
    - If file:
      - Stores temporarily on disk (e.g., `/tmp`).
    - Creates a Celery task containing:
      - File path.
      - Media type and any platform hints.
    - Returns:
      ```
      { "task_id": "uuid", "status": "queued" }
      ```

- **`GET /tasks/{task_id}`**
  - Returns:
    ```
    {
      "status": "queued" | "running" | "completed" | "error",
      "result": { ... }
    }
    ```

- **`GET /health`**
  - Returns basic health info and optionally MPS support status.

- **Optional: WebSocket `/ws/results`**
  - Pushes progress updates to the extension (e.g., layer-by-layer status).

### 3.2 Tech

- FastAPI and Uvicorn.
- Celery for asynchronous tasks.
- Redis as broker and result backend.
- yt-dlp for platform-aware video downloads.
- Docker and docker-compose for local development.

---

## 4. Detection Pipeline

Each Celery worker:

- Loads models and tools at startup.
- Processes tasks from Redis.
- Runs three layers:
  - Layer 1 – Metadata analysis.
  - Layer 2 – Neural classification.
  - Layer 3 – Behavioral / temporal analysis.
- Aggregates scores and explanations.
- Stores the result back in Redis keyed by `task_id`.

### 4.1 Layer 1 – Metadata Analysis

- **Inputs**
  - Local video or image file path.

- **Tools**
  - ExifTool and `pymediainfo` to extract:
    - Container, codec, encoder.
    - Bitrate, resolution, fps.
    - EXIF and other metadata where available.

- **Signals**
  - Missing or stripped camera metadata.
  - Non-standard fps or suspicious encoder chains.

- **Output**
  ```
  {
    "metadata_score": 0.0,
    "metadata_flags": [
      "no_camera_metadata"
    ],
    "metadata_explanation": "Video has no camera metadata and appears re-encoded."
  }
  ```

---

### 4.2 Layer 2 – Neural Network Classification

- **Goals**
  - Detect visual and audio artifacts characteristic of deepfakes and synthetic media.

- **Tools**
  - PyTorch (MPS for Apple Silicon in dev, CPU/GPU in prod).
  - FFmpeg and OpenCV for frame and audio extraction.
  - MediaPipe for face detection and landmarks.
  - librosa for audio features and spectrograms.

#### Visual branch

- **Steps**
  - Sample frames at 2–5 fps.
  - Detect faces and generate crops.
  - Run crops through a deepfake classifier (e.g., MesoNet, EfficientNet, CLIP-based).
  - Aggregate frame scores into one `nn_visual_score`.

#### Audio branch

- **Steps**
  - Extract the audio with FFmpeg.
  - Compute mel-spectrograms using librosa.
  - Run an audio deepfake model to get `nn_audio_score`.

#### Output

```
{
  "nn_visual_score": 0.8,
  "nn_audio_score": 0.7,
  "nn_explanations": [
    "Frame-level facial artifacts similar to known deepfake generators.",
    "Audio spectrum resembles synthetic speech."
  ]
}
```

---

### 4.3 Layer 3 – Behavioral and Temporal Analysis

- **Goals**
  - Catch sophisticated fakes by analyzing motion, blink patterns, and audio-visual sync.

- **Tools**
  - OpenCV and MediaPipe for facial landmarks.
  - Temporal statistics for landmarks and embeddings.
  - Simple SyncNet-style logic for lip-sync.

- **Signals**
  - **Blink patterns**
    - Estimate blink rate and compare with common human ranges.
  - **Head pose**
    - Estimate pose per frame and look for unnatural or overly rigid motion.
  - **Temporal coherence**
    - Track how landmark positions or embeddings change; suspicious smoothness or jitter can signal manipulation.
  - **Audio-visual sync**
    - Compare mouth opening and lip movement with audio energy or rough phoneme timing.

- **Output**

```
{
  "behavioral_score": 0.75,
  "behavioral_flags": [
    "lip_sync_mismatch",
    "low_blink_rate"
  ],
  "behavioral_explanation": "Lip movements are not well aligned with the speech; blink rate is unusually low."
}
```

---

## 5. Score Aggregation and Explanations

### 5.1 Aggregation

For videos with audio, truthy uses a weighted ensemble:

- `w_metadata = 0.15`
- `w_neural = 0.50` (fusion of visual and audio scores)
- `w_behavioral = 0.35`

The overall score is:

```
overall_score = w_metadata * metadata_score
              + w_neural   * nn_score
              + w_behavioral * behavioral_score
```

Weights are tunable per media type and dataset. Ensemble fusion is used because it is typically more robust than a single detector.

### 5.2 Labels and Schema

Mapping `overall_score`:

- 0.0–0.3 → Likely Authentic
- 0.3–0.6 → Uncertain
- 0.6–0.8 → Likely AI-Generated
- 0.8–1.0 → High Confidence AI

**Example final result:**

```
{
  "task_id": "uuid",
  "status": "completed",
  "overall_score": 0.78,
  "label": "Likely AI-Generated",
  "layer_scores": {
    "metadata": 0.2,
    "neural_visual": 0.85,
    "neural_audio": 0.7,
    "behavioral": 0.8
  },
  "flags": [
    "lip_sync_mismatch",
    "low_blink_rate"
  ],
  "explanations": [
    "Lip movements are not well aligned with the audio.",
    "Blink rate is lower than typical human patterns.",
    "Frame-level artifacts resemble known deepfakes."
  ],
  "conditions": [
    "short_clip_10s"
  ]
}
```

The API returns this JSON from `GET /tasks/{task_id}`, and the extension renders the label, score, and explanations.

---

## 6. Responsibilities and Extensibility

- **Extension team**
  - Manifest, content script, popup UI.
  - URL extraction and tab capture.
  - Calling API endpoints and rendering results.

- **Backend and infra team**
  - FastAPI routes, Celery, Redis.
  - File storage and security.
  - Deployment and monitoring.

- **ML and detection team**
  - `metadata.py`, `neural.py`, `behavioral.py`.
  - Model loading, device selection, and performance tuning.
  - Evaluating new models and adjusting weights.

The architecture is modular so that new detectors can be added as extra sub-layers or ensemble members without changing the public API or the extension contract.

---
```

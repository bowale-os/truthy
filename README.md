```md
# truthy – Responsible AI Media Detector

truthy is an open-source **browser extension + backend** that analyzes online images, audio, and video to estimate whether content is **AI-generated** (deepfakes, synthetic speech, etc.).

Instead of returning a binary *real/fake* label, truthy provides an **explainable confidence score**, highlighting *why* content may be synthetic and how certain the system is.

---

## ✨ Features

### 🌐 Works Across the Web
- Chrome Extension (Manifest V3)
- Supports:
  - YouTube videos (including Shorts)
  - TikTok, Instagram Reels, X (Twitter) videos
  - Any HTML5 `<video>` element
  - Short captured clips

### 🧠 Layered Detection Pipeline
1. **Layer 1 – Metadata**
   - Codec and encoder inspection
   - Timestamp inconsistencies
   - Missing or abnormal EXIF / camera metadata

2. **Layer 2 – Neural Models**
   - Image, video, and audio deepfake detectors
   - PyTorch-based models trained on synthetic media artifacts

3. **Layer 3 – Behavioral Analysis**
   - Temporal and physiological signals
   - Blink rate anomalies
   - Head pose consistency
   - Lip-sync alignment with audio

### 🔍 Explainable Results
- Confidence score between **0.0 – 1.0**
- Human-readable labels:
  - Likely Authentic
  - Uncertain
  - Likely AI-Generated
  - High Confidence AI
- Clear explanations such as:
  - “Lip movements poorly align with audio”
  - “Blink rate is unusually low for natural speech”

### 🔓 100% Open Source
- Built with FastAPI, Celery, Redis, PyTorch, and standard browser APIs
- Modular and extensible:
  - Plug in new models
  - Add new detection layers
  - Swap datasets or heuristics

---

## 🏗 High-Level Architecture

```

Chrome Extension / Web App
│
▼
FastAPI API (task creation & result retrieval)
│
▼
Celery Workers (Python)
├─ Layer 1: Metadata Analysis
├─ Layer 2: Neural Classification (image/audio/video)
└─ Layer 3: Behavioral / Temporal Analysis
│
▼
Score Aggregation + Explanation Generation
│
▼
JSON Result → Rendered in Extension UI

````

For a detailed breakdown of each component and detection layer, see  
[`docs/architecture.md`](docs/architecture.md)

---

## 🛠 Tech Stack

### Frontend / Extension
- Chrome Extension (Manifest V3)
- TypeScript / JavaScript

### Backend
- FastAPI + Uvicorn
- Celery + Redis (task queue & result store)
- yt-dlp
- FFmpeg
- OpenCV
- MediaPipe

### Machine Learning & Signal Processing
- PyTorch (Apple Silicon MPS in dev; CPU/GPU in production)
- librosa
- ASVspoof-style audio detection models
- Frame-based deepfake detectors:
  - MesoNet
  - EfficientNet
  - CLIP-style classifiers

---

## 🚀 Quick Start (Development)

### 1️⃣ Backend Setup

```bash
# clone repository
git clone https://github.com/your-org/truthy.git
cd truthy

# create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# install backend dependencies
pip install -r backend/requirements.txt

# start Redis (via Docker)
docker run -p 6379:6379 redis

# start Celery worker
cd backend
celery -A app.celery_app worker --loglevel=INFO

# start FastAPI server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
````

#### Backend Endpoints

* `POST /analyze/video` – Submit a URL or uploaded file
* `GET /tasks/{task_id}` – Retrieve task status and results
* `GET /health` – Health check and hardware info (CPU / GPU / MPS)

---

### 2️⃣ Chrome Extension Setup

```bash
cd extension
npm install
npm run build
```

Then:

1. Open `chrome://extensions/`
2. Enable **Developer Mode**
3. Click **Load unpacked**
4. Select the `extension/dist` directory

---

## 🧪 How to Use

1. Navigate to a page with a video (e.g., YouTube Shorts).
2. Click the **truthy** extension icon or floating **Scan** button.
3. truthy will:

   * Capture the video URL or clip
   * Send it to the backend
   * Run the full 3-layer detection pipeline
4. The popup displays:

   * Confidence score and label
   * Key reasons behind the assessment

---

## 🤝 Responsible AI Principles

truthy is designed with **Responsible AI** in mind:

* **Transparency**
  Always shows uncertainty and reasoning—never claims perfect accuracy.

* **Human-in-the-Loop**
  Encourages human judgment for high-stakes decisions.

* **Fairness**
  Tested across diverse faces, voices, accents, and languages.

* **Privacy**
  No permanent storage of user media. Files are retained only as long as needed for analysis.

---

## 📁 Project Structure

```
.
├── README.md
├── docs/
│   └── architecture.md
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI routes
│   │   ├── schemas.py         # Pydantic models
│   │   ├── workers.py         # Celery tasks
│   │   ├── detection/
│   │   │   ├── metadata.py    # Layer 1
│   │   │   ├── neural.py      # Layer 2
│   │   │   └── behavioral.py  # Layer 3
│   │   └── aggregation.py     # Scoring + explanations
│   └── requirements.txt
└── extension/
    ├── manifest.json
    ├── src/
    │   ├── content-script.ts
    │   ├── background.ts
    │   └── popup.tsx
    └── package.json
```

---

## 📜 License

MIT License.
See `LICENSE` for details.

---

## 🌱 Contributing

Contributions are welcome!

* Open issues for bugs or feature requests
* Submit pull requests for:

  * New detection models
  * Performance improvements
  * UI/UX enhancements
  * Documentation updates

---

## 🔗 Disclaimer

truthy provides probabilistic assessments, not definitive judgments.
Results should **never** be treated as legal, journalistic, or forensic proof.

---

**truthy — building trust in a synthetic media world.**

```

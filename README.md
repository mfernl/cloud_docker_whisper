# Whisper Transcription API — Dockerized & Deployed on Render

A containerized, cloud-deployed version of [`Transcript_whisper`](https://github.com/mfernl/Transcript_whisper), my Bachelor's thesis project: a FastAPI service that wraps OpenAI Whisper to evaluate automatic transcription of air traffic control ↔ aircraft radio communications (originally built during a 9-month internship at Indra Sistemas).

This repository focuses on the containerization and deployment side: packaging the service with Docker, adapting it to run without a GPU, and shipping it as a live, publicly reachable API.

## Live demo

- **API base URL:** https://cloud-docker-whisper.onrender.com
- **Interactive docs (Swagger UI):** https://cloud-docker-whisper.onrender.com/docs
- **OpenAPI spec:** https://cloud-docker-whisper.onrender.com/openapi.json

> Deployed on Render's free tier, which spins down after periods of inactivity. If the service has been idle, the first request can take up to a minute while it wakes back up; requests after that are fast.

## What it does

- **Batch transcription** (`PUT /upload`): upload a `.wav` file, get back a timestamped transcript.
- **Session-based streaming transcription** (`GET /crearRTsession`, `PUT /broadcast`, `GET /cerrarRTsession`): open a session, push audio chunks to it over HTTP as they become available, and close it to get the merged transcript. This simulates real-time transcription through short-lived HTTP requests rather than a persistent WebSocket connection.
- **JWT-based auth**: token login/logout, with an admin account gating new user registration.
- **Custom keyword detection**: admins manage a dictionary of "important words" (CSV upload); the service flags and logs their appearance in any transcription, with who detected it and when.
- **Monitoring endpoints**: `/appstatus` (uptime, connected clients), `/hoststatus` (CPU/RAM/GPU usage), `/appstatistics` (query and transcription counters).
- **Self-hosted Swagger UI** at `/docs`, independent of FastAPI's default docs.

## Why this exists

The original project was built and tuned for GPU inference. Making it deployable on a free cloud tier meant re-thinking that: no GPU, a single low-memory container, and a Whisper model small enough to run acceptable inference on CPU alone. The `tiny` Whisper model and a single-worker transcription queue (backed by a semaphore) replace what was originally a GPU-parallel setup, trading raw throughput for something that actually runs within the resource limits of a free-tier deployment.

## Tech stack

- **API:** FastAPI, Uvicorn
- **Speech-to-text:** OpenAI Whisper (CPU inference)
- **Auth:** JWT (python-jose), bcrypt password hashing
- **Persistence:** SQLAlchemy + SQLite
- **Audio processing:** pydub + ffmpeg
- **Testing:** pytest, httpx (async test client), jiwer (word-error-rate evaluation)
- **Containerization:** Docker, Docker Compose
- **Deployment:** Render (Web Service, free tier)

## Running locally

### With Docker Compose (recommended)

```bash
docker compose up --build
```

The API will be available at `http://localhost:8000`, with docs at `http://localhost:8000/docs`.

### With Docker directly

```bash
docker build -t whisper-api .
docker run -p 8000:8000 whisper-api
```

### Without Docker

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

## Docker design notes

- **Non-root user:** the container runs as a dedicated `appuser`, not root, following least-privilege practice.
- **CPU-only PyTorch build:** installed explicitly from PyTorch's CPU wheel index to avoid pulling in unused CUDA dependencies and keep the image smaller.
- **Layer caching:** dependencies are installed via a BuildKit cache mount before the rest of the source is copied in, so rebuilds after a code change don't reinstall every package.
- **`ffmpeg`** is installed at the OS level, since `pydub` shells out to it for audio decoding.

## Deployment

The image is deployed directly from this repository's `Dockerfile` as a Render Web Service. No orchestration layer beyond Render's own container runtime; a persistent volume is not used in production, so the SQLite database and downloaded Whisper model are ephemeral across deploys.

## Testing

The original project's test suite (`pytest`, `pytest-asyncio`, `httpx`) covers authentication, session handling, batch and streaming transcription, concurrent-session behavior, and transcription quality via word-error-rate (WER) scoring against known reference audio.

## Author

Marco Fernández Llamas — [github.com/mfernl](https://github.com/mfernl)

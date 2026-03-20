<!-- verified-against: gpu-worker v41 -->
<!-- Patent notice: Dual-model transcription pipeline (US 64/010,891), embedding-based rejection (US 64/011,207). Licensed under Apache 2.0 with royalty-free patent grant. See PATENTS. -->

# GPU Worker Reference

The GPU worker (`gpu-worker/`) is a unified Docker container running all ML models on a single GPU. The same image runs on RunPod cloud (serverless) and locally on WSL.

---

## Architecture

```
gpu-worker container
├── entrypoint.sh
│   ├── RUNPOD_POD_ID set? → handler.py (runpod.serverless.start)
│   └── else → local_server.py (FastAPI on port 5125)
├── handler.py (all GPU actions)
├── local_server.py (FastAPI wrapper)
├── /voices/ (WAV reference samples)
└── /voices/.cache/ (pre-baked .pt files)
```

**Platform detection:** `RUNPOD_POD_ID` or `RUNPOD_ENDPOINT_ID` → RunPod cloud mode. Neither → local GPU mode (FastAPI).

---

## Actions (7 handlers in handler.py)

### Generator handlers (yield progress, then result)

| Action | Input | Output | Model |
|--------|-------|--------|-------|
| `transcribe` | audio_b64, language, preset, speakers? | text, segments, words | faster-whisper + WhisperX |
| `diarize` | audio_b64, word_segments, agent_id, speakers? | speaker_segments, speaker_map | PyAnnote |
| `synthesize_stream` | text, voice, conditionals_b64?, TTS params, split_chars | audio_b64 chunks (per sentence) | Chatterbox TTS |

### Sync handlers (return single result)

| Action | Input | Output | Model |
|--------|-------|--------|-------|
| `synthesize` | text, voice, conditionals_b64?, TTS params | audio_b64 (WAV) | Chatterbox TTS |
| `compute_conditionals` | wav_b64, exaggeration | conditionals_b64 (.pt) | Chatterbox |
| `enroll` | agent_id, speakers (name → npy_b64) | enrolled count | PyAnnote |
| `embed` | texts[], task_type? | embeddings[] (768D) | nomic-embed-text-v1.5 |

---

## Transcription Pipeline (Dual Model)

Two models run in parallel via `ThreadPoolExecutor(max_workers=2)`:

```
Audio (WebM/MP3/WAV)
├── Thread 1: faster-whisper (WHAT — content)
│   medium.en (fast) or large-v3 (quality)
│   → segments with timestamps
└── Thread 2: WhisperX (word alignment)
    → words with precise timestamps
```

**Presets:**

| Preset | faster-whisper | WhisperX | VRAM |
|--------|---------------|----------|------|
| `english-fast` | medium.en | small.en | ~3 GB |
| `english-quality` | large-v3 | medium.en | ~6 GB |
| `multilingual` | large-v3 | large-v3-turbo | ~7 GB |

---

## TTS: Chatterbox Pool

**Problem:** Chatterbox model instances have mutable state (`model.conds`). Concurrent calls with different voices corrupt output.

**Solution:** Pool of isolated instances, each with its own `threading.Lock()`:

```python
_chatterbox_pool = []  # List of (model, lock) tuples

def acquire_chatterbox():
    idx = _pool_index % len(_chatterbox_pool)
    _pool_index += 1
    return _chatterbox_pool[idx]
```

**Pool size:** Auto-detected: `(VRAM - 4GB) / 3.5GB / 2`, max 6.
- RunPod 48GB A40: 3 instances
- Local 24GB 4090: 1 instance

**Streaming TTS** (`synthesize_stream`):
1. Split text at sentence boundaries
2. Merge short fragments
3. Generate each sentence independently
4. Yield audio chunks as they complete — low latency on first chunk

**Voice Conditionals** (`.pt` files):
- Pre-computed Chatterbox embeddings (voice identity + prosody)
- Load in <100ms vs ~2s from WAV
- Fallback chain: `conditionals_b64` param → cached `.pt` → compute from `.wav` → `zara.wav`

---

## Models: Baked Into Image

All models are pre-downloaded into the Docker image at build time. Zero network calls at runtime.

| Model | Purpose |
|-------|---------|
| faster-whisper medium.en + large-v3 | Transcription |
| WhisperX small.en + medium.en + large-v3-turbo | Word alignment |
| Chatterbox TTS | Voice synthesis |
| nomic-embed-text-v1.5 | Semantic embeddings (768D) |
| PyAnnote diarization-3.1 + embedding | Speaker identification |

**Startup prewarm:** `local_server.py._prewarm()` loads all models from disk to GPU before accepting requests. First request latency equals any subsequent request.

---

## Local Server (local_server.py)

FastAPI wrapper matching RunPod's API contract:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/runsync` | POST | Synchronous — blocks until complete |
| `/run` | POST | Async — returns `{id}`, poll `/stream/{id}` |
| `/stream/{job_id}` | GET | Poll for streaming chunks |
| `/status/{job_id}` | GET | Job status |
| `/health` | GET | GPU health: name, VRAM free, voices |

---

## VRAM Budget (24 GB 4090 local)

```
Chatterbox TTS (1 instance):   ~3.5 GB
faster-whisper (medium.en):    ~2 GB
faster-whisper (large-v3):     ~5 GB
WhisperX + PyAnnote:           ~5 GB
nomic-embed-text:              ~0.5 GB
Headroom:                      ~8 GB
```

---

## Build & Deploy

```bash
cd gpu-worker
./build.sh v41

# Push to Docker Hub
docker push yaniv256/gpu-worker:v41

# Run locally
docker run -d --gpus all -p 5125:5125 \
  -v /path/to/voices:/voices:ro \
  yaniv256/gpu-worker:v41
```

**TTS Parameters** (chatterbox-tts ≥0.1.6): `temperature`, `repetition_penalty`, `min_p`, `top_p`, `exaggeration`, `cfg_weight`. All optional.

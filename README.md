# S2T · Tamil (தமிழ்)

Streaming speech-to-text service for **Tamil** built on Zipformer2 (sherpa-onnx) + FastAPI.
One container, one language, two endpoints: live WebSocket streaming and offline batch transcription.

## Quick start

### Docker (recommended)

```bash
# Build
docker build -t s2t-tamil .

# Run — the model is baked into the image; same hardening as production
docker run -d \
  -p 6007:6007 \
  --read-only --cap-drop=ALL --security-opt no-new-privileges \
  --name s2t-tamil \
  s2t-tamil
```

The image is a two-stage build: packages are installed on `python:3.13-slim-trixie`
and copied onto distroless `python3-debian13` (no shell, no package manager, no
pip, non-root uid 65532). About 210 MB plus the 72 MB model, versus 309 MB
without the model for the previous `python:3.12-slim` image.

### Without Docker

```bash
pip install -r requirements.txt
python -m app          # reads ./models/tamil/
```

## Model files

Place the four model files in `models/tamil/`:

```
models/
  tamil/
    encoder.onnx
    decoder.onnx
    joiner.onnx
    tokens.txt
```

The Docker build copies `models/tamil/` into the image at `/models/tamil`.

## API

### Live streaming — `WS /v1/stream?sample_rate=16000`

Send 16-bit little-endian mono PCM frames (~100 ms each). Send text frame `eof` to flush.

```jsonc
{"type": "ready",   "session": "…", "language": "tamil"}
{"type": "partial", "text": "…"}
{"type": "final",   "text": "…", "start": 0.0, "end": 2.1}
{"type": "done",    "text": "…", "segments": […]}
```

### Offline batch — `POST /v1/transcribe`

```bash
curl -F files=@audio.wav http://localhost:6007/v1/transcribe
```

```json
{
  "language": "tamil",
  "results": [
    {"filename": "audio.wav", "duration": 10.5, "text": "…",
     "segments": [{"text": "…", "start": 0.0, "end": 10.5}]}
  ]
}
```

Accepts WAV, FLAC, OGG, MP3 at any sample rate (mono or stereo).

### Ops

```bash
curl http://localhost:6007/health    # liveness + readiness
curl http://localhost:6007/metrics  # Prometheus
```

## Configuration

| Env var | Default | Description |
|---|---|---|
| `S2T_LANGUAGE` | `tamil` | model folder name |
| `S2T_MODEL_DIR` | `models` (`/models` in Docker) | local model root |
| `S2T_DECODE_WORKERS` | `2` | set to container vCPU count |
| `S2T_MAX_STREAMS` | `40` | max concurrent live streams |
| `S2T_MAX_OFFLINE` | `8` | max concurrent offline files |
| `S2T_PROVIDER` | `cpu` | `cuda` requires a CUDA sherpa-onnx build |
| `S2T_PORT` | `8000` (`6007` in Docker) | listen port (Cloud Run sets `8080`) |
| `S2T_MAX_UPLOAD_MB` | `100` | per file; Cloud Run caps a whole request at 32 MB, so production uses `30` |
| `S2T_CORS_ORIGIN_REGEX` | empty | browser origins allowed (CORS + WebSocket `Origin` check), full-match regex; empty disables both |

`/docs`, `/redoc` and `/openapi.json` are disabled.

## Deployment

Production runs on Cloud Run as `stt-tamil`, behind
`https://stt-agent.chatbucket.chat/stt-tamil` (global HTTPS load balancer +
Cloud Armor WAF, Google-managed certificate). The infrastructure is Terraform in
[gke-infra-terraform](https://github.com/nandak99-coin/gke-infra-terraform)
(`modules/stt-cloudrun`, `envs/prod/stt.tf`); only `/stt-tamil/v1/stream`,
`/stt-tamil/v1/transcribe` and `/stt-tamil/health` are reachable, and only from
the allowed origins.

1. **GCP Build & Push** builds, smoke-tests (read-only, no capabilities,
   transcribes `tests/sample.wav`) and Trivy-scans the image on every push and
   pull request, and pushes it to Artifact Registry as `<short sha>` on `main`.
2. **GCP Deploy (Cloud Run)** (manual, `production` environment) rolls that tag
   out; traffic moves only once the new revision passes its startup probe, and
   goes back to the previous revision if the public health check fails.

```
wss://stt-agent.chatbucket.chat/stt-tamil/v1/stream?sample_rate=16000
curl -H 'Origin: https://todozee.chatbucket.chat' -F files=@audio.wav https://stt-agent.chatbucket.chat/stt-tamil/v1/transcribe
```

## Health check

`GET /health` (liveness + readiness). Cloud Run probes it; locally use `make health`.

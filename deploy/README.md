# Combo Deployment: speaches + Wyoming + subgen

Unified Whisper deployment: one model loaded once on NVIDIA GPU, serving three protocols.

## Architecture

```
  Home Assistant          Bazarr / curl              Plex / Jellyfin
       |                       |                          |
  Wyoming (tcp:10300)     OpenAI API (http:8000)     webhooks (http:9000)
       v                       v                          v
 wyoming-openai  ------>  speaches (GPU)  <------     subgen (CPU)
                          large-v3, int8_float16
                          num_workers=2
```

| Service | Role | GPU | Port |
|---------|------|-----|------|
| **speaches** | Whisper engine (single model load) | NVIDIA | 8000 |
| **wyoming-openai** | Wyoming protocol bridge for Home Assistant | no | 10300 |
| **subgen** | Bazarr ASR + Plex/Jellyfin subtitle webhooks | no | 9000 |

## Prerequisites

- Docker with NVIDIA Container Toolkit (`nvidia-smi` works inside containers)
- ~4 GB VRAM free (large-v3 int8_float16 with 2 workers)

## Setup

### 1. Configure environment

```bash
cd deploy
cp .env.example .env
```

Edit `.env` with your actual media paths and server tokens.

The `WHISPER_PROMPT` is pre-filled with German smart-home vocabulary for Home Assistant.

### 2. Start speaches first

The first start downloads the large-v3 model (~3 GB). Wait for it to become healthy.

```bash
docker compose up -d speaches
docker compose logs -f speaches   # wait for "Application startup complete"
```

Verify GPU usage:
```bash
nvidia-smi
```

### 3. Test the OpenAI API

```bash
curl -X POST http://localhost:8000/v1/audio/transcriptions \
  -F "file=@test.wav" \
  -F "model=Systran/faster-whisper-large-v3" \
  -F "language=de" \
  -F "response_format=json"
```

### 4. Bring up all services

```bash
docker compose up -d
```

### 5. Configure integrations

**Home Assistant**: Add Wyoming integration pointing to `<docker-host-ip>:10300`.

**Bazarr**: Set Whisper provider endpoint to `http://<docker-host-ip>:9000`.

**Plex/Jellyfin**: Configure webhooks to `http://<docker-host-ip>:9000/plex` or `/jellyfin`.

## Subgen external API mode

This fork adds `WHISPER_API_URL` support to subgen. When set, subgen delegates all transcription to the remote speaches API instead of loading a local Whisper model. All existing subgen features (queue, dedup, webhooks, Bazarr /asr) work unchanged.

New environment variables:
- `WHISPER_API_URL` - OpenAI-compatible transcription endpoint (e.g. `http://speaches:8000/v1/audio/transcriptions`)
- `WHISPER_API_MODEL` - model name to request (default: `Systran/faster-whisper-large-v3`)
- `WHISPER_API_LANGUAGE` - default language code (e.g. `de`)

When `WHISPER_API_URL` is not set, subgen falls back to its normal local-model behavior.

## Pulling upstream changes

```bash
git fetch upstream
git merge upstream/main
```

The `deploy/` directory doesn't exist upstream so it will never conflict.

## Troubleshooting

**speaches won't start**: Check `docker compose logs speaches`. Ensure NVIDIA runtime is available (`docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi`).

**Wyoming not connecting**: Ensure speaches is healthy first (`docker compose ps`). wyoming-openai waits for the healthcheck.

**Subtitles not generating**: Check subgen logs (`docker compose logs subgen`). Verify it shows "External Whisper API mode" at startup.

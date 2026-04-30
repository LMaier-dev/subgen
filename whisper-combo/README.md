# Whisper Combo: STT + TTS + Wyoming + Subtitles

Single NVIDIA GPU serving speech-to-text, text-to-speech, Home Assistant voice, and subtitle generation.

## Architecture

```
  Home Assistant (DE)    Home Assistant (EN)    Bazarr / curl        Plex / Jellyfin
       |                      |                      |                     |
  Wyoming tcp:10300      Wyoming tcp:10301      OpenAI API :8000      webhooks :9000
       v                      v                      v                     v
 wyoming-openai-de      wyoming-openai-en      speaches (GPU)  <--   subgen (CPU)
   STT: speaches          STT: speaches         large-v3
   TTS: pockettts-de      TTS: speaches/Kokoro  int8_float32
              |                   |               num_workers=2
              v                   v
      Pocket TTS (CPU)     Kokoro 82M (GPU)
      german_24l           ONNX via speaches
      voice cloning        28 English voices
```

| Service | Role | GPU | Port |
|---------|------|-----|------|
| **speaches** | Whisper STT + Kokoro TTS (single GPU model load) | NVIDIA | 8000 |
| **pockettts-de** | German TTS with voice cloning | CPU | 49112 |
| **wyoming-openai-de** | Wyoming bridge for HA (German) | no | 10300 |
| **wyoming-openai-en** | Wyoming bridge for HA (English) | no | 10301 |
| **subgen** | Bazarr ASR + Plex/Jellyfin subtitle webhooks | no | 9000 |

## Prerequisites

- Docker with NVIDIA Container Toolkit
- ~4 GB VRAM free (large-v3 int8_float32 + Kokoro ONNX, 2 workers)
- ~4 GB RAM for Pocket TTS (CPU)
- HuggingFace token with access to `kyutai/pocket-tts` (accept license at https://huggingface.co/kyutai/pocket-tts)

## Setup

### 1. Configure environment

```bash
cp .env.example .env
# Edit with your actual values
```

Required variables:
- `HF_TOKEN` - HuggingFace token (for Pocket TTS voice cloning weights)
- `WHISPER_PROMPT` - German smart-home vocabulary for better HA recognition
- `TV` / `MOVIES` - media library paths for subgen
- `PLEX_TOKEN` / `PLEX_SERVER` - Plex integration
- `JELLYFIN_TOKEN` / `JELLYFIN_SERVER` - Jellyfin integration

### 2. Deploy

```bash
docker compose up -d
```

First start downloads models (~3 GB for Whisper, ~200 MB for Pocket TTS per language). Wait for healthchecks to pass.

---

## Usage: Home Assistant (Wyoming)

Add Wyoming integrations in HA under Settings > Devices & Services > Add Integration > Wyoming.

### German voice pipeline
- **Address:** `<docker-host-ip>`
- **Port:** `10300`
- STT: Whisper large-v3 (GPU)
- TTS: Pocket TTS `german_24l` (CPU, voice cloning capable)

### English voice pipeline
- **Address:** `<docker-host-ip>`
- **Port:** `10301`
- STT: Whisper large-v3 (GPU)
- TTS: Kokoro 82M (GPU)

Then assign each to a Voice Assistant under Settings > Voice Assistants.

---

## Usage: OpenAI-compatible API

All endpoints are available directly on the Docker host, or via the Caddy gateway at `ai.senseapp.net` (requires `Authorization: Bearer <key>` header, adds `/en/` and `/de/` language prefixes for TTS).

### Speech-to-Text (any language)

```bash
# Direct
curl -X POST http://<host>:8000/v1/audio/transcriptions \
  -F "file=@audio.wav" \
  -F "model=Systran/faster-whisper-large-v3" \
  -F "language=de" \
  -F "response_format=json"

# Via gateway
curl -X POST https://ai.senseapp.net/v1/audio/transcriptions \
  -H "Authorization: Bearer $API_KEY" \
  -F "file=@audio.wav" \
  -F "model=Systran/faster-whisper-large-v3" \
  -F "language=de" \
  -F "response_format=json"
```

Supported `response_format`: `json`, `text`, `srt`, `vtt`, `verbose_json`

### Text-to-Speech: English (Kokoro)

```bash
# Direct
curl -X POST http://<host>:8000/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model": "speaches-ai/Kokoro-82M-v1.0-ONNX", "voice": "af_heart", "input": "Hello world"}' \
  --output speech.mp3

# Via gateway (note /en/ prefix)
curl -X POST https://ai.senseapp.net/en/v1/audio/speech \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "speaches-ai/Kokoro-82M-v1.0-ONNX", "voice": "af_heart", "input": "Hello world"}' \
  --output speech.mp3
```

#### English voices (Kokoro)

| Female (US) | Male (US) | Female (UK) | Male (UK) |
|------------|-----------|-------------|-----------|
| af_heart | am_adam | bf_alice | bm_daniel |
| af_alloy | am_echo | bf_emma | bm_fable |
| af_aoede | am_eric | bf_isabella | bm_george |
| af_bella | am_fenrir | bf_lily | bm_lewis |
| af_jessica | am_liam | | |
| af_kore | am_michael | | |
| af_nicole | am_onyx | | |
| af_nova | am_puck | | |
| af_river | am_santa | | |
| af_sarah | | | |
| af_sky | | | |

### Text-to-Speech: German (Pocket TTS)

```bash
# Direct
curl -X POST http://<host>:49112/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model": "german_24l", "voice": "jean", "input": "Hallo Welt, ich bin ein Test."}' \
  --output speech.mp3

# Via gateway (note /de/ prefix)
curl -X POST https://ai.senseapp.net/de/v1/audio/speech \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "german_24l", "voice": "jean", "input": "Hallo Welt"}' \
  --output speech.mp3
```

#### German voices (Pocket TTS built-in)

alba, anna, azelma, bill_boerst, caro_davy, charles, cosette, eponine, eve, fantine, george, jane, javert, jean, marius, mary, michael, paul, peter_yearsley, stuart_bell, vera

#### Cloned voices

Any WAV file placed in `/app/voices/` becomes available as a voice. Currently: `bernd_das_brot`

### Adding a cloned voice

```bash
# Copy a WAV file (mono, 16-bit, 22050 Hz recommended, 10-30s of clean speech)
docker cp my_voice.wav pockettts-de:/app/voices/

# First request with new voice triggers embedding extraction (~10s)
curl -X POST http://<host>:49112/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model": "german_24l", "voice": "my_voice", "input": "Test der neuen Stimme."}' \
  --output test.mp3

# Subsequent calls use cached embeddings (instant)
```

---

## Usage: Subtitles (subgen)

### Bazarr
Set Whisper provider endpoint to `http://<docker-host-ip>:9000`.

### Plex
Configure webhook to `http://<docker-host-ip>:9000/plex`.

### Jellyfin
Configure webhook to `http://<docker-host-ip>:9000/jellyfin`.

---

## Parallel request handling

- **speaches**: `num_workers=2` means two transcription jobs run in parallel on GPU. A voice command (~3s) completes in ~1-2s even while a movie is being transcribed.
- **subgen**: `CONCURRENT_TRANSCRIPTIONS=2` sends up to 2 media files to speaches simultaneously.
- **Pocket TTS**: Runs on CPU independently of GPU workloads.

---

## Troubleshooting

**speaches won't start**: Check `docker compose logs speaches`. Ensure NVIDIA runtime is available.

**Pocket TTS "could not download weights"**: Accept the license at https://huggingface.co/kyutai/pocket-tts and verify `HF_TOKEN` is set in `.env`. The weights only need to download once per language (cached in `pockettts-cache` volume).

**Wyoming not connecting**: Ensure speaches is healthy first (`docker compose ps`). Wyoming bridges wait for healthchecks.

**No cloned voice output**: Check that the WAV file exists in `/app/voices/` inside the container (not just on the host). First call takes ~10s to compute embeddings.

# Open-Source AI Customer Service Voice Agent

Production-oriented reference implementation for a browser-based customer-service agent with:

- React + TypeScript conversational UI
- Full-duplex binary audio over WebSocket
- AudioWorklet microphone capture at 16 kHz PCM16
- Silero VAD endpointing and barge-in detection
- faster-whisper speech-to-text
- Ollama-hosted open-weight chat and embedding models
- Qdrant retrieval-augmented generation
- Kokoro ONNX text-to-speech
- PostgreSQL conversation/audit persistence
- Prometheus metrics and structured logs
- Docker Compose local deployment

The runtime contains no OpenAI dependency. GPT-5.6, GPT-5.4, and GPT-5.5 are used only as an optional software-development workflow, documented in `docs/AI_DEVELOPMENT_WORKFLOW.md`.

## Architecture modes

### Mode A — controlled turn-taking
The client continuously captures audio, but the agent waits for the user utterance to end before generating a response. This is the lowest-risk production baseline.

### Mode B — full duplex with interruption
The microphone remains active while the agent speaks. When Silero detects new user speech, the backend cancels the active LLM/TTS generation, emits `assistant_interrupted`, and the browser immediately clears queued audio. This codebase enables Mode B by default.

For public telephone/SIP workloads, use the same domain services behind self-hosted LiveKit/WebRTC rather than exposing raw audio WebSockets directly. See `docs/ARCHITECTURE.md`.

## Prerequisites

- Docker Desktop or Docker Engine with Compose
- Ollama installed on the host
- 16 GB RAM minimum for a small local model; GPU strongly recommended for concurrent sessions
- Modern Chromium, Edge, Firefox, or Safari browser

## 1. Prepare configuration

```bash
make setup
```

Review `.env`. For a CPU-only developer machine, keep:

```dotenv
STT_MODEL=small.en
STT_DEVICE=cpu
STT_COMPUTE_TYPE=int8
OLLAMA_CHAT_MODEL=qwen3:14b
```

For NVIDIA production nodes, use a GPU-appropriate STT model and set:

```dotenv
STT_MODEL=distil-large-v3
STT_DEVICE=cuda
STT_COMPUTE_TYPE=float16
```

## 2. Install Ollama models

```bash
ollama pull qwen3:14b
ollama pull nomic-embed-text
ollama serve
```

Any Ollama chat model can be selected through `OLLAMA_CHAT_MODEL`. Validate tool use, latency, multilingual behavior, and license terms before production approval.

## 3. Download the TTS model

```bash
make models
```

This downloads the Kokoro v1.0 ONNX model and voice bundle into `./models`. The model is Apache-2.0 and the `kokoro-onnx` runtime is MIT licensed.

## 4. Start the platform

```bash
make up
make seed
```

Open:

- Voice UI: `http://localhost:3000`
- Backend health: `http://localhost:8000/health/ready`
- Qdrant: `http://localhost:6333/dashboard`

The browser must grant microphone permission. Headphones reduce acoustic echo and false barge-in events.

## 5. Observability

```bash
make observability
```

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3001` (`admin` / `admin` for local development only)

## WebSocket protocol

Endpoint:

```text
/ws/voice/{session_id}?token=<jwt-when-auth-enabled>
```

Client-to-server:

- Binary: raw little-endian PCM16 mono at 16 kHz
- JSON `{"type":"ping"}`
- JSON `{"type":"text","text":"..."}` for test automation
- JSON `{"type":"playback_complete","generation_id":7}` after all scheduled audio for a completed generation has ended
- JSON `{"type":"stop"}`

Server-to-client JSON events:

- `session_ready`
- `speech_started`
- `transcript_partial`
- `transcript_final`
- `assistant_text_delta`
- `assistant_speaking`
- `assistant_interrupted`
- `assistant_done`
- `handoff_requested`
- `error`

Server audio is binary with a 12-byte little-endian header:

```text
bytes 0..3   ASCII AUD0
bytes 4..7   uint32 sample rate
bytes 8..11  uint32 generation id
bytes 12..N  PCM16 mono samples
```

## Production deployment rules

1. Terminate TLS at an ingress proxy and expose only `wss://`.
2. Set `AUTH_DISABLED=false`; issue short-lived JWTs and validate tenant/customer claims.
3. Run one model-bearing backend worker per GPU allocation. Scale replicas horizontally; do not use multiple Uvicorn workers that duplicate model memory inside one container.
4. Apply session quotas at ingress and application layers.
5. Separate PII from transcripts, encrypt storage, enforce retention, and redact logs.
6. Pin container images and Python/Node lockfiles after qualification.
7. Run STT, LLM, TTS, and Qdrant on private networks; no direct public ports.
8. Use a one-time WebSocket ticket or secure session cookie in production; do not expose a reusable bearer token in URL logs.
9. Add a human handoff adapter before enabling the agent for real customers.
10. Evaluate task completion, hallucination, policy compliance, WER, interruption accuracy, and latency on representative calls before rollout.
11. Treat all model licenses and training-data obligations as a formal release gate.

## Target latency budget

| Stage | Target |
|---|---:|
| VAD end-of-turn | 350–700 ms |
| Final STT | 250–900 ms GPU |
| Retrieval | <150 ms |
| First LLM token | 300–1,200 ms |
| First TTS audio | 250–700 ms |
| Barge-in stop | <200 ms |

CPU-only deployments will usually exceed these figures. Full-duplex customer support should be sized on measured p95 and p99 latency, not average latency.

## Repository structure

```text
backend/                 FastAPI voice orchestration and model adapters
frontend/                React voice client and audio pipeline
data/                    Seed knowledge base
models/                  Downloaded Kokoro artifacts, excluded from Git
deploy/                  Prometheus and production notes
docs/                    Architecture, technical design, pseudocode, AI workflow
scripts/                 Model bootstrap scripts
```

## Known boundaries

- faster-whisper is used in an incremental utterance adapter: provisional text is generated from snapshots and final text is committed at VAD end. Replace the `SpeechToText` adapter with Whisper-Streaming or sherpa-onnx when token-level streaming ASR is mandatory.
- Browser acoustic echo cancellation varies by device. WebRTC/LiveKit provides a better transport for internet and telephony-grade media.
- The included handoff is an event contract, not a CRM/contact-center integration.

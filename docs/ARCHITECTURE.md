# Solution Design and Technical Architecture

## 1. Business objective

Provide a voice-first customer-service agent that answers grounded questions, preserves context, performs controlled support actions, and transfers to a human when confidence or policy conditions require escalation.

## 2. Design principles

- Open-source runtime with replaceable model adapters
- Streaming-first, cancellation-safe async design
- Deterministic policy controls outside the LLM
- Retrieval grounding with source metadata
- Tenant and customer isolation
- Human escalation as a first-class outcome
- Observable latency, errors, and conversation quality
- Fail closed for authentication and high-risk actions

## 3. Logical architecture

```text
┌──────────────────────────────── Browser ────────────────────────────────┐
│ React UI                                                               │
│  ├─ AudioWorklet: mic -> PCM16/16 kHz                                 │
│  ├─ WebSocket control + binary audio                                  │
│  ├─ Audio queue: PCM16 -> Web Audio                                   │
│  └─ Barge-in handler: clear queued agent audio                        │
└───────────────────────────────┬────────────────────────────────────────┘
                                │ WSS
┌───────────────────────────────▼────────────────────────────────────────┐
│ FastAPI Voice Gateway                                                   │
│  ├─ JWT/session/rate validation                                        │
│  ├─ Per-session state machine                                          │
│  ├─ Silero VAD + endpointing                                           │
│  ├─ Cancellation generation token                                      │
│  ├─ faster-whisper STT                                                  │
│  ├─ Policy and escalation engine                                       │
│  ├─ RAG orchestration                                                   │
│  ├─ Ollama streaming chat                                               │
│  └─ Kokoro sentence-level streaming TTS                                │
└──────────────┬─────────────────────┬──────────────────────┬─────────────┘
               │                     │                      │
       ┌───────▼───────┐     ┌──────▼──────┐      ┌───────▼────────┐
       │ PostgreSQL     │     │ Qdrant       │      │ Ollama          │
       │ sessions/audit │     │ knowledge    │      │ LLM/embeddings  │
       └───────────────┘     └─────────────┘      └────────────────┘
```

## 4. Session state machine

```text
CONNECTED
  -> LISTENING
  -> USER_SPEAKING
  -> TRANSCRIBING
  -> RETRIEVING
  -> GENERATING
  -> AGENT_SPEAKING
  -> LISTENING

AGENT_SPEAKING + user speech start
  -> CANCEL_GENERATION
  -> CLEAR_CLIENT_AUDIO
  -> USER_SPEAKING

Any state + policy escalation
  -> HANDOFF_REQUESTED
```

Every generated response has a monotonically increasing `generation_id`. Audio packets from an older generation are ignored by the client. Cancellation increments the generation id and cancels both the Ollama stream and TTS worker.

## 5. V1 and V2 roadmap

### V1: web voice agent

- Secure WebSocket
- Automatic VAD endpointing
- Incremental transcript snapshots
- RAG and streaming text
- Sentence-level streaming TTS
- Barge-in cancellation
- Browser echo cancellation

This repository implements V1.

### V2: phone-call-grade full duplex

Use self-hosted LiveKit Server and LiveKit Agents as the media plane:

```text
Browser / SIP trunk
       │ WebRTC / SIP
       ▼
Self-hosted LiveKit
       │ audio tracks + data channel
       ▼
Agent worker
       ├─ streaming STT
       ├─ turn detector / VAD
       ├─ support orchestrator
       ├─ Ollama/vLLM
       └─ streaming TTS
```

Why change the transport in V2:

- WebRTC provides jitter buffering, congestion control, packet-loss recovery, and mature acoustic-processing paths.
- SIP integration requires a media server rather than a browser-specific socket protocol.
- Agent workers can be scheduled independently from the web API.
- Horizontal media routing is cleaner than sticky raw WebSocket sessions.

The domain services in this repository (`retrieval`, `llm`, `store`, policy rules) can be reused behind LiveKit.

## 6. Retrieval design

Qdrant payload schema:

```json
{
  "document_id": "refund-policy",
  "title": "Refund policy",
  "content": "...",
  "source": "support-policy",
  "tenant_id": "default",
  "locale": "en-US",
  "effective_from": "2026-01-01",
  "effective_to": null,
  "security_groups": ["customers"]
}
```

Production filtering must include tenant, locale, effective date, product, and security group before semantic scoring. Never rely on prompt text to enforce access control.

## 7. Safety and policy layer

The policy engine executes before and after the LLM:

- Authentication and customer identity verification
- PII redaction and retention policy
- Prompt-injection filtering of retrieved content
- Action allowlist and parameter validation
- Financial/refund approval thresholds
- Low-confidence and repeated-failure escalation
- Threat, abuse, self-harm, legal, or regulated-domain routing as required by the business
- Output checks for unsupported commitments or fabricated policy

LLM output must never directly execute a backend action. It produces an action proposal; deterministic application code validates and executes it.

## 8. Availability and scaling

- One active WebSocket remains pinned to one backend replica.
- A backend replica should host one STT and one TTS model instance; capacity is governed by inference concurrency, not HTTP request throughput.
- Use Kubernetes pod disruption budgets and graceful connection draining.
- Postgres and Qdrant require backups, tested restore procedures, and multi-zone storage.
- Ollama is suitable for local and controlled deployments. For high concurrency, substitute an OpenAI-compatible open-source inference server such as vLLM behind the same adapter.

## 9. Security zones

```text
Internet
  -> WAF / Ingress / TLS
  -> Voice Gateway subnet
  -> Private AI services subnet
  -> Data subnet
```

Mandatory controls:

- WSS only
- Short-lived JWT with tenant/customer/session claims
- Origin allowlist
- Maximum message size and audio duration
- Connection/session rate limits
- No model or database ports exposed publicly
- Secrets from a secret manager, never `.env` in production
- Immutable audit trail for support actions
- Transcript encryption and configurable deletion

## 10. Non-functional acceptance criteria

- p95 barge-in stop under 200 ms in supported network conditions
- p95 end-of-user-turn to first audible agent speech under 2.5 seconds on qualified hardware
- No cross-tenant retrieval in automated security tests
- 100% of high-risk actions require deterministic validation
- Graceful behavior when Qdrant, Ollama, STT, TTS, or Postgres is unavailable
- Load test demonstrates stable memory with long-lived sessions
- Conversation evaluation suite covers grounding, policy, escalation, and task completion

# Detailed Technical Design

## Component responsibilities

### React client

- Requests microphone permission with browser echo cancellation enabled.
- Runs an AudioWorklet so audio capture does not block the UI thread.
- Resamples microphone data to 16 kHz and sends PCM16 binary frames.
- Displays partial/final user transcripts and streamed assistant text.
- Schedules PCM audio through Web Audio.
- Immediately stops scheduled playback on `assistant_interrupted`.

### FastAPI voice gateway

- Owns WebSocket authentication and lifecycle.
- Creates one `VoiceSession` object per connection.
- Serializes all socket writes through an async lock.
- Accepts binary audio continuously while LLM/TTS tasks run.
- Cancels response tasks when VAD reports user speech.
- Enforces utterance/session duration limits.

### VAD

- Accumulates arbitrary WebSocket frame sizes into 512-sample windows.
- Uses one stateful Silero `VADIterator` per session.
- Emits start/end events and resets after a committed utterance.

### STT

- Uses a shared faster-whisper model.
- Runs blocking inference in a worker thread.
- Produces provisional transcripts from bounded snapshots.
- Produces the authoritative transcript only after end-of-turn.

### RAG

- Obtains query embeddings from Ollama.
- Searches Qdrant with mandatory payload filters.
- Formats bounded context with source labels.
- Returns no context rather than failing the whole conversation when Qdrant is unavailable.

### LLM

- Calls Ollama `/api/chat` with streaming enabled.
- Uses a strict customer-support system instruction.
- Emits text deltas and sentence segments.
- Does not execute tools directly.

### TTS

- Uses Kokoro ONNX through a lazy singleton.
- Synthesizes short sentence segments in a worker thread.
- Converts float audio to PCM16 and sends generation-tagged packets.
- Stops when the parent response task is cancelled.

### Persistence

PostgreSQL tables:

- `conversation_turn`: user and assistant transcript, session, tenant, timing metadata
- `support_event`: interruption, handoff, errors, tool/action audit

The application continues in degraded mode when transcript persistence fails, but any business action should fail closed if its audit record cannot be written.

## Concurrency model

```text
WebSocket receive loop
  ├─ handle audio synchronously (<1 ms VAD work per frame)
  ├─ schedule partial STT task (bounded by lock)
  └─ schedule final response task

Final response task
  ├─ final STT
  ├─ persistence
  ├─ retrieval
  ├─ LLM stream producer
  └─ TTS queue consumer
```

There must be only one active final response task per session. A new user speech-start event cancels that task and invalidates its generation id. The browser acknowledges `playback_complete` only after the server has sent `assistant_done` and every audio source for that generation has ended; this closes the race between server generation completion and client playback completion.

## Failure behavior

| Failure | Behavior |
|---|---|
| Ollama unavailable | Emit recoverable error and request human handoff |
| Qdrant unavailable | Continue without retrieved context; answer only safe generic questions |
| Postgres unavailable | Continue conversation, emit audit warning; block state-changing tools |
| STT failure | Ask user to repeat; retain no invalid transcript |
| TTS failure | Continue streaming text; UI remains usable |
| WebSocket disconnect | Cancel inference and release session resources |
| Oversized utterance | Stop collection, transcribe bounded audio, tell user to use shorter statements |
| Repeated low confidence | Handoff |

## API contracts

### Health

```http
GET /health/live
GET /health/ready
GET /metrics
```

### WebSocket authentication

Development can use `AUTH_DISABLED=true`. Production uses:

```text
wss://support.example.com/ws/voice/{session_id}?token=<short-lived-jwt>
```

Required claims:

```json
{
  "sub": "customer-id",
  "tenant_id": "tenant-id",
  "session_id": "same-as-path",
  "exp": 1780000000,
  "aud": "voice-agent"
}
```

## Pseudocode

```text
on websocket connect:
    validate origin, JWT, tenant and session quota
    create per-session VAD and state
    send session_ready

while websocket open:
    message = receive()

    if message is PCM audio:
        vad_events = vad.process(message)

        if speech_started:
            if response_task is active:
                generation_id += 1
                cancel response_task
                send assistant_interrupted
            start utterance buffer with pre-roll
            send speech_started

        if user is speaking:
            append bounded audio
            periodically schedule provisional STT

        if speech_ended:
            snapshot utterance
            clear VAD/utterance state
            response_task = create_task(process_turn(snapshot))

async process_turn(audio):
    transcript = await stt.transcribe(audio)
    if transcript empty: return
    send transcript_final
    persist user turn

    if deterministic escalation rule matches:
        send handoff_requested
        return

    contexts = await qdrant.retrieve(transcript, tenant filters)
    messages = build_prompt(history, transcript, contexts)
    generation_id += 1

    start tts_worker(sentence_queue, generation_id)
    async for token in ollama.stream(messages):
        send assistant_text_delta(token, generation_id)
        append token to sentence buffer
        for completed_sentence in buffer:
            sentence_queue.put(completed_sentence)

    flush remaining sentence
    sentence_queue.put(STOP)
    await tts_worker
    persist assistant turn
    send assistant_done
```

## Test strategy

- Unit: sentence segmentation, protocol encoding, policy decisions, tenant filters
- Component: Ollama/Qdrant/Postgres adapters against disposable containers
- WebSocket contract: binary frames, interruption, disconnect, oversized input
- Audio: prerecorded noisy and accented utterances with expected transcripts
- Conversation: golden support scenarios and adversarial prompt injection
- Load: long-lived concurrent sockets, memory, GPU utilization, p95 stage latency
- Chaos: terminate Qdrant/Ollama/Postgres during active sessions
- Security: JWT tampering, origin bypass, tenant-filter bypass, oversized messages

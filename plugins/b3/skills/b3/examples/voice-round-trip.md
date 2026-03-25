# Example: Voice Round-Trip

What happens when a user says "Hello" and Claude responds with speech.

## The Flow (WebRTC — Primary Path)

```
1. Browser captures audio (browser/js/voice-record.js)
   └── WebRTC data channel → b3-gpu-rtc sidecar (port 5126)

2. Sidecar bridges to GPU worker (b3-gpu-rtc → local_server.py)
   └── HTTP POST /run or WS stream_start → handler.py
   └── GPU worker: faster-whisper + WhisperX → transcription

3. Sidecar returns result over data channel → browser
   └── Browser injects transcription into daemon via control channel

4. Daemon injects transcription into PTY (daemon/rtc.rs → PTY stdin)
   └── Claude sees: [MIC] yaniv: Hello

5. Claude calls voice_say() MCP tool (mcp/voice.rs)
   └── MCP → daemon web server → TtsStream event → browser

6. Browser dispatches TTS to GPU (browser/js/gpu.js)
   ├── WebRTC data channel → b3-gpu-rtc → local_server.py
   ├── Text chunked at sentence boundaries
   ├── Each chunk → GPU worker (Chatterbox TTS)
   └── Audio streamed back over data channel as chunks complete

7. Browser plays audio (browser/js/voice-play.js)
   └── LED chromatophore animates based on emotion (browser/js/led.js)
```

## The Flow (Cloudflare Tunnel — Fallback)

```
1. Browser captures audio (browser/js/voice-record.js)
   └── POST /api/agents/{id}/audio-upload → daemon via Cloudflare tunnel

2. Daemon receives audio, dispatches to GPU (daemon/web.rs)
   └── GPU worker: faster-whisper + WhisperX → transcription

3. Daemon injects transcription into PTY (bridge/puller.rs)
   └── Claude sees: [MIC] yaniv: Hello

4. Claude calls voice_say() MCP tool (mcp/voice.rs)
   └── MCP → daemon web server → TtsStream event → browser via WebSocket

5. Browser dispatches TTS to GPU (browser/js/gpu.js)
   ├── HTTP POST to GPU worker (local or RunPod)
   ├── Polls /stream/{jobId} for audio chunks
   └── Audio arrives as chunks complete

6. Browser plays audio (browser/js/voice-play.js)
   └── LED chromatophore animates based on emotion
```

## Timeline (WebRTC)

```
0.0s    User says "Hello"
0.8s    VAD detects end of speech
0.9s    Audio sent over WebRTC data channel to GPU sidecar
1.5s    GPU returns transcription (peer-to-peer, no server hop)
1.6s    "[MIC] yaniv: Hello" injected into PTY
2.2s    Claude calls voice_say()
3.0s    First sentence audio arrives at browser over data channel
3.1s    Browser plays: "Hey! I can hear you."
```

## Streaming STT (Long Recordings)

For recordings longer than ~20 seconds, the browser streams partial transcriptions:

```
0.0s     Recording starts, audio chunks sent continuously
20.0s    Segment boundary: browser finds quiet point (3s search zone)
20.5s    First segment sent to GPU for transcription
21.5s    Partial result injected as marquee text in voice bar
22.0s    Recording continues, new audio accumulates
40.0s    Second segment boundary
42.0s    Second partial injected
...
N.0s     User stops recording
N.1s     Finalize: remaining audio sent with expected_chunks count
N+1.0s   GPU waits for all chunks, transcribes final segment
N+1.5s   Full transcription assembled + garbage filter applied
N+2.0s   "[MIC] yaniv: <full transcription>" injected into PTY
```

**Key details:**
- `SEGMENT_TARGET_SEC = 20` — target segment length
- `SEARCH_ZONE_SEC = 3` — how far to search for a quiet point
- Chunk-count synchronization prevents race conditions on finalize
- Garbage filter (embedding-based) removes noise transcriptions on both short and streaming paths
- Transport locking prevents mixed WS/RTC corruption during a single stream

## GPU Routing

```
voice_say() called
    │
    ▼
Daemon sends TtsStream event to browser
    │
    ▼
Browser checks GPU WebRTC connection
    ├── Connected? → Send over data channel to sidecar
    │                 (sidecar bridges to local_server.py)
    └── Not connected? → HTTP POST to GPU worker
                          (local 2s timeout → RunPod fallback)
```

**Key:** Audio never routes through the EC2 server. All voice traffic is peer-to-peer: browser ↔ daemon (via WebRTC or tunnel) and browser ↔ GPU (via WebRTC sidecar or direct HTTP).

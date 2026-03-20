# Example: Voice Round-Trip

What happens when a user says "Hello" and Claude responds with speech.

## The Flow

```
1. Browser captures audio (browser/js/voice-record.js)
   └── POST /api/agents/{id}/audio-upload → daemon via Cloudflare tunnel

2. Daemon receives audio, dispatches to GPU (daemon/web.rs)
   └── GPU worker: faster-whisper + WhisperX → transcription

3. Daemon injects transcription into PTY (bridge/puller.rs)
   └── Claude sees: [MIC] yaniv: Hello

4. Claude calls voice_say() MCP tool (mcp/voice.rs)
   └── MCP → daemon web server at localhost:3100

5. Daemon dispatches TTS to GPU worker (daemon/web.rs)
   ├── Chunks text at sentence boundaries
   ├── Each chunk → GPU worker (Chatterbox TTS)
   └── Audio streamed to browser via WebSocket as chunks complete

6. Browser plays audio (browser/js/voice-play.js)
   └── LED chromatophore animates based on emotion (browser/js/led.js)
```

## Timeline

```
0.0s    User says "Hello"
0.8s    VAD detects end of speech
0.9s    Audio uploaded to daemon (peer-to-peer, no server hop)
1.8s    GPU returns transcription
1.9s    "[MIC] yaniv: Hello" injected into PTY
2.5s    Claude calls voice_say()
3.5s    First sentence audio arrives at browser
3.6s    Browser plays: "Hey! I can hear you."
```

**Key:** Audio never routes through the server. Browser → daemon → GPU → daemon → browser. All peer-to-peer via Cloudflare tunnel.

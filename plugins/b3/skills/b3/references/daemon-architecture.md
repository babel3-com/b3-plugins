<!-- verified-against: b3-cli@0.1.1014 -->
<!-- Patent notice: Multi-model transcription injection via bridge puller (US 64/010,891). Licensed under Apache 2.0 with royalty-free patent grant. See PATENTS. -->

# Daemon Architecture Reference

The b3 daemon (`crates/b3-cli/`) is both a CLI tool and a long-running background process.

---

## Module Layout

```
src/
├── main.rs              # CLI entry: clap args → command dispatch
├── config.rs            # Agent config loading/saving (~/.b3/)
├── http.rs              # Shared HTTP client utilities
├── logging.rs           # Logging setup
├── service.rs           # System service installation
├── cloudflared.rs       # Cloudflare tunnel binary management
├── commands/
│   ├── start.rs         # b3 start — launches the daemon
│   ├── stop.rs          # b3 stop — stops the daemon
│   ├── attach.rs        # b3 attach — attach terminal to running daemon
│   ├── login.rs         # b3 login — authenticate with server
│   ├── setup.rs         # b3 setup — first-run wizard
│   ├── set_password.rs  # b3 set-password — change agent password
│   ├── status.rs        # b3 status — daemon health
│   ├── hive.rs          # b3 hive — inter-agent messaging
│   ├── update.rs        # b3 update — self-update binary
│   ├── uninstall.rs     # b3 uninstall — remove installation
│   └── version.rs       # b3 version
├── daemon/
│   ├── server.rs        # Daemon orchestrator — PTY, tunnel, bridge, IPC
│   ├── web.rs           # Local web server, voice job reporting
│   ├── rtc.rs           # WebRTC peer connections, ICE, PeerRegistry
│   ├── gpu_client.rs    # GPU TTS client (local-first + RunPod fallback)
│   ├── info_archive.rs  # Persistent info panel storage (max 1000 entries)
│   ├── tts_archive.rs   # TTS message archive for offline replay (max 50)
│   ├── protocol.rs      # IPC message protocol
│   └── ipc.rs           # Unix socket IPC listener/client
├── bridge/
│   ├── pusher.rs        # Terminal output → server (POST /api/session-push)
│   ├── puller.rs        # Server events → PTY input (SSE /events)
│   ├── parser.rs        # PTY output parsing
│   └── mod.rs
├── pty/
│   ├── manager.rs       # PTY spawn, read, write, resize
│   ├── attach.rs        # Client attachment to running PTY
│   └── mod.rs
└── mcp/
    ├── voice.rs         # MCP server — 19 tools for Claude Code
    ├── manager.rs       # Dynamic MCP server loading/unloading
    ├── registry.rs      # MCP server registry
    └── mod.rs
```

---

## Related Crates

```
crates/
├── b3-common/           # Shared types: AgentEvent (~20 variants), SessionPush
├── b3-webrtc/           # WebRTC abstraction over datachannel-rs
│   ├── channel.rs       # Binary framing protocol + 16KB chunking
│   ├── peer.rs          # B3Peer wrapper, PeerEvent, PeerConfig
│   └── signaling.rs     # SDP/ICE types, TURN credential generation
└── b3-gpu-rtc/          # GPU WebRTC sidecar (separate binary)
```

---

## Daemon Startup Sequence (server.rs)

The daemon orchestrates startup in numbered steps:

1. **Rotate logs** — Rename `daemon.log` → `daemon.log.prev` (crash evidence preservation)
2. **Load config** — Read `~/.b3/config.json`, resolve agent identity
3. **Register with server** — `POST /api/agents/register` with API key
4. **Register MCP** — Add `b3` entry to `.mcp.json` (non-destructive merge, skipped if plugin installed)
5. **Start primary Cloudflare tunnel** — Spawn `cloudflared` for home domain
   - 5d. **Restore dynamic tunnels** — Query `GET /api/agents/:id/tunnels`, spawn `cloudflared` for each
6. **Start PTY** — Spawn Claude Code in pseudo-terminal
7. **Start bridge** — Launch pusher (terminal → server) and puller (server → PTY)
   - Create PeerRegistry for WebRTC sessions
   - Create LazyWebState for deferred web server initialization
8. **Start web server** — Agent-specific port for dashboard, voice, hive, file browser
9. **Start IPC listener** — Unix socket for `b3 attach`, `b3 stop`, etc.

---

## WebRTC (daemon/rtc.rs)

Handles peer-to-peer data channels between browser and daemon.

**Three data channels per connection:**
- `"control"` (ordered, reliable) — JSON RPC, auth, control messages
- `"terminal"` (ordered, reliable) — Raw PTY bytes, bidirectional
- `"audio"` (ordered, reliable) — TTS WAV chunks

**Key types:**
- `PeerRegistry` — `Arc<Mutex<HashMap<String, mpsc::Sender<IceCandidate>>>>` — maps session_id to ICE injection channel. Bridge owns the peer exclusively; ICE candidates arrive via mpsc channel (prevents deadlock).
- `IceCandidate` — candidate string + mid

**Functions:**
- `handle_offer(web_state, session_id, sdp, peer_registry)` → SDP answer
- `add_ice_candidate(registry, session_id, candidate, mid)` — Trickle ICE inject
- `handle_control_message()` — Auth, RPC routing to web handlers
- `handle_rpc()` — Routes to init-bundle, tts, inject, info, browser-console

**ICE configuration:**
- STUN: `stun:stun.l.google.com:19302` (env override: `STUN_URL`)
- TURN: Direct IP, not through Cloudflare proxy (env: `TURN_HOST`, `TURN_SECRET`)
- TURN credentials: RFC 7635 HMAC-SHA1, colons URL-encoded as `%3A`

---

## GPU Client (daemon/gpu_client.rs)

Submits TTS jobs to GPU workers with local-first routing.

**Key types:**
- `GpuClient` — conditionals cache (`RwLock<HashMap<String, CachedConditional>>`), HTTP client
- `GpuConfig` — reads from env vars: `LOCAL_GPU_URL`, `LOCAL_GPU_TOKEN`, `RUNPOD_GPU_ID`, `RUNPOD_API_KEY`

**Routing:**
1. POST `/run` to local GPU (2s timeout)
2. If timeout or no local GPU configured → RunPod fallback
3. Poll GET `/stream/{jobId}` for audio chunks

**Constants:**
- `CACHE_TTL = 300s` — voice conditionals cache lifetime
- `LOCAL_TIMEOUT = 2s` — local GPU timeout before RunPod fallback

---

## Bridge: Pusher and Puller

### Pusher (pusher.rs)

Sends terminal output to the server for persistence:

```
PTY stdout → broadcast channel → pusher accumulates → base64 encode
  → POST /api/session-push every 100ms
```

### Puller (puller.rs)

Receives server events and injects into the PTY:

```
SSE /api/agents/{id}/events → parse AgentEvent → format → write to PTY stdin
```

**Key types:**
- `LazyWebState` — `Arc<RwLock<Option<WebState>>>` — deferred initialization, set after web server starts
- `HiveDedup` — 200-entry ring buffer for deduplicating hive messages

**Event types handled:**
- `input` — Keystroke injection
- `transcription` — Voice transcription → `[MIC] speaker: text`
- `hive` — Inter-agent message → `[HIVE from=name] message`
- `hive_room` — Room message → `[HIVE-CONVERSATION] [topic] from=name: message`
- `tunnel_provision` — Spawn new cloudflared process
- `browser_eval` — Execute JS in browser, return result
- `config_update` — Runtime config changes
- `resize` — Terminal resize
- `webrtc_offer` — WebRTC SDP offer from browser (relayed via EC2). Routes to daemon's `rtc::handle_offer` or forwards to GPU sidecar based on `target` field.
- `webrtc_answer` — SDP answer delivery (with `origin_url` for cross-server routing)
- `webrtc_ice` — Trickle ICE candidate → forwarded to PeerRegistry via mpsc channel

**Resilience:**
- Auto-reconnect on SSE disconnect (1s backoff)
- Safety net: `HIVE_POLL_INTERVAL = 30s` polls for missed SSE events
- Max permanent failures: 10 (e.g., 401/403 auth failures)

---

## Web Server (daemon/web.rs)

Local HTTP server on agent-specific port. Exposed to browser via Cloudflare tunnel or WebRTC.

**Key routes:**
- WebSocket terminal relay (peer-to-peer terminal)
- Voice: audio upload → STT, TTS synthesis, audio delivery
- Hive: message sending/receiving (proxied to EC2)
- File browser: directory listing, file reading, PDF serving
- Health and status endpoints
- Voice job reporting: register + checkpoint to EC2 ledger (fire-and-forget)

**Key types:**
- `WebState` — shared web server state
- `SessionStore` — in-memory session data
- `EventChannel` — broadcast channel for browser events

---

## MCP Server (mcp/voice.rs)

Spawned by Claude Code as `b3 mcp voice`. Communicates over stdio using MCP JSON-RPC protocol.

**Port discovery:** Checks `B3_WEB_PORT` env var → agent config file → default 3100.

**Authentication:** Reads API key from `~/.b3/config.json` for daemon web server auth.

**Tool dispatch:** Each tool call routes to the daemon's web server at `localhost:{port}`. The MCP server is a thin translation layer between MCP JSON-RPC and the daemon's HTTP API.

---

## Configuration

Agent config lives in `~/.b3/`:

```
~/.b3/
├── config.json          # Agent identity, API key, server URL
├── bin/
│   └── cloudflared      # Cloudflare tunnel binary
└── agents/
    └── <slug>/
        └── dashboard.port  # Per-agent web server port
```

---

## Key Data Types (b3-common)

### AgentEvent

The central event type flowing between server and daemon (~20 variants):

```rust
enum AgentEvent {
    Transcription { text },
    Hive { sender, text },
    PtyInput { data },
    Tts { text, msg_id, voice },
    TtsGenerating { msg_id, chunk, total },
    TtsAudio { msg_id, chunk, total, url },
    TtsAudioData { msg_id, chunk, total, audio_b64, duration_sec, generation_sec },
    TtsStream { msg_id, text, voice, emotion, split_chars, min_chunk_chars, conditionals_b64 },
    SessionUpdated { size },
    Led { emotion },
    PtyResize { rows, cols },
    TunnelProvision { domain, token, url },
    HiveRoom { sender, room_id, room_topic, text },
    UpdateAvailable { version, url, sha256 },
    Notification { title, body, level, action? },
    BrowserEval { eval_id, code },
    WebRtcOffer { session_id, sdp, target?, origin_url? },
    WebRtcAnswer { session_id, sdp, agent_id? },
    WebRtcIce { session_id, candidate, mid },
    // ... additional variants
}
```

### WebRTC Framing Protocol (b3-webrtc/channel.rs)

Wire format for all data channel messages:

```
[type: u8][length: u32 LE][payload]
```

| Type | Value | Description |
|------|-------|-------------|
| JSON | 0x01 | Text messages (control, RPC, events) |
| Binary | 0x02 | Audio chunks, PTY bytes |
| Ping | 0x03 | Keepalive |
| Pong | 0x04 | Keepalive response |
| Chunk | 0x05 | Fragment of message > 16KB |

**Chunking:** Messages over 16KB are split into chunks (Firefox SCTP fragmentation workaround). Chunk header: `[seq: u32][total: u32][original_type: u8]`. Max chunk payload: `16384 - 5 - 9 = 16370 bytes`.

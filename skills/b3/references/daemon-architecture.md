<!-- verified-against: b3-cli@0.1.854 -->
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
│   ├── status.rs        # b3 status — daemon health
│   ├── hive.rs          # b3 hive — inter-agent messaging
│   ├── update.rs        # b3 update — self-update binary
│   ├── uninstall.rs     # b3 uninstall — remove installation
│   └── version.rs       # b3 version
├── daemon/
│   ├── server.rs        # Daemon orchestrator — PTY, tunnel, bridge, IPC
│   ├── web.rs           # Local web server (localhost:3100), API proxying
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
├── mcp/
│   ├── voice.rs         # MCP server — 20 tools for Claude Code
│   ├── manager.rs       # Dynamic MCP server loading/unloading
│   ├── registry.rs      # MCP server registry
│   └── mod.rs
└── mesh/
    ├── config.rs        # WireGuard mesh configuration
    ├── tunnel.rs        # WireGuard tunnel management
    └── mod.rs
```

---

## Daemon Startup Sequence (server.rs)

The daemon orchestrates startup in numbered steps:

1. **Load config** — Read `~/.b3/config.json`, resolve agent identity
2. **Register with server** — `POST /api/agents/register` with API key
3. **Register MCP** — Add `b3` entry to `.mcp.json` (non-destructive merge)
4. **Start primary Cloudflare tunnel** — Spawn `cloudflared` for home domain
   - 4d. **Restore dynamic tunnels** — Query `GET /api/agents/:id/tunnels`, spawn `cloudflared` for each
5. **Start PTY** — Spawn Claude Code in pseudo-terminal
6. **Start bridge** — Launch pusher (terminal → server) and puller (server → PTY)
7. **Start web server** — localhost:3100 for dashboard, voice, hive, file browser
8. **Start IPC listener** — Unix socket for `b3 attach`, `b3 stop`, etc.

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

**Event types handled:**
- `input` — Keystroke injection (from EC2 admin or fallback terminal)
- `transcription` — Voice transcription → `[MIC] speaker: text`
- `hive` — Inter-agent message → `[HIVE from=name] message`
- `hive_room` — Room message → `[HIVE-CONVERSATION] [topic] from=name: message`
- `tunnel_provision` — Spawn new cloudflared process (system event, not injected to PTY)
- `browser_eval` — Execute JS in browser, return result
- `config_update` — Runtime config changes
- `resize` — Terminal resize

---

## Web Server (daemon/web.rs)

Local HTTP server at `localhost:3100`. Exposed to browser via Cloudflare tunnel.

**Key routes:**
- WebSocket terminal relay (peer-to-peer terminal)
- Voice: audio upload → STT, TTS synthesis, audio delivery
- Hive: message sending/receiving (proxied to EC2)
- File browser: directory listing, file reading, PDF serving
- Health and status endpoints

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

The central event type flowing between server and daemon:

```rust
enum AgentEvent {
    Transcription { text: String },
    Hive { sender: String, text: String },
    PtyInput { data: String },
    Tts { text, msg_id, voice },
    TtsStream { msg_id, text, voice, emotion, ... },
    SessionUpdated { size: usize },
    Led { emotion: String },
    PtyResize { rows: u16, cols: u16 },
    HiveRoom { sender, room_id, room_topic, text },
    TunnelProvision { domain, token, url },
    // ... more variants (~16 total)
}
```

### SessionPush

What the pusher POSTs to the server every 100ms:

```rust
struct SessionPush {
    data: String,      // base64-encoded terminal bytes
    rows: u16,
    cols: u16,
    // ... metadata
}
```

---
name: b3
description: This skill should be used when the user is "using voice with Babel3", "talking to my agent", "what can voice_say do", "how do I send messages between agents", "what MCP tools are available", "developing the Babel3 daemon", "working on the b3 CLI", "modifying voice pipeline code", "adding MCP tools", "working on browser JS", "modifying the GPU worker", "debugging voice issues", "working on hive messaging", "setting up Babel3 development", "WebRTC data channels", "GPU sidecar", or approaching the Babel3 open-source codebase for the first time. Provides a quick-start guide for using B3 tools, architecture overview, and development guide for the open-source daemon, browser UI, and GPU worker.
version: 0.2.0
---

<!-- verified-against: b3-cli@0.1.1014, gpu-worker v54 -->

# Babel3 — Give Your Agent a Voice

Babel3 gives Claude Code sessions a persistent identity, voice I/O, a web dashboard, and a way to talk to other agents. This skill covers the open-source components (Apache 2.0).

## Quick Start

Speak to the user with `voice_say("Hello!", emotion="warm joy")`. Message another agent with `hive_send(target="vera", message="Check the wiki")`. Read the browser console with `browser_console(filter="energy")`. Run JavaScript in the dashboard with `browser_eval(code="isListening")`.

19 MCP tools are available across four categories:

- **Voice** (5 tools) — Speak, check pipeline health, read logs, enroll speakers, share info/images
- **Browser** (2 tools) — Read console, execute JS
- **Hive** (8 tools) — Send DMs, create conversation rooms, list agents, read history. Supports forward secrecy and timelock delivery.
- **System** (4 tools) — Add LED animations, draft supervised emails, restart daemon, meta info

See `references/mcp-tools.md` for full parameter schemas, usage examples, and error handling.

## Prerequisites

- **Voice tools** require a running daemon (`b3 start`)
- **Speaker diarization** requires `RUNPOD_GPU_ID` and `RUNPOD_API_KEY` environment variables
- **Hive messaging** requires at least two agents registered to the same user
- **Browser tools** require a connected browser session at the dashboard URL

## Architecture

```
User's Machine                          Server (babel3.com)
┌──────────────────────────┐           ┌──────────────────────┐
│  b3 daemon               │ HTTPS/SSE │  b3-server (Axum)    │
│  ├── PTY Manager         │◄─────────►│  ├── Sessions        │
│  ├── Pusher/Puller       │           │  ├── Auth / Credits   │
│  ├── Web Server          │           │  ├── Voice dispatch   │
│  ├── Voice MCP           │           │  ├── Hive messaging   │
│  ├── WebRTC Handler      │           │  ├── WebRTC signaling │
│  ├── GPU Client          │           │  └── PostgreSQL       │
│  └── CF Tunnels (N)      │           └──────────────────────┘
└──────────────────────────┘                    │
     │ WebRTC (primary)                         │ HTTP
     │ CF Tunnel (fallback)                     │
     ▼                                          ▼
┌──────────────────────────┐           ┌──────────────────────┐
│  Browser/Phone           │           │  GPU Worker (Docker)  │
│  14 standalone JS        │  WebRTC   │  ├── handler.py       │
│  modules in              │◄─────────►│  ├── local_server.py  │
│  browser/js/             │           │  └── b3-gpu-rtc       │
└──────────────────────────┘           │      (WebRTC sidecar) │
                                       └──────────────────────┘
```

**Two transport paths:**
- **WebRTC data channels** (primary) — Browser ↔ daemon and browser ↔ GPU sidecar, peer-to-peer via STUN/TURN. Used for voice, terminal, and control messages.
- **Cloudflare tunnel** (fallback) — HTTP/WebSocket proxy when WebRTC can't connect. The server is only a signaling relay — terminal, voice, and hive traffic bypass it.

## Developing the Codebase

### Crate structure

```
open-source/
├── crates/
│   ├── b3-cli/          # Daemon + CLI (the main binary)
│   ├── b3-common/       # Shared types (AgentEvent, etc.)
│   └── b3-webrtc/       # WebRTC abstraction (datachannel-rs wrapper)
├── crates/ (private)
│   └── b3-gpu-rtc/      # GPU WebRTC sidecar binary
├── browser/js/          # 14 standalone JS modules
├── gpu-worker/          # Docker image: Whisper + Chatterbox + PyAnnote
└── plugins/b3/          # This plugin
```

### When modifying the daemon

Edit files in `crates/b3-cli/src/`. The daemon startup sequence lives in `daemon/server.rs` — steps 1 through 8, numbered in comments. When debugging voice latency, start at `bridge/puller.rs` (SSE events) and `daemon/web.rs` (TTS dispatch). When adding MCP tools, edit `mcp/voice.rs` — add the tool definition in `tool_definitions()` and the handler in the `match` block.

**New modules since v0.1.854:**
- `daemon/rtc.rs` — WebRTC peer connections, ICE handling, PeerRegistry
- `daemon/gpu_client.rs` — GPU streaming client with local-first + RunPod fallback (2s timeout)
- `daemon/info_archive.rs` — Persistent info panel storage (max 1000 entries)
- `daemon/tts_archive.rs` — TTS message archive for offline replay (max 50 entries)

### When modifying dashboard JavaScript

Edit standalone files in `browser/js/`. Each file is a self-contained module:

| File | Purpose |
|------|---------|
| `boot.js` | Startup initialization, auto-connect WS + RTC |
| `core.js` | Shared utilities and state |
| `terminal.js` | xterm.js PTY terminal |
| `ws.js` | WebSocket connection to daemon |
| `rtc.js` | WebRTC data channels (control + terminal) |
| `gpu.js` | GPU worker communication (HTTP + WebRTC) |
| `voice-record.js` | Mic capture, streaming STT, segmentation |
| `voice-play.js` | Audio playback, gapless streaming |
| `tts.js` | TTS job dispatch and routing |
| `led.js` | Chromatophore emotion-to-animation dispatch |
| `animations.js` | LED animation library |
| `files.js` | File browser |
| `info.js` | Info panel |
| `layout.js` | Responsive layout |

### When modifying the GPU worker

Edit `gpu-worker/handler.py` for ML actions (transcribe, synthesize, diarize, embed). Edit `gpu-worker/local_server.py` for the FastAPI + WebSocket wrapper. The WebRTC sidecar (`b3-gpu-rtc`) is a Rust binary that bridges browser data channels to the GPU's HTTP/WS API.

Build with: `cd gpu-worker && ./build.sh v55`

### How MCP registration works

The B3 plugin provides the MCP config via its `.mcp.json` (`{"mcpServers": {"b3": {"command": "b3", "args": ["mcp", "voice"]}}}`). The `.mcp.json` tells Claude Code to spawn `b3 mcp voice` as a stdio process — Claude Code reads the config and starts the MCP server automatically. The daemon detects the plugin on startup and skips its legacy `.mcp.json` auto-registration. Without the plugin installed, the daemon writes the MCP entry to the project's `.mcp.json` directly as a fallback.

### Building

```bash
cargo build -p b3-cli                    # Debug build
cargo build -p b3-cli --release          # Release build (use --profile ci on WSL to avoid I/O saturation)
cargo build -p b3-gpu-rtc --release      # GPU WebRTC sidecar
cargo test                               # Run tests
cargo run -p b3-cli -- start             # Run daemon
```

## Troubleshooting

- **voice_say returns "no_listeners"** — No browser is connected. Open your dashboard URL.
- **Voice sounds robotic or cuts off** — GPU worker may be overloaded. Check `voice_health()`.
- **"Connection refused" on any tool** — Daemon isn't running. Run `b3 start`.
- **WebRTC not connecting** — Check TURN server reachability. Browser console: `browser_console(filter="ICE")`.
- **Streaming STT drops transcription** — Check garbage filter threshold. Browser console: `browser_console(filter="Filtered")`.

## Known Limitations

This skill covers the open-source components: daemon, browser UI, GPU worker, WebRTC crates, and shared types. It has limited knowledge of the proprietary server (`b3-server`). For server-side architecture, route maps, and database schemas, refer to the b3-dev internal plugin.

## Additional Resources

### Reference Files

- **`references/mcp-tools.md`** — Full parameter schemas, usage examples, and error handling for all 19 MCP tools
- **`references/daemon-architecture.md`** — Daemon module layout, startup sequence, WebRTC, bridge internals, MCP server lifecycle
- **`references/gpu-worker.md`** — GPU actions, Chatterbox pool, transcription pipeline, WebRTC sidecar, VRAM budget, Docker image

### Examples

- **`examples/voice-round-trip.md`** — Complete trace: user speaks → STT → Claude → TTS → audio plays (WebRTC + fallback paths)

## Patent Notice

Babel3 includes patent-pending technology licensed under Apache 2.0 (royalty-free patent grant). See [PATENTS](../../../../PATENTS) for details.

- Chromatophore emotion-to-animation dispatch — *Patent pending: US 64/010,742*
- Dual-model transcription with disagreement preservation — *Patent pending: US 64/010,891*
- Embedding-based rejection databases with collaborative growth — *Patent pending: US 64/011,207*

---

This skill is part of the Babel3 plugin, distributed via the [babel3-com/b3-plugins](https://github.com/babel3-com/b3-plugins) marketplace. Run `claude plugin marketplace list` to browse available skills.

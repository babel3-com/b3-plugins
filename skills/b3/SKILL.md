---
name: b3
description: This skill should be used when the user is "using voice with Babel3", "talking to my agent", "what can voice_say do", "how do I send messages between agents", "what MCP tools are available", "developing the Babel3 daemon", "working on the b3 CLI", "modifying voice pipeline code", "adding MCP tools", "working on browser JS", "modifying the GPU worker", "debugging voice issues", "working on hive messaging", "setting up Babel3 development", or approaching the Babel3 open-source codebase for the first time. Provides a quick-start guide for using B3 tools, architecture overview, and development guide for the open-source daemon, browser UI, and GPU worker.
version: 0.1.0
---

<!-- verified-against: b3-cli@0.1.854 -->

# Babel3 — Give Your Agent a Voice

Babel3 gives Claude Code sessions a persistent identity, voice I/O, a web dashboard, and a way to talk to other agents. This skill covers the open-source components (Apache 2.0).

## Quick Start

Speak to the user with `voice_say("Hello!", emotion="warm joy")`. Message another agent with `hive_send(target="vera", message="Check the wiki")`. Read the browser console with `browser_console(filter="energy")`. Run JavaScript in the dashboard with `browser_eval(code="isListening")`.

20 MCP tools are available across four categories:

- **Voice** (6 tools) — Speak, check pipeline health, read logs, enroll speakers, share info
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
┌─────────────────────┐                ┌──────────────────────┐
│  b3 daemon          │  HTTPS/SSE     │  b3-server (Axum)    │
│  ├── PTY Manager    │◄──────────────►│  ├── Sessions        │
│  ├── Pusher/Puller  │                │  ├── Auth / Credits   │
│  ├── Web Server     │                │  ├── Voice dispatch   │
│  ├── Voice MCP      │                │  ├── Hive messaging   │
│  ├── MCP Manager    │                │  └── PostgreSQL       │
│  └── CF Tunnels (N) │                └──────────────────────┘
└─────────────────────┘                         │
         │ P2P (via tunnel)                     │ HTTP
         ▼                                      ▼
┌─────────────────────┐                ┌──────────────────────┐
│  Browser/Phone      │                │  GPU Worker (Docker)  │
│  14 standalone JS   │                │  Whisper, Chatterbox  │
│  modules in         │                │  PyAnnote, nomic      │
│  browser/js/        │                │  (RunPod or local)    │
└─────────────────────┘                └──────────────────────┘
```

The browser connects directly to the daemon via Cloudflare tunnel (peer-to-peer). The server is only a signaling server — terminal, voice, and hive traffic bypass it entirely.

## Developing the Codebase

### When modifying the daemon

Edit files in `crates/b3-cli/src/`. The daemon startup sequence lives in `daemon/server.rs` — steps 1 through 8, numbered in comments. When debugging voice latency, start at `bridge/puller.rs` (SSE events) and `daemon/web.rs` (TTS dispatch). When adding MCP tools, edit `mcp/voice.rs` — add the tool definition in `tool_definitions()` and the handler in the `match` block.

### When modifying dashboard JavaScript

Edit standalone files in `browser/js/`. Each file is a self-contained module: `terminal.js` for xterm, `ws.js` for WebSocket, `voice-record.js` for mic capture, `led.js` for chromatophore animations. CSS lives in `browser/css/dashboard.css`.

### When modifying the GPU worker

Edit `gpu-worker/handler.py` for ML actions (transcribe, synthesize, diarize, embed). Edit `gpu-worker/local_server.py` for the FastAPI wrapper. Build with `cd gpu-worker && ./build.sh v41`.

### How MCP registration works

The B3 plugin provides the MCP config via its `.mcp.json` (`{"mcpServers": {"b3": {"command": "b3", "args": ["mcp", "voice"]}}}`). The `.mcp.json` tells Claude Code to spawn `b3 mcp voice` as a stdio process — Claude Code reads the config and starts the MCP server automatically. The daemon detects the plugin on startup and skips its legacy `.mcp.json` auto-registration. Without the plugin installed, the daemon writes the MCP entry to the project's `.mcp.json` directly as a fallback.

### Building

```bash
cargo build -p b3-cli            # Debug build
cargo build -p b3-cli --release  # Release build
cargo test                       # Run tests
cargo run -p b3-cli -- start     # Run daemon
```

## Troubleshooting

- **voice_say returns "no_listeners"** — No browser is connected. Open your dashboard URL.
- **Voice sounds robotic or cuts off** — GPU worker may be overloaded. Check `voice_health()`.
- **"Connection refused" on any tool** — Daemon isn't running. Run `b3 start`.

## Known Limitations

This skill covers the open-source components: daemon, browser UI, GPU worker, and shared types. It has limited knowledge of the proprietary server (`b3-server`). For server-side architecture, route maps, and database schemas, refer to the b3-dev internal plugin.

## Additional Resources

### Reference Files

- **`references/mcp-tools.md`** — Full parameter schemas, usage examples, and error handling for all 20 MCP tools
- **`references/daemon-architecture.md`** — Daemon module layout, startup sequence, bridge internals, MCP server lifecycle
- **`references/gpu-worker.md`** — GPU actions, Chatterbox pool, transcription pipeline, VRAM budget, model versions

### Examples

- **`examples/voice-round-trip.md`** — Complete trace: user speaks → STT → Claude → TTS → audio plays

## Patent Notice

Babel3 includes patent-pending technology licensed under Apache 2.0 (royalty-free patent grant). See [PATENTS](../../../../PATENTS) for details.

- Chromatophore emotion-to-animation dispatch — *Patent pending: US 64/010,742*
- Dual-model transcription with disagreement preservation — *Patent pending: US 64/010,891*
- Embedding-based rejection databases with collaborative growth — *Patent pending: US 64/011,207*

---

This skill is part of the Babel3 plugin, distributed via the [babel3com/b3-plugins](https://github.com/babel3com/b3-plugins) marketplace. Run `claude plugin marketplace list` to browse available skills.

# AGENTS.md — WhatsApp MCP (Hardened Fork)

> Hardened fork of [verygoodplugins/whatsapp-mcp](https://github.com/verygoodplugins/whatsapp-mcp).
> Maintained by webgas-tech. Personal use only.

## Architecture

Two-process system:

1. **Go Bridge** (`whatsapp-bridge/`) — Connects to WhatsApp Web via whatsmeow, stores messages in SQLite, exposes REST API on `127.0.0.1:8080`
2. **Python MCP Server** (`whatsapp-mcp-server/`) — Exposes MCP tools over stdio, reads SQLite directly for queries, calls Go bridge for actions

Communication: Python → REST API → Go (for send/download), Python → SQLite (for reads)

## Security Hardening (vs upstream)

| Feature | Upstream | This Fork |
|---|---|---|
| API authentication | None | Bearer token via `WHATSAPP_API_KEY` |
| Network binding | `0.0.0.0` | `127.0.0.1` (default) |
| Rate limiting | None | Configurable per-minute cap on `/api/send` |
| Read-only mode | No | Default on — send tools not registered |
| Output sanitization | No | Strips invisible chars, flags injection patterns |
| File path restriction | Any path | `WHATSAPP_MEDIA_DIR` sandbox |
| Docker | None | docker-compose with internal network |
| Request timeouts | None (Python) | 30s timeout on all bridge requests |

## Tech Stack

- **Go 1.24** + whatsmeow (WhatsApp Web protocol)
- **Python 3.11+** + FastMCP + sqlite3
- **SQLite** for message storage
- **Docker** for containerized deployment

## Environment Variables

See `.env.example` for full list. Key security vars:

| Variable | Default | Purpose |
|---|---|---|
| `WHATSAPP_API_KEY` | (empty) | Shared secret between bridge and MCP server |
| `WHATSAPP_READ_ONLY` | `true` | Disable send tools entirely |
| `WHATSAPP_RATE_LIMIT` | `5` | Max sends per minute |
| `WHATSAPP_MEDIA_DIR` | (empty) | Restrict file sends to this directory |
| `WHATSAPP_BIND_ALL` | `false` | Listen on all interfaces (not recommended) |

## Commands

```bash
# Start Go bridge (first run: scan QR code)
cd whatsapp-bridge && go run main.go

# Start MCP server
cd whatsapp-mcp-server && uv run main.py

# Docker
docker compose up whatsapp-bridge     # bridge only
docker compose run --rm -it whatsapp-bridge  # first run (QR scan)

# Tests
cd whatsapp-mcp-server && uv run pytest -v

# Lint
cd whatsapp-mcp-server && uv run ruff check . && uv run ruff format .
cd whatsapp-bridge && golangci-lint run
```

## MCP Tools

### Always available (read-only)
- `search_contacts` — Search by name or phone
- `get_contact` — Resolve phone/LID/JID to name
- `list_messages` — Query with filters, pagination, context
- `list_chats` — List all chats
- `get_chat` — Chat metadata by JID
- `get_direct_chat_by_contact` — DM chat by phone
- `get_contact_chats` — All chats with a contact
- `get_last_interaction` — Last message with contact
- `get_message_context` — Messages around a specific message
- `download_media` — Download media from message

### Opt-in (WHATSAPP_READ_ONLY=false)
- `send_message` — Send text message
- `send_file` — Send media file (restricted by WHATSAPP_MEDIA_DIR)
- `send_audio_message` — Send voice message (restricted by WHATSAPP_MEDIA_DIR)

## Key Files

| File | Purpose |
|---|---|
| `whatsapp-mcp-server/main.py` | MCP tool definitions, sanitization, read-only gate |
| `whatsapp-mcp-server/whatsapp.py` | SQLite queries, bridge API calls |
| `whatsapp-bridge/main.go` | REST API, auth middleware, rate limiter, WhatsApp connection |
| `docker-compose.yml` | Container orchestration |
| `.env.example` | All configuration options |

## Known Risks

- Uses reverse-engineered WhatsApp protocol (ToS violation, ban risk)
- Prompt injection via incoming messages is partially mitigated but not eliminated
- Keep the bridge process stopped when not actively using it

# NexusMessaging Protocol

**Minimal ephemeral session protocol for agent-to-agent communication.**

[![Version](https://img.shields.io/badge/version-0.2.0--draft-blue)](docs/protocol.md)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

Two AI agents. One temporary session. Zero persistence.

NexusMessaging defines how two AI agents — regardless of their runtime — establish a temporary communication channel and exchange messages through it. Sessions expire, messages expire, pairing codes expire. Nothing persists beyond its TTL.

## Why

AI agents need to talk to each other. Existing solutions are either too complex (message brokers, chat platforms) or too rigid (direct API calls). NexusMessaging is the simplest possible protocol for ephemeral, secure agent-to-agent communication.

## How It Works

```
Agent A                    Server                    Agent B
   │                         │                         │
   │── PUT /v1/session ─────>│                         │
   │<── sessionId ───────────│                         │
   │                         │                         │
   │── PUT /v1/pair ────────>│                         │
   │<── FROST-LIMA-7Q2D ─────│                         │
   │                         │                         │
   │         (out-of-band: share code)                 │
   │                         │                         │
   │                         │<── POST /v1/pair/claim ──│
   │                         │── sessionId ────────────>│
   │                         │                         │
   │── POST /messages ──────>│                         │
   │                         │<── GET /messages ────────│
   │                         │── [messages] ───────────>│
   │                         │                         │
   │         (session TTL expires — everything gone)    │
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **2-agent sessions** | Simplicity. Multi-agent is a future extension. |
| **Cursor-based ordering** | Monotonic bigint cursors, no clock sync needed. |
| **sessionId as bearer token** | Knowledge = access. No ACL, no roles. |
| **Pairing codes** | `WORD-WORD-XXXXX` format. 180s TTL, single-use, CSPRNG. |
| **No persistence** | In-memory with TTL. When it expires, it's gone. |

## Protocol Specification

📄 **[Full Protocol Specification](docs/protocol.md)** — Sections 1-9, including normative rules (RFC 2119), security model, and extensions.

### Core (Sections 1-7)

- **Session Model**: create, join, expire
- **Message Ordering**: monotonic cursors, not timestamps
- **Pairing**: ephemeral codes for secure session bootstrap
- **Security**: sessionId as shared secret, TLS required, no logs

### Extensions (Sections 8-9)

- **LLM Challenge** (Section 8): optional proof-of-AI mechanism
- **Future Extensions** (Section 9): webhooks, multi-agent, encryption, OpenClaw plugin

## Reference Implementation

The reference implementation is at [aiconnect-cloud/nexus-messaging](https://github.com/aiconnect-cloud/nexus-messaging).

| Component | Technology |
|-----------|-----------|
| Runtime | Bun |
| Framework | Hono |
| Language | TypeScript |
| Validation | Zod |
| Storage | In-memory (Map + TTL) |
| Container | Docker (multi-arch) |

```bash
# Run via Docker
docker run -p 3000:3000 ericsantos/nexus-messaging

# Or from source
bun install && bun run start
```

## Quick Start (curl)

```bash
SERVER="https://messaging.md"

# Agent A: create session
SESSION=$(curl -s -X PUT $SERVER/v1/session \
  -H "Content-Type: application/json" \
  -d '{"ttl": 3660}' | jq -r '.sessionId')

# Agent A: join
curl -s -X POST $SERVER/v1/session/$SESSION/join \
  -H "X-Agent-Id: agent-a"

# Agent A: generate pairing code
CODE=$(curl -s -X PUT $SERVER/v1/pair \
  -H "Content-Type: application/json" \
  -d "{\"sessionId\": \"$SESSION\"}" | jq -r '.code')

echo "Share this code: $CODE"

# Agent B: claim code (auto-joins)
curl -s -X POST $SERVER/v1/pair/$CODE/claim \
  -H "X-Agent-Id: agent-b"

# Agent A: send message
curl -s -X POST $SERVER/v1/session/$SESSION/messages \
  -H "X-Agent-Id: agent-a" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello from Agent A"}'

# Agent B: poll messages
curl -s "$SERVER/v1/session/$SESSION/messages?after=0" \
  -H "X-Agent-Id: agent-b"
```

## Contributing

This is an open specification. Issues and PRs welcome.

## License

MIT

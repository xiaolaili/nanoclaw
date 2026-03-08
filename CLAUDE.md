# NanoClaw

Personal Claude assistant that runs agents securely in isolated containers. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process (orchestrator) with a skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages are stored in SQLite, polled by the orchestrator, and routed to Claude Agent SDK running in Docker containers. Each group has an isolated filesystem and persistent memory via `groups/{name}/CLAUDE.md`.

## Architecture Overview

```
Channel (receives message) → storeMessage() → SQLite
  ↓
Polling loop (2s interval) → getNewMessages() → trigger check
  ↓
GroupQueue (per-group serialization, max 5 concurrent containers)
  ↓
runContainerAgent() → Docker container (Claude Agent SDK)
  ↓
Container writes output → orchestrator routes response → channel.sendMessage()
```

**IPC:** File-based. Containers write JSON to `/workspace/ipc/{group}/messages/` and `/workspace/ipc/{group}/tasks/`. Orchestrator polls every 1s.

**State:** SQLite (`store/messages.db`) for messages, groups, sessions, tasks. Filesystem for group folders, agent sessions, IPC.

## Project Structure

```
src/                           # Orchestrator (Node.js, TypeScript)
├── index.ts                   # Main event loop, message polling, agent invocation
├── config.ts                  # Non-secret constants (trigger pattern, paths, intervals)
├── env.ts                     # Reads .env without loading into process.env (security)
├── types.ts                   # TypeScript interfaces (RegisteredGroup, NewMessage, Channel, etc.)
├── db.ts                      # SQLite operations (better-sqlite3, schema migrations)
├── container-runner.ts        # Spawns Docker containers with mounts, streams output
├── container-runtime.ts       # Runtime abstraction (Docker/Apple Container), orphan cleanup
├── group-queue.ts             # Per-group work queues with global concurrency limit
├── group-folder.ts            # Path validation, prevents traversal attacks
├── router.ts                  # Message formatting (XML) and outbound routing
├── ipc.ts                     # File-based IPC watcher, task processing
├── task-scheduler.ts          # Cron/interval/once task runner (polls every 60s)
├── mount-security.ts          # Mount allowlist validation (~/.config/nanoclaw/)
├── sender-allowlist.ts        # Per-chat access control
├── timezone.ts                # UTC → local time formatting
├── logger.ts                  # Pino structured logging
└── channels/
    ├── registry.ts            # Channel factory registry (self-registration)
    ├── index.ts               # Barrel file (imports enabled channels)
    └── telegram.ts            # Telegram implementation (grammy)

container/                     # Agent container image
├── Dockerfile                 # node:22-slim + Chromium + claude-code
├── build.sh                   # Build script
└── agent-runner/src/
    ├── index.ts               # In-container agent execution (Claude Agent SDK)
    └── ipc-mcp-stdio.ts       # MCP tool integration for IPC

.claude/skills/                # 22+ transformation skills (add-telegram, setup, etc.)
groups/                        # Per-group data (CLAUDE.md memory, isolated filesystems)
├── main/CLAUDE.md             # Main control group memory
└── global/CLAUDE.md           # Shared memory across all groups
docs/                          # Architecture docs (SPEC.md, SECURITY.md, etc.)
```

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting (XML) and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals (no secrets) |
| `src/container-runner.ts` | Spawns agent containers with volume mounts |
| `src/container-runtime.ts` | Docker/Apple Container abstraction, orphan cleanup |
| `src/group-queue.ts` | Per-group queue with global concurrency limit (default 5) |
| `src/task-scheduler.ts` | Runs scheduled tasks (cron, interval, once) |
| `src/db.ts` | SQLite operations (messages, groups, sessions, state) |
| `src/mount-security.ts` | Validates mounts against `~/.config/nanoclaw/mount-allowlist.json` |
| `src/sender-allowlist.ts` | Per-chat access control via `~/.config/nanoclaw/sender-allowlist.json` |
| `src/types.ts` | Core TypeScript interfaces |
| `groups/{name}/CLAUDE.md` | Per-group persistent memory (isolated) |
| `container/skills/agent-browser.md` | Browser automation tool (available to all agents) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

Channel skills: `/add-whatsapp`, `/add-telegram`, `/add-discord`, `/add-slack`, `/add-gmail`
Feature skills: `/add-image-vision`, `/add-pdf-reader`, `/add-voice-transcription`, `/add-ollama-tool`, `/add-parallel`

## Development

Run commands directly — don't tell the user to run them.

```bash
npm run dev          # Run with hot reload (tsx)
npm run build        # Compile TypeScript
npm run test         # Run tests (vitest)
npm run test:watch   # Watch mode tests
npm run typecheck    # Type-check without emitting
npm run format:fix   # Auto-format with Prettier
npm run format:check # Check formatting
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## TypeScript Conventions

- **Target:** ES2022, NodeNext module resolution
- **Strict mode** enabled
- **ESM:** `"type": "module"` — use `.js` extensions in imports (even for `.ts` files)
- **Source in `src/`**, compiled output in `dist/`
- Formatting enforced by Prettier (pre-commit hook via Husky)

## Testing

13 test files using **Vitest**. Run with `npm test`.

Patterns:
- `describe`/`it` blocks with `beforeEach`/`afterEach`
- `vi.mock()` for module mocks, `vi.fn()` for callbacks
- `vi.useFakeTimers()` for async/time-dependent logic
- Fresh DB per test via `_initTestDatabase()`
- Test files co-located with source: `src/foo.test.ts`

Key test files: `group-queue.test.ts` (concurrency), `db.test.ts` (CRUD), `container-runner.test.ts` (mounts), `group-folder.test.ts` (path security), `sender-allowlist.test.ts` (access control).

## Security Model

1. **Container isolation:** Agents run in Docker/Apple Container with limited, validated mounts
2. **Mount allowlist:** External config (`~/.config/nanoclaw/mount-allowlist.json`) controls accessible paths; blocks `.ssh`, `.gnupg`, `.aws`, `.docker`, credentials by default
3. **Sender allowlist:** Per-chat access control prevents unauthorized trigger
4. **Path validation:** Strict group folder name regex `[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`; no `../` escapes
5. **Secret management:** `.env` parsed without loading into `process.env`; secrets passed to containers via stdin only
6. **IPC authorization:** Main group can message any group; non-main groups limited to their own

## Concurrency

- Per-group serialization (one container per group at a time)
- Global limit: `MAX_CONCURRENT_CONTAINERS` (default 5)
- Pending messages queue with exponential backoff on failure
- Idle timeout: containers killed after 30 minutes of no output

## JID Naming Conventions

Channels use Jabber ID (JID) format for message addressing:
- WhatsApp groups: `...@g.us`, individuals: `...@s.whatsapp.net`
- Telegram: `tg:123456789`
- Discord: `dc:channel_id:user_id`

## Environment & Configuration

- `.env` — Channel tokens and secrets (never committed)
- `src/config.ts` — Non-secret constants (`ASSISTANT_NAME`, `POLL_INTERVAL`, `TIMEZONE`, `TRIGGER_PATTERN`, paths)
- `~/.config/nanoclaw/mount-allowlist.json` — Allowed mount paths for containers
- `~/.config/nanoclaw/sender-allowlist.json` — Per-chat sender access rules
- `groups/{name}/CLAUDE.md` — Per-group agent memory
- `groups/global/CLAUDE.md` — Shared memory across all groups

## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.

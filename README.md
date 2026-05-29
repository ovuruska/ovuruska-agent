# ovuruska-agent

Custom Hermes Agent configuration for `@TftUstadiBot` on Telegram, alias
**Sarıların Sülo**. Multi-domain dispatch router + three specialist worker
profiles backed by their own MCP servers.

## What's in here

This repo tracks the configuration overlay that lives at `~/.hermes/`.
It is NOT the Hermes Agent core (that's at
[ovuruska/hermes-agent](https://github.com/ovuruska/hermes-agent), a fork
of NousResearch/hermes-agent). It IS the per-user config that turns a
generic Hermes install into the specific bot.

```
.
├── SOUL.md                       — multi-domain router persona (Sarıların Sülo)
├── config.yaml                   — global Hermes config (model, toolsets, display, …)
└── profiles/
    ├── tft/                      — TFT analytics worker
    │   ├── SOUL.md               — Spartan tone, item / comp / build query rules
    │   └── config.yaml           — model + MCP server pointer
    ├── hn/                       — Hacker News reporter worker
    │   ├── SOUL.md               — strict format spec, mirror tool calls
    │   └── config.yaml
    └── semcache/                 — Semantic cache worker (SET / GET / cache_stats)
        ├── SOUL.md               — caveman key rules, strict input parser
        └── config.yaml
```

Backing MCP servers and Python code live in separate repos:

| Worker profile | Backing MCP server (separate repo)             |
|----------------|------------------------------------------------|
| `tft`          | [ovuruska/tftlegends](https://github.com/ovuruska/tftlegends) `tft_agent/server.py` |
| `hn`           | [ovuruska/tftlegends](https://github.com/ovuruska/tftlegends) `hn_agent/server.py`  |
| `semcache`     | [ovuruska/semcache](https://github.com/ovuruska/semcache) `src/semcache/server.py`  |

## Identity

The bot is **Sarıların Sülo** — male, Spartan, brutal voice. Never
apologizes, never narrates plans, never uses emoji or exclamation marks.
Default user-facing language is Turkish, switches to English on English
input. The bot's Telegram username is `@TftUstadiBot`.

## Dispatch protocol

The router does NOT answer questions. For every inbound message it:

1. Classifies the domain (TFT, HN, semcache, or fallback)
2. Calls `kanban_create(assignee=<profile>, title=…, body=…)` to spawn a
   task for the matching worker
3. Calls `kanban_subscribe(task_id=…)` so the gateway's kanban-notifier
   delivers the worker's result directly to the user's chat
4. Emits exactly one em-dash character (`—`) as a reply to the user's
   message (Telegram quotes the original), then ends the turn

The notifier closes the loop with the worker's verbatim output ~30–60s
later. The router never sees the result.

### Routing priority

1. `SET ` / `GET ` / `cache_stats` prefix → `semcache` (overrides everything)
2. TFT continuation context (numbered reference following a prior TFT
   response) → `tft`
3. TFT trigger keywords (champion / item / comp / trait names, Turkish
   component words) → `tft`
4. HN trigger keywords ("Hacker News", "HN", "bugün neler", tech news) → `hn`
5. Anything else → single-line fallback

## Install on a fresh machine

```bash
# 1. Clone Hermes Agent fork
git clone git@github.com:ovuruska/hermes-agent.git ~/.hermes/hermes-agent
cd ~/.hermes/hermes-agent
python3.11 -m venv venv
venv/bin/pip install -e .

# 2. Clone this repo and stage configs to ~/.hermes/
git clone git@github.com:ovuruska/ovuruska-agent.git /tmp/ovuruska-agent
cp /tmp/ovuruska-agent/SOUL.md ~/.hermes/SOUL.md
cp /tmp/ovuruska-agent/config.yaml ~/.hermes/config.yaml
mkdir -p ~/.hermes/profiles
cp -r /tmp/ovuruska-agent/profiles/* ~/.hermes/profiles/

# 3. Provide .env (gitignored — not in any of these repos)
# Required keys: TELEGRAM_BOT_TOKEN, TELEGRAM_HOME_CHANNEL,
# TELEGRAM_ALLOW_ALL_USERS=true, TELEGRAM_GROUP_ALLOWED_CHATS=*,
# TELEGRAM_REQUIRE_MENTION=true, ANTHROPIC_API_KEY (or equivalent).

# 4. Clone the backing MCP server repos
git clone git@github.com:ovuruska/tftlegends.git ~/Software/personal/tftlegends
git clone git@github.com:ovuruska/semcache.git    ~/Software/personal/semcache

# 5. Bring up Qdrant (semcache backend) via launchd
cd ~/Software/personal/semcache
cp deploy/com.semcache.qdrant.plist     ~/Library/LaunchAgents/
cp deploy/com.semcache.ttl-sweeper.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.semcache.qdrant.plist
launchctl load ~/Library/LaunchAgents/com.semcache.ttl-sweeper.plist

# 6. Start Hermes gateway via launchd
launchctl load ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

After (5) the bot starts polling Telegram and responding under the rules
in `SOUL.md` + `profiles/*/SOUL.md`.

## Notes on tracked files

- `config.yaml` is the **whole** Hermes global config, not a slim
  override. It is large because Hermes config schema is large; the parts
  that actually matter for this bot are: `model.default`, `agent.reasoning_effort`,
  `display.tool_progress`, the `platform_toolsets.telegram` allowlist,
  and the `mcp_servers` map. The rest is Hermes defaults.
- Profile `config.yaml` files are copies of the global, with the
  `mcp_servers` block re-pointed at that profile's backing MCP server.
- `.env` is **never** tracked here. All secrets stay in
  `~/.hermes/.env` on the host machine.

## Companion repos

- [ovuruska/tftlegends](https://github.com/ovuruska/tftlegends) — TFT
  crawler, ETLs, and the `tft_agent` + `hn_agent` MCP servers
- [ovuruska/semcache](https://github.com/ovuruska/semcache) — semantic
  cache MCP server (Qdrant + DeepInfra embeddings)
- [ovuruska/hermes-agent](https://github.com/ovuruska/hermes-agent) —
  fork of NousResearch/hermes-agent with two cherry-picked patches
  (`kanban_subscribe` MCP tool + notifier `task.result` verbatim forward)

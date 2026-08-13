# Yomi for Olares

An [Olares Application Chart](https://www.olares.com/docs/zh/developer/develop/package/chart) (OAC)
for [yomi](https://github.com/Crescent617/yomi) — a self-hosted, always-on AI inbox agent.

The chart runs the yomi daemon as a long-lived workload on Olares: the entire `$HOME` persisted
on user storage (browsable in the Files app), model configuration via install parameters, and a
WebSocket entrance for remote daemon access.

> ⚠️ This repository is **not** part of the upstream yomi project. yomi itself is licensed
> under [AGPL-3.0](https://github.com/Crescent617/yomi/blob/main/LICENSE).

## Repository layout

This repository **is** the chart (OAC):

```
Chart.yaml
OlaresManifest.yaml     # entrances, permissions, resources, install-time envs
owners                  # required only for Olares Market submission
values.yaml             # image coordinates and resource limits
templates/
  deployment.yaml       # single-container yomi daemon, env-based model config
  service.yaml          # WebSocket port 57231
docker/
  Dockerfile            # debian + yomi + basic tools, all sha256-verified
```

## Install on Olares

The chart pulls `image.repository:image.tag` as configured in `values.yaml` (amd64 only —
upstream yomi publishes no linux/arm64 releases).

Install via **Studio** or the Market's custom installation, loading this chart directory.

### Install-time parameters

All optional, editable later in the app's Settings page, applied on change (the app restarts
automatically):

| Parameter | Maps to | Description |
| --- | --- | --- |
| `MODEL_PROVIDER` | `YOMI_PROVIDER` | `openai` / `openai_response` / `anthropic` (default `openai`) |
| `MODEL_ENDPOINT` | `YOMI_API_BASE` | OpenAI-compatible API base URL |
| `MODEL_API_KEY` | `YOMI_API_KEY` | Model API key (never written to disk) |
| `MODEL_ID` | `YOMI_MODEL` | Model id, e.g. `gpt-5`, `kimi-k3` |
| `TZ` | `TZ` | Container timezone (default `Etc/UTC`) |

These use yomi's native env-override mechanism: they build an in-memory model named `from_env`,
pinned as the default via `YOMI_DEFAULT_MODEL=from_env`. **No config file is required.**

## Configuring yomi

Everything below lives in `Home/yomi` (the container's `$HOME`), editable in the Files app.

### Model

Use the Settings-page parameters above. If you later write models into `config.toml`, note the
precedence: `YOMI_*` environment variables **override** the same settings in the file, and
`YOMI_DEFAULT_MODEL=from_env` stays the default unless you switch models in-session.

### Channels (Feishu/Lark)

Channels are the one thing yomi only reads from a file. To enable the Feishu channel, create
`Home/yomi/.yomi/config.toml`:

```toml
[[channels]]
name = "feishu"
enabled = true
require_mention = true
reply_in_thread = true

[channels.platform]
type = "feishu"
app_id = "cli_xxx"
app_secret = "xxx"
```

Restart the app to apply. Multi-model setups, custom `system_prompt`, `[env]` injection and
everything else also go into this file — see yomi's
[CONFIG.md](https://github.com/Crescent617/yomi/blob/main/docs/CONFIG.md) for the full schema.

### Skills

Drop skills into `Home/yomi/.agents/skills/` (or `Home/yomi/.yomi/skills/`), then restart the
app or run `yomi daemon reload`. If a skill needs extra CLIs (e.g. the `lark-*` skills need
Node.js + `@larksuite/cli`), ask the agent to install them — everything under `$HOME` persists.

### Remote attach (TUI)

Find the `yomi-ws` entrance URL on the app's detail page, install the **same** yomi version
locally (the wire handshake enforces it), then:

```bash
YOMI_SOCKET=wss://<yomi-ws-entrance-url> yomi tui
YOMI_SOCKET=wss://<yomi-ws-entrance-url> yomi daemon status
```

## Security notes

- `Home/yomi` holds all runtime state (OAuth tokens, sessions, and `config.toml` if you create
  one). Isolation relies on Olares per-user storage.
- The `yomi-ws` entrance grants **full daemon control** (arbitrary command execution inside the
  pod). It is protected by Olares platform authentication (`authLevel: internal`) — do not
  lower it to `public`.

# claude-cowork-local

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) against
**free OpenCode Zen models** and **OpenCode Go** through a
self-hosted translation proxy.

A single install script produces a working setup with two pre-configured
profiles you can launch with `claude --settings ~/.claude/settings.<profile>.json`.

---

## What this is

Claude Code speaks the **Anthropic Messages API** (`/v1/messages`). Most
non-Anthropic model providers — including OpenCode — speak the **OpenAI Chat Completions API**
(`/v1/chat/completions`). They are not wire-compatible.

This repository is a thin wrapper that:

1. Runs the [cucoleadan/opencode-cowork-proxy](https://github.com/cucoleadan/opencode-cowork-proxy)
   worker locally on `http://localhost:8787` (using Bun, no build step).
2. Generates ready-to-use `settings.*.json` profiles for Claude Code that
   point at the local worker.
3. Provides a small set of management commands (`cowork-start`, `cowork-stop`,
   `cowork-status`, `cowork-log`).

The result is a Claude Code installation that can hot-swap between free
Zen models and paid OpenCode Go models.

---

## Why not just use the upstream worker on Cloudflare

The upstream project deploys to [Cloudflare Workers](https://workers.cloudflare.com/).
This has a specific operational drawback for the free
Zen tier:

- Cloudflare Workers share a small pool of egress IP addresses across all
  users on the platform. OpenCode Zen enforces a per-IP rate limit on the
  free model pool. Under sustained use, requests routed through Cloudflare
  start returning `HTTP 429 FreeUsageLimitError` with a `Retry-After` of
  several hours.

Running the same code locally moves egress to the user's own IP, which is
not subject to the shared rate limit.

## Quick start

```bash
git clone ...
cd claude-cowork-local
./install.sh
```

The installer will:

1. Verify or install `bun` (Bun 1.1+).
2. Install worker dependencies (just `hono`).
3. Prompt for API keys (see [API keys](#api-keys) below).
4. Render `settings.{zen,go}.json` into `~/.claude/` with the
   supplied keys.
5. Symlink `cowork-start`, `cowork-stop`, `cowork-status`, `cowork-log` into
   `~/.local/bin/`.
6. Optionally install and enable a systemd unit.
7. Start the worker and run a smoke test.

Then launch Claude Code with a profile:

```bash
claude --settings ~/.claude/settings.zen.json
```

```bash
claude --settings ~/.claude/settings.go.json
```

To make a profile the default (no `--settings` flag needed):

```bash
cp ~/.claude/settings.zen.json ~/.claude/settings.json
```

---

## Prerequisites

- Linux or macOS. On Windows, use WSL2.
- `curl` (used by the installer to fetch Bun if missing).
- **Bun** 1.1 or later — installed automatically if absent.
- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI** —
  this package is a backend for it.

No other system packages are required. The worker binds to
`127.0.0.1:8787` by default.

---

## API keys

Two API keys are referenced across the profiles. The
installer accepts any subset — profiles for which no key is provided are
simply not generated.

| Profile | Key format | Where to obtain | Card required |
|---|---|---|---|
| Zen (free tier) | `sk-…` | OpenCode workspace settings, under API keys | No for free models |
| OpenCode Go | `sk-…` (same key as Zen) | Same as Zen | **Yes** — subscription required |

The Zen and Go profiles share the same OpenCode workspace key. If the
workspace has a payment method attached, both work; otherwise Go models
return `401 No payment method`. The Zen profile works on free models
without a payment method.

---

## Installation

### Interactive

```bash
./install.sh
```

### Non-interactive

```bash
ZEN_KEY=sk-… ./install.sh
```

The `GO_KEY` defaults to `ZEN_KEY` if not provided. Add
`--no-systemd` to skip the systemd unit installation:

```bash
ZEN_KEY=sk-… PROXY_PORT=9999 ./install.sh --no-systemd
```

Available installer environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `ZEN_KEY` | *(prompted)* | OpenCode workspace API key |
| `GO_KEY`  | `$ZEN_KEY` | Go API key (usually identical to `ZEN_KEY`) |
| `PROXY_PORT` | `8787` | TCP port the worker binds to |

---

## Usage

### Switching profiles

The two settings files in `~/.claude/` are independent configurations.
Switch by passing `--settings` at launch:

```bash
claude --settings ~/.claude/settings.zen.json
claude --settings ~/.claude/settings.go.json
```

To make one of them the implicit default, copy it to `~/.claude/settings.json`.

---

## Worker management

After installation, the following commands are available in `PATH`
(via symlinks in `~/.local/bin/`):

| Command | Effect |
|---|---|
| `cowork-start`  | Start the worker. No-op if already listening on the configured port. |
| `cowork-stop`   | Kill the worker process and confirm the port is free. |
| `cowork-status` | Print PID, parent PID, port binding, and the last 20 log lines. |
| `cowork-log`    | `tail -F` the worker log. |

Manual equivalents:

```bash
./bin/start.sh
./bin/stop.sh
./bin/status.sh
```

If the systemd unit is installed (default behavior on Linux with
`systemctl` and sufficient privileges):

```bash
sudo systemctl status  claude-cowork-local
sudo systemctl restart claude-cowork-local
sudo journalctl -u     claude-cowork-local -f
```

The worker is reparented to PID 1 by the start script's subshell, so it
survives the parent shell exiting. After a system reboot, run
`cowork-start` (or rely on the systemd unit) to bring it back up.

Logs are written to `${XDG_LOG_HOME:-$HOME/.local/log}/claude-cowork-local/worker.log`.

---

## Configuration

### Changing the port

```bash
PROXY_PORT=9999 ./install.sh         # initial install
PROXY_PORT=9999 ./bin/start.sh       # one-off start
```

If a systemd unit was previously installed on the old port, regenerate
it by re-running `./install.sh` with the new `PROXY_PORT`.

### Adding another upstream

To route to a third provider (for example Groq), edit
`worker/src/index.ts`:

```ts
const GROQ_UPSTREAM = "https://api.groq.com/openai/v1";
```

…add the corresponding prefix handling in `routeConfig`, and add the route
to the root endpoint's `routes` object. Then restart the worker:

```bash
cowork-stop && cowork-start
```

A new settings template at `settings/settings.groq.template.json` follows
the same pattern as the existing two.
---

## Project structure

```
claude-cowork-local/
├── README.md
├── LICENSE
├── .gitignore
├── install.sh                  # one-shot installer
├── bin/
│   ├── start.sh
│   ├── stop.sh
│   ├── status.sh
│   └── log.sh
├── settings/
│   ├── settings.zen.template.json
│   └── settings.go.template.json
└── worker/                     # fork of cucoleadan/opencode-cowork-proxy
    ├── package.json
    ├── wrangler.toml
    ├── tsconfig.json
    └── src/
        ├── index.ts            # router
        ├── server.ts           # Bun entrypoint
        ├── auth.ts             # API key extraction/validation
        ├── cache.ts            # prompt cache key helpers
        └── translate/
            ├── request/        # Anthropic ↔ OpenAI
            ├── response/       # Anthropic ↔ OpenAI
            └── stream/         # SSE: Anthropic ↔ OpenAI
```

---

## License

MIT. See `LICENSE`.

The worker source is a derivative of
[cucoleadan/opencode-cowork-proxy](https://github.com/cucoleadan/opencode-cowork-proxy)
by `@cucoleadan`, also MIT licensed. The original copyright is preserved in
the source files.

---

## Credits

- `cucoleadan/opencode-cowork-proxy` — the upstream worker this package
  embeds.
- [Bun](https://bun.sh) — the JavaScript runtime used to run the worker
  with no build step.
- [Hono](https://hono.dev) — the HTTP framework used by the worker.
- [OpenCode](https://opencode.ai) — operator of the Zen and Go model
  services.
- [Anthropic](https://anthropic.com) — author of Claude Code.
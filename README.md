# Codex + LiteLLM Stack

Route the Codex CLI and desktop app through a LiteLLM gateway with one setup
script. LiteLLM logs requests, tracks spend, and issues a separate virtual key
for each person. Codex clients never receive the OpenAI API key.

Bring an OpenAI API key, the Codex CLI (`npm i -g @openai/codex`), and Docker
Compose.

## Setup with a coding agent

Paste this into Codex from an empty directory:

> Clone `https://github.com/BerriAI/codex-litellm-stack`, cd into it, and set up
> my Codex CLI and desktop app to use its LiteLLM gateway. Read `AGENTS.md` and
> the skills in `.agents/skills/`, then follow `codex-litellm-setup`. Ask for my
> OpenAI API key when the script needs it. Never invent or print a key. Finish
> by running `./scripts/doctor.sh` and a real Codex turn, confirm the request in
> the gateway logs, and list every file changed outside the repo.

The repo includes three skills:

| Skill | Job |
|---|---|
| `codex-litellm-setup` | preflight, setup, and end-to-end verification |
| `codex-litellm-troubleshoot` | diagnose gateway, key, model, and streaming failures |
| `codex-litellm-fleet` | prepare secret-free MDM files and per-user keys |

## Manual setup

```bash
git clone https://github.com/BerriAI/codex-litellm-stack
cd codex-litellm-stack
./scripts/setup.sh
```

The script:

- creates the ignored `docker-compose/litellm.env` at mode 600;
- starts LiteLLM and Postgres;
- mints a virtual key for Codex;
- backs up and writes `~/.codex/config.toml`;
- installs `fast`, `deep`, and `astra` profile files; and
- on macOS, installs a LaunchAgent for the desktop app.

Use another port if 4000 is occupied:

```bash
./scripts/setup.sh --port 4010
```

Start Codex and verify the request path:

```bash
./dist/start-codex.sh
./scripts/doctor.sh
codex exec "reply with the word CONNECTED and nothing else"
docker compose -f docker-compose/docker-compose.yml logs litellm --tail 20
```

The log should contain a POST to `/v1/responses`.

## The three keys

| Key | Stored in | Used by |
|---|---|---|
| `OPENAI_API_KEY` | `docker-compose/litellm.env` | LiteLLM, to authenticate upstream OpenAI requests |
| `LITELLM_MASTER_KEY` | the same file | gateway administration and virtual-key issuance |
| virtual key | `dist/start-codex.sh` and the LaunchAgent | Codex requests; budget and revoke it independently |

The OpenAI key is stored only on the gateway host. LiteLLM sends it to
`api.openai.com` over TLS; it is never distributed to Codex clients.

## Codex configuration

Setup writes this base configuration:

```toml
model = "gpt-5.6-terra"
model_provider = "litellm"

[model_providers.litellm]
name = "LiteLLM Gateway"
base_url = "http://localhost:4000/v1"
env_key = "LITELLM_API_KEY"
wire_api = "responses"
stream_idle_timeout_ms = 7200000
```

Current Codex versions load named profiles from separate files:

| Command | File | Model |
|---|---|---|
| `codex --profile fast` | `~/.codex/fast.config.toml` | `gpt-5.6-luna`, low effort |
| `codex --profile deep` | `~/.codex/deep.config.toml` | `gpt-5.6-sol`, high effort |
| `codex --profile astra` | `~/.codex/astra.config.toml` | `gpt-6-astra`, high effort |

Model IDs pass through unchanged. The gateway publishes the explicit model
list plus a `"*"` catch-all. Remove the catch-all to restrict access.

## Desktop app

The desktop app reads `~/.codex/config.toml`, but apps opened from Finder, the
Dock, or Spotlight do not inherit shell variables. On macOS, setup installs
`~/Library/LaunchAgents/com.codex-litellm-stack.env.plist` so the app receives
its virtual key at login. Quit and reopen Codex after setup.

The plist is mode 600, but every process in the login session can read the
virtual key. Keep the desktop integration only if that tradeoff is acceptable:

```bash
./scripts/setup.sh --no-desktop
./scripts/uninstall-desktop-env.sh
```

## Fleet rollout

```bash
./scripts/setup.sh --public-url https://llm-gw.corp.example --enterprise
```

This writes four secret-free files to `dist/`: `config.toml` and the three
profile files. Distribute them to `~/.codex/` with MDM and deliver one
`LITELLM_API_KEY` per person through your secret manager.

Before a fleet rollout, provide TLS, durable Postgres, unbuffered SSE, and a
rollback path. The included Compose stack is a local target, not a production
deployment.

## Repository layout

```text
AGENTS.md                  instructions for coding agents
.agents/skills/            setup, troubleshooting, and fleet procedures
docker-compose/            LiteLLM, Postgres, config, and profile templates
scripts/setup.sh           configure and verify the stack
scripts/doctor.sh          test the complete request path
docs/index.html            GitHub Pages site
dist/                      generated files containing a live virtual key
```

## Options

```text
./scripts/setup.sh
  --port <n>            gateway port (default 4000)
  --public-url <url>    URL clients use
  -y, --yes             fail instead of prompting
  --enterprise          generate MDM files
  --no-desktop          skip the macOS LaunchAgent

./scripts/doctor.sh
  --base <url>          gateway URL
  --model <name>        model to test (default gpt-5.6-terra)
```

`docker compose down -v` deletes every issued virtual key and all spend
history. Use `restart litellm` unless you intend to erase the Postgres volume.

## License

MIT. See [LICENSE](LICENSE).

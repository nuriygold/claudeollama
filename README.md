# claudeollama

`claudeollama` is not a stock Claude Code install.
`claudeollama` is a local launcher that runs Claude Code against a LiteLLM proxy through a small local filtering proxy.

It is a local launcher that starts Claude Code inside a controlled wrapper:

- Claude Code talks to a temporary local HTTP filter proxy
- that proxy forwards to an existing LiteLLM instance
- LiteLLM then routes to the actual upstream model provider
- Claude runs under a clean temporary `HOME`
- MCP servers are injected from a dedicated `claudeollamamcps.json` file

In other words, this build is better understood as a Claude Code-compatible agent shell running through local infrastructure, not as the traditional Claude consumer app flow.

## How this build works

At runtime `bin/claudeollama` does all of the following:

1. starts local LiteLLM automatically when `REAL_LITELLM_URL` is loopback
2. starts a temporary local proxy on `127.0.0.1:<FILTER_PORT>`
3. strips `output_config` from JSON requests before forwarding them upstream
4. creates a clean temporary `HOME`
5. copies only the wrapper config files into that temporary home
6. launches `claude` with `ANTHROPIC_BASE_URL` pointed at the local filter proxy
7. passes `--mcp-config` with the resolved `claudeollamamcps.json` path when available

This means the launcher itself is the product. There is no separate application server or library layer that owns this behavior.

## How this differs from traditional Claude Code

Compared with a normal Claude Code setup, this launcher has several important differences:

- it does not talk directly to Anthropic by default
- it uses LiteLLM as the model backend
- it rewrites requests through a local compatibility proxy
- it launches in a clean temporary `HOME` instead of reusing your normal Claude session state
- it can run without the normal Claude consumer login flow because auth can come from local wrapper configuration and API-style credentials
- MCP loading is driven by a dedicated MCP config file instead of relying only on standard local Claude registration

Practically, treat this environment as a wrapped Claude runtime with special config resolution, not as a plain `claude` install.

## Files that matter

- `bin/claudeollama` — launcher script, filter-proxy generator, clean-home setup, Claude invocation, MCP injection
- `bin/bootstrap-claudeollama-config` — copies your real MCP config into this repo for local use without committing secrets
- `config/ollama.config.yaml` — repo-local LiteLLM config for `gpt-oss:20b` via Ollama
- `config/ollama.env` — minimal environment for the local LiteLLM launcher
- `.claude/settings.local.json` — repo-local Claude settings used by default
- `.claude/claudeollamamcps.json.example` — safe MCP starter config you can copy locally

## Expected local dependencies

- `claude`
- `python3`
- Python package `httpx` for the local filter proxy transport (`python3 -m pip install httpx`)
- `curl`
- `lsof`

## Required configuration concepts

There are three distinct configuration layers to keep straight:

### 1. LiteLLM upstream config

If `REAL_LITELLM_URL` points at a local loopback address, `claudeollama` starts LiteLLM for you from `config/ollama.config.yaml`. If you point `REAL_LITELLM_URL` at something remote, it expects that service to already be running.

The launcher expects:

- upstream base URL: `REAL_LITELLM_URL` (default `http://127.0.0.1:4300`)
- auth key when required: `LITELLM_KEY`
- model name passed through Claude env: `ANTHROPIC_MODEL`

### 2. Claude merged config

The launcher creates a temporary clean `HOME`, then copies the wrapper config files into it.

The effective config sources are:

- this repo's `.claude` if present
- your real `~/.claude`
- any explicit `CLAUDE_CONFIG_DIR` overlay supplied at launch time

Later layers win.

This is why repo-local config can override home-level defaults even though Claude is running under a temporary home.

As a stability guard, the launcher strips merged `hooks` from the clean-home `settings.json` before Claude starts. That prevents unrelated `SessionStart` automations from your real home or parent-directory config from firing inside the isolated wrapper session.

### 3. MCP config for this wrapper

This build has a dedicated MCP config path.

Resolution order is:

1. `MCP_CONFIG_PATH` when explicitly set
2. `~/.claude/claudeollamamcps.json`
3. repo-local `.claude/claudeollamamcps.json` copied into the merged clean home
4. merged `CLAUDE_CONFIG_DIR/claudeollamamcps.json` when neither direct source exists
5. no MCP config if none of those exist

The important operational detail is that this wrapper uses `claudeollamamcps.json`, not the normal general-purpose Claude MCP registration path as its primary integration surface.

For local operator use, the usual real file is:

- `~/.claude/claudeollamamcps.json`

`bin/bootstrap-claudeollama-config` does this:

- if `~/.claude/claudeollamamcps.json` already exists, it leaves it in place and reminds you that the launcher prefers it
- otherwise it seeds `~/.claude/claudeollamamcps.json` from the checked-in example

The committed repo example is the safe template. Personal tokens should live in your home MCP config or in environment variables, not in the tracked repo-local file.

## Telegram wiring in this build

Telegram for this environment should be wired into the `claudeollamamcps.json` file used by the launcher.

Current live path:

- MCP config file: `~/.claude/claudeollamamcps.json`
- Telegram bridge server: `/Users/claw/openclaw/workspace/plugins/telegram/server.ts`
- Telegram state: `~/.claude/channels/telegram/.env`
- Telegram access control: `~/.claude/channels/telegram/access.json`

Example MCP entry:

```json
{
  "telegram": {
    "command": "bun",
    "args": [
      "/Users/claw/openclaw/workspace/plugins/telegram/server.ts"
    ]
  }
}
```

The Telegram bridge also has local runtime dependencies in its own directory, so if the bridge fails to start, check the plugin directory first:

- `/Users/claw/openclaw/workspace/plugins/telegram/package.json`
- `/Users/claw/openclaw/workspace/plugins/telegram/node_modules`

## Usage

```bash
bin/bootstrap-claudeollama-config
bin/claudeollama
```

`bin/bootstrap-claudeollama-config` does this:

- if `~/.claude/claudeollamamcps.json` exists, it leaves that file as the preferred source
- otherwise it seeds `~/.claude/claudeollamamcps.json` from the checked-in example

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="..."
export SUPABASE_ACCESS_TOKEN="..."
export VERCEL_API_KEY="..."

REAL_LITELLM_URL=http://127.0.0.1:4300 \
ANTHROPIC_MODEL=gpt-oss:20b \
MCP_CONFIG_PATH="$HOME/.claude/claudeollamamcps.json" \
bin/claudeollama
```

## Environment variables
## LiteLLM config example

The checked-in config keeps the default `gpt-oss:20b` model mapping for the local Ollama server:

```yaml
model_list:
  - model_name: gpt-oss:20b
    litellm_params:
      model: ollama_chat/gpt-oss:20b
      api_base: http://127.0.0.1:11434
      keep_alive: 8m
    model_info:
      supports_function_calling: true

litellm_settings:
  drop_params: true
```

The repo already includes `config/ollama.config.yaml` and `config/ollama.env` for the local model path.

## Environment

Supported variables:

- `REAL_LITELLM_URL`
- `FILTER_PORT`
- `ANTHROPIC_MODEL`
- `CLAUDE_CONFIG_DIR`
- `MCP_CONFIG_PATH`
- `LITELLM_KEY`
- `FILTER_SCRIPT_PATH`
- `FILTER_LOG_PATH`
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS`
- `FILTER_UPSTREAM_TIMEOUT`
- `FILTER_UPSTREAM_CONNECT_TIMEOUT`
- `FILTER_UPSTREAM_RETRIES`
- `FILTER_UPSTREAM_RETRY_BACKOFF`
- `FILTER_MAX_CONNECTIONS`
- `FILTER_MAX_KEEPALIVE_CONNECTIONS`

For launcher auth, export the client key as `LITELLM_KEY` before launch if the downstream LiteLLM path requires auth and you do not want the wrapper to prompt later.

```bash
export LITELLM_KEY="<your-litellm-key>"
```

If your LiteLLM server is configured to require a master key, that server-side config is typically named `LITELLM_MASTER_KEY` in the LiteLLM process configuration.

Default behavior:

- `CLAUDE_CONFIG_DIR` defaults to this repo's `.claude` directory when present
- `MCP_CONFIG_PATH` defaults to `claudeollamamcps.json` in the merged Claude config directory
- if that resolved MCP file is missing, the launcher continues without `--mcp-config`

## Operational notes

- The launcher creates a temporary clean `HOME` specifically so it does not reuse an existing Claude login session.
- The clean-home launch also starts from `env -i`, so inherited wrapper-mode environment from the parent Claude process is not passed through into the nested Claude session.
- If `REAL_LITELLM_URL` is local loopback, the launcher starts LiteLLM automatically from `config/ollama.config.yaml`.
- The nested Claude session enforces a minimum `CLAUDE_CODE_MAX_OUTPUT_TOKENS` floor of 256000 so long responses do not trip the lower inherited ceiling.
- The clean-home session keeps only the Superpowers plugin enabled by default.
- Temporary artifacts are written under `/tmp` or `$TMPDIR` unless overridden, and the temp clean home is removed with Python `shutil.rmtree` to avoid noisy macOS `rm` failures on transient npm cache trees.
- The launcher now waits briefly for `FILTER_PORT` to be released before rebinding, and the embedded TCP server enables address reuse so immediate restarts do not trip over macOS socket teardown timing.
- If an already-running filter proxy is listening on `FILTER_PORT`, the launcher reuses it instead of trying to replace it.
- The filter proxy binds to `127.0.0.1` only by default via `FILTER_BIND_HOST`.
- The proxy now rejects unexpected `Host` headers; override the allowlist with `FILTER_ALLOWED_HOSTS` only when you intentionally need more than `127.0.0.1`, `localhost`, or the current hostname.
- Upstream forwarding now uses a reusable `httpx` client with connection pooling plus bounded retries for transient GET/HEAD/OPTIONS failures; tune it with the `FILTER_UPSTREAM_*` and `FILTER_MAX_*` variables when needed.
- Changes to proxy behavior must be made in `bin/claudeollama` because the proxy script is generated there at runtime.
- If MCP servers appear to be missing inside a launched session, check `MCP_CONFIG_PATH` resolution before checking standard Claude local MCP registration.

## Verification

There is no formal test suite today.

Before launch, the wrapper copies config files from these sources in this order:

- your real `~/.claude`
- this repo's `.claude` if present
- any explicit `CLAUDE_CONFIG_DIR` overlay supplied at launch time

Later layers win, so project-local config can override home-level defaults.

Primary verification steps:

```bash
bash -n bin/claudeollama
```

Then run the launcher:

```bash
bin/claudeollama
```

A healthy run should show all of the following:

- the local filter proxy starts and listens on `FILTER_PORT`
- Claude launches with `ANTHROPIC_BASE_URL` pointed at the filter proxy
- when configured, Claude launches with `--mcp-config` pointing at the resolved `claudeollamamcps.json`

When debugging failures:

- inspect the configured filter log path
- confirm the resolved MCP config file exists and contains the expected server entries
- confirm bridge-specific runtimes, such as Telegram, have their own local dependencies installed

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

`claudeollama` is a thin local launcher that runs Claude Code against a local LiteLLM proxy through a temporary local filtering proxy.

The key compatibility behavior is that the launcher inserts a local HTTP proxy that strips `output_config` from JSON requests before forwarding them to LiteLLM. Claude is then started in a clean temporary `HOME` so it does not reuse an existing Claude login session.

## Common commands

### Run the launcher

```bash
bin/claudeollama
```

### Run with a non-default LiteLLM endpoint or model

```bash
REAL_LITELLM_URL=http://127.0.0.1:4300 ANTHROPIC_MODEL=gpt-oss:20b bin/claudeollama
```

### Run with a specific Claude or MCP config

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="..."
export SUPABASE_ACCESS_TOKEN="..."
export VERCEL_API_KEY="..."

CLAUDE_CONFIG_DIR="$HOME/.claude" MCP_CONFIG_PATH="$HOME/.claude/claudeollamamcps.json" bin/claudeollama
```

### Check shell syntax

```bash
bash -n bin/claudeollama
```

### Run the embedded filter proxy directly for debugging

There is no standalone checked-in proxy file. The launcher writes a temporary Python script to `FILTER_SCRIPT_PATH` and runs it in the background.

Useful environment variables when debugging the launcher:

```bash
FILTER_PORT=4301 FILTER_LOG_PATH=/tmp/claudeollama-filter.log FILTER_SCRIPT_PATH=/tmp/claudeollama_strip_output_config_proxy.py bin/claudeollama
```

### Inspect LiteLLM reachability manually

```bash
curl -sS http://127.0.0.1:4300/v1/models
curl -sS http://127.0.0.1:4301/v1/models -H "Authorization: Bearer <key>"
```

## Development dependencies

The README documents these local dependencies:

- `claude`
- `python3`
- `curl`
- `lsof`
- a local LiteLLM instance when `REAL_LITELLM_URL` points at loopback, defaulting to `http://127.0.0.1:4300`

## Architecture

### High-level flow

1. `bin/claudeollama` starts local LiteLLM automatically when `REAL_LITELLM_URL` is loopback.
2. The launcher kills any existing listener on `FILTER_PORT`.
3. It writes a temporary Python HTTP proxy script to `FILTER_SCRIPT_PATH`.
4. That proxy forwards requests to `REAL_LITELLM_URL`, recursively removing `output_config` from JSON request bodies.
5. The launcher checks that the local filter port is listening before continuing.
6. It creates a temporary clean home directory with `mktemp -d`.
7. It copies only the wrapper config files into that clean home, then launches `claude` under `env -i`, pointing `ANTHROPIC_BASE_URL` at the filter proxy and passing through the selected model and MCP config.

### Important implementation details

- The launcher is the product. There is no separate library or multi-module app structure here.
- The embedded Python proxy is generated inside the shell script, so changes to proxy behavior are made in `bin/claudeollama`, not in a separate Python file.
- Proxy behavior is intentionally narrow: it strips `output_config`, rejects unexpected `Host` headers, and otherwise forwards headers/body with minimal changes.
- The proxy binds to `127.0.0.1` only by default, exposes `FILTER_BIND_HOST` and `FILTER_ALLOWED_HOSTS` for explicit overrides, and uses `socketserver.ThreadingTCPServer` with address reuse enabled plus a short launcher-side port-release wait so immediate restarts do not fail on macOS bind timing.
- Before starting a new proxy, the launcher checks whether `FILTER_PORT` is already owned by a listener and reuses it instead of stealing the port from an external service-managed instance.
- Upstream transport now uses a shared `httpx` client with pooling and bounded retries for transient idempotent-request failures; stability tuning lives in the `FILTER_UPSTREAM_*` and `FILTER_MAX_*` environment variables.
- Claude runs with `env -i`, so environment propagation is explicit. If a tool stops working, check whether the required variable is being passed through in the `env -i` block.
- The nested Claude session enforces a minimum `CLAUDE_CODE_MAX_OUTPUT_TOKENS` floor of 256000 so long responses do not trip the lower inherited ceiling.
- The merged clean-home `settings.json` has its `hooks` key stripped before launch so home or parent `SessionStart` automations do not run inside the isolated wrapper session.
- The launcher only forwards explicit script arguments into the nested Claude invocation, which prevents inherited outer-session `--print` mode from breaking the inner interactive launch.
- The clean HOME only keeps the Superpowers plugin enabled by default.
- Temporary artifacts are written to `/tmp` or `$TMPDIR` by default: the generated Python proxy, proxy log, LiteLLM model response captures, and the temporary Claude home directory. Temp-home cleanup uses Python `shutil.rmtree` to avoid noisy `rm` teardown failures on transient npm cache trees.

## Key configuration knobs

Environment variables supported by the launcher:

- `REAL_LITELLM_URL`
- `FILTER_PORT`
- `ANTHROPIC_MODEL`
- `CLAUDE_CONFIG_DIR`
- `MCP_CONFIG_PATH`
- `LITELLM_KEY`
- `FILTER_SCRIPT_PATH`
- `FILTER_LOG_PATH`
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS`

## Files that matter

- `bin/claudeollama`: the main launcher, proxy generator, environment isolation logic, and Claude invocation.
- `config/ollama.config.yaml`: repo-local LiteLLM model mapping for `gpt-oss:20b` via Ollama.
- `README.md`: usage expectations and required local dependencies.
- `.claude/settings.local.json`: local Claude Code permissions checked into this repo for common shell and GitHub auth commands.

## Change guidance

When editing this repo, preserve these invariants unless the task explicitly changes them:

- Claude must continue to run against the local filter proxy, not directly against the upstream LiteLLM URL.
- The filter proxy must continue stripping `output_config` from JSON request bodies.
- Claude should continue launching in a temporary clean `HOME`.
- The launcher should fail fast when the filter proxy is unreachable.
- No em dashes in docs or messages for this repo.
- Prefer internet-backed verification over stale local assumptions when validating changes.
- For Codex-created testing or deployment branches in this repo, use the `joy/` prefix. This prefix preference is for Codex work only, not for other CLIs or tools.

## Verification

There is no formal test suite in the repository today.

For changes here, the main verification path is:

```bash
bash -n bin/claudeollama
```

Then run the launcher against a reachable LiteLLM instance:

```bash
bin/claudeollama
```

Successful verification should show all of the following:

- the local filter proxy starts and listens on `FILTER_PORT`
- Claude launches with `ANTHROPIC_BASE_URL` pointed at the filter proxy rather than the upstream LiteLLM URL

When debugging failures, inspect the filter log path printed or configured via `FILTER_LOG_PATH`.

Temporary artifacts are expected during runs. The generated proxy script, filter log, model response captures, and clean temporary `HOME` are written under `/tmp` or `$TMPDIR` unless overridden.

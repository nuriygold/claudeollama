# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

`claudelitellm` is a thin local launcher that runs Claude Code against an existing LiteLLM proxy through a temporary local filtering proxy.

The key compatibility behavior is that the launcher inserts a local HTTP proxy that strips `output_config` from JSON requests before forwarding them to LiteLLM. Claude is then started in a clean temporary `HOME` so it does not reuse an existing Claude login session.

## Common commands

### Run the launcher

```bash
bin/claudelitellm
```

### Run with a non-default LiteLLM endpoint or model

```bash
REAL_LITELLM_URL=http://127.0.0.1:4000 ANTHROPIC_MODEL=gpt-5.4 bin/claudelitellm
```

### Run with a specific Claude or MCP config

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="..."
export SUPABASE_ACCESS_TOKEN="..."
export VERCEL_API_KEY="..."

CLAUDE_CONFIG_DIR="$HOME/.claude" MCP_CONFIG_PATH="$HOME/.claude/claudelitellmmcps.json" bin/claudelitellm
```

### Check shell syntax

```bash
bash -n bin/claudelitellm
```

### Run the embedded filter proxy directly for debugging

There is no standalone checked-in proxy file. The launcher writes a temporary Python script to `FILTER_SCRIPT_PATH` and runs it in the background.

Useful environment variables when debugging the launcher:

```bash
FILTER_PORT=4001 FILTER_LOG_PATH=/tmp/litellm-filter.log FILTER_SCRIPT_PATH=/tmp/litellm_strip_output_config_proxy.py bin/claudelitellm
```

### Inspect LiteLLM reachability manually

```bash
curl -sS http://127.0.0.1:4000/v1/models
curl -sS http://127.0.0.1:4001/v1/models -H "Authorization: Bearer <key>"
```

## Development dependencies

The README documents these local dependencies:

- `claude`
- `python3`
- `curl`
- `lsof`
- a running LiteLLM instance, defaulting to `http://127.0.0.1:4000`

## Architecture

### High-level flow

1. `bin/claudelitellm` validates that the upstream LiteLLM `/v1/models` endpoint is reachable.
2. If LiteLLM responds with `401` or `403`, the launcher prompts for `LITELLM_KEY` and retries.
3. The launcher kills any existing listener on `FILTER_PORT`.
4. It writes a temporary Python HTTP proxy script to `FILTER_SCRIPT_PATH`.
5. That proxy forwards requests to `REAL_LITELLM_URL`, recursively removing `output_config` from JSON request bodies.
6. The launcher verifies the filter proxy by calling its `/v1/models` endpoint.
7. It creates a temporary clean home directory with `mktemp -d`.
8. It launches `claude` under `env -i`, pointing `ANTHROPIC_BASE_URL` at the filter proxy and passing through the selected model and MCP config.

### Important implementation details

- The launcher is the product. There is no separate library or multi-module app structure here.
- The embedded Python proxy is generated inside the shell script, so changes to proxy behavior are made in `bin/claudelitellm`, not in a separate Python file.
- Proxy behavior is intentionally narrow: it strips `output_config`, rejects unexpected `Host` headers, and otherwise forwards headers/body with minimal changes.
- The proxy binds to `127.0.0.1` only by default, exposes `FILTER_BIND_HOST` and `FILTER_ALLOWED_HOSTS` for explicit overrides, and uses `socketserver.ThreadingTCPServer`.
- Upstream transport now uses a shared `httpx` client with pooling and bounded retries for transient idempotent-request failures; stability tuning lives in the `FILTER_UPSTREAM_*` and `FILTER_MAX_*` environment variables.
- Claude runs with `env -i`, so environment propagation is explicit. If a tool stops working, check whether the required variable is being passed through in the `env -i` block.
- The merged clean-home `settings.json` has its `hooks` key stripped before launch so home or parent `SessionStart` automations do not run inside the isolated wrapper session.
- The launcher only forwards explicit script arguments into the nested Claude invocation, which prevents inherited outer-session `--print` mode from breaking the inner interactive launch.
- `GH_CONFIG_DIR` is forwarded from the user environment so GitHub auth can still work inside the clean HOME session.
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
- `LITELLM_MODELS_OUTPUT`
- `FILTER_MODELS_OUTPUT`

## Files that matter

- `bin/claudelitellm`: the main launcher, proxy generator, environment isolation logic, and Claude invocation.
- `config/litellm.config.yaml.example`: example LiteLLM model mapping for `gpt-5.4` via Azure and use of `LITELLM_MASTER_KEY`.
- `README.md`: usage expectations and required local dependencies.
- `.claude/settings.local.json`: local Claude Code permissions checked into this repo for common shell and GitHub auth commands.

## Change guidance

When editing this repo, preserve these invariants unless the task explicitly changes them:

- Claude must continue to run against the local filter proxy, not directly against the upstream LiteLLM URL.
- The filter proxy must continue stripping `output_config` from JSON request bodies.
- Claude should continue launching in a temporary clean `HOME`.
- The launcher should fail fast when LiteLLM or the filter proxy is unreachable.
- No em dashes in docs or messages for this repo.
- Prefer internet-backed verification over stale local assumptions when validating changes.
- For Codex-created testing or deployment branches in this repo, use the `joy/` prefix. This prefix preference is for Codex work only, not for other CLIs or tools.

## Verification

There is no formal test suite in the repository today.

For changes here, the main verification path is:

```bash
bash -n bin/claudelitellm
```

Then run the launcher against a reachable LiteLLM instance:

```bash
bin/claudelitellm
```

Successful verification should show all of the following:

- the upstream LiteLLM `/v1/models` endpoint returns `200`, either anonymously or with `LITELLM_KEY`
- the local filter proxy `/v1/models` endpoint also returns a successful response when queried with the same auth
- Claude launches with `ANTHROPIC_BASE_URL` pointed at the filter proxy rather than the upstream LiteLLM URL

When debugging failures, inspect the filter log path printed or configured via `FILTER_LOG_PATH`.

Temporary artifacts are expected during runs. The generated proxy script, filter log, model response captures, and clean temporary `HOME` are written under `/tmp` or `$TMPDIR` unless overridden.

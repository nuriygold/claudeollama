# claudelitellm

`claudelitellm` is a local launcher that runs Claude Code against an existing LiteLLM proxy through a small local filtering proxy.

By default it builds a temporary merged config home so Claude Code can use existing `.claude` and `.codex` settings even though the launcher runs under a clean temporary `HOME`.

## Files

- `bin/claudelitellm` — launcher script
- `bin/bootstrap-claude-config` — copies your real MCP config into this repo for local use without committing secrets
- `config/litellm.config.yaml.example` — example LiteLLM config
- `.claude/settings.local.json` — repo-local Claude settings used by default
- `.claude/claudelitellmmcps.json.example` — safe MCP starter config you can copy locally

## Expected local dependencies

- `claude`
- `python3`
- `curl`
- `lsof`
- a running LiteLLM instance, defaulting to `http://127.0.0.1:4000`

## Usage

```bash
export LITELLM_KEY="<your-litellm-key>"
bin/bootstrap-claude-config
bin/claudelitellm
```

`bin/bootstrap-claude-config` does this:

- if `~/.claude/claudelitellmmcps.json` exists, it copies that real file into `.claude/claudelitellmmcps.json` for this repo
- otherwise it seeds `.claude/claudelitellmmcps.json` from the checked-in example

The real `.claude/claudelitellmmcps.json` inside this repo is gitignored so you can keep local auth there without publishing it.

## Environment

Optional variables:

- `REAL_LITELLM_URL`
- `FILTER_PORT`
- `ANTHROPIC_MODEL`
- `CLAUDE_CONFIG_DIR`
- `MCP_CONFIG_PATH`
- `LITELLM_KEY`

For launcher auth, export the client key as `LITELLM_KEY` before launch.

```bash
export LITELLM_KEY="<your-litellm-key>"
```

If your LiteLLM server is configured to require a master key, that server-side config is typically named `LITELLM_MASTER_KEY` in the LiteLLM process configuration.

Default behavior:

- `CLAUDE_CONFIG_DIR` defaults to this repo's `.claude` directory when present
- `MCP_CONFIG_PATH` defaults to `.claude/claudelitellmmcps.json` in this repo
- if that repo-local MCP file is missing, the launcher falls back to `~/.claude/claudelitellmmcps.json`

## Notes

The launcher creates a temporary clean HOME for Claude Code so it does not reuse an existing Claude login session.

Before launch it overlays config directories in this order:

- this repo's `.claude` and `.codex` if present
- your real `~/.claude` and `~/.codex`
- any `.claude` or `.codex` directories found while walking from `/` down to the current working directory

Later layers win, so project-local config can override home-level defaults.

The checked-in MCP example includes the same server shape as the existing local setup, including `memory`, so it is easy to mirror a pre-existing Claude MCP environment without committing live credentials.

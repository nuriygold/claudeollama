# claudelitellm

`claudelitellm` is a local launcher that runs Claude Code against an existing LiteLLM proxy through a small local filtering proxy.

By default it now prefers this repo's `.claude` directory for Claude settings, while still falling back to your real `~/.claude/claudelitellmmcps.json` when that is the only MCP config available.

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

Default behavior:

- `CLAUDE_CONFIG_DIR` defaults to this repo's `.claude` directory when present
- `MCP_CONFIG_PATH` defaults to `.claude/claudelitellmmcps.json` in this repo
- if that repo-local MCP file is missing, the launcher falls back to `~/.claude/claudelitellmmcps.json`

## Notes

The launcher creates a temporary clean HOME for Claude Code so it does not reuse an existing Claude login session.

The checked-in MCP example includes the same server shape as the existing local setup, including `memory`, so it is easy to mirror a pre-existing Claude MCP environment without committing live credentials.

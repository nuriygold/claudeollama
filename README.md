# claudelitellm

`claudelitellm` is a local launcher that runs Claude Code against a LiteLLM proxy through a small local filtering proxy.

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
- `litellm` if you want the launcher to auto-start the repo-local LiteLLM on `http://127.0.0.1:4000`

## Usage

```bash
bin/bootstrap-claude-config
bin/claudelitellm
```

If `REAL_LITELLM_URL` points at the default local endpoint and nothing is listening on port `4000`, the launcher will try to start LiteLLM itself using:

- `config/litellm.env`
- `config/litellm.config.yaml`

`bin/bootstrap-claude-config` does this:

- if `~/.claude/claudelitellmmcps.json` exists, it copies that real file into `.claude/claudelitellmmcps.json` for this repo
- otherwise it seeds `.claude/claudelitellmmcps.json` from the checked-in example

The real `.claude/claudelitellmmcps.json` inside this repo is gitignored so you can keep local auth there without publishing it.

## LiteLLM config example

The checked-in example config keeps the default `gpt-5.4` model mapping for Azure:

```yaml
model_list:
  - model_name: gpt-5.4
    litellm_params:
      model: azure/gpt-5.4
      api_base: os.environ/AZURE_API_BASE
      api_key: os.environ/AZURE_API_KEY
      api_version: os.environ/AZURE_API_VERSION

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

litellm_settings:
  drop_params: true
```

Copy `config/litellm.config.yaml.example` to `config/litellm.config.yaml` before starting the repo-local LiteLLM instance.

## Environment

Optional variables:

- `REAL_LITELLM_URL`
- `FILTER_PORT`
- `ANTHROPIC_MODEL`
- `CLAUDE_CONFIG_DIR`
- `MCP_CONFIG_PATH`
- `LITELLM_KEY`

For launcher auth, export the client key as `LITELLM_KEY` before launch if you are connecting to an already-running LiteLLM that requires auth and you do not want the launcher to prompt.

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

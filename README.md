# claudelitellm

`claudelitellm` is a local launcher that runs Claude Code against an existing LiteLLM proxy through a small local filtering proxy.

## Files

- `bin/claudelitellm` — launcher script
- `config/litellm.config.yaml.example` — example LiteLLM config

## Expected local dependencies

- `claude`
- `python3`
- `curl`
- `lsof`
- a running LiteLLM instance, defaulting to `http://127.0.0.1:4000`

## Usage

```bash
bin/claudelitellm
```

## Environment

Optional variables:

- `REAL_LITELLM_URL`
- `FILTER_PORT`
- `ANTHROPIC_MODEL`
- `CLAUDE_CONFIG_DIR`
- `MCP_CONFIG_PATH`
- `LITELLM_KEY`

## Notes

The launcher creates a temporary clean HOME for Claude Code so it does not reuse an existing Claude login session.

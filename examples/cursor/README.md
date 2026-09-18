# Cursor integration examples

Use the [integration guide](../../docs/integrations/cursor.md) for the workflow on
your actual project.

`mcp.json.example` is a project-bound stdio MCP template with Cursor's documented
fields (`type`, `command`, `args`). Replace both absolute paths and merge the
`awr` server into an existing `.cursor/mcp.json` or `~/.cursor/mcp.json`.
Preserve other servers. `cwd` is omitted: `--project` is absolute, so AWR does
not need it. This file is a template, not an installed server or a lifecycle
hook. Inspect Cursor **Customize** and **Output → MCP Logs** to confirm a live
connection. Do not commit a project `mcp.json` that points at a machine-local
AWR root or debug binary.

The reusable CLI walkthrough remains [examples/codex/lifecycle.py](../codex/README.md).
Running it does not start Cursor, register MCP, or prove native tool invocation.

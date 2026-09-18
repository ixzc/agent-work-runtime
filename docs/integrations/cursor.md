# Using AWR from Cursor

> **Layer: L1 host note.** Use the [L0 session workflow](session-workflow.md)
> for the shared CLI/MCP path. This page records Cursor merge paths, Cloud
> Agent limits, and a dated check. It does not add an L2 installer.
> See [host integration layers](README.md).

AWR supplies project facts, claims and checkpoints. Cursor owns the model, Agent
Window and native conversation IDs. Checked against **Cursor 3.18.9** and the
[official MCP documentation](https://cursor.com/docs/mcp) on **2026-09-17**.

## MCP merge paths

Install **0.4.0** and use absolute paths to the packaged binaries:

```sh
npm install -g @originoneai/agent-work-runtime@0.4.0
# or: python -m pip install agent-work-runtime==0.4.0
command -v awr
command -v awr-mcp
```

Contributors may instead `cargo build --locked -p awr-cli -p awr-mcp`. Initialize
the project first; starting `awr-mcp` does not create a database. Merge
[the stdio template](../../examples/cursor/mcp.json.example) into one of:

| Scope | Path |
| --- | --- |
| Project | `.cursor/mcp.json` in the opened workspace |
| User | `~/.cursor/mcp.json` |

Prefer a single `awr` definition. When both files exist, confirm in **Customize**
and by calling `awr_project_status` which server the Agent Window actually
attached. Do not commit a project file with a machine-local AWR root or
credentials. `${workspaceFolder}` is the folder that contains `.cursor/mcp.json`,
not automatically the AWR `--project` root. If the Cursor CLI omits a
project-scoped server, use the user file or the desktop Agent Window.

```json
{
  "mcpServers": {
    "awr": {
      "type": "stdio",
      "command": "/absolute/path/to/awr-mcp",
      "args": ["--project", "/absolute/path/to/initialized/project"]
    }
  }
}
```

`type` is required in Cursor's STDIO field table. `cwd` is not in that set and
is safe to omit when `--project` is absolute. After editing, reload or toggle the
server in **Customize**. Open **Output → MCP Logs** if it fails to start. A
configured server is not a live connection.

Current AWR defaults to **grouped** MCP tools (`awr_query`, `awr_context`, …).
On 2026-09-17 this Agent Window also invoked the flat names `awr_project_status`
and `awr_work_ready`. If `tools/list` shows domain tools, call through them as in
the [L0 workflow](session-workflow.md#generic-mcp). Set
`AWR_MCP_TOOL_EXPOSURE_MODE=flat` only when the host cannot route through domains.

## Bind identity

There is no `--client cursor`. Use L0 generic identity with a host-prefixed
conversation ID. `--provider cursor` on `session start` is a display label only.

```sh
awrj client bind --client generic \
  --external-session cursor:CURSOR_CONVERSATION_ID \
  --work "$AWR_WORK" --session "$AWR_SESSION"
```

`--session` attaches to a **still-active** AWR session. `--from-session` resumes
a predecessor and binds the new conversation to the successor. Do not pass both.
`--client` is required. `--client cursor` is `InvalidInput` on bind/hook/show.
`awr client install --client cursor` is `Unsupported` (Codex-only L2). Neither
error means a missing binary or a failed MCP connection.

## Shared HTTP

AWR uses static bearer tokens, not Cursor OAuth. Do not commit the token. HTTP
tools require an explicit `project` key.

Local desktop smoke test only (same host as the service):

```json
{
  "mcpServers": {
    "awr": {
      "url": "http://127.0.0.1:8080/mcp",
      "headers": {
        "Authorization": "Bearer ${env:AWR_ENGINEERING_TOKEN}"
      }
    }
  }
}
```

Cloud Agents cannot reach laptop loopback. They need a reachable HTTPS front
door and project roots on that server:

```json
{
  "mcpServers": {
    "awr": {
      "url": "https://awr.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:AWR_ENGINEERING_TOKEN}"
      }
    }
  }
}
```

This note does not verify Cloud Agent, team-marketplace or enterprise-allowlist
deployment.

## Hooks and verification

Cursor documents `sessionStart`, `sessionEnd` and `preCompact` in
`.cursor/hooks.json` / `~/.cursor/hooks.json`.
[Official hook documentation](https://cursor.com/docs/agent/hooks)
This note does not install them. Use the [L0 checkpoint path](session-workflow.md)
until a live hook trigger and checkpoint receipt exist.

On 2026-09-17 this host initialized a copy of `examples/basic`, completed the
L0 CLI lifecycle with `--provider cursor`, and called `awr_project_status` plus
`awr_work_ready` from the Cursor 3.18.9 Agent Window. The server reported project
`AWR example`, `EXAMPLE-001` ready, `freshness_basis: source_verified_readonly`.
That confirms stdio discovery and those two read tools in this client. It does
not claim a Cursor model turn, grouped-catalog recheck, Cloud Agent access, an
installed hook, or business acceptance.

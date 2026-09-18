# Using AWR from Cursor

AWR supplies current project facts, work context and recoverable session memory.
Cursor performs the work with its own tools and models. The local workflow below
uses AWR's CLI and stdio MCP against the same initialized project. Current AWR
**0.4.0** packages include session/claim lifecycle and the
[shared HTTP service](../reference/mcp-service.md) (both since 0.3.3) for
multiple projects and clients.

This guide was checked against **Cursor 3.18.9** and the
[official MCP documentation](https://cursor.com/docs/mcp) on **2026-09-17**.
Cursor's conversation ID, Agent Window and Cloud Agents are separate from AWR
session IDs. Provider/model labels in AWR records do not configure or invoke Cursor.

## Install and bind the project

Install **0.4.0** and resolve absolute paths to `awr` and `awr-mcp`:

```sh
npm install -g @originoneai/agent-work-runtime@0.4.0
# or, in a Python virtual environment
python -m pip install agent-work-runtime==0.4.0
command -v awr
command -v awr-mcp
```

From an AWR checkout, contributors can instead build:

```sh
cargo build --locked -p awr-cli -p awr-mcp
```

Use absolute executable and project paths. Initialize the target project with a
reviewed source manifest as described in [the basic example](../../examples/basic/README.md).
Starting `awr-mcp` does not create or migrate a database. The server binds to one
canonical project root at startup. Give servers for different projects distinct
names and explicit roots.

## Project or user MCP configuration

Merge [the MCP configuration template](../../examples/cursor/mcp.json.example)
into the receiving client's `mcpServers` object, replacing both absolute paths.
Preserve existing servers. Cursor loads MCP from:

| Scope | Path |
| --- | --- |
| Project | `.cursor/mcp.json` in the opened workspace |
| User | `~/.cursor/mcp.json` |

Prefer a single definition for the `awr` server. When both files exist, confirm
in **Customize** and by calling `awr_project_status` which server the Agent
Window actually attached. Do not commit a project file that contains a
machine-local AWR root, credentials or another project's identity.
`${workspaceFolder}` is the folder that contains `.cursor/mcp.json`; it is the
right substitution for a binary built in that workspace, not automatically the
AWR `--project` root. If the Cursor CLI omits a project-scoped server, use the
user file or the desktop Agent Window.

The documented stdio entry is:

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

`type` is required in Cursor's STDIO field table. AWR only needs an absolute
`command` and `--project`; `cwd` is not in that field set and is safe to omit
when `--project` is absolute. Cursor interpolates `${userHome}`,
`${workspaceFolder}`, `${workspaceFolderBasename}`, `${pathSeparator}`, `${/}`
and `${env:NAME}` in `command`, `args`, `env`, `url` and `headers`.

After editing configuration, reload the window or toggle the server in
**Customize**. A configured server is not proof of an active connection. Open
**Output → MCP Logs** if the server fails to start. In chat, confirm a connected
`awr` server, call `awr_project_status`, and check the project identity before
work. Cursor asks for MCP tool approval by default; Auto-review still classifies
non-allowlisted tools.

The minimum connected set to confirm before work is:

| Read tools | Mutation tools |
| --- | --- |
| `awr_project_status` | `awr_work_transition` |
| `awr_work_ready` | `awr_event_append` |
| `awr_work_get` | `awr_evidence_record` |
| `awr_context_compile` | |
| `awr_search` | |

Full lifecycle and continuity tools (session start/checkpoint/resume/claim,
waits, operations, source reindex, work prepare/manage, change preview/apply,
compaction) are listed in [the MCP reference](../../crates/awr-mcp/README.md).
Prefer those over shell when the Agent Window already has a live `awr` server.
The server does not require a model API key.

Desktop and the Agent Window can launch a local executable. Cloud Agents cannot
run that stdio command against a laptop filesystem; they need a reachable
[shared Streamable HTTP service](#shared-http) whose registered roots exist on
the server.

## Manual session workflow

These are commands for Cursor's terminal tool or the operator's terminal. Run one
step at a time and inspect the response. The snippets use Bash, `jq`, an already
initialized project, and a work key selected from `ready`. Replace the example
identity with the current agent/provider/model. Keep the receipt directory in the
handoff; it contains local work context.

```sh
AWR_BIN=/absolute/path/to/awr
AWR_PROJECT=/absolute/path/to/initialized/project
AWR_WORK=EXAMPLE-001
AWR_AGENT=cursor-primary
AWR_MODEL=your-current-model
AWR_NOTES=$(mktemp -d "${TMPDIR:-/tmp}/awr-cursor.XXXXXX")
awrj() { "$AWR_BIN" --project "$AWR_PROJECT" --json "$@"; }

awrj session list --active
awrj ready
```

If a session already exists for the current work, inspect it with `session show`
and use its AWR session ID. For new work without a session, start and claim it:

```sh
AWR_REV=$(awrj status | jq -er '.project_revision')
awrj session start --work "$AWR_WORK" --agent "$AWR_AGENT" \
  --provider cursor --model "$AWR_MODEL" --claim --ttl-ms 3600000 \
  --expected-revision "$AWR_REV" > "$AWR_NOTES/start.json"
AWR_SESSION=$(jq -er '.session.id' "$AWR_NOTES/start.json")
```

The returned claim is runtime ownership. It does not rewrite the source work's
status or owner. Use the appropriate AWR work transition before progress; inspect
conflicts rather than acquiring a competing claim. The AWR session ID is distinct
from the Cursor chat or Agent Window conversation ID. `--provider cursor` on
`session start` is metadata only. Client binders do not have a `cursor` enum;
`--client` accepts `codex`, `kimi` or `generic`.

For the same work handed over from Codex, Grok, Kimi or an earlier Cursor
session, use this **alternative**. Supply the predecessor's AWR session ID from
its handoff:

```sh
AWR_PREDECESSOR=the-recorded-awr-session-id
awrj session show "$AWR_PREDECESSOR"
AWR_REV=$(awrj status | jq -er '.project_revision')
awrj session resume --from-session "$AWR_PREDECESSOR" \
  --agent "$AWR_AGENT" --provider cursor --model "$AWR_MODEL" \
  --budget 5000 --expected-revision "$AWR_REV" > "$AWR_NOTES/resume.json"
jq -e '.context_ready' "$AWR_NOTES/resume.json"
AWR_SESSION=$(jq -er '.resumed.session.id' "$AWR_NOTES/resume.json")
```

Read L0 for orientation, then L1 for execution:

```sh
awrj context bootstrap --session "$AWR_SESSION" --budget 1000 \
  > "$AWR_NOTES/bootstrap.json"
jq -e '.context.complete' "$AWR_NOTES/bootstrap.json"

awrj context compile --work "$AWR_WORK" --session "$AWR_SESSION" \
  --budget 5000 > "$AWR_NOTES/context.json"
jq -e '.completeness.complete and (.work_context != null)' "$AWR_NOTES/context.json"
```

Read the actual context, required facts and gaps, not just the boolean printed by
`jq`. L0 explicitly reports `execution_context_complete: false`. Supply concrete
`--path`, `--tag` or `--goal` inputs when the rules need them. `--source-sha` can
bind evidence currency to the full commit under review.

If hard facts exceed a budget, the command fails with `BudgetExceeded`. Inspect
its `required` count and explicitly increase the budget or clarify scope. Do not
delete hard facts to make the call pass.

With a connected MCP server, the L1 read can instead use `awr_context_compile`
with these arguments, substituting the real AWR session ID:

```json
{"session":"<awr-session-id>","budget":5000}
```

Check `completeness.complete`, `work_context.context_hash` and its gap/omission
metadata. The five MCP read tools verify source freshness without refreshing the
persistent index. On `SourceStale`, inspect the source change, run the CLI's
`source reindex`, then read again. CLI context reads can refresh that index.
A transport's output limit is separate from AWR's token budget; incomplete or
visibly truncated delivery must be recovered before execution.

## Checkpoint before handoff or planned compaction

Save the hash of the last context actually used, a factual digest, the exact next
action and every unresolved loop. Replace the illustrative text below with the
current work's actual progress. Fetch the current revision after any source/work
mutations, but do not replace the last-used context hash with an invented hash.

```sh
AWR_CONTEXT_HASH=$(jq -er '.work_context.context_hash' "$AWR_NOTES/context.json")
AWR_REV=$(awrj status | jq -er '.project_revision')
awrj session checkpoint --session "$AWR_SESSION" \
  --context-hash "$AWR_CONTEXT_HASH" \
  --digest "Implemented the selected change; review results are still pending." \
  --next-action "Inspect the review result and address the remaining issue." \
  --open-loop "Independent review is unfinished." \
  --expected-revision "$AWR_REV" > "$AWR_NOTES/checkpoint.json"
awrj session show "$AWR_SESSION"
```

Checkpointing automatically records observed source/runtime changes. The supplied
digest and hash remain caller assertions; JSON reports `context_hash_verified:
false`. A successful save advances the project revision twice. Always use the
returned/current revision rather than adding one yourself. Incomplete save
attempts are retained for inspection and are never valid recovery checkpoints.

## Resume the work

For an actual handoff or a new Cursor chat that should continue the same AWR
work, explicitly name the predecessor:

```sh
AWR_REV=$(awrj status | jq -er '.project_revision')
awrj session resume --from-session "$AWR_SESSION" \
  --agent "$AWR_AGENT" --provider cursor --model "$AWR_MODEL" \
  --budget 5000 --expected-revision "$AWR_REV" > "$AWR_NOTES/resume.json"
jq -e '.context_ready' "$AWR_NOTES/resume.json"
AWR_PREDECESSOR=$AWR_SESSION
AWR_SESSION=$(jq -er '.resumed.session.id' "$AWR_NOTES/resume.json")
awrj session show "$AWR_SESSION"
awrj context compile --session "$AWR_SESSION" --budget 5000 \
  > "$AWR_NOTES/context.json"
jq -e '.completeness.complete and (.work_context != null)' "$AWR_NOTES/context.json"
```

Read the recovered context and verify the inherited checkpoint, next action,
unresolved loops and current source changes. Resume creates a new AWR session;
it does not start or switch a Cursor conversation. Default resume transfers a
still-live claim with its original expiration, without extending its TTL. After
an expired/released claim, explicitly acquire a new claim when ready; do not
infer ownership from a predecessor's history.

If the same AWR session remains active after Cursor compaction, reload context
for that session. A new AWR session is not required for every compact. Use resume
when there is a real session handoff. After a crash or lost response, inspect
`session show <predecessor>` and its successor before retrying. Even a nonzero
resume response can contain `resumed.session.id` with `context_ready: false`:
that successor already exists. Fix its context inputs and compile for it instead
of repeating the transition. No checkpoint means source/session-start recovery
with explicit missing-memory gaps.

When stopping work, checkpoint first, then close the session with the observed
revision and an appropriate outcome:

```sh
AWR_REV=$(awrj status | jq -er '.project_revision')
awrj session end --session "$AWR_SESSION" --outcome incomplete \
  --expected-revision "$AWR_REV"
```

An ended session releases its claims. It does not complete the source work;
`work complete` still requires the task's evidence and acceptance bindings.

## Bind a Cursor conversation

`client bind` maps a native conversation ID onto AWR work. It does not acquire a
claim. `--client generic` records the binder as `generic:<conversation>`; that is
separate from `--provider cursor` on `session start`. There is no `--client
cursor`. On bind, hook and show, `--client cursor` is `InvalidInput`. On
`client install`, a non-Codex `--client` (including `cursor`) is `Unsupported`.
Do not treat either error as a missing binary or a failed MCP connection.

To attach a Cursor conversation ID to a **still-active** AWR session, pass
`--session`. That does not resume or create a successor:

```sh
awrj client bind --client generic --external-session CURSOR_CONVERSATION_ID \
  --work "$AWR_WORK" --session "$AWR_SESSION"
```

To continue after a handoff or an ended predecessor, pass `--from-session`. That
creates a successor session, then binds the new conversation to it:

```sh
awrj client bind --client generic --external-session CURSOR_CONVERSATION_ID \
  --work "$AWR_WORK" --from-session "$AWR_PREDECESSOR"
```

Do not pass both flags. `awr client install` currently installs automatic
adapters for Codex only. Other clients, including Cursor, use this generic
binder and the manual checkpoint process below.

## Shared HTTP

Cursor remote MCP entries use a URL, not a local executable. AWR's shared
service uses static bearer tokens, not Cursor's OAuth client registration. Put
the token in the environment; do not commit it. HTTP tools require an explicit
`project` key. See [the shared service guide](../reference/mcp-service.md).

### Local desktop check

A loopback URL is only a desktop Streamable HTTP smoke test. The service and the
Cursor client must run on the same host:

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

### Cloud Agents

Cloud Agents cannot reach an operator laptop's loopback. They need a reachable
HTTPS front door, project roots that exist on that server, and the client's
bearer via `${env:…}`:

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

This guide does not verify Cloud Agent, team-marketplace or enterprise-allowlist
deployment, and it does not register an OAuth client with Cursor.

## Hooks and verification boundary

Cursor documents project and user hooks, including `sessionStart`, `sessionEnd`
and `preCompact`. Hook files live at `.cursor/hooks.json` or `~/.cursor/hooks.json`.
[Official hook documentation](https://cursor.com/docs/agent/hooks)

This integration supplies manual checkpoint/resume and the generic client binder.
It does not install Cursor hooks. Configuration presence, synthetic receiver
checks and local fixture results do not prove native activation, MCP tool
invocation or business acceptance. Verify an actual trigger and a checkpoint
receipt before claiming automatic AWR persistence. If events are missing, hooks
are inactive, or delivery is unverified, use the manual checkpoint process above.

Use the [executable lifecycle example](../../examples/codex/README.md) to check
AWR CLI behavior on a disposable project. That script invokes AWR, not Cursor.

On 2026-09-17 this host built `awr` / `awr-mcp` from source and initialized a copy
of `examples/basic`. The CLI completed session start, bootstrap, L1 compile,
checkpoint and session end with `--provider cursor`. `awr client install
--client cursor` returned `Unsupported` as documented.

The same host then merged the stdio template into Cursor 3.18.9
(`~/.cursor/mcp.json` and project `.cursor/mcp.json`), reloaded MCP, and called
`awr_project_status` plus `awr_work_ready` from the Agent Window. The connected
server reported project `AWR example`, `EXAMPLE-001` ready, `freshness_basis:
source_verified_readonly`, and `source_refresh_performed: false`. That confirms
stdio discovery and those two read tools in this client. It does not claim a
Cursor model turn, Cloud Agent or team-marketplace deployment, an installed hook,
or business acceptance.

# NoFray Agent Plugin

Use NoFray from a compatible AI agent to manage actionable work, extract
grounded tasks from client-held transcripts, and import external task data
through a single confirmed plan. The
plugin combines:

- the local NoFray MCP server at `http://127.0.0.1:7341/mcp`;
- a focused task-management skill that recognizes tasks, action items, actions, todos,
  follow-ups, next steps, work items, reminders, taken, actiepunten, acties,
  vervolgacties, vervolgstappen, werkpunten, and herinneringen;
- a server-versioned task-extraction contract, so the client uses the same
  prompt and response schema as NoFray without uploading the transcript to MCP;
- a generic import skill for scoped source conversion, mapping, preview,
  approval, resumable batching, and verification, with TaskNotes and Obsidian
  among its source adapters;
- the safe NoFray proposal workflow for ordinary writes and the canonical
  import workflow for migrations.

Codex surfaces use the bundled NoFray logo and brand color from the native
`.codex-plugin` presentation manifest. These visual assets do not alter skill
activation or MCP behavior.

Because the MCP server runs inside the NoFray macOS app, this is a desktop-only
plugin. NoFray and the AI client must run on the same Mac.

## Requirements

- NoFray installed on macOS.
- A workspace mounted in NoFray.
- A client that supports Agent Plugins 1.0 skills and Streamable HTTP MCP.
- Port `7341` available on the Mac.

## Prepare NoFray

1. Open NoFray and mount the workspace the agent should use.
2. Open **Settings > MCP**.
3. Enable **Enable local MCP server**.
4. Keep the port at `7341`.
5. Leave **Require authentication** off for the portable plugin setup.
6. Select **Test connection** and confirm that the tool count is shown.

With authentication disabled, NoFray still listens only on `127.0.0.1` and
continues to enforce Host validation, Origin validation, request-size limits,
and rate limits. Other applications running on the same Mac can nevertheless
reach the server while it is enabled. Turn the server off when it is not
needed, or use the authenticated setup described below.

## Install in Codex CLI

Add this repository as a plugin marketplace:

```sh
codex plugin marketplace add yvos/nofray-plugin-marketplace
```

Install the NoFray plugin from that marketplace:

```sh
codex plugin add nofray@nofray-marketplace
```

Confirm that it is installed:

```sh
codex plugin list
```

Restart or refresh Codex if the newly installed plugin is not visible in an
already open session.

## Install in Claude Code

Add this repository as a Claude Code plugin marketplace:

```sh
claude plugin marketplace add yvos/nofray-plugin-marketplace
```

Install the NoFray plugin:

```sh
claude plugin install nofray@nofray-marketplace
```

Confirm that it is installed:

```sh
claude plugin list
```

Start a new Claude Code session. If installing from inside an existing session,
run `/reload-plugins` when Claude Code asks you to activate the new plugin. Use
`/mcp` to confirm that the plugin-provided `nofray` server is connected.

## Install in ChatGPT or a managed Codex workspace

A workspace administrator can import the marketplace from GitHub:

1. Open **Workspace settings > Plugins**.
2. Select **Add > Import marketplace**.
3. Enter `https://github.com/yvos/nofray-plugin-marketplace` as **Source**.
4. Leave **Path** empty because `.agents/plugins/marketplace.json` is at the
   repository root.
5. Use `main` as the branch, or leave the branch empty to follow the default.
6. Import the marketplace and review the result.
7. Open the NoFray plugin and choose the appropriate installation policy.
8. Install or enable it for the intended users or roles.

The MCP declaration makes the imported plugin desktop-only. It cannot reach a
NoFray server running on a different Mac or from ChatGPT on the web.

## Install in another compatible client

Load the `AgentPlugin` directory as an Agent Plugins 1.0 package using the
client's documented local-directory or repository installation flow. The
client must support both Agent Skills and the `streamable-http` MCP transport
to provide the full NoFray experience.

The current compatibility list is maintained at
<https://agent-plugins.org/compatible-clients>.

## Verify the complete connection

After installation, start a new agent session and ask:

> Show my open action items in NoFray.

A working installation should let the agent discover the NoFray tools, read
the workspace configuration, list canonical field values when needed, and
search tasks. For a write test, explicitly ask it to create a disposable task;
the agent should use the proposal, resolution, and apply tools in sequence.

For a transcript test, supply a short transcript and ask the agent to extract
and create its grounded action items. The agent should first call
`nofray_get_task_extraction_contract`, keep the transcript in the client, then
use the ordinary task discovery and proposal workflow for each intended write.

For an import test, supply a small supported export, such as a TaskNotes folder
or zip, and ask the agent to preview it. The agent should limit source discovery
to the requested records and approved dependencies, show a complete conversion
ledger, and wait for approval before applying the server-authored confirmation
value. Interrupted imports can resume by querying
`nofray_get_import_status`.

## Authentication

The portable plugin deliberately contains no bearer token. Do not add a token
to `mcp.json`, commit one to the repository, or paste one into `SKILL.md`.

To require authentication:

1. Enable **Require authentication** in **NoFray > Settings > MCP**.
2. Select **Connect AI Client…**.
3. Choose the client and use the generated client-specific setup command.
4. Disable or override the plugin-provided credential-free MCP entry if the
   client would otherwise try both connections.

The bundled skills remain useful, but the portable `mcp.json` entry
cannot authenticate by itself. Token storage and header configuration must be
handled by the client.

## Custom port

The distributed plugin targets port `7341`. Keeping this port is the simplest
and recommended setup. If another process occupies it, either:

- choose another port in NoFray and configure the MCP endpoint directly with
  **Connect AI Client…**; or
- maintain a private copy of the plugin and change the URL in `mcp.json` to the
  same port.

The port in NoFray and the port in the client configuration must match.

## Update or remove in Codex CLI

Refresh the Git marketplace and then update or reinstall the plugin as offered
by the client:

```sh
codex plugin marketplace upgrade nofray-marketplace
codex plugin list
```

Remove the plugin with:

```sh
codex plugin remove nofray@nofray-marketplace
```

Remove the marketplace separately only when it is no longer used by any
installed plugin.

## Update or remove in Claude Code

Refresh the marketplace and update the installed plugin:

```sh
claude plugin marketplace update nofray-marketplace
claude plugin update nofray@nofray-marketplace
```

Restart Claude Code to use the updated plugin. Remove the plugin with:

```sh
claude plugin uninstall nofray@nofray-marketplace
```

The interactive equivalents are available through `/plugin`. Removing the
marketplace itself also uninstalls plugins installed from it.

## Troubleshooting

### The plugin is installed but no NoFray tools are available

- Confirm that NoFray is running and a workspace is mounted.
- Confirm that the MCP server is enabled.
- Confirm that NoFray and the agent client run on the same Mac.
- Confirm that both sides use port `7341`.
- Run **Test connection** in NoFray.
- Start a fresh agent session after installing or updating the plugin.
- Check whether the client supports Streamable HTTP MCP in Agent Plugins.
- In Claude Code, run `/mcp` and `/reload-plugins` to inspect and reload the
  plugin-provided connection.

### The connection is unauthorized

Authentication is enabled in NoFray, but the portable plugin has no token.
Either disable authentication for the loopback-only setup or configure the
client through **Connect AI Client…**.

### The connection is refused or lost

- Toggle the local MCP server off and on in NoFray.
- Check that another process is not using the configured port.
- Review the MCP diagnostics in NoFray's logs.
- Retry from a new client session after the server reports that it is running.

### Read calls work but writes do not

NoFray writes require all three server-owned steps:

1. `nofray_create_change_proposal`
2. `nofray_resolve_change_proposal`
3. `nofray_apply_change_proposal`

The agent must pass server-authored IDs, generations, hashes, and resolutions
unchanged. It must also use canonical status and priority IDs returned by
`nofray_list_field_values`.

## Package layout

```text
AgentPlugin/
├── .codex-plugin/
│   └── plugin.json
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── assets/
│   ├── composer-icon.svg
│   └── logo.svg
├── plugin.json
├── mcp.json
├── README.md
└── skills/
    ├── nofray-task-management/
    │   └── SKILL.md
    └── nofray-import/
        └── SKILL.md
```

The repository-level `.agents/plugins/marketplace.json` makes this package
installable as a Codex marketplace entry. The repository-level
`.claude-plugin/marketplace.json`, plugin manifest, and `.mcp.json` provide the
native Claude Code installation route. Installation, permissions, updates, and
enablement remain controlled by the client.

## Specifications

- Agent Plugins 1.0: <https://agent-plugins.org/specification>
- Agent Skills: <https://agentskills.io/specification>
- Model Context Protocol: <https://modelcontextprotocol.io/specification>

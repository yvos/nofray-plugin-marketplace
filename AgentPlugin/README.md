# NoFray Agent Plugin

Use NoFray from a compatible AI agent to manage actionable work, extract
grounded tasks from client-held transcripts, and import external task data
through a single confirmed plan. The
plugin combines:

- the local NoFray MCP server at `http://127.0.0.1:7341/mcp`;
- a focused task-management skill that recognizes tasks, action items, actions, todos,
  follow-ups, next steps, work items, reminders, taken, actiepunten, acties,
  vervolgacties, vervolgstappen, werkpunten, and herinneringen;
- a read-only meeting-analysis prompt tool, so the client uses the same
  system prompt as NoFray without uploading the transcript to MCP;
- a generic import skill for scoped source conversion, mapping, preview,
  approval, resumable batching, and verification, with TaskNotes and Obsidian
  among its source adapters;
- the v2 NoFray proposal workflow for ordinary writes and the canonical v2
  import workflow.

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

## Contract version

This release uses the v2 MCP and import contract as its only regular mutation
and import surface. A client must discover the mounted workspace before every
mutation session and require the live `recordMutationContractV2` and
`recordMutationV2` capabilities, plus `recordReferenceCandidatesV2` when it
resolves relations. The client uses the schemas returned by
`nofray_get_workspace_configuration`. The plugin does not fall back to a
legacy request envelope when a capability is absent.

Mutation field operations are exact and sparse:

```json
{
  "operation": "set",
  "value": "2026-09-30T17:00:00+02:00"
}
```

Use `{ "operation": "clear" }` without a `value` to clear a field; omit a
field to leave it unchanged. The root request's `context` object must carry the
same discovered `workspaceID`, `sessionID`, `generation`, `schemaDigest`, and
`capabilityDigest`. The recurrence value retains its canonical storage
`version: 1` member inside the recurrence object; that is separate from the
v2 MCP envelope.

When the live proposal tool schema advertises Inbox actions, explicit Inbox
membership uses the same proposal/resolve/apply tools:
`request: {"action":"addToInbox","recordType":"task","recordID":"<task ID>"}`
or `action: "clearInbox"`, with the discovered context and no generic
`fields`. `inboxAddedAt` remains lifecycle-only. Automatic MCP create
admission is separate: it follows the live workspace setting and applies only
to tasks without projects, tags, contexts or assignees.

Apply returns canonical readback verified against the writer's record ID and
revision. An error with `writeState: "committed"` and
`verificationStatus: "unverified"` includes the known operation/record ID,
revision and a bounded reason. Follow `recoveryAction: "repeatApply"` by
resubmitting unchanged apply arguments: this verifies the existing receipt
without writing again, even after generation advances. Do not create another
record. If verification remains unavailable, inspect the returned record ID.
Receipts are caller/session/token-bound and kept in a bounded in-memory cache
for up to ten minutes. An already verified receipt retains evidence of the
original operation; use get-record tools to inspect subsequent changes.
`writeState: "unknown"` requires inspection before any retry that could
create a record. `notAttempted` describes the current invocation only.
New writes still reject stale generation/context with `refreshProposal`;
discover fresh context and obtain a new server-authored preview/resolution.

The live capability contract may be narrower than the raw mdbase schema. Typed
collection fields (`aliases`, `tags`, `contexts`, `assignees`, `projects` (legacy: `projectLinks`),
`reminders`, `methods`, and `affiliations`) do not accept `set` with `null`:
use `set` with an empty array for an empty collection or `clear` to remove the
field. Optional temporal and recurrence values accept `null` only when the
discovered field contract advertises it. The client must use the advertised
`allowsNull` value and verify the canonical readback.
Projects can expose `tags`, `scheduled`, and `due`, while contacts can expose
`tags`; their availability and exact schema are returned by discovery. A
missing named field reported in `unavailableFields` is a source-health warning,
not an authored empty value, and must remain visible until an explicit repair.

Import previews use a version 2 translation manifest. Every observed source
field must occur once in `fieldDispositions` as `direct`, `transformed`,
`metadataOnly`, or `omitted`. Relationship decisions go in
`relationMappings` with an owner reference, source token, target record type,
and an explicit `existingRecordID`, `uploadedSourceReference`, `ambiguous`,
`unresolved`, or `wrongType` resolution. Fetch all preview sections:
`records`, `diagnostics`, `fieldDispositions`, and `relationResolutions`.
Apply only the newest server preview after user approval; a changed manifest
creates a new preview and plan hash.

`translationManifest.sourceInventory` is required in every import preview. It
is a client-declared pre-conversion array of `TaskNotesImportObservedField`
objects, each carrying `sourceField`, `recordType`, `valueShape`,
`observedRecordCount`, and `observedValueCount`. The field disposition ledger
must cover that inventory exactly once. The uploaded canonical records remain
separate target inputs containing only `{sourceReference, recordType, markdown}`;
their fields are validated against the live target schema. For example, a
source `foo` transformed to canonical `due`, and an approved omission of
source `approved`, remain visible in the inventory and ledger even though the
canonical Markdown needs only `due` (and any explicitly preserved metadata):

```json
{
  "version": 2,
  "sourceInventory": [
    {"sourceField": "title", "recordType": "task", "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1},
    {"sourceField": "foo", "recordType": "task", "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1},
    {"sourceField": "approved", "recordType": "task", "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1}
  ],
  "fieldDispositions": [
    {"ledgerID": "task:title", "sourceField": "title", "sourceRecordTypes": ["task"], "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1, "disposition": "direct", "targets": [{"recordType": "task", "field": "title"}], "decisionStatus": "exact"},
    {"ledgerID": "task:foo", "sourceField": "foo", "sourceRecordTypes": ["task"], "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1, "disposition": "transformed", "targets": [{"recordType": "task", "field": "due"}], "transformation": {"summary": "Map foo to canonical due", "sourceFields": ["foo"]}, "decisionStatus": "approved"},
    {"ledgerID": "task:approved", "sourceField": "approved", "sourceRecordTypes": ["task"], "valueShape": "scalar", "observedRecordCount": 1, "observedValueCount": 1, "disposition": "omitted", "targets": [], "decisionStatus": "approved"}
  ],
  "relationMappings": []
}
```

When discovery also advertises `recordMutationChangeSetV2`, the same proposal
tools accept `request.action: "changeSet"` for exactly one Task operation
(create or update), plus optional Project and Contact creates. The total is at
most 16 operations, so at most 15 dependency creates may accompany the Task.
Each create has a `localReference`; the Task update uses a target `recordID`.
Preview rejects multiple Task operations, Project or Contact updates,
unsupported record types, invalid relation types, cycles, and graphs outside
this one-Task-plus-dependencies shape before any write. Resolve duplicate
choices in a `choices` dictionary keyed by operation ID, using `createNew` or
`useExisting` with the server candidate ID. Apply the returned resolution
unchanged and read back every affected record.

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
the workspace configuration and live `recordMutationContractV2` capability,
list canonical field values when needed, and search tasks. For a write test,
explicitly ask it to create a disposable task; the agent should pass the v2
context-bound request through the proposal, resolution, and apply tools in
sequence, then verify the canonical readback.

For a transcript test, first call `nofray_get_meeting_analysis_prompt`.
During the AI foundation delivery it reports `promptUnavailable`: the new
analysis prompts are still being supplied. Explain that status and stop the
extraction without inventing replacement instructions or asking the user to
reconnect. Ordinary task creation, discovery, proposals and imports continue
to work. When a prompt is available later, the transcript stays in the client
and each intended write uses the ordinary authorized proposal workflow.

For an import test, paste a task list or supply a small readable export, such as
a Markdown collection, table, TaskNotes folder, or zip, and ask the agent to
preview it. No source schema or Obsidian metadata is required. The agent should limit source discovery
to the requested records and approved dependencies, show a complete conversion
ledger with a v2 translation manifest and explicit relation mappings, and wait
for approval before applying the server-authored confirmation value.
Assignees and project links use the manifest's explicit typed resolutions;
`valueMappings` admits only status and priority. Equal project titles are valid
when their identities differ.
Interrupted imports can resume by querying
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

NoFray writes use the v2 request and require all three server-owned steps:

1. `nofray_create_change_proposal`
2. `nofray_resolve_change_proposal`
3. `nofray_apply_change_proposal`

The agent must pass server-authored IDs, generations, hashes, and resolutions
unchanged, together with the discovered workspace/session and schema and
capability digests. It must also use canonical status and priority IDs returned
by `nofray_list_field_values`, and verify each applied record with its canonical
get tool.

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
├── CHANGELOG.md
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

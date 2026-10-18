---
name: nofray-task-management
description: Use NoFray when the user wants to find, inspect, create, update, schedule, assign, complete, or organize tasks and equivalent actionable work, including action items, todos, follow-ups, next steps, reminders, taken, actiepunten, acties, vervolgstappen, werkpunten, or herinneringen in their local NoFray workspace.
---

# NoFray task management

Use the NoFray MCP server as the source of truth for the mounted workspace. The
NoFray macOS app must be running with its local MCP server enabled. The bundled
connection targets `http://127.0.0.1:7341/mcp`.

The regular mutation tools use the v2 contract. Start each mutation session by
calling `nofray_get_workspace_configuration` for the needed record types and
require the live `recordMutationContractV2` and `recordMutationV2`
capabilities. Use `recordReferenceCandidatesV2` when relation candidates are
needed. Each must be advertised by the current server.
There is no legacy mutation fallback. If a required capability is absent,
report that the requested mutation is unavailable instead of guessing a field
shape.

Check the action-specific `capabilities` as well as the feature capabilities
before proposing a write. They describe the available MCP routes and current
workspace permissions. Schema field presence alone does not grant permission.
When `canComplete` is false, report completion as unavailable; do not bypass
the missing lifecycle route by setting a completed status through generic
fields. A compound change set also requires the corresponding project/contact
creation and relation permissions for each requested operation.

## Extract tasks from a transcript

When the user supplies a transcript and asks to turn actionable commitments into
NoFray tasks:

1. Keep the transcript and its interpretation with the client. Identify explicit
   commitments, decisions, existing-task changes and uncertainty from the
   original passages, preserving their source references. Do not upload
   transcript text to MCP or ask NoFray to generate meeting minutes.
2. For each candidate, form a source-grounded intent with a proposed title,
   explicit owner or date when present, a source reference, and create/update/
   clear/unchanged intent. Keep discussion facts and decisions separate from
   actionable work; do not infer an assignment from general discussion.
3. Run bounded NoFray lookups for plausible existing tasks, contacts and
   projects before using IDs. Use only returned IDs. An ambiguous match is not a
   resolution.
4. Keep observations and unresolved identity/date choices visible. Apply changes
   only within the user's authorization.
5. Use discovery and the proposal workflow below for every intended write. NoFray
   owns canonical task discovery, review, mutation and read-back; the client
   owns transcript interpretation and any presentation of the source context.

## Read meetings and their tasks

When the user asks what happened in a meeting, which tasks belong to it, or
asks to turn those tasks into another result, use the read-only meeting routes
when discovery advertises `meetingRead`:

1. Call `nofray_search_meetings` with the narrowest useful title, date range, or
   participant filter. Keep the returned meeting ID and match evidence.
2. Call `nofray_get_meeting` with that ID. Its default scope is `meetingOnly`;
   request `scope: "series"` only when the user asks for a recurring series.
   State `status` (`openAndCompleted`, `openOnly`, or `completedOnly`) and
   `archive` (`includeAll`, `activeOnly`, or `archivedOnly`) when the question
   depends on completion or archive state. Do not assume an active-only list.
3. Use the returned current task projection and `linkedMeetings`/provenance to
   build the requested briefing or other derived answer. Read each task with
   `nofray_get_task` when more task detail is needed; never replace a missing
   task with a title guess.
4. Call `nofray_get_meeting_document` only for a required body. Pass exactly one
   role (`current`, `preparation`, `notes`, `minutesDraft`, or `draft`) or one
   immutable `documentID` for a historical `minutesVersion`, `sourceAnalysis`,
   or `sourceSnapshot`. Those roles require a concrete document ID. Follow its
   revision-bound cursor for a large body and keep the response context.

Preparation is background, own notes and an explicitly selected transcript are
sources, minutes are output, and immutable history is evidence. Transcript
responses are metadata-only. Preserve `missingTaskIDs`, omitted coverage,
stale-context errors, and document revisions in the answer instead of filling
gaps from other meetings or arbitrary files. Meeting reads do not authorize
meeting-document writes; task changes still use the proposal, resolution,
apply, and canonical readback workflow below.

## Discover before acting

1. Call `nofray_get_workspace_configuration` before every mutation session for
   workspace context, the live schemas, `recordMutationContractV2`, and the
   context values bound to the request.
2. Before setting status or priority, call `nofray_list_field_values` and pass
   the returned canonical IDs unchanged.
3. Before creating or updating actionable work, call `nofray_search_tasks` to
   find possible duplicates. Use `nofray_get_task` when the exact record must be
   inspected.
4. Use the project and contact search/get tools when a task refers to either.
   Keep relation candidates scoped to their target record type. Stable IDs may
   resolve mechanically; title, alias, path, basename, and fuzzy matches require
   an explicit choice.

Do not write when the user only asked to search, inspect, summarize, or explain.

## Apply requested changes safely

For a user-requested task, project, or contact mutation, follow the complete
server-owned v2 workflow:

1. Build the v2 request with the exact live field operations. An omitted field
   is unchanged; `{ "operation": "set", "value": ... }` replaces one field;
   `{ "operation": "clear" }` carries no value. Copy `workspaceID`,
   `sessionID`, `generation`, `schemaDigest`, and `capabilityDigest` from the
   same discovery snapshot.
2. Call `nofray_create_change_proposal` with the v2 requested operation.
3. Call `nofray_resolve_change_proposal` with the returned `previewID`, even
   when the preview recommends creating a new record. If the server reports
   `requiresChoice`, obtain or infer only the choice justified by the request
   and returned candidates.
4. Call `nofray_apply_change_proposal` only for the requested mutation. Pass the
   resolution unchanged and set the required confirmation flag.
5. Inspect the canonical readback returned by apply for every applied record.
   Use `nofray_get_task`, `nofray_get_project`, or `nofray_get_contact` for
   subsequent current-state reads. Report canonical fields,
   unavailable-field health, lifecycle state, record revision, and the returned
   schema/capability digests. A transport success without matching readback is
   not a completed write.

When `canChangeInboxMembership` is available and the live proposal tool schema
advertises Inbox actions, explicitly add or remove a task using the same workflow with
`request: {"action":"addToInbox","recordType":"task","recordID":"<discovered ID>"}`
or `action: "clearInbox"`. Pass the usual discovered `context`; omit
`fields`. NoFray owns the timestamp. Do not set the lifecycle-only
`inboxAddedAt` field through generic mutations.

Automatic Inbox placement on creation follows the live workspace setting. When
enabled, every new MCP task enters the Inbox, regardless of projects, tags,
contexts or assignees, including change-set creates. Updates and duplicate reuse
do not automatically enter the Inbox. For an explicit Inbox request, check the
verified canonical readback and use the lifecycle action if membership is absent
(for example, when automatic placement is disabled or the app version predates
this behavior).

For recurrence, send the complete replacement object with its canonical
`version: 1`, `rule`, `timing`, `anchor`, `timeZone`, and `history`. That `1`
identifies the recurrence storage envelope; it is not an alternate MCP version.
Treat reminders as an atomic replacement and validate absolute instants,
relative offsets, stable IDs, duplicate IDs, and the effective scheduled/due
anchor before proposing them.

The live capability contract may narrow a nullable raw schema for typed
collections. `aliases`, `tags`, `contexts`, `assignees`, `projects` (legacy: `projectLinks`),
`reminders`, `methods`, and `affiliations` reject `set` with `null`; use `set`
with `[]` for an empty collection or `clear` to remove it. Optional temporal
and recurrence fields accept `null` only when their discovered `allowsNull`
value is true. Never rely on a raw schema's nullable branch when the live
semantic capability disallows it.

Projects may expose `tags`, `scheduled`, and `due`; contacts may expose
`tags`. Send project dates as the live contract's temporal string and verify
them in canonical readback just like task dates. If readback reports one of
these named fields in `unavailableFields`, preserve the warning and do not
interpret the missing field as an authored empty value.

When discovery advertises `recordMutationChangeSetV2`, the same proposal tools
accept a bounded `request` with `action: "changeSet"` for exactly one Task
operation (create or update), plus optional Project and Contact creates. The
total is at most 16 operations, so at most 15 dependency creates may accompany
the Task. Use an `operationID` for every operation, `localReference` for
creates, and a target type scoped `recordID` for the Task update. Multiple Task
operations, named Project/Contact updates, unsupported record types, invalid
relation types, cycles, and graphs outside this one-Task-plus-dependencies
shape are rejected at preview. Resolve duplicate choices in the server-shaped
`choices` dictionary keyed by operation ID (`createNew` or `useExisting` with
its candidate ID), then pass the returned resolution unchanged and read back
every affected record.

Never invent or reconstruct preview IDs, proposal IDs, resolution hashes,
capability digests, workspace IDs, session IDs, generations, or other
server-authored integrity values.

## Recover visibly

- If the MCP server is unavailable, ask the user to open NoFray, mount the
  intended workspace, enable the local MCP server, and keep port `7341` when
  using the bundled plugin configuration.
- If authentication is enabled, direct the user to **Connect AI Client…** in
  NoFray settings. The bundled plugin contains no bearer token.
- Treat typed MCP errors as actionable state. Refresh workspace configuration
  after session, generation, capability, or workspace-change errors; repeat
  discovery instead of fabricating stale values.
- Report ambiguous duplicate candidates instead of silently overwriting one.
- If apply reports `writeState: "committed"` with
  `verificationStatus: "unverified"` and `recoveryAction: "repeatApply"`,
  retain the returned record ID/revision and repeat the exact apply arguments
  to verify the existing write. Do not make a new create proposal or operation
  ID. If verification still fails, report the known write and unresolved
  verification separately and inspect the record by ID.
- `writeState: "unknown"` does not mean that nothing was written. Follow
  `inspectRecord` before another create. A stale/context error with
  `refreshProposal` means this invocation did not attempt a write; it does not
  prove an earlier invocation failed to write. Receipts use a bounded,
  session-bound in-memory cache and expire after ten minutes. Missing/expired
  receipts and restarts require
  canonical inspection before a new create.

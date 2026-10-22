---
name: nofray-task-management
description: Use NoFray to find, inspect, create, update, schedule, assign, complete, or organize tasks and action items, or to create and edit projects, contacts, and their notes in the local workspace. Includes taken, actiepunten, todos, reminders, projecten, contacten, and notities.
---

# NoFray task, project, and contact management

Use the NoFray MCP server as the source of truth for the mounted workspace. The
NoFray macOS app must be running with its local MCP server enabled. The bundled
connection targets `http://127.0.0.1:7341/mcp`. NoFray supports independent
Task, Project, and Contact records; a Project or Contact mutation does not
require creating or changing a Task.

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
Projects and Contacts use their own `nofray_search_projects` /
`nofray_get_project` and `nofray_search_contacts` / `nofray_get_contact` routes.
Search the requested record type when its identity is unclear, then use the
returned stable ID. Before mutation, require the live
`recordMutationContractV2` and `recordMutationV2` features and the relevant
action capability. `canRenameProjects` is the live permission for editing any
Project fields; `canEditContacts` controls Contact updates. Respect the other
create and lifecycle capabilities individually.
Complete or reopen a task through `updateRecord` by setting its configured
status. Check `canComplete` for transitions into or out of completion as well as
`canEdit` and the live status field contract. NoFray owns completion dates,
recurrence advancement and linked minutes checkbox synchronization. Send only
the requested fields; do not calculate dates, recurrence history or companion
checkbox edits. A recurring completion may return the active next occurrence;
report the canonical apply result. Preserve typed lifecycle errors rather than
working around restrictions. A compound change set also requires
the corresponding project/contact creation and relation permissions for each
requested operation.

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
   `nofray_get_task` only for missing fields, full task descriptions, or a required
   later current-state check. Status comparisons can use the bundle's canonical
   task fields directly; exporting full descriptions may require individual reads.
   Never replace a missing task with a title guess.
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
2. Reuse canonical status/priority IDs from discovery's `taskFieldInformation`
   when sufficient; otherwise call `nofray_list_field_values`. Both describe the
   same effective catalog. `isEnabled` guides new suggestions; hidden values
   remain readable/filterable and hiding does not grant or revoke write rights.
   Pass canonical IDs unchanged and reuse them until workspace/configuration changes.
3. Before creating actionable work or identifying an update target, use bounded
   `nofray_search_tasks` to find possible duplicates. An update to an exact task ID
   already returned by NoFray does not need another broad duplicate search.
   Use `nofray_get_task` only when the required record fields are missing.
4. Use the project and contact search/get tools when a task refers to either.
   Keep relation candidates scoped to their target record type. Stable IDs may
   resolve mechanically; title, alias, path, basename, and fuzzy matches require
   an explicit choice.

Project and Contact search/get are also standalone user-facing operations; a
Task relation is not required. Do not run a Task search before a direct
Project/Contact create or edit unless the user asks about related Tasks.

Do not write when the user only asked to search, inspect, summarize, or explain.

Reuse sufficient read results within the current operation. Parallelize independent
reads when useful, but serialize writes against their admitted context. A successful
write can advance the generation: discover fresh context before the next proposal,
and refresh after typed stale-context errors. Never increment a generation yourself
or reuse an old receipt as current workspace context.

## Query an exact task subset

When the live `nofray_search_tasks` schema advertises structured selection, use
its native `list`, completion `status`, `archive`, and exact `filters` rather
than downloading every task and recreating NoFray rules. List and filters
intersect. Values within one field are OR; different fields are AND. Use record
IDs for projects/assignees and the returned canonical values for other fields.
Without structured selection, the existing text search remains available.
Follow the returned cursor unchanged and preserve `coverage`, `totalCount` and
effective selection in claims about completeness. A stale cursor requires a
fresh search; do not manufacture offsets or apply UI filters implicitly.

## Archive, restore and delete

When the live schema and action-specific capabilities permit it, use the same
proposal/resolve/confirmed-apply flow with `request` containing exactly `action`,
`recordType` and `recordID`. Actions are `archiveRecord`, `unarchiveRecord` and
`deleteRecord`; types are `task`, `project` and `contact`. Do not set lifecycle
fields through generic `fields`, and do not use these actions inside a changeSet.
Task archive also clears Inbox membership; restoring does not recreate it.
NoFray owns reference safety and related meeting cleanup. A blocked deletion
must remain blocked; do not remove relations as a workaround.
A verified deletion returns outcome `deleted` and `canonicalRecord: null`.
An unverified committed deletion uses the same exact `repeatApply` recovery;
never submit a replacement delete or infer success from a missing projection.

## Apply requested changes safely

For a user-requested task, project, or contact mutation, including a direct
Project/Contact request with no related Task, follow the complete server-owned
v2 workflow:

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
   A verified matching readback completes this write's verification; do not add
   an automatic get-record call solely to verify it again.
   Use `nofray_get_task`, `nofray_get_project`, or `nofray_get_contact` for
   subsequent current-state reads. Report canonical fields,
   unavailable-field health, lifecycle state, record revision, and the returned
   schema/capability digests. A transport success without matching readback is
   not a completed write.

Project fields use `title`, `aliases`, `tags`, `scheduled`, `due`, and `notes`;
Contact fields use `displayName`, `aliases`, `tags`, `methods`, `affiliations`,
and `notes`, subject to the live record contract. A Project or Contact's
`notes` field is its Markdown body; Task narrative uses `details`. These body
fields are sent as strings, and literal Markdown is accepted directly in the
string value, including headings, lists, and links. No separate Markdown wrapper
is needed. For a request to add or revise part of existing notes, read the
current full body first when it is not already available, then submit the
intended full replacement in one `notes` `set`. Preserve unrelated text and
formatting. Use `clear` only when the user asks to remove the notes; omitting
`notes` leaves it unchanged. Never send `set(null)` for these body fields.

Examples include “Create project Atlas with these notes”, “Add this paragraph
to project Atlas”, and “Update only Noor's contact notes”.

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

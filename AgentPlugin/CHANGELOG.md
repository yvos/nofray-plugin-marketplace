# Changelog

## 0.7.9 - 2026-10-20

- Query native task lists and exact field subsets with snapshot-bound pagination
  and coverage; reuse shared field meanings and enabled enum values.
- Archive, restore and delete tasks, projects and contacts through the existing
  confirmed V2 proposal workflow and NoFray lifecycle services.
- Distinguish verified deletion from unavailable canonical readback; exact retries
  verify the original operation without repeating its write.
- Retire the canonical task, project, and contact import route. The plugin now
  exposes ordinary task management and meeting-read workflows; the import skill
  and import tools are no longer part of the package.
- Add the read-only `nofray_search_meetings`, `nofray_get_meeting`, and
  `nofray_get_meeting_document` routes with snapshot context, explicit meeting
  scope/task filters, bounded document bodies, and metadata-only transcript
  references.
- Remove the obsolete `nofray_get_meeting_metadata` route. Meeting and task
  reads use the live NoFray app contract; external MDBase files are not runtime
  authority.

## 0.7.8 - 2026-10-19

- Complete and reopen through the existing status update route. NoFray owns
  completion dates, recurring occurrences and linked minutes checkboxes; clients
  report the canonical result and respect live completion authority.
- Reuse sufficient meeting/task reads, discovered field values and verified apply
  readback. Refresh mutation context after writes without repeating broad target searches.

## 0.7.6 - 2026-09-29

- Follow the current two-stage compact meeting generation descriptors; raw facts
  are not task writes or import records. Saved-minutes validation returns findings.

## 0.7.4 - 2026-09-10

- Publish versioned generation and saved-minutes validation prompts through
  `nofray_get_meeting_analysis_prompt`, including their hashes and wire
  contracts. The transcript remains client-side and the prompt tool remains
  read-only.

## 0.7.3 - 2026-09-10

- Require action-specific live capabilities before proposing a mutation;
  schema fields alone do not grant write or lifecycle permission.
- Respect unavailable completion actions and check project/contact and relation
  permissions separately for compound changes.

## 0.7.2 - 2026-09-09

- Document explicit Inbox add/clear actions through the proposal workflow when
  the connected NoFray runtime advertises them in its live tool schema.
- Distinguish committed writes with pending verification from unknown outcomes,
  and explain how to repeat apply safely using the original arguments.
- Use apply's canonical receipt as write evidence and get-record tools for
  subsequent current-state reads. Clarify bounded receipt lifetime and stale
  context recovery.

## 0.7.1 - 2026-09-09

- Handle deliberately unavailable meeting-analysis prompts without fallback
  instructions or reconnect/update loops. Ordinary task management and imports
  retain their existing contract.

## 0.7.0 - 2026-09-09

- Clarify that import sources may be pasted tasks, free text, or any readable
  export. TaskNotes and Obsidian conventions are optional interpretation hints.
- Use only V2 manifest relation mappings for assignees and project links;
  `valueMappings` now covers status and priority only.
- Replace the task-extraction contract discovery tool with the read-only
  `nofray_get_meeting_analysis_prompt` tool. It publishes the same current
  prompt, calculated hash/version, and small client-side examples used by the
  native meeting preview.

## 0.6.0 - 2026-09-05

This prerelease aligns the plugin with NoFray's schema-driven v2 MCP and import
contract.

### Changed

- Ordinary Task, Project, and Contact mutations discover live schemas and the
  `recordMutationContractV2` capability before constructing a request.
- Mutation fields use the exact sparse `set`/`clear` operation shape, with
  omitted fields meaning unchanged and the request bound to its workspace,
  session, generation, schema digest, and capability digest.
- Recurrence and reminders are treated as complete atomic values. Recurrence's
  nested `version: 1` remains canonical storage metadata and is not an MCP
  compatibility version.
- Successful writes require canonical post-write readback, including field
  health, lifecycle state, revision, and schema/capability digests.
- Imports require a client-declared `translationManifest.sourceInventory` of
  `TaskNotesImportObservedField` entries, a complete field-disposition ledger,
  and explicit typed relation mappings in the sealed preview plan. Canonical
  uploads remain `{sourceReference, recordType, markdown}` target inputs, so a
  transformed or approved-omitted source field need not be present in the
  canonical Markdown. Ambiguous or unresolved relationships remain visible and
  block apply.
- Relation candidates are record-type-scoped; only exact IDs and explicit
  sealed source-reference bindings can resolve without a user choice.
- When advertised, `recordMutationChangeSetV2` enables the existing proposal
  tools to plan and apply exactly one Task operation plus optional Project and
  Contact creates, with local references, keyed duplicate choices, and complete
  readback. The total is at most 16 operations, so at most 15 dependency
  creates accompany the Task. Multiple Task operations and graphs outside this
  shape are rejected during preview.
- The old mutation/import envelope is not a supported fallback in this
  prerelease. Clients must use the live v2 contract.

### Validation

The implementation and transport, live-client, and end-to-end acceptance gates
are tracked by the NoFray implementation plan. This changelog does not claim a
test result until those gates have produced executable evidence.

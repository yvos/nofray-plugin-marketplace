# Changelog

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

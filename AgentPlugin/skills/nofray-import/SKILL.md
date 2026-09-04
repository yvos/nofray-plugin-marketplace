---
name: nofray-import
description: Use NoFray when the user wants to import or migrate tasks, projects, or contacts from another task manager, Markdown collection, Obsidian vault, TaskNotes folder, or supported export into their local NoFray workspace.
---

# Import external data into NoFray

Use NoFray's canonical import tools. Treat each source format as an adapter into
one server-owned, previewed batch workflow. Never replace missing import results
with ordinary per-record change proposals: those writes were not part of the
confirmed import plan.

## Establish source scope

The client reads and converts the source; NoFray never opens a supplied path or
connects to the source system.

1. Use only the export, folder, archive, or records the user supplied. Identify
   the source system and format, then select only the requested tasks, projects,
   and contacts.
2. Confirm once whether archived records are included and whether referenced
   projects or contacts should be imported as dependencies. Never broaden the
   scan to unrelated records or notes.
3. Never modify, rename, move, normalize, stage, or create a filtered copy of
   source files. If the client cannot read a local folder, ask for a supported
   export, zip, or client filesystem access; NoFray source-path permission cannot
   solve it.
4. Follow the source adapter's documented semantics. If a source field or value
   is unclear, expose the ambiguity instead of guessing.

## Build a complete conversion ledger

Call `nofray_get_workspace_configuration` and `nofray_list_field_values` before
conversion. Inventory every observed source field and account for it exactly
once as:

- **Directly mapped**: exact one-to-one mapping or rename.
- **Proposed transformation**: merge, split, value-type change, title fallback,
  or other semantic conversion. State its rules and obtain explicit approval.
- **Preserved only as metadata**: retained in frontmatter, but not a functional
  writable NoFray field and not guaranteed to appear in NoFray's UI.
- **Proposed omission**: dropped only after explicit approval.

Never describe retained frontmatter as a functional field merely because its
bytes survive. Do not hide metadata-only or omitted fields in an "unmapped"
count.

Resolve every project or contact reference against the selected source. Upload
the resolved dependency record with the records that reference it. If a source
relationship has no NoFray equivalent, preserve it as approved source metadata
or omit it after approval. Stop on absent or ambiguous references.

Convert recurrence only after semantic approval, using the complete canonical
envelope, for example:

```yaml
recurrence:
  version: 1
  rule: "DTSTART:20260728;FREQ=WEEKLY;INTERVAL=1;BYDAY=TU"
  timing: calendar
  anchor: scheduled
  timeZone: Europe/Amsterdam
  history: []
```

The client must preserve the exact recurrence rule and explicitly choose
`timing` (`calendar` or `afterCompletion`), `anchor` (`scheduled` or `due`), and
a valid IANA time zone. Do not guess these semantics.

## Convert and upload

Convert each selected source object into canonical NoFray Markdown. Each upload
record contains exactly `sourceReference`, `recordType` (`task`, `project`, or
`contact`), and `markdown`. Use a stable source-relative reference or opaque
export identifier, never an absolute path. Preserve the body and unrelated
frontmatter. Do not synthesize a missing stable ID; NoFray derives one
deterministically.

Use only the writable fields advertised by the workspace configuration. Tasks
use canonical status and priority values. Projects and contacts have no status;
set `archived` only from explicit source evidence (otherwise omit it or use the
advertised default). A contact method contains exactly `id`, `kind` (`email` or
`phone`), and `value`; do not invent label or primary fields. An affiliation
contains `id`, `organization`, and an optional `role`; do not invent a primary
flag.

Reparse every converted record with a YAML-aware parser or proven lossless
top-level patcher. Require one frontmatter block, unique keys, a non-empty title,
the expected type, and unchanged body and preserved metadata.

Upload all selected records and approved dependencies in the same sealed upload:

1. Call `nofray_begin_import` with an accurate `sourceKind`, a human-readable
   `sourceLabel`, and the exact record count.
2. Call `nofray_upload_import_batch` in ordered zero-based batches within the
   returned record and byte limits. Retry only the identical acknowledged batch.
3. Never mutate an acknowledged batch. Rebuild from the untouched source and
   begin a new upload when conversion semantics change.

## Source adapters

For TaskNotes or an Obsidian vault, use the configured task and archive folders
to select task records. Inspect configured project and person/contact folders
only to resolve approved dependencies. Resolve project wikilinks against those
selected records. A link to another task is hierarchy, not a NoFray project;
preserve it as approved source metadata or omit it after approval.

For another source, derive mappings from its documented export schema and the
actual supplied data. Keep source-specific parsing and discovery rules at this
adapter boundary; the upload, preview, confirmation, apply, and resume workflow
below stays identical for every source.

## Preview and resolve

1. First call `nofray_preview_import` with an empty `valueMappings` array.
2. Fetch every `records` and `diagnostics` page with
   `nofray_get_import_preview_page`. Maintain a client ledger from each
   `sourceReference` to its returned `recordReference`.
3. Present creates, identical records, conflicts, deferred records, exclusions,
   diagnostics, and the complete conversion ledger.
4. For each `requiredMappings` entry, recommend a target but obtain explicit
   approval. Status and priority targets come from `nofray_list_field_values`.
   Assignee targets are Contact IDs from existing or concurrently uploaded
   contact records; pass these IDs unchanged.
5. Re-preview the same sealed upload with only approved mappings, fetch all pages
   again, and use only this newest preview.

If the preview still has conflicts, deferred or excluded records, or error
diagnostics, report them before seeking apply approval. Never silently filter or
repair them after apply.

## Approve, apply, and resume

After showing the complete final preview, obtain natural-language approval from
the user for that exact plan. The user does not need to type an integrity hash.
Then:

1. Pass the preview's server-authored `previewID`, `planHash`,
   `targetWorkspaceID`, and `confirmationValue` unchanged to
   `nofray_apply_import`.
2. Call `nofray_get_import_status` before resuming an interrupted apply. It
   returns applied and remaining counts plus the current rolling expiry.
3. Continue `nofray_apply_import` without custom record references until
   `hasMore == false`. Every successful batch renews the preview lease.
4. Preserve receipts and report verified imported counts separately from
   identical, deferred, excluded, or conflicting records.

Hard stop: if the final desired state is absent from the confirmed preview, do
not apply it and do not repair the result through the ordinary proposal tools.
Rebuild and re-confirm one complete import plan instead.

If transport is lost, query status after reconnecting with the same authorized
principal. If the server reports an expired or invalid preview, stop and create
a fresh preview; never fabricate or reuse stale integrity values.

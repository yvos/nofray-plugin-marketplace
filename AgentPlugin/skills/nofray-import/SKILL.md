---
name: nofray-import
description: Use NoFray when the user wants to import or migrate tasks, projects, or contacts from pasted text, arbitrary source files, another task manager, Markdown, an Obsidian vault, or an export into their local NoFray workspace.
---

# Import external data into NoFray

Use NoFray's canonical import tools. Treat each source format as an adapter into
one server-owned, previewed batch workflow. Never replace missing import results
with ordinary per-record change proposals: those writes were not part of the
confirmed import plan.

The import workflow uses the v2 target contract discovered from the mounted
workspace. Call `nofray_get_workspace_configuration` before conversion and use
only the live schemas and advertised capabilities. Confirm that the registered
v2 import tools are available before uploading. There is no legacy import
envelope or silent lossy fallback. If the required v2 mutation capability or
import tool is absent, stop and report the missing capability.

## Establish source scope

The client reads and converts the source; NoFray never opens a supplied path or
connects to the source system.
The source format is unrestricted: pasted task lists, free text, Markdown,
tables, and other readable exports all use the same target contract. No source
schema, TaskNotes installation, or Obsidian metadata is required. For free text,
describe observed source components (such as the task text) in the inventory;
do not invent source fields just to match the target schema.

1. Use only the export, folder, archive, or records the user supplied. Identify
   the source system and format, then select only the requested tasks, projects,
   and contacts.
2. If scope is not already explicit, ask one grouped question: import all
   supplied projects and contacts (recommended), or only referenced
   dependencies. Include archived-record inclusion in that same question where
   practical. Never ask separate project and contact scope questions, and never
   broaden the scan to unrelated records or notes.
3. Never modify, rename, move, normalize, stage, or create a filtered copy of
   source files. If the client cannot read a local folder, ask for a supported
   export, zip, or client filesystem access; NoFray source-path permission cannot
   solve it.
4. Use documented source semantics when available and inspect the actual
   supplied content. If a source field or value is unclear, expose the ambiguity
   instead of guessing.

## Build a complete conversion ledger

Before conversion, call `nofray_get_workspace_configuration` with `recordTypes`
covering every target record type, verify the advertised `recordMutationV2`
and `recordMutationContractV2` capabilities, confirm the v2 import tools are
registered, and call `nofray_list_field_values`. Treat the returned live,
resolved mdbase schemas as the target-format authority; do not rely on a copied
schema. Inventory every observed source field in a complete client-owned ledger.
`translationManifest.sourceInventory` is required and must contain one
`TaskNotesImportObservedField` for every observed `(recordType, sourceField)`;
each entry declares `sourceField`, `recordType`, `valueShape`,
`observedRecordCount`, and `observedValueCount`. The field disposition ledger
must cover the same source entries exactly once. This is the source's
pre-conversion inventory, separate from canonical upload validation.
For each field record its value shape,
record-type coverage, proposed target, category, and exact merge, empty-value,
separator, and duplicate rules. Account for every field exactly once as:

- **Directly mapped**: exact one-to-one mapping or rename.
- **Proposed transformation**: merge, split, value-type change, title fallback,
  or other semantic conversion.
- **Preserved only as metadata**: retained in frontmatter, but not a functional
  writable NoFray field and not guaranteed to appear in NoFray's UI.
- **Proposed omission**: dropped only after explicit approval.

Exact direct mappings need no approval. Group every material transformation
and omission into one approval round; do not ask field-by-field questions.

Never describe retained frontmatter as a functional field merely because its
bytes survive. Do not hide metadata-only or omitted fields in an "unmapped"
count.

For example, if source `foo` becomes canonical `due` and source `approved` is
omitted after approval, both source fields remain in `sourceInventory` and
`fieldDispositions`; the uploaded record's canonical Markdown contains `due`
and does not need `foo` or `approved`. The upload contract is still exactly
`{sourceReference, recordType, markdown}`, and its target fields are validated
against the live target schema.

Resolve every project or contact reference against the selected source. Upload
the resolved dependency record with the records that reference it. If a source
relationship has no NoFray equivalent, preserve it as approved source metadata
or omit it after approval. Stop on absent or ambiguous references.

For every preview, include a v2 translation manifest, including when every
source field maps directly. Each observed field appears exactly once as `direct`,
`transformed`, `metadataOnly`, or `omitted`, with the target field and bounded
transformation explanation where applicable. Each relation mapping carries its
owner source reference, field, source token, target record type, and explicit
resolution. A changed manifest requires a new preview and plan hash.

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

Obsidian aliases and wikilinks can provide candidate evidence when present;
they are optional hints, not identity or input requirements. Distinct projects
may share a title; resolve them by stable IDs or explicit source references.

For another source, derive mappings from the actual supplied data and any
available export documentation. Propose field renames, merges, splits, and
body construction through the conversion ledger. Keep source-specific parsing
and discovery rules at this adapter boundary; the upload, preview,
confirmation, apply, and resume workflow below stays identical for every source.

## Preview and resolve

1. First call `nofray_preview_import` with the sealed upload, an empty
   `valueMappings` array, and the complete v2 translation manifest.
2. Fetch every `records` and `diagnostics` page with
   `nofray_get_import_preview_page`. Maintain a client ledger from each
   `sourceReference` to its returned `recordReference`.
3. Present a concise but confidence-building final preview: create, identical,
   conflict, deferred, and excluded counts; grouped diagnostics; a complete-ledger
   summary; pending decisions; the title-based filename policy and any
   collision/fallback exceptions; and representative
   `sourceReference` → `targetPath` examples. Fetch every page internally and
   state that full ledger and preview rows are available on request.
4. For each `requiredMappings` entry, recommend a target but obtain explicit
   approval. Status and priority targets come from `nofray_list_field_values`.
   These value mappings cover status and priority only. Resolve assignees and
   project links exclusively through `translationManifest.relationMappings`,
   using existing canonical IDs or explicit uploaded source references. Do not
   add a second assignee mapping to `valueMappings`.
5. Re-preview the same sealed upload with only approved value mappings and the
   unchanged approved translation manifest, fetch all pages again, and use only
   this newest preview. Relation mappings must remain explicit in the manifest.

If the preview still has conflicts, deferred or excluded records, or error
diagnostics, report them before seeking apply approval. Never silently filter or
repair them after apply.

## Approve, apply, and resume

After showing the complete final preview, obtain one natural-language approval
from the user for that exact sealed plan. The user does not need to type an
integrity hash.
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

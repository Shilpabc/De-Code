---
name: heat-chart-populator
description: >
  Stage 2 of the Heat Chart Automation Framework. Use this skill whenever a user wants to
  populate a Material Heat Chart from a BOM Extractor JSON output. Triggers include:
  "populate the heat chart", "create heat chart from BOM", "fill SR No and description",
  "map BOM to heat chart", "start the heat chart", or whenever a BOM JSON with title_block
  and bom[] fields is present. Also triggers when the user pastes JSON from the
  bom-drawing-extractor skill. IMPORTANT: On first use in a workspace, this skill has no
  registry configured yet — it must ask the user for their Material Registry details
  (Notion database URL, or a manual-paste format) before it can run Stage 2 at all.
  Once configured, that configuration should be re-used for the rest of the session
  without asking again, and the user can offer to save it permanently in their own
  workspace (e.g. a project config file) so future sessions skip setup.
pipeline_stage: 2
depends_on: bom-drawing-extractor
feeds_into: material-registry-matcher
schema_version: hc-1.0
---

# Heat Chart Populator — Stage 2 (Generic, Config-Driven)

## Role
You are a materials documentation engineer operating across Stage 1 and Stage 2 of the
Heat Chart Automation Framework. This skill matches BOM line items against a Material
Registry to populate traceability fields (heat/cast number, test certificate number,
date, manufacturer) on a Heat Chart.

This version is **generic** — it is not wired to any specific company's database. It
does not know your registry location, your Notion column names, or your status values
until you tell it. Treat configuration as a precondition for everything else in this
skill, not a footnote.

---

## ⚡ Step 0 — Registry Configuration (mandatory before first run)

**Do not attempt to fetch or match anything until a registry source is configured.**

Check, in order:
1. Has the user already provided registry config **earlier in this conversation**?
2. Does the user's workspace contain a saved config (e.g. a `heat-chart-config.json`,
   a note in a project file, or something they point you to)? If they reference one,
   read it before asking.
3. If neither — **this is first use. Stop and ask.** Do not guess column names, do not
   assume a Notion database, do not silently fall back to manual paste without offering
   the Notion path first.

### What to ask for

Ask the user, in plain language, for:

1. **Registry source** — Notion database (needs a URL/view link), a spreadsheet/CSV,
   or "I'll paste it manually each time."
2. **Column mapping** — what their registry actually calls these fields. Show the
   canonical fields this skill needs and ask them to map their own column names to it:

   | Canonical Field | What it represents                          | Their column name? |
   |-----------------|----------------------------------------------|---------------------|
   | `spec_grade`    | Material grade as specified on the drawing    | ?                   |
   | `spec_size`     | Size/thickness as specified                   | ?                   |
   | `ct_ht_no`      | Cast/heat/charge/coil traceability number     | ?                   |
   | `tc_no`         | Test certificate / MTC / MTR number           | ?                   |
   | `tc_date`       | Test certificate date                         | ?                   |
   | `actual_grade`  | Grade actually received (may differ from spec)| ?                   |
   | `lab_mfg`       | Testing lab or mill/manufacturer              | ?                   |

   If the user doesn't know offhand and has a sample export or screenshot, offer to
   infer the mapping from that instead of making them type it out field by field.

3. **Status filter** — which statuses in their registry should be **excluded** from
   matching (e.g. rejected, returned-to-supplier equivalents). Default to asking rather
   than assuming any status is excluded — do not exclude a "consumed" or "issued"
   equivalent by default, since heat charts are typically prepared after material is
   already in use.

4. **Notion-specific only**: database URL/view URL, and confirm `notion-query-database-view`
   (or equivalent search/fetch tool) is available.

### After they answer

- Build a config object in memory for this session (see schema below) and confirm it
  back to the user in one short summary before running anything: *"Got it — using your
  [Source] registry, mapping [grade column] → spec_grade, [size column] → spec_size,
  excluding rows where status is [X, Y]. Proceeding."*
- Offer, once, to save this as a reusable config file in their project so they don't
  need to repeat setup next time. Do not insist if they decline.
- From this point in the session onward, treat the registry as configured — do not
  re-ask.

### Config schema (save this if the user wants persistence)

```json
{
  "registry_source": "notion | csv | manual",
  "notion_database_url": "<url, if source is notion>",
  "notion_collection_id": "<optional, if known>",
  "column_mapping": {
    "spec_grade": "<their column name>",
    "spec_size": "<their column name>",
    "ct_ht_no": "<their column name>",
    "ct_ht_no_fallback": "<their fallback column, optional>",
    "tc_no": "<their column name>",
    "tc_no_fallback": "<their fallback column, optional>",
    "tc_date": "<their column name>",
    "actual_grade": "<their column name>",
    "lab_mfg": "<their column name>",
    "lab_mfg_fallback": "<their fallback column, optional>"
  },
  "extra_display_fields": {
    "job_no": "<their column name, optional>",
    "allocated_to": "<their column name, optional>",
    "inspection_status": "<their column name, optional>",
    "product_no": "<their column name, optional>",
    "material_title": "<their column name, optional>"
  },
  "excluded_statuses": ["<status value 1>", "<status value 2>"]
}
```

If the user has no extra display fields, leave that object empty — these are optional
context shown in the match UI, not required for matching.

---

## ⚡ Registry Fetch Timing (once configured)

Once a registry source is configured for the session, the same eagerness rule from the
original framework applies: if the `bom-drawing-extractor` skill (Stage 1) has just
completed in this same conversation, fetch the registry **immediately, in the same
turn**, before delivering the Stage 1 artifact — don't wait for the user to ask for
Stage 2 explicitly. The goal is that by the time the engineer is looking at Stage 1
output, the registry status line already shows loaded, not loading.

This only applies once configuration exists. On a genuinely first-time setup, fetching
nothing and asking the configuration questions above takes priority over speed.

---

## Fetching the Registry (Notion source)

Use the equivalent of `notion-query-database-view` against the configured database URL,
paginating with `next_cursor` / `start_cursor` until `has_more` is false.

**Treat fetching as incomplete by default.** Specifically:

- Track and report a running count: pages fetched, entries collected so far, whether
  `has_more` is still `true`. Keep paginating while `has_more` is `true`, regardless of
  how many calls that takes.
- **Never substitute targeted/row-by-row search for completing pagination.** If a bulk
  fetch is slow or a query attempt fails, retry the SAME paginated fetch — do not pivot
  to searching per-BOM-row as a workaround. Per-row searching against an incomplete
  registry is how an exact-match entry gets reported as "no match": it exists but was
  never pulled into memory before matching ran.
- If pagination is genuinely interrupted and cannot be completed (tool failure after
  retries, not just "many pages"), say so explicitly — e.g. "Only fetched 2 of an
  unknown total pages before the registry tool failed; matching below is against a
  partial registry and may show false no-match results." A silent partial fetch is a
  worse failure than a slow one.
- Do not report the registry as loaded (`✅ {N} entries loaded`) until `has_more` is
  confirmed `false`. An in-progress count is not a loaded count.

**Filter**: Exclude rows where the configured status field matches one of the user's
`excluded_statuses`. Include everything else. Do not maintain this as a hardcoded
allowlist — a denylist of genuinely disqualifying statuses is the correct shape, since
an allowlist silently drops any new/unanticipated status value without warning.

### Mapping fetched rows to canonical fields

Using the session's `column_mapping`, map each fetched record to:

```json
{
  "spec_grade": "<mapped grade column>",
  "spec_size": "<mapped size column>",
  "ct_ht_no": "<mapped CT/HT column, or fallback if blank>",
  "tc_no": "<mapped TC column, or fallback if blank>",
  "tc_date": "<mapped date column, formatted DD.MM.YYYY>",
  "actual_grade": "<mapped actual-grade column>",
  "lab_mfg": "<mapped lab/mfg column, or fallback if blank>",
  "job_no": "<mapped extra field, if configured>",
  "allocated_to": "<mapped extra field, if configured>",
  "inspection_status": "<mapped extra field, if configured>",
  "product_no": "<mapped extra field, if configured>",
  "material_title": "<mapped extra field, if configured>",
  "source_url": "<page/row url, if available>"
}
```

Skip entries where both `spec_grade` and `spec_size` are blank.

---

## Step 2 — Normalise and Match

For each BOM row, find all matching registry entries.

### Normalize function
- Strip whitespace, uppercase, collapse separators (`-`,`/`,`.`,`_`,` `) to single space

### Match priority (in order)
1. **Exact**: `norm(spec_grade) == norm(entry.spec_grade)` AND `norm(spec_size) == norm(entry.spec_size)`
2. **Grade family**: One grade string contains the other AND exact size match
3. **No match**: No registry entry found for this material

### Match outcomes per row

| Matches found | Outcome           | UI behaviour                                                          |
|---------------|--------------------|------------------------------------------------------------------------|
| Exactly 1     | `auto_matched`     | Utilized fields filled automatically, row STILL requires manual Approve click — never pre-approved |
| 2 or more     | `needs_selection`  | All matching entries shown as radio button options, then manual Approve |
| 0             | `no_match`         | Utilized and TC fields left blank for manual entry                   |

**No row is ever auto-approved, regardless of match count.** `auto_matched` only means
the utilized grade/size/CT/TC fields were pre-filled from a single exact registry hit —
it does not mean the row is approved. The engineer must review and click Approve on
every included row before "Generate Heat Chart" becomes available. This is a deliberate
human-in-the-loop checkpoint: even a confident exact match should be eyeballed before
it's stamped into a traceability document, since the registry itself can be wrong
(wrong status, stale entry, duplicate heat number).

---

## Step 3 — Build Row Objects

For each BOM item produce a row object:

```json
{
  "sr_no": "1",
  "description": "Shell Plate",
  "spec_grade": "IS 2062 E250",
  "spec_size": "8mm THK",
  "util_grade": "IS 2062 E250 GR.B",
  "util_size": "8mm THK",
  "ct_ht_no": "HT-2526-001",
  "tc_no": "TC-2526-A01",
  "tc_date": "12.11.2025",
  "lab_mfg": "EXAMPLE STEEL MILL",
  "status": "auto_matched",
  "matches": [],
  "selected_idx": 0,
  "approved": false
}
```

For `auto_matched` rows: fields are pre-filled, but `approved` starts as `false` — the
engineer must still click Approve. Pre-filled is not pre-approved.

For `needs_selection` rows: `util_grade`, `util_size`, `ct_ht_no`, `tc_no`, `tc_date`,
`lab_mfg` are all `""` and `approved` is `false` until engineer selects, then approves.

For `no_match` rows: all util and TC fields are `""`, `approved` is `false` until
engineer fills manually and clicks approve.

---

## Step 4 — Extend the Unified Pipeline Artifact (Mandatory)

**Do not build a separate artifact.** This skill never produces its own standalone `.jsx`
file. It extends the SAME component that `bom-drawing-extractor` is building in this turn
(or, in standalone re-entry, the existing `{drawing_no}_HeatChart_Pipeline.jsx` if the
user has it open or has pasted its contents).

### Unified Pipeline Artifact — required structure

One `export default function HeatChartPipeline()` component containing:

- **Shared header**: equipment/customer banner + a 3-stage progress bar
  (Stage 1: BOM Extraction | Stage 2: Review & Match | Stage 3: Heat Chart), each stage
  clickable once reached, current stage highlighted, completed stages ✓.
- **`Stage1View`**: the full tabbed extraction view (Title Block / Design Data / BOM /
  Nozzle Schedule / Revision / Notes) — this is what `bom-drawing-extractor` Step 4
  describes. Do not rebuild this; reuse it as the Stage 1 panel of the unified component.
- **`Stage2View`** (this skill's responsibility): the review-and-match table described
  below, operating on a `rows` array held in the parent component's state
  (`useState(INITIAL_BOM_ROWS)`), passed down with a `setRows` updater.
- **`Stage3View`** (built by `material-registry-matcher` logic, same component): the
  final heat chart, rendered from the same `rows` state once Stage 2 is approved.

A single `useState(1)` in the parent component tracks which stage is visible. Switching
stages is a state change, not a new artifact, not a new message, not a new file.

### Stage 2 review table — columns and behavior

Columns: Incl. (toggle) | SR | Description | Spec Grade | Spec Size |
**Registry Match / Utilized** | CT/HT No. | TC No. | TC Date | Lab/MFG | Status | Approve

Each row carries an `included: true/false` toggle (default `true`) — excluded rows stay
visible, dimmed, below a divider, and do not count toward the generate-gate or appear in
Stage 3. This lets the engineer remove BOM items that don't belong on the heat chart
without losing the extraction record.

**The "Registry Match / Utilized" column:**
- `auto_matched` rows: `util_grade` and `util_size` in green text (read-only). Small grey
  tooltip with whatever extra display fields were configured (e.g. job/allocation info).
- `needs_selection` rows: Radio button list of all matching entries. Each option:
  `Actual Grade / Spec Size · CT/HT No · TC No · TC Date · Lab/MFG`. Sub-line shows
  configured extra fields if present. If a candidate's grade family doesn't actually
  match the spec (e.g. registry has the right size but wrong/lower grade), flag it
  inline in red — do not silently offer it as an equal option.
- `no_match` rows: Two text inputs — "Utilized Grade" and "Utilized Size", plus editable
  CT/HT No., TC No., TC Date, Lab/MFG inputs.

**Approve column**: per-row "Approve" button, disabled until a selection/manual entry
exists; ✓ green tick once approved. "Approve all selected" bulk action for rows already
on `selected`/`auto_matched`.

**"+ Add Item to Heat Chart" panel** below the table: lets the engineer add a row not in
the original BOM (bought-out items, additional plates) with the same fields, manually.

**"✓ Generate Heat Chart →"** (advances the shared stage state to 3): enabled only when
every `included` row is either `approved: true` or `status === "no_match"` with at least
the description/grade filled.

### Registry status (shown in Stage 1, not a separate tab)

Add a small status line under the Stage 1 metric cards, populated as soon as the
registry fetch completes in the background:
- `🔄 Loading Material Registry from [Source]...` while fetching
- `✅ {N} entries loaded from [Source]` on success, with "View ↗" link if a URL exists
- `⚠ Registry fetch failed — paste registry manually` + fallback textarea on failure

By the time the engineer reaches Stage 2 (in a session where the registry was already
configured), this should already read "✅ ... loaded."

---

## Step 5 — Status Badge Reference

| Status            | Badge           | Color |
|-------------------|-----------------|-------|
| `auto_matched`    | ✓ Auto-matched  | Green |
| `needs_selection` | ⬦ Choose        | Amber |
| `selected`        | ◉ Selected      | Blue  |
| `no_match`        | ✕ No match      | Red   |
| `approved`        | ✓ Approved      | Green |
| `matched` (final) | ✓ Matched       | Green |
| `pending` (final) | ⚠ Pending       | Amber |

---

## Step 6 — Final Heat Chart JSON

On "Generate Heat Chart", produce the `hc-1.0` output:

```json
{
  "schema_version": "hc-1.0",
  "header": { "...from title_block...": "" },
  "rows": [
    {
      "sr_no": "1",
      "description": "Shell Plate",
      "spec_grade": "IS 2062 E250",
      "spec_size": "8mm THK",
      "util_grade": "IS 2062 E250 GR.B",
      "util_size": "8mm THK",
      "ct_ht_no": "HT-2526-001",
      "tc_no": "TC-2526-A01",
      "tc_date": "12.11.2025",
      "lab_mfg": "EXAMPLE STEEL MILL",
      "status": "matched"
    }
  ],
  "meta": {
    "stage": 3,
    "total_rows": 8,
    "matched": 7,
    "pending": 1,
    "drawing_no": "<from title block>",
    "registry_source": "<notion | csv | manual>",
    "registry_url": "<configured url, if any>",
    "completed_at": "<ISO timestamp>"
  }
}
```

Rows with no match: `ct_ht_no`, `tc_no`, `tc_date`, `lab_mfg` = `"PENDING"`, `status` = `"pending"`.

---

## Fallback: Manual Registry Input (when Notion/source is unreachable, or user chose manual)

Show fallback textarea with pipe-separated format hint. Apply column aliases on top of
whatever the user configured in Step 0:

| Canonical    | Also accept                                                     |
|--------------|-----------------------------------------------------------------|
| `spec_grade` | Grade, Material Grade, IS Grade, ASTM Grade                    |
| `spec_size`  | Size, Thickness, THK, NB, Size Spec                            |
| `ct_ht_no`   | CT No, HT No, Heat No, Cast No, Charge No, Coil No             |
| `tc_no`      | TC No, MTC No, MTR No, Certificate No                          |
| `tc_date`    | Date, Test Date, Cert Date, TC Date                            |
| `actual_grade` | Actual Grade, As Received                                    |
| `lab_mfg`    | Lab, Manufacturer, Mill, Steel Mill, Lab MFG, Mill TC Manufacturer |

---

## Error Handling

| Condition                              | Action                                                        |
|----------------------------------------|---------------------------------------------------------------|
| No registry configured yet             | Run Step 0 setup before anything else — do not proceed        |
| BOM JSON missing                       | Ask user to paste Stage 1 output                              |
| Registry fetch fails (network/auth)    | Show amber warning; offer manual paste fallback               |
| Registry returns 0 usable rows         | Warn: check status filter or registry population              |
| Registry has 0 parseable rows (manual) | Show first 2 lines, ask user to check format                  |
| `material` blank on BOM row            | Set `spec_grade = "NOT SPECIFIED"`, warn                       |
| `size` blank on BOM row                | Set `spec_size = "NOT SPECIFIED"`, warn                       |
| All rows show no_match                 | Warn: registry may be wrong project, wrong format, or column mapping is off |
| Multiple hits same grade+size          | Show all as dropdown options, newest TC date first            |

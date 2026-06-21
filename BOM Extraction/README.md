# pdf-drawing-to-json

A Claude skill that converts any technical/engineering drawing PDF — mechanical, structural, civil,
electrical, fabrication — into structured JSON, plus an interactive React artifact for browsing the result.

---

## What

Given a technical drawing PDF, this skill:

1. Locates the data-bearing regions of the drawing (title block, tables, notes, revision history) and
   ignores the graphical/geometric content.
2. Extracts those regions into a **fixed-shape JSON envelope**:
   - `title_block` — whatever identification fields are actually present
   - `tables` — an array of `{ name, columns, rows }`, one entry per data table found on the drawing
   - `revision_history`, `notes`, `other_info`, `uncertain_fields`
3. Builds an interactive `.jsx` artifact with one tab per detected table, plus fixed tabs for the title
   block, revision history, notes, and raw JSON.

It does **not** assume which tables a drawing contains. A mechanical drawing might have a Bill of
Materials and a Nozzle Schedule; a structural drawing might have a Rebar Schedule; an electrical drawing
might have a Cable Schedule. The skill detects whatever is actually printed on the drawing rather than
forcing it into a predefined category.

---

## Why

Most BOM/drawing-extraction tooling (including the skill this one was derived from) hardcodes the
schema to one discipline: fixed section names (`bom`, `nozzle_schedule`), a domain abbreviation
dictionary, and discipline-specific UI (e.g. material-grade color badges). That works well for one
recurring drawing type and breaks immediately outside it.

This skill exists for the opposite case: **one extraction tool that works across disciplines**, at the
cost of giving up a fully fixed schema.

The trade-off is explicit and intentional:

| | Fixed-schema skill (e.g. discipline-specific) | This skill |
|---|---|---|
| Output shape | Identical keys every run | Identical *top-level* keys; table contents vary by drawing |
| Coverage | One discipline, done well | Any discipline, generically |
| Downstream parsing | Can hardcode field access | Must iterate `tables[]` and read `columns`/`rows` dynamically |
| Domain vocabulary | Built-in abbreviation reference | None — abbreviations preserved verbatim, never expanded |

If you need guaranteed fixed keys for a recurring drawing type (so a downstream script can safely do
`data["bom"][0]["material"]` every time), build a dedicated skill for that drawing type instead of relying
on this one. This skill is for breadth, not for pipeline-grade consistency on a single format.

---

## How

### Extraction strategy
- **Text-first.** The skill runs `pdfplumber` text extraction across all pages before touching anything
  visual, and classifies pages as data-bearing vs. graphical-only based on word count and detected tables.
  Only data-bearing pages are processed further.
- **Image rendering is a fallback, not a default.** It only triggers when text extraction returns garbled
  or empty results for a table that's clearly present (e.g. scanned PDFs) — and even then, only the
  bottom data-region of the page is rasterized, not the full drawing.

### JSON shape
```json
{
  "drawing_meta": { "discipline_guess": "", "drawing_type": "" },
  "title_block": {},
  "tables": [
    { "name": "", "columns": [""], "rows": [{}] }
  ],
  "revision_history": [{ "rev": "", "date": "", "description": "", "drawn": "", "checked": "", "approved": "" }],
  "notes": [""],
  "other_info": [{ "section": "", "label": "", "value": "", "source": "" }],
  "uncertain_fields": [""]
}
```
- `tables` is an array because a drawing can contain more than one named table.
- Row objects use the table's own printed column headers as keys — there is no canonical column list to
  conform to.
- `title_block` only includes keys for fields actually present on that drawing — no padding with empty
  placeholders for fields that don't apply.

### Artifact generation
The `.jsx` artifact is built dynamically: one tab is generated per entry in `tables[]`, using that table's
own columns as the table header. There's no per-table custom rendering logic (no material-grade badges,
no discipline-specific color coding) — styling stays generic so it doesn't silently assume a discipline.

### Installation
Drop the `pdf-drawing-to-json/` folder into your skills directory (e.g. `/mnt/skills/user/` in Claude.ai,
or wherever your Claude environment loads user skills from). The skill triggers automatically when you
upload a technical drawing PDF and ask to extract, parse, or convert it — no explicit invocation needed.

### Known limitations
- No domain abbreviation expansion — abbreviations print exactly as found on the drawing.
- `discipline_guess` is a best-effort label for the artifact header, not a verified classification — don't
  build logic downstream that depends on it being correct.
- Variable table structure means downstream consumers must handle `tables[]` generically (loop + read
  `columns`/`rows`), not via fixed key access.

---

## Derived from

This skill is a discipline-generic rewrite of an internal pressure-vessel-specific BOM extractor. The
fixed-schema version remains the better choice if you're working repeatedly with the same drawing type and
want guaranteed, hardcoded output keys.

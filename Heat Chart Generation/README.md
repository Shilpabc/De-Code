# heat-chart-populator

Stage 2 of a 3-stage Heat Chart Automation Framework. Matches BOM line items
(extracted from an engineering drawing) against a Material Registry to fill in
traceability fields — heat/cast number, test certificate number, date, mill/lab —
on a Material Heat Chart.

## Why this exists

The original version of this skill was built against one specific Notion database,
with column names, a database URL, and status values all hardcoded into the skill
file. It worked, but only for that one registry — anyone else (or any other project)
using it would need to fork and hand-edit the skill itself to point at their own data.

This version moves all of that out of the skill and into a **session config** the
skill asks for on first use. The matching logic, status handling, and artifact
structure are unchanged; what changed is that nothing here assumes whose registry
it's reading.

## What it does

- Takes a BOM JSON (the output of the `bom-drawing-extractor` skill — Stage 1) and a
  Material Registry (Notion database, CSV, or manual paste) and matches each BOM row
  against the registry by grade + size.
- Classifies every row as `auto_matched` (exactly one hit), `needs_selection`
  (multiple hits — engineer picks), or `no_match` (none — engineer fills manually).
- Extends a shared 3-stage React artifact (Stage 1 Extraction → Stage 2 Review/Match
  → Stage 3 Heat Chart) rather than producing a standalone file — Stage 2 is a state
  change on the same component Stage 1 already built.
- Never auto-approves a row, even on an exact match. Every row needs a manual
  "Approve" click before the heat chart can generate — matches get pre-filled, not
  pre-confirmed.
- Outputs a structured `hc-1.0` JSON heat chart, ready for `material-registry-matcher`
  (Stage 3) to finish.

## What it does not do

- It does not know your registry's column names, database location, or status
  values until you tell it. There is no default Notion database wired in.
- It does not skip the human-approval step, regardless of match confidence.
- It does not silently exclude registry rows by status unless you've told it which
  statuses to exclude — it asks rather than assuming, and defaults to including
  ambiguous/new status values rather than dropping them.
- It does not build its own standalone artifact — it's designed to extend the
  `bom-drawing-extractor` artifact, not run fully standalone end to end. You can
  still re-enter Stage 2 on its own if you paste/upload the Stage 1 BOM JSON.

## Setup (first use)

There's no config file shipped with this skill — it interviews you the first time
it's invoked in a session/workspace where it doesn't already know your registry:

1. **Registry source** — Notion database, CSV/spreadsheet, or manual paste each time.
2. **Column mapping** — what your registry calls grade, size, heat/cast number, TC
   number, TC date, actual grade received, and lab/mill. The skill shows you its
   canonical field list and you map your columns onto it (or hand it a sample export
   to infer from).
3. **Status filter** — which status values should be excluded from matching (e.g.
   rejected, returned-to-supplier). Everything else is included by default —
   "consumed" or "issued" material is *not* excluded by default, since heat charts
   are usually built after material is already in use.
4. (Notion only) the database/view URL.

The skill confirms the config back to you in one line before running anything, and
offers once to save it to a file in your project so you don't repeat setup next
session. It won't nag if you decline, and it won't re-ask mid-session once
configured.

## How to invoke

- After `bom-drawing-extractor` finishes in the same conversation, this skill should
  trigger on its own — registry fetch happens in the background while you're still
  looking at Stage 1 output, *if* a registry is already configured for the session.
- Standalone: paste or reference a Stage 1 BOM JSON and say something like "populate
  the heat chart" or "map this BOM to the registry."

## File layout

```
heat-chart-populator/
└── SKILL.md
```

Single-file skill — under the 500-line guidance, so no `references/` or `scripts/`
subfolder was needed. If you end up running this against several genuinely different
registries (not just successor projects to the same one), consider splitting config
persistence into a small script rather than relying on the model to hold mapping
state across a long conversation.

## Known limitations

- Not yet tested end-to-end against a live non-Notion source (CSV/manual path is
  spec'd but the original was only exercised against one Notion workspace).
- Grade-family matching ("one grade string contains the other") is a heuristic, not
  a real metallurgical equivalence check — it will surface a candidate match, but a
  human still has to judge whether e.g. a higher grade is an acceptable substitution.
- Column-mapping inference from a sample export is described but depends on the
  sample being representative; a sparse or unusual sample may produce a mapping that
  needs correcting by hand.

## Maintainer's note

This and the other generic registry-style skills in this repo (`po-registry-manager`,
`enquiry-registry-manager`, `work-order-registry-manager`, `material-registry-manager`,
`register-management`) overlap conceptually — all of them are "talk to a Notion
register, generically." Resist consolidating them into one mega-skill until the
config-driven pattern has more real-world mileage across different registry shapes.
Premature consolidation here would trade working, separately-triggered skills for one
skill with a much harder-to-get-right routing problem.

# po-registry-manager

Full CRUD manager for a Notion-backed Purchase Order register: create entries from a
PO PDF or manual input, query by PO number/supplier/status/project/date, update
status through a configurable lifecycle (Open → Partial → Received / On Hold /
Cancelled), and generate pipeline dashboards or overdue reports.

## Why this exists

The original version of this skill was built against one specific company's Notion
database — hardcoded data source ID, hardcoded property names, a hardcoded status
lifecycle, and a hardcoded list of real client/project names baked into the schema
documentation as the only valid values for a "Project ID" field. It also carried a
fully worked reference example at the bottom: a real PO number, a real supplier name,
and real line-item values from an actual transaction.

None of that travels. Anyone using this skill for their own registry would need to
fork the file and hand-edit field names, status values, and the project list before
it would work at all — and the client list and worked example are exactly the kind of
data that shouldn't be sitting in a publicly shared skill file regardless of reuse.

This version moves all company-specific detail out of the skill and into a session
config the skill asks for on first use. The CRUD logic, edge-case handling, and
dashboard structure are unchanged. The client/project list and the worked example were
not genericized — they were deleted outright, because there's no safe placeholder
version of someone else's confidential customer list.

## What it does

- Extracts PO header and line-item data from an uploaded PDF, confirms a short list
  of fields that typically aren't on the PDF itself (indent number/date, project
  reference, inspection type, required delivery date) before writing anything.
- Creates all line items for a PO in a single batched Notion write, not a loop.
- Queries the register by PO number, supplier, status, or project, and builds a
  dashboard (counts + ₹ totals by status, plus an overdue list) using whatever status
  labels the user's registry actually has.
- Updates status through whatever lifecycle the user defines — not a fixed five-state
  model — including marking partial/full receipt, putting a PO on hold, cancelling,
  and a soft-delete pattern (Notion has no hard delete via the API, so this archives
  by status + a `[VOID]` title prefix instead).

## What it does not do

- It does not know your Notion database, your field names, or your status values
  until you tell it. There is no default database wired in.
- It does not ship with — or invent — a list of project, client, or customer names.
  If your `project_ref` field is a constrained list rather than free text, you supply
  your own values every time you set this up in a new workspace. This is the one
  place genericizing the original wasn't good enough; the list itself needed to not
  exist in a shared file.
- It does not include a worked example with real transaction data. The original's
  "Reference Example" section was a real PO — number, supplier, six line items, and a
  real total — and was removed rather than anonymized, the same way you wouldn't
  anonymize-and-keep a customer list either.
- It does not hard-delete Notion pages — the API doesn't support it, so cancellation
  is always a soft archive, and the skill says so rather than implying otherwise.

## Setup (first use)

No config file ships with this skill. On first invocation in a workspace where it
doesn't already know your registry, it asks for:

1. **Notion target** — database/view URL.
2. **Title pattern** — how you want the page title built (e.g.
   `{PO Number} · {Supplier} · {category}`).
3. **Field mapping** — what your registry calls each of ~24 canonical fields (PO
   number, dates, supplier, status, line-item fields, etc.). You can hand it a sample
   export instead of typing the mapping out by hand.
4. **Status lifecycle** — your actual status values and how they flow, not the
   original's fixed `Open → Partial → Received / On Hold / Cancelled`.
5. **Inspection options**, if that field exists in your schema.
6. **Project/client reference values**, if that field is a constrained list. This is
   asked fresh every time, on purpose — see above.

It confirms the config back in one line before writing anything, and offers once to
save it to a project file so you don't repeat setup next session. No re-asking
mid-session once configured.

## How to invoke

Upload a PO PDF and say "log this PO," or describe a PO manually. For queries: "show
open POs," "PO dashboard," "find PO [number]," "supplier spend," "mark PO [number]
received," etc. Full trigger list is in the skill's frontmatter.

## File layout

```
po-registry-manager/
└── SKILL.md
```

Single-file skill, well under the 500-line guidance — no `references/` or `scripts/`
needed at this size.

## Known limitations

- PDF extraction quality depends on the PO format — multi-currency, multi-tax-regime,
  or heavily templated PDFs from unfamiliar formats may need manual correction after
  extraction, same as the original.
- The status-lifecycle config is taken at face value; the skill doesn't validate that
  a user-defined lifecycle is internally consistent (e.g. a "Received" state that
  can't actually be reached from "Open" in their flow).
- Like `heat-chart-populator`, this hasn't been run end-to-end against a fresh,
  unconfigured workspace yet — the config-driven path is spec'd from the original's
  working hardcoded version, not separately battle-tested.

## Maintainer's note

Same note as `heat-chart-populator`: this overlaps conceptually with the other
generic registry skills in this repo. Don't consolidate into one mega-skill until the
config-driven pattern has more real mileage across genuinely different registry
shapes — and specifically for this one, don't let a future "let's merge the registry
skills" pass reintroduce a shared client/project list as a convenience default. That
list staying empty until a user supplies their own, every time, is a deliberate
choice, not a gap to "fix" later.

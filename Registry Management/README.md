# register-management

A generic register builder and manager for manufacturing/industrial registers in Notion.

---

## Why

Every Notion-backed register in this ecosystem — Material Registry, PO Registry, Work
Order Registry, Enquiry Desk — was built the same way: one skill, hardcoded to one Notion
database, with every field name typed directly into the skill's logic. That's fine when you
know in advance exactly which registers you'll ever need. It breaks the moment you don't.

The actual recurring need isn't "a material registry" or "a customer registry" — it's the
same five-step shape over and over: **define fields → scaffold a Notion DB → create entries
→ bulk import → query and update**. The fields change every time; the shape doesn't.

`register-management` exists to capture the shape once, so creating a tenth register type
doesn't mean writing a tenth hardcoded skill. New register types should never require
opening this file and editing it — only the *config* changes.

**What this skill is not**: a replacement for the MPIPL-specific registry skills
(`mpipl-material-registry`, `mpipl-po-registry`, `mpipl-wo-manager`,
`mpipl-enquiry-desk`/`mpipl-inquiry-manager`). Those stay as dedicated skills because they
encode workflow logic specific to those domains (heat chart linkage, TC traceability, PO
status transitions) that goes beyond generic CRUD. This skill is for the *next* register
type that doesn't have a home yet.

---

## What

A two-phase system:

**Phase 1 — Define.** Interview the user (or read a sample file) to determine what a new
register needs to track. Produces a config file —
`registers/{slug}-register.json` — that fully describes the register: title field, status
workflow, every field with its type and options. Four manufacturing-relevant templates ship
as starting points (Customer Master, Equipment/Asset, Drawing, Subcontractor/Vendor
Approval), but any register type can be defined from scratch.

**Phase 2/3 — Build & Operate.** Translate the config into a Notion DDL schema, create the
actual Notion database (`Notion:notion-create-database`), write the resulting database/data
source IDs back into the config, and from then on drive every create/read/update/bulk-import
operation off the config — never off a hardcoded field name.

A running index (`registers/_index.json`) tracks every register this skill has created, so
a future session can find "the Customer Register" without re-deriving its schema.

### What it does NOT do

- Does not retrofit itself onto pre-existing hardcoded registry skills — those keep their
  own logic.
- Does not auto-detect schema from an arbitrary pre-existing Notion DB with no config (out
  of scope for v1 — see Known limitations).
- Does not run without a confirmed field list. It will propose fields but will not create a
  Notion database silently — creating a DB is harder to undo than asking one question.

---

## How

### Creating a new register

Say what you want to track. Be as specific or as vague as you like:

> "I need a register for tracking calibration of measuring instruments"

The skill will:
1. Propose a field list (using a template if your request matches one, or building fresh)
2. Show it to you as a table for review/edits
3. On confirmation, write `registers/calibration-register.json`
4. Scaffold the actual Notion database from that config
5. Write the resulting Notion database/data source IDs back into the config
6. Add an entry to `registers/_index.json`

### Using an existing register

Once a register exists, just talk about it naturally — the skill loads its config from the
index and operates against the live Notion DB:

> "Log a new vendor in the subcontractor register: ABC Fabricators, NDT scope, approved"
> "Show me all equipment overdue for calibration"
> "Mark drawing DWG-2024-0091 as Approved"
> "Import this Excel into the customer register" *(with file attached)*

### Changing a register's structure later

> "Add a 'Last Service Date' field to the equipment register"

This updates both the live Notion schema (`Notion:notion-update-data-source`) and the local
config — the two are kept in sync deliberately; nothing in this skill ever trusts one
without the other.

### File layout

```
register-management/
├── SKILL.md                          — the skill logic (read by Claude, not by you)
├── README.md                         — this file
└── registers/
    ├── _index.json                   — list of all registers created
    └── {slug}-register.json          — one config per register, created on demand
```

---

## Known limitations (as of first build)

- **Untested against a live Notion workspace.** The skill has been written and structurally
  validated, but no register has actually been created end-to-end yet. Treat the first
  real use as a test, not a guarantee.
- **No schema-drift detection automation.** If someone edits the Notion DB directly (adds a
  field outside this skill), the config will silently go stale until something forces a
  comparison. The skill is instructed to flag mismatches when it notices them, not to poll
  for them.
- **No deletion/archival workflow defined.** Only define → build → operate are specified.
  Retiring a register isn't yet handled.

---

## Maintainers' note for the `decode` repo

If this skill gets used and proves the config-driven pattern out, the four MPIPL-specific
registry skills are candidates to eventually be re-expressed as configs *on top of* this
engine rather than as standalone skills — but that's a deliberate future migration, not
something to do opportunistically. Don't merge them until this skill has real mileage.

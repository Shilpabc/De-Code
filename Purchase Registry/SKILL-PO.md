---
name: po-registry-manager
description: >
  Full CRUD Purchase Order Registry manager for a Notion-backed PO register.
  Use this skill for ANY PO operation: CREATE new entries from PDF/manual input,
  READ/query the register by PO number, supplier, status, project, or date, UPDATE
  status (Open to Partial to Received / On Hold / Cancelled), and generate pipeline
  dashboards or overdue reports. Trigger phrases: "log this PO", "new PO", "add to
  PO register", "upload purchase order", "mark received", "mark partial", "PO
  status", "cancel PO", "hold PO", "find PO", "search PO", "show open POs",
  "overdue POs", "supplier spend", "PO dashboard", "PO pipeline", "update PO",
  "delivery status", "how many POs", or any mention of PO numbers or supplier
  names in a purchase-order-register context. Config-driven, no hardcoded
  company/database — see Step 0 in the skill body for first-use setup.

compatibility:
  required_tools:
    - Notion MCP (notion-fetch, notion-query-database-view, notion-create-pages,
      notion-update-page, notion-search)
---

# PO Registry Manager (Generic, Config-Driven)

Manages a **Purchase Order Register** Notion database with full
Create / Read / Update / Archive (soft-delete) operations.

This version is **generic** — it is not wired to any specific company's database,
project list, or supplier history. It does not know your registry location, your
exact Notion field names, your status values, or your project/client taxonomy until
you tell it. Configuration is a precondition for every operation below, not optional
setup.

---

## ⚡ Step 0 — Registry Configuration (mandatory before first run)

**Do not attempt to read, write, or query anything until configuration exists.**

Check, in order:
1. Has the user already provided registry config **earlier in this conversation**?
2. Does the user's workspace contain a saved config (e.g. `po-registry-config.json`,
   or something they point you to)? If referenced, read it before asking.
3. If neither — **this is first use. Stop and ask.** Do not guess field names, do not
   assume a Notion database, do not invent a project/client list.

### What to ask for

1. **Notion target** — database URL (or view URL), and confirm Notion MCP access.
2. **Title field convention** — how they want the page title built (e.g.
   `{PO Number} · {Supplier} · {Item category}`) — ask for their preferred pattern
   rather than assuming one.
3. **Schema field names** — show the canonical fields this skill needs and ask the
   user to map their own Notion property names onto them:

   | Canonical Field | Represents | Type | Their Notion property name? |
   |---|---|---|---|
   | `po_number` | PO/voucher number | text | ? |
   | `po_date` | PO date | date | ? |
   | `indent_number` | Internal requisition ref | text | ? |
   | `indent_date` | Requisition date | date | ? |
   | `project_ref` | Project/client/job reference | text | ? |
   | `supplier_name` | Supplier name | text | ? |
   | `supplier_code` | GSTIN or internal code | text | ? |
   | `payment_terms` | Payment terms | text | ? |
   | `delivery_terms` | Delivery terms | text | ? |
   | `inspection` | Inspection type | select | ? |
   | `required_delivery_date` | Required delivery date | date | ? |
   | `status` | PO status | select | ? |
   | `gst_pct` | GST % | number | ? |
   | `discount_pct` | Discount % (optional) | number | ? |
   | `nett_po_value` | Total incl. tax | number (currency) | ? |
   | `item_description` | Line item description | text | ? |
   | `item_code` | Line item code | text | ? |
   | `quantity` | Quantity | text or number | ? |
   | `unit` | Unit (Nos/Mtr/Kgs/etc.) | text | ? |
   | `size_spec` | Size/spec | text | ? |
   | `make` | Make/type | text | ? |
   | `moc_spec` | Material-of-construction spec | text | ? |
   | `weight` | Weight | number | ? |
   | `basic_rate` | Rate per unit | number (currency) | ? |
   | `line_value` | Line amount (pre-tax) | number (currency) | ? |
   | `material_received_date` | Date material arrived | date | ? |

   If the user has a sample export or screenshot of their database, offer to infer
   the mapping from that instead of making them fill the table by hand.

4. **Status lifecycle** — what statuses exist and how they flow (e.g.
   `Open → Partial → Received`, plus terminal/hold states like `Cancelled`,
   `On Hold`). Ask rather than assume a fixed five-state model.
5. **Inspection options** (if the field exists) — what values are valid (e.g.
   internal / third-party / client / none). Ask for their actual labels.
6. **Project/client reference values** — if `project_ref` is a constrained select
   field rather than free text, ask the user for their actual list, or to confirm it's
   free text. **Never invent or carry over a project/client list — this is
   confidential business data and must come from the user fresh, every workspace.**

### After they answer

- Build a config object in memory for this session (see schema below) and confirm it
  back in one short summary: *"Got it — logging to [database], [N] fields mapped,
  status flow is [X → Y → Z], [N] project references provided. Proceeding."*
- Offer, once, to save this as a reusable config file in the user's project. Don't
  insist if declined.
- From this point in the session onward, treat the registry as configured.

### Config schema (save this if the user wants persistence)

```json
{
  "notion_database_url": "<url>",
  "notion_data_source_id": "<id, if known>",
  "title_pattern": "{po_number} · {supplier_short} · {item_category}",
  "field_mapping": {
    "po_number": "<their property name>",
    "po_date": "<their property name>",
    "indent_number": "<their property name>",
    "indent_date": "<their property name>",
    "project_ref": "<their property name>",
    "supplier_name": "<their property name>",
    "supplier_code": "<their property name>",
    "payment_terms": "<their property name>",
    "delivery_terms": "<their property name>",
    "inspection": "<their property name>",
    "required_delivery_date": "<their property name>",
    "status": "<their property name>",
    "gst_pct": "<their property name>",
    "discount_pct": "<their property name>",
    "nett_po_value": "<their property name>",
    "item_description": "<their property name>",
    "item_code": "<their property name>",
    "quantity": "<their property name>",
    "unit": "<their property name>",
    "size_spec": "<their property name>",
    "make": "<their property name>",
    "moc_spec": "<their property name>",
    "weight": "<their property name>",
    "basic_rate": "<their property name>",
    "line_value": "<their property name>",
    "material_received_date": "<their property name>"
  },
  "status_lifecycle": {
    "flow": ["Open", "Partial", "Received"],
    "side_states": ["Cancelled", "On Hold"]
  },
  "inspection_options": [],
  "project_ref_options": [],
  "unit_options": []
}
```

If a field doesn't exist in the user's registry, leave it blank and skip it on
create/update — do not force a value into a field they don't have.

---

## OPERATION: CREATE — PDF → Notion

### Step 1 — Extract from PDF
Read all pages. Extract:

**Header (shared across all line items):**
- PO Number, PO Date → YYYY-MM-DD
- Supplier Name + GSTIN (as Supplier Code)
- Payment Terms, Delivery Terms
- GST % (if split as SGST+CGST, sum them, e.g. 9+9=18)
- Nett PO Value (grand total incl. tax)
- Required Delivery Date if stated

**Per line item:**
- Item Description, Item Code, Qty, Unit, Size Spec, Make, MOC Specification
- Weight (if stated), Basic Rate, Line Value

### Step 2 — Ask user (MANDATORY, do not skip)
Present this block and wait for reply before touching Notion:

---
**Before I log this PO, please confirm:**

1. **Indent Number** — internal requisition reference (if used)
2. **Indent Date** — (DD-MM-YYYY)
3. **Project/Client reference** — which project or client does this belong to?
   (Use the session's configured project reference list if one exists; otherwise
   ask the user to type it.)
4. **Inspection** — per the configured options *(or ask if no default exists)*
5. **Required Delivery Date** *(if not on PO — DD-MM-YYYY)*
---

Wait for reply. If user says "skip" for any field, omit it.

### Step 3 — Build Title
Use the session's configured `title_pattern`. If none was set, ask once, then reuse.

### Step 4 — Create all rows in one API call

Map extracted + user-confirmed values onto the session's `field_mapping`. Send all
line items in a single `notion-create-pages` batch call, not a loop.

**Rules:**
- Respect each field's configured type (text vs number vs date) — do not assume
  `Quantity` is numeric; many registries store it as text to preserve formatting.
- Omit optional fields (e.g. discount) if not stated on the PO and not required by
  the user's schema.
- Use Notion's expanded date format for any date property:
  `"date:{Property}:start": "YYYY-MM-DD"`, `"date:{Property}:is_datetime": 0`.

### Step 5 — Confirm with summary table

| # | Item Description | Qty | Unit | Line Value |
|---|-----------------|-----|------|-------------|
| 1 | ... | ... | ... | ... |

> ✅ **{N} line items from {PO Number} logged. Status: {initial status}**
> 👉 [Verify in PO Register →]({configured database URL})

---

## OPERATION: CREATE — Manual Entry (no PDF)

If user provides details verbally or in text:
1. Collect all header fields interactively
2. Ask for line items one by one or in a table
3. Follow Steps 3–5 above

---

## OPERATION: READ — Query & Search

### Search by PO Number
`notion-search` → query = PO Number string
Show matching rows: PO Number, Supplier, Status, Nett Value, Items.

### Query open / overdue POs
Use `notion-query-database-view` on the configured database, then filter client-side
using the session's configured status lifecycle:
- **Open**: status = the configured "open" state
- **Overdue**: status = "open" AND Required Delivery Date < today
- **Partial**: status = the configured "partial" state, if one exists
- **Cancelled/On Hold**: status = a configured side-state

### Query by supplier
Search query = supplier name → group results by status.

### Query by project
Filter rows where the configured `project_ref` field matches.

### Dashboard summary (when user asks "show dashboard" or "pipeline")
Query all rows. Compute and display, using the session's actual status labels rather
than assuming a fixed five-state model:
```
📊 PO Register Dashboard — {today}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Total POs logged:   {N}
{status label}:        {n} (₹ {sum})
{status label}:        {n} (₹ {sum})
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Overdue (past RDD): {n} POs
```
List overdue POs: PO Number · Supplier · RDD · Days overdue

---

## OPERATION: UPDATE — Status Changes

### Mark Material Received (full)
1. `notion-search` query = PO Number → get page IDs for all line items
2. `notion-update-page` each row:
   - status field = configured "received" state
   - material received date = today
3. Confirm: ✅ PO {N} marked Received.

### Mark Partial Receipt
Same as above but status = configured "partial" state. Ask user which line items
received if partial.

### Cancel a PO
- status = configured "cancelled" state
- Ask user to confirm: "Cancel PO {N}? This cannot be undone in the register."

### Put On Hold / Reopen
- status = configured hold state, or back to the configured "open" state

### Update Required Delivery Date
- update the configured date field with the new date

### Update Inspection type
- update the configured inspection field with one of the configured options

---

## OPERATION: SOFT-DELETE / ARCHIVE

Notion does not support hard delete via MCP. Instead:
- Set status to the configured "cancelled" state
- Prepend `[VOID] ` to the title field
- Tell user: "Archived as Cancelled. To permanently delete, open Notion and delete
  the page manually."

---

## Edge Case Rules

| Situation | Handling |
|-----------|----------|
| GST split as two taxes (e.g. SGST+CGST) | Sum both into one total % |
| Discount column empty | Omit the discount field entirely |
| Weight embedded in remarks text | Parse the number out (e.g. "Wt: 1.7 Kg" → 1.7) |
| Multi-page PDF | Read all pages before building the item list |
| Unit naming variants | Normalize obvious variants (e.g. "Nos." → "Nos") to whatever the user's configured unit values are |
| Nett PO Value | Grand total after all taxes (not the pre-tax subtotal) |
| Line Value | Pre-tax amount for that specific line |
| Supplier Code missing | Fall back to Supplier GSTIN if available |
| Inspection not stated | Ask the user; do not assume a default without one configured |
| Required Delivery Date not on PO | Ask the user |
| Project/client reference not stated | Ask the user — never guess or reuse a value from a different PO |

---

## Quick Reference: Notion API Patterns

### Date fields — always use expanded format
```
"date:{Property Name}:start": "2026-05-13"
"date:{Property Name}:is_datetime": 0
```

### Number vs Text
Respect whatever type the user confirmed during Step 0 mapping — do not assume a
field is numeric or text-based without checking the configured schema. Currency and
weight fields are typically numeric; quantity and reference-number fields are often
stored as text to preserve formatting (leading zeros, decimal display, etc.).

---

## Proactive Suggestions (offer these when relevant)

1. **Overdue Alert** — After logging a PO, offer: "Want me to check for any overdue POs?"
2. **Supplier Spend Report** — "Would you like a spend summary by supplier?"
3. **Project-wise PO Pipeline** — "I can show all open POs grouped by project."
4. **Partial → Received Workflow** — When marking partial, ask: "Do you want to set a follow-up reminder for the remaining items?"
5. **Batch Status Update** — "Mark all received POs from {supplier} in one go?"

---

## Error Handling

| Condition | Action |
|---|---|
| No registry configured yet | Run Step 0 setup before anything else — do not proceed |
| PDF unreadable/blank | Ask user to re-upload or provide details manually |
| Notion fetch/write fails (network/auth) | Show warning; ask user to retry or check Notion connection |
| Database returns 0 rows on dashboard query | Warn: check filters or registry population |
| Status value not in configured lifecycle | Ask the user to confirm the correct status rather than guessing the closest match |
| Field referenced has no mapping configured | Skip it and tell the user it was skipped, rather than silently omitting it |

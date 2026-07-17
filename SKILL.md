---
name: influencer-order-creation
description: >
  Reads new influencer order rows from the "Influencer orders - Final" tab in the TuCo Kids
  Shopify tracking sheet, creates LIVE Shopify orders directly (no manual draft-review step),
  and writes the resulting order ID back into column N ("Actual Shopify order ID") for every
  row belonging to that order. Maintains a persisted processed_orders.json ledger (independent
  of column N) so a customer can never get a duplicate order even if a previous run's sheet
  write failed after Shopify already confirmed the order — safe to re-run any time. Also has a
  separate `track` mode that checks orders placed in the last 48 hours for fulfillment tracking
  info (AWB/tracking number, tracking link, courier) and backfills columns O/P/Q once a courier
  has picked up the order. Use when the user says "create influencer orders", "run influencer
  order creation", "process influencer orders", "push new influencer orders to Shopify",
  "/influencer-order-creation", "check tracking for influencer orders", "capture AWB numbers",
  or says new rows have been added to the Influencer orders - Final sheet.
---

# Influencer Order Creation

Turns rows in the "Influencer orders - Final" sheet into live Shopify orders on the Tuco
Kids store (tucokids.com, INR), via the Shopify MCP's `graphql_mutation`/`graphql_query`
tools. No draft-order review step — each order is created and completed in the same run.

---

## Step 0 — Load Config

```bash
cat ~/.claude/skills/influencer-order-creation/config.json
```

This has `creds_path` (Google service account key), `sheet_id`, `tab` name, the fixed column
layout (A–Q: order details, N = order ID, O = Tracking Link, P = AWB Number, Q = Courier
Partner), a `variant_cache` (Shopify Product ID → default Variant ID) built up over past runs,
and `tracking_check_window_hours` (default 48 — see the Tracking Check mode below). If the
file is missing, stop and tell the user — this skill was scaffolded with a known-good config
on 2026-07-13 and shouldn't normally be absent.

**Two modes, chosen by the argument passed:**
- No argument (default) → **Order Creation** (Steps 1–8 below): process pending rows into
  live orders.
- `track` → **Tracking Check** (see the dedicated section near the end): backfill AWB/tracking
  info for recently-created orders that have since been fulfilled. Skip straight there if
  that's what was asked for — the two modes don't share steps 1–8.

---

## Step 1 — Fetch All Rows

```python
import warnings; warnings.filterwarnings('ignore')
from google.oauth2.service_account import Credentials
from googleapiclient.discovery import build
import json

with open('/Users/anshuman788/.claude/skills/influencer-order-creation/config.json') as f:
    cfg = json.load(f)

creds = Credentials.from_service_account_file(
    cfg['creds_path'], scopes=['https://www.googleapis.com/auth/spreadsheets']
)
svc = build('sheets', 'v4', credentials=creds)

res = svc.spreadsheets().values().get(
    spreadsheetId=cfg['sheet_id'], range=f"'{cfg['tab']}'!A2:Q5000"
).execute()
rows = res.get('values', [])
```

Each row (0-indexed in `rows`, sheet row = `i + 2`):
`[Order Number, Order Type, Name, Phone, Email, Address Line, City, State, Pincode,
Shopify PID, Kit/Product Name, Quantity, Discount %, Actual Shopify order ID, Tracking Link,
AWB Number, Courier Partner]`

Pad short rows to 17 columns before indexing (trailing blanks are often omitted by the
Sheets API). Order Creation mode only reads/writes columns A–N; columns O–Q are Tracking
Check mode's territory.

---

## Step 2 — Group Into Orders, Find What's New

Group rows by Order Number (column A), preserving their sheet row numbers. An order is
**pending** (needs creation) if column N (index 13) is blank on its first row. If some rows
of the same order have an ID and others don't, treat the order as already processed but
flag this specific case to the user as an anomaly worth checking manually — don't touch it.

If there are zero pending orders, tell the user "No new orders to process" and stop.

---

## Step 2.5 — Cross-Check the Processed-Orders Ledger (the anti-duplicate safeguard)

Column N being blank is the primary "needs creation" signal, but it can lie if a previous
run created the Shopify order successfully and then failed to write column N back (a
dropped connection, a Sheets API hiccup, etc.). To make sure that failure mode can never
result in the same customer getting two live orders, cross-check a second, independent
record before creating anything:

```python
import os, json
ledger_path = os.path.expanduser('~/.claude/skills/influencer-order-creation/processed_orders.json')
ledger = json.load(open(ledger_path)) if os.path.exists(ledger_path) else {}
```

`ledger` maps Order Number → `{"shopify_order_name": "TK302xxx", "created_at": "<iso ts>"}`
for every order this skill has ever successfully completed, going back to its first run.

For each order you determined "pending" in Step 2 (blank column N):
- **If its Order Number is already in `ledger`:** do NOT create it again. This is exactly
  the failure mode above — Shopify already has this order. Instead, just backfill column N
  from the ledger's `shopify_order_name` for all of that order's rows, and mention this
  self-heal explicitly in the run summary ("column N was blank for `<Order Number>` but it
  was already created as `<name>` per the ledger — backfilled, not recreated").
- **If it's not in the ledger:** proceed to actually create it in Step 3–5.

This ledger is the source of truth for "has this customer's order already been placed" —
column N is a convenience mirror of it inside the sheet.

---

## Step 3 — Resolve Product Variant IDs (cache-first)

For every distinct Shopify PID across the pending orders not already in
`cfg['variant_cache']`, resolve its default variant via:

```graphql
query {
  p0: product(id: "gid://shopify/Product/<PID>") { title variants(first: 5) { nodes { id price } } }
  p1: product(id: "gid://shopify/Product/<PID2>") { title variants(first: 5) { nodes { id price } } }
}
```

(Batch as many PIDs as needed into one query using aliases `p0, p1, ...`, same way as
looking up multiple products at once.) Take `variants.nodes[0].id` as the variant to use. If
a product has more than one variant, don't guess — flag that order as failed with "product
`<title>` has multiple variants, need which one" and skip it rather than picking wrong.

After resolving, update and persist the cache:

```python
cfg['variant_cache'][pid] = variant_id_numeric  # just the numeric id, no gid:// prefix
with open('/Users/anshuman788/.claude/skills/influencer-order-creation/config.json', 'w') as f:
    json.dump(cfg, f, indent=2)
```

If a `product` query returns null (PID doesn't exist), fail that order with a clear reason
and move on — don't block the rest of the batch.

---

## Step 4 — Build Each Order's Input

For each pending order:

**Name split:** first word → `firstName`, remainder → `lastName` (if only one word, use it
for both).

**Phone (E.164, always +91 for this store):**
```python
def fmt_phone(p):
    p = str(p).strip().replace(' ', '').replace('-', '')
    if p.startswith('+'):
        return p
    if p.startswith('91') and len(p) == 12:
        return '+' + p
    return '+91' + p[-10:]
```

**State → Shopify province code for India.** If the State cell is already a 2-letter code,
uppercase and use it directly. Otherwise look up (case-insensitive, trim):

```python
STATE_MAP = {
    "andhra pradesh": "AP", "arunachal pradesh": "AR", "assam": "AS", "bihar": "BR",
    "chhattisgarh": "CT", "goa": "GA", "gujarat": "GJ", "haryana": "HR",
    "himachal pradesh": "HP", "jharkhand": "JH", "karnataka": "KA", "kerala": "KL",
    "madhya pradesh": "MP", "mp": "MP", "maharashtra": "MH", "manipur": "MN",
    "meghalaya": "ML", "mizoram": "MZ", "nagaland": "NL", "odisha": "OR",
    "punjab": "PB", "rajasthan": "RJ", "rajsthan": "RJ", "sikkim": "SK",
    "tamil nadu": "TN", "telangana": "TG", "tripura": "TR", "uttar pradesh": "UP",
    "uttarakhand": "UT", "west bengal": "WB",
    "andaman and nicobar islands": "AN", "chandigarh": "CH",
    "dadra and nagar haveli and daman and diu": "DN", "delhi": "DL", "new delhi": "DL",
    "jammu and kashmir": "JK", "ladakh": "LA", "lakshadweep": "LD", "puducherry": "PY",
}
```
If it doesn't match anything, leave `provinceCode` unset rather than guessing, and note it
in the run summary so the user can fix the sheet cell.

**Quantity:** blank → `1`. **Discount %:** blank → `100`.

**Tag / order type:** from column B (`Barter` → `Influencer-Barter`, `Paid` →
`Influencer-Paid`). If blank, infer from the Order Number prefix (`BARTER`/`PAID`); if
neither, use `Influencer-Unspecified` and flag it in the summary.

**No shipping line** — this has been the standing preference since the first batch
(2026-07-13); don't add `shippingLine` unless the user asks for one this run.

Build the `DraftOrderInput` (this exact shape is validated and known-good — don't
re-derive it via `graphql_schema`):

```json
{
  "email": "...",
  "phone": "+91...",
  "note": "Influencer order sheet ref: <Order Number> | Name: <Name>",
  "tags": ["<Influencer-Barter|Influencer-Paid|Influencer-Unspecified>", "<Order Number>"],
  "shippingAddress": {
    "firstName": "...", "lastName": "...", "address1": "...", "city": "...",
    "zip": "...", "countryCode": "IN", "phone": "+91...", "provinceCode": "XX"
  },
  "billingAddress": { "...same as shippingAddress..." },
  "lineItems": [
    { "variantId": "gid://shopify/ProductVariant/<id>", "quantity": <qty> }
  ],
  "appliedDiscount": {
    "value": <discount_pct>,
    "valueType": "PERCENTAGE",
    "title": "Influencer Collab - <discount_pct>% Off",
    "description": "<tag_type>"
  }
}
```

---

## Step 5 — Create + Complete Each Order (batches of 5, no pause for review)

For each batch of up to 5 pending orders, run ONE `graphql_mutation` call that creates all 5
drafts via aliases:

```graphql
mutation createBatch($input0: DraftOrderInput!, $input1: DraftOrderInput!, ...) {
  d0: draftOrderCreate(input: $input0) {
    draftOrder { id name totalPriceSet { presentmentMoney { amount currencyCode } } tags }
    userErrors { field message }
  }
  d1: draftOrderCreate(input: $input1) { ... same shape ... }
  ...
}
```

Then, for every `dN` that came back with no `userErrors`, immediately run ONE
`graphql_mutation` call that completes that same batch:

```graphql
mutation completeBatch($id0: ID!, $id1: ID!, ...) {
  c0: draftOrderComplete(id: $id0) {
    draftOrder { name order { id name displayFinancialStatus } }
    userErrors { field message }
  }
  c1: draftOrderComplete(id: $id1) { ... same shape ... }
  ...
}
```

If `draftOrderCreate` returns `userErrors` for a given alias (e.g. bad phone, invalid
address), record that order as **failed** with the error message — do not attempt to
complete it, and do not write anything to column N for it (so it stays "pending" and gets
retried next run once the sheet is fixed). Same if `draftOrderComplete` errors.

Take `order.name` (e.g. `TK302xxx`) from each successfully completed order — this is what
gets written back to the sheet, not the internal `gid://shopify/Order/...` id.

**Immediately** (before the Step 6 sheet write, for each order as soon as it completes) append
it to the ledger and persist to disk:

```python
from datetime import datetime, timezone
ledger[order_number] = {
    "shopify_order_name": order_name,
    "created_at": datetime.now(timezone.utc).isoformat()
}
with open(ledger_path, 'w') as f:
    json.dump(ledger, f, indent=2)
```

This order is why the ledger is the safeguard, not the sheet: the ledger write happens the
moment Shopify confirms the order exists, before anything that could fail on the Sheets side.

---

## Step 6 — Write Order IDs Back to Column N (after each batch)

After each batch completes, write the resulting order name into column N for **every row**
belonging to that order (an order can span multiple line-item rows). Do this incrementally
per batch, not only at the very end, so progress isn't lost if a later batch fails.

```python
updates = []  # list of {"range": f"'{tab}'!N{row}", "values": [[order_name]]}
for row_num, order_name in rows_to_update:
    updates.append({"range": f"'{cfg['tab']}'!N{row_num}", "values": [[order_name]]})

svc.spreadsheets().values().batchUpdate(
    spreadsheetId=cfg['sheet_id'],
    body={"valueInputOption": "RAW", "data": updates}
).execute()
```

If this write fails after Step 5 already succeeded, that's fine and expected to be
recoverable — the ledger already has the record, so next run's Step 2.5 will backfill
column N instead of creating a duplicate.

---

## Step 7 — Save the Run Checkpoint

```python
last_run_path = os.path.expanduser('~/.claude/skills/influencer-order-creation/last_run.json')
json.dump({
    "run_at": datetime.now(timezone.utc).isoformat(),
    "orders_created": [...],   # Order Number -> shopify_order_name, this run only
    "orders_failed": [...],    # Order Number -> reason, this run only
    "orders_backfilled": [...],# Order Number -> shopify_order_name, self-healed from ledger this run
    "total_orders_in_ledger": len(ledger),
}, open(last_run_path, 'w'), indent=2)
```

This is a per-run audit trail (what happened last time, and how big the cumulative ledger
is) — the `processed_orders.json` ledger from Step 2.5 is what actually prevents duplicates;
this file is for humans/Claude to see run history at a glance.

---

## Step 8 — Summarize

Tell the user, per run:
- N orders created (list Order Number → Shopify order name, e.g. `INF_BARTER_INFLUENCER_1400 → TK302760`)
- N orders backfilled (already existed in the ledger, column N was just blank — self-healed, not recreated)
- N orders failed, with the specific reason for each (bad phone, unmapped state, multi-variant product, missing PID, etc.) — these rows stay blank in column N and will be retried automatically next run once fixed
- Any new PID→variant mappings added to the cache this run
- Total ₹ value processed (post-discount — should be ₹0 for 100%-off barter orders, flag anything that isn't since that's the common failure mode to catch)

---

## Tracking Check Mode (`track`)

Separate from Order Creation — run this when the user asks to check/capture tracking, AWB
numbers, or courier info for influencer orders. It does **not** create any orders; it only
reads Shopify and backfills columns O/P/Q for orders already in the ledger.

**Why a 48-hour window, and why separate from Order Creation:** fulfillment (courier pickup +
AWB assignment) happens some time after the order is created, not at the same moment, so
checking for tracking info during Order Creation would almost always find nothing. Rather
than re-checking the entire historical ledger forever (wasted API calls, and older orders are
presumably already handled through the older sheet/process), only orders created within
`cfg['tracking_check_window_hours']` (default 48) that don't yet have tracking captured are
worth checking on each run. This was an explicit 2026-07-17 decision — don't widen the window
without asking.

### TC-Step 1 — Find Candidates

```python
import json, os
from datetime import datetime, timezone, timedelta

with open(os.path.expanduser('~/.claude/skills/influencer-order-creation/config.json')) as f:
    cfg = json.load(f)
ledger_path = os.path.expanduser('~/.claude/skills/influencer-order-creation/processed_orders.json')
ledger = json.load(open(ledger_path)) if os.path.exists(ledger_path) else {}

cutoff = datetime.now(timezone.utc) - timedelta(hours=cfg.get('tracking_check_window_hours', 48))
candidates = {
    onum: entry['shopify_order_name']
    for onum, entry in ledger.items()
    if not entry.get('tracking_captured') and datetime.fromisoformat(entry['created_at']) >= cutoff
}
```

If `candidates` is empty, tell the user "No orders in the last {window}h are pending a
tracking check" and stop (this is the normal case for most runs, not an error).

### TC-Step 2 — Batch-Query Shopify for Fulfillment/Tracking Info

Query by order `name` (not by GID — the ledger only stores the human-readable order name),
batching up to ~20 orders per call with an `OR`-joined query string:

```graphql
query {
  orders(first: 20, query: "name:TK303630 OR name:TK303631 OR ...") {
    nodes {
      id
      name
      displayFulfillmentStatus
      fulfillments(first: 3) {
        status
        trackingInfo { number url company }
      }
    }
  }
}
```

For each returned order with a non-empty `fulfillments` list, take the first fulfillment's
`trackingInfo[0]` — `number` → AWB Number (col P), `url` → Tracking Link (col O), `company` →
Courier Partner (col Q). Orders still `UNFULFILLED` (empty `fulfillments`) are simply left
alone — they'll be picked up again on the next Tracking Check run as long as they're still
inside the window.

### TC-Step 3 — Update Ledger + Sheet

For every order with tracking now available:

```python
now = datetime.now(timezone.utc).isoformat()
ledger[onum].update({
    "tracking_captured": True,
    "tracking_number": info["number"],
    "tracking_url": info["url"],
    "courier": info["company"],
    "tracking_captured_at": now,
})
json.dump(ledger, open(ledger_path, 'w'), indent=2)
```

Then look up that order's row numbers in the sheet (fetch column A fresh and match on Order
Number — don't assume rows haven't shifted since Order Creation last ran) and write, for
every row belonging to that order: `O{row}` = tracking url, `P{row}` = tracking number,
`Q{row}` = courier. Use a single batched `values.batchUpdate` call across all rows/orders in
this pass.

### TC-Step 4 — Summarize

Tell the user:
- N orders now tracked (Order Number → courier + AWB number)
- N orders still unfulfilled / not yet checked out (will be retried next run, as long as
  they're still within the window)
- N orders that fell outside the window this run without ever getting tracking captured —
  flag these explicitly, since it likely means fulfillment is stuck or the window needs
  revisiting

---

## Arguments

- `/influencer-order-creation` — process all pending (new) rows in "Influencer orders - Final"
  (Order Creation mode)
- `/influencer-order-creation track` — check the last 48h of created orders for fulfillment
  tracking info and backfill columns O/P/Q (Tracking Check mode)
- No dry-run / draft mode for order creation — per the 2026-07-13 decision, this always
  creates live orders. If the user wants a review step for a specific run, they'll say so
  explicitly; don't ask by default each time.

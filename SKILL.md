---
name: reschedule-engine
slug: reschedule-engine
description: Use when processing a customer rescheduling Excel, classifying pull-in / push-out / cancel, joining F&O order lines and purchase order lines, calculating Bossard feedback ETD/ETA, writing Rescheduling.xlsx, or filling a Reply sheet. Don't use when posting into F&O, changing prices, or writing a general sales report.
---

# Reschedule engine

Paperclip self-contained bundle. Spec v1.5 (11 September 2026). Customer templates, F&O fields, and worked packets are in this file. Do not look for sibling `templates.md`, `fo-fields.md`, or `examples.md`.

On a live task, use the customer workbook and F&O extracts **attached to the issue**. Gold packet names below identify layouts only; they are not a local `Input/` / `Output/` path.

Scope: **only Pull-in rows enter the engine and `Rescheduling.xlsx`**. Push-out is no action (remain date). Cancel is a user note only.

## Pipeline

1. Consolidate request → SO lines (one working row per request line).
2. Classify Action → type.
3. Drop Push-out and Cancel from the engine (they still appear on the customer Reply).
4. For Pull-in only: inventory, then supplier date on Purchase order lines (see **Supplier date**), then delivery zone.
5. Calculate recommended delivery date `I` and fill Reply / `Rescheduling.xlsx` as before.
6. Human review when a flag applies.
7. Write `Rescheduling.xlsx` (Pull-in only) then write back ETA / remarks / tracking on the customer Reply.

## 1. Consolidate

**Inputs**

- Customer request workbook (one of the templates in **Customer templates** below).
- F&O **Order lines** extract.
- F&O **Purchase order lines** extract.

There is no standalone inventory file. Inventory = SO reservation status + open PO remainder.

**SO join key (in order)** — attach at most one sales-order line

1. Customer PO# + External PN + quantity + need or due date.
2. Then Bossard item number.

Keep **one working row per request line**. Do not merge several lines that share the same PN on one PO. If the customer PO exists in F&O but this PN is not on it, do not steal another PO's SO line. Unmatched rows stay in the working set with remarks `Unmatched SO/PO line`.

**Bossard item for supply** (does not attach an SO line)

1. Matched SO `Item number`, if present.
2. Else request customer PN (`External` / aliases) on Order lines → unique `Item number`. If several items match, prefer the Order line whose `Customer requisition` / `Customer reference` equals the request PO.
3. Else request customer PO on Order lines, only if that PO maps to **one** Bossard item.
4. Else request `bossard_item` / digits extracted from the PN.

If several distinct Bossard items remain, remarks `Multiple Bossard items` and do not guess. Using another Order line only to read `Item number` is allowed; attaching that SO line is not.

Attach SO/PO fields listed in **F&O fields**. Treat `0` and `#N/A` in UNIQUE dumps as empty, not codes.

## 2. Classify

Map the customer Action / Message / Signal onto a type. Unknown labels: ask, do not invent a fourth engine type.

| Customer labels (spec + gold files) | Type | Route |
| --- | --- | --- |
| Expedite, Rescheduling In, Reschedule In, Pull-In, reschedule in | Pull-in | Engine → date → `Rescheduling.xlsx` → review |
| Delay, Rescheduling Out, Reschedule Out, Res-Out, reschedule out | Push-out | No calculation. Remain date. Not a result row |
| Cancel, Partial Cancel, Cancel process | Cancel | User note only. Not a result row |
| Remain, Unconfirmed, Unconfirmed PO, Delivery is on Time, VDP forecast, confirmed, Please confirm the order | Remain | Same route as Push-out |

Buyer remarks such as `Need immediate shipment` do **not** override a Remain/on-time Message into Pull-in. Type comes from the Action/Message/Signal column.

## 3. Inventory, ETA, zone (Pull-in only)

| Check | Rule |
| --- | --- |
| Inventory | Available: `Reserved ordered`, `Reserved physical`. Not available: `On order`, `Long-term`, `Line partial` / `Line partially reserved`, `Inbound` / `Inbound processing` (+ partial), `Quality` / `Quality processing` (+ partial) |
| Supplier ETA | `Not shipped` = full open qty. `Shipped partially` = remainder. `Shipped in full` = no open supply. Approval `In external review` = keep ETA and flag it |
| Delivery zone | Read SO **Mode of delivery**, set transport days from **Mode of delivery — transport days** below |
| Availability `O` | If reservation is `Reserved ordered` / `Reserved physical`: stock is on hand, `O` = packet as-of date (do **not** add 6 working days). Otherwise: Purchase order line `Confirmed receipt date` + **6 working days** (Mon–Fri unless the task gives a calendar). Same +6 whether the PO line was found by Bossard item or by vendor PO number |

Working-row flags to carry forward: inventory available Y/N, resolved Bossard item, ETA date used, shipped status, mode of delivery, transport days, `O`.

If several PO receipts could supply the line, do not pick silently — remarks `Multiple availability date` and send to review.

### Supplier date

Same formula for the new item path and the old vendor-PO# path. Do not fill the raw `Confirmed receipt date`.

1. Take the request customer PN, then customer PO (the customer codes on the reschedule file).
2. Resolve Bossard `Item number` from **Order lines** as above.
3. Look up **Purchase order lines** by that Bossard item. Open supply first (`Not shipped` / remainder).
4. If that misses, look up **Purchase order lines** by the request PO number (vendor PO# is usually not the customer PO).
5. Vendor delivery date = `Confirmed receipt date` + **6 working days** + transport (929 = +3, 998 = +5, other known = 0). Then fill `I` as in **Calculate `I`**.

## 4. Calculate `I` (Pull-in / Expedite only)

Column `I` is the field being calculated (Bossard feedback delivery date). `G` / `H` / `I` / `M` / `N` / `O` are **engine fields**, not Excel column letters on the customer sheet.

| Symbol | Meaning |
| --- | --- |
| `H` | New request delivery date (need / reschedule / new dock) |
| `G` | Current committed delivery date (SO confirmed receipt, else request current date) |
| `M` | Requested receipt date on SO |
| `N` | Confirmed ship date on SO |
| `O` | Availability from step 3 |

1. If `G` does not equal `M`: remarks `Unmatching committed date`. Still calculate `I`.
2. Get the supplier date from **Supplier date** (Bossard item on Order lines first, then Purchase order lines, then + **6 working days** + transport). Never use ship date `N`.
3. **In stock** (`Reserved ordered` / `Reserved physical`): `I` = `H` − **3 working days**. Do not go before as-of.
4. **Has supplier PO date** (not in stock): fill `I` with that vendor-delivery + 6 date. If it is after `H`, `I` = `G` (cannot pull in). Write `I` on the same Reply / `Rescheduling.xlsx` columns as before.
5. **Unmatched** (no SO and no PO date): still calculate `I` = `H` − 3 working days and flag review. Never invent a match.

Column `F` on the result file is always `Expedite` / `Rescheduling In` (the Pull-in label). Column `J` is remarks when a flag applies.

## 5. Human review

Reviewer accepts or overwrites `I` and `J`. **Mandatory** when remarks is any of:

- `Unmatched SO/PO line`
- `Multiple availability date`
- `Multiple Bossard items`
- `Unmatching committed date`
- approval = `In external review`

On Paperclip: comment the trace, keep those rows out of the accepted result until the requester (or a review comment) accepts `I`/`J`. Do not mark the whole packet done while mandatory-review rows are open.

## 6. Outputs

### 6.1 `Rescheduling.xlsx` (internal)

Format `.xlsx`. Existing Rescheduling header. Date format **DD-MMM-YY**. Push-out and Cancel rows are **not present**.

| Col | Field | Source |
| --- | --- | --- |
| A | Order created date | SO created date or request date |
| B | Customer PO# | Request |
| C | Bossard item number | Matched SO `Item number`, else Order-lines mapping from the request PN/PO. `N/A` if unmapped |
| D | External PN# | Request |
| E | Order quantity | Line quantity |
| F | Message | Expedite / Rescheduling In only |
| G | Current committed delivery date | SO confirmed date |
| H | New request delivery date | Need date |
| I | Bossard feedback delivery date | Calculate + review |
| J | Remarks | Always filled when a flag applies |

What is not in the result file: Push-out / Delay (note: line received and skipped); Cancel / Partial Cancel (note: cancel label found, not rescheduled).

### 6.2 Customer Reply workbook

Gold packets always keep an `Original` sheet (inbound, never mutated) and a `Reply` sheet. Write a **new dest file**. Refuse if source and dest resolve to the same path.

After `Rescheduling.xlsx` exists, join back on **Customer PO# + External PN + quantity** (optional tracking number if present).

Overwrite **only** that template's reply columns (see **Customer templates**). Spec canonical names:

| Request column | Taken from | Write rule |
| --- | --- | --- |
| ETA / ETD / Confirm Date / Supplier confirmation ETA | Column `I` | Replace with accepted feedback date |
| REMARKS / comment | Column `J` | Replace with engine / reviewer remarks. If `J` is empty, leave remarks empty or a short accepted-date note |
| TRACKING | No dedicated result column | Copy tracking from the original request line if filled. Else PO / F&O tracking if available. Else blank |

Do not insert new request rows. Do not change Action, dates, qty, or price.

**Push-out and Cancel on the Reply sheet:** spec says leave those request lines unchanged. Gold Output still copies every inbound row onto `Reply` and often fills the date/comment columns with remain date, shipped + invoice, hold, or TBA so the customer gets an answer. Follow the gold template for that customer: copy all rows to `Reply`; fill reply columns for Pull-in from `I`/`J`; for Push-out/Cancel/Remain fill only if that customer's gold style does (remain committed date, shipped note, hold). Never add them to `Rescheduling.xlsx`.

## 7. Working days and dates

- Working days = Mon–Fri. Do not invent public holidays unless the task attaches a calendar.
- `O` = Purchase order line `Confirmed receipt date` + 6 working days when inbound (item path or vendor-PO# path); in-stock uses packet as-of (skip Sat/Sun). Do not write the raw vendor date.
- Result file dates: `DD-MMM-YY` (example `18-Sep-26`).
- Customer Reply dates: keep that workbook's existing cell date format. Do not stringify to text unless the gold column is already text (`shipped on 10 Sep`, `TBA`, split-qty notes).

## 8. Stop conditions

- Missing F&O extract → blocked, name the missing file.
- New template with no mapping in **Customer templates** → comment headers you saw; ask which columns are PO, PN, qty, need date, action, reply.
- Mandatory review flags → `in_review`, not `done`.
- Partial qty / pack-size / shipped-split answers are allowed on the customer Reply as text (see Datalogic gold); they still do not create extra `Rescheduling.xlsx` rows unless the request itself was split lines.

---

# F&O fields and select values

Source extracts in gold packets: `Order lines_20260805.xlsx`, `Purchase order lines_20260805.xlsx`. Use **Sheet1** only. Sheet2 is a UNIQUE dump (`#N/A` / `0` = empty).

## Order lines (SO)

| Field | Used for |
| --- | --- |
| Customer account / Customer name | Identity |
| Sales order / Internal line | SO key |
| Customer requisition | Customer PO# on the SO. Also used to resolve Bossard item when the request PN is absent or maps to several items |
| Customer reference | Alternate customer PO# on the SO |
| External | Customer PN on the SO. First key from the reschedule file to Bossard `Item number` |
| Item number | Bossard item. Look up Purchase order lines by this **before** vendor PO# |
| Quantity / Deliver remainder | Qty |
| Line status | Must be Open order in current extracts |
| Order complete | Yes / No |
| Reservation status | Inventory flag |
| Warehouse | PEN |
| Requested receipt date | `M` |
| Confirmed ship date | `N` |
| Confirmed receipt date | Committed delivery `G` and part of `O` |
| Mode of delivery | Transport days |
| Created date and time | Result column A |

## Purchase order lines (PO)

| Field | Used for |
| --- | --- |
| Purchase order / Line number | Vendor PO key. Fallback only: match the **request** PO number here if the Bossard-item lookup missed (vendor PO# is usually not the customer PO) |
| Vendor account / Name | Supplier |
| Item number | Bossard item — primary lookup key from Order lines |
| Quantity / Deliver remainder | Open supply |
| Shipped | Not shipped / Shipped partially / Shipped in full |
| Warehouse | PEN |
| Confirmed shipping date | Supplier ship ETA |
| Confirmed receipt date | Supplier delivery date. `O` / `I` use **this date + 6 working days** (+ transport). Same formula for the Bossard-item path and the vendor-PO# path |
| Approval status | Approved / Confirmed / In external review |
| Line status | Open order |

## Inventory flag

Available:

- Reserved ordered
- Reserved physical

Not available:

- On order
- Long-term
- Line partial / Line partially reserved
- Inbound / Inbound processing / Inbound processing - partial
- Quality / Quality processing / Quality processing - partial

`None` on reservation → treat as not available and flag if the line is Pull-in.

## Shipped / approval

- Not shipped → full open qty.
- Shipped partially → remainder (`Deliver remainder`).
- Shipped in full → no open supply.
- Approved or Confirmed → use ETA.
- In external review → keep ETA, remarks flag, mandatory review.

## Mode of delivery — transport days

| Code | Description | Transport days |
| --- | --- | --- |
| 929 | KL Delivery | 3 |
| 946 | Logistics Shipment (Normal) | 0 |
| 948 | Shipment to be passed to sales | 0 |
| 960 | Express Shipment | 0 |
| 961 | Export Ex-Works (Normal) | 0 |
| 987 | Shipment for Monday | 0 |
| 988 | Shipment for Tuesday | 0 |
| 989 | Shipment for Wednesday | 0 |
| 990 | Shipment for Thursday | 0 |
| 991 | Shipment for Friday | 0 |
| 992 | Logistics Shipment (Express) | 0 |
| 997 | Local Ex-Works | 0 |
| 998 | JB Delivery | 5 |

Unknown code → 0 days and remarks that the code is unconfirmed. Do not invent a new day count.

## Working-row fields after join

Carry at least: request identity, type, resolved Bossard item, `G`, `H`, `M`, `N`, `O`, reservation, shipped, approval, mode, transport days, match status, remarks flags.

---

# Customer templates

Detect the layout from **header names**, not from the filename and not from Excel letters. Map names → roles (PO, PN, qty, need date, committed date, reply). Known layouts below are gold documentation, not a required letter map.

Copy `Original` through unchanged. Put answers on `Reply`. Reply column names differ by customer; map them onto spec ETA / REMARKS / TRACKING.

SO join always: customer PO + external PN + quantity (+ need/due date, then Bossard item). Supplier date is a separate lookup: request PN/PO → Order lines Bossard item → Purchase order lines `Confirmed receipt date` + 6 working days.

## Abinaya / BOSSARD (M) SDN BHD

Files: `Abinaya_15SEP26_BOSSARD (M) SDN BHD_ORI.xls` → `… Reply.xls`. Sheets `Original` (17 cols), `Reply` (19 cols).

| Role | Columns |
| --- | --- |
| External PN | Item Code |
| Customer PO | Order |
| Line | Pos. / Conc |
| Qty | Ordered / Back Order |
| Current committed | Current Del. Date / Conf. Date |
| Need / dock | Needed ETD, New Request Dock Date |
| Action | Message |
| Reply ETA | Bossard's ETD |
| Reply remarks | Bossard's comment |

Message values: `Delivery is on Time`, `Reschedule Out`, `Reschedule In`, `Unconfirmed PO`.

Gold Reply fills Bossard's ETD on on-time, pull-in, push-out, and most unconfirmed lines (often Needed ETD / current−7d). Hold comments used: `PO On Hold - Price no longer valid`, `PO On Hold - Revise PO quantity from 1958pcs to 2000pcs`.

## Datalogic W38

File: `Datalogic- Rescheduled Report - W38'26.xlsx`. 35 columns. Same headers on Original and Reply.

| Role | Columns |
| --- | --- |
| Material / PN | Material, MPN Material, Mnfr Part Number |
| Customer PO | Purchase Order + PO line |
| Qty | PO Qty |
| Current date | Delivery Date |
| Need date | New Delivery Date, Rescheduling dt–GR |
| Action | Message **and** Exception Message |
| Reply ETA | Confirm Date (date **or** free text) |
| Reply remarks | Remark |

Message: mostly `VDP forecast` (Remain — leave Confirm Date blank). Actionable: `pending confirm`, `reschedule in`, `confirmed`, blank + Exception `Reschedule out` / `Cancel process` / `Reschedule in`.

Gold Confirm Date may be a date, `shipped on 10 Sep`, split qty (`12,600pcs - 18/09/2026`, `24/09/2026 - 17k, remaining 3k on 13/10/2026`), or `TBA`. VDP forecast rows stay blank.

## Growri / Honeywell

Files: `Growri_15SEP26_Honeywell_ORI.xlsx` → `… Reply.xlsx`. 22 columns.

| Role | Columns |
| --- | --- |
| PN | ItemCode / MPN |
| Customer PO | PO + PO Line / concat |
| Qty | Order Qty |
| Current | Confirm Del. Date, Cur. Plan Del. Date |
| Need | Need Date |
| Action | Message |
| Reply ETA | Bossard Feedback ETA |
| Reply remarks | Remarks / Awb# / Inv# |

Do not write Vendor Feedback ETD (that is not Bossard). Message: `Delivery is on Time`, `Res-Out`, `Please confirm the order`, `Please share the shipment info.`

Gold: on-time and Res-Out get Bossard Feedback ETA = current plan/confirm date. Confirm-order line may have empty ETA plus a revise-qty remark.

## KMWE

File: `KMWE_May_04052026.xlsx`. Original 10 cols, Reply adds `ETD`.

No Action column. Treat as open-PO confirmation: fill `ETD` where F&O can support a date or a shipped/hold note; leave blank when unmatched and not shipped.

Gold `ETD` mixes dates (`2026-05-06`) and text (`Shipped 29/4/26`, `20/5/26 as goods is in transit`, `pending rev PO`, `19/6/26 as no stock available for pull in`). Supplier Remarks stays empty in gold.

## LimWeiSheng

File: `LimWeiSheng_8SEP26_Reschedule7.xlsx`. Original header ends `Supplier confirmation`; Reply renames last col to `Supplier confirmation ETA`.

| Role | Columns |
| --- | --- |
| PN | PartName |
| Customer PO | Purchase Order + Line |
| Qty | Quantity |
| Current | Dock Date |
| Need | Reschedule Date |
| Action | Action Message (`Pull-In` on every gold line) |
| Reply ETA | Supplier confirmation ETA |

All gold rows are Pull-in. Confirmation ETA is usually a few days before the requested Reschedule Date; if pull-in is impossible it stays on Dock Date.

## Nurnajwa / WW38

Files: `Nurnajwa_17SEP26_WW38 Ori.xlsx` → `… Ori & Reply.xlsx`. Original 18 cols, Reply adds Bossard's ETA + Bossard's comment.

| Role | Columns |
| --- | --- |
| PN | Item / MPN |
| Customer PO | Purchase Order + PO Pos |
| Qty | Order Qty |
| Current | Planned Delivery Date, Confirmed Delivery Date |
| Need | Rescheduled Date |
| Action | Signal Description |
| Reply ETA | Bossard's ETA |
| Reply remarks | Bossard's comment |

Signal: `Reschedule In`, `Reschedule Out`, blank (unconfirmed). `Confirmed` col `Yes`/`No` is inbound — do not overwrite.

Gold push-out often remain/ship: ETA = confirmed or ship date, comment `SHIPPED.INV#…` or `Request PO Copy`. Pull-in gets a calculated ETA. Hold: `PO On Hold - Price no longer valid`. Treat `1970-01-01` confirmed dates as empty.

## BOSSARD M SDN BHD (Benchmark)

File: `BOSSARD M SDN BHD.xlsx`. Headers include `po num`, `Part`, `MPN`, `NEED Date`, `DUE Date`, `Action`, last three columns `ETA`, `REMARKS`, `TRACKING`.

| Role | Columns |
| --- | --- |
| External PN | Part (SO External). MPN is manufacturer code, not Bossard item |
| Customer PO | po num |
| Line | Pos |
| Qty | Quantity |
| Current committed | DUE Date |
| Need | NEED Date |
| Action | Action (`Delay` / `Expedite`) |
| Reply | ETA, REMARKS, TRACKING only |

Join on po num + Part + qty. Do not use filename alone; Abinaya is a different layout even when the supplier name is BOSSARD (M) SDN BHD.

## Canonical spec names

If a new file is literally `BOSSARD M SDN BHD.xlsx` with last three columns ETA, REMARKS, TRACKING: update only those three on matching Pull-in lines after `Rescheduling.xlsx` exists. Map `po num` → PO and `Part` → External PN.

## New template

If headers match none of the above: comment the header row, propose PO / PN / qty / need / action / reply mapping, wait. Do not invent reply headers.

---

# Worked packets

These are gold outcomes, not extra rules. If a gold cell conflicts with the engine rules above, follow the engine rules and note the deviation in the task comment.

## Packet map

| Input | Output | Template |
| --- | --- | --- |
| `Abinaya_15SEP26_BOSSARD (M) SDN BHD_ORI.xls` | `Abinaya_15SEP26_BOSSARD (M) SDN BHD Reply.xls` | Abinaya |
| `Datalogic- Rescheduled Report - W38'26.xlsx` | same name | Datalogic |
| `Growri_15SEP26_Honeywell_ORI.xlsx` | `Growri_15SEP26_Honeywell Reply.xlsx` | Honeywell |
| `KMWE_May_04052026.xlsx` | same name | KMWE |
| `LimWeiSheng_8SEP26_Reschedule7.xlsx` | same name | LimWeiSheng |
| `Nurnajwa_17SEP26_WW38 Ori.xlsx` | `Nurnajwa_17SEP26_WW38 Ori & Reply.xlsx` | Nurnajwa |
| `Order lines_20260805.xlsx` | — | F&O SO |
| `Purchase order lines_20260805.xlsx` | — | F&O PO |

Each customer workbook already contains `Original` + `Reply` in gold pairs. F&O extracts are Input-only and shared. On Paperclip, treat these names as layout identifiers; the live packet is the files attached to the task.

## Classify from gold

- Abinaya 123 data rows: 77 on-time, 24 Reschedule Out, 18 Unconfirmed PO, 4 Reschedule In. Only the 4 In rows belong in `Rescheduling.xlsx`.
- LimWeiSheng 38 data rows: all `Pull-In` → all engine rows.
- Nurnajwa: 7 In, 14 Out, 2 blank/unconfirmed. Engine = 7 In rows.
- Datalogic: 504 `VDP forecast` stay blank on Confirm Date. Fill Confirm Date / Remark on pending confirm, reschedule in, cancel, and exception reschedule-out/shipped lines.
- Growri 6 rows: 3 on-time (ETA = plan date), 1 Res-Out (ETA = confirm date 2026-09-24, need 2027-03-26 ignored by engine), 1 confirm-order (qty remark, ETA empty), 1 shipment-info (ETA = 2026-09-17).

## Date behaviour from gold (illustrative)

LimWeiSheng pull-in accepted early:

- Dock 2026-11-23, request 2026-10-22, confirmation ETA 2026-10-19.

LimWeiSheng pull-in not met (remain dock):

- Dock 2026-09-14, request 2026-09-09, confirmation ETA 2026-09-14.

Nurnajwa pull-in:

- Signal In, rescheduled 2026-09-23, Bossard's ETA 2026-09-22.
- Signal In, rescheduled 2026-09-30, confirmed delivery 2026-10-30, ETA 2026-10-27 (could not fully meet; later than request, still earlier than some confirmed dates).

Nurnajwa push-out shipped:

- Out, rescheduled 2027-01-07, ETA 2026-09-01, comment `SHIPPED.INV#CSI00077357`. Not a `Rescheduling.xlsx` row.

Abinaya hold:

- Message on-time or unconfirmed, Bossard's ETD empty, comment `PO On Hold - Price no longer valid`.

Datalogic split supply:

- `reschedule in` 20000 pcs → Confirm Date `24/09/2026 - 17k, remaining 3k on 13/10/2026`.
- Exception `Reschedule out` already shipped → `shipped on 10 Sep`.

## Join reminder from spec

Do not collapse Abinaya / Nurnajwa / Datalogic lines that repeat the same PN on one PO with different qty or dates. One request line → one Reply row → at most one result row if type is Pull-in.

Do not attach another PO's sales-order line to fill the join. You **may** read that line's `Item number` so Purchase order lines can supply `Confirmed receipt date` + 6 working days.

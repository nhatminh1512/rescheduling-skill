---
name: reschedule-engine
description: Use when processing a customer rescheduling Excel, classifying pull-in / push-out / cancel, joining F&O order lines and purchase order lines, calculating Bossard feedback ETD/ETA, writing Rescheduling.xlsx, or filling a Reply sheet. Don't use when posting into F&O, changing prices, or writing a general sales report.
---

# Reschedule engine

To-be spec v1.5 (11 September 2026). Gold packets live in `Input/` (inbound + F&O) and `Output/` (filled Reply). Load [templates.md](references/templates.md), [fo-fields.md](references/fo-fields.md), and [examples.md](references/examples.md) instead of guessing columns.

Scope: **only Pull-in rows enter the engine and `Rescheduling.xlsx`**. Push-out is no action (remain date). Cancel is a user note only.

## Pipeline

1. Consolidate request → SO and PO lines.
2. Classify Action → type.
3. Drop Push-out and Cancel from the engine (they still appear on the customer Reply).
4. For Pull-in only: inventory, supplier ETA, delivery zone.
5. Calculate recommended delivery date `I`.
6. Human review when a flag applies.
7. Write `Rescheduling.xlsx` (Pull-in only) then write back ETA / remarks / tracking on the customer Reply.

## 1. Consolidate

**Inputs**

- Customer request workbook (one of the templates in `templates.md`).
- F&O **Order lines** extract.
- F&O **Purchase order lines** extract.

There is no standalone inventory file. Inventory = SO reservation status + open PO remainder.

**Join key (in order)**

1. Customer PO# + External PN + quantity + need or due date.
2. Then Bossard item number.

Keep **one working row per request line**. Do not merge several lines that share the same PN on one PO. Unmatched rows stay in the working set with remarks `Unmatched SO/PO line`.

Attach SO/PO fields listed in `fo-fields.md`. Treat `0` and `#N/A` in UNIQUE dumps as empty, not codes.

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
| Delivery zone | Read SO **Mode of delivery**, set transport days from `fo-fields.md` |
| Availability `O` | If reservation is `Reserved ordered` / `Reserved physical`: stock is on hand, `O` = packet as-of date (do **not** add 6 working days). Otherwise: confirmed receipt + **6 working days** (Mon–Fri unless the task gives a calendar) |

Working-row flags to carry forward: inventory available Y/N, ETA date used, shipped status, mode of delivery, transport days, `O`.

If several PO receipts could supply the line, do not pick silently — remarks `Multiple availability date` and send to review.

## 4. Calculate `I` (Pull-in / Expedite only)

Column `I` is the field being calculated (Bossard feedback delivery date).

| Symbol | Meaning |
| --- | --- |
| `H` | New request delivery date (need / reschedule / new dock) |
| `G` | Current committed delivery date (SO confirmed receipt, else request current date) |
| `M` | Requested receipt date on SO |
| `N` | Confirmed ship date on SO |
| `O` | Availability from step 3 |

1. If `G` does not equal `M`: remarks `Unmatching committed date`. Still calculate `I`.
2. Look up **Purchase order lines** by customer PO number, then by Bossard item (F&O vendor PO numbers are not the customer PO). Take supplier `Confirmed receipt date`.
3. **In stock** (`Reserved ordered` / `Reserved physical`): `I` = `H` − **3 working days**. Do not go before as-of.
4. **Has supplier PO receipt**: `I` = confirmed receipt + **6 working days** + transport (929 = +3, 998 = +5). If that date is after `H`, `I` = `G` (cannot pull in).
5. **Unmatched** (no SO/PO): still calculate `I` = `H` − 3 working days and flag review. Never invent a match. Never use ship date `N`.

Column `F` on the result file is always `Expedite` / `Rescheduling In` (the Pull-in label). Column `J` is remarks when a flag applies.

## 5. Human review

Reviewer accepts or overwrites `I` and `J`. **Mandatory** when remarks is any of:

- `Unmatched SO/PO line`
- `Multiple availability date`
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
| C | Bossard item number | SO. `N/A` if unmapped |
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

Overwrite **only** that template's reply columns (see `templates.md`). Spec canonical names:

| Request column | Taken from | Write rule |
| --- | --- | --- |
| ETA / ETD / Confirm Date / Supplier confirmation ETA | Column `I` | Replace with accepted feedback date |
| REMARKS / comment | Column `J` | Replace with engine / reviewer remarks. If `J` is empty, leave remarks empty or a short accepted-date note |
| TRACKING | No dedicated result column | Copy tracking from the original request line if filled. Else PO / F&O tracking if available. Else blank |

Do not insert new request rows. Do not change Action, dates, qty, or price.

**Push-out and Cancel on the Reply sheet:** spec says leave those request lines unchanged. Gold Output still copies every inbound row onto `Reply` and often fills the date/comment columns with remain date, shipped + invoice, hold, or TBA so the customer gets an answer. Follow the gold template for that customer: copy all rows to `Reply`; fill reply columns for Pull-in from `I`/`J`; for Push-out/Cancel/Remain fill only if that customer's gold style does (remain committed date, shipped note, hold). Never add them to `Rescheduling.xlsx`.

## 7. Working days and dates

- Working days = Mon–Fri. Do not invent public holidays unless the task attaches a calendar.
- `O` = confirmed receipt + 6 working days when inbound; in-stock uses packet as-of (skip Sat/Sun).
- Result file dates: `DD-MMM-YY` (example `18-Sep-26`).
- Customer Reply dates: keep that workbook's existing cell date format. Do not stringify to text unless the gold column is already text (`shipped on 10 Sep`, `TBA`, split-qty notes).

## 8. Stop conditions

- Missing F&O extract → blocked, name the missing file.
- New template with no mapping in `templates.md` → comment headers you saw; ask which columns are PO, PN, qty, need date, action, reply.
- Mandatory review flags → `in_review`, not `done`.
- Partial qty / pack-size / shipped-split answers are allowed on the customer Reply as text (see Datalogic gold); they still do not create extra `Rescheduling.xlsx` rows unless the request itself was split lines.

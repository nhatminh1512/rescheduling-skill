---
name: rescheduling
description: >
  Classify a Bossard customer reschedule workbook as pull-in, push-out, or cancel,
  join F&O sales-order and purchase-order lines, and calculate the Pull-in ETD.
  Use when the task is a reschedule, pull-in, push-out, cancel, ETD/ETA reply, or
  writing feedback dates onto a customer request workbook. After the date is
  calculated, write it onto the original request sheet. Do not create a Reply
  sheet, a second workbook, or Rescheduling.xlsx. Do not use when posting into
  F&O, changing prices, or writing a general sales report.
metadata:
  author: Bossard
  version: "1.6"
---

# Reschedule engine

To-be spec v1.6 (7 October 2026). Load [templates.md](references/templates.md), [fo-fields.md](references/fo-fields.md), and [examples.md](references/examples.md) instead of guessing columns.

Scope: only Pull-in rows are calculated. The only deliverable is the original request workbook with the calculated ETD written onto its request sheet. Do not create `Rescheduling.xlsx`, a `Reply` sheet, or a second reply workbook.

## Pipeline

1. Read the customer request workbook and the F&O order-line and purchase-order-line extracts.
2. Classify Action → type.
3. Leave Push-out and Cancel unchanged. Do not calculate them.
4. For Pull-in only: inventory, supplier ETA, delivery zone.
5. Calculate the feedback ETD.
6. Flag human review when a rule below applies.
7. Write that ETD, and remarks when a flag applies, onto the original request sheet.

## 1. Consolidate

Inputs:

- Customer request workbook (one of the templates in `templates.md`).
- F&O Order lines extract.
- F&O Purchase order lines extract.

There is no standalone inventory file. Inventory = SO reservation status + open PO remainder.

Join key, in order:

1. Customer PO# + External PN + quantity + need or due date.
2. Then Bossard item number.

Keep one working row per request line. Do not merge several lines that share the same PN on one PO. Unmatched rows stay in the working set with remarks `Unmatched SO/PO line`.

Attach SO/PO fields listed in `fo-fields.md`. Treat `0` and `#N/A` in UNIQUE dumps as empty, not codes.

## 2. Classify

Map the customer Action / Message / Signal onto a type. Unknown labels: ask, do not invent a fourth engine type.

| Customer labels | Type | Route |
| --- | --- | --- |
| Expedite, Rescheduling In, Reschedule In, Pull-In, reschedule in | Pull-in | Calculate ETD, write it onto the request sheet |
| Delay, Rescheduling Out, Reschedule Out, Res-Out, reschedule out | Push-out | No calculation. Leave the row unchanged |
| Cancel, Partial Cancel, Cancel process | Cancel | User note only. Leave the row unchanged |
| Remain, Unconfirmed, Unconfirmed PO, Delivery is on Time, VDP forecast, confirmed, Please confirm the order | Remain | Same as Push-out. Leave the row unchanged |

Buyer remarks such as `Need immediate shipment` do not override a Remain/on-time Message into Pull-in. Type comes from the Action/Message/Signal column.

## 3. Inventory, ETA, zone (Pull-in only)

| Check | Rule |
| --- | --- |
| Inventory | Available: `Reserved ordered`, `Reserved physical`. Not available: `On order`, `Long-term`, `Line partial` / `Line partially reserved`, `Inbound` / `Inbound processing` (+ partial), `Quality` / `Quality processing` (+ partial) |
| Supplier ETA | `Not shipped` = full open qty. `Shipped partially` = remainder. `Shipped in full` = no open supply. Approval `In external review` = keep ETA and flag it |
| Delivery zone | Read SO Mode of delivery, set transport days from `fo-fields.md` |
| Availability `O` | Confirmed receipt date + 6 working days (Mon–Fri unless the task gives a calendar) |

Carry forward: inventory available Y/N, ETA date used, shipped status, mode of delivery, transport days, `O`.

If several PO receipts could supply the line, do not pick silently. Remarks `Multiple availability date` and send to review.

## 4. Calculate ETD (Pull-in / Expedite only)

The ETD is the Bossard feedback delivery date written back to the request sheet.

| Symbol | Meaning |
| --- | --- |
| `H` | New request delivery date (need / reschedule / new dock) |
| `G` | Current committed delivery date (SO confirmed receipt, else request current date) |
| `M` | Requested receipt date on SO |
| `N` | Confirmed ship date on SO |
| `O` | Availability from step 3 |

1. If `G` does not equal `M`: remarks `Unmatching committed date`. Still calculate the ETD.
2. If `O` meets `H` (`O` ≤ `H`): ETD = `O` + transport days (929 +3, 998 +5, other known codes +0).
3. If `O` does not meet `H`: ETD = `N`.

## 5. Human review

Reviewer accepts or overwrites the ETD and remarks. Mandatory when remarks is any of:

- `Unmatched SO/PO line`
- `Multiple availability date`
- `Unmatching committed date`
- approval = `In external review`

On Paperclip: comment the trace. Still write the calculated ETD onto the request sheet, with the flag in the remarks column, so the reviewer can see it. Do not mark the packet done while mandatory-review rows are open.

## 6. Write back

Write into the original request workbook. Same path the customer sent, same request sheet. Do not add a `Reply` sheet. Do not save a second workbook. Do not create `Rescheduling.xlsx`.

Overwrite only the ETD column and, when a flag applies, the remarks column named in `templates.md`. If the sheet has no ETD column, append `Bossard ETD` as the last column. If it has no remarks column and a flag applies, append `Bossard remarks` after it. Do not insert request rows. Do not change Action, dates, qty, or price.

| Request column | Write |
| --- | --- |
| ETA / ETD / Confirm Date / Supplier confirmation ETA / Bossard's ETD / Bossard's ETA | Calculated ETD. Keep the sheet's existing date format. `DD-MMM-YY` only if that column is already text |
| REMARKS / comment / Bossard's comment | Engine or reviewer remarks. If there is no flag, leave remarks empty |
| TRACKING | Copy only if the column already exists and the request or F&O line already has a tracking value. Never invent one |

Push-out, Cancel, and Remain rows are not calculated and are not written. Leave those cells as the customer sent them.

## 7. Working days and dates

- Working days = Mon–Fri. Do not invent public holidays unless the task attaches a calendar.
- `O` = confirmed receipt + 6 working days (skip Sat/Sun).
- Write dates in the request column's existing cell format. Do not stringify to text unless that column is already text (`shipped on 10 Sep`, `TBA`, split-qty notes).

## 8. Stop conditions

- Missing F&O extract → blocked, name the missing file.
- New template with no mapping in `templates.md` → comment the headers you saw; ask which columns are PO, PN, qty, need date, action, and the ETD column. Do not invent headers.
- Mandatory review flags → `in_review`, not `done`. The ETD is still on the request sheet.
- Partial qty / pack-size / shipped-split answers may be text in the ETD cell. They do not create extra rows.

---
description: Customer workbook layouts and the column to overwrite with the calculated ETD. Read when detecting a template or writing back.
---

# Customer templates

Detect the template from headers, not from the filename. Write the calculated ETD onto the request sheet of the original workbook. Do not add a `Reply` sheet and do not save a second file.

Join always: customer PO + external PN + quantity (+ need/due date, then Bossard item).

## Abinaya / BOSSARD (M) SDN BHD

Sheet the customer sent. Write `Bossard's ETD` and, when a flag applies, `Bossard's comment`. If those columns are missing, append them.

| Role | Columns |
| --- | --- |
| External PN | Item Code |
| Customer PO | Order |
| Line | Pos. / Conc |
| Qty | Ordered / Back Order |
| Current committed | Current Del. Date / Conf. Date |
| Need / dock | Needed ETD, New Request Dock Date |
| Action | Message |
| ETD write-back | Bossard's ETD |
| Remarks write-back | Bossard's comment |

Message values: `Delivery is on Time`, `Reschedule Out`, `Reschedule In`, `Unconfirmed PO`. Only `Reschedule In` is calculated.

## Datalogic W38

Same headers the customer sent. Write `Confirm Date` and, when a flag applies, `Remark`.

| Role | Columns |
| --- | --- |
| Material / PN | Material, MPN Material, Mnfr Part Number |
| Customer PO | Purchase Order + PO line |
| Qty | PO Qty |
| Current date | Delivery Date |
| Need date | New Delivery Date, Rescheduling dt–GR |
| Action | Message and Exception Message |
| ETD write-back | Confirm Date (date or free text) |
| Remarks write-back | Remark |

`VDP forecast` is Remain — do not write Confirm Date. Actionable Pull-in: `reschedule in`, and blank + Exception `Reschedule in`.

## Growri / Honeywell

Write `Bossard Feedback ETA` and, when a flag applies, `Remarks`. Do not write Vendor Feedback ETD.

| Role | Columns |
| --- | --- |
| PN | ItemCode / MPN |
| Customer PO | PO + PO Line / concat |
| Qty | Order Qty |
| Current | Confirm Del. Date, Cur. Plan Del. Date |
| Need | Need Date |
| Action | Message |
| ETD write-back | Bossard Feedback ETA |
| Remarks write-back | Remarks |

Only a Pull-in label is calculated. `Delivery is on Time`, `Res-Out`, `Please confirm the order`, and `Please share the shipment info.` stay unchanged.

## KMWE

No Action column. Treat as open-PO confirmation: write `ETD` on the same sheet where F&O can support a date. Leave the cell blank when unmatched. Append `ETD` if the column is missing.

## LimWeiSheng

Write `Supplier confirmation ETA`. Append that column if the header still says `Supplier confirmation`.

| Role | Columns |
| --- | --- |
| PN | PartName |
| Customer PO | Purchase Order + Line |
| Qty | Quantity |
| Current | Dock Date |
| Need | Reschedule Date |
| Action | Action Message |
| ETD write-back | Supplier confirmation ETA |

`Pull-In` is calculated. If the pull-in cannot be met, write the Dock Date into `Supplier confirmation ETA`.

## Nurnajwa / WW38

Write `Bossard's ETA` and, when a flag applies, `Bossard's comment`. Append those columns on the request sheet if they are missing. Do not add a second sheet.

| Role | Columns |
| --- | --- |
| PN | Item / MPN |
| Customer PO | Purchase Order + PO Pos |
| Qty | Order Qty |
| Current | Planned Delivery Date, Confirmed Delivery Date |
| Need | Rescheduled Date |
| Action | Signal Description |
| ETD write-back | Bossard's ETA |
| Remarks write-back | Bossard's comment |

Only `Reschedule In` is calculated. Do not overwrite the inbound `Confirmed` column. Treat `1970-01-01` confirmed dates as empty.

## Canonical spec names

If the file already has ETA, REMARKS, or TRACKING as its last columns, update ETA and REMARKS on matching Pull-in lines only. Leave TRACKING unchanged unless a tracking value already exists.

## New template

If headers match none of the above: comment the header row, propose PO / PN / qty / need / action / ETD mapping, and wait. Do not invent a reply workbook.

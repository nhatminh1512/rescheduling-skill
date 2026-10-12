# Customer templates

Detect the layout from **header names**, not from the filename and not from Excel letters. `G` / `H` / `I` in the spec are engine fields — they are not “column G/H/I” on the customer sheet.

CLI/web: the model maps names → roles (PO, PN, qty, need date, committed date, reply). Known layouts below are gold documentation and the `--no-deepseek` fallback, not a required letter map.

Copy `Original` through unchanged. Put answers on `Reply`. Reply column names differ by customer; map them onto spec ETA / REMARKS / TRACKING.

Join always: customer PO + external PN + quantity (+ need/due date, then Bossard item).

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

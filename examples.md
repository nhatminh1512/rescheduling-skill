# Worked packets

These are gold outcomes, not extra rules. If a gold cell conflicts with `SKILL.md`, follow the skill and note the deviation in the task comment.

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

Each customer workbook already contains `Original` + `Reply` in both folders. `Input` is the paired example; `Output` is the deliverable shape. F&O extracts are Input-only and shared.

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

---
description: Historical gold packets. Date examples only. Do not copy their Reply sheet or Rescheduling.xlsx output.
---

# Worked packets

These are date examples, not extra output rules. v1.6 writes the ETD onto the original request sheet. If a gold cell conflicts with `SKILL.md`, follow the skill.

Gold files previously saved a `Reply` sheet and, for some packets, a separate `Rescheduling.xlsx`. Do not reproduce either file. Use the gold rows only to check classification and the calculated date.

## Packet map

| Input | Template |
| --- | --- |
| `Abinaya_15SEP26_BOSSARD (M) SDN BHD_ORI.xls` | Abinaya |
| `Datalogic- Rescheduled Report - W38'26.xlsx` | Datalogic |
| `Growri_15SEP26_Honeywell_ORI.xlsx` | Honeywell |
| `KMWE_May_04052026.xlsx` | KMWE |
| `LimWeiSheng_8SEP26_Reschedule7.xlsx` | LimWeiSheng |
| `Nurnajwa_17SEP26_WW38 Ori.xlsx` | Nurnajwa |
| `Order lines_20260805.xlsx` | F&O SO |
| `Purchase order lines_20260805.xlsx` | F&O PO |

## Classify from gold

- Abinaya 123 data rows: 77 on-time, 24 Reschedule Out, 18 Unconfirmed PO, 4 Reschedule In. Only the 4 In rows get an ETD.
- LimWeiSheng 38 data rows: all `Pull-In` → all get an ETD.
- Nurnajwa: 7 In, 14 Out, 2 blank/unconfirmed. Engine writes 7 In rows.
- Datalogic: 504 `VDP forecast` stay blank. Write Confirm Date only on reschedule-in lines.
- Growri 6 rows: on-time, Res-Out, confirm-order, and shipment-info stay unchanged unless the Message is a Pull-in label.

## Date behaviour from gold (illustrative)

LimWeiSheng pull-in accepted early:

- Dock 2026-11-23, request 2026-10-22, confirmation ETA 2026-10-19.

LimWeiSheng pull-in not met (remain dock):

- Dock 2026-09-14, request 2026-09-09, confirmation ETA 2026-09-14.

Nurnajwa pull-in:

- Signal In, rescheduled 2026-09-23, Bossard's ETA 2026-09-22.
- Signal In, rescheduled 2026-09-30, confirmed delivery 2026-10-30, ETA 2026-10-27.

## Join reminder

Do not collapse lines that repeat the same PN on one PO with different qty or dates. One request line → one ETD cell if the type is Pull-in.

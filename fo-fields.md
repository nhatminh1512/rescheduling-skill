# F&O fields and select values

Source extracts in gold packets: `Order lines_20260805.xlsx`, `Purchase order lines_20260805.xlsx`. Use **Sheet1** only. Sheet2 is a UNIQUE dump (`#N/A` / `0` = empty).

## Order lines (SO)

| Field | Used for |
| --- | --- |
| Customer account / Customer name | Identity |
| Sales order / Internal line | SO key |
| Customer requisition | Customer PO# |
| External | External PN |
| Item number | Bossard item number |
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
| Purchase order / Line number | PO key |
| Vendor account / Name | Supplier |
| Item number | Bossard item |
| Quantity / Deliver remainder | Open supply |
| Shipped | Not shipped / Shipped partially / Shipped in full |
| Warehouse | PEN |
| Confirmed shipping date | Supplier ship ETA |
| Confirmed receipt date | Supplier receipt ETA; `O` = this date + 6 working days when supply is inbound |
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

Carry at least: request identity, type, `G`, `H`, `M`, `N`, `O`, reservation, shipped, approval, mode, transport days, match status, remarks flags.

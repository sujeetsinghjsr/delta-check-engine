# MiFID II Delta Check Engine — State Machine Design

> ANZ Bank London Branch | Publicis Sapient | 2026

## Overview

Design of the **Stage 3 Aggregator / Delta Check** component for MiFID II regulatory
reporting. Determines whether a trade event should be reported as NEWT, REPL, or CANC
and manages the full report lifecycle.

## State Machine — MIFID_STATUS

```
                    ┌─────────────────────────────────┐
                    │                                 │
Write 2             ▼                                 │
    → MIFID_STATUS = PENDING                          │
                    │                                 │
Phase A             ▼                                 │
    → MIFID_STATUS = READING (lock)                   │
                    │                                 │
Phase B             ▼                                 │ IS_NRPT = Y
    IS_NRPT = N → continue                            │
    IS_NRPT = Y → MIFID_STATUS = SUPPRESSED ──────────┘
                    │
Phase C             ▼
    Delta check: compare outbound vs last submission
    No delta  → not reportable → SKIP
    Delta found → determine MIFID_ACTION:
        No prior record → NEWT
        Economic field changed → REPL
        Cancel event → CANC
    Write-back to persistence:
        QUANTITY, DECR_INCR, MIFID_ACTION
                    │
Phase D             ▼
    ARM/APA submission
    Success → MIFID_STATUS = COMPLETE
    Failure → MIFID_STATUS = ERROR → retry / dead-letter
```

## MIFID_ACTION Decision Logic

| Condition | MIFID_ACTION |
|---|---|
| No prior ARM record for this MIFID_LINK_ID | NEWT (New Transaction) |
| Economic field changed vs last submission | REPL (Replace) |
| Cancel / Cancel & Rebook event | CANC (Cancel) |

## Economic Fields (change triggers REPL)

| Field | ARM Code | Description |
|---|---|---|
| ISIN | F41 | Financial instrument identifier |
| Buyer Identification Code | F7 | MiFID buyer LEI |
| Notional INCR/DECR | F32 | Quantity direction flag |
| Net Amount | F35 | Notional value |
| Price | F33 | Trade price |
| Price Currency | F34 | Currency of price |
| Quantity | F30 | Trade quantity |
| Quantity Currency | F31 | Currency of quantity |
| Seller Identification Code | F16 | MiFID seller LEI |

## DECR_INCR Flag Logic

```
Worked example:
    New trade:        Quantity = 100  →  reportable qty = 100  →  DECR_INCR = NULL
    Partial unwind:   Quantity = 70   →  delta = -30           →  reportable qty = 30
                                                                →  DECR_INCR = DECR
    Correction:       Must carry DECR flag from previous report
                      Without persistence, system cannot know it was originally DECR
                      → correction would be reported incorrectly
```

## Phase C Write-Back SQL

```sql
UPDATE RDP_MIFID_TRADE_DATA_PERSISTENCE
SET
    QUANTITY          = :new_quantity,
    DECR_INCR         = :decr_incr_flag,   -- INCR / DECR / NULL
    MIFID_ACTION      = :action,            -- NEWT / REPL / CANC
    UPDATED_TIMESTAMP = CURRENT_TIMESTAMP
WHERE
    MIFID_LINK_ID = :link_id
    AND MIFID_STATUS = 'READING';
```

## Technology Stack

SQL · MiFID II ARM RTS 22 · State Machine Design · Regulatory Reporting

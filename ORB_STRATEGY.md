# ORB Strategy V2 — Pine Script Spec

## 1. Global System Parameters

| Parameter | Value |
|-----------|-------|
| Ticker Universe | GC, ES, NQ, YM, MGC, MES, MNQ, MYM |
| Execution Timeframe | 5-minute (5m) |
| ORB Source Timeframe | 15-minute (15m) via `request.security()` |
| ORB Session Window | 08:00–08:15 ET daily |
| Session Reset | 18:00 ET (futures session boundary) |
| Default Max Entries/Session | 2 (per instrument, user-adjustable) |
| Default Time Cutoff | 4:00 PM ET (per instrument, user-adjustable) |
| Take Profit | 2:1 RR |
| Stop Loss → Break-Even | Move to BE at 1:1 RR |

---

## 2. Breakout Validation Logic

Three conditions must be met in sequence on the 5m chart:

1. **Candle_1**: close only must be beyond the 15m ORB High (long) or ORB Low (short). Open may still be inside.
2. **Candle_2+**: at least one subsequent candle must have both open AND close beyond the ORB boundary (full body outside).
3. **Wick Cleared**: Candle_1 close must break the extreme of the most recent 2-candle pair immediately before it:
   - **Long**: C1 close > highest high of the last [green, red] candle pair
   - **Short**: C1 close < lowest low of the last [red, green] candle pair

---

## 3. Liquidity Definition (Replaces Gap Filter)

- Liquidity sweep = price revisiting a prior confirmed swing high or swing low
- **Swing Detection**: 3-bar pivot algorithm
  - Swing High: `high[1] > high[0] and high[1] > high[2]` → level = `high[1]`
  - Swing Low: `low[1] < low[0] and low[1] < low[2]` → level = `low[1]`
- Scan from 18:00 ET session open to current bar, use most recent confirmed pivot
- Liquidity sweep is a required condition before BOS entry fires
- No separate pre-market gap filter

---

## 4. Entry Setup Matrix (Pullback Phase)

*Validated breakout must occur before any pullback/entry triggers.*

| Setup ID | Pullback Depth | Liquidity Sweep Required | Default Enabled |
|----------|---------------|--------------------------|-----------------|
| **A** | Deep — breaches through entire ORB zone | Yes | Yes |
| **B** | Mid — touches ORB 50% midpoint | Yes | Yes |
| **C** | Shallow — touches ORB boundary edge | Yes | Yes |
| **2B** | Mid — touches ORB 50% midpoint | No | Yes |
| **2C** | Shallow — touches ORB boundary edge | No | Yes |

> **2A is strictly invalid** — deep breach without liquidity sweep = no trade, ever.

- All 5 grades fire real entries by default
- Each grade individually toggleable ON/OFF via user input
- Each grade has its own user-selectable label color
- Entry labeled with grade on the entry candle

---

## 5. Entry Trigger (BOS)

- Entry fires on close of the BOS candle (body closes beyond structural high/low)
- Structural level = most recent confirmed swing high (long) or swing low (short) from 3-bar pivot scan
- `process_orders_on_close = true`
- Volume is **not** a filter — visual context only (see Section 8)

---

## 6. Secondary Entry Logic (Max 1 Add-on Per Day)

### Scenario 1: Re-Entry (Post-BE Hunt)
- Position 1 hits 1:1 RR → SL moved to BE → price stops out at BE
- If price then sweeps liquidity (prior H/L) and/or revisits ORB zone → new BOS fires → re-entry
- Managed with standard 2:1 TP and BE at 1:1 RR
- Implemented as a new `strategy.entry()` call

### Scenario 2: Scale-In (Pyramiding)
- Position 1 active and floating in profit (not yet at BE or TP)
- Price pulls back, sweeps liquidity and/or revisits ORB zone, new BOS fires in same direction
- Implemented as a new `strategy.entry()` call, own TP/SL from its own BOS candle

---

## 7. Instrument Rules

| Instrument | Default Max Trades/Session | Default Time Cutoff | Notes |
|------------|---------------------------|---------------------|-------|
| NQ | 2 | 4:00 PM ET | User-adjustable |
| MNQ | 2 | 4:00 PM ET | User-adjustable |
| ES | 2 | 4:00 PM ET | User-adjustable |
| MES | 2 | 4:00 PM ET | User-adjustable |
| YM | 2 | 4:00 PM ET | User-adjustable |
| MYM | 2 | 4:00 PM ET | User-adjustable |
| GC | 2 | 4:00 PM ET | User-adjustable |
| MGC | 2 | 4:00 PM ET | User-adjustable |

---

## 8. Visual Elements

| Element | Default | User Toggle | Colors |
|---------|---------|-------------|--------|
| ORB box | ON | — | User-selectable |
| ORB midpoint dashed line | ON | — | User-selectable |
| ORB box/midpoint end time | 5:00 PM ET | Adjustable time input | — |
| Swing high triangles (above wick) | ON | Yes | User-selectable |
| Swing low triangles (below wick) | ON | Yes | User-selectable |
| Setup grade label on entry candle | ON | Per grade | Per grade, user-selectable |
| Entry / SL / TP lines | ON | — | User-selectable |
| Volume context (vol vs avg overlay) | OFF | Yes | User-selectable |
| VWAP | OFF | Yes | User-selectable |
| Prior Day High/Low | OFF | Yes | User-selectable |

---

## 9. User-Configurable Inputs

| Input | Group | Default |
|-------|-------|---------|
| Commission per side ($) | Backtest Settings | 2.00 |
| Slippage (ticks) | Backtest Settings | 1 |
| Max trades/session (per instrument × 8) | Instrument Rules | 2 |
| Time cutoff (per instrument × 8) | Instrument Rules | 4:00 PM ET |
| ORB box end time | ORB Settings | 5:00 PM ET |
| Enable grade A | Setup Grades | ON |
| Enable grade B | Setup Grades | ON |
| Enable grade C | Setup Grades | ON |
| Enable grade 2B | Setup Grades | ON |
| Enable grade 2C | Setup Grades | ON |
| Color: grade A label | Colors | User |
| Color: grade B label | Colors | User |
| Color: grade C label | Colors | User |
| Color: grade 2B label | Colors | User |
| Color: grade 2C label | Colors | User |
| Color: swing high triangle | Colors | User |
| Color: swing low triangle | Colors | User |
| Color: ORB box | Colors | User |
| Color: ORB midpoint | Colors | User |
| Color: VWAP | Colors | User |
| Color: PDH/PDL | Colors | User |
| Volume lookback (N bars for avg) | Filters | 20 |
| Show volume context | Filters | OFF |
| Show VWAP | Reference Lines | OFF |
| Show Prior Day High/Low | Reference Lines | OFF |

---

## 10. Pine Script v6 Notes

- `//@version=6`
- `request.security()` for 15m ORB data — called at top level, not inside `if` blocks
- `strategy()` commission/slippage require `const` — set via TradingView Properties tab
- All multi-line booleans use intermediate variables (no `and` at line start/end)
- All local vars explicitly typed (`float`, `bool`, `int`)
- `math.avg()` removed — use `(a + b) / 2`
- `barmerge.lookahead_off` only

---

## 11. File Locations

- V1 script: `orb_strategy.pine`
- V2 script: `orb_strategy_v2.pine` (to be created)
- GitHub repo: `orb-strategy` (private)

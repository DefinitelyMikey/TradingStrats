# TradingBot — ORB Strategy Plan

## Overview

TradingView Pine Script indicator + strategy. 15-min ORB variation starting at 8:00 AM ET.
Instruments: MNQ, NQ, ES, MES, MGC.

---

## 1. ORB Definition

- **Timeframe**: 15-min chart only (hardcoded)
- **ORB candle**: 8:00 AM ET candle (single 15-min candle, 8:00–8:15 AM)
- **ORB high**: wick high of 8:00 AM candle
- **ORB low**: wick low of 8:00 AM candle
- **ORB midpoint**: (ORB high + ORB low) / 2
- **ORB resets**: daily

---

## 2. Setup Conditions

### Setup Quality

| Grade | Condition |
|-------|-----------|
| **A+** | Price breaks ORB level → retraces to ORB midpoint → B.O.S. triggers |
| **Valid** | Price touches ORB range OR revisits a previous swing high/low before B.O.S. |

### SELL Setup (Short)

1. Price breaks **below ORB low** (swing low forms outside ORB range)
2. Price retraces back up to at least ORB midpoint (A+ condition)
3. Find most recent **black → blue** candle pair (anywhere in lookback)
4. Structure level = `min(low of black candle, low of blue candle)` — wick low of either
5. **B.O.S. trigger**: candle **body closes below** that structure level
6. **Invalidation**: any candle body closes **above ORB high** → setup void

### BUY Setup (Long)

1. Price breaks **above ORB high** (swing high forms outside ORB range)
2. Price retraces back down to at least ORB midpoint (A+ condition)
3. Find most recent **blue → black** candle pair (anywhere in lookback)
4. Structure level = `max(high of blue candle, high of black candle)` — wick high of either
5. **B.O.S. trigger**: candle **body closes above** that structure level
6. **Invalidation**: any candle body closes **below ORB low** → setup void

---

## 3. Entry / Stop / Target

- **Entry**: close of B.O.S. candle (`process_orders_on_close = true`)
  - Short entry = body low of B.O.S. candle (= close of bearish candle)
  - Long entry = body high of B.O.S. candle (= close of bullish candle)
- **Stop loss**:
  - Short: body high of B.O.S. candle
  - Long: body low of B.O.S. candle
- **Risk (R)**: |body high − body low| of B.O.S. candle
- **Trade management**:
  - At **1R**: move SL to breakeven (entry price)
  - At **2R**: full exit (all-in / all-out)

---

## 4. Instrument Rules

| Instrument | Time Cutoff | Max Trades/Session |
|------------|-------------|-------------------|
| NQ | None | Unlimited |
| ES | None | Unlimited |
| MNQ | None | Unlimited |
| MES | None | Unlimited |
| MGC | 12:00 PM ET | 2 |

---

## 5. Strategy Direction

- **Bi-directional** — takes both longs and shorts
- No directional filter (no pre-market bias requirement)
- Direction determined purely by which B.O.S. fires

---

## 6. Visual Elements

- ORB box: shaded rectangle, stops at 8:15 AM (spans ORB candle only)
- ORB midpoint: dashed horizontal line, stops at 8:15 AM
- B.O.S. level: dashed horizontal line, active while setup is live
- Post-entry: entry line, SL line (updates to BE at 1R), TP1 dotted, TP2 solid
- All colors: user-selectable via `input.color()`
- No alerts
- No ORB size filter

---

## 7. User-Configurable Options

| Option | Group | Default |
|--------|-------|---------|
| Commission per side ($) | Backtest Settings | 2.00 |
| Slippage (ticks) | Backtest Settings | 1 |
| Min B.O.S. body size (ATR ratio) | Filters | 0.15 |
| One trade at a time | Filters | ON |
| Allow re-entry after stop | Filters | ON |
| Enable pre-market gap filter | Filters | OFF |
| Max allowable gap (%) | Filters | 0.5 |
| Show VWAP | Reference Lines | OFF |
| Show Prior Day High/Low | Reference Lines | OFF |
| All color inputs | Colors | Various |

---

## 8. Implementation Notes

- `process_orders_on_close = true` — entry fills at B.O.S. candle close to preserve 1:2 R:R
- Swing pair scan: look back 100 bars (i=1 to 100) for most recent pair — skip current bar (i=0)
- Black = bearish (close < open), Blue = bullish (close > open)
- Entry = close, SL = open (works for both short and long B.O.S. candles)
- Risk = |close − open| of B.O.S. candle
- Session reset at 18:00 ET (futures new session)
- MGC detected via `str.contains(syminfo.ticker, "MGC")`
- `request.security()` calls at top level (not inside if blocks) to avoid Pine Script warnings
- All setup state managed with `var` variables reset on new_session

---

## 9. Pine Script v6 Migration Notes

Script is written in **Pine Script v6** (`//@version=6`). Key v5→v6 differences hit during implementation:

### Breaking Changes

| Issue | v5 | v6 Fix |
|-------|----|--------|
| `strategy()` commission/slippage | `commission_value=i_commission` worked | Requires `const float/int` — inputs not allowed. Remove from `strategy()`. Set via TradingView **Strategy Settings → Properties** tab instead. |
| Multi-line boolean with `and` | `and` at start of continuation line worked | `and` at start OR end of line both fail (CE10013/CE10156). Use **intermediate bool variables** — one expression per line. |
| `math.avg()` | `math.avg(high, low)` | Removed. Use `(high + low) / 2`. |
| `barmerge.lookahead_on` | Used in `request.security()` | Deprecated. Use `barmerge.lookahead_off`. Safe here because `[1]` indexing already reads confirmed prior-bar data. |
| Local variable types | `gap_pct = ...` inside `if` block | Must be explicit: `float gap_pct = ...` |
| Loop variable types | `c_bull = close[i] > open[i]` inside `for` | Must be explicit: `bool c_bull = close[i] > open[i]` |
| Local risk variable | `_r = t_sl - t_entry` inside `if` | Must be explicit: `float _r = t_sl - t_entry` |

### Commission / Slippage Workflow
Since `strategy()` requires compile-time constants, users set these in TradingView UI:
1. Add strategy to chart
2. Click ⚙️ gear icon on the strategy name
3. Go to **Properties** tab
4. Set **Commission** ($ per contract) and **Slippage** (ticks)

The `i_commission` and `i_slippage` inputs in the script are retained as reference labels only (tooltips explain this).

---

## 10. File Location

- V1 Script: `orb_strategy.pine`
- V2 Script: `orb_strategy_v2.pine`
- GitHub repo: `orb-strategy` (private)

---

## 11. V2 Script Notes

- Runs on **5m chart**; 15m ORB sourced via `request.security()` using `ta.valuewhen`
- **Breakout validation**: C1 close outside ORB + clears last candle pair extreme (wick cleared); C2+ full body (open AND close) outside ORB
- **Swing detection**: 3-bar pivot — `high[1] > high[0] and high[1] > high[2]`
- **Pullback depth**: 1=shallow (ORB boundary), 2=mid (ORB midpoint), 3=deep (through ORB)
- **Liquidity sweep**: price revisits last confirmed session pivot before BOS fires
- **Setup 2A** (deep + no liquidity) = strictly invalid — `grade_str()` returns `""`, never trades
- **Grade filter**: each grade (A/B/C/2B/2C) individually toggleable via input; all ON by default
- **BE management**: tracks up to 2 simultaneous open entries via `s1_*` / `s2_*` vars; BE moves stop to entry at 1:1 RR
- **Re-entry**: `pos_closed` detection resets `bos_lvl_long/short` so new pivots form fresh BOS targets; `brk_valid` and pullback depth NOT reset (breakout still valid)
- **`pyramiding=2`** supports primary + scale-in simultaneously
- `new_session = et_hour == 18 and et_minute == 0` — same as V1
- Commission/slippage: set in TradingView Properties tab (same as V1)

# TradingBot — ORB Strategy Plan

## Overview

TradingView Pine Script indicator + strategy. 15-min ORB variation starting at 8:00 AM ET.
Instruments: MNQ, NQ, ES, MES, MGC.

---

## 1. ORB Definition

- **Timeframe**: 5-min execution chart
- **ORB candle**: the 8:00–8:15 AM ET window — built natively from the three 5m bars (08:00, 08:05, 08:10), high/low aggregated, finalized at the 08:10 close
- **ORB high**: wick high of 8:00 AM candle
- **ORB low**: wick low of 8:00 AM candle
- **ORB midpoint**: (ORB high + ORB low) / 2
- **ORB resets**: daily

---

## 2. Setup Conditions

No grading. A setup either completes all of the steps below (in order) and fires, or it doesn't.

The engine tracks exactly **two candle-pair levels** every bar (including pre-8AM):

- **`gr_high`** = high of the most recent **green→red** (bullish→bearish) pair — the higher of the two candle highs.
- **`rg_low`** = low of the most recent **red→green** (bearish→bullish) pair — the lower of the two candle lows.

These do double duty: a short's structural level = `gr_high` and its floor = `rg_low`; a long's structural level = `rg_low` and its ceiling = `gr_high`.

### SELL Setup (Short)

1. **Breakout** — two **consecutive** bearish bars both body-close below ORB low (`close < open AND close < ORB low`, twice in a row). Latches for the session.
2. **Structural / liquidity level** = `gr_high` (high of most recent green→red pair). If no green→red pair forms after the break, falls back to the most recent **pre-8AM** `gr_high`.
3. **Dual revisit** (any order, not necessarily the same candle; "revisit" = price touches): price touches the structural level (`high >= gr_high`) **AND** price touches into the ORB band (`high >= ORB low`). Evaluated only on bars *after* the breakout bar.
4. **Floor** — once both revisits are done, lock `floor = rg_low` (low of most recent red→green pair).
5. **Floor transfer** — a candle that *wicks* below the floor but doesn't *body-close* below it (`low < floor AND close >= floor`) moves the floor **down** to that wick's low. Repeats until a body closes below.
6. **Entry** — a **bearish** bar body-closes below the valid floor (`close < floor`).

### BUY Setup (Long) — exact mirror

1. **Breakout** — two consecutive bullish bars both body-close above ORB high.
2. **Structural / liquidity level** = `rg_low` (low of most recent red→green pair), with pre-8AM fallback.
3. **Dual revisit** — price touches the structural level (`low <= rg_low`) **AND** touches into the ORB band (`low <= ORB high`).
4. **Ceiling** — lock `ceiling = gr_high` (high of most recent green→red pair).
5. **Ceiling transfer** — a wick piercing above without a body-close-above moves the ceiling **up** to that wick's high.
6. **Entry** — a **bullish** bar body-closes above the valid ceiling.

---

## 3. Entry / Stop / Target

The **trigger candle** is the bar that body-closes beyond the valid floor (short) / ceiling (long).

- **Entry**: close of the trigger candle (`process_orders_on_close = true`)
- **Stop loss**: body open of the trigger candle
  - Short trigger is bearish → open is above the close → SL sits above entry ✓
  - Long trigger is bullish → open is below the close → SL sits below entry ✓
- **Risk (R)**: `|close − open|` of the trigger candle
- **Target**: 2R (`close ∓ 2R`)
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
- Direction determined purely by which setup completes (short floor-break vs long ceiling-break)

---

## 6. Visual Elements

- ORB box: shaded rectangle, extends rightward until `i_orb_end` (default 17:00 ET)
- ORB midpoint: dashed horizontal line, extends to `i_orb_end`
- Liquidity / structural level: dashed line drawn after the breakout, while waiting for the revisit; cleared once revisited
- Floor / ceiling: dashed line drawn once locked; moves as the level transfers
- Entry/exit: TradingView's built-in strategy markers (no custom entry/SL/TP lines)
- All level lines toggle together via **Show Level Lines**
- All colors: user-selectable via `input.color()`
- Optional: VWAP, prior-day high/low, volume-spike background
- No alerts; no ORB size filter; no grade labels

---

## 7. User-Configurable Options

| Option | Group | Default |
|--------|-------|---------|
| Commission per side ($) | Backtest Settings | 2.00 (reference only) |
| Slippage (ticks) | Backtest Settings | 1 (reference only) |
| Per-instrument Max Trades/Session | Instrument Rules | 2 each |
| Per-instrument Time Cutoff (HHMM ET) | Instrument Rules | 1600 each |
| ORB Box End Time (HHMM ET) | ORB Settings | 1700 |
| Show Level Lines | Visuals | ON |
| Show Volume Context | Visuals | OFF |
| Show VWAP | Visuals | OFF |
| Show Prior Day High/Low | Visuals | OFF |
| Volume Avg Lookback (bars) | Visuals | 20 |
| All color inputs | Colors | Various |

---

## 8. Implementation Notes

- `process_orders_on_close = true` — entry fills at the trigger candle's close to preserve 1:2 R:R
- Candle-pair tracking runs **every bar** (including pre-8AM) so `gr_high` / `rg_low` are always the most recent pair levels and the pre-8AM fallback is automatic
- Pair detection uses the prior bar: green→red = `close[1] > open[1] AND close < open`; red→green = `close[1] < open[1] AND close > open`
- Breakout = two consecutive body-closes beyond the ORB edge; latches via `bear_brk` / `bull_brk`
- Revisit is gated on `bear_brk[1]` / `bull_brk[1]` so the breakout candle itself can't satisfy it
- Floor/ceiling locks at the moment both revisits complete, then only moves via piercing wicks
- Bearish = close < open, Bullish = close > open
- Entry = close, SL = open (works for both short and long trigger candles)
- Setup state resets **at each entry** and **on position close** (revisit flags + floor/ceiling cleared; breakout stays latched) so re-entry requires a fresh revisit
- Session reset at 18:00 ET (futures new session) clears all state
- MGC detected via `str.contains(syminfo.ticker, "MGC")`
- `request.security()` calls at top level (not inside if blocks) to avoid Pine Script warnings
- All setup state managed with `var` variables; `max_lines_count=500` for the level lines

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

- V2 Script: `orb_strategy_v2.pine`
- GitHub repo: `orb-strategy` (private)

---

## 11. V2 Script Notes

Signal engine rewritten 2026-06-10. The old grading/pullback-depth/pivot-sweep system was removed entirely.

- Runs on **5m chart**; ORB built natively from the three 5m bars in the 08:00–08:15 window (no `request.security` — avoids the cross-timeframe lookahead delay that captured the prior session's ORB on the execution chart)
- **Two tracked levels**: `gr_high` (most recent green→red pair high) and `rg_low` (most recent red→green pair low). Short structural = `gr_high`, short floor = `rg_low`; long structural = `rg_low`, long ceiling = `gr_high`
- **Breakout**: two consecutive bearish (long: bullish) bars both body-close beyond the ORB edge; `bear_brk` / `bull_brk` latch for the session
- **Dual revisit**: structural-level touch + ORB-band touch, any order, gated on the prior bar's breakout latch
- **Floor / ceiling**: locks to `rg_low` / `gr_high` when both revisits complete; transfers via piercing wicks (`low < floor AND close >= floor` → `floor := low`)
- **Entry**: directional bar body-closes beyond the valid floor/ceiling
- **BE management**: tracks up to 2 simultaneous open entries via `s1_*` / `s2_*` vars; BE moves stop to entry at 1:1 RR
- **Re-entry**: setup state (revisit flags + floor/ceiling + their lines) resets at each entry and on `pos_closed`; `bear_brk` / `bull_brk` stay latched, so a fresh revisit is required before re-entry
- **`pyramiding=2`** supports a second simultaneous entry if a fresh setup completes before the first closes
- `new_session = et_hour == 18 and et_minute == 0` — same as V1
- Commission/slippage: set in TradingView Properties tab (same as V1)
- **Removed**: grade inputs/colors/labels, `grade_str()`, `grade_col()`, pullback-depth vars, `ta.pivothigh/pivotlow` swing detection, swing markers, old C1/C2 breakout rules

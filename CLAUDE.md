# TradingBot — ORB Strategy Plan

## Overview

TradingView Pine Script strategy. Opening-range breakout starting at 8:00 AM ET, with a
**user-selectable ORB duration** (5/10/15/20/30/60 min). Default reference instruments: MES & MNQ.
Backtest defaults baked into `strategy()`: $50,000 capital, $0.72/side commission, 5% margin.

---

## 1. ORB Definition

- **Timeframe**: 5-min execution chart (trade here). The ORB box + midpoint also render on **any** chart timeframe (see Section 6)
- **ORB candle**: the 8:00 AM ET window — **length user-selectable** via `i_orb_minutes` (5/10/15/20/30/60 min, default 15 = 08:00–08:15). Built natively by aggregating every 5m bar in `[orb_start_min, orb_end_min)` (minutes-of-day math, `orb_start_min = 480`), finalized on the final in-window bar (`orb_end_min - 5`)
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

1. **Breakout** — two **consecutive** bars both body-close below ORB low (`close < ORB low`, twice in a row). **Candle color does not matter** — only the close being outside the range. Latches for the session.
2. **Structural / liquidity level** = `gr_high` (high of most recent green→red pair). If no green→red pair forms after the break, falls back to the most recent **pre-8AM** `gr_high`.
3. **Dual revisit** (any order, not necessarily the same candle; "revisit" = price touches): price touches the structural level (`high >= gr_high`) **AND** price touches into the ORB band (`high >= ORB low`). Evaluated only on bars *after* the breakout bar.
4. **Floor** — once both revisits are done, lock `floor = rg_low` (low of most recent red→green pair).
5. **Floor transfer** (both, most-recent governs): (a) each new red→green pair **re-syncs** the floor to that pair's low (up or down); (b) a candle that *wicks* below the floor but doesn't *body-close* below it (`low < floor AND close >= floor`) moves the floor **down** to that wick's low. Repeats until a body closes below.
6. **Entry** — a **bearish** bar body-closes below the valid floor (`close < floor`).

### BUY Setup (Long) — exact mirror

1. **Breakout** — two consecutive bars both body-close above ORB high (any color).
2. **Structural / liquidity level** = `rg_low` (low of most recent red→green pair), with pre-8AM fallback.
3. **Dual revisit** — price touches the structural level (`low <= rg_low`) **AND** touches into the ORB band (`low <= ORB high`).
4. **Ceiling** — lock `ceiling = gr_high` (high of most recent green→red pair).
5. **Ceiling transfer** (both, most-recent governs): (a) each new green→red pair **re-syncs** the ceiling to that pair's high (up or down); (b) a wick piercing above without a body-close-above moves the ceiling **up** to that wick's high.
6. **Entry** — a **bullish** bar body-closes above the valid ceiling.

---

## 3. Entry / Stop / Target

The **trigger candle** is the bar that body-closes beyond the valid floor (short) / ceiling (long).

- **Entry**: close of the trigger candle (`process_orders_on_close = true`)
- **Stop loss**: body open of the trigger candle (default)
  - Short trigger is bearish → open is above the close → SL sits above entry ✓
  - Long trigger is bullish → open is below the close → SL sits below entry ✓
- **Risk (R)**: `|close − open|` of the trigger candle
- **Target**: 2R (`close ∓ 2R`) (default)
- **Trade management**:
  - At **1R**: move SL to breakeven (entry price) — toggleable, see Section 7
  - At **2R**: full exit (all-in / all-out)
  - **4:45 PM ET**: any open position is force-flattened (EOD flatten), see Section 4

### Custom SL/TP Multiples (optional)

When **Enable Custom SL/TP Multiples** is ON (default OFF), the fixed SL=open / TP=2R rule above is replaced:

- **Stop loss**: `entry ∓ (SL Multiple × R)` where `R = |close − open|` of the trigger candle
- **Take profit**: `entry ± (TP Multiple × R)`
- Defaults for the multiples (1.0 / 2.0) reproduce the original SL=open / TP=2R behavior exactly
- Breakeven distance (`|entry − SL|`) automatically scales with the SL Multiple, so the 1R breakeven trigger stays consistent with the custom stop

---

## 4. Trading Rules (global — all instruments)

| Rule | Value |
|------|-------|
| Max trades per day | **1** (all instruments) |
| Trading window | **8:00 AM – 12:00 PM ET** (`et_hhmm >= 800 and et_hhmm < 1200`) |
| EOD flatten | Any open position is force-closed at **4:45 PM ET** (`et_hhmm == 1645`) via `strategy.close_all()` |
| After the day's trade closes | all setup drawing stops (ORB box/midpoint keep extending) |

Rules are identical for every instrument — no per-instrument max-trades or time-cutoff settings.

---

## 5. Strategy Direction

- **Bi-directional** — takes both longs and shorts
- No directional filter (no pre-market bias requirement)
- Direction determined purely by which setup completes (short floor-break vs long ceiling-break)

---

## 6. Visual Elements

- ORB box: shaded rectangle, extends rightward until `i_orb_end` (default 17:00 ET). **Renders on any chart timeframe** — sourced from a self-contained 5m calc (`f_orb_box()`) via `request.security("5", …, lookahead_off)` and drawn with `xloc.bar_time` / `time_close`, so the box is identical on 1m/5m/15m/1h. Trade execution + setup lines stay 5m-only
- ORB midpoint: dashed horizontal line, extends to `i_orb_end` (also cross-timeframe)
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
| Commission per side ($) — MES/MNQ | Backtest Settings | 0.72 (display ref; live value baked into `strategy()`) |
| Slippage (ticks) | Backtest Settings | 1 (reference only; not baked) |
| ORB Duration (min) | ORB Settings | 15 — dropdown: 5/10/15/20/30/60 |
| ORB Box End Time (HHMM ET) | ORB Settings | 1700 |
| Move SL to Breakeven at 1R | ORB Settings | ON |
| Enable Custom SL/TP Multiples | ORB Settings | OFF |
| SL Multiple (x candle range) | ORB Settings | 1.0 |
| TP Multiple (x candle range) | ORB Settings | 2.0 |
| Reverse Signals (Buy↔Sell) | ORB Settings | OFF |
| Show Level Lines | Visuals | ON |
| Show Volume Context | Visuals | OFF |
| Show VWAP | Visuals | OFF |
| Show Prior Day High/Low | Visuals | OFF |
| Volume Avg Lookback (bars) | Visuals | 20 |
| All color inputs | Colors | Various |

**Baked into `strategy()` (literal constants — these actually apply in the backtest):**
`initial_capital=50000` · `commission_type=cash_per_contract` · `commission_value=0.72` (Tradovate MES/MNQ all-in: $0.39 comm + $0.22 exch + $0.09 clearing + $0.02 NFA) · `margin_long=5` · `margin_short=5` (so $50K can afford 1 MNQ ≈ $61.7K notional). Manual Properties overrides mask these — reset Properties to **Defaults** to apply.

---

## 8. Implementation Notes

- `process_orders_on_close = true` — entry fills at the trigger candle's close to preserve 1:2 R:R
- Candle-pair tracking runs **every bar** (including pre-8AM) so `gr_high` / `rg_low` are always the most recent pair levels and the pre-8AM fallback is automatic
- Pair detection uses the prior bar: green→red = `close[1] > open[1] AND close < open`; red→green = `close[1] < open[1] AND close > open`
- Breakout = two consecutive body-closes beyond the ORB edge (close outside the range; candle color irrelevant); latches via `bear_brk` / `bull_brk`
- Revisit is gated on `bear_brk[1]` / `bull_brk[1]` so the breakout candle itself can't satisfy it
- Floor/ceiling locks at the moment both revisits complete, then keeps moving: re-syncs to each new red→green / green→red pair (up or down) AND bumps via piercing wicks (down for floor, up for ceiling) — most recent event per bar wins
- Bearish = close < open, Bullish = close > open
- Entry = close, SL = open (works for both short and long trigger candles)
- **One trade per day, all instruments** — `max_trades = 1`; entry gated on `entry_count < max_trades`. No per-instrument settings.
- **Trading window** — `time_ok = et_hhmm >= 800 and et_hhmm < 1200`; entries only between 8:00 AM and 12:00 PM ET
- **Drawing suppression** — `day_complete` flips true on `pos_closed`; while true, the liquidity / floor / ceiling lines and volume highlight stop drawing (ORB box + midpoint are NOT gated and keep extending)
- Setup state still resets at entry and on position close, but with a 1-trade cap no re-entry fires within a day
- Session reset at 18:00 ET (futures new session) clears all state (incl. `day_complete`)
- `request.security()` calls at top level (not inside if blocks) to avoid Pine Script warnings
- All setup state managed with `var` variables; `max_lines_count=500` for the level lines

---

## 9. Pine Script v6 Migration Notes

Script is written in **Pine Script v6** (`//@version=6`). Key v5→v6 differences hit during implementation:

### Breaking Changes

| Issue | v5 | v6 Fix |
|-------|----|--------|
| `strategy()` commission/slippage | `commission_value=i_commission` worked | Requires `const float/int` — **inputs** not allowed, but **literal constants are**. Now baked in: `commission_value=0.72`, `commission_type=strategy.commission.cash_per_contract`. (Slippage still set in Properties.) |
| Multi-line boolean with `and` | `and` at start of continuation line worked | `and` at start OR end of line both fail (CE10013/CE10156). Use **intermediate bool variables** — one expression per line. |
| `math.avg()` | `math.avg(high, low)` | Removed. Use `(high + low) / 2`. |
| `barmerge.lookahead_on` | Used in `request.security()` | Deprecated. Use `barmerge.lookahead_off`. Safe here because `[1]` indexing already reads confirmed prior-bar data. |
| Local variable types | `gap_pct = ...` inside `if` block | Must be explicit: `float gap_pct = ...` |
| Loop variable types | `c_bull = close[i] > open[i]` inside `for` | Must be explicit: `bool c_bull = close[i] > open[i]` |
| Local risk variable | `_r = t_sl - t_entry` inside `if` | Must be explicit: `float _r = t_sl - t_entry` |

### Commission / Slippage Workflow
`strategy()` accepts compile-time **constants** (literals), just not inputs:
- **Commission** — baked in as a literal: `commission_type=strategy.commission.cash_per_contract`, `commission_value=0.72` ($0.72/side Tradovate MES/MNQ all-in). Applies in-backtest; overridable in Properties.
- **Initial capital / margin** — also baked: `initial_capital=50000`, `margin_long=5`, `margin_short=5`.
- **Slippage** — still set in Properties → ticks (not baked).

⚠️ Manual changes in the Properties tab **override** these baked-in defaults and persist. After updating the script, use Properties → **Defaults → Reset settings** so the new values take effect.

The `i_commission` / `i_slippage` inputs remain as display references only (tooltips explain this).

---

## 10. File Location

- Script: `orb_strategy_v3.pine`
- GitHub repo: `orb-strategy` (private)

---

## 11. V3 Script Notes

Signal engine rewritten 2026-06-10. The old grading/pullback-depth/pivot-sweep system was removed entirely.

- Runs on **5m chart**; ORB built natively from the three 5m bars in the 08:00–08:15 window (no `request.security` — avoids the cross-timeframe lookahead delay that captured the prior session's ORB on the execution chart)
- **Two tracked levels**: `gr_high` (most recent green→red pair high) and `rg_low` (most recent red→green pair low). Short structural = `gr_high`, short floor = `rg_low`; long structural = `rg_low`, long ceiling = `gr_high`
- **Breakout**: two consecutive bars both body-close beyond the ORB edge (close outside the range, any color); `bear_brk` / `bull_brk` latch for the session
- **Dual revisit**: structural-level touch + ORB-band touch, any order, gated on the prior bar's breakout latch
- **Floor / ceiling**: locks to `rg_low` / `gr_high` when both revisits complete, then keeps moving via two paths — re-sync to each new pair (`if floor_set and pair_rg: floor_short := rg_low`) and the piercing-wick bump (`low < floor AND close >= floor → floor := low`); same mirror for the ceiling
- **Entry**: directional bar body-closes beyond the valid floor/ceiling, gated on `entry_count < max_trades` (=1) and `time_ok` (8:00 AM–12:00 PM ET)
- **One trade per day**: `max_trades = 1`, global for all instruments. The old per-instrument max-trades / time-cutoff inputs and the instrument-detection block were removed
- **Drawing stops on exit**: `day_complete` flips true on `pos_closed`; liquidity / floor / ceiling lines + volume highlight are gated on `not day_complete`. ORB box + midpoint are NOT gated
- **BE management**: tracks up to 2 slots via `s1_*` / `s2_*` vars; BE moves stop to entry at 1:1 RR (with a 1-trade cap, only slot 1 is ever used); gated on `i_move_sl_to_be` (default ON) — when OFF, the original SL placed at entry is never moved
- **Custom SL/TP multiples**: gated on `i_custom_sl_tp` (default OFF). When ON, `_sl`/`_tp` at entry are computed as `close ∓ i_sl_mult * _r` / `close ± i_tp_mult * _r` instead of `open` / `close ± 2*_r`. Defaults (1.0 / 2.0) reproduce original behavior. Composes cleanly with `i_move_sl_to_be` — BE distance (`|entry - sl|`) scales with `i_sl_mult` automatically
- **EOD flatten**: `if et_hhmm == 1645 and strategy.position_size != 0: strategy.close_all()`, placed after entries and before BE management; forces any open position closed by 4:45 PM ET regardless of SL/TP/BE state
- **Reverse signals**: `i_reverse_signals` (default OFF). When ON, `buy_signal` (ceiling break) opens a SHORT and `sell_signal` (floor break) opens a LONG instead — direction is fully inverted. SL/TP are recomputed for the actual (reversed) direction rather than reused from the original-direction formulas: non-custom SL becomes `close ∓ R` (mirrors `open` not being on the correct side post-flip) and TP becomes `close ± 2R`; custom SL/TP multiples apply the same way, just on the flipped side. `s1_is_long`/`s2_is_long` reflect the actual position direction so BE management works unchanged
- `new_session = et_hour == 18 and et_minute == 0` — same as V1
- Commission: baked into `strategy()` ($0.72/side); slippage still set in Properties (see Section 9)
- **Removed**: grade inputs/colors/labels, `grade_str()`, `grade_col()`, pullback-depth vars, `ta.pivothigh/pivotlow` swing detection, swing markers, old C1/C2 breakout rules

### V3.1 additions (2026-06-16)

- **Selectable ORB duration**: `i_orb_minutes` dropdown (5/10/15/20/30/60 min). Engine window generalized from hardcoded `800/805/810` to minutes-of-day `[orb_start_min, orb_end_min)` where `orb_start_min=480`, `orb_end_min=480+i_orb_minutes`; finalizes on the `orb_end_min-5` bar. `in_orb_window` + the breakout gate auto-follow. Engine stays native 5m
- **Cross-timeframe ORB box**: box + midpoint now render on any chart timeframe. Values come from a self-contained `f_orb_box()` pulled via `request.security(syminfo.tickerid, "5", …, lookahead_off)` (own 18:00 reset → no prior-session capture, unlike the old build); drawn with `xloc.bar_time` + `time_close`. Trade engine + setup lines stay 5m-only (visual-only consistency). A strategy still *executes* on the chart's TF — trade/read results on 5m; other TFs are look-only
- **Backtest defaults baked into `strategy()`**: `initial_capital=50000`, `commission_type=cash_per_contract` + `commission_value=0.72` (Tradovate MES/MNQ all-in), `margin_long=5`/`margin_short=5` (so $50K affords 1 MNQ ≈ $61.7K notional). Reverses the old "no commission in `strategy()`" note — literal constants compile fine; only inputs don't. Manual Properties overrides persist → reset to Defaults to apply

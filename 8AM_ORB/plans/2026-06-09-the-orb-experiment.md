# The ORB Experiment — Implementation Plan

> **Revised 2026-06-10.** The original plan (grading A/B/C/2B/2C + pullback-depth + C1/C2 breakout + 3-bar pivot swings) was **superseded**. The signal engine was rewritten to a **breakout → dual-revisit → floor/ceiling-transfer** model with no grading. This document reflects the engine as actually shipped in `The_Orb_Experiment.pine`.
>
> **Renamed 2026-06-18.** The script (formerly `orb_strategy_v3.pine`) and this plan were renamed to **The ORB Experiment**; the optimizer was refined to R-based metrics + sample gates + heatmap (see Task 15).

**Goal:** `The_Orb_Experiment.pine` — a Pine Script v6 strategy that trades 5m ORB breakouts using a two-consecutive-close breakout, a dual-revisit gate (structural level + ORB band), and a transferring floor/ceiling trigger, with per-instrument trade limits and a level-line visual suite.

**Architecture:** Single Pine Script file on a 5m chart. The ORB is built natively from the three 5m bars in the 08:00–08:15 window (no `request.security` — that introduced a cross-timeframe lookahead delay that captured the prior session's ORB). State in `var` variables reset on the 18:00 ET session boundary. Entries/exits via `strategy.entry()` / `strategy.exit()` with `process_orders_on_close=true`, `pyramiding=2`, `max_lines_count=500`.

**Core model — two tracked candle-pair levels (every bar, incl. pre-8AM):**

- `gr_high` = high of the most recent **green→red** (bullish→bearish) pair.
- `rg_low`  = low of the most recent **red→green** (bearish→bullish) pair.

| | Structural / liquidity level (revisit) | Trigger level (body-close beyond) |
|---|---|---|
| **Short** | `gr_high` | floor = `rg_low` |
| **Long**  | `rg_low`  | ceiling = `gr_high` |

**Tech Stack:** Pine Script v6, TradingView strategy overlay, `request.security()` for prior-day H/L only, `ta.vwap` / `ta.sma`.

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `The_Orb_Experiment.pine` | Create / maintain | Entire strategy — single file |
| `CLAUDE.md` | Spec of record | Sections 1–11 describe the shipped engine |

---

## Task 1: Skeleton + All Inputs

- [ ] **Inputs:** Backtest Settings (commission/slippage, reference-only), ORB Settings (`i_orb_end`), Visuals (`i_show_levels`, `i_show_volume`, `i_show_vwap`, `i_show_pdhl`, `i_vol_lookback`), Colors (`i_col_orb_box`, `i_col_orb_mid`, `i_col_liq`, `i_col_floor`, `i_col_vwap`, `i_col_pdhl`, `i_col_vol_hi`). Trading rules are global constants (1 trade/day, 8 AM–12 PM ET) — no per-instrument inputs.

> **No** grade inputs, **no** grade colors, **no** swing-marker toggle/colors, **no** pivot-lookback input — all removed.

```pine
strategy("The ORB Experiment", overlay=true, process_orders_on_close=true,
    pyramiding=2, max_bars_back=500, max_lines_count=500,
    default_qty_type=strategy.fixed, default_qty_value=1)
```

- [ ] Paste into TradingView (MNQ1! 5m) → 0 compile errors, empty chart.
- [ ] Commit: `feat(v3): script skeleton and input declarations`

---

## Task 2: Time Utilities, Instrument Detection, ORB Build, State, Reset

- [ ] **Time:** `et_hour` / `et_minute` / `et_hhmm`; `in_orb_window = et_hhmm==800 or 805 or 810`; `new_session = et_hour==18 and et_minute==0`.
- [ ] **Trading window + cap (global):** `max_trades = 1`; `time_ok = et_hhmm >= 800 and et_hhmm < 1200`. (No instrument detection — rules are identical for every symbol.)
- [ ] **ORB build (native 5m):** accumulate `orb_high`/`orb_low` across the three 5m bars in the 08:00–08:15 window (`et_hhmm` 800/805/810); record `orb_anchor = bar_index` at 08:00; latch `orb_set` + `orb_mid` at the 08:10 close. No `request.security` — building from the chart's own bars avoids the cross-timeframe lookahead delay that otherwise captured the prior session's ORB.
- [ ] **State variables** (the new set):

```pine
// ORB
var bool orb_set = false
var float orb_high = na, orb_low = na, orb_mid = na

// Candle-pair levels (tracked every bar, incl. pre-8AM)
var float gr_high = na   // high of most recent green→red pair
var float rg_low  = na   // low  of most recent red→green pair

// Breakout (two consecutive body-closes beyond ORB)
var bool bear_brk = false, bull_brk = false

// Dual-revisit gates
var bool rev_struct_short = false, rev_orb_short = false
var bool rev_struct_long  = false, rev_orb_long  = false

// Armed + locked trigger levels
var bool  armed_short = false, armed_long = false
var bool  floor_set = false,   ceil_set = false
var float floor_short = na,    ceil_long = na

// orb_anchor = bar_index of the 08:00 bar (box left edge)
// Counter + BE slots (s1_*, s2_*) + visual objects (orb_box_obj, orb_mid_line,
//   liq_line_s, floor_line_s, liq_line_l, ceil_line_l)
var int orb_anchor = na
var int entry_count = 0
```

- [ ] **Session reset (`if new_session`)** clears every var above (levels → na, flags → false, lines → na, `entry_count := 0`).
- [ ] **ORB build:** at `et_hhmm==800` start `orb_high/low` + record `orb_anchor`; at 805/810 fold in `max(high)`/`min(low)`; at 810 set `orb_mid` and latch `orb_set := true`. Downstream logic is gated on `not in_orb_window`, so the build bars never trigger signals.
- [ ] Commit: `feat(v3): time utils, native ORB build, state vars, session reset`

---

## Task 3: Candle-Pair Tracking

Tracks the two levels every bar (no `orb_set` gate) so pre-8AM levels populate the fallback.

```pine
bool prev_bull = close[1] > open[1]
bool prev_bear = close[1] < open[1]
bool cur_bull  = close > open
bool cur_bear  = close < open
bool pair_gr = prev_bull and cur_bear   // green → red
bool pair_rg = prev_bear and cur_bull   // red → green
if pair_gr
    gr_high := math.max(high, high[1])
if pair_rg
    rg_low  := math.min(low,  low[1])
```

- [ ] Commit: `feat(v3): green→red / red→green candle-pair level tracking`

---

## Task 4: Breakout — Two Consecutive Body-Closes Beyond ORB

Replaces the old C1/C2 logic. Two consecutive body-closes beyond the ORB edge — **candle color does not matter**, only that the close is outside the range. `bear_brk` / `bull_brk` latch for the session.

```pine
if orb_set and not in_orb_window
    bool bear_now  = close   < orb_low
    bool bear_prev = close[1] < orb_low
    bool bull_now  = close   > orb_high
    bool bull_prev = close[1] > orb_high
    if not bear_brk and bear_now and bear_prev
        bear_brk := true
    if not bull_brk and bull_now and bull_prev
        bull_brk := true
```

- [ ] Verify with a temporary `plotshape(bear_brk, ...)`, then remove.
- [ ] Commit: `feat(v3): two-consecutive-close breakout latch`

---

## Task 5: Dual Revisit (Structural Level + ORB Band)

Gated on `bear_brk[1]` / `bull_brk[1]` so the breakout candle itself can't satisfy the revisit. Order-independent; each gate latches once touched.

```pine
if bear_brk[1]
    if not rev_struct_short and not na(gr_high) and high >= gr_high
        rev_struct_short := true
    if not rev_orb_short and high >= orb_low
        rev_orb_short := true

if bull_brk[1]
    if not rev_struct_long and not na(rg_low) and low <= rg_low
        rev_struct_long := true
    if not rev_orb_long and low <= orb_high
        rev_orb_long := true
```

- [ ] Commit: `feat(v3): dual-revisit gate (structural + ORB band)`

---

## Task 6: Arm + Lock Trigger Level + Transfer

Arms when breakout + both revisits hold. Floor/ceiling locks to the most recent pair level, then keeps moving via **two** paths (most recent per bar governs): (a) re-sync to each new pair (`pair_rg` / `pair_gr` → up or down), and (b) a piercing wick (a wick beyond the level whose body does **not** close beyond — down for floor, up for ceiling).

```pine
if not armed_short and bear_brk and rev_struct_short and rev_orb_short
    armed_short := true
if armed_short and not floor_set and not na(rg_low)
    floor_short := rg_low
    floor_set   := true
if floor_set and pair_rg
    floor_short := rg_low
if floor_set and not na(floor_short) and low < floor_short and close >= floor_short
    floor_short := low

if not armed_long and bull_brk and rev_struct_long and rev_orb_long
    armed_long := true
if armed_long and not ceil_set and not na(gr_high)
    ceil_long := gr_high
    ceil_set  := true
if ceil_set and pair_gr
    ceil_long := gr_high
if ceil_set and not na(ceil_long) and high > ceil_long and close <= ceil_long
    ceil_long := high
```

- [ ] Commit: `feat(v3): arm + floor/ceiling lock + wick transfer`

---

## Task 7: Entry Trigger + Setup Reset

Entry when armed and a directional bar **body-closes** beyond the valid trigger level. On entry the setup resets (revisit flags + floor/ceiling + their lines) so re-entry needs a fresh revisit; `bear_brk`/`bull_brk` stay latched.

```pine
bool bull_bar = close > open
bool bear_bar = close < open

bool s_armed = armed_short and floor_set and not na(floor_short)
bool sell_signal = s_armed and bear_bar and close < floor_short and entry_count < max_trades and time_ok

bool l_armed = armed_long and ceil_set and not na(ceil_long)
bool buy_signal = l_armed and bull_bar and close > ceil_long and entry_count < max_trades and time_ok
```

- Entry = `close`; SL = `open` (trigger candle body open); R = `|close-open|`; TP = `close ∓ 2R`.
- Fill BE slot `s1_*` (else `s2_*`), `entry_count += 1`, then reset that side's `armed_*` / `rev_*` / `*_set` / level + clear its lines.

> No grade labels are drawn — entries use TradingView's built-in markers.

- [ ] Verify entries only fire after a completed setup; check Strategy Tester.
- [ ] Commit: `feat(v3): body-close entry trigger + per-entry setup reset`

---

## Task 8: Breakeven Management

Move SL to entry at 1:1 RR, per slot (`s1_*`, `s2_*`), via `strategy.exit(stop=entry, limit=tp)`; mark `be_done`.

```pine
if not na(s1_entry) and not s1_be_done and s1_id != ""
    float _r  = math.abs(s1_entry - s1_sl)
    bool  hit = s1_is_long ? high >= s1_entry + _r : low <= s1_entry - _r
    if hit
        strategy.exit(s1_id + "x", s1_id, stop=s1_entry, limit=s1_tp)
        s1_be_done := true
// (s2_* mirror)
```

- [ ] Commit: `feat(v3): breakeven management at 1:1 RR`

---

## Task 9: Position-Close Handling

On a mid-session close (TP or BE stop-out), clear both BE slots and reset both setups + their lines. Also set `day_complete := true` — with the 1-trade cap this ends trading and drawing for the day. Breakout latches stay true.

```pine
bool pos_closed = strategy.position_size == 0 and
                  strategy.position_size[1] != 0 and
                  not new_session
if pos_closed
    day_complete := true
    // clear s1_*/s2_*, armed_*, rev_*, *_set, floor_short/ceil_long,
    // and liq_line_*/floor_line_s/ceil_line_l (→ na)
```

- [ ] Commit: `feat(v3): position-close reset + day_complete`

---

## Task 10: ORB Visuals + Level Lines

- [ ] **ORB box + midpoint:** drawn on the ORB bar, extended right until `et_hhmm <= i_orb_end`.
- [ ] **Liquidity line** (per side): drawn after breakout while waiting for the structural revisit; stops once `rev_struct_*` latches.
- [ ] **Floor / ceiling line** (per side): drawn once locked; `set_y1/set_y2` follow the level as it transfers; `set_x2` extends right.
- [ ] All level lines gated on `i_show_levels` **and `not day_complete`** (drawing stops after the day's trade closes); ORB box + midpoint are NOT gated. Colors `i_col_liq` / `i_col_floor`. Lines are handed off to `na` (not deleted) on reset so history persists; `max_lines_count=500`.
- [ ] Commit: `feat(v3): ORB box/mid + liquidity & floor/ceiling level lines`

---

## Task 11: Reference Lines + Volume Context

- [ ] VWAP (`ta.vwap(hlc3)`), PDH/PDL (`request.security("D", high[1]/low[1])`) — all toggle-gated.
- [ ] Volume context: `bgcolor` highlight on entry bars where `volume > ta.sma(volume, i_vol_lookback)` and `i_show_volume` and `not day_complete`.
- [ ] Commit: `feat(v3): VWAP, PDH/PDL, volume-context highlight`

---

## Task 12: Polish, CLAUDE.md, Push

- [ ] Header comment block reflects the breakout → revisit → floor/ceiling engine.
- [ ] `CLAUDE.md` sections 2/3/6/7/8/11 match the shipped logic (done 2026-06-10).
- [ ] Full compile check on MNQ1! 5m.
- [ ] Final commit + `git push origin master`.

---

## Task 13: Selectable ORB Duration + Cross-Timeframe Box (2026-06-16)

- [x] **`i_orb_minutes` dropdown** (5/10/15/20/30/60 min). Engine window generalized from hardcoded `800/805/810` to minutes-of-day `[orb_start_min, orb_end_min)` (`orb_start_min=480`, `orb_end_min=480+i_orb_minutes`); finalizes on the `orb_end_min-5` bar. Engine stays native 5m.
- [x] **Cross-timeframe ORB box**: box + midpoint render on any chart TF via a self-contained `f_orb_box()` pulled through `request.security(syminfo.tickerid, "5", …, lookahead_off)` (own 18:00 reset → no prior-session capture); drawn with `xloc.bar_time` + `time_close`. Trade engine + setup lines stay 5m-only (visual-only consistency).

---

## Task 14: Backtest Defaults Baked Into `strategy()` (2026-06-16)

- [x] `initial_capital=50000`, `commission_type=cash_per_contract` + `commission_value=0.80` (Tradovate MES/MNQ all-in), `slippage=1`, `margin_long=5`/`margin_short=5` (so $50K affords 1 MNQ ≈ $61.7K notional). Literal constants compile fine in `strategy()`; only inputs don't.

---

## Task 15: SL/TP Optimizer — R-Based Metrics + Sample Gates + Heatmap (2026-06-18)

Advisory grid (`i_optimize`, default OFF) testing every SL×TP multiple combo (SL 0.2→3.0, TP 0.2→5.0, step 0.2 = 15×25 = 375) on the SAME entries the strategy takes. Self-contained sim in size-375 `var` arrays; opens one sim per combo on each real entry (`opt_fire` hook, capturing `opt_islong` so it respects Reverse Signals); advances open sims each bar; renders a **center** table on `barstate.islast or barstate.islastconfirmedhistory`. Table created ONCE (no delete+recreate — that blanked it, the `CW10015` symptom). Never changes the strategy's own trades.

- [x] **Fill model matches live engine**: SL/TP tick-snapped; entry = close ± slippage (market); TP = exact level (limit); SL/BE = level ± slippage (stop); EOD = close ± slippage; commission ×2. Intrabar SL/TP tie → SL (pessimistic). Run tester with **bar magnifier OFF** for closest agreement.
- [x] **R-based metrics** (replaced dollar goals that drifted to grid edges): `1R = |entry − initial SL| × pointvalue`, fixed at entry. Goals (`i_goal`), all computed every bar:
  - **Expectancy (R)** = `a_sumR / trades` (replaced "Most Profit")
  - **Calmar (R)** = `a_equityR / a_maxddR` (total R ÷ worst R-drawdown; favors interior plateaus)
  - **Win Rate %** = `100 × wins / trades`
- [x] **Sample gates** (user inputs, quant rule-of-thumb tooltips): `i_min_trades` (default 30) + `i_min_losses` (default 5). Eligible only if `trades ≥ min AND losses ≥ min`; ineligible cells render grey + `—\n(n=XX)`.
- [x] **Relative heatmap**: per-render min/max over eligible cells → green (best) → red (worst), fully opaque. **Plateau ★** when all 4 orthogonal neighbors within 10%. **Best cell** gets aqua bg + black text + `▶ ◀`.
- [x] **Chart dimming** when optimizer ON (`bgcolor(color.new(color.black, 70))`).
- [ ] **Open — sample size**: Essential plan gives ~28–29 total 5m trades, below the 30 gate. Grid *shape* is encouraging but not yet tradeable; needs more 5m history (plan upgrade or Python export). Higher chart TF does NOT add 5m samples (strategy re-executes on that TF).

---

## Self-Review: Spec Coverage Check

| Spec point | Covered In |
|---|---|
| 5m execution / ORB built natively from 5m bars (08:00–08:15) | Task 2 |
| Two-consecutive-close breakout (color-agnostic) | Task 4 |
| `gr_high` / `rg_low` pair tracking (+ pre-8AM fallback) | Task 3 |
| Dual revisit (structural + ORB band, any order) | Task 5 |
| Revisit can't be satisfied by the breakout candle | Task 5 (`*_brk[1]` gate) |
| Floor/ceiling lock at arming | Task 6 |
| Floor/ceiling transfer — re-sync to new pair + piercing-wick bump | Task 6 |
| Body-close entry trigger | Task 7 |
| Entry = close, SL = open, TP = 2R | Task 7 |
| Setup reset at entry (re-arm needs fresh revisit) | Task 7 |
| BE at 1:1 RR | Task 8 |
| Position-close reset + `day_complete` | Task 9 |
| One trade per day, all instruments (`max_trades=1`) | Task 2/7 |
| Trading window 8 AM–12 PM ET (`time_ok`) | Task 2/7 |
| Setup drawing stops after trade closes (`day_complete`) | Task 9/10 |
| ORB box → `i_orb_end`, midpoint line (never gated) | Task 10 |
| Liquidity + floor/ceiling level lines | Task 10 |
| VWAP / PDH-PDL / volume context (OFF by default) | Task 11 |
| All colors user-selectable | Task 1 |
| Session reset 18:00 ET | Task 2 |
| **Removed:** grading, pullback depth, pivot swings, swing markers, C1/C2, per-instrument rules + instrument detection | n/a |

# ORB Strategy V2 Implementation Plan

> **Revised 2026-06-10.** The original plan (grading A/B/C/2B/2C + pullback-depth + C1/C2 breakout + 3-bar pivot swings) was **superseded**. The signal engine was rewritten to a **breakout → dual-revisit → floor/ceiling-transfer** model with no grading. This document reflects the engine as actually shipped in `orb_strategy_v2.pine`.

**Goal:** `orb_strategy_v2.pine` — a Pine Script v6 strategy that trades 5m ORB breakouts using a two-consecutive-close breakout, a dual-revisit gate (structural level + ORB band), and a transferring floor/ceiling trigger, with per-instrument trade limits and a level-line visual suite.

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
| `orb_strategy_v2.pine` | Create / maintain | Entire V2 strategy — single file |
| `CLAUDE.md` | Spec of record | Sections 1–11 describe the shipped engine |

---

## Task 1: Skeleton + All Inputs

- [ ] **Inputs:** Backtest Settings (commission/slippage, reference-only), Instrument Rules (8 instruments × max-trades + cutoff), ORB Settings (`i_orb_end`), Visuals (`i_show_levels`, `i_show_volume`, `i_show_vwap`, `i_show_pdhl`, `i_vol_lookback`), Colors (`i_col_orb_box`, `i_col_orb_mid`, `i_col_liq`, `i_col_floor`, `i_col_vwap`, `i_col_pdhl`, `i_col_vol_hi`).

> **No** grade inputs, **no** grade colors, **no** swing-marker toggle/colors, **no** pivot-lookback input — all removed.

```pine
strategy("ORB Strategy V2", overlay=true, process_orders_on_close=true,
    pyramiding=2, max_bars_back=500, max_lines_count=500,
    default_qty_type=strategy.fixed, default_qty_value=1)
```

- [ ] Paste into TradingView (MNQ1! 5m) → 0 compile errors, empty chart.
- [ ] Commit: `feat(v2): script skeleton and input declarations`

---

## Task 2: Time Utilities, Instrument Detection, ORB Build, State, Reset

- [ ] **Time:** `et_hour` / `et_minute` / `et_hhmm`; `in_orb_window = et_hhmm==800 or 805 or 810`; `new_session = et_hour==18 and et_minute==0`.
- [ ] **Instrument detection + limits:** `inst_max` / `inst_cut` ternary by ticker; `time_ok = et_hhmm < inst_cut`.
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
- [ ] Commit: `feat(v2): time utils, native ORB build, state vars, session reset`

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

- [ ] Commit: `feat(v2): green→red / red→green candle-pair level tracking`

---

## Task 4: Breakout — Two Consecutive Body-Closes Beyond ORB

Replaces the old C1/C2 logic. `bear_brk` / `bull_brk` latch for the session.

```pine
if orb_set and not in_orb_window
    bool bear_now  = close   < orb_low  and close   < open
    bool bear_prev = close[1] < orb_low and close[1] < open[1]
    bool bull_now  = close   > orb_high and close   > open
    bool bull_prev = close[1] > orb_high and close[1] > open[1]
    if not bear_brk and bear_now and bear_prev
        bear_brk := true
    if not bull_brk and bull_now and bull_prev
        bull_brk := true
```

- [ ] Verify with a temporary `plotshape(bear_brk, ...)`, then remove.
- [ ] Commit: `feat(v2): two-consecutive-close breakout latch`

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

- [ ] Commit: `feat(v2): dual-revisit gate (structural + ORB band)`

---

## Task 6: Arm + Lock Trigger Level + Transfer

Arms when breakout + both revisits hold. Floor/ceiling locks to the most recent pair level, then transfers only via piercing wicks (a wick beyond the level whose body does **not** close beyond).

```pine
if not armed_short and bear_brk and rev_struct_short and rev_orb_short
    armed_short := true
if armed_short and not floor_set and not na(rg_low)
    floor_short := rg_low
    floor_set   := true
if floor_set and not na(floor_short) and low < floor_short and close >= floor_short
    floor_short := low

if not armed_long and bull_brk and rev_struct_long and rev_orb_long
    armed_long := true
if armed_long and not ceil_set and not na(gr_high)
    ceil_long := gr_high
    ceil_set  := true
if ceil_set and not na(ceil_long) and high > ceil_long and close <= ceil_long
    ceil_long := high
```

- [ ] Commit: `feat(v2): arm + floor/ceiling lock + wick transfer`

---

## Task 7: Entry Trigger + Setup Reset

Entry when armed and a directional bar **body-closes** beyond the valid trigger level. On entry the setup resets (revisit flags + floor/ceiling + their lines) so re-entry needs a fresh revisit; `bear_brk`/`bull_brk` stay latched.

```pine
bool bull_bar = close > open
bool bear_bar = close < open

bool s_armed = armed_short and floor_set and not na(floor_short)
bool sell_signal = s_armed and bear_bar and close < floor_short and entry_count < inst_max and time_ok

bool l_armed = armed_long and ceil_set and not na(ceil_long)
bool buy_signal = l_armed and bull_bar and close > ceil_long and entry_count < inst_max and time_ok
```

- Entry = `close`; SL = `open` (trigger candle body open); R = `|close-open|`; TP = `close ∓ 2R`.
- Fill BE slot `s1_*` (else `s2_*`), `entry_count += 1`, then reset that side's `armed_*` / `rev_*` / `*_set` / level + clear its lines.

> No grade labels are drawn — entries use TradingView's built-in markers.

- [ ] Verify entries only fire after a completed setup; check Strategy Tester.
- [ ] Commit: `feat(v2): body-close entry trigger + per-entry setup reset`

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

- [ ] Commit: `feat(v2): breakeven management at 1:1 RR`

---

## Task 9: Position-Close Handling (Re-Entry Reset)

On a mid-session close (TP or BE stop-out), clear both BE slots **and** reset both setups + their lines so a fresh setup can build for re-entry. Breakout latches stay true.

```pine
bool pos_closed = strategy.position_size == 0 and
                  strategy.position_size[1] != 0 and
                  not new_session
if pos_closed
    // clear s1_*/s2_*, armed_*, rev_*, *_set, floor_short/ceil_long,
    // and liq_line_*/floor_line_s/ceil_line_l (→ na)
```

- [ ] Commit: `feat(v2): position-close reset for clean re-entry`

---

## Task 10: ORB Visuals + Level Lines

- [ ] **ORB box + midpoint:** drawn on the ORB bar, extended right until `et_hhmm <= i_orb_end`.
- [ ] **Liquidity line** (per side): drawn after breakout while waiting for the structural revisit; stops once `rev_struct_*` latches.
- [ ] **Floor / ceiling line** (per side): drawn once locked; `set_y1/set_y2` follow the level as it transfers; `set_x2` extends right.
- [ ] All level lines gated on `i_show_levels`; colors `i_col_liq` / `i_col_floor`. Lines are handed off to `na` (not deleted) on reset so history persists; `max_lines_count=500`.
- [ ] Commit: `feat(v2): ORB box/mid + liquidity & floor/ceiling level lines`

---

## Task 11: Reference Lines + Volume Context

- [ ] VWAP (`ta.vwap(hlc3)`), PDH/PDL (`request.security("D", high[1]/low[1])`) — all toggle-gated.
- [ ] Volume context: `bgcolor` highlight on entry bars where `volume > ta.sma(volume, i_vol_lookback)` and `i_show_volume`.
- [ ] Commit: `feat(v2): VWAP, PDH/PDL, volume-context highlight`

---

## Task 12: Polish, CLAUDE.md, Push

- [ ] Header comment block reflects the breakout → revisit → floor/ceiling engine.
- [ ] `CLAUDE.md` sections 2/3/6/7/8/11 match the shipped logic (done 2026-06-10).
- [ ] Full compile check on MNQ1! 5m.
- [ ] Final commit + `git push origin master`.

---

## Self-Review: Spec Coverage Check

| Spec point | Covered In |
|---|---|
| 5m execution / ORB built natively from 5m bars (08:00–08:15) | Task 2 |
| Two-consecutive-close breakout | Task 4 |
| `gr_high` / `rg_low` pair tracking (+ pre-8AM fallback) | Task 3 |
| Dual revisit (structural + ORB band, any order) | Task 5 |
| Revisit can't be satisfied by the breakout candle | Task 5 (`*_brk[1]` gate) |
| Floor/ceiling lock at arming | Task 6 |
| Floor/ceiling wick transfer | Task 6 |
| Body-close entry trigger | Task 7 |
| Entry = close, SL = open, TP = 2R | Task 7 |
| Setup reset at entry (re-arm needs fresh revisit) | Task 7 |
| BE at 1:1 RR (2 slots) | Task 8 |
| Re-entry reset on position close | Task 9 |
| Scale-in (`pyramiding=2`) | Task 7/9 |
| Per-instrument max trades / cutoff | Task 1/2 |
| ORB box → `i_orb_end`, midpoint line | Task 10 |
| Liquidity + floor/ceiling level lines | Task 10 |
| VWAP / PDH-PDL / volume context (OFF by default) | Task 11 |
| All colors user-selectable | Task 1 |
| Session reset 18:00 ET | Task 2 |
| **Removed:** grading, pullback depth, pivot swings, swing markers, C1/C2 | n/a |

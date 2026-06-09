# ORB Strategy V2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `orb_strategy_v2.pine` — a Pine Script v6 strategy that trades 5m ORB breakouts with validated C1/C2 breakout logic, 3-bar pivot swing detection, liquidity sweep gating, 5-grade setup classification, per-instrument trade limits, and full visual suite.

**Architecture:** Single Pine Script file running on 5m chart. 15m ORB data fetched via `request.security()`. State managed with `var` variables reset on 18:00 ET session boundary. All entry/exit via `strategy.entry()` / `strategy.exit()` with `process_orders_on_close=true` and `pyramiding=2`.

**Tech Stack:** Pine Script v6, TradingView strategy overlay, `request.security()` for 15m data, `ta.vwap` / `ta.sma` / pivot logic built-in.

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `orb_strategy_v2.pine` | Create | Entire V2 strategy — single file |
| `ORB_STRATEGY.md` | Reference only | Spec — do not modify during implementation |
| `CLAUDE.md` | Append after completion | Add V2 notes to Section 10 |

---

## Task 1: Script Skeleton + All Inputs

**Files:**
- Create: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Create the file with strategy declaration and all input groups**

```pine
//@version=6

// ─────────────────────────────────────────────────────────────────────────────
//  ORB STRATEGY V2
//  Timeframe : 5m (execution) + 15m (ORB source via request.security)
//  Session   : 8:00 AM ET ORB | 18:00 ET reset
//  Instruments: NQ, MNQ, ES, MES, YM, MYM, GC, MGC
// ─────────────────────────────────────────────────────────────────────────────

// ── Backtest Settings ────────────────────────────────────────────────────────
i_commission = input.float(
    2.00, "Commission Per Side ($)", minval=0, step=0.25,
    group="⚙️ Backtest Settings",
    tooltip="Reference only — Pine Script v6 requires constant values in strategy(). Set via Strategy Settings → Properties → Commission.")

i_slippage = input.int(
    1, "Slippage (Ticks)", minval=0,
    group="⚙️ Backtest Settings",
    tooltip="Reference only. Set via Strategy Settings → Properties → Slippage.")

// ── Instrument Rules ─────────────────────────────────────────────────────────
i_max_nq   = input.int(2,    "NQ  — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_nq   = input.int(1600, "NQ  — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_mnq  = input.int(2,    "MNQ — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_mnq  = input.int(1600, "MNQ — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_es   = input.int(2,    "ES  — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_es   = input.int(1600, "ES  — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_mes  = input.int(2,    "MES — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_mes  = input.int(1600, "MES — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_ym   = input.int(2,    "YM  — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_ym   = input.int(1600, "YM  — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_mym  = input.int(2,    "MYM — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_mym  = input.int(1600, "MYM — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_gc   = input.int(2,    "GC  — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_gc   = input.int(1600, "GC  — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")
i_max_mgc  = input.int(2,    "MGC — Max Trades/Session", minval=1, group="📊 Instrument Rules")
i_cut_mgc  = input.int(1600, "MGC — Time Cutoff (HHMM ET)", minval=0, maxval=2359, group="📊 Instrument Rules")

// ── ORB Settings ─────────────────────────────────────────────────────────────
i_orb_end = input.int(1700, "ORB Box End Time (HHMM ET)", minval=800, maxval=2359,
    group="📐 ORB Settings",
    tooltip="ORB box and midpoint line extend rightward until this time. Default 17:00 ET.")

// ── Setup Grades ─────────────────────────────────────────────────────────────
i_grade_a  = input.bool(true, "Enable Grade A",  group="🏆 Setup Grades",
    tooltip="Deep pullback (through full ORB zone) + liquidity sweep. Highest quality.")
i_grade_b  = input.bool(true, "Enable Grade B",  group="🏆 Setup Grades",
    tooltip="Mid pullback (ORB midpoint touch) + liquidity sweep.")
i_grade_c  = input.bool(true, "Enable Grade C",  group="🏆 Setup Grades",
    tooltip="Shallow pullback (ORB boundary touch) + liquidity sweep.")
i_grade_2b = input.bool(true, "Enable Grade 2B", group="🏆 Setup Grades",
    tooltip="Mid pullback, no liquidity sweep.")
i_grade_2c = input.bool(true, "Enable Grade 2C", group="🏆 Setup Grades",
    tooltip="Shallow pullback, no liquidity sweep.")

// ── Visuals ───────────────────────────────────────────────────────────────────
i_show_swing  = input.bool(true,  "Show Swing High/Low Markers", group="👁 Visuals")
i_show_volume = input.bool(false, "Show Volume Context",         group="👁 Visuals")
i_show_vwap   = input.bool(false, "Show VWAP",                   group="👁 Visuals")
i_show_pdhl   = input.bool(false, "Show Prior Day High/Low",     group="👁 Visuals")
i_vol_lookback = input.int(20, "Volume Avg Lookback (bars)", minval=5, maxval=100, group="👁 Visuals",
    tooltip="Number of bars used to compute average volume for context display.")

// ── Colors ────────────────────────────────────────────────────────────────────
i_col_orb_box  = input.color(color.new(color.blue,  80), "ORB Box",          group="🎨 Colors")
i_col_orb_mid  = input.color(color.new(color.blue,  30), "ORB Midpoint",     group="🎨 Colors")
i_col_swing_h  = input.color(color.new(color.lime,  0),  "Swing High",       group="🎨 Colors")
i_col_swing_l  = input.color(color.new(color.red,   0),  "Swing Low",        group="🎨 Colors")
i_col_grade_a  = input.color(color.new(color.green, 0),  "Grade A Label",    group="🎨 Colors")
i_col_grade_b  = input.color(color.new(color.lime,  0),  "Grade B Label",    group="🎨 Colors")
i_col_grade_c  = input.color(color.new(color.yellow,0),  "Grade C Label",    group="🎨 Colors")
i_col_grade_2b = input.color(color.new(color.orange,0),  "Grade 2B Label",   group="🎨 Colors")
i_col_grade_2c = input.color(color.new(color.red,   0),  "Grade 2C Label",   group="🎨 Colors")
i_col_vwap     = input.color(color.new(color.purple,0),  "VWAP",             group="🎨 Colors")
i_col_pdhl     = input.color(color.new(color.gray,  0),  "Prior Day H/L",    group="🎨 Colors")
i_col_vol_hi   = input.color(color.new(color.yellow,0),  "Volume Spike",     group="🎨 Colors")

strategy("ORB Strategy V2", overlay=true, process_orders_on_close=true,
     pyramiding=2, max_bars_back=500,
     default_qty_type=strategy.fixed, default_qty_value=1)
```

- [ ] **Step 2: Paste into TradingView Pine Script editor on MNQ1! 5m chart**

Expected: 0 compile errors. Strategy loads with empty chart (no signals yet — no logic written).

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): script skeleton and all input declarations"
```

---

## Task 2: Time Utilities, ORB Data, Session Reset, State Variables

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine` — append after inputs, before `strategy()` call

- [ ] **Step 1: Add time utilities and instrument detection (append after inputs, before strategy() call)**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  TIME UTILITIES
// ─────────────────────────────────────────────────────────────────────────────

int et_hour   = hour(time,   "America/New_York")
int et_minute = minute(time, "America/New_York")
int et_hhmm   = et_hour * 100 + et_minute

bool is_orb_bar  = et_hour == 8 and et_minute == 0
bool new_session = ta.change(time("D", "1800-1801", "America/New_York")) != 0

// ─────────────────────────────────────────────────────────────────────────────
//  INSTRUMENT DETECTION
// ─────────────────────────────────────────────────────────────────────────────

bool is_mnq = str.contains(syminfo.ticker, "MNQ")
bool is_nq  = str.contains(syminfo.ticker, "NQ")  and not is_mnq
bool is_mes = str.contains(syminfo.ticker, "MES")
bool is_es  = str.contains(syminfo.ticker, "ES")  and not is_mes
bool is_mym = str.contains(syminfo.ticker, "MYM")
bool is_ym  = str.contains(syminfo.ticker, "YM")  and not is_mym
bool is_mgc = str.contains(syminfo.ticker, "MGC")
bool is_gc  = str.contains(syminfo.ticker, "GC")  and not is_mgc

int  inst_max_trades = is_nq  ? i_max_nq  : is_mnq ? i_max_mnq :
                       is_es  ? i_max_es  : is_mes ? i_max_mes :
                       is_ym  ? i_max_ym  : is_mym ? i_max_mym :
                       is_gc  ? i_max_gc  : is_mgc ? i_max_mgc : i_max_nq

int  inst_cutoff     = is_nq  ? i_cut_nq  : is_mnq ? i_cut_mnq :
                       is_es  ? i_cut_es  : is_mes ? i_cut_mes :
                       is_ym  ? i_cut_ym  : is_mym ? i_cut_mym :
                       is_gc  ? i_cut_gc  : is_mgc ? i_cut_mgc : i_cut_nq

bool time_ok = et_hhmm < inst_cutoff

// ─────────────────────────────────────────────────────────────────────────────
//  15M ORB DATA  (evaluated in 15m context, persists on 5m chart)
// ─────────────────────────────────────────────────────────────────────────────

float orb_high_raw = request.security(syminfo.tickerid, "15",
    ta.valuewhen(
        hour(time, "America/New_York") == 8 and minute(time, "America/New_York") == 0,
        high, 0),
    barmerge.gaps_off, barmerge.lookahead_off)

float orb_low_raw  = request.security(syminfo.tickerid, "15",
    ta.valuewhen(
        hour(time, "America/New_York") == 8 and minute(time, "America/New_York") == 0,
        low, 0),
    barmerge.gaps_off, barmerge.lookahead_off)
```

- [ ] **Step 2: Add all state variables**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  STATE VARIABLES
// ─────────────────────────────────────────────────────────────────────────────

// ORB
var bool  orb_set     = false
var float orb_high    = na
var float orb_low     = na
var float orb_mid     = na

// Breakout validation
var bool  brk_c1_long    = false
var bool  brk_c1_short   = false
var bool  brk_c2_long    = false
var bool  brk_c2_short   = false
var bool  brk_valid_long  = false
var bool  brk_valid_short = false

// Pullback depth: 0=none, 1=shallow (boundary), 2=mid, 3=deep (through ORB)
var int   pull_depth_long  = 0
var int   pull_depth_short = 0

// Liquidity sweep
var bool  liq_long  = false
var bool  liq_short = false

// Swing levels (3-bar pivot, updated each bar)
var float swing_high = na
var float swing_low  = na

// BOS levels (updated during pullback phase — most recent pivot in pullback)
var float bos_lvl_long  = na
var float bos_lvl_short = na

// Trade tracking
var int   entry_count   = 0

// BE management (tracks up to 2 open entries)
var float e1_entry   = na
var float e1_sl      = na
var float e1_tp      = na
var bool  e1_be_done = false
var bool  e1_long    = false

var float e2_entry   = na
var float e2_sl      = na
var float e2_tp      = na
var bool  e2_be_done = false
var bool  e2_long    = false

// Visual boxes (persist across bars)
var box   orb_box    = na
var line  orb_midline = na
```

- [ ] **Step 3: Add session reset**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  SESSION RESET  (18:00 ET)
// ─────────────────────────────────────────────────────────────────────────────

if new_session
    orb_set          := false
    orb_high         := na
    orb_low          := na
    orb_mid          := na
    brk_c1_long      := false
    brk_c1_short     := false
    brk_c2_long      := false
    brk_c2_short     := false
    brk_valid_long   := false
    brk_valid_short  := false
    pull_depth_long  := 0
    pull_depth_short := 0
    liq_long         := false
    liq_short        := false
    swing_high       := na
    swing_low        := na
    bos_lvl_long     := na
    bos_lvl_short    := na
    entry_count      := 0
    e1_entry         := na
    e1_sl            := na
    e1_tp            := na
    e1_be_done       := false
    e1_long          := false
    e2_entry         := na
    e2_sl            := na
    e2_tp            := na
    e2_be_done       := false
    e2_long          := false

// ─────────────────────────────────────────────────────────────────────────────
//  ORB LATCH  (set once on the 8:00 AM bar)
// ─────────────────────────────────────────────────────────────────────────────

if is_orb_bar and not orb_set
    orb_set  := true
    orb_high := orb_high_raw
    orb_low  := orb_low_raw
    orb_mid  := (orb_high_raw + orb_low_raw) / 2
```

- [ ] **Step 4: Paste updated file into TradingView**

Expected: 0 errors. ORB box not yet drawn — that comes in Task 3.

- [ ] **Step 5: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): time utils, ORB data fetch, state vars, session reset"
```

---

## Task 3: Swing High/Low Detection + ORB Box Visual

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine` — append logic and visuals

- [ ] **Step 1: Add 3-bar pivot swing detection**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  SWING HIGH / LOW  (3-bar pivot, session-scoped)
// ─────────────────────────────────────────────────────────────────────────────

bool pivot_h = high[1] > high[0] and high[1] > high[2]
bool pivot_l = low[1]  < low[0]  and low[1]  < low[2]

if orb_set and pivot_h
    swing_high := high[1]

if orb_set and pivot_l
    swing_low := low[1]
```

- [ ] **Step 2: Add ORB box and midpoint line drawing**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  ORB VISUALS  (box + midpoint line)
// ─────────────────────────────────────────────────────────────────────────────

// Draw box on the bar ORB is first set
if is_orb_bar and orb_set and na(orb_box)
    orb_box     := box.new(bar_index, orb_high, bar_index + 1, orb_low,
                            border_color=i_col_orb_box,
                            bgcolor=i_col_orb_box)
    orb_midline := line.new(bar_index, orb_mid, bar_index + 1, orb_mid,
                             color=i_col_orb_mid,
                             style=line.style_dashed, width=1)

// Extend box and midpoint line rightward until orb_end_time
if orb_set and not na(orb_box) and et_hhmm <= i_orb_end
    box.set_right(orb_box, bar_index + 1)
    line.set_x2(orb_midline, bar_index + 1)
```

- [ ] **Step 3: Add swing marker visuals**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  SWING MARKERS  (triangles offset above/below wick)
// ─────────────────────────────────────────────────────────────────────────────

float tick_offset = syminfo.mintick * 8

if i_show_swing and orb_set and pivot_h
    label.new(bar_index[1], high[1] + tick_offset,
              "▲", style=label.style_none,
              textcolor=i_col_swing_h, size=size.small)

if i_show_swing and orb_set and pivot_l
    label.new(bar_index[1], low[1] - tick_offset,
              "▼", style=label.style_none,
              textcolor=i_col_swing_l, size=size.small)
```

- [ ] **Step 4: Paste into TradingView on MNQ1! 5m chart**

Expected: ORB box appears on the 8:00 AM zone, extending to 17:00. Midpoint dashed line extends same width. Swing ▲/▼ markers appear on confirmed pivots throughout the session.

- [ ] **Step 5: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): swing detection, ORB box, midpoint line, swing markers"
```

---

## Task 4: Breakout Validation (C1, C2+, Wick Cleared)

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Add breakout validation logic**

C1 valid = close outside ORB AND close clears the wick extreme of the last [green,red] (buy) or [red,green] (sell) candle pair.
C2+ valid = both open AND close outside ORB, on any bar after C1.

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  BREAKOUT VALIDATION
// ─────────────────────────────────────────────────────────────────────────────

if orb_set and not is_orb_bar

    // ── C1 Long: close above ORB high AND clears last [green,red] pair high ──
    if not brk_c1_long and close > orb_high
        float pair_ext = na
        for i = 1 to 50
            bool g = close[i + 1] > open[i + 1]
            bool r = close[i]     < open[i]
            if g and r
                pair_ext := math.max(high[i], high[i + 1])
                break
        if not na(pair_ext) and close > pair_ext
            brk_c1_long := true

    // ── C1 Short: close below ORB low AND clears last [red,green] pair low ──
    if not brk_c1_short and close < orb_low
        float pair_ext = na
        for i = 1 to 50
            bool r = close[i + 1] < open[i + 1]
            bool g = close[i]     > open[i]
            if r and g
                pair_ext := math.min(low[i], low[i + 1])
                break
        if not na(pair_ext) and close < pair_ext
            brk_c1_short := true

    // ── C2+ Long: full body above ORB high (after C1) ──
    if brk_c1_long and not brk_c2_long
        bool c2_body_long = open > orb_high and close > orb_high
        if c2_body_long
            brk_c2_long := true

    // ── C2+ Short: full body below ORB low (after C1) ──
    if brk_c1_short and not brk_c2_short
        bool c2_body_short = open < orb_low and close < orb_low
        if c2_body_short
            brk_c2_short := true

    // ── Valid breakout ──
    if brk_c1_long  and brk_c2_long  and not brk_valid_long
        brk_valid_long  := true
    if brk_c1_short and brk_c2_short and not brk_valid_short
        brk_valid_short := true
```

- [ ] **Step 2: Paste into TradingView**

Expected: 0 errors. No visible change yet (breakout state is internal). Add a temporary `plotshape(brk_valid_long, "BO Long", shape.triangleup, location.abovebar, color.green)` to verify breakout fires correctly on the chart, then remove before committing.

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): C1/C2/wick-cleared breakout validation logic"
```

---

## Task 5: Pullback Depth Tracking, Liquidity Sweep, Setup Grade

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Add pullback depth tracking and liquidity sweep detection**

After a valid breakout, track how deep the pullback goes and whether it sweeps the last pivot swing level.

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  PULLBACK DEPTH + LIQUIDITY SWEEP
// ─────────────────────────────────────────────────────────────────────────────

if orb_set and not is_orb_bar

    // ── Long breakout pullback (price coming back down after ORB high break) ──
    if brk_valid_long
        // Depth 1 = shallow: wick touches ORB high (boundary)
        if pull_depth_long < 1 and low <= orb_high
            pull_depth_long := 1
        // Depth 2 = mid: wick touches ORB midpoint
        if pull_depth_long < 2 and low <= orb_mid
            pull_depth_long := 2
        // Depth 3 = deep: wick breaches through ORB low
        if pull_depth_long < 3 and low <= orb_low
            pull_depth_long := 3
        // Liquidity sweep: price revisits last confirmed swing low
        if not liq_long and not na(swing_low) and low <= swing_low
            liq_long := true

    // ── Short breakout pullback (price coming back up after ORB low break) ──
    if brk_valid_short
        // Depth 1 = shallow: wick touches ORB low (boundary)
        if pull_depth_short < 1 and high >= orb_low
            pull_depth_short := 1
        // Depth 2 = mid: wick touches ORB midpoint
        if pull_depth_short < 2 and high >= orb_mid
            pull_depth_short := 2
        // Depth 3 = deep: wick breaches through ORB high
        if pull_depth_short < 3 and high >= orb_high
            pull_depth_short := 3
        // Liquidity sweep: price revisits last confirmed swing high
        if not liq_short and not na(swing_high) and high >= swing_high
            liq_short := true
```

- [ ] **Step 2: Add setup grade determination function**

Grade is computed at BOS time from pullback depth and liquidity state.

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  SETUP GRADE DETERMINATION
// ─────────────────────────────────────────────────────────────────────────────

// Returns grade string ("A","B","C","2B","2C") or "" if invalid/disabled
grade_for(int depth, bool liq, bool is_long) =>
    string g = ""
    if depth == 3
        g := liq ? "A" : ""   // 2A = invalid, never trade
    else if depth == 2
        g := liq ? "B" : "2B"
    else if depth == 1
        g := liq ? "C" : "2C"
    bool enabled = (g == "A"  and i_grade_a)  or
                   (g == "B"  and i_grade_b)  or
                   (g == "C"  and i_grade_c)  or
                   (g == "2B" and i_grade_2b) or
                   (g == "2C" and i_grade_2c)
    enabled ? g : ""

// Grade color lookup
grade_color(string g) =>
    g == "A"  ? i_col_grade_a  :
    g == "B"  ? i_col_grade_b  :
    g == "C"  ? i_col_grade_c  :
    g == "2B" ? i_col_grade_2b :
    g == "2C" ? i_col_grade_2c : color.gray
```

- [ ] **Step 3: Update BOS level during pullback phase**

During the pullback phase (after valid breakout), update the BOS level from new pivot swings that form.

```pine
// Update BOS level from pivots that form DURING the pullback phase
if brk_valid_long  and pull_depth_long  > 0 and pivot_h
    bos_lvl_long  := high[1]

if brk_valid_short and pull_depth_short > 0 and pivot_l
    bos_lvl_short := low[1]
```

- [ ] **Step 4: Paste into TradingView**

Expected: 0 errors. No new visuals yet. Temporarily add `label.new(bar_index, high, str.tostring(pull_depth_long), style=label.style_none)` to verify depth tracking, then remove.

- [ ] **Step 5: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): pullback depth, liquidity sweep, setup grade logic"
```

---

## Task 6: BOS Entry Trigger + Instrument Rules

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Add BOS entry trigger and strategy calls**

Entry fires when: valid breakout + pullback exists + BOS level set + grade enabled + grade close + time OK + entry count < max.

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  BOS ENTRY TRIGGER
// ─────────────────────────────────────────────────────────────────────────────

float body_high = math.max(open, close)
float body_low  = math.min(open, close)
bool  bull_bar  = close > open
bool  bear_bar  = close < open

// BOS long: bull candle body close above bos_lvl_long
bool bos_long_cond = brk_valid_long  and pull_depth_long  > 0 and
                     not na(bos_lvl_long)  and bull_bar and body_high > bos_lvl_long

// BOS short: bear candle body close below bos_lvl_short
bool bos_short_cond = brk_valid_short and pull_depth_short > 0 and
                      not na(bos_lvl_short) and bear_bar and body_low < bos_lvl_short

// Grade + guards
string cur_grade_long  = grade_for(pull_depth_long,  liq_long,  true)
string cur_grade_short = grade_for(pull_depth_short, liq_short, false)

bool buy_signal  = bos_long_cond  and cur_grade_long  != "" and entry_count < inst_max_trades and time_ok
bool sell_signal = bos_short_cond and cur_grade_short != "" and entry_count < inst_max_trades and time_ok

// ── Fire long entry ──
if buy_signal
    float _r   = math.abs(close - open)
    float _sl  = open                    // body low of BOS candle
    float _tp  = close + 2.0 * _r       // 2:1 RR from close
    string _id = "L" + str.tostring(entry_count + 1)
    strategy.entry(_id, strategy.long)
    strategy.exit(_id + "x", _id, stop=_sl, limit=_tp)
    // Store for BE management
    if na(e1_entry)
        e1_entry   := close
        e1_sl      := _sl
        e1_tp      := _tp
        e1_be_done := false
        e1_long    := true
    else
        e2_entry   := close
        e2_sl      := _sl
        e2_tp      := _tp
        e2_be_done := false
        e2_long    := true
    entry_count := entry_count + 1
    // Draw grade label on entry candle
    label.new(bar_index, high + syminfo.mintick * 10, cur_grade_long,
              style=label.style_label_down, color=grade_color(cur_grade_long),
              textcolor=color.white, size=size.small)

// ── Fire short entry ──
if sell_signal
    float _r   = math.abs(close - open)
    float _sl  = open                    // body high of BOS candle
    float _tp  = close - 2.0 * _r       // 2:1 RR from close
    string _id = "S" + str.tostring(entry_count + 1)
    strategy.entry(_id, strategy.short)
    strategy.exit(_id + "x", _id, stop=_sl, limit=_tp)
    if na(e1_entry)
        e1_entry   := close
        e1_sl      := _sl
        e1_tp      := _tp
        e1_be_done := false
        e1_long    := false
    else
        e2_entry   := close
        e2_sl      := _sl
        e2_tp      := _tp
        e2_be_done := false
        e2_long    := false
    entry_count := entry_count + 1
    label.new(bar_index, low - syminfo.mintick * 10, cur_grade_short,
              style=label.style_label_up, color=grade_color(cur_grade_short),
              textcolor=color.white, size=size.small)
```

- [ ] **Step 2: Paste into TradingView on MNQ1! 5m**

Expected: strategy entries fire with grade labels (A/B/C/2B/2C) on BOS candles. Verify entries only appear after a validated breakout, not on random candles. Check the Strategy Tester tab shows trades.

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): BOS entry trigger, grade labels, instrument guards"
```

---

## Task 7: Breakeven Management

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Add BE management — move stop to entry when 1R profit reached**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  BREAKEVEN MANAGEMENT  (move SL to entry at 1:1 RR)
// ─────────────────────────────────────────────────────────────────────────────

// Entry 1 BE
if not na(e1_entry) and not e1_be_done
    float e1_r   = math.abs(e1_entry - e1_sl)
    bool  e1_hit = e1_long ? high >= e1_entry + e1_r : low <= e1_entry - e1_r
    if e1_hit
        string _id = "L" + str.tostring(na(e2_entry) ? entry_count : entry_count - 1)
        if e1_long
            strategy.exit(_id + "x", _id, stop=e1_entry, limit=e1_tp)
        else
            strategy.exit(_id + "x", _id, stop=e1_entry, limit=e1_tp)
        e1_be_done := true

// Entry 2 BE
if not na(e2_entry) and not e2_be_done
    float e2_r   = math.abs(e2_entry - e2_sl)
    bool  e2_hit = e2_long ? high >= e2_entry + e2_r : low <= e2_entry - e2_r
    if e2_hit
        string _id = "L" + str.tostring(entry_count)
        if e2_long
            strategy.exit(_id + "x", _id, stop=e2_entry, limit=e2_tp)
        else
            strategy.exit(_id + "x", _id, stop=e2_entry, limit=e2_tp)
        e2_be_done := true

// Clear slot when position is fully closed
if strategy.position_size == 0
    e1_entry   := na
    e1_sl      := na
    e1_tp      := na
    e1_be_done := false
    e2_entry   := na
    e2_sl      := na
    e2_tp      := na
    e2_be_done := false
```

- [ ] **Step 2: Paste and verify in TradingView**

Run a backtest on MNQ1! over a few months. In the list of closed trades, verify that stops that moved to BE show an exit at entry price (not original SL price) when BE-stopped trades appear.

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): breakeven management at 1:1 RR"
```

---

## Task 8: Secondary Entry Logic (Re-Entry + Scale-In)

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

The primary entry logic already supports re-entry and scale-in organically because:
- `brk_valid_long/short` stays `true` after the first entry (not reset)
- `bos_lvl_long/short` continues updating from new pivots during the pullback
- `entry_count < inst_max_trades` gate prevents exceeding the per-session limit
- `pyramiding=2` allows up to 2 simultaneous open entries

The only additional logic needed: **re-entry after BE stop-out** requires resetting the BOS level and pullback depth so a fresh setup forms.

- [ ] **Step 1: Add re-entry reset on BE stop-out**

Detect when position size goes from non-zero to zero mid-session (stopped at BE) and reset BOS level + pullback depth to allow a new setup.

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  RE-ENTRY RESET  (after BE stop-out, allow fresh BOS to form)
// ─────────────────────────────────────────────────────────────────────────────

// Detect position closed mid-session (not at end of day)
bool pos_just_closed = strategy.position_size == 0 and
                       strategy.position_size[1] != 0 and
                       not new_session and
                       entry_count < inst_max_trades

if pos_just_closed
    // Reset BOS levels and pullback state so new pivots form fresh targets
    bos_lvl_long  := na
    bos_lvl_short := na
    // Do NOT reset pull_depth or liq — those still reflect the overall setup
    // Do NOT reset brk_valid — breakout is still valid
    // New BOS level will be set from next pivot that forms
```

- [ ] **Step 2: Paste into TradingView and verify**

Test scenario: find a session on MNQ1! where price hits 1R, moves SL to BE, then gets stopped at BE. Confirm the strategy can fire a second entry on a new BOS after that point (if `entry_count < inst_max_trades`).

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): re-entry reset on BE stop-out for secondary entry logic"
```

---

## Task 9: Reference Lines, Volume Context (VWAP, PDH/PDL, Volume)

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine`

- [ ] **Step 1: Add VWAP, PDH/PDL, and volume context (append near end of file)**

```pine
// ─────────────────────────────────────────────────────────────────────────────
//  REFERENCE LINES
// ─────────────────────────────────────────────────────────────────────────────

// VWAP
float vwap_val = ta.vwap(hlc3)
plot(i_show_vwap ? vwap_val : na, "VWAP", color=i_col_vwap, linewidth=1)

// Prior Day High / Low
float pdh_raw = request.security(syminfo.tickerid, "D",
    high[1], barmerge.gaps_off, barmerge.lookahead_off)
float pdl_raw = request.security(syminfo.tickerid, "D",
    low[1],  barmerge.gaps_off, barmerge.lookahead_off)

plot(i_show_pdhl ? pdh_raw : na, "PDH", color=i_col_pdhl,
     style=plot.style_stepline, linewidth=1)
plot(i_show_pdhl ? pdl_raw : na, "PDL", color=i_col_pdhl,
     style=plot.style_stepline, linewidth=1)

// ─────────────────────────────────────────────────────────────────────────────
//  VOLUME CONTEXT  (visual only, not a filter)
// ─────────────────────────────────────────────────────────────────────────────

float vol_avg   = ta.sma(volume, i_vol_lookback)
bool  vol_spike = volume > vol_avg

// Highlight BOS candle background when volume is above average
bgcolor(i_show_volume and (buy_signal or sell_signal) and vol_spike ?
        color.new(i_col_vol_hi, 75) : na)
```

- [ ] **Step 2: Paste into TradingView**

Expected: VWAP appears when toggled ON. PDH/PDL step lines appear. Volume highlight shows a faint yellow background on BOS candles where volume exceeded the average.

- [ ] **Step 3: Commit**

```bash
git add orb_strategy_v2.pine
git commit -m "feat(v2): VWAP, PDH/PDL reference lines, volume context highlight"
```

---

## Task 10: Final Polish, CLAUDE.md Update, Push

**Files:**
- Modify: `TradingStrat/orb_strategy_v2.pine` — header comment clean-up
- Modify: `TradingStrat/CLAUDE.md` — append V2 section

- [ ] **Step 1: Add descriptive header comment at top of file**

```pine
// ═════════════════════════════════════════════════════════════════════════════
//  ORB Strategy V2  |  Pine Script v6
//  Chart     : 5-minute
//  ORB Source: 15-minute (via request.security)
//  Session   : 8:00 AM ET open  |  Reset: 18:00 ET
//  Instruments: NQ, MNQ, ES, MES, YM, MYM, GC, MGC
//
//  Entry logic:
//    1. Breakout validated: C1 close outside ORB + wick cleared,
//       C2+ full body outside ORB
//    2. Pullback: price returns toward ORB zone, sweeps prior swing (liquidity)
//    3. BOS: bull/bear body close beyond most recent pullback pivot
//    Grade assigned at entry (A/B/C/2B/2C) based on pullback depth + liq sweep
//
//  Trade mgmt: SL = BOS candle body open | TP = 2:1 RR | BE at 1:1 RR
//  Max entries per session: user-adjustable per instrument (default 2)
// ═════════════════════════════════════════════════════════════════════════════
```

- [ ] **Step 2: Append V2 section to CLAUDE.md**

Add to Section 10 of `TradingStrat/CLAUDE.md`:

```markdown
## 11. V2 Script Notes

- Script: `orb_strategy_v2.pine`
- Runs on 5m chart; 15m ORB sourced via `request.security()`
- Breakout = C1 close outside + wick cleared (last candle pair), C2+ full body outside
- Swing detection: 3-bar pivot (`high[1] > high[0] and high[1] > high[2]`)
- Pullback depth: 1=shallow (ORB boundary), 2=mid (ORB midpoint), 3=deep (through ORB)
- Liquidity sweep = price revisits last confirmed session pivot before BOS
- Setup 2A (deep + no liquidity) is strictly invalid — never fires
- `pyramiding=2` — supports re-entry + scale-in up to inst_max_trades per session
- BE management tracks up to 2 simultaneous open entries (e1/e2 vars)
- Commission/slippage: set in TradingView Properties tab (same as V1)
```

- [ ] **Step 3: Full compile check on MNQ1! 5m — verify end-to-end**

Checklist:
- [ ] 0 compile errors
- [ ] ORB box appears at 8:00 AM, extends to 17:00 ET
- [ ] ORB midpoint dashed line matches box width
- [ ] Swing ▲/▼ markers appear on confirmed pivots
- [ ] Grade labels appear on entry candles
- [ ] Strategy Tester shows trades with 2:1 exits and BE stops
- [ ] VWAP / PDH/PDL toggles work
- [ ] All color inputs take effect
- [ ] Instrument Rules panel shows 8 pairs of inputs

- [ ] **Step 4: Final commit and push**

```bash
git add orb_strategy_v2.pine CLAUDE.md
git commit -m "feat(v2): ORB Strategy V2 complete — 5m/15m hybrid, C1/C2 breakout, pivot swing detection, 5-grade setup classification"
git push origin master
```

---

## Self-Review: Spec Coverage Check

| Spec Section | Covered In |
|---|---|
| 5m execution / 15m ORB | Task 2 (request.security) |
| C1 close only outside | Task 4 |
| C2+ full body outside | Task 4 |
| Wick cleared — last candle pair | Task 4 |
| 3-bar pivot swing detection | Task 3 |
| Liquidity sweep replaces gap filter | Task 5 |
| Pullback depth (shallow/mid/deep) | Task 5 |
| Grade A/B/C/2B/2C classification | Task 5 |
| 2A strictly invalid | Task 5 (grade_for returns "") |
| Per-grade enable toggle | Task 5 (grade_for checks inputs) |
| BOS entry trigger | Task 6 |
| Grade label on entry candle | Task 6 |
| 2:1 TP, SL = BOS body open | Task 6 |
| BE at 1:1 RR | Task 7 |
| Re-entry after BE stop-out | Task 8 |
| Scale-in (pyramiding=2) | Task 8 (organic from entry logic) |
| Per-instrument max trades | Task 6 (inst_max_trades) |
| Per-instrument time cutoff | Task 2 (inst_cutoff) |
| All defaults = 2 trades / 16:00 | Task 1 (inputs) |
| ORB box extends to 17:00 ET | Task 3 |
| ORB box end time user-adjustable | Task 1 (i_orb_end input) |
| Swing markers above/below wick | Task 3 |
| Volume context OFF by default | Task 1 + Task 9 |
| VWAP OFF by default | Task 1 + Task 9 |
| PDH/PDL OFF by default | Task 1 + Task 9 |
| All colors user-selectable | Task 1 (16 color inputs) |
| Session reset 18:00 ET | Task 2 |
| GC/MGC same rules as others | Task 1 (all default 2/1600) |
| YM/MYM included | Task 1 + Task 2 |

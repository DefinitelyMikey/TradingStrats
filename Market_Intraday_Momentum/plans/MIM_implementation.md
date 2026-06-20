# MIM — Implementation & Validation Plan

Build record for the **Market Intraday Momentum** strategy. Spec: [`../CLAUDE.md`](../CLAUDE.md).
Two deliverables, **identical signal core**:

| File | Type | Job |
|---|---|---|
| [`../MIM_Backtest.pine`](../MIM_Backtest.pine) | `strategy{}` | Backtest: entries/exits, stops, sizing, commission, debug table |
| [`../MIM_Alerts.pine`](../MIM_Alerts.pine) | `indicator{}` | Live alerts: direction + entry/exit pings, JSON webhook message |

---

## Locked design decisions (from the grill)

1. **Order-fill model — `process_orders_on_close = false`.** Entries are *submitted* on the bar that closes at `T_LAST30` and **fill at the next bar's open**, so Mode B never transacts on the bar that completes its own signal → no lookahead. The session flatten (`strategy.close_all`) is submitted on the **2nd-to-last in-session bar** so it fills at the **last bar's open** → the position is provably flat for the final bar, on **both RTH-only and 24h charts**. No per-order POOC exists in Pine, so this single global setting is the lever.

2. **Native chart-TF anchor capture.** Each anchor price = the `close` of the bar whose `time_close` lands on the anchor minute (`P_open30`, `P_secondLast`, `P_last30`); `P_prevClose` is the prior session's captured close carried across the boundary with `var`. **No `request.security` for anchors** — the sibling ORB build documented that cross-TF `security` captured the *prior* session's window (a lookahead trap). This requires the chart TF to divide 30 → enforced by a hard guard.

3. **Hard `runtime.error` on an invalid TF** (non-intraday, sub-minute, or not dividing 30) in **both** scripts. Backtest validity is sacred; a silently-wrong TF is worse than a blank chart.

4. **`request.security` only for daily series** — ATR-TF stop source, VIX gate, trend lookback — every one with `[1]`/`[n]` offset + `lookahead=barmerge.lookahead_off`, so historical values never repaint.

5. **Realism baked as literals** (v6 forbids inputs in `strategy()`): `commission_value=0.80` cash/contract/side, `slippage=1`, `initial_capital=50000`, `margin_long=margin_short=5`. The `i_commission` / `i_slippage` inputs are **display references only** — change them in Properties to actually move the backtest, then reset to Defaults.

6. **Filter internals** (all default OFF = academic every-signal-day baseline):
   - **Vol filter:** proxy = 30-min `ATR` as %-of-price, sampled once/day at `T_OPEN30` into a native `var array`; threshold = `array.percentile_linear_interpolation` over the **prior** `volLookbackDays` (today pushed *after* the decision → no lookahead); warm-up passes.
   - **VIX gate:** `request.security(i_vixSym,"D",close[1],…)`; gold → set `i_vixSym = CBOE:GVZ` or leave off.
   - **Trend filter:** `trendDir = sign(close / dailyClose[252] − 1)`; flat year (0) **blocks** (safety).
   - **Stops:** ATR + fixed-tick, the **tighter governs** (`math.min`); stop anchored to the actual fill via `strategy.position_avg_price` once the position opens.
   - **Sizing:** RiskBased = `floor(riskDollars / (stopDist × pointvalue))` capped at `maxContracts`; **no stop → fall back to fixed qty + warning label**; computed `qty < 1` → skip the trade (still marks the day used).

---

## 3 honest deviations from the spec

| # | Spec says | Build does | Why |
|---|---|---|---|
| 1 | Mode A "full 30-minute hold" (§4) | Hold ≈ **29 min** on 1-min | The universal early-flatten (fill at last bar's open) is what guarantees no overnight on every chart type. Cost = one chart-bar. |
| 2 | Commission **0.50**/side (§8) | **0.80**/side | Your real Tradovate MES/MNQ all-in (matches the ORB build → strategies stay comparable). |
| 3 | Month-end skip = T−3…T+3 (§7) | `dayofmonth ≤ 3 or ≥ 26` | The "before month-end" side can't be known live without lookahead. Documented approximation. |

### Per-TF exit haircut (consequence of decision #1)
The flatten fills one **chart-bar** before the close, so coarser TFs surrender more of the final window:

| Chart TF | Final-window bars | Approx hold | Verdict |
|---|---|---|---|
| 1-min | 30 | ~29 min | **recommended** |
| 5-min | 6 | ~25 min | ok |
| 15-min | 2 | ~15 min | big haircut |
| 30-min | 1 | ~0 → **trades suppressed** (`windowOK=false` + red label) | don't use |

---

## §9 Acceptance criteria → how to verify in TradingView

> Pine only executes inside TradingView; nothing here was machine-run. Walk these on a
> **1-minute** chart of **MNQ1!** (and spot-check **MGC1!** with the gold preset). Turn on
> **Show Debug Table**. Use the **Strategy Tester → List of Trades** for entry/exit times.

1. **Signal math** — pick a day; from the debug table read `r_firstHalf` / `r_restOfDay`. Hand-check against (10:00 price ÷ prior-16:00 close − 1) and (15:30 price ÷ prior-16:00 close − 1). Match within tick rounding.
2. **One trade/day** — List of Trades shows ≤1 entry per session; confirm it resets across a weekend and a holiday.
3. **Always flat at close** — every exit timestamps at/just-before the session close; no row spans into the next session. (Strategy Tester → no overnight bars between exit and next entry.)
4. **Mode divergence** — find a day where the first-half-hour and rest-of-day signs disagree; switch `Signal Mode` A↔B and confirm the entry direction flips on that day.
5. **r12 gate** — toggle `Use r12 Confirmation` on → trade count strictly drops; every remaining day has `sign(r_secondLast) == tradeDir`.
6. **Direction gate** — `LongOnly` → zero short rows; `ShortOnly` → zero long rows.
7. **Stops** — enable ATR-only, then tick-only, then both; verify the exit price sits at `entry ∓ stopDist` and that with both on the **tighter** distance is used; on a day the stop isn't hit, the close-flatten still fires.
8. **Sizing** — RiskBased with a stop: `qty == floor(riskDollars / (stopDist × pointvalue))`, capped at `maxContracts`; RiskBased with **no** stop → orange warning label + fixed qty; force `qty<1` (tiny riskDollars) → that day is skipped.
9. **Gold preset** — set session `820 / 1330`; the trade now runs the 13:00→13:30 window, flattening ~13:29, not 15:30→16:00.
10. **No lookahead / no repaint** — add bars (let new bars print, or scroll history load) and confirm historical entries/exits don't move; Mode B entry is the bar *after* signal completion.
11. **Timezone/DST** — check a spring-forward and a fall-back date: anchors still land on 10:00 / 15:00 / 15:30 / 16:00 wall-clock ET.
12. **Symbol-agnostic** — load MES/MNQ/MYM/MGC (and full-size ES/NQ/YM/GC) unchanged; tick/point math comes from `syminfo`.
13. **Trend gate** — `Use Trend Filter` on: zero longs on days whose trailing 252-day return is negative, zero shorts when positive; off → unchanged.
14. **Trend no-lookahead** — historical `trendDir`-driven entries don't change as bars append.

### Alerts script spot-checks
- On the confirmed `T_LAST30` bar, a **MIM L / MIM S** marker prints matching the backtest's direction for that day; an **flat** marker prints at the close.
- Create an alert on "Any alert() function call" → the firing message is valid JSON with stable keys `strat,sym,mode,dir,px,t`.
- The three `alertcondition()` options (Long/Short/Exit) appear in the alert dialog.

---

## Watch-list for the first paste (likely friction, not known bugs)

- **`array.percentile_linear_interpolation`** — confirm the exact identifier compiles in your v6 build; if not, swap to a manual rolling percentile over `volHist`.
- **`request.security` symbols** — `CBOE:VIX` / `CBOE:GVZ` need data entitlement on your plan; if absent the gate returns `na` (treated as "fail" → blocks when the gate is on).
- **Trend lookback history** — `close[252]` on the daily series needs ≥252 daily bars loaded; on a short history the trend gate returns `na`→0→blocks. Expected on fresh symbols.
- **Properties override** — after editing the script, reset Strategy Tester → Properties to **Defaults** so the baked commission/slippage apply.

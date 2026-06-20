# CLAUDE.md — Market Intraday Momentum (MIM) Strategy

**Purpose:** Implementation spec for Claude Code. Build the "Market Intraday Momentum" strategy in **Pine Script v6** for TradingView as **two separate scripts**: (1) a backtestable `strategy{}` and (2) a live-alert `indicator{}`. They share identical signal logic; only the output layer differs.

**Do not infer mechanics — they are fully specified below.** Where a default is given, use it; expose it as a user input.

---

## 0. What this strategy is (one paragraph)

Intraday return persistence on equity-index (and, weaker, gold) futures, driven by gamma-hedging flow from option/LETF market makers plus infrequent institutional rebalancing and late-informed trading. The day's early/whole-day direction predicts the **last 30 minutes**. The strategy takes **one trade per day**: at the start of the final 30-minute window it goes long or short based on a signal computed earlier in the session, holds to the cash-session close, and never holds overnight. Research basis: Gao, Han, Li & Zhou, *Market Intraday Momentum*, Journal of Financial Economics 2018 (first-half-hour → last-half-hour, SPY 1993–2013, timing Sharpe ≈ 1.08 vs 0.29 buy-and-hold); Baltussen, Da, Lammers & Martens, *Hedging Demand and Market Intraday Momentum*, JFE 2021 (rest-of-day → last 30 min across 60+ futures, gamma-hedging mechanism, reverts next days).

---

## 1. Time model (read first — everything keys off this)

All session logic uses timezone **`America/New_York`** (handles DST automatically — never hardcode a UTC offset).

The session is defined by two user inputs (`sessionStart`, `sessionEnd`) so one script works for both index and gold. Within a session, define these anchor timestamps:

| Anchor | Definition | Index preset (09:30–16:00 ET) | Gold preset (08:20–13:30 ET) |
|---|---|---|---|
| `SESSION_OPEN` | input | 09:30 | 08:20 |
| `SESSION_CLOSE` | input | 16:00 | 13:30 |
| `T_OPEN30` | `SESSION_OPEN + 30min` | 10:00 | 08:50 |
| `T_SECONDLAST` | `SESSION_CLOSE − 60min` | 15:00 | 12:30 |
| `T_LAST30` | `SESSION_CLOSE − 30min` | 15:30 | 13:00 |

Anchor **prices** (capture the price *at* each anchor — the close of the bar whose close-time equals the anchor):
- `P_prevClose` = price at the **previous** session's `SESSION_CLOSE` (prior RTH close; carries the overnight gap into the first-half-hour return).
- `P_open30` = price at `T_OPEN30`.
- `P_secondLast` = price at `T_SECONDLAST`.
- `P_last30` = price at `T_LAST30`.

**Chart timeframe:** the script must run on an intraday timeframe that divides evenly into the anchors (1, 2, 3, 5, 6, 10, 15, or 30 min). **Recommended: 1-minute** for tightest entry fidelity (see §4 on the one-bar entry rule). The script should `runtime.error()` (or warn via a status label) if the active timeframe is not intraday or does not divide 30 minutes.

> Gold note: COMEX gold trades nearly 24h electronically, but its meaningful liquidity/settlement close is **13:30 ET** (pit close). Defaults reflect that. The user can override `sessionStart`/`sessionEnd` per instrument. **`syminfo.mintick` and `syminfo.pointvalue` must be used for all tick/dollar math** so the script stays symbol-agnostic — do not hardcode contract specs.

---

## 2. The three returns

Compute at the moment each becomes known (all from captured anchor prices):

```
r_firstHalf   = P_open30    / P_prevClose   - 1     // known at T_OPEN30   (Mode A signal)
r_restOfDay   = P_last30     / P_prevClose   - 1     // known at T_LAST30   (Mode B signal)
r_secondLast  = P_last30     / P_secondLast  - 1     // known at T_LAST30   (r12 confirmation)
```

`r_secondLast` is the Gao-Han "r12" — the second-to-last half-hour (e.g., 15:00→15:30 for index). In the paper it is **independent of and complementary to** the main signal; adding it raises in-sample R² from 1.6% → 2.6%.

---

## 3. Signal construction

**Inputs:**
- `signalMode` ∈ {`A_FirstHalf`, `B_RestOfDay`} — default **`B_RestOfDay`** (stronger/more robust on futures per Baltussen).
- `useR12Confirm` (bool) — default **false**.
- `directionMode` ∈ {`Both`, `LongOnly`, `ShortOnly`} — default **`Both`**.

**Raw signal:**
```
mainRet = (signalMode == A_FirstHalf) ? r_firstHalf : r_restOfDay
rawDir  = mainRet > 0 ? +1 : mainRet < 0 ? -1 : 0     // 0 = no trade (flat day, skip)
```

**r12 confirmation (if `useR12Confirm`):** require the second-to-last half-hour to agree.
```
if useR12Confirm and sign(r_secondLast) != rawDir: rawDir = 0   // block the trade
```

**Direction gate:**
```
if directionMode == LongOnly  and rawDir == -1: rawDir = 0
if directionMode == ShortOnly and rawDir == +1: rawDir = 0
```

**Trend filter gate (if `useTrendFilter`):** block counter-trend trades — only permit longs in an up-trending year, shorts in a down-trending year. This is **Strategy 3 (Time-Series Momentum)**; inputs and the `trendDir` computation live in §7. It only ever zeroes a signal — it never creates one.
```
if useTrendFilter and rawDir != trendDir: rawDir = 0   // trendDir from §7; blocks counter-trend
```

`tradeDir` = final `rawDir` ∈ {+1, −1, 0}.

> Empirical note for the user (not code): longs are structurally cleaner than shorts here (overnight drift gives index futures a long bias; Gao-Han report higher R² when the first-half-hour return is positive). `LongOnly` may smooth a prop equity curve at the cost of ~half the trades.

---

## 4. Entry / exit timing (non-repainting rules — critical for backtest validity)

**One trade per day. No overnight.** Enforce a `var bool tradedToday` flag, reset at each new session.

**Entry:**
- **Mode A:** `mainRet` is known at `T_OPEN30` (10:00). Enter at the **open of the `T_LAST30` bar** (15:30). Full 30-minute hold. No lookahead.
- **Mode B (and any time `useR12Confirm` is on):** `mainRet`/`r_secondLast` are only known at `T_LAST30` (15:30 close). **Compute the signal on the bar that closes at `T_LAST30`, then enter at the open of the *next* bar.** On a 1-min chart this is a 1-minute delay (29-min hold); on 5-min it's a 5-minute delay (25-min hold). **This is the standard non-repainting compromise — do not enter on the same bar that completes the signal.** This is the main reason to run on a 1-minute chart.

**Exit (flatten):** at `SESSION_CLOSE`. Close the position on the last in-session bar (the bar whose close-time == `SESSION_CLOSE`). Use `strategy.close()` in the strategy script. The position must be flat every day — assert this in testing.

**Stops** (if enabled, see §5) may exit earlier than the close.

**No-repaint requirements:**
- All signal decisions use **confirmed bar closes** only (`barstate.isconfirmed` for the alert script).
- Any `request.security()` call uses `lookahead=barmerge.lookahead_off` and references the **prior** completed value (`[1]`) where appropriate.
- Adding future bars to history must not change historical signals (test for this explicitly, §9).

---

## 5. Stops (backtest script) — two independent toggles

Both can be enabled simultaneously. **If both are on, the effective stop is the tighter (closer) of the two** — i.e., whichever would be hit first.

**Input group A — ATR stop:**
- `useAtrStop` (bool) — default **false**.
- `atrTimeframe` — default **`30`** (30-min ATR).
- `atrLength` — default **14**.
- `atrMult` — default **2.0**.
- Stop distance (price) = `atrMult * atr_value`, where `atr_value = request.security(syminfo.tickerid, atrTimeframe, ta.atr(atrLength)[1], lookahead=barmerge.lookahead_off)`.

**Input group B — fixed-tick stop:**
- `useTickStop` (bool) — default **false**.
- `fixedTicks` — default **40** (override per instrument; see table).
- Stop distance (price) = `fixedTicks * syminfo.mintick`.

**Suggested `fixedTicks` starting points (tune to your tick data — these are not optimized):**

| Instrument | Tick size | ~Suggested fixedTicks | ≈ Stop distance |
|---|---|---|---|
| ES / MES | 0.25 | 32 | 8.0 pts |
| NQ / MNQ | 0.25 | 80 | 20.0 pts |
| YM / MYM | 1.0 | 40 | 40 pts |
| GC / MGC | 0.10 | 50 | 5.0 pts |

**Effective stop logic:**
```
stopDist = +inf
if useAtrStop:  stopDist = min(stopDist, atrMult * atr_value)
if useTickStop: stopDist = min(stopDist, fixedTicks * syminfo.mintick)
hasStop = stopDist < +inf
stopPrice = (tradeDir == +1) ? entryPrice - stopDist : entryPrice + stopDist
```

Implement via `strategy.exit(stop=stopPrice)` paired with the entry. The session-close flatten still applies if the stop never triggers.

---

## 6. Position sizing (backtest script)

**Input:** `sizingMode` ∈ {`Fixed`, `RiskBased`} — default **`Fixed`**.

- **Fixed:** `qty = qtyFixed` (input, default **1** contract).
- **RiskBased:** `qty = floor(riskDollars / (stopDist * syminfo.pointvalue))`, capped at `maxContracts`.
  - `riskDollars` input, default **200**. `maxContracts` input, default **10**.
  - **Requires `hasStop == true`.** If no stop is enabled, **fall back to `qtyFixed`** and surface a one-time warning label ("RiskBased sizing requires a stop; using fixed qty").
  - If both stops are on, size against the **effective (tighter) `stopDist`** from §5.
  - Guard: if computed `qty < 1`, set `qty = 0` (skip trade) rather than rounding up.

---

## 7. Optional filters (toggles — all default OFF so the baseline = academic every-signal-day)

This is where the user A/B tests. The volatility filter is the single biggest expectancy lever in the research (predictive R² ≈ 0.6% on low-volatility days vs 3.3% on high-volatility days; near-zero edge on quiet days), so it must be available, but default off so the baseline matches the published every-day strategy.

**Volatility filter:**
- `useVolFilter` (bool) — default **false**.
- Metric: opening volatility proxy = `atr_value` (the same 30-min ATR from §5, or `ta.atr(atrLength)` on chart TF) as a **percent of price**, evaluated at `T_OPEN30`.
- `volLookbackDays` — default **60**. `volPercentile` — default **50** (the user can raise toward ~66 for top-tercile behavior).
- Rule: take the trade only if today's opening-vol proxy ≥ the `volPercentile`-th percentile of its own values over `volLookbackDays`. Use `ta.percentile_linear_interpolation` (or a manual rolling percentile) over a daily-sampled series; compute with no lookahead.

**VIX gate (sub-toggle of the vol filter):**
- `useVixGate` (bool) — default **false**. `vixThreshold` — default **18**.
- `vixClose = request.security("CBOE:VIX", "D", close[1], lookahead=barmerge.lookahead_off)`. Take the trade only if `vixClose >= vixThreshold`.
- **Gold caveat:** VIX is an equity-index vol measure — not meaningful for MGC/GC. For gold, either leave `useVixGate` off or point it at `CBOE:GVZ` (gold VIX). Document this in the input tooltip. The ATR-percentile filter is the cross-instrument one.

**Month-end exclusion (optional, research-backed minor edge):**
- `useMonthEndFilter` (bool) — default **false**.
- Gao-Han find predictability is weaker in the 7 days around month-end (T−3…T+3) due to lower institutional volume. Approximate: skip the trade if the session date is within 3 trading days of a month boundary (either side). Document the approximation.

**Trend filter — Strategy 3 (Time-Series Momentum overlay):**
- `useTrendFilter` (bool) — default **false**.
- `trendLookbackDays` — default **252** (one trading year).
- **What it does (plain terms):** a "direction permission slip." It checks whether the instrument has trended up or down over the past year and only permits trades that agree with that yearly direction — longs in an up year, shorts in a down year. It generates **no trades on its own**; it only zeroes out counter-trend signals from §3. This is why it is a filter toggle, not a separate strategy or script.
- **Why it helps:** counter-trend trades are where the worst losses cluster. Deleting "long in a bear / short in a bull" removes a category of losers (it does not add winners), smoothing the equity curve and lowering the odds of a streak deep enough to trip a prop trailing drawdown — pure, academically-backed risk reduction.
- **Research basis:** Moskowitz, Ooi & Pedersen, *Time Series Momentum*, JFE 2012 — 58 futures, 1985–2009; the 12-month return predicts the next move, significant in 52 of 58 markets.
- **Computation (the `trendDir` used by the §3 gate):**
  ```
  yearAgoClose = request.security(syminfo.tickerid, "D", close[trendLookbackDays], lookahead=barmerge.lookahead_off)
  trendDir     = sign( close / yearAgoClose - 1 )   // +1 up year, -1 down year, 0 flat
  ```
  Use only completed daily bars (the `[trendLookbackDays]` offset on a confirmed daily series prevents lookahead). On a flat year (`trendDir == 0`), treat as "no permission" → block (or document as pass-through; default to block for safety).

---

## 8. The two deliverables

### Script 1 — `MIM_Backtest.pine`
- `//@version=6`, `strategy("MIM Backtest", overlay=true, calc_on_every_tick=false, process_orders_on_close=false, pyramiding=0, default_qty_type=strategy.fixed, ...)`.
- Wire commission + slippage as the user-facing realism layer:
  - `commissionPerSide` input (default **0.50** USD/contract/side) → `strategy(commission_type=strategy.commission.cash_per_contract, commission_value=commissionPerSide)`.
  - `slippageTicks` input (default **1**) → `strategy(slippage=slippageTicks)`.
  - Note in a tooltip that prop firms set their own commissions; the user should match them to TopStep/Lucid/Apex actuals.
- All inputs from §3, §5, §6, §7, plus session inputs from §1.
- Plot session markers and (optionally) a debug `table` showing today's `r_firstHalf`, `r_restOfDay`, `r_secondLast`, `tradeDir`, chosen `qty`, and `stopPrice` — gate behind a `showDebug` toggle.
- Entries/exits per §4–§6; one trade/day; flat at close.

### Script 2 — `MIM_Alerts.pine`
- `//@version=6`, `indicator("MIM Alerts", overlay=true)`.
- Identical signal logic (§1–§4, §7). No `strategy.*` calls, no sizing/commission.
- Fire alerts on **confirmed bars** only:
  - **Entry alert** at the entry bar (§4 timing) — long and short variants.
  - **Exit alert** at `SESSION_CLOSE`.
  - Use `alert()` with a structured, parseable message (JSON-friendly), e.g.:
    `{"strat":"MIM","sym":"{{ticker}}","mode":"B","dir":"LONG","px":{{close}},"t":"{{time}}"}`
  - Also expose `alertcondition()` for long-entry, short-entry, and exit so the user can wire TradingView alerts. The structured message is intended to feed a downstream webhook/bot (the user runs a Telegram bot), so keep keys stable and documented.
- The alert script is for **discretionary execution and notification** — it tells the user the direction and entry at the start of the last 30-minute window. For Mode B/r12, the alert can only be accurate at `T_LAST30` close (signal completion); document that the actionable alert arrives then.

---

## 9. Acceptance criteria / test cases (Claude Code must verify before "done")

1. **Signal math:** on a hand-picked day, `r_firstHalf` and `r_restOfDay` match a manual calc from prior-close and anchor prices (within tick rounding).
2. **One trade per day:** never more than one entry per session; `tradedToday` resets correctly across the session boundary and across weekends/holidays.
3. **Always flat at close:** no position is ever carried past `SESSION_CLOSE`; no overnight exposure on any day in the backtest.
4. **Mode divergence:** Mode A and Mode B produce *different* entries on days where the first-half-hour and rest-of-day signs disagree (prove with at least one example).
5. **r12 gate:** enabling `useR12Confirm` strictly reduces trade count and blocks every trade where `sign(r_secondLast) != mainDir`.
6. **Direction gate:** `LongOnly` takes zero shorts; `ShortOnly` zero longs.
7. **Stops:** ATR and fixed-tick stops each trigger at the correct price; with both on, the tighter governs; session-close flatten still fires when no stop is hit.
8. **Sizing:** RiskBased `qty` equals `floor(riskDollars / (stopDist × pointvalue))`, respects `maxContracts`, and **falls back to fixed qty with a warning when no stop is enabled**; `qty < 1` ⇒ skip.
9. **Session inputs / gold:** setting the gold preset repositions the window so the last-30 trade runs 13:00–13:30 ET, not 15:30–16:00.
10. **No lookahead / no repaint:** signals computed on confirmed closes; entry is the bar *after* signal completion for Mode B/r12; historical signals do not change when more bars are appended; `request.security` uses `lookahead_off` + `[1]`.
11. **Timezone/DST:** anchors land on the correct wall-clock times across a spring-forward and fall-back date.
12. **Symbol-agnostic:** runs unmodified on MES, MNQ, MYM, MGC, ES, NQ, YM, GC, deriving tick/point values from `syminfo`.
13. **Trend filter gate:** with `useTrendFilter` on, zero long trades occur on days when the trailing 252-day return is negative, and zero shorts when it is positive; with it off, behavior is unchanged.
14. **Trend filter no-lookahead:** `trendDir` uses only completed daily bars; the `close[trendLookbackDays]` reference and `request.security` resolve with `lookahead_off`, and historical `trendDir` values don't change when bars are appended.

---

## 10. Parameter quick-reference (defaults)

| Input | Default | Notes |
|---|---|---|
| `signalMode` | `B_RestOfDay` | A = first-half-hour, B = rest-of-day |
| `useR12Confirm` | false | require 2nd-to-last half-hour to agree |
| `directionMode` | `Both` | Both / LongOnly / ShortOnly |
| `sessionStart` / `sessionEnd` | 09:30 / 16:00 ET | gold preset 08:20 / 13:30 |
| `useAtrStop` | false | dist = `atrMult × ATR(atrLength @ atrTimeframe)` |
| `atrTimeframe` / `atrLength` / `atrMult` | 30 / 14 / 2.0 | |
| `useTickStop` | false | dist = `fixedTicks × mintick` |
| `fixedTicks` | 40 | override per instrument (table §5) |
| `sizingMode` | `Fixed` | RiskBased needs a stop; else falls back |
| `qtyFixed` | 1 | |
| `riskDollars` / `maxContracts` | 200 / 10 | RiskBased |
| `useVolFilter` | false | ATR-percentile gate |
| `volLookbackDays` / `volPercentile` | 60 / 50 | raise pctile toward 66 for top-tercile |
| `useVixGate` | false | `vixThreshold` 18; gold → GVZ or off |
| `useMonthEndFilter` | false | skip T−3…T+3 around month end |
| `useTrendFilter` | false | **Strategy 3** — only trade with the 252-day trend |
| `trendLookbackDays` | 252 | one trading year for the trend sign |
| `commissionPerSide` / `slippageTicks` | 0.50 / 1 | backtest realism; match prop firm |
| `showDebug` | false | on-chart table of returns/signal |

---

## 11. Prop-firm notes (context for the user; affects defaults, not core logic)

- **Cutoffs are non-binding here:** the strategy exits at the cash close (16:00 ET index / 13:30 ET gold), which is earlier than every firm's flatten time (Apex 16:59 ET, TopStep 15:10 CT = 16:10 ET, Lucid 16:45 ET). No conflict.
- **Gold routing:** as of June 2026 **Apex has metals (GC/MGC) suspended** — run gold only on Lucid/TopStep until relisted. Index micros are fine on all three.
- **Fit:** one short, defined-risk trade per day with a small max adverse excursion → smooth, even daily P&L. This is well-suited to trailing-drawdown and consistency rules. Recommend the user sets `riskDollars` to ~8% of the trailing-drawdown buffer (e.g., ~$200 on a 50K Apex account) and applies an external daily profit cap to stay consistency-compliant.

---

## 12. Honesty caveats (carry into code comments)

- The **unconditional** edge has decayed since ~2013 (Rosa 2022, *Journal of Futures Markets*, finds the naive overnight→last-half-hour version's out-of-sample predictability disappears; momentum broadly weaker post-2009). What survives is the **conditional** edge — high-volatility/high-volume/news days. That is precisely why `useVolFilter` exists; the user should compare filtered vs unfiltered on their own Sierra Chart tick data (2018–2026) before funding, and re-check the post-2021 slope on MNQ specifically.
- Published Sharpe figures are gross of micro-contract commissions/slippage and were measured on ETFs/large futures. Expect realized performance below paper figures on micros — hence the explicit commission/slippage inputs.
- Mode B (rest-of-day) entries arrive late by one bar by design (§4). Do not "fix" this by entering on the signal bar — that introduces lookahead and inflates backtest results.
- The trend filter (§7, Strategy 3) helps most on instruments that trend cleanly — **MGC/GC (gold) above all**. On the index micros (MES/MNQ/MYM) the yearly trend is choppier, so expect a smaller benefit, occasionally a slight drag from skipping a good counter-trend day. Keep it default-off and A/B test per instrument rather than assuming it always helps.

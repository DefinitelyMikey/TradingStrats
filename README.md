# TradingStrats

Monorepo of trading strategies. Each subfolder = one self-contained strategy (Pine Script, plans, docs).

## Strategies

| Folder | Strategy | Platform | Status |
|--------|----------|----------|--------|
| [`8AM_ORB/`](8AM_ORB/) | Opening-Range Breakout — 8:00 AM ET, selectable ORB duration (5/10/15/20/30/60 min). Breakout → dual-revisit → floor/ceiling engine. Default instruments MES & MNQ. | TradingView Pine Script | In research |
| [`Market_Intraday_Momentum/`](Market_Intraday_Momentum/) | Market Intraday Momentum — one trade/day, enters the final 30-min window long/short on a rest-of-day (or first-half-hour) signal, flat by the cash close, never overnight. Two scripts: `MIM_Backtest.pine` (strategy) + `MIM_Alerts.pine` (alerts). Index micros + gold. | TradingView Pine Script | In research |

## Conventions

- One strategy per top-level folder.
- Each strategy carries its own `CLAUDE.md` (design spec) and `plans/`.
- `.codegraph/` indexes are machine-generated and git-ignored repo-wide (`**/.codegraph/`).

# TradingStrats

Monorepo of trading strategies. Each subfolder = one self-contained strategy (Pine Script, plans, docs).

## Strategies

| Folder | Strategy | Platform | Status |
|--------|----------|----------|--------|
| [`8AM_ORB/`](8AM_ORB/) | Opening-Range Breakout — 8:00 AM ET, selectable ORB duration (5/10/15/20/30/60 min). Breakout → dual-revisit → floor/ceiling engine. Default instruments MES & MNQ. | TradingView Pine Script | In research |

## Conventions

- One strategy per top-level folder.
- Each strategy carries its own `CLAUDE.md` (design spec) and `plans/`.
- `.codegraph/` indexes are machine-generated and git-ignored repo-wide (`**/.codegraph/`).

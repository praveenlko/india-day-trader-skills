# India Day Trader skills (NSE)

Nine Claude skills for intraday trading on the NSE, adapted from
[Jakeschincariol/day-trader-skill](https://github.com/Jakeschincariol/day-trader-skill) (MIT).
Scan Nifty charts for setups, track headlines, time the entry, plan the stop and target
(in Rs 0.05 ticks), replay the trade 10,000 times from the stock's own recent bars, size it
to your rupee budget, check it against your limits, and get a ticket to place in your own
broker app. Nothing here places an order.

| command | job |
| --- | --- |
| `/india-trader-scanner` | Scans the Nifty 500 (or any NSE list) for opening-range breakouts, 20-day breakouts, flags, oversold bounces |
| `/india-trader-news` | Yahoo Finance headlines and the move so far for the stocks you watch |
| `/india-trader-entry` | Watches the 9:15-9:30 range bar by bar and tells you when a 5-minute bar closes above it |
| `/india-trader-exit` | Stop and target before you enter; flat by 15:15 IST |
| `/india-trader-timemachine` | 10,000 replays of the rest of the session plus the same rule on the last 60 sessions |
| `/india-trader-budget` | Whole shares that fit your rupee budget and max loss per trade |
| `/india-trader-risk` | PASS or BLOCK against your limits |
| `/india-trader-broker` | The ticket (paper by default) with the manual NSE / MIS steps; never sends an order |
| `/india-trader-alerts` | Stop, target and flat-by alerts; NSE clock |

## What changed from the US version

- Clock: IST (UTC+5:30, no daylight saving). Session 9:15-15:30, opening range 9:15-9:30,
  no new entries after 14:30, flat by 15:15 (brokers square off MIS around 15:15-15:25).
- Calendar: NSE 2026 trading holidays. **2027 is not published yet**: until NSE publishes it,
  only weekends count as closed in 2027. Add the dates to `NSE_HOLIDAYS` in the scripts.
- Symbols: plain NSE symbols (`RELIANCE`, `M&M`, `BAJAJ-AUTO`); Yahoo gets `.NS` automatically.
- Universe: NSE's Nifty 50/100/200/500 CSVs or every listed equity; EQ series only.
- Money: rupees. Config keys end in `_inr`. Defaults assume a Rs 1,00,000 account:
  max loss per trade Rs 500, daily loss Rs 1,500, max order Rs 50,000.
- Dropped: Pelosi tracker and SEC earnings analyst (US-only), Alpaca / IBKR order tools,
  the US pattern-day-trader rule.
- Not included: NSE/BSE corporate announcements, F&O, brokerage/STT/charges in the risk maths.

## Install

```
cp -r india-trader-* ~/.claude/skills/
```

Python 3.8+, standard library only. Data comes from free, unofficial Yahoo Finance endpoints
and NSE's archive CSVs: both can lag, rate-limit or change. Verify prices in your broker app.

Not financial advice and not SEBI-registered research. Most intraday traders lose money.
Start on paper.

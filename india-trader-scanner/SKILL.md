---
name: india-trader-scanner
description: >-
  Chart Scanner. Scans thousands of charts for setups: opening-range
  breakouts, flags, oversold bounces. Builds the list of NSE shares from
  NSE's own Nifty lists (Nifty 500 by default, or every listed equity),
  runs a pre-market scan of daily closes (20-day closing-high breakout, bull
  flag, reclaimed the 20-day average, oversold, oversold bounce) and an
  intraday opening-range breakout scan on 5-minute bars, then writes
  india-trader/watchlist.json for india-trader-news and india-trader-entry. Free Yahoo data,
  bar by bar. Use when the user says "scan the market", "find me setups",
  "what is breaking out", "opening range breakouts", "scan for bull flags",
  "oversold stocks", "pre-market scan", "build my watchlist", "what should I
  look at today", or at the start of a trading day before /india-trader-news.
argument-hint: "[universe | daily | orb] [--at HH:MM]"
---

# india-trader-scanner

The Chart Scanner is the first step of a day trade. It reads hundreds of
free charts and lists the ones that match a rule. That is all it does. A scan
is a list of charts that match a rule, not a list of stocks to buy, and a
setup on a chart is not a prediction. Most people who day trade lose money;
the scanner does not change that. It only saves you from flipping through
500 charts by hand.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/scan.py" universe                       # -> india-trader/universe.txt
python3 "${CLAUDE_SKILL_DIR}/scan.py" daily                          # before the open -> india-trader/watchlist.json
python3 "${CLAUDE_SKILL_DIR}/scan.py" daily --symbols RELIANCE,TCS,INFY  # or --watchlist FILE, --limit 200
python3 "${CLAUDE_SKILL_DIR}/scan.py" orb                            # after 9:35 AM IST -> india-trader/watchlist.json
python3 "${CLAUDE_SKILL_DIR}/scan.py" orb --at 09:45                 # cut the clock at 9:45 AM IST
python3 "${CLAUDE_SKILL_DIR}/scan.py" orb --watchlist india-trader/watchlist.json --json
```

Flags: `--min-price 50` (rupees) and `--min-volume` (daily 500,000, orb 0),
`--or-minutes 15`, `--max-confirm 200` (orb stage 2),
`--now "2026-11-04 09:50"` (IST) to pin the clock, `--offline PATH`
to read saved Yahoo files, `--no-write`, `--show N`, `--json`. Exit code 0
when the scan found something, 1 when it found nothing or could not run (the
reason is one plain line).

## What it reads and writes

| file | what |
| --- | --- |
| `india-trader/universe.txt` | written by `universe`: one `SYMBOL\|Name` a line, read by `daily` and `orb` |
| `india-trader/watchlist.json` | written by `daily` and `orb`: `mode`, `as_of`, `universe`, `scanned`, `failed`, `hits[]`. india-trader-news and india-trader-entry read `hits[].symbol` |
| `india-trader/config.json` | read by `orb`: `no_entries_after` (default 14:30) |

## The universe

`universe` downloads one of NSE's own CSV lists from nsearchives.nseindia.com and
keeps the plain `EQ`-series shares:

```bash
python3 "${CLAUDE_SKILL_DIR}/scan.py" universe                    # Nifty 500 (default, about 500 liquid names)
python3 "${CLAUDE_SKILL_DIR}/scan.py" universe --index nifty50    # nifty50 | nifty100 | nifty200 | nifty500 | all
python3 "${CLAUDE_SKILL_DIR}/scan.py" universe --file ind_nifty500list.csv   # a CSV you downloaded yourself
```

- Other series (BE trade-for-trade, SM, ...) are dropped: they cannot be day traded.
- Symbols are plain NSE symbols (`RELIANCE`, `M&M`, `BAJAJ-AUTO`). Yahoo gets them
  with a `.NS` suffix; the scripts add it themselves. `^NSEI` (Nifty 50) is kept as is.
- `--index all` is every listed equity (about 2,000+), most of them too thin to day
  trade. Nifty 200 or 500 is the sensible universe. If NSE's site refuses the
  download (it sometimes does), save the CSV from niftyindices.com
  (Indices > the index > Constituents) and use `--file`.
- Intraday (MIS) and short selling are available on many, not all, NSE stocks. Check
  your broker's MIS list; the scanner does not know it.

## The daily scan (before the open)

Yahoo's spark endpoint returns closes for 20 symbols a request, so the Nifty 500
takes about 25 requests, well under a minute with the polite pause (0.3 s). Liquidity comes from the spark meta: last price at
least `--min-price` and last session's volume at least `--min-volume`. Run it
before the open or after the close: during the session the volume is the
day's so far, and today's daily bar is left out until 3:30 PM.

The setups are computed from daily closes only (spark has no highs or lows),
so they are close cousins of a scan that uses highs and lows:

| setup | the rule on closes |
| --- | --- |
| `breakout_20d` | the last close is above every close of the 20 sessions before it |
| `bull_flag` | up more than 10 % over the 10 sessions before the last 5, then the last 5 closes inside a 5 % range |
| `reclaim_ma20` | the close before was under its 20-day average, the last close is above it |
| `oversold` | RSI 14 (Wilder) under 30 |
| `oversold_bounce` | RSI 14 was under 30 yesterday and today closed up |

A closing-high breakout fires more often than an intraday-high one. Hits are
listed most traded first (price times volume).

## The opening-range breakout scan (during the session)

The opening range is the first `--or-minutes` (15) of the session: the 9:15,
9:35 and 9:40 five-minute bars. A hit is the first completed 5-minute bar
to CLOSE above the range high, at that bar's close time.

1. Stage 1, every symbol: spark closes of today's 5-minute bars, 20 a
   request. A chart goes on only if a completed bar after the range closed at
   or above the highest close of the range. That is a necessary condition
   (the range high is at least every range close), so no breakout is lost.
2. Stage 2, each candidate: the v8 chart with real highs and lows, one
   request each. Range high = the highest high of the range bars, range low =
   the lowest low. On a strong morning most charts are candidates, so stage 2
   checks at most `--max-confirm` (200) of them, the most traded first (price
   times volume so far), and prints PARTIAL with the count it skipped.
   `--max-confirm 0` checks them all.

Timing, live on Tue Oct 20 2026 (a strong morning): 100 liquid names at
10:01 AM took 35 seconds (5 spark requests, 70 v8 charts, 51 of 91 usable
charts broke out). The whole universe at 10:03 AM took about 4 minutes (298
spark requests, then 200 v8 charts): 2,448 usable charts, 1,782 candidates,
the 200 most traded checked, 119 of those broke out. For a fast answer, scan
the daily watchlist or a list of names you trade (`--watchlist`, `--symbols`).

A chart is usable only when every 5-minute bar from 9:15 to the cutoff is
there; a missing bar means a halt or a thin stock. Null closes are dropped,
a bar off the 5-minute grid (the live tick, like 10:58:07) is dropped, and a
bar counts only once it is complete (start + 5 minutes). Before 9:35 AM, on a
weekend or holiday, after the close, or after `no_entries_after` (an expired
setup) it prints one plain line and writes nothing. NSE has no scheduled early closes
(the Diwali Muhurat session is a special evening session and is not covered).

## Data and limits

- Yahoo's chart and spark endpoints are free, unofficial and can lag. Bars are
  5-minute bars, bar by bar, never tick by tick. query2 first, query1 on a 429
  or 5xx, one retry after a 5-second wait on a 429.
- Partial results are said as partial: "Fetched 4,980 of 5,259 charts (279
  failed or had no data)". Never round a partial scan up to "the whole market".
- NSE holidays for 2026 (NSE circular, cross-checked Nov 2026). NSE has not published 2027 yet: until it does, only weekends count as closed in 2027.
- The Yahoo `.NS` charts are free, unofficial and delayed; NSE data from Yahoo can lag by a minute or more.

## Steps

1. Once a week or so: `scan.py universe`.
2. Before the open: `scan.py daily`. Tell the user the counts (fetched,
   liquid, scanned, hits) and the top few by setup.
3. After 9:50 AM IST (10:15 is the classic): `scan.py orb`. Tell the user how
   many usable charts broke their range and when, and that a later hit is
   closer to the price now.
4. The user picks one chart. Hand it on: `/india-trader-news TICKER` (what is moving
   it), then `/india-trader-entry TICKER` (the entry bar by bar). Never pick for them.


## Rules

- Say what was scanned: how many symbols, how many charts came back, how many
  were usable, and the time of the data.
- Say "matched a rule", never "a buy", "a winner" or "a good trade".
- Never print a P&L, a return or a profit figure.
- A hit is stale once the price has moved on. Re-run before acting on it.

## What it never does

It never places, sizes or checks an order, never says which stock to buy,
never calls data real time, and never promises that a setup works.

## Safety

- Education and tooling, not financial advice. A PASS is a rules check, not a recommendation.
- Nothing in this pack places an order. india-trader-broker only prepares a ticket and records the user's yes;
  the user places the order themselves in their own broker app (Zerodha Kite, Dhan, Upstox, Groww, ...).
- Paper trading by default (`"paper": true`). Going live is the user's own decision, in their own config.
- Intraday (MIS) trading is high risk. SEBI studies of Indian equity day traders have repeatedly found that
  most of them lose money. Start on paper, and risk only money you can afford to lose.
- No options (F&O), margin or leverage beyond what the user's broker gives MIS, and no short selling, unless
  the config allows it and the user confirms.
- Broker keys and passwords never go in the chat.
- This is not SEBI-registered advice and not a research report.

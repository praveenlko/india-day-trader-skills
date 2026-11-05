---
name: india-trader-alerts
description: >-
  Price Alerts. Rings when a price, a stop or the opening bell hits. Turns
  the trade plan into three alerts in one go (the stop, the target and "flat
  by close" at your flat-by time), plus above / below / stop / target levels,
  percent moves from the previous close, any time of day, and the NSE
  opening bell, each ringing once so you are not spammed. Knows NSE hours
  (9:15 to 3:30 IST) with the 2026 holidays, works out IST itself, fetches
  free delayed quotes (Yahoo chart JSON, `.NS` symbols) one ticker at a time, and writes india-trader/prices.json for sizing and the
  risk check. Use when the user says "alert me when", "tell me if RELIANCE hits",
  "watch my stop", "watch my trade", "remind me to close by 3:15", "is the
  market open", "when does the market open", "get me a price", or right
  after /india-trader-exit writes a plan.
argument-hint: "[ticker] [price]"
---

# india-trader-alerts

Price Alerts watches the numbers so the user does not have to stare at a
chart. An alert is information, never an order: when a stop alert rings,
nothing is sold by this skill. With a bracket order the stop-loss and the
take-profit already sit at the broker and do the work; the alerts tell the
user what happened and when it is time to be flat.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/alerts.py" clock                          # NSE open or closed, next open
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --from-plan                # stop, target, flat by close from india-trader/plan.json
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --opening-bell
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --symbol RELIANCE --stop 358.32
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --symbol RELIANCE --target 370.04
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --at 15:15 --label "flat by close" --before-close
python3 "${CLAUDE_SKILL_DIR}/alerts.py" add --symbol KO --pct 3
python3 "${CLAUDE_SKILL_DIR}/alerts.py" check                          # exit 0 = something rang
python3 "${CLAUDE_SKILL_DIR}/alerts.py" quote RELIANCE --write             # -> india-trader/prices.json
python3 "${CLAUDE_SKILL_DIR}/alerts.py" quote --holdings --write       # portfolio + plan tickers
python3 "${CLAUDE_SKILL_DIR}/alerts.py" list
python3 "${CLAUDE_SKILL_DIR}/alerts.py" reset tsla-plan-stop
```

`--now "2026-11-04 15:15"` pins the clock (IST); `--offline DIR`
reads saved quote files instead of the network.

## What it reads and writes

| file | what |
| --- | --- |
| `india-trader/plan.json` | the trade from india-trader-exit, for `add --from-plan` |
| `india-trader/alerts.json` | the rules (`alerts.example.json` in this folder) |
| `india-trader/state/alerts.json` | which rules rang and when, so each rings once |
| `india-trader/prices.json` | the latest quotes, merged in, for india-trader-budget and india-trader-risk |
| `india-trader/portfolio.json` | the tickers held, for `quote --holdings` |

## Rule types

| type | rings when | rings again |
| --- | --- | --- |
| `above` / `target` | price at or above `price` | after `reset`, or on its own if `"rearm": true` and the price fell back |
| `below` / `stop` | price at or below `price` | same |
| `pct_move` | moved `pct` percent from the previous close (or `ref`), optional `direction` up or down | once per day at most |
| `time` | the IST clock passed `at` ("15:15") on a trading day | once per trading day |
| `opening_bell` | the NSE is open | once per trading day |

A rule with `"date"` is watched on that IST date only; `--from-plan`
rules carry the plan's date, so yesterday's stop never rings today. A `time`
rule with `"before_close": true` moves earlier on an early close; NSE has none scheduled, so it rings at your flat-by time.

## Quotes

- `yahoo` (the only source here): `query2.finance.yahoo.com/v8/finance/chart/<TICKER>`, an
  unofficial free JSON endpoint, one request per ticker, half a second apart.
  On a 429 it waits and tries once more; on a 429 or a 5xx after that it
  tries `query1`. It identifies itself as `day-trader-skill/2.0`.
- Free quotes are unofficial, delayed and can lag. They are fine for alerts
  and for checking a size, not for timing an order. The broker's price is the
  one that fills. Each price in `india-trader/prices.json` keeps its quote time, and
  india-trader-risk only uses one under 15 minutes old.
- If Yahoo is down, read the price from your broker app and write it into
  `india-trader/prices.json` as `{"as_of": "...Z", "prices": {"RELIANCE": 1415.30}}`.

## The NSE clock

Regular session 9:15 a.m. to 3:30 p.m. IST, closed on weekends and on these NSE
holidays (NSE circular; cross-checked on two sites, Nov 2026):

- 2026: Jan 15 (Maharashtra municipal elections), Jan 26, Mar 3 (Holi), Mar 26,
  Mar 31, Apr 3 (Good Friday), Apr 14, May 1, May 28 (Bakri Id), Jun 26
  (Muharram), Sep 14, Oct 2, Oct 20 (Dussehra), Nov 10 (Diwali Balipratipada),
  Nov 24, Dec 25.
- 2027: NSE has not published it yet. Only weekends are closed until the list
  is added (edit `NSE_HOLIDAYS` in `alerts.py`, `entry.py`, `exit.py`, `check.py`,
  `scan.py`). The tool says the holiday list is unknown for dates it does not cover.
- The Diwali Muhurat evening session (Sun Nov 8, 2026) is not a regular
  session and is not covered.

IST has no daylight saving, so there is no time zone file to worry about.

## Steps

1. Right after india-trader-exit writes the plan, run `add --from-plan`. Confirm
   the three alerts back to the user: the stop, the target and the flat-by time.
2. Run `check` while the trade is open: every few minutes from a Claude
   scheduled task or `/loop`. Each run fetches only the tickers in today's
   rules and rings each alert once. A check reads the latest delayed quote;
   it is not a live feed, and a price can pass a level between two checks.
3. Tell the user what rang in one line each. A stop or target alert means the
   bracket order has probably filled: check `get_open_position` and record the
   fill with india-trader-broker. "Flat by close" means close anything still open:
   india-trader-broker `flatten`, then india-trader-risk, then the user's yes. Never act
   for them.


## Rules

- An alert is a fact about a price or the clock. Say it as a fact.
- Never present a quote as live. Free quotes are delayed and unofficial.
- Confirm every new rule back to the user: ticker, type, level, day.

## What it never does

It never places, cancels or changes an order, never claims a quote is
live, and never predicts where a price is going.

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

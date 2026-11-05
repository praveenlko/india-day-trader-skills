---
name: india-trader-broker
description: >-
  Order Ticket. Turns an approved order into a plain ticket you place yourself in your
  Indian broker app, paper trading by default, never without your yes. Shows each order
  in plain words (a bracket shows its stop-loss, target and the max loss at the stop),
  records your yes word for word, spells out the manual NSE / MIS steps (entry, SL-M
  stop, limit target, since Indian brokers no longer offer bracket orders), builds the
  sell ticket that closes a day trade by your flat-by time, records fills, and logs each
  trade's max loss so india-trader-risk can hold the daily limit. It does not connect to
  any broker and never sends an order. Use when the user says "place the order", "buy
  it", "execute", "close my position", "flatten", "get me out", "set up paper trading",
  or after /india-trader-risk passes.
argument-hint: "[order number]"
---

# india-trader-broker

The Order Ticket is the last step of the pack. **It never places an order.** There is
no broker connection in the India version: it prepares a ticket, records the user's
yes, and tells them exactly what to enter in their own broker app. An order is only
ever "ready" after two gates, every time, for every order:

1. **india-trader-risk passed this exact order today** (matched by fingerprint).
2. **The user typed an explicit yes for this one order** in the chat.

Then the user places it themselves. No yes, no ticket.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/ticket.py" show                     # every ticket, READY or not
python3 "${CLAUDE_SKILL_DIR}/ticket.py" approve --index 1 --reply "yes"
python3 "${CLAUDE_SKILL_DIR}/ticket.py" result --index 1 --status placed --filled-qty 35 --filled-price 1415.30
python3 "${CLAUDE_SKILL_DIR}/ticket.py" fill --symbol RELIANCE --side sell --qty 35 --price 1417.40
python3 "${CLAUDE_SKILL_DIR}/ticket.py" flatten --symbol RELIANCE --qty 35
python3 "${CLAUDE_SKILL_DIR}/ticket.py" log
```

`ticket.py` never talks to a broker. It renders tickets, checks that a reply is a real
yes for that order, builds the flatten ticket, and keeps the log in
`india-trader/state/broker.json`. `--now "2026-11-04 09:50"` pins the clock (IST).

## What it reads and writes

| file | what |
| --- | --- |
| `india-trader/orders.json` | the proposed orders (india-trader-budget writes it; `flatten` replaces it) |
| `india-trader/state/risk.json` | india-trader-risk's verdicts; only a PASS from today is READY |
| `india-trader/state/broker.json` | every yes, no, placed, failed and filled, with each entry's max loss |
| `india-trader/portfolio.json` | cash and shares, updated by `fill` and by `result --filled-qty` |
| `india-trader/plan.json` | the flat-by time for `flatten` |

## Paper first

`"paper": true` (the default in `india-trader/config.json`) means the ticket is a practice
trade: record it with `result`, and india-trader-alerts watches it, but nobody is asked to
place anything. With `"paper": false` the ticket also prints the manual steps below. Going
live is the user's own decision; this skill never switches it.

## How an Indian intraday order is placed (what the ticket prints)

Indian brokers (Zerodha, Dhan, Upstox, Groww, Angel One, ...) stopped offering bracket
orders, so the one proposed bracket becomes three orders the user places and watches:

1. **Entry**: NSE, BUY, the symbol, the quantity, product **MIS** (intraday), market or limit.
2. **Stop-loss**: after the entry fills, a SELL with order type **SL-M** (or SL) and the
   stop as the trigger price.
3. **Target**: a SELL **LIMIT** at the target price.

When the stop or the target fills, the user cancels the other one. Prices move in
Rs 0.05 ticks, which india-trader-exit already respects. MIS positions are squared off
automatically by the broker around 3:15 to 3:25 PM (with a penalty at some brokers), so
`flat_by` defaults to 15:15 and india-trader-alerts rings then.

Whether a stock is allowed for MIS, and at what leverage, depends on the broker's own list.
Check it in the app before taking the trade.

## Steps: the entry

1. `ticket.py show`. Only READY tickets go to the user. A ticket that is NOT READY or
   BLOCKED goes back to `/india-trader-risk`.
2. Show ONE ticket and ask: "Take this trade? Reply yes or no." One order per question.
   Never offer "yes to all".
3. Run `ticket.py approve --index N --reply "<the user's reply, word for word>"`. If it
   prints "Not recorded", the reply was not an explicit yes for this order (for example
   "sure", "yes to all", or a yes that names a different stock or amount). Ask again.
   Never turn a reply into a yes yourself.
4. After "Recorded your yes", show the manual steps (live) or note it as paper.
5. When the user tells you what happened, record it: `ticket.py result --index N --status
   placed --filled-qty Q --filled-price P`, or `--status denied`, or `--status failed` with
   `--note`. A failed order is not retried without a new check and a new yes. The log
   keeps the entry's max loss for india-trader-risk's daily limit.

## Steps: while it is open, and out by the close

- india-trader-alerts watches the stop, the target and the flat-by time
  (`alerts.py add --from-plan`). Alerts are information: the user does the selling.
- When the stop-loss or the target fills, record it:
  `ticket.py fill --symbol RELIANCE --side sell --qty 35 --price 1414.25`.
- By `flat_by` (15:15 IST), anything still open gets closed:
  1. Ask the user how many shares they still hold (their broker's positions page).
  2. `ticket.py flatten --symbol RELIANCE --qty 35` writes a market sell for those shares to
     `india-trader/orders.json`.
  3. `/india-trader-risk`. A sell that only closes what the user holds is never blocked by
     the daily limits or the entry window.
  4. `ticket.py show`, then the user's yes for that one order, as above.
  5. Cancel any open stop-loss or target order at the broker first, then sell or square off.

## What it never does

- Never places an order, connects to a broker or asks for API keys, passwords or OTPs.
- Never runs from a scheduled or background task: nobody is there to say yes.
- Never promises a fill price: a market order and a stop fill at whatever the market gives.
- Never switches `"paper"` to live. The user does that themselves.

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

---
name: india-trader-budget
description: >-
  Budget Manager. Sizes every position to your budget and your max loss per
  trade. Reads the trade plan from india-trader-exit (entry, stop, target), finds
  the whole shares that fit inside four limits (max loss per trade, position
  cap, max order, cash after your buffer), says which limit decided it, and
  writes ONE bracket order (entry, stop-loss, take-profit) to
  india-trader/orders.json as a neutral order ticket (symbol, side, qty, stop-loss, take-profit) that
  india-trader-broker turns into manual NSE / MIS steps. A stop
  too far away for your max loss is a plain refusal, not a smaller order.
  Nothing is placed. Use when the user says "how many shares", "size this
  trade", "how much should I buy", "I have ₹1,00,000", "I only want to risk ₹500
  a trade", "set my rules", "keep 10% in cash", or after /india-trader-exit writes
  a plan.
argument-hint: "[budget in rupees]"
---

# india-trader-budget

The Budget Manager turns a plan ("buy RELIANCE at 364.18, stop 358.32") into an
exact order ("buy 8 shares"), inside rules the user sets once. The stop
decides the size: the further away the stop, the fewer shares. It writes a
proposal only. india-trader-risk checks it and india-trader-broker asks the user yes or
no. Day trading loses most people money. Sizing to a max loss keeps each
loss small; it does not make a trade win.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/size.py"                        # india-trader/plan.json + india-trader/config.json
python3 "${CLAUDE_SKILL_DIR}/size.py" --write                # write india-trader/orders.json
python3 "${CLAUDE_SKILL_DIR}/size.py" --limit --write        # a limit order at the plan's entry
python3 "${CLAUDE_SKILL_DIR}/size.py" --limit 364.00 --json  # a limit order at 364.00
python3 "${CLAUDE_SKILL_DIR}/size.py" --weights HDFCBANK=40,KO=60 --config split.json   # weights mode, below
```

`--now "2026-11-04 09:50"` pins the clock (IST). A plan for
another day is refused: make a new one with india-trader-exit.

## What it reads and writes

| file | what |
| --- | --- |
| `india-trader/config.json` | the rules (start from `config.example.json` in this folder) |
| `india-trader/plan.json` | the trade: entry, stop, target, setup, flat-by time (india-trader-exit writes it) |
| `india-trader/prices.json` | optional: a fresh price above the plan's entry means a bigger risk (india-trader-alerts writes it) |
| `india-trader/portfolio.json` | optional: its cash caps the size (india-trader-broker keeps it) |
| `india-trader/orders.json` | the one proposed bracket order it writes with `--write` |

## Step 1: set the rules once

If `india-trader/config.json` does not exist, copy `config.example.json` into
`india-trader/config.json` and ask the user for each value in plain words:

- **budget**: how much money this is for. Only money they can afford to lose.
- **cash_buffer_pct**: how much always stays in cash (default 10).
- **max_loss_per_trade_inr**: the most one trade may lose if the stop fills at the stop (default 50).
- **max_position_pct**, **max_order_inr**, **min_order_inr**: the biggest position (75% of the
  budget), the biggest order (₹4,000) and the smallest order (₹5).
- The day limits india-trader-risk enforces: **max_daily_loss_inr** (150), **max_trades_per_day** (3),
  **min_reward_risk** (1.0), **require_stop** (true), **no_entries_before** / **no_entries_after**
  ("09:30" / "14:30") and **flat_by** ("15:15"), all IST.
- **blocked**: tickers they never want to trade.
- Keep **paper** true until they have watched it work on a paper account.

## Step 2: size the trade

```bash
python3 "${CLAUDE_SKILL_DIR}/size.py" --write
```

Whole shares only: NSE equity shares cannot be fractional. The size is
the smallest of four limits:

| limit | shares |
| --- | --- |
| max loss | `max_loss_per_trade_inr` / risk a share (entry minus stop) |
| position cap | budget x `max_position_pct` / entry |
| max order | `max_order_inr` / entry |
| cash | (budget minus the cash buffer) / entry, never more than the cash in `india-trader/portfolio.json` |

The entry it sizes at: the limit price with `--limit`. For a market order, the
plan's entry, or the price in `india-trader/prices.json` if it is fresh (under 15
minutes old) and higher. A price that ran up since the plan means a bigger
loss at the same stop, so fewer shares. A lower price never shrinks the risk.

It refuses, and writes no order, when: the size is zero shares, the order is
under the minimum, reward:risk is under `min_reward_risk`, the stop is not
below the entry, the target is not above it (the move already happened), the
plan is for another day, the ticker is blocked, or the plan is a short. Say
the reason to the user in one line.

## Step 3: show the user

Show the four limits, which one decided the size, the shares, the rupees
and the max loss at the stop. Say plainly that a stop is an order, not a
promise: in a fast market it can fill below the stop. Then hand over to
`/india-trader-risk`.


## Weights mode (kept from v1)

`--weights HDFCBANK=22.04,KO=10.86` or `--targets FILE` (`targets.example.json`
here) splits a budget across a list by weight: the cash buffer comes off the
top, weights are renormalised to 100, any stock over `max_position_pct` is
pinned at the cap and its excess shared out, slices under the minimum order
are dropped, and amounts round to whole rupees so nothing is left idle by
accident. With `"max_position_pct": 25` and `"fractional": true`, the example
file gives HDFCBANK ₹1,250 (capped), AXP ₹1,119, GOOGL ₹823, KO ₹708, BAC ₹600.
With `"fractional": false` (the example config) it buys whole shares and
needs prices (`--prices india-trader/prices.json`). These are plain buys with no stop: india-trader-risk blocks them while
`require_stop` is true and counts each one against `max_trades_per_day`. Use
a separate config (`--config`) for a long-term split.

## Rules

- Proposals only. This skill never calls a broker tool.
- Never size money the user has not said is for this. Ask for the budget and the max loss.
- Every number comes from the config, the plan or the prices. Nothing is guessed.
- No projections: never say what a trade will make. A target is a plan, not a forecast.
- Free Yahoo prices are unofficial and can lag. The broker's price is the one that fills.

## What it never does

It never places, approves or schedules an order, never raises a limit on its
own, never shrinks a stop to fit more shares, and never sizes options,
margin or short positions.

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

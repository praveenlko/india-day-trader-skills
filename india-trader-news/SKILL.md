---
name: india-trader-news
description: >-
  News Monitor. Tracks breaking news on every stock you watch and flags what
  moves it: Yahoo Finance headlines per ticker (India edition) and how far each stock is
  from yesterday's close. Company filings are not included: SEC EDGAR does not
  cover NSE companies, so check the NSE / BSE corporate announcements yourself. Shows only what is new since the
  last check, newest first, in IST, tags each item (earnings,
  analyst, deal, halt, regulatory, macro...) and never repeats one. Use when
  the user says "any news on RELIANCE", "why is this moving", "what's the news on
  my watchlist", "morning briefing", "pre-market news", "monitor the news",
  "did anything break", or after /india-trader-scanner finds setups.
argument-hint: "[tickers]"
---

# india-trader-news

A chart can look perfect and still be about to gap on an earnings date, a
halt or a lawsuit. Before a setup becomes a trade, this skill reads the two
feeds that carry those first, for every stock on the watchlist, and tells you
only what is new. It also prints how far each stock has moved today, so a
headline and a move sit side by side.

Tool in this folder (standard library Python, no keys):

```bash
python3 "${CLAUDE_SKILL_DIR}/news.py"                         # watchlist + plan + portfolio
python3 "${CLAUDE_SKILL_DIR}/news.py" --tickers RELIANCE TCS --hours 12
python3 "${CLAUDE_SKILL_DIR}/news.py" --json
```

## What it reads and writes

| file / feed | what |
| --- | --- |
| `--tickers`, else `india-trader/watchlist.json` | the scanner's hits (india-trader-scanner) |
| `india-trader/plan.json` | the trade you are planning (india-trader-exit) |
| `india-trader/portfolio.json` | what you hold (india-trader-broker records fills there) |
| Yahoo Finance headline RSS | `https://feeds.finance.yahoo.com/rss/2.0/headline?s=RELIANCE.NS&region=IN&lang=en-IN` |
| Yahoo Finance chart endpoint | the last price against yesterday's close (unofficial, can lag) |
| NSE / BSE corporate announcements | not fetched (no free stable feed). Results, board meetings and dividends: open nseindia.com > Corporate Filings or bseindia.com yourself |
| `india-trader/state/news.json` | what has been shown (kept 30 days) |

## The rules

- **Headlines are leads, not facts.** Before telling the user something
  happened, open the story or the filing and read it. Quote the source and
  link it. Never summarise a paywalled story you could not read: say so.
- **Filings beat headlines.** If a headline and the company's NSE / BSE
  announcement disagree, the announcement is what the company told the exchange.
- **Say what it touches.** Every item lists the tickers it touches and its
  tags: earnings, analyst, legal, deal, capital, people, risk, halt,
  regulatory, macro. Filings marked `!` carry an important 8-K item: earnings
  results (2.02), an officer change (5.02), an impairment (2.06), a delisting
  notice (3.01), past financials no longer reliable (4.02), bankruptcy (1.03)
  or a cyber incident (1.05).
- **MOVING** marks a stock 3 % or more from yesterday's close (`--move-pct`).
  A move with no headline is worth saying out loud too.
- **No predictions.** Report what was published and how far the price is from
  yesterday's close. Never say what the stock will do next.

## Steps

1. Run the check: `python3 "${CLAUDE_SKILL_DIR}/news.py"`. Exit code 1 means
   nothing new. `--all` shows items already seen, `--json` gives everything
   machine readable, `--dry-run` leaves the state alone.
2. Read the items that matter: every `!` filing, anything tagged earnings,
   halt, legal, regulatory or risk, and every MOVING ticker first.
3. Write a short brief, per ticker: what happened, the source and link, the
   move, and one line on why it matters to the setup (for example "earnings
   are this week: an overnight gap can jump past any stop").
4. If a story changes the picture, say so before `/india-trader-entry` and
   `/india-trader-exit`: an earnings date inside the trade, a halt, an offering.
   Never act on a headline alone.

During the session: `/loop 15m /india-trader-news`, or a scheduled task each
morning before the open.


## What it never does

- Never invents or embellishes a story, a quote or a number.
- Never trades on news, and never tells the user to buy or sell because of a
  headline.
- Never hammers a server: one request per feed per ticker, a pause between
  calls.

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

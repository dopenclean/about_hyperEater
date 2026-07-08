# How HyperEater Trades, In Plain English

## The one-line version

HyperEater is a fully mechanical trend-following bot. It does not predict anything
and it does not use any AI. It watches 21 major crypto markets, waits for a price
breakout in the direction of an established trend, buys or sells with a small fixed
risk, cuts losers quickly, and lets the rare big winner run for days or weeks.

Every rule is written in code, tested against 27 months of real market history, and
applied the same way every time. No opinions, no news reading, no gut feeling.

## Why we trade this way

We ran an AI-driven version of this bot for eleven weeks with real money. We kept a
complete record of every decision it made. The verdict was clear: the AI picked
direction no better than a coin flip, and it cost real money in API fees on top of
the trading losses. The full record is preserved in the dashboard under
"Grok era report", every trade, every reason, every dollar.

Before replacing it, we tested the new mechanical rules against 27 months of price
history on the same exchange, including all fees and slippage. Only after the rules
passed that test did we switch the bot over. This document explains those rules.

## The core idea

Crypto markets spend most of their time going nowhere. Once in a while, a market
starts a real trend and moves a very long way. Nobody can reliably predict when
that happens. So instead of predicting, the bot does this:

1. It takes a small, cheap position every time a market pushes into new ground.
2. If the push fails, it loses a small, known amount and moves on.
3. If the push turns into a real trend, it stays in and follows it as far as it goes.

Most attempts fail. That is expected and priced in. Roughly one trade in three or
four works, and the winners are much larger than the losers. One or two exceptional
trends a year carry most of the profit. The job of every rule below is either to
find those trends or to keep the failed attempts cheap.

## Step by step: what the bot does every four hours

The bot works on the 4-hour candle clock. A candle is just the price record of one
4-hour block. Candles close at 00:00, 04:00, 08:00, 12:00, 16:00 and 20:00 UTC.
About two minutes after each close, the bot runs this checklist on all 21 coins.

### Step 1: Is there a trend at all? (the trend gate)

The bot compares the current price with two moving averages: the average price of
the last 50 candles and the last 200 candles (called EMAs, averages that weight
recent prices more).

- Price above the 200 average, and the 50 average also above the 200 average:
  the market is in an uptrend. The bot is only allowed to buy.
- Price below the 200 average, and the 50 also below the 200: downtrend.
  The bot is only allowed to sell short.
- Anything mixed: no trend. The bot does nothing with that coin.

This single rule keeps the bot from fighting the market. It never buys into a
falling market and never shorts a rising one.

### Step 2: Is the market breaking into new ground? (the breakout trigger)

Within an uptrend, the bot waits for the candle to close above the highest price
of the previous 55 candles, which is roughly the last nine days. Closing above
that level means the market just did something it could not do for nine days.
That is the footprint of real buying pressure, and it is how big trends start.
For downtrends the rule is mirrored: a close below the 55-candle low.

The bot only acts on the first close beyond the level. If price has already run
far past it (more than half an ATR, explained next), the bot considers the move
already gone and skips it rather than chase.

### Step 3: How far away does the safety stop go? (measuring with ATR)

ATR is a standard measure of how much a market typically moves in one candle.
Think of it as the market's normal step size. The bot places its stop-loss at
2 ATR away from the entry price. A calm coin gets a tight stop, a wild coin gets
a wide one, so the stop sits outside normal noise in both cases.

### Step 4: How much money goes in? (position sizing)

The bot risks 3 percent of the current account balance per trade. Risk means the
amount lost if the stop is hit, not the position size. The position size is
calculated backwards from the stop distance so that a stop-out always costs about
the same 3 percent.

Because the percentage is taken from the live balance, the dollar risk shrinks
automatically during losing streaks. Ten straight losses cost about 26 percent of
the account, not 30, and the account can never be wiped out by any streak.

Additional limits on every entry: at most 3 positions open at the same time, one
per coin; a position can never be larger than the account's margin cap; and the
position is shrunk if the order book on that coin is too thin to exit cleanly.

### Step 5: The entry itself

The bot places a limit order at the current price. If it does not fill within 30
seconds, the unfilled part is escalated to a market order. The moment the position
exists, the stop-loss is placed directly on the exchange as a standing order. That
matters: the stop lives at the exchange, not inside the bot, so the position stays
protected even if the bot or the server goes down.

### Step 6: Managing the trade (this is where the money is made)

There is no take-profit order. Trends are occasionally enormous and a take-profit
would cut them short. Instead the trade can end in exactly four ways:

1. The initial stop. The breakout failed. Loss: about 1R, meaning one unit of the
   3 percent risk. This is the most common outcome and it is fine.

2. The ratchet ladder. As the trade moves into profit, the stop-loss climbs behind
   it in fixed steps and never moves back:
   - at +1.3R profit, the stop moves to the entry price. From here the trade
     cannot lose anymore.
   - at +2.3R the stop locks in +1R of profit.
   - at +3.2R it locks +2R, and so on up the ladder, one R at a time.
   A winner that reverses still pays out whatever was locked. A winner that keeps
   trending is never interrupted.

3. The trend flip. If the 4-hour trend gate turns fully against the position, the
   bot closes it at market without waiting for the stop. The reason to be in the
   trade no longer exists.

4. The time stop. If 20 candles pass (about three days) and the trade never
   reached even half an R of profit, the bot exits. A breakout that goes nowhere
   for days was a false one, and the money is freed for the next attempt.

Every open position is checked every 10 minutes. A watchdog confirms the stop
order still exists on the exchange and replaces it if anything is missing.

### Step 7: The emergency brake (circuit breaker)

The bot keeps a record of its recent results. If one coin-and-direction combination
takes an extreme run of losses, it is benched for 24 hours. If the whole account
takes losses far beyond anything the strategy produced in testing, all trading
pauses for 48 hours. These thresholds are set wide on purpose: normal losing
streaks are part of the plan and do not trigger anything. The brake exists to catch
malfunctions, not drawdowns.

## What the testing showed

The rules above were simulated over 27 months of real Hyperliquid price history,
with fees and slippage included and with deliberately pessimistic assumptions
wherever the data was ambiguous. The headline results:

- 401 trades, about 16 per month across the whole universe.
- Around 35 percent of trades won. Losers were held for a day or two on average,
  winners for about four days, and the largest winner ran for over four months.
- Total result: +91R. At 3 percent risk on a small account that compounds
  meaningfully, and the same rules scale to any account size.
- Worst peak-to-valley drawdown along the way: about 21R. Losing streaks of 5 to
  8 trades happened regularly and were always recovered within the tested period.
- Both halves of the test period were profitable on their own, and the result held
  up across a grid of nearby parameter settings, which is evidence the rules are
  not tuned to one lucky configuration.

Two honest caveats. First, a large share of the tested profit came from a small
number of huge trend trades, roughly one per year. Months can pass where the bot
only grinds small losses and small wins while waiting for the next one. Patience
is a structural part of this strategy, not a flaw. Second, a backtest is history,
not prophecy. The future can be worse. The sizing rules exist precisely so that a
worse future is survivable.

## What it costs to run

- Market data comes free from the exchange's public API.
- All analysis is computed locally. There are no AI or data subscription costs.
- The only recurring costs are exchange trading fees, roughly one percent of
  account value per month at the tested trade frequency (already included in all
  backtest figures above), and a small server.

## Where to watch it

The private dashboard shows the live account, open positions, the decision log for
every 4-hour cycle, and a radar view of how close each coin currently is to its
breakout trigger. The complete archive of the retired AI era, with every trade and
the reasons it was taken, stays available from the dashboard under "Grok era
report" as a permanent before-and-after record.

## Glossary

- Candle: the price record of one fixed time block (here, 4 hours).
- EMA: a moving average of price that weights recent candles more heavily.
- ATR: the average size of one candle's price movement; the market's step size.
- Breakout: a close beyond the highest high or lowest low of a lookback window.
- R: one unit of planned risk. A trade risking 3 percent that loses 1R lost
  3 percent; a trade that makes 4R made 12 percent.
- Stop-loss: a standing exchange order that closes the position at a set price.
- Drawdown: the drop from an account's peak value to its lowest point after it.

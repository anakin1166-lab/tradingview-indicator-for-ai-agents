# AI Agent Lens

**A TradingView indicator built for AI agents.**

Indicators were built for human eyes: colors, arrows, lines. An AI agent can't trade on colors. AI Agent Lens breaks down every closed candle into clean, structured JSON and sends it to any AI agent the moment the candle closes. Your agent reads the chart candle by candle, understands the context, and turns it into clear trading signals. You make the final call.

It works with any agent. It's built to shine with long-horizon reasoning models like **Claude Fable 5.2**.

Open source · MIT license · Pine Script v6 · No repainting

---

## What your agent receives on every candle

One JSON packet per closed candle with everything a trader looks at:

- **Candle breakdown:** body, wicks, range versus ATR, where it closed in its range, gap, pattern (engulfing, hammer, doji, inside bar and more)
- **The last 5 candles** in the same format, so the agent reads the sequence, not one bar
- **Regime:** `TREND_UP`, `TREND_DOWN`, `RANGE` or `HIGH_VOLATILITY`
- **Trend:** EMA 20/50/200 stack, 200 EMA slope, higher-timeframe bias
- **Momentum and strength:** RSI, MACD histogram, ADX, DI+ and DI-
- **Volume and volatility:** relative volume, ATR, ATR percentile
- **Structure:** swing highs and lows, breakouts and breakdowns
- **Levels:** previous day high and low, VWAP
- **Long and short scores (0 to 100)** with the five factors behind each score
- **Risk map:** entry, ATR stop, 1.5R and 3R targets, and live trade status

Full field list: [docs/PACKET_SCHEMA.md](docs/PACKET_SCHEMA.md) · Example: [examples/sample_packet.json](examples/sample_packet.json)

---

## How it works

```
 TradingView chart                 Your AI agent                    You
 ┌───────────────────┐  webhook   ┌──────────────────────────┐    ┌──────────────┐
 │  AI Agent Lens    │ ─────────► │ reads the JSON packet    │ ─► │ signal:      │
 │  closes a candle, │  JSON per  │ breaks down the candles  │    │ LONG / SHORT │
 │  builds a packet  │  candle    │ applies your rules       │    │ WAIT / SKIP  │
 └───────────────────┘            └──────────────────────────┘    │ entry, stop, │
                                                                  │ targets, why │
                                                                  └──────────────┘
```

1. The indicator runs on your TradingView chart and calculates everything on each closed candle.
2. One TradingView alert sends the packet to your agent's webhook URL.
3. Your agent (Claude Code with Fable 5.2, an n8n flow, your own bot) reads it with the prompt in [agent/AGENT_PROMPT.md](agent/AGENT_PROMPT.md) and returns a signal: decision, confidence, entry, stop, targets, position size, invalidation and why it could fail.

---

## Quick start

### Step 1. Add the indicator to TradingView (1 minute)

1. Open [ai-agent-lens.pine](ai-agent-lens.pine) and copy all of it.
2. In TradingView, open the **Pine Editor** at the bottom of the chart.
3. Delete the default code, paste, and click **Save**, then **Add to chart**.

You'll see EMAs, key levels, LONG/SHORT score labels, entry/stop/target lines, and the **Agent Readout** panel on the right.

### Step 2. Create one alert (1 minute)

1. Open the alert dialog (clock icon, or `Alt + A`).
2. **Condition:** `AI Agent Lens` → `Any alert() function call`.
3. **Notifications:** tick **Webhook URL** and paste your agent's URL (Step 3).
4. Leave the message box as it is. The indicator writes the message itself.
5. Click **Create**.

One alert covers one chart (one symbol on one timeframe). Repeat it for every chart you want your agent to watch.

> Webhook alerts need a paid TradingView plan with two-factor authentication turned on. On the Essential and Plus plans, alerts expire after 2 months, so recreate them when they do.

### Step 3. Connect your agent

Pick one.

#### Option A: Claude Code with Fable 5.2 (recommended)

```bash
git clone https://github.com/roan-maverick/tradingview-indicator-for-ai-agents
cd tradingview-indicator-for-ai-agents

# 1. Start the receiver (Python 3, no installs)
python3 examples/webhook_receiver.py --port 8080 --token YOUR_SECRET

# 2. In a second terminal, give it a public HTTPS address
cloudflared tunnel --url http://localhost:8080
```

Your webhook URL for Step 2 is:

```
https://<your-tunnel-address>/webhook?token=YOUR_SECRET
```

Every packet is saved to `packets/latest.json`, `packets/latest/<SYMBOL>_<TF>.json` and `packets/history.jsonl`.

Now open Claude Code in the same folder:

```bash
claude
```

and ask:

```
Use agent/AGENT_PROMPT.md as your instructions. Read packets/latest.json
and give me the signal for this candle.
```

More things to ask:

```
Read every file in packets/latest/ and rank the charts by setup quality.

Combine the HYPE 1h and 4h packets: the 4h sets direction, the 1h sets timing.

Read the last 50 lines of packets/history.jsonl for BTC and tell me how
the momentum and structure changed over the session.
```

#### Option B: No-code (n8n, Make, Zapier)

1. Create a **Webhook** trigger and copy its URL into the TradingView alert.
2. Add an **AI** step (Claude or any LLM). Use the block in [agent/AGENT_PROMPT.md](agent/AGENT_PROMPT.md) as the system prompt and the webhook body as the user message.
3. Send the reply to Telegram, Discord, Slack or email.

#### Option C: Your own bot

Point the webhook at your server. The body is plain JSON with the schema `ai-agent-lens/v1`. Check the `schema` field, then use the fields directly or pass the packet to your model with the agent prompt.

---

## What a signal looks like

Example output from the agent prompt ([examples/sample_signal.json](examples/sample_signal.json)):

```json
{
  "decision": "LONG",
  "confidence": 74,
  "entry": 70.71,
  "stop": 69.882,
  "target_1": 71.952,
  "target_2": 73.194,
  "candle_read": "Three candles of steady higher closes, a bearish engulfing shakeout that held above the last swing low, then a bullish engulfing on 1.31x volume that closed in the top quarter of its range and above the last swing high.",
  "invalidation": "A 1h close back below 70.48 or a touch of the 69.882 stop",
  "why_this_could_fail": "This could be a relief rally inside a larger daily pullback."
}
```

The example files are illustrative, not live market data or a real trade.

---

## The Agent Readout panel

Everything in the packet is also on the chart, in a panel built to be read at a glance by people and by agents that can see the screen:

| Row | Shows |
|---|---|
| Regime | `TREND_UP`, `TREND_DOWN`, `RANGE`, `HIGH_VOLATILITY` |
| Long / Short score | 0 to 100 |
| EMA stack · slope | `BULLISH · RISING` |
| HTF bias | `UP`, `DOWN`, `FLAT`, `N/A` |
| RSI, MACD, ADX, relative volume, ATR | Current values |
| Candle | Direction and pattern |
| Structure | Bias and breakout event |
| Levels | Previous day high and low, VWAP |
| Trade | Side, status, entry, stop, targets |
| State | `LENS\|HYPEUSDT.P\|60\|TREND_UP\|L92\|S0\|LONG:ACTIVE` |

The same values also appear in TradingView's **Data Window**.

---

## How the scores work

Each score is out of 100: five factors, 20 points each.

| Factor | Long side gets points when |
|---|---|
| Trend | EMA 20 > 50 > 200 and close above EMA 20 |
| Momentum | RSI above 50 (not overbought), MACD histogram positive and rising |
| Strength | DI+ above DI- and ADX at or above 20 |
| Volume | Bullish candle on above-average volume |
| Higher timeframe | 4h close above its 50 EMA |

The short score mirrors it. A **setup** fires when a score crosses 70 on a closed candle and the regime does not point the other way. While a trade is open, repeat signals in the same direction are ignored, so one move gives one signal. Full rules: [docs/SPECIFICATION.md](docs/SPECIFICATION.md).

---

## Settings

| Setting | Default | What it changes |
|---|---|---|
| Setup score threshold | 70 | Higher = fewer, stronger setups |
| Higher timeframe | 240 (4h) | Use D on a 4h chart, W on a daily chart |
| Block setups in HIGH_VOLATILITY | On | Skips setups during volatility spikes |
| Stop distance | 1.5 x ATR | Wider or tighter stops |
| Target 1 / Target 2 | 1.5R / 3R | Reward multiples |
| Send JSON packet | Every bar close | Or only setups and trade updates, or off |
| Recent candles in packet | 5 | 1 to 10 |
| Show packet preview on chart | Off | Prints the live packet next to price, useful for checking your setup |

---

## FAQ

**Does it repaint?**
No. Setups, trade tracking and packets use closed candles only. Higher-timeframe and daily values use only bars that have already closed. Swing points appear 5 bars after the pivot and never move.

**Which markets and timeframes?**
Any symbol on TradingView. Crypto, stocks, forex, futures. Intraday timeframes (5m to 4h) get the most out of VWAP and previous-day levels. Symbols without volume data get `null` volume fields.

**Can I use it on the free TradingView plan?**
Yes. The indicator, panel and Data Window all work. Sending packets to an agent needs webhook alerts, which need a paid plan.

**How many packets will I get?**
One per closed candle per chart. A 1h chart sends 24 a day. Switch "Send JSON packet" to "Setups and trade updates" to receive only the important ones.

**Does it place trades?**
No. It produces data and signals. Your agent can be connected to a broker if you choose, but this repository never places orders.

---

## Repository

```
tradingview-indicator-for-ai-agents/
├── ai-agent-lens.pine           the indicator (Pine Script v6)
├── agent/AGENT_PROMPT.md        system prompt for your AI agent
├── docs/SPECIFICATION.md        every calculation and rule
├── docs/PACKET_SCHEMA.md        every field in the JSON packet
├── examples/webhook_receiver.py receive packets locally (Python 3, no installs)
├── examples/sample_packet.json  example packet
├── examples/sample_signal.json  example agent signal
└── LICENSE                      MIT
```

---

## Disclaimer

This project is for education and research. It is not financial advice. The indicator describes the chart using fixed rules; it does not predict the future. Signals from any AI agent can be wrong. Test on paper first, size small, and never risk money you can't afford to lose.

## License

MIT. Use it, fork it, build on it.

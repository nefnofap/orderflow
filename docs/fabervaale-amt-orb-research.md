# Fabervaale (Fabio Valentini) — AMT / ORB Strategy: Research & Algorithm Framework

> **Disclaimer:** Fabio Valentini's exact numeric rules are proprietary and not fully
> published. This document is a faithful **reconstruction** assembled from his public
> teaching and community-built indicators that replicate his style. Treat every
> threshold as a *starting parameter to backtest*, not a guaranteed setting. This is
> educational material, **not financial advice**. Sources are linked inline; content
> was rephrased for compliance with licensing restrictions.

---

## 0. Who "Faber-VAALE" actually is

"Faber-VAALE" is the online handle **Fabervaale** — i.e. **Fabio Valentini**, a NASDAQ
(NQ/MNQ) futures *scalper* who trades the New York session. The strategy he is known for
is **not** the textbook Toby Crabel "Opening Range Breakout." It is an
**Auction Market Theory (AMT) + order-flow** method built on Volume Profile, VWAP and
Cumulative Volume Delta (CVD). The "ORB" piece is just the *opening-range / initial
balance* phase **inside** his auction framework.

---

## 1. Philosophy — Auction Market Theory (the "why")

The market is a continuous two-way auction searching for the price where the most
business gets done. It alternates between two states:

- **Balance** — price rotates around fair value; range-bound, choppy, deep liquidity,
  compressed volatility. **No edge -> sit out.**
- **Imbalance** — one side overwhelms the other; price trends and travels fast toward a
  *new* area of balance. **This is the tradeable state.**

The most-repeated idea in his community: *most retail traders lose because they trade
inside balance.* So the algorithm must first do **regime detection** (balance vs
imbalance), and only then generate signals.

Refs: [nexusfi AMT](https://nexusfi.com/a/market-structure/auction-market-theory),
[tastylive](https://www.tastylive.com/news-insights/the-market-isn-t-controlled-by-a-single-algo-it-s-an-auction-),
[chartfanatics](https://www.chartfanatics.com/strategies/auction-market-strategy),
[tradezella](https://www.tradezella.com/strategies/auction-market-strategy).

---

## 2. The toolstack (the "what he looks at")

| Tool | What it provides | Key objects |
|---|---|---|
| **Volume Profile** | Where volume traded by *price* | **POC** (fairest price / magnet), **VAH/VAL** (value-area bounds ~70% of volume), **HVN** (magnet/support), **LVN** (rejection / fast-travel vacuum) |
| **VWAP + StdDev bands** | Session fair value + stretch | VWAP, +/-1/2/3 sigma. Band 1 = normal drift, band 2 = stretched, band 3 = exhaustion/snapback |
| **CVD** | Aggressor pressure (buy-market vs sell-market volume) | CVD trend = who controls; **divergence** vs price = weakening move |
| **Footprint / DOM** | Order flow inside the bar | **Absorption** (huge delta, little price move = large passive limit player) and **Exhaustion** (aggression drying up) |

Two order-flow signatures do the heavy lifting:

- **Absorption** = high volume / high delta but little price movement -> a large player is
  soaking up aggression; often a reversal or continuation-after-pause signal.
- **Exhaustion / divergence** = price makes a new high/low but CVD does not -> the move is
  running out of fuel.

Refs: [DeepCharts CVD](https://www.deepcharts.com/helpcenter/deepdom/article/cumulative-volume-delta),
[nexusfi CVD](https://nexusfi.com/wiki/trading-wiki/Cumulative-Delta),
[orderflowlabs footprint](https://orderflowlabs.com/blogs/theblog/footprint-chart-guide),
[VWAP deviation bands](https://theotrade.com/how-to-set-up-and-trade-vwap-deviation-bands/).

---

## 3. The playbook — two mirror-image models

### Model A — Trend Continuation ("breakout seeking new balance")
- **Context:** price leaving a balance area -> imbalance beginning.
- **Trigger zone:** a key level breaks (VAH/VAL, prior balance edge, opening range) and
  price *accepts* beyond it.
- **Confirmation:** CVD pushing in the breakout direction; LVN ahead (vacuum to travel);
  ideally a pullback/retest that holds.
- **Target:** next HVN/POC or value edge where a new balance is likely to form.

### Model B — Mean Reversion ("failed auction back to POC")
- **Context:** a breakout *attempt fails* — price pokes outside value or stretches to VWAP
  band 2-3 and cannot get acceptance.
- **Trigger zone:** value-area edge or an extreme VWAP deviation.
- **Confirmation:** **absorption** at the extreme + CVD **divergence** -> failed auction.
- **Target:** back to **POC / VWAP** (the magnet).

Decision tree is symmetric: *Is value being accepted (-> continuation) or rejected
(-> reversion)?* Order flow (CVD / absorption) is the tiebreaker.

---

## 4. Execution, session, risk, edge cases

- **Instrument / session:** NASDAQ futures (NQ/MNQ), New York session focus.
- **Style:** scalper — many trades, small targets, tight stops, very tight drawdown, then
  **compounding** size as equity grows.
- **Entry archetypes:** *risk entry* (at the level, lower win rate) vs *confirmation
  entry* (wait for order flow, higher win rate — he leans here).
- **Stops:** structural — just beyond the level/swing that defines the idea. If violated
  with acceptance, the thesis is dead.
- **Targets:** next auction reference — POC, opposite value edge, VWAP, next HVN; partials
  at first magnet, runner toward the next.
- **The no-trade filter is the edge:** when in balance, do nothing. This keeps drawdown
  small.

### The opening range / "ORB" mapping
The first minutes of the NY session define the **initial balance / opening range** — just
the day's *first* balance area. The "ORB" trade = a **Model A continuation** when price
accepts outside the opening range with CVD confirming; fading a *failed* opening-range
break = **Model B**. No separate ORB engine is needed.

### Edge cases
- **False breakout / liquidity sweep:** poke + no acceptance + CVD divergence -> that's a
  Model B signal, not a failure.
- **News spikes:** avoid entries into scheduled high-impact releases.
- **Balance days:** reduce size or stand down.

---

## 5. Algorithm framework (6 layers)

```
Layer 0  DATA & SESSION CLOCK     bars, volume, (bid/ask or tick delta), anchor reset
Layer 1  FEATURE ENGINE           VWAP+bands, Volume Profile (POC/VAH/VAL/HVN/LVN),
                                  CVD, absorption/exhaustion, Opening Range, ATR, PDH/PDL
Layer 2  REGIME CLASSIFIER        BALANCE vs IMBALANCE  (gates everything)
Layer 3  SETUP DETECTORS          Model A continuation / Model B reversion
Layer 4  CONFLUENCE SCORER        weighted sum >= threshold => emit signal
Layer 5  RISK / TRADE MANAGER     entry, structural stop, targets, sizing, caps, cooldown
```

### Feature definitions (OHLCV-only friendly)
- **Proxy delta per bar:** `delta = volume * ((close-low) - (high-close)) / (high-low)`;
  `CVD = cumulative sum(delta)` (intraday reset). Or use lower-timeframe up/down volume
  for a truer delta.
- **Absorption:** `volume > k1*avgVolume(N)` AND `range < k2*ATR` (start k1~1.8, k2~0.5).
- **Exhaustion/divergence:** new price extreme over M bars without a new CVD extreme.
- **Volume Profile:** session histogram by price bin -> POC (max bin), VAH/VAL (~70%),
  LVN (local minima), HVN (local maxima).
- **VWAP bands:** session-anchored VWAP +/- {1,2,3} x stdev of typical price.
- **Opening Range:** high/low of first `OR_minutes` after the anchor open (test 5/15/30).
- **Regime metric:** value-area width / ATR + CVD slope. Narrow + flat = balance.

### Confluence scorer (the signal)
| Factor | Weight |
|---|---|
| Regime aligned (imbalance for A / clean failed-auction for B) | 0.30 |
| CVD aligned / diverging | 0.25 |
| At a real structural level (POC/VA edge/OR/LVN) | 0.15 |
| Absorption present | 0.15 |
| Confirming bar (acceptance / rejection wick) | 0.10 |
| LVN vacuum (continuation) | 0.05 |

Emit when score >= threshold (start 0.60).

### Risk management
- **Sizing:** fixed-fractional `size = equity*risk_pct / (|entry-stop| * point_value)`;
  start 0.25-0.5%; compound off live equity.
- **Stop:** structural (beyond level / swing / absorption candle / LVN).
- **Targets:** TP1 = first magnet (partials + move to BE); TP2 = next value edge/HVN.
- **Guardrails:** max trades/day, daily loss limit, cooldown after a trade, no entries
  late in session, flatten before close (intraday).

---

## 6. Pine Script v6 implementation

See [`fabervaale_amt_orb_strategy.pine`](../fabervaale_amt_orb_strategy.pine). Highlights:

- **Multi-instrument:** futures, stocks, FX and crypto **perpetuals**. "24h" mode anchors
  VWAP/profile/opening-range to exchange midnight for 24/7 perps.
- **Adaptive profile bin width** from the prior day's range -> meaningful POC/VAH/VAL on
  any symbol without manual tuning. Built with `map<int,float>` (no deep-history indexing).
- **CVD** via OHLC proxy (works everywhere) or lower-timeframe intrabar delta.
- Full **risk manager**: risk-% sizing (fractional for crypto), structural stops,
  R/magnet targets, scale-out, daily caps, cooldown, EOD flatten, time-exit.
- Chart visuals (VWAP bands, POC/VAH/VAL, OR, PDH/PDL), regime shading, and
  `alertcondition`/`alert()` for webhook automation.

### Suggested starting presets
- **NQ/MNQ (RTH):** Session mode, OR=15m, `0930-1600` America/New_York, CVD=Intrabar 1m,
  Risk 0.5%, EOD flatten on.
- **BTCUSDT.P / ETH perps:** 24h mode, OR=15-30m (or disabled), fractional qty on, EOD
  flatten off, time-exit optional.

### Validation plan
1. Walk-forward optimize (`thr`, weights, `orMinutes`, `expFactor`, `absVolFactor`) —
   never a single in-sample fit.
2. Model realistic commission + slippage (scalping P&L is cost-sensitive).
3. Check performance **split by regime** — should be near-flat in balance.
4. Forward-test on sim before live capital.

---

## 7. Source list

- Auction Market Theory: nexusfi, tastylive, financialtechwiz
- Fabio Valentini method: chartfanatics, tradezella, forex.in.rs, econolearn (Fabervaale)
- CVD / order flow: DeepCharts, nexusfi, orderflowlabs, TradingView community scripts
  (CVD Absorption/Exhaustion; "Fabio Valentini Pro Scalper" by PickMyTrade)
- VWAP deviation bands: theotrade, financialtechwiz

*Content rephrased for compliance with licensing restrictions; attribution provided via
inline links.*

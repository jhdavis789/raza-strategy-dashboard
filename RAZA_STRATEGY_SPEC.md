# RAZA Strategy Language (RSL) Specification

Version 1.0

## Overview

RSL is a JSON-based language for describing systematic trading strategies over leveraged ETFs and macro signals. You write a strategy as a JSON object, paste it into the RAZA dashboard, and it backtests the strategy against historical data, showing performance versus benchmarks.

**Workflow:** Give this document to an LLM along with your trading ideas. The LLM generates RSL JSON. You paste it into the RAZA dashboard Strategy tab and click Run Backtest.

**LLM instructions:** When a user gives you this document and asks you to help build a strategy, your job is to translate their ideas into valid RSL JSON. Ask clarifying questions to sharpen the strategy — but keep it to 1-2 focused questions at a time, not a barrage. Good questions: "What should we do when the signal is ambiguous — stay in the current position or go to cash?" or "Do you want a volatility overlay to scale down in choppy markets?" Don't ask obvious things the user has already answered. Once you have enough to start, generate the JSON and suggest they test it — they can always iterate.

**Architecture:** A strategy has four sections:
- **signals** — named data pipelines (source → transforms)
- **layers** — independent strategy sleeves that run in parallel
- **overlays** — portfolio-level risk adjustments
- **config** — backtest parameters

---

## Top-Level Schema

```json
{
  "signals": { ... },
  "layers": [ ... ],
  "overlays": [ ... ],
  "config": { ... }
}
```

All four keys are required. `overlays` can be an empty array `[]`.

---

## Signals

Named pipelines that transform a raw data source into a trading signal. Each signal has a `source` and a `transforms` array (can be empty for raw data).

```json
"signals": {
  "my_signal_name": {
    "source": "FRED:VIXCLS",
    "transforms": [
      {"fn": "zscore", "window": 252}
    ]
  }
}
```

Signal names are used in rule conditions and must be valid JS identifiers (letters, numbers, underscores).

### Sources

| Prefix | Example | Description |
|--------|---------|-------------|
| `ETF:<TICKER>` | `ETF:TQQQ` | Daily close price of an ETF or index |
| `FRED:<SERIES>` | `FRED:VIXCLS` | FRED economic data series (must be loaded in Signals tab first) |
| `SIGNAL:<name>` | `SIGNAL:vix_z` | Reference another named signal (for composition) |
| `SMA:<source>:<window>` | `SMA:ETF:SPY:200` | Shorthand: SMA of a source |
| `EMA:<source>:<window>` | `EMA:ETF:SPY:50` | Shorthand: EMA of a source |
| `RETURNS:<source>:<window>` | `RETURNS:ETF:SPY:21` | Shorthand: N-day return |
| `VOL:<source>:<window>` | `VOL:ETF:TQQQ:60` | Shorthand: rolling annualized volatility |

**Available ETF tickers:** TQQQ, SOXL, SPXL, UPRO, TNA, UDOW, TECL, FNGU, SQQQ, SPXS, SPXU, TZA, SDOW, TECS, LABU, FAS, ERX, CURE, DPST, NAIL, DFEN, RETL, WANT, FAZ, ERY, WEBS, UCO, SCO, BOIL, KOLD, NUGT, DUST, AGQ, ZSL, UGL, GLL, TMF, TMV, TYD, TYO, QQQ, SPY, IWM, DIA, XLF, XLE, XLV, XBI, XHB, ITA, XRT, SMH, GDX, SLV.

**Available FRED series** (pre-loaded in the dashboard — no API key needed):
- Rates: DGS2, DGS5, DGS10, DGS30, DFF, SOFR
- Spreads: T10Y2Y, T10Y3M, BAMLC0A0CM, BAMLH0A0HYM2
- Volatility: VIXCLS, VXNCLS (NASDAQ-100 VIX), VXDCLS (DJIA VIX), RVXCLS (Russell 2000 VIX)
- Macro: UNRATE, CPIAUCSL, UMCSENT, INDPRO, PAYEMS
- Monetary: M2SL, WALCL
- Dollar: DTWEXBGS
- Indexes: NASDAQCOM, NASDAQ100, NASDAQXNDX (NDX Total Return), NASDAQXCMP (Composite TR), DJIA, SP500, NASDAQNQGI (Global), NASDAQNQUS500LCT (US 500 LC TR)
- Sectors: NASDAQNDXT (NDX Tech), NASDAQSOX (Semiconductor), NASDAQBANK, NASDAQIXF (Financial 100), NASDAQIXHC (Health Care), NASDAQCELS (Clean Energy), NASDAQNYGBIG (AI & Big Data), NASDAQNQUSS (US Small Cap), NASDAQNQMAREITT (REIT TR)

**Beyond the pre-loaded series:** FRED has 800,000+ data series. The NASDAQ family alone has 9,500+ daily indexes covering every sector, geography, factor, and theme imaginable — biotechnology, cybersecurity, water, dividends, momentum, gold miners, and more. Any valid FRED series ID works as a signal source via `FRED:<SERIES_ID>`. The dashboard will auto-fetch it when you run the backtest (requires a free FRED API key).

**LLM guidance for discovering FRED series:** When building a strategy, think about what economic or market conditions should drive the decision. Then search FRED for relevant series. Examples:
- User says "I want to be defensive when credit is deteriorating" → use `FRED:BAMLH0A0HYM2` (HY spreads) or search FRED for sector-specific credit indexes
- User says "go risk-on when semiconductors are leading" → use `FRED:NASDAQSOX` (PHLX Semi index) or the embedded SOXL ETF data
- User says "I want to trade around NASDAQ biotech trends" → search FRED for `NASDAQBIOTECHNOLOGY` or similar NASDAQ sector indexes
- User says "use a clean energy signal" → `FRED:NASDAQCELS` is pre-loaded, or search for other NASDAQ green/renewable series

To find series IDs, search the FRED website (fred.stlouisfed.org) or use the FRED API search endpoint. All daily-frequency FRED series work. Monthly/quarterly series are forward-filled to trading dates automatically.

### Transform Functions

Transforms are applied in order. Each takes the array from the previous step and outputs a new array of the same length (aligned to trading dates). Null values propagate — if an input is null, the output is null for that day.

| fn | params | formula | notes |
|----|--------|---------|-------|
| `returns` | `window: int` | `x[t] / x[t-N] - 1` | N-day percentage return |
| `log_returns` | `window: int` | `ln(x[t] / x[t-N])` | N-day log return |
| `sma` | `window: int` | `(1/N) * sum(x[t-N+1..t])` | Simple moving average |
| `ema` | `window: int` | EMA with alpha = 2/(N+1) | Exponential moving average |
| `zscore` | `window: int` | `(x[t] - mean(x[t-N..t-1])) / stdev(x[t-N..t-1])` | Rolling z-score, trailing only (no look-ahead) |
| `pctile` | `window: int` | Fraction of values in [t-N..t-1] that are <= x[t] | Rolling percentile rank [0, 1] |
| `diff` | — | `x[t] - x[t-1]` | First difference |
| `lag` | `periods: int` | `x[t-N]` | Shift signal backward in time |
| `abs` | — | `abs(x[t])` | Absolute value |
| `sign` | — | `-1, 0, or +1` | Sign of the value |
| `clip` | `min: num, max: num` | `clamp(x[t], min, max)` | Restrict to range |
| `scale` | `value: num` | `x[t] * value` | Multiply by constant |
| `add` | `value: num` | `x[t] + value` | Add constant |
| `ratio` | `ref: source_string` | `x[t] / ref[t]` | Divide by another data source |
| `subtract_ref` | `ref: source_string` | `x[t] - ref[t]` | Subtract another data source |
| `rolling_max` | `window: int` | `max(x[t-N+1..t])` | Rolling maximum |
| `rolling_min` | `window: int` | `min(x[t-N+1..t])` | Rolling minimum |
| `rolling_vol` | `window: int` | `stdev(daily_returns[t-N..t]) * sqrt(252)` | Rolling annualized volatility |
| `drawdown` | — | `x[t] / cumulative_max(x[0..t]) - 1` | Current drawdown from peak (always <= 0) |
| `crossover` | `ref: source_string` | `+1` if crossed above, `-1` if below, `0` else | Detects crossings |
| `invert` | — | `1 / x[t]` | Reciprocal |
| `normalize` | `window: int` | `(x[t] - rolling_min) / (rolling_max - rolling_min)` | Normalize to [0, 1] over window |

**Example: price-to-SMA ratio as a trend signal:**
```json
{
  "source": "ETF:SPY",
  "transforms": [
    {"fn": "ratio", "ref": "SMA:ETF:SPY:200"},
    {"fn": "add", "value": -1}
  ]
}
```
This gives: `SPY_price / SPY_200d_SMA - 1`. Positive = above SMA, negative = below.

---

## Layers

Layers are independent strategy sleeves that run in parallel. Each layer manages its own slice of capital.

```json
"layers": [
  {
    "name": "Core Equity",
    "weight": 0.60,
    "rules": [ ... ]
  },
  {
    "name": "Bond Hedge",
    "weight": 0.25,
    "rules": [ ... ]
  }
]
```

- `name` (string) — descriptive label, shown in trade log
- `weight` (number) — fraction of capital allocated to this layer
- `rules` (array) — ordered rule list (see below)

**Position aggregation:** On each day, each layer produces its own positions. The final portfolio position for each ticker is:

```
final_weight[ticker] = sum over all layers of (layer.weight * layer_position[ticker])
```

Weights do NOT need to sum to 1.0. If they sum to less, the remainder is implicitly cash. If they sum to more, the strategy is levered.

---

## Rules

Rules are the decision logic within each layer. They form a **tree** — rules can nest to arbitrary depth, enabling regimes within regimes.

### Basic rule (leaf node):
```json
{
  "name": "Go Long",
  "when": {"gt": ["momentum", 0]},
  "positions": {"TQQQ": 1.0}
}
```

### Nested rule (branch node):
```json
{
  "name": "Bull Regime",
  "when": {"gt": ["spy_mom", 0]},
  "rules": [
    {
      "name": "Low Vol Bull",
      "when": {"lt": ["vix_z", 0.5]},
      "positions": {"TQQQ": 1.0}
    },
    {
      "name": "High Vol Bull",
      "when": true,
      "positions": {"UPRO": 0.7, "SPY": 0.3}
    }
  ]
}
```

A rule has either `positions` (leaf — terminal allocation) or `rules` (branch — recurse deeper). Never both.

**Evaluation:** Rules are evaluated top-to-bottom. The first rule whose `when` condition is true is selected. If it's a branch, recurse into its sub-rules. If no rule matches, the layer is 100% cash for that day.

**Always include a fallback:** The last rule in any list should have `"when": true` to ensure something always matches.

### Positions

A dict mapping ticker symbols to weights within this layer:
```json
"positions": {"TQQQ": 0.7, "TMF": 0.3}
```
- Positive weight = long position
- Negative weight = short position
- `"_cash": 1.0` or `{}` = all cash
- Weights within a single rule's positions should typically sum to 1.0 (representing 100% of this layer's capital), but this is not enforced

### Condition Operators

Conditions are nested JSON expressions. Operands can be: a signal name (string), a number, or an inline source string.

| Operator | Syntax | Meaning |
|----------|--------|---------|
| `gt` | `{"gt": ["signal_name", 0.5]}` | signal > 0.5 |
| `lt` | `{"lt": ["signal_name", -1.0]}` | signal < -1.0 |
| `gte` | `{"gte": ["signal_name", 0]}` | signal >= 0 |
| `lte` | `{"lte": ["signal_name", 100]}` | signal <= 100 |
| `eq` | `{"eq": ["signal_name", 0]}` | signal == 0 (within 1e-9) |
| `and` | `{"and": [cond1, cond2, ...]}` | all conditions true |
| `or` | `{"or": [cond1, cond2, ...]}` | any condition true |
| `not` | `{"not": condition}` | negate |
| `between` | `{"between": ["sig", 0.2, 0.8]}` | 0.2 <= sig <= 0.8 |
| `cross_above` | `{"cross_above": ["sig", 0]}` | sig was <= 0 yesterday, > 0 today |
| `cross_below` | `{"cross_below": ["sig", 0]}` | sig was >= 0 yesterday, < 0 today |
| `true` | `true` | always matches (use as fallback) |

**Comparing two signals:**
```json
{"gt": ["fast_sma", "slow_sma"]}
```
Both operands are looked up as signal names. If a name isn't found in signals, it's treated as a number.

---

## Overlays

Portfolio-level adjustments applied after all layer positions are aggregated. Processed in order.

```json
"overlays": [
  {"type": "vol_target", "target": 0.15, "lookback": 60, "max_leverage": 2.0, "min_exposure": 0.1},
  {"type": "drawdown_stop", "threshold": -0.20, "resume_after": 10},
  {"type": "position_limit", "max_single": 0.50, "max_gross": 2.0}
]
```

| Type | Params | Effect |
|------|--------|--------|
| `vol_target` | `target` (annualized), `lookback` (days), `max_leverage`, `min_exposure` | Scales all positions by `target / realized_vol`. Realized vol = stdev of portfolio daily returns over lookback × sqrt(252). Scale factor capped at max_leverage, floored at min_exposure. |
| `drawdown_stop` | `threshold` (e.g. -0.20), `resume_after` (days) | When strategy drawdown from peak exceeds threshold, override all positions to cash. Resume normal trading after resume_after days. |
| `position_limit` | `max_single`, `max_gross` | Cap any single ticker's absolute weight at max_single. Cap total gross exposure (sum of absolute weights) at max_gross. If exceeded, scale all positions proportionally. |

Use an empty array `[]` for no overlays.

---

## Config

```json
"config": {
  "name": "My Strategy Name",
  "description": "Brief description of the approach",
  "start": "2012-01-01",
  "end": null,
  "benchmarks": ["SPY", "QQQ"],
  "cost_bps": 5,
  "rebalance": "daily"
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | required | Strategy name (shown in charts) |
| `description` | string | `""` | Description |
| `start` | date string or null | `null` (= earliest data) | Backtest start date |
| `end` | date string or null | `null` (= latest data) | Backtest end date |
| `benchmarks` | string[] | `["SPY"]` | Tickers to compare against |
| `cost_bps` | number | `5` | Round-trip transaction cost in basis points, applied on each position change |
| `rebalance` | string | `"daily"` | Rebalance frequency (only "daily" supported in v1) |
| ~~risk_free~~ | — | — | **Not configurable.** Derived automatically from market data (Fed Funds Rate via FRED:DFF). Sharpe/Sortino always use actual market rates. |

---

## Complete Examples

### Example 1: Simple Momentum

Long TQQQ when SPY has positive 60-day momentum, otherwise hold SPY.

```json
{
  "signals": {
    "spy_mom": {
      "source": "RETURNS:ETF:SPY:60",
      "transforms": []
    }
  },
  "layers": [
    {
      "name": "Momentum",
      "weight": 1.0,
      "rules": [
        {"name": "Risk-on", "when": {"gt": ["spy_mom", 0]}, "positions": {"TQQQ": 1.0}},
        {"name": "Risk-off", "when": true, "positions": {"SPY": 1.0}}
      ]
    }
  ],
  "overlays": [],
  "config": {
    "name": "Simple Momentum",
    "description": "TQQQ when SPY 60d return > 0, else SPY",
    "benchmarks": ["SPY", "QQQ", "TQQQ"],
    "cost_bps": 5
  }
}
```

### Example 2: VIX Mean Reversion with Nested Regimes

Three-level regime tree: market direction → volatility level → trend strength.

```json
{
  "signals": {
    "spy_mom": {"source": "RETURNS:ETF:SPY:60", "transforms": []},
    "vix_z": {"source": "FRED:VIXCLS", "transforms": [{"fn": "zscore", "window": 252}]},
    "tqqq_trend": {
      "source": "ETF:TQQQ",
      "transforms": [{"fn": "ratio", "ref": "SMA:ETF:TQQQ:50"}, {"fn": "add", "value": -1}]
    }
  },
  "layers": [
    {
      "name": "VIX Regime",
      "weight": 1.0,
      "rules": [
        {
          "name": "Bull Market",
          "when": {"gt": ["spy_mom", 0]},
          "rules": [
            {
              "name": "Low Vol Bull",
              "when": {"lt": ["vix_z", 0.5]},
              "rules": [
                {"name": "Strong uptrend", "when": {"gt": ["tqqq_trend", 0.02]}, "positions": {"TQQQ": 1.0}},
                {"name": "Mild uptrend", "when": true, "positions": {"UPRO": 0.7, "QQQ": 0.3}}
              ]
            },
            {"name": "High Vol Bull", "when": true, "positions": {"SPY": 0.8, "TMF": 0.2}}
          ]
        },
        {
          "name": "Bear Market",
          "when": true,
          "rules": [
            {"name": "Crash", "when": {"gt": ["vix_z", 2.0]}, "positions": {"_cash": 1.0}},
            {"name": "Mild bear", "when": true, "positions": {"SPY": 0.3, "TMF": 0.4, "GDX": 0.3}}
          ]
        }
      ]
    }
  ],
  "overlays": [],
  "config": {
    "name": "VIX Regime Strategy",
    "description": "Nested regime tree: direction -> vol -> trend",
    "benchmarks": ["SPY", "QQQ"],
    "cost_bps": 5,
    "start": "2011-01-01"
  }
}
```

### Example 3: Multi-Layer Portfolio with Hedging

Three independent layers: equity core, rate hedge, tail protection.

```json
{
  "signals": {
    "spy_mom": {"source": "RETURNS:ETF:SPY:60", "transforms": []},
    "hy_spread_z": {"source": "FRED:BAMLH0A0HYM2", "transforms": [{"fn": "zscore", "window": 504}]},
    "curve": {"source": "FRED:T10Y2Y", "transforms": []},
    "vix_level": {"source": "FRED:VIXCLS", "transforms": []}
  },
  "layers": [
    {
      "name": "Core Equity",
      "weight": 0.60,
      "rules": [
        {
          "name": "Risk-on",
          "when": {"and": [{"gt": ["spy_mom", 0]}, {"lt": ["hy_spread_z", 1.5]}]},
          "positions": {"TQQQ": 0.5, "SOXL": 0.3, "UPRO": 0.2}
        },
        {"name": "Defensive", "when": true, "positions": {"SPY": 0.6, "QQQ": 0.4}}
      ]
    },
    {
      "name": "Rate Hedge",
      "weight": 0.25,
      "rules": [
        {"name": "Curve steep", "when": {"gt": ["curve", 0.5]}, "positions": {"TMF": 1.0}},
        {"name": "Curve flat/inverted", "when": true, "positions": {"TMV": 0.5, "_cash": 0.5}}
      ]
    },
    {
      "name": "Tail Protection",
      "weight": 0.15,
      "rules": [
        {"name": "High VIX", "when": {"gt": ["vix_level", 25]}, "positions": {"_cash": 1.0}},
        {"name": "Normal", "when": true, "positions": {"SQQQ": 0.3, "TMF": 0.7}}
      ]
    }
  ],
  "overlays": [
    {"type": "vol_target", "target": 0.20, "lookback": 60, "max_leverage": 1.5, "min_exposure": 0.2}
  ],
  "config": {
    "name": "Multi-Layer Hedged Portfolio",
    "description": "Equity core (60%) + rate hedge (25%) + tail protection (15%) with vol targeting",
    "benchmarks": ["SPY", "QQQ"],
    "cost_bps": 10,
    "start": "2012-01-01"
  }
}
```

### Example 4: Pairs / Relative Value

Long financials when credit conditions are easy, short when stressed.

```json
{
  "signals": {
    "ig_z": {"source": "FRED:BAMLC0A0CM", "transforms": [{"fn": "zscore", "window": 252}]},
    "fin_mom": {"source": "RETURNS:ETF:XLF:21", "transforms": []}
  },
  "layers": [
    {
      "name": "Credit Regime",
      "weight": 1.0,
      "rules": [
        {
          "name": "Tight spreads + momentum",
          "when": {"and": [{"lt": ["ig_z", -0.5]}, {"gt": ["fin_mom", 0]}]},
          "positions": {"FAS": 0.7, "SQQQ": 0.3}
        },
        {
          "name": "Wide spreads",
          "when": {"gt": ["ig_z", 1.0]},
          "positions": {"FAZ": 0.5, "TMF": 0.5}
        },
        {"name": "Neutral", "when": true, "positions": {"SPY": 0.5, "XLF": 0.5}}
      ]
    }
  ],
  "overlays": [
    {"type": "position_limit", "max_single": 0.70, "max_gross": 1.5}
  ],
  "config": {
    "name": "Credit Regime Pairs",
    "description": "Long FAS / short SQQQ when credit easy; long FAZ / TMF when stressed",
    "benchmarks": ["SPY", "XLF"],
    "cost_bps": 10
  }
}
```

---

## Patterns Cookbook

### Momentum
```json
"source": "RETURNS:ETF:SPY:60", "transforms": []
// Condition: {"gt": ["spy_mom", 0]}
```

### Mean Reversion
```json
"source": "FRED:VIXCLS", "transforms": [{"fn": "zscore", "window": 252}]
// Buy when z > 2 (signal elevated, expect reversion), sell when z < -1
```

### Trend Following (price vs SMA)
```json
"source": "ETF:SPY",
"transforms": [{"fn": "ratio", "ref": "SMA:ETF:SPY:200"}, {"fn": "add", "value": -1}]
// Positive = above 200d SMA (uptrend)
```

### Carry / Yield Curve
```json
"source": "FRED:T10Y2Y", "transforms": []
// Positive = normal curve (carry positive), negative = inverted (recession risk)
```

### Hedged Beta
One layer long equity, another layer short via inverse ETF:
```json
"layers": [
  {"name": "Long", "weight": 0.7, "rules": [...]},
  {"name": "Hedge", "weight": 0.3, "rules": [{"when": true, "positions": {"SQQQ": 1.0}}]}
]
```

### Volatility Targeting
Use the `vol_target` overlay to dynamically scale exposure:
```json
"overlays": [{"type": "vol_target", "target": 0.12, "lookback": 60, "max_leverage": 2.0, "min_exposure": 0.1}]
```

### Drawdown Circuit Breaker
```json
"overlays": [{"type": "drawdown_stop", "threshold": -0.15, "resume_after": 5}]
```

### Multi-Regime (nested)
Nest `rules` inside `rules` for regime-within-regime logic. There is no depth limit.

---

## Constraints & Limitations

1. **No intraday data** — all signals and returns are daily close-to-close
2. **No short selling costs** — short positions via inverse ETFs (SQQQ, etc.) rather than direct shorts; ETF expense ratios are already embedded in prices
3. **Forward-fill only** — FRED signals are forward-filled to trading dates; gaps appear as stale values
4. **No fractional rebalancing** — positions snap to target weights each day (no drift)
5. **Transaction costs are simplified** — flat bps on notional change, no market impact or slippage modeling
6. **FRED auto-loading** — the dashboard will automatically fetch any FRED series referenced in your strategy when you click Run Backtest (requires a free API key for non-embedded series). 39 common series are pre-embedded and need no key.
7. **Cash earns 0%** — the `_cash` position returns nothing. The risk-free rate for Sharpe/Sortino is derived from market data (Fed Funds Rate) automatically — never user-specified.
8. **Daily rebalance only** — weekly/monthly coming in v2

## Extending RSL

The engine is designed for extensibility:
- **New data sources:** Add a new prefix handler in `resolveSource()` (e.g., `CRYPTO:BTC`)
- **New transforms:** Add a case in `applyTransform()` — input array in, output array out
- **New overlays:** Add a case in `applyOverlay()` — positions in, adjusted positions out
- **New condition operators:** Add a case in `evalCondition()`

Each extension point is a single function with a switch/case pattern.

---

## LLM Strategy Development Guide

When helping a user build a strategy, follow this approach:

**1. Start with their idea, not with questions.** If the user gives you enough to write an initial strategy, write it. A testable first draft is more valuable than a perfect plan. They can iterate after seeing results.

**2. Ask 1-2 sharpening questions when needed.** Good times to ask:
- "What asset should this go to when conditions are bad — cash, bonds, or inverse?" (risk-off behavior)
- "Should we scale position sizes with conviction, or go all-in/all-out?" (binary vs. graduated)
- "Any sectors or themes you want to tilt toward?" (helps pick FRED signals)

Don't ask more than two questions before generating something testable. Avoid asking things the user has already implied.

**3. Suggest signals they haven't considered.** When a user describes a macro view, think about what data could express it:
- Yield curve shape → T10Y2Y, T10Y3M
- Credit stress → BAMLH0A0HYM2 (HY OAS), BAMLC0A0CM (IG OAS)
- Sector momentum → NASDAQSOX, NASDAQNDXT, NASDAQBANK, NASDAQIXHC, etc.
- Risk appetite → VIXCLS, VXNCLS, compare to historical z-scores
- The NASDAQ index family on FRED has 9,500+ daily indexes — if the user cares about a specific theme (AI, clean energy, biotech, REITs, dividends, water, transportation), there's likely a NASDAQ index for it. Search FRED for `NASDAQ` + the theme keyword.

**4. After generating the JSON, suggest next steps.** Brief, not exhaustive:
- "Try adjusting the start date to see how it performs in different regimes"
- "You could add a vol_target overlay to smooth the ride"
- "Consider splitting into layers if you want to hedge rates separately"

**5. Iterate based on results.** When the user reports backtest results, help them interpret and refine. If Sharpe is low, ask about the drawdown profile. If turnover is high, suggest wider thresholds. Keep suggestions actionable and concrete.

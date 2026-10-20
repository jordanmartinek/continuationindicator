# PACPE — Price Action Continuation Probability Engine

A momentum-aware **price-action analysis engine** for TradingView (Pine Script
v6) that estimates how strongly current market behaviour supports *continuation*
of the existing directional move over the next several candles.

It is deliberately **not** a moving-average crossover, a bullish/bearish candle
counter, or a conventional trend follower. It continuously measures the evolving
balance between **directional continuation evidence** and **opposing
(countertrend) pressure**, so that:

- strong, established momentum is *resistant* to minor countertrend candles, and
- repeated or violent countertrend evidence *progressively* erodes confidence
  and can trigger a disciplined transition to neutral/opposite conditions.

> **Honesty first.** The two 0–100 outputs are **heuristic continuation
> scores**, not validated probabilities. An optional calibration module reports
> *empirical* continuation rates per score bucket and is always displayed
> separately from the scores. See [Scores vs. probabilities](#5-scores-vs-probabilities).

---

## Table of contents

1. [Installation](#1-installation)
2. [Scoring architecture](#2-scoring-architecture)
3. [Formulas](#3-formulas)
4. [How countertrend candles affect momentum](#4-how-countertrend-candles-affect-momentum)
5. [Scores vs. probabilities](#5-scores-vs-probabilities)
6. [Recommended defaults for intraday futures](#6-recommended-defaults-for-intraday-futures)
7. [Testing checklist (NQ / ES)](#7-testing-checklist-nq--es)
8. [Required-scenario behaviour](#8-required-scenario-behaviour)
9. [Known limitations](#9-known-limitations)
10. [Future improvements](#10-future-improvements)

---

## 1. Installation

1. Open TradingView and load any chart (e.g. **NQ1!** or **ES1!**).
2. Open the **Pine Editor** (bottom panel).
3. Open [`PACPE.pine`](./PACPE.pine), copy its entire contents, and paste them
   into the Pine Editor (replace any template code).
4. Click **Save**, then **Add to chart**.
5. The indicator plots as an **overlay**. A dashboard appears in the upper-right
   corner by default.
6. To view the three core series (bull score, bear score, momentum) in their own
   pane: open the indicator's *Data Window* values, or add `plot`s to a pane via
   right-click → *Move to → New pane above/below*. The score/momentum/pressure
   plots ship with `display = display.data_window` to keep the price chart clean.
7. Create alerts via the chart's **Alert** dialog → *Condition: PACPE* → choose
   one of the provided alert conditions.

Pine v6 compiles the script with no errors. No external data, APIs, Python, or ML
libraries are used.

---

## 2. Scoring architecture

The engine is built from eleven modular components. Each produces **normalized**
features (ratios in `0..1` or ATR-normalized, squashed values in `-1..+1`) that
are combined with configurable weights.

| # | Component | File section | Output |
|---|-----------|--------------|--------|
| 1 | **Candle Anatomy Analyzer** | §2 | ratios, character label, `candleQuality ∈ [-1,1]` |
| 2 | **Directional Efficiency Analyzer** | §3 | `candleEff`, `netEffFast/Mid ∈ [-1,1]` |
| 3 | **Momentum Engine** | §5 | `momentumScore ∈ [-100,100]`, `momAccel` |
| 4 | **Countertrend Penalty Engine** | §8 | `ctPenalty ≥ 0`, `ctPressure ∈ [0,100]` |
| 5 | **Structure & Pullback Analyzer** | §6 | `structScore ∈ [-1,1]`, `pullbackDepth`, breaks |
| 6 | **Market Regime Detector** | §7 | `regime` label, `chopScore`, `volFactor` |
| 7 | **Momentum Persistence Engine** | §9 | `evidenceState ∈ [-1,1]` (bounded memory) |
| 8 | **Continuation Scoring Engine** | §10 | `bullScore`, `bearScore ∈ [0,100]` |
| 9 | **Historical Outcome Calibration** | §12 | empirical rate per bucket (optional) |
| 10 | **Dashboard & Visualization** | §13–15 | table, markers, shading, panel plots |
| 11 | **Alert Engine** | §16 | 9 edge-triggered alert conditions |

### Data-flow summary

```
OHLC ─▶ Candle Anatomy ─▶ quality / efficiency / wick-rejection
     ├▶ Momentum Engine ─▶ momentumScore, momAccel
     ├▶ Structure ───────▶ HH/HL/LH/LL, structScore, breaks, pullbackDepth
     └▶ Regime ──────────▶ chopScore, volFactor
                               │
              ┌────────────────┴───────────────┐
              ▼                                 ▼
   Countertrend Penalty  ──▶  Persistence (evidenceState)  ──▶  Continuation Scores
              │                                                        │
              └──────────────────────────────────────────────────────┘
                               ▼
         Directional State + Confidence Quality + (optional) Empirical rates
                               ▼
                 Dashboard · Markers · Alerts
```

All features are **ATR-normalized** where appropriate so the engine adapts to
different instruments and volatility regimes. Near-duplicate features are kept at
low weight to avoid double-counting.

---

## 3. Formulas

Notation: `body = |close − open|`, `rng = high − low`, `closeLoc = (close − low)/rng`,
`dirSign = sign(close − open)`, `ATR = ta.atr(len)` (guarded non-zero),
`squash(x) = x / (1 + |x|)` (soft limiter into `(-1,1)`), `clamp` bounds a value.

### 3.1 Candle quality (`-1..+1`)

```
bodyComp  = squash(body/ATR) · dirSign · wBody
closeComp = (closeLoc − 0.5) · 2 · wClose
wickComp  = (lowerWick/rng − upperWick/rng) · wWick
candleQuality = clamp( squash( (bodyComp + closeComp + wickComp) · 0.6 ), −1, 1 )
```

### 3.2 Directional efficiency (`-1..+1`)

```
oppRatio = opposingWick / rng                      // wick against candle direction
extremeCloseness = dirSign>0 ? closeLoc : 1−closeLoc
candleEff = clamp( (body/rng)·(1−oppRatio)·extremeCloseness, 0, 1 ) · dirSign · wEff
netEff(len) = (close − close[len]) / Σ|close[k]−close[k+1]|   // signed path efficiency
```

### 3.3 Momentum score (`-100..+100`)

```
dispN = squash( (close − close[N]) / ATR · kN · momSens )   // N = fast/mid/slow
consecComp  = squash(consecRun / 4) · consecW
bodyConsist = squash( mean_fast( dirSign · body/ATR ) · 1.5 )
rangeTrend  = squash( (SMA(rng,fast) − SMA(rng,slow)) / ATR ) · sign(dispFast)

momRaw = ( dispFast·1.2 + dispMid·0.9 + dispSlow·0.6
         + consecComp·0.8 + bodyConsist·0.9 + netEffFast·effW + rangeTrend·0.4 ) / 5.8
momentumScore = clamp( squash(momRaw) · 100, −100, 100 )
momAccel = EMA(momentumScore, fast) − EMA(momentumScore, mid)   // >0 strengthening
```

### 3.4 Persistence / evidence memory (`-1..+1`)

```
proEvidence = clamp( (candleQuality + 0.8·candleEff + 0.6·wickRej
                      + 0.5·wStruct·structScore) / 2.9, −1, 1 )

evidenceState ← (1−alpha)·evidenceState + alpha·proEvidence      // EMA accumulation
evidenceState ← evidenceState + 0.5·ctSigned                    // countertrend bite
evidenceState ← evidenceState − evidenceState·clamp(decay·(0.5+chopScore), 0, 0.9)
evidenceState ← clamp(evidenceState, −1, 1)
```

where `ctSigned = −ctPenalty` when momentum is bullish, `+ctPenalty` when bearish.

### 3.5 Continuation scores (`0..100`, do NOT sum to 100)

Per side `s ∈ {bull, bear}` (bear uses the negated/mirrored feature halves):

```
raw_s = ( 0.30·mom_s + 0.22·ev_s + 0.14·cq_s + 0.12·eff_s
        + 0.12·str_s + 0.06·wr_s + holdBonus_s ) · volFactor − ctSub_s
raw_s ← raw_s · (1 − 0.5·chopScore)               // chop pulls both toward neutral
score_s = EMA( clamp(raw_s·100, 0, 100), round(1/scoreSmooth) )
```

- `mom/ev/cq/eff/str/wr` = the directional half (`clamp(x,0,1)` on the signed
  feature) of momentum, evidence, candle quality, efficiency, structure, wick.
- `holdBonus` rewards a *shallow pullback that holds structure*.
- `ctSub` subtracts countertrend penalty from the side currently under pressure.
- `volFactor` shrinks confidence when big ranges make little net progress.

**Documented weight contribution** (base, before regime/chop/countertrend
modifiers): momentum 30%, persistent evidence 22%, candle quality 14%, directional
+ net efficiency 12%, structure 12%, wick rejection 6%. These sum to 96%; the
remaining headroom is the structure-hold bonus. Countertrend pressure is applied
**subtractively** so it can only reduce the dominant side.

### 3.6 Directional state

`dominant = bullScore − bearScore`, then thresholds (`thStrong/thCont/thWeak`)
and `momAccel`/`ctPressure` classify into: *Strong Bullish Continuation →
Bullish Continuation → Weakening Bullish Momentum → Neutral/Choppy → Weakening
Bearish Momentum → Bearish Continuation → Strong Bearish Continuation.*

---

## 4. How countertrend candles affect momentum

This is the **core** of the engine (Pine section §8). A candle opposing the
established momentum produces a **penalty** whose size depends on nine context
factors:

1. **Strength of established momentum** — dampens the penalty of a *single minor*
   candle via `(1 − ctDampen·momStrength·…)`, but strong severity overrides it.
2. **Body size vs. ATR** — larger opposing bodies hurt more.
3. **Close location** — closing *against* the trend (bearish candle near its low
   in an uptrend) is worse than closing near its high.
4. **Wick geometry / rejection** — an adverse wick adds to severity.
5. **Persistence** — each additional consecutive countertrend candle adds a
   bounded increment (`min(ctRun,6)·0.12·ctPersistW`).
6. **Engulf / overwhelm** — opposing body larger than the prior directional body
   *and* closing beyond it adds severity.
7. **Structure break** — breaking a meaningful higher-low (uptrend) / lower-high
   (downtrend) adds `structBreakW`.
8. **Pullback depth** — deeper / structurally-damaging pullbacks amplify.
9. **Impulse state** — a penalty bites more when the impulse is already weakening
   (`momAccel` against the trend).

These combine into `severity`, then a **nonlinear** scaling makes ordinary
candles small and extreme candles disproportionate:

```
severity   = base + persist + structBreak + 0.4·pullbackDepth + weakenAmp
severityNL = severity ^ ctExp                       // ctExp ≈ 1.8 (nonlinear)
dampen     = 1 − ctDampen · momStrength · clamp(1 − 0.5·severity, 0, 1)
ctPenalty  = clamp( severityNL · dampen · ctSens · 0.25, 0, 3 )
```

The penalty then (a) reduces `evidenceState` on the momentum side, and
(b) subtracts from the dominant continuation score. Because the dampening term
*itself shrinks as severity grows*, strong momentum protects against noise **but
never makes the indicator immune to a genuine reversal**.

---

## 5. Scores vs. probabilities

- **Continuation Score (heuristic):** `bullScore` / `bearScore` (0–100). A
  transparent weighted blend of normalized features. A score of 80 means
  "strong supporting evidence", **not** "80% chance".
- **Estimated / empirical rate (optional, calibration module):** For each
  confirmed signal at/above the recording threshold, the engine snapshots the
  score and direction, then evaluates — **only after the horizon elapses** — a
  precisely defined event:

  > *Does price move at least `targetATR` ATR in the signal direction before
  > moving `adverseATR` ATR against it, within `horizon` completed candles?*

  Outcomes are tallied into score buckets (50–59, 60–69, 70–79, 80–89, 90–100)
  and the **empirical continuation rate** is shown **only when the sample size
  meets `minSample`**. These rates are labelled *empirical* in the dashboard and
  are never substituted for the heuristic score.

The code **never** invents accuracy figures or claims profitability. If you
disable calibration, the dashboard explicitly states the scores are heuristic.

---

## 6. Recommended defaults for intraday futures

Tuned for **NQ/ES on 1–5 minute charts**. Start here, then adjust to taste.

| Input | Default | Intraday-futures note |
|-------|---------|-----------------------|
| Fast / Intermediate / Slow lookback | 5 / 13 / 34 | Fibonacci-spaced; responsive but stable. |
| ATR length | 14 | Standard; normalizes NQ vs ES automatically. |
| Average body length | 20 | ~20 bars of context. |
| Momentum sensitivity | 1.0 | Raise to 1.2–1.4 for fast-moving NQ. |
| Countertrend sensitivity | 1.0 | 1.1–1.3 if you want quicker de-risking. |
| Countertrend nonlinearity | 1.8 | Keeps noise small, punishes violent candles. |
| Momentum dampening of penalty | 0.6 | Protects strong trends from single candles. |
| Evidence persistence (alpha) | 0.30 | Lower (0.2) = more memory on 1-min noise. |
| Chop decay per bar | 0.08 | Raise to 0.12 on choppy midday sessions. |
| Pivot sensitivity | 3 | 2 for 1-min scalping, 4–5 for 5-min swings. |
| Structure lookback | 20 | ~1 bar-session context on 5-min. |
| Compression / Expansion ATR | 0.65 / 1.5 | Matches typical NQ/ES range distribution. |
| Chop sensitivity | 1.0 | Raise to flag midday balance faster. |
| Score smoothing (alpha) | 0.35 | Lower for steadier dashboard reads. |
| Strong / Cont / Weak thresholds | 75 / 60 / 45 | Conservative; raise `thStrong` to 80 for selectivity. |
| Calibration horizon | 10 bars | ~10–50 min depending on timeframe. |
| Target / Adverse (ATR) | 1.0 / 1.0 | Symmetric 1R event; set 1.5/1.0 for trend days. |
| Minimum sample size | 20 | Suppresses noisy early buckets. |

**RTH tip:** On ES/NQ, consider higher `chopSens` during the lunch session and a
higher `thStrong` into the close to avoid late-session whipsaw.

---

## 7. Testing checklist (NQ / ES)

Run across **trending, reversing, and choppy** sessions on NQ1! and ES1!
(1-min and 5-min):

- [ ] **Trending up (open drive):** state reaches *Strong/Bullish Continuation*,
      momentum > +40, structure *Bullish (HH/HL)*, countertrend pressure low.
- [ ] **Trending down:** mirror of the above on the bearish side.
- [ ] **Single opposing candle in a strong trend:** score dips **modestly**,
      state does **not** flip (Scenario A / D).
- [ ] **Progressive deterioration:** countertrend pressure rises over several
      bars, momentum weakens, state steps down to *Weakening* then *Neutral*
      only when a structure break confirms (Scenario B).
- [ ] **Shallow pullback that holds + fresh impulse:** modest dip, then renewed
      strengthening; *Pullback in Trend* regime appears (Scenario C).
- [ ] **Midday chop / balance:** regime shows *Chop/Sideways*, both scores decay,
      no repeated strong bull **and** bear signals (Scenario E).
- [ ] **Violent reversal bar (news):** large opposing displacement + structure
      break → sharp confidence drop, momentum transitions, *R* reversal marker,
      persistence does **not** block the flip (Scenario F).
- [ ] **Compression before expansion:** regime *Low-Vol Compression*, then
      *Directional Expansion* on breakout.
- [ ] **Calibration sanity:** empirical rates generally increase with bucket; low
      buckets suppressed until `minSample` reached; rates never shown *as* scores.
- [ ] **No repaint:** reload chart / replay — confirmed-bar markers and buckets do
      not shift historically.
- [ ] **Alerts:** each of the 9 conditions fires once on the triggering confirmed
      bar, not every bar while true.

---

## 8. Required-scenario behaviour

| Scenario | Expected | Mechanism |
|----------|----------|-----------|
| **A** Strong bull + 1 bearish candle | Bull stays elevated, modest dip, no flip | momentum dampening of a single small penalty |
| **B** Bull then progressive decay | Confidence falls stepwise; flip only on break | persistence + rising ctRun + structure-break amp |
| **C** Rally, shallow pullback, new impulse | Small dip, structure intact, re-strengthens | hold-bonus + renewed displacement |
| **D** Bear trend + 1 bullish rejection candle | Bear stays strong; small reduction only | wick rejection weighted by context, no auto-flip |
| **E** Sideways chop | Both scores decay; regime = Chop | chopScore decay + score shrink |
| **F** Violent countertrend + break + follow-through | Sharp drop, momentum transitions | nonlinear severity overrides dampening |

---

## 9. Known limitations

- **OHLC cannot reveal order-flow.** Wick/"absorption-like" logic is purely
  *geometric* rejection evidence. Ordinary candles cannot confirm resting-order
  absorption, bid/ask imbalance, or order-book liquidity. Treat it as such.
- **Heuristic, not validated probability.** Scores are an opinion from weighted
  features, not a statistically validated forecast.
- **Calibration caveats:** (1) overlapping prediction windows are **not
  independent** observations, so empirical rates are *indicative*, not
  inferential; (2) sample sizes on a single chart's history are limited; small
  buckets are suppressed; (3) calibration reflects the *loaded history only* and
  does not generalize across regimes automatically.
- **Pivot lag:** swing highs/lows confirm `pivotLen` bars late (by design, to
  avoid repainting), so structure reads trail price slightly.
- **Parameter sensitivity:** aggressive sensitivity settings can over-react;
  defaults favour stability.
- **Lower-timeframe noise:** sub-minute data is noisier; prefer lower `alpha`
  (more memory) and higher `chopSens`.

---

## 10. Future improvements

Implementable without compromising the engine's integrity:

- **Higher-timeframe context filter** (`request.security`, non-repainting) to bias
  continuation toward the dominant HTF direction.
- **Volume / up-down-volume confirmation** where instrument volume is reliable.
- **Rolling, regime-segmented calibration** (separate buckets for trend vs. chop).
- **Non-overlapping sampling option** for more statistically honest rates.
- **Session-aware thresholds** (open drive vs. lunch vs. close).
- **Walk-forward export** of bucket stats for offline statistical review.
- **Multi-instrument normalization presets** (NQ/ES/CL/GC) as input templates.

---

*Priority: a reliable, context-sensitive price-action engine first — never
sacrificing logical consistency or statistical honesty for an impressive-looking
percentage.*

# Pricing and Quoting Correlated Combo Contracts

**A case study in why naive independence pricing fails on correlated multi-leg contracts, and what to do instead.**

This project was built to demonstrate the core quantitative skill set behind pricing and market-making correlated combo contracts — the kind found in prediction-market parlays, same-game sports parlays, and multi-asset derivative structures. Rather than sourcing data from a prediction market directly, it uses BTC and ETH price data from Binance's public API as a liquid, easily accessible proxy for two correlated event legs. The methodology — correlation modeling via copulas, out-of-sample validation, probability-to-price conversion, and inventory-aware quoting — generalizes directly to any correlated multi-leg contract, prediction markets included.

---

## 1. The Problem

Combo contracts (parlays, multi-leg spreads, "X and Y" event bets) are typically priced by multiplying the probabilities of each individual leg — an approach that is only correct if the legs are statistically independent. In practice, event legs are rarely independent: a blowout sports game correlates with the total score; a market-wide risk-off move correlates across crypto assets; an economic surprise correlates across related prediction-market contracts.

**This project asks and answers three questions:**

1. How badly does the naive independence assumption misprice a real correlated combo?
2. Does a copula-based model correct this, and does the correction generalize out-of-sample?
3. Does the corrected pricing translate into a measurable trading/quoting advantage?

---

## 2. Data

- **Source:** Binance public market data API (`data-api.binance.vision`), no authentication required
- **Assets:** BTC/USDT and ETH/USDT
- **Granularity:** Hourly and daily klines, up to ~10,000 hourly candles (~14 months) for the largest experiment
- **Combo definition:** *"BTC hourly return > 1% AND ETH hourly return > 1% simultaneously"* — a rare, tail-oriented event chosen because tail co-movement is exactly where naive pricing breaks down hardest, and exactly the regime combo/parlay pricing needs to get right

---

## 3. Finding 1 — Correlation Is Strong and Persistent

Hourly BTC/ETH returns show a correlation of **0.84**, confirmed visually via a scatter plot showing a tight diagonal cloud rather than a random spread. This correlation held at both hourly (0.84) and daily (0.82) granularity, indicating it's a structural property of the relationship rather than an artifact of one time horizon.

---

## 4. Finding 2 — Naive Pricing Fails, and Fails Worse in the Tails

| Threshold | Naive independence | Empirical (actual) | Error |
|---|---|---|---|
| Any positive move | 25.5% | 41.9% | ~1.6x underestimate |
| >1% move (tail) | 0.03% | 0.90% | **~32x underestimate** |

The mispricing gets dramatically worse as the threshold gets more extreme — the signature of **tail dependence**: correlated assets don't just move together on average, they move together *especially* during large moves. This is precisely the regime where combo/parlay pricing carries the most risk, and where naive pricing is most dangerous.

---

## 5. Method — Copula-Based Correlation Modeling

Two copula families were fit to model the joint distribution of BTC/ETH returns:

- **Gaussian copula** — standard, but underestimates tail dependence
- **t-copula** (df=4) — explicitly models tail dependence, expected to outperform Gaussian at extreme thresholds

**Results at the 1% tail threshold:**

| Model | Estimate | Error vs. empirical (0.90%) |
|---|---|---|
| Naive independence | 0.03% | -97% |
| Gaussian copula | 1.03% | +14% |
| **t-copula** | **0.96%** | **+6.5%** |

The t-copula outperformed the Gaussian copula at the tail threshold, consistent with theory: modeling tail dependence explicitly pays off exactly where naive assumptions fail hardest.

---

## 6. Validation — Out-of-Sample Testing

To rule out overfitting, both copula models were fit **only** on the first 70% of the data (train) and evaluated against the **unseen** final 30% (test):

| Model | Train-fit estimate | Test-set ground truth |
|---|---|---|
| Naive independence | 0.031% | — |
| Gaussian copula | 0.890% | — |
| **t-copula** | **0.920%** | — |
| **Empirical (test, unseen)** | — | **1.000%** |

Both copula models, trained without ever seeing the test period, correctly anticipated the tail co-occurrence rate in unseen data — confirming the correlation structure is a real, generalizable property of BTC/ETH, not a curve-fitting artifact.

---

## 7. From Probability to Price

Converting probabilities into binary-contract prices (cents, where a $1 payout contract is fairly priced at its probability) makes the mispricing concrete in dollar terms:

- A naive market maker selling 1,000 contracts at the naive price (0.03¢) collects **$0.31** in premium
- The expected payout, using the true probability, is **$10.00**
- **Expected P&L: -$9.69** — the naive market maker loses ~97% of collected premium in expectation, on this single combo

This is the sentence that makes the whole project concrete: **ignoring correlation isn't a minor modeling error — it's a business-ending pricing mistake at scale.**

---

## 8. Quoting Engine — Inventory-Aware Market Making

A quoting engine was built that:
- Computes a blended fair value from Gaussian + t-copula estimates, refit on a rolling window
- Sets bid/ask spread width based on **model disagreement** (Gaussian vs. t-copula, used as an uncertainty proxy) and **inventory imbalance** (spread widens and skews as position grows, discouraging further one-sided accumulation — a simplified Avellaneda-Stoikov-style approach)
- Simulates synthetic order flow trading against the quotes and tracks running inventory and P&L

### Experimental design

Naive and copula-based quoting were tested head-to-head across **40 independent, non-overlapping historical windows**, using **identical synthetic order flow** (same random seed) in both cases per window — isolating the pricing/quoting *logic* as the only variable, rather than confounding results with random order-flow luck.

### Results

An initial unpaired comparison of the two P&L distributions looked statistically inconclusive (signal/noise ratio 0.19x) — but this was the wrong test: trial-to-trial variance is dominated by whether the rare tail event fired in that specific window, a factor that affects both strategies identically and therefore isn't part of the actual question being asked.

**A paired comparison (copula P&L − naive P&L per trial) revealed the true effect:**

| Metric | Result |
|---|---|
| Copula won | **40 / 40 trials** |
| Mean improvement | **+$0.696 per trial** |
| Std. dev. of improvement | $0.479 |
| Signal/noise ratio | **9.19x** |
| Paired t-test p-value | **2.6 × 10⁻¹¹** |
| Sign test p-value | **9.1 × 10⁻¹³** |
| Minimum improvement (worst case) | +$0.096 (still positive) |

Copula-based quoting outperformed naive quoting in every single trial, with a small but statistically overwhelming and perfectly consistent edge. Splitting by whether the tail event fired shows the advantage roughly doubles when it does (+$0.723 vs. +$0.362), consistent with the theoretical prediction that correlation modeling matters most exactly when correlated extreme moves occur.

---

## 9. Honest Limitations

- **Effect size is small relative to overall P&L variance.** The ~$0.70 average edge is real and highly significant, but it's a modest improvement (~12%) relative to average trial P&L (~$6), not a transformative one. This is consistent with how real market-making edges typically look — thin, but reliable and compounding.
- **200-hour trial windows were long enough that the tail event fired in 37/40 windows**, limiting the "no event" comparison group (n=3) and understating what a cleaner, shorter-window experiment might show.
- **Order flow is synthetic**, not sourced from a real order book or real RFQ stream. A live implementation against actual Binance order book data (or real prediction-market RFQ data) would be a natural next step.
- **The Gaussian copula's rolling refit uses a relatively small window (300 observations)**; a production system would likely use adaptive window sizing and more robust correlation estimation under regime changes.

---

## 10. Summary

| Phase | Finding |
|---|---|
| Correlation | BTC/ETH hourly returns correlate at 0.84 |
| Naive pricing | Underestimates tail co-occurrence by ~32x |
| Copula modeling | t-copula recovers true probability within ~6.5% at the tail |
| Validation | Confirmed out-of-sample — not overfitting |
| Dollar impact | Naive pricing implies a ~97% expected loss on premium collected |
| Quoting | Copula-based quotes beat naive quotes in 40/40 paired trials (p < 10⁻¹¹) |

**Bottom line:** ignoring correlation between combo legs is not a minor modeling simplification — it is a systematic, quantifiable, and costly error, and it's correctable with standard quantitative tools (copulas, out-of-sample validation, and inventory-aware quoting logic).

---

## Tech Stack

Python, pandas, numpy, scipy, the `copulas` package, Binance public market data API. Full code available in `quoting_engine.py`.

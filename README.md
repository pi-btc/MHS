# [Research]Correlated Deterioration: A One-Line Fix to LlamaRisk's Market Health Score

May 15, 2026

LlamaRisk's Market Health Score adds up seven risk dimensions as if they're independent. They're not—during stress, they all move together. We add a single correction factor derived from Random Matrix Theory that automatically penalises the score when dimensions co-deteriorate. It changes nothing during normal conditions and imposed a 14.3% penalty on the day of maximum 2024 stress and 10.9% on the eve of the October 2025 flash crash.

---

## The Problem in One Sentence

LlamaRisk's own methodology says risk dimensions *"interact and amplify one another"*—but the scoring formula treats them as independent.

Here's the formula they use:

```
Health Score = 0.30 × debt_ceiling
             + 0.15 × volatility
             + 0.15 × momentum
             + 0.10 × soft_liq_collateral
             + 0.10 × soft_liq_efficiency
             + 0.10 × bad_debt
             + 0.05 × collateral_ratio
             + 0.05 × borrower_concentration
```

This is a weighted average. Weighted averages assume the inputs carry independent information. When ETH price drops 20% in a day, volatility spikes, momentum turns negative, soft liquidation collateral rises, and collateral ratio falls—all at the same time, all for the same reason. The formula counts this as four separate bad signals. It's really one.

---

## The Fix

We don't change any dimension or any weight. We just ask one question after computing the score:

> **"Are these dimensions moving independently, or are they all pointing the same direction?"**

If independent → leave the score alone.  
If co-moving → scale the score down.

The measure we use is **Effective Rank**—a single number between 1 and 8 that summarises how many truly independent dimensions the scoring system currently contains. We compute it using Random Matrix Theory (Marchenko-Pastur cleaning) on a 60-day rolling window of dimension scores.

**Normal conditions:** Effective Rank ≈ 3.8 → correction factor ≈ 1.0 → score unchanged.  
**Stress conditions:** Effective Rank drops → correction factor < 1 → score penalised.

The adjusted score is simply:

```
Health_adjusted = Health_original × min(1, eff_rank(t) / eff_rank_baseline)
```

That's it.

---

## What the Data Shows

We ran this on all five crvUSD markets with complete history (WETH, WBTC, wstETH, tBTC, sfrxETH) from October 2023 to May 2026.

*<img width="2084" height="2331" alt="rmt_health_correction" src="https://github.com/user-attachments/assets/99803880-52e7-4a02-93a4-847a6b14fdcf" />
*



**The 2024 baseline (full year):**  
Correction factor mean = 0.997. The method is essentially inactive. No permanent downward bias.

**July 5, 2024 — maximum penalty in the full sample:**

| Metric | Value |
|--------|-------|
| Original score | 0.613 |
| Adjusted score | 0.525 |
| Effective Rank | 3.27 (vs baseline 3.81) |
| Penalty | **14.3%** |

The original score said "borderline." The adjusted score said "elevated risk." Collateral ratio, soft-liq collateral, and price momentum were all deteriorating simultaneously—three dimensions, one cause (falling ETH/BTC prices).

**October 9–10, 2025 — flash crash:**

| Date | Original | Adjusted | Eff. Rank | Penalty |
|------|----------|----------|-----------|---------|
| Oct 7 | 0.751 | 0.751 | 4.31 | 0% |
| Oct 9 | 0.701 | 0.618 | 3.36 | **11.9%** |
| Oct 10 (crash) | 0.685 | 0.611 | 3.40 | **10.9%** |
| Oct 17 | 0.749 | 0.749 | 4.00 | 0% |

One day before the crash, effective rank dropped from 4.3 to 3.4. The original score showed 0.70 ("healthy"). The adjusted score showed 0.62. The correction fired before the event and turned off cleanly after recovery—it's not sticky.

---

## Why This Matters for Risk Monitoring

The weighted average has a specific failure mode: it underestimates risk precisely when risk is most dangerous, because that's when all dimensions co-deteriorate. A score of 0.60 achieved through eight independent moderate signals is genuinely less dangerous than a score of 0.60 achieved through eight highly correlated signals that all trace back to a single collateral price move.

The effective rank correction distinguishes these two cases. The original score cannot.

---

## What We're Proposing

**No changes to LlamaRisk's existing dimensions or weights.**

One addition to the aggregation step:
1. Maintain a 60-day rolling buffer of dimension score vectors
2. Compute effective rank via Marchenko-Pastur-cleaned correlation matrix
3. Multiply the composite score by `min(1, eff_rank(t) / baseline)`
4. Surface both scores in the portal, plus effective rank as an auxiliary indicator

Computational cost: negligible (8×8 matrix operations per daily update). No new data sources required.

---

## Full Paper

The complete methodology, derivations, and robustness analysis are in the accompanying working paper:

> *Correlated Deterioration: Adding an Effective Rank Correction to LlamaRisk's crvUSD Market Health Scores* (May 2026)

All code is available at [repository link:https://github.com/pi-btc/MHS]

*Feedback welcome. Particularly interested in: (1) whether the 60-day window should be TVL-weighted across markets, (2) whether the baseline period should be updated dynamically, (3) whether this approach generalises to Curve lending markets beyond crvUSD.*

---

## References

LlamaRisk (2025). *Curve Market Health Scores Methodology*  https://llamarisk.com/research/curve-market-health-methodology

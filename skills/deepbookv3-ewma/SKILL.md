---
name: deepbookv3-ewma
description: Understand and calculate Exponentially Weighted Moving Average (EWMA) Gas Price Penalties for takers in DeepBook V3.
---

# DeepBook V3: EWMA Gas Price Penalty

DeepBook V3 implements an Exponentially Weighted Moving Average (EWMA) gas price tracking system to protect liquidity makers from toxic taker transactions during gas price spikes (e.g., front-running or arbitrage during network congestion).

---

## 1. Key Concepts

### Purpose of EWMA
- During periods of high network congestion, sophisticated traders might pay abnormally high gas fees to front-run or take stale maker limit orders before makers can update or cancel them.
- To disincentivize this, DeepBook V3 tracks historical network gas prices and applies an **additional taker fee penalty** when the current transaction's gas price is exceptionally high.

### Trigger Conditions
The gas penalty is only applied when **all** of the following conditions are met:
1. EWMA tracking is enabled for the specific pool.
2. The current transaction's gas price is above the historical average: $\text{Current Gas Price} \ge \text{Mean}$.
3. The volatility-adjusted deviation is above the threshold: $\text{Z-Score} > \text{Z-Score Threshold}$.

---

## 2. Mathematical Formulas

### Z-Score Calculation
The z-score measures how many standard deviations the current gas price is from the historical mean:
$$\text{Z-Score} = \cfrac{\text{Current Gas Price} - \text{Mean}}{\text{Standard Deviation}}$$

$$\text{Standard Deviation} = \sqrt{\text{Variance}}$$

### Penalty Threshold Formula
The exact gas price at which the penalty begins to apply is:
$$\text{Penalty Threshold} = \text{Mean} + (\text{Z-Score Threshold} \times \text{Standard Deviation})$$

### Configuration Parameters (Example)

| Parameter | Type / Value | Meaning |
|---|---|---|
| **Alpha** | `u64` / `0.1` ($100,000,000$ scaled) | Weight given to the new gas price data point (10% new data, 90% history). |
| **Z-Score Threshold** | `u64` / `3.0` | Trigger point. Statistically, only $\sim 0.15\%$ of transactions exceed 3 standard deviations above the mean in a normal distribution. |
| **Additional Taker Fee** | `u64` / `0.1%` ($1,000,000$ scaled) | The penalty fee added to the standard taker fee if the z-score threshold is breached. |

#### Example Walkthrough:
If the pool's current tracked parameters are:
- $\text{Mean} = 1,478$
- $\text{Standard Deviation} = 6,578$
- $\text{Z-Score Threshold} = 3.0$

$$\text{Penalty Threshold} = 1,478 + (3.0 \times 6,578) \approx 21,212 \text{ gas units}$$

- **Scenario A (Normal)**: Gas Price $= 1,000 \implies$ No penalty.
- **Scenario B (Elevated)**: Gas Price $= 15,000 \implies \text{Z-Score} = 2.06 \implies$ No penalty (below 3.0).
- **Scenario C (Spike)**: Gas Price $= 25,000 \implies \text{Z-Score} = 3.58 \implies$ **Penalty Triggered** (additional 0.1% fee is applied to the taker order).

---

## 3. Technical Implementation Details

- **Update Frequency**: The pool's EWMA parameters are updated whenever an order is submitted to the book.
- **Gas Optimization**: Updates are tracked once per transaction. If a Programmable Transaction Block submits multiple orders, only the first order updates the EWMA historical state (checking the timestamp), preventing redundant math and reducing gas fees.
- **Independence**: EWMA states are tracked independently for each pool.

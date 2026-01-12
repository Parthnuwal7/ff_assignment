# Stock Price Prediction Model – Futures First Assignment

## Overview

This project investigates the relationship between a provided `Data` signal and next-day stock price behavior. Rather than blindly fitting a model, I followed a **hypothesis-driven approach**: systematically testing multiple plausible hypotheses, rejecting or accepting them based on empirical evidence, and finally building a model that reflects the *true learnable relationship* in the data.

**Note:** The `Data` signal does **not** predict price direction or level. It predicts **volatility magnitude**.

While this model cannot tell you which direction the price will move, it tells you how much it might move. This has real applications:

Position Sizing: On high-volatility days (extreme ΔData), reduce position size to manage risk
Options Pricing: Higher expected magnitude → higher option premium
Stop-Loss Placement: Wider stops on volatile days, tighter on calm days

---

## Key Findings (TL;DR)

| What I Tested | Result | Verdict |
|---------------|--------|---------|
| Level correlation (Data vs Price) | r = 0.38 | ❌ Spurious (non-stationary) |
| Same-day delta correlation | r ≈ 0 | ❌ No signal |
| Lagged delta correlation (ΔData → ΔPrice[t+1]) | r ≈ 0 | ❌ No linear effect |
| Directional prediction (sign matching) | 47% accuracy | ❌ Worse than random |
| Conditional direction (extreme ΔData days) | ~50/50 up/down | ❌ No asymmetry |
| **Magnitude prediction (\|ΔData\| → \|ΔPrice[t+1]\|)** | **2× volatility on extreme days** | ✅ **Accepted** |

**Final Model Output:** Given today's price `P`, the model predicts tomorrow's price will lie within approximately **P ± 25 points** (based on MAE), with larger ranges during periods of extreme Data changes.

---

## Repository Structure

```
ff_assignment/
├── FF_assignment.ipynb    # Main analysis notebook (all code + explanations)
├── Data.csv               # Independent variable dataset
├── StockPrice.csv         # Dependent variable (stock prices)
└── README.md              # This file
```

---

## Methodology

### 1. Data Preprocessing

- **Merged datasets** on `Date` column using left join
- **Identified US market holidays** (MLK Day, 4th July, Thanksgiving, etc.) by analyzing null price entries
- **Dropped non-trading days** since dependent variable (Price) is absent
- **Created derived features:**
  - `delta_data` = day-over-day change in Data
  - `delta_price` = day-over-day change in Price
  - `delta_price_t_plus_1` = next day's price change (target for prediction)

### 2. Hypothesis Testing Framework

I approached this problem scientifically — testing hypotheses one by one, using appropriate metrics for each, and only accepting what the evidence supports.

---

## Hypothesis Results (Detailed)

### ❌ Hypothesis 1: Levels are correlated
**Question:** Are `Data` and `Price` directly correlated?

**Test:** Pearson correlation on levels

**Result:** r = 0.38 (moderate positive)

**Why Rejected:** Both series are non-stationary (trending). Correlation between non-stationary series is misleading — it captures shared trends, not causal relationships. This is a classic case of spurious correlation.

---

### ❌ Hypothesis 2: Same-day changes are correlated
**Question:** Does ΔData[t] influence ΔPrice[t]?

**Test:** Correlation between same-day deltas

**Result:** r ≈ 0

**Why Rejected:** No contemporaneous relationship exists. The Data signal doesn't move prices on the same day.

---

### ❌ Hypothesis 3: Lagged changes predict next-day price
**Question:** Does ΔData[t] predict ΔPrice[t+1]? (This is the core assumption in the assignment)

**Test:** 
- Correlation: r ≈ 0
- Linear Regression: R² ≈ 0, MAE ≈ 25

**Why Rejected:** No linear mean effect. The magnitude of Data changes doesn't predict the magnitude OR direction of next-day price changes in a linear fashion.

**Important Nuance:** This rejects *linear mean influence*, not necessarily non-linear or conditional effects.

---

### ❌ Hypothesis 4: Directional prediction works
**Question:** Maybe the relationship is directional — sign of ΔData predicts sign of ΔPrice[t+1]?

**Test:** Sign agreement percentage

**Result:** 47% accuracy (random baseline = 50%)

**Why Rejected:** Worse than a coin flip. No directional predictability exists.

---

### ❌ Hypothesis 5: Extreme Data days cause directional bias
**Question:** On days with extreme ΔData (top/bottom 10%), is there a directional skew in next-day price?

**Test:** Conditional directional rates within buckets

**Result:**
| ΔData Bucket | % Up | % Down |
|--------------|------|--------|
| Bottom 10%   | ~50% | ~50%   |
| Middle 80%   | ~50% | ~50%   |
| Top 10%      | ~50% | ~50%   |

**Why Rejected:** No asymmetry. Even extreme Data days don't predict direction.

---

### ✅ Hypothesis 6: Magnitude relationship exists
**Question:** Does |ΔData[t]| predict |ΔPrice[t+1]|?

**Test:** Compare absolute magnitude of price moves across Data buckets

**Result:**
| ΔData Bucket | Mean |ΔPrice[t+1]| | Median |ΔPrice[t+1]| |
|--------------|----------------------|------------------------|
| Bottom 10%   | ~40                  | ~30                    |
| Middle 80%   | ~20                  | ~15                    |
| Top 10%      | ~45                  | ~35                    |

**Why Accepted:** 
- Clear separation between buckets
- ~2× increase in volatility on extreme ΔData days
- Robust across both mean and median (not driven by outliers)
- Effect is symmetric (sign-independent)

**This is the only hypothesis supported by evidence.**

---

##  Final Model

### Core Model (Magnitude Prediction)
```
Target: |ΔPrice[t+1]|
Features: |ΔData[t]|
Algorithm: Linear Regression with StandardScaler
```

**Results:**
| Metric | Model   | Naive Baseline |
|--------|---------|----------------|
| MAE    | 24.9969 | 25.7938        |
| RMSE   | 35.4561 | 37.9886        |

The model slightly beats the naive baseline, confirming the magnitude relationship is learnable.

## Supplementary Analysis

### Random Forest (Non-Linear Directional Test)
To exhaustively test if *any* non-linear directional signal exists:
- Features: ΔData, lagged ΔData, lagged ΔPrice
- Model: RandomForestClassifier
- **Result: ~50% accuracy** (equivalent to random guessing)

Conclusion: Even non-linear models cannot predict direction.

### ARIMAX (Time Series Control)
To check if Data has any effect after controlling for price autocorrelation:
- Model: ARIMAX(1,0,1) with ΔData as exogenous variable
- **Result:** ΔData coefficient is NOT statistically significant (p >> 0.05)

Conclusion: Once past price dynamics are accounted for, the Data signal has no effect on mean returns. Its influence is purely on volatility.

---

## 💡 Key Insights

1. **The Data signal is a volatility indicator, not a price predictor.** It tells you *how much* the price might move, not *which direction*.

2. **Regime changes matter.** Pre-2020 volatility was lower; post-2022 volatility spiked significantly. External factors (geopolitics, macro events) drive these shifts — outside scope of this assignment but worth noting.

3. **Direction is fundamentally unpredictable** from this dataset. Even with non-linear models and lagged features, we cannot beat 50% accuracy.

4. **The correct framing:** Given today's price P, the model estimates tomorrow's price will lie within **P ± predicted_magnitude** — a range, not a point estimate.

---

## 🏃 How to Run

1. Clone this repository
2. Ensure Python 3.8+ with the following packages:
   ```
   pandas, numpy, matplotlib, seaborn, scikit-learn, statsmodels
   ```
3. Open `FF_assignment.ipynb` in Jupyter/VS Code
4. Run all cells sequentially (data files must be in the same directory)

---

## 📝 Assumptions Made

1. Only the relationship between provided datasets is modeled (as per assignment constraints)
2. External factors (macro, news, sentiment) are explicitly ignored
3. Non-trading days (holidays) are dropped since Price data is unavailable
4. Chronological train/test split (80/20) to respect temporal ordering

---

## What I Learned

- **Hypothesis testing > blind modeling.** Starting with assumptions and systematically rejecting them led to a defensible, interpretable result.
- **Negative results are results.** Showing that direction cannot be predicted is as valuable as finding a predictive signal.
- **Volatility signals are underrated.** Even if you can't predict direction, knowing *how much* the price might move has real trading applications (options pricing, position sizing).

---

## 📧 Author

Submitted as part of Python Developer Intern (II) 2026 Machine Learning Assignment – Futures First by Parth Nuwal

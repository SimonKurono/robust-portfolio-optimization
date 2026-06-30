# Robust Portfolio Optimization — Goldfarb-Iyengar SOCP Backtest

An empirical backtest of the [Goldfarb-Iyengar (2003)](references/goldfarb-iyengar-2003-robust-portfolio.pdf)
robust minimum-variance portfolio optimizer on the full S&P 500 universe (1990–2026).

The optimizer solves a **Second-Order Cone Program (SOCP)** that finds the minimum-variance
portfolio robust to factor model uncertainty within an ellipsoidal uncertainty set of size ω.
As ω → 0 the solution converges to the classical Markowitz minimum-variance portfolio.

---

## What This Implements

| Component | Detail |
|-----------|--------|
| **Factor model** | Goldfarb-Iyengar factor structure with Fama-French 5 + Momentum (6 factors) |
| **Parameter estimation** | Rolling 60-month OLS, producing μ, G, F-cov, and residual variances |
| **Optimizer** | SOCP (GI eq. 32) via [CVXPY](https://www.cvxpy.org/) |
| **Universe** | ~503 current S&P 500 constituents, Bloomberg TRI data |
| **Backtest** | Walk-forward, monthly rebalancing, 1990-01 → 2026-04 |
| **Benchmarks** | Equal-weight, Markowitz min-var, Ledoit-Wolf shrinkage min-var |
| **Sensitivity** | Three uncertainty-set sizes: ω ∈ {0.90, 0.95, 0.99} |
| **Dual analysis** | σ\_opt time series to diagnose when uncertainty constraints bind |

---

## Repository Structure

```
.
├── robust_portfolio_backtest.ipynb   # Main notebook (Steps 1–7)
├── requirements.txt
├── data/
│   ├── README.md                     # Data sourcing instructions
│   ├── F-F_Research_Data_5_Factors_2x3_daily.csv
│   ├── F-F_Research_Data_Factors_daily.csv
│   ├── F-F_Momentum_Factor_daily.csv
│   └── processed/                    # Parquet files; reproduce via Steps 1–2
│       ├── monthly_returns.parquet
│       ├── monthly_excess_returns.parquet
│       ├── monthly_factors.parquet
│       ├── macro_daily.parquet
│       └── spx_benchmark.parquet
├── results/                          # Backtest outputs (pkl + figures)
└── references/                       # Source papers
    ├── goldfarb-iyengar-2003-robust-portfolio.pdf
    ├── blanchet-chen-zhou-wasserstein-robust-mv.pdf
    └── ledoit-wolf-nonlinear-shrinkage.pdf
```

---

## Quickstart

```bash
# 1. Clone
git clone https://github.com/SimonKurono/robust-portfolio-optimization.git
cd robust-portfolio-optimization

# 2. Install dependencies
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Open the notebook
jupyter notebook robust_portfolio_backtest.ipynb
```

The processed parquet files in `data/processed/` are included, so you can run the notebook
from **Step 2 onwards** without Bloomberg access. To reproduce from raw data, see `data/README.md`.

---

## Notebook Structure

| Step | Title | Description |
|------|-------|-------------|
| 1 | Data Loading & Cleaning | Parse Bloomberg TRI, Fama-French factors, macro data; save parquets |
| 2 | GI Parameter Estimation | Rolling 60-month factor model estimation (μ, G, F-cov, D) |
| 3 | SOCP Solver | CVXPY implementation of GI eq. 32; spectral precomputation for speed |
| 4 | Sanity Checks | Toy 20-stock universe; verify Markowitz recovery at ρ = 0 |
| 5 | Benchmark Strategies | Equal-weight, Markowitz, Ledoit-Wolf covariance shrinkage |
| 6 | Rolling Backtest Engine | Walk-forward engine; serialises results to `results/` |
| 7 | Performance & Dual Analysis | Sharpe/drawdown metrics, bootstrap significance, σ\_opt diagnostics |

---

## References

- Goldfarb, D. & Iyengar, G. (2003). *Robust Portfolio Selection Problems.* Mathematics of Operations Research.
- Blanchet, J., Chen, L. & Zhou, X.Y. (2022). *Distributionally Robust Mean-Variance Portfolio Selection with Wasserstein Distances.* Management Science.
- Ledoit, O. & Wolf, M. (2020). *Nonlinear Shrinkage Estimation of Large-Dimensional Covariance Matrices.* Annals of Statistics.

---

## License

MIT

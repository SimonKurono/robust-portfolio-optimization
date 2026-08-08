# CLAUDE.md — Robust Portfolio Optimization Project

## WHO I AM

Simon — second year UBC student, B.Com Computer Science and Business 
with a Mathematics minor (Class of 2028). Currently interning at 
Recruit Holdings / Indeed Japan in Tokyo. Writing a quantitative 
finance whitepaper on the Goldfarb-Iyengar (2003) robust portfolio 
optimization framework, targeted at quant firm recruiters.

---

## CRITICAL INSTRUCTION — HOW TO HELP ME

I want to write every single line of code myself.

Your job is to give me step-by-step guidance — explain what I need 
to do next, why I am doing it, what the output should look like, 
and what to verify. Do NOT write full code blocks for me unless I 
explicitly ask.

When guiding me:
1. Tell me what the next step is in plain English
2. Explain why we are doing it
3. Tell me what libraries/functions I will need
4. Tell me what the output should look like
5. Tell me what to verify before moving on
6. Wait for me to write the code and show you the output
7. Then tell me what to do next

If I am stuck and explicitly ask you to show me the code, 
you may show a small targeted example. But default to guidance 
not code generation.

---

## THE PAPER — SUMMARY

The paper derives the Lagrangian dual of the Goldfarb-Iyengar (2003) 
robust minimum variance SOCP and provides economic interpretations of 
all dual variables. The empirical contribution is the first large-scale 
rigorous backtest of the GI framework on real S&P 500 data with dual 
variable tracking over time.

---

## THE MATH — WHAT THE SOCP LOOKS LIKE

GI factor model:
    r = mu + V^T f + epsilon

Uncertainty sets:
    S_m: box uncertainty on mean returns — |mu_i - mu0_i| <= gamma_i
    S_v: ellipsoidal uncertainty on factor loadings — ||W_i||_G <= rho_i
    S_d: box uncertainty on idiosyncratic variance — d_i in [d_lower, d_bar_i]

Primal SOCP (GI equation 32) — minimize worst-case portfolio variance:

    minimize    lam + delta

    subject to:
    C1: phi^T D_bar phi <= delta
        SOC: ||[2*D_bar^{1/2}*phi; 1-delta]|| <= 1+delta
    C2: (mu0 - gamma)^T phi >= alpha        (worst-case return floor)
    C3: phi_i <= zeta_i  for all i          (absolute value upper bound)
    C4: -phi_i <= zeta_i for all i          (absolute value lower bound)
    C5: 1^T phi = 1                         (budget constraint)
    C6: tau + 1^T t <= lam                  (variance budget allocation)
    C7: sigma <= 1/theta_max                (S-procedure validity)
    C8: ||[2r; sigma-tau]|| <= sigma+tau    (uncertainty coupling SOC)
        where r = rho^T zeta
    C9: ||[2*w_i; 1-sigma*theta_i-t_i]|| <= 1-sigma*theta_i+t_i  for i=1..m
        where w = A @ phi
        where A = Q^T H^{1/2} G^{1/2} V0

Key matrices:
    H = G^{-1/2} F G^{-1/2}        (m x m — combines factor risk + estimation uncertainty)
    H = Q Lambda Q^T                 (spectral decomposition)
    theta_i = eigenvalues of H       (FIXED parameters — NOT decision variables)
    theta_max = max eigenvalue of H
    A = Q^T H^{1/2} G^{1/2} V0     (m x n — maps weights to rotated factor exposures)

Decision variables: phi, lam, delta, zeta, sigma, tau, t

CRITICAL NAMING CONVENTION:
    lam       = factor variance budget DECISION VARIABLE (never call this lambda_i)
    theta_i   = eigenvalues of H (FIXED parameters from data)
    Never use lambda for eigenvalues in code — always theta

Dual program:
    g = mu2*alpha - nu + mu7*(sigma - 1/theta_max)
        - (1/4)*c^T M^{-1} c
        - (rho^T M^{-1} c)^2 / (4*sigma)

    where c = mu2*(mu0-gamma) - (p-q) - nu*1
    where M = D_bar + A^T diag(1/(1-sigma*theta_i)) A

Dual variables to track empirically (novel contribution):
    mu2  = shadow price on return constraint
    mu7  = shadow price on S-procedure validity (C7)
    mu9i = 1/(1-sigma*theta_i) for each factor i
    sigma_opt = optimal S-procedure multiplier

Verify after each solve:
    mu8  ≈ 1/sigma_opt              (within 1e-4)
    mu9i ≈ 1/(1-sigma_opt*theta_i)  (within 1e-4)
    mu1  ≈ 1                         (within 1e-4)
    mu6  ≈ 1                         (within 1e-4)
    weights.sum() ≈ 1.0              (within 1e-6)
    all weights >= -1e-6             (long only)

---

## PARAMETER ESTIMATION (GI Section 5)

At each rolling window of T=60 months, for each stock i:

OLS regression: r_it = mu_i + V_i^T f_t + eps_it
Design matrix: A_reg = [1, f_1, ..., f_m]  (T x m+1)
OLS estimate: x_hat_i = [mu_i, V_1i, ..., V_mi]
Residual variance: s2_i = ||r_i - A_reg x_hat_i||^2 / (T-m-1)

Factor covariance: F = cov(factor_returns)  (m x m)

G matrix (GI eq 57):
    B = factor_returns.T  (m x T)
    G = inv(B @ B.T - (1/T) * (B @ ones) @ (ones.T @ B.T))

Critical values (omega = 0.95):
    c_full = scipy.stats.f.ppf(0.95, m+1, T-m-1)
    c_mean = scipy.stats.f.ppf(0.95, 1, T-m-1)

Uncertainty parameters:
    rho_i   = sqrt((m+1) * c_full * s2_i)
    gamma_i = sqrt(inv(A_reg^T A_reg)[0,0] * c_mean * s2_i)
    d_bar_i = s2_i
    mu0_i   = x_hat_i[0]
    V0[:,i] = x_hat_i[1:]

---

## BACKTEST DESIGN DECISIONS

### Issue 1 — Estimation approach: ROLLING WINDOW
- Window length: T = 60 months
- Updated at every rebalancing date
- Never use any data from period t or later when estimating at t
- At time t, use data from months [t-60, t) exclusively

### Issue 2 — Constituent handling: POINT-IN-TIME DATA
Data source: WRDS CRSP (crsp.msp500list, crsp.msf, crsp.msedelist)

When a stock is removed from S&P 500:
- Simplified forced liquidation (Option A):
  Include the stock's full monthly return for the month it was removed
  Set its weight to zero from the following month onward
  Reallocate capital proportionally to remaining holdings at next rebalancing
- Note in paper that intra-month precision would require daily CRSP data

Delisting return handling:
- If dlret is available: use dlret as the final return
- If stock delisted for bad reason (dlstcd 500-584) and dlret missing: use -30%
  (Shumway 1997 convention)
- Otherwise: use reported monthly return

### Issue 3 — Transaction costs: COST-AWARE OPTIMIZATION
Phase 1 (start here): no transaction costs, report gross returns
Phase 2 (when Phase 1 works): add transaction costs to SOCP objective

Phase 2 formulation — add turnover penalty to objective:
    minimize lam + delta + c * ||phi - phi_prev_drifted||_1

Where:
    c = transaction cost per unit turnover (test at 0.0005 and 0.002)
    phi_prev_drifted = weights after market drift, before rebalancing
    ||.||_1 = sum of absolute weight changes (convex, preserves SOCP structure)

The l1 norm is convex so the SOCP structure is preserved.
Implement using standard l1 to SOC conversion via auxiliary variables.

Always report both:
    Gross return (no costs)
    Net return at 5bps (c=0.0005)
    Net return at 20bps (c=0.002)

### Issue 4 — New and departing stocks: BOTH RULES

Rule 1 — Minimum history requirement:
    Only include stock i in the optimization at time t if it has
    at least 30 valid return observations in the current 60-month window
    Stocks with fewer than 30 observations are excluded from that period

Rule 2 — New entrant requirement:
    A stock entering the S&P 500 must accumulate 30 months of history
    before it can appear in the portfolio
    Until then it is in the eligible universe but excluded from optimization

Implementation: at each period, compute valid_stocks as those with
    notna().sum() >= 30 in the current estimation window
    The NaN encoding from point-in-time construction handles this automatically

---

## BENCHMARKS TO COMPARE AGAINST

1. GI Robust — omega=0.95 (main strategy)
2. GI Robust — omega=0.90 (sensitivity)
3. GI Robust — omega=0.99 (sensitivity)
4. Equal weight (1/N) — rebalanced monthly
5. Markowitz with sample covariance (OLS)
6. Markowitz with Ledoit-Wolf shrinkage (sklearn)
7. Risk Parity (1/volatility weighting)
8. S&P 500 total return index (passive benchmark)

---

## PERFORMANCE METRICS

Return metrics (annualized):
    Geometric mean return: (product of (1+r_monthly))^12 - 1
    Volatility: std(monthly returns) * sqrt(12)
    Sharpe ratio: annualized excess return / annualized volatility

Risk metrics:
    Maximum drawdown
    Calmar ratio: annualized return / abs(max drawdown)

Cost metrics:
    Average monthly turnover: mean(||phi_t - phi_{t-1}_drifted||_1) * 12
    Net return at 5bps: gross return - turnover * 0.0005 * 12
    Net return at 20bps: gross return - turnover * 0.002 * 12

Statistical significance:
    Stationary bootstrap (Politis-Romano) for Sharpe ratio standard errors
    Null hypothesis: GI Sharpe - Benchmark Sharpe = 0
    Report p-values for each comparison

---

## DUAL VARIABLE ANALYSIS (NOVEL CONTRIBUTION)

At each rebalancing date record:
    mu2: shadow price on return constraint (C2)
    mu7: shadow price on S-procedure validity (C7)
    mu9i: per-factor risk prices, vector of length m=6
    sigma_opt: optimal S-procedure multiplier

Analysis to run:
    Plot mu2 and mu7 over time vs VIX index
    Plot mu2 and mu7 over time vs MOVE index
    Identify dates where mu7 > 0 (S-procedure bound binding)
    Show which factor had highest mu9i in each crisis period
    Compute correlations: mu2 vs VIX, mu7 vs VIX, mu7 vs MOVE
    Show top-10 dates with highest mu7
    Overlay NBER recession shading on all time series plots

Hypotheses to test:
    mu2 spikes during 2008 and 2020 (return constraint costly in crises)
    mu7 > 0 during 2008 and 2020 (dominant factor direction binding)
    Market factor (i=1) mu9i is highest during stress periods

---

## DATA SOURCES AND STRUCTURE

### WRDS Tables (pull once, save as parquet)

Table 1 — Constituent history:
    crsp.msp500list
    Columns: permno, start, ending
    Filter: none
    Save: data/wrds/constituents.parquet

Table 2 — Monthly stock returns:
    crsp.msf
    Columns: permno, date, ret, retx, prc, shrout, exchcd, shrcd
    Filter: shrcd IN (10,11), exchcd IN (1,2,3), date >= 1989-01-01
    Filter: permno IN (all permnos ever in S&P 500 from Table 1)
    Save: data/wrds/monthly_returns_raw.parquet

Table 3 — Delisting returns:
    crsp.msedelist
    Columns: permno, dlstdt, dlret, dlstcd
    Filter: permno IN (same as Table 2)
    Save: data/wrds/delistings.parquet

### Processed tables (derived from raw tables)

Return panel (pivot of Table 2, point-in-time encoded):
    Rows = monthly dates
    Columns = permno
    Values = adjusted returns (NaN if stock not in S&P 500 that month)
    Save: data/wrds/return_panel.parquet

Excess return panel:
    Same as return panel minus monthly RF from Ken French
    Save: data/wrds/excess_return_panel.parquet

### Ken French Factor Data (already downloaded)

Files in data/:
    F-F_Research_Data_5_Factors_2x3_daily.csv    — FF5 daily
    F-F_Research_Data_5_Factors_2x3.csv          — FF5 monthly
    F-F_Momentum_Factor_daily.csv                — Momentum daily
    F-F_Research_Data_Factors_daily.csv          — FF3 daily (backup)

Processed and saved as:
    data/factors_monthly.parquet
    data/factors_daily.parquet

Columns: date, MktRF, SMB, HML, RMW, CMA, MOM, RF
All values in decimal (already divided by 100)

### Bloomberg Data (already downloaded, in data/)

    SPX_CUR_DATA.csv      — current S&P 500 constituent total return index
                            survivorship biased — use only for cross-checks
    SPX_RETURN_DATA.csv   — S&P 500 index total return (benchmark)
    VIX_DATA.csv          — VIX index PX_LAST daily
    MOVE_DATA.csv         — MOVE index PX_LAST daily
    USGG10Y_DATA.csv      — 10yr Treasury yield daily
    GBT3GVT_DATA.csv      — 3m T-bill yield daily
    WRDS_SPY_DATA_MASTER.csv — SPY ETF from WRDS

### FRED Data (to download if not already)

    BAMLH0A0HYM2 — US high yield spread (from 1996)
    USREC        — NBER recession indicator (binary)
    
Download from fred.stlouisfed.org, save as:
    data/macro/high_yield_spread.parquet
    data/macro/nber_recessions.parquet

---

## SOLVER

Primary: MOSEK
    Free academic license at mosek.com — register with UBC email
    License file goes at: ~/mosek/mosek.lic
    Install: pip install cvxpy mosek

Fallback: ECOS
    pip install cvxpy (includes ECOS)
    Use only if MOSEK unavailable — slower and less reliable for large SOCPs

---

## NUMERICAL STABILITY RULES

Always apply before solving:
    Add 1e-8 * I regularization to G if condition number > 1e10
    Clip negative eigenvalues of H to zero before computing H^{1/2}
    Check condition number of M before inverting — warn if > 1e12
    If problem infeasible, try alpha=0 before logging as infeasible

---

## FILE STRUCTURE

robust_portfolio/
├── CLAUDE.md                          ← this file
├── robust_portfolio_backtest.ipynb    ← main notebook (everything here)
├── data/
│   ├── wrds/
│   │   ├── constituents.parquet
│   │   ├── monthly_returns_raw.parquet
│   │   ├── delistings.parquet
│   │   ├── return_panel.parquet
│   │   └── excess_return_panel.parquet
│   ├── factors_monthly.parquet
│   ├── factors_daily.parquet
│   ├── macro/
│   │   ├── high_yield_spread.parquet
│   │   └── nber_recessions.parquet
│   └── [Bloomberg CSVs already here]
└── results/
    ├── backtest_omega090.pkl
    ├── backtest_omega095.pkl
    ├── backtest_omega099.pkl
    └── figures/

---

## NOTEBOOK STRUCTURE

All work goes in robust_portfolio_backtest.ipynb.
Use markdown headers for each section.
Each section has:
    Markdown cell: what this section does and why
    Code cells: implementation written by Simon
    Output cells: results and verification checks

Sections in order:
    1. Data Loading and Cleaning
    2. Parameter Estimation (rolling regression)
    3. SOCP Implementation (CVXPY)
    4. Sanity Checks (toy example, 20 stocks)
    5. Benchmark Implementations
    6. Rolling Backtest Engine
    7. Transaction Cost Extension (Phase 2)
    8. Performance Analysis
    9. Dual Variable Analysis (novel contribution)

Do not proceed from one section to the next until all 
verification checks in the current section pass.

---

## CURRENT STATUS

Math chapter: COMPLETE
    - Primal SOCP derived and explained
    - Lagrangian fully written
    - All stationarity conditions derived
    - Dual program derived and written
    - Strong duality proved via Slater's condition
    - KKT complementary slackness conditions written
    - Economic interpretations for all dual variables
    - Three literature connections:
        Corollary 1: Markowitz recovery as rho -> 0
        Remark 1: Ledoit-Wolf shrinkage connection
        Proposition 1: BCZ Wasserstein DRO generalization

Empirical chapter: IN PROGRESS
    - Data downloaded (Bloomberg, Ken French, FRED)
    - WRDS access obtained
    - Starting with Table 1, 2, 3 pulls from CRSP
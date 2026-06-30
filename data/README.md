# Data Sources

## Included (public domain)

| File | Source | Description |
|------|--------|-------------|
| `F-F_Research_Data_5_Factors_2x3_daily.csv` | [Ken French Data Library](https://mfrench.dartmouth.edu/data_library.aspx) | Fama-French 5 factors, daily |
| `F-F_Research_Data_5_Factors_2x3.csv` | Ken French Data Library | Fama-French 5 factors, monthly |
| `F-F_Research_Data_Factors_daily.csv` | Ken French Data Library | Fama-French 3 factors, daily |
| `F-F_Momentum_Factor_daily.csv` | Ken French Data Library | Momentum factor, daily |
| `processed/` | Derived | Parquet files produced by Steps 1–2 of the main notebook |

## Not included (proprietary — Bloomberg terminal required)

| File | Bloomberg ticker / BDH field | Description |
|------|-------------------------------|-------------|
| `SPX_CUR_DATA.csv` | SPX Index members, `PX_LAST` TRI | S&P 500 total return index per constituent |
| `SPX_RETURN_DATA.csv` | SPX Index, `DAY_TO_DAY_TOT_RETURN_GROSS_DVDS` | S&P 500 index-level daily returns |
| `WRDS_SPY_DATA_MASTER.csv` | WRDS / CRSP | SPY holding-level data from WRDS |
| `GB3GVT_DATA.csv` | `GB3GVT Index`, `PX_LAST` | 3-month US T-bill yield |
| `MOVE_DATA.csv` | `MOVE Index`, `PX_LAST` | ICE BofA MOVE bond volatility index |
| `USGG10Y_DATA.csv` | `USGG10YR Index`, `PX_LAST` | 10-year US Treasury yield |
| `VIX_DATA.csv` | `VIX Index`, `PX_LAST` | CBOE VIX equity volatility index |

To reproduce the full backtest from raw data, pull the Bloomberg fields above via BDH,
save them as CSV in this directory matching the filenames above, then run Steps 1–2 of
`robust_portfolio_backtest.ipynb` to regenerate `processed/`.

The processed parquet files are included in the repo so the notebook can be run from
Step 2 onwards without Bloomberg access.

# Portfolio Value-at-Risk (VaR) & Optimization

A parametric Value-at-Risk analysis and portfolio optimization for a 5-stock Indian equity portfolio (ICICI, HDFC, Kotak, Infosys, Reliance), benchmarked against the NIFTY 50.

## What this project does

1. **Cleans and processes** daily closing price data for 5 stocks + NIFTY (253 trading days, Mar 2025–Mar 2026).
2. **Computes daily log returns**, individual asset statistics (mean, variance, std. dev.), and the asset covariance matrix.
3. **Estimates 1-day Value-at-Risk** at 95% and 99% confidence using the **parametric (variance-covariance) method**.
4. **Compares three portfolio weighting schemes**:
   - **Equal weights** (20% each) — baseline
   - **Manual weights** — optimized in Excel using Solver, subject to no short-selling
   - **Algorithmic weights** — optimized in Python with `scipy.optimize.minimize` (SLSQP), minimizing portfolio volatility
5. **Calculates beta** for each stock relative to NIFTY, and summarizes all weight profiles side by side.
6. **Backtests the VaR model** with a 60-day rolling window and a Kupiec proportion-of-failures test, to check whether the model's predicted breach rate actually holds up out-of-sample.
7. **Computes Expected Shortfall (CVaR)** alongside VaR, since VaR alone says nothing about how bad the tail beyond it actually is.
8. **Cross-checks parametric VaR against historical simulation VaR** (empirical quantiles, no normality assumption).
9. **Traces the efficient frontier and solves for the maximum-Sharpe portfolio**, going beyond pure minimum-variance optimization.
10. **Visualizes** the efficient frontier, cumulative portfolio growth vs. NIFTY, and the VaR backtest with breach points marked.

## Results

### VaR & Expected Shortfall (parametric)

| Weighting Scheme | 95% VaR | 95% ES (CVaR) | 99% VaR | 99% ES (CVaR) |
|---|---|---|---|---|
| Equal Weights | -4.18% | -5.19% | -5.82% | -6.63% |
| Manual (Excel Solver) | -2.55% | -3.16% | -3.55% | -4.04% |
| Algorithmic (SciPy) | -1.79% | -2.22% | -2.50% | -2.85% |

The algorithmic optimizer reduced 95% daily VaR by more than half relative to equal weighting, primarily by concentrating allocation in the lower-volatility, lower-covariance names (ICICI, Infosys, Reliance) and capping the two high-volatility names (HDFC, Kotak) at their 5% floor. Expected Shortfall is consistently 15-25% worse than VaR across all three schemes — a reminder that VaR alone understates how bad the tail actually is.

### Parametric vs. historical simulation VaR

| Weighting Scheme | 95% VaR (Parametric) | 95% VaR (Historical) | 99% VaR (Parametric) | 99% VaR (Historical) |
|---|---|---|---|---|
| Equal Weights | -4.18% | -1.09% | -5.82% | -2.99% |
| Manual (Excel Solver) | -2.55% | -1.38% | -3.55% | -3.06% |
| Algorithmic (SciPy) | -1.79% | -1.23% | -2.50% | -2.88% |

Parametric VaR is noticeably more conservative than historical simulation at the 95% level for all three schemes — largely driven by Kotak's very high standard deviation (10.3% daily) inflating the normal-distribution tail more than the actual empirical return distribution warrants. At 99%, the picture is more mixed: for Equal Weights and Manual Optimized, the gap between the two methods narrows, but for the Algorithmic portfolio it actually **flips sign** — historical simulation VaR (-2.88%) is worse than parametric (-2.50%), meaning the empirical tail is fatter than the normal approximation predicts once the portfolio is concentrated in fewer names. That's a useful reminder not to assume the normal approximation's error runs in one consistent direction across every portfolio.

### VaR backtest (60-day rolling window, 192 out-of-sample days)

| Confidence | Breaches | Observed Rate | Expected Rate | Kupiec Test (p-value) |
|---|---|---|---|---|
| 95% | 5 | 2.60% | 5.00% | 0.069 — not rejected |
| 99% | 4 | 2.08% | 1.00% | 0.178 — not rejected |

Both models pass the Kupiec test at the 5% significance level, though the 95% VaR model is somewhat conservative (fewer breaches than expected) and the 99% model shows more breaches than its target — worth flagging as a direction for further tuning (e.g. a longer rolling window or fat-tailed distribution).

### Efficient frontier & max Sharpe

The maximum-Sharpe optimization landed on a corner solution (80% Kotak) with a **negative** Sharpe ratio. This isn't a bug — 4 of the 5 stocks had a negative average daily return over this sample period, so there's no allocation with positive expected excess return; the optimizer is just finding the least-bad option. It's a useful illustration of why short-sample return estimates shouldn't be taken at face value for forward-looking allocation decisions.

**Betas vs. NIFTY:** ICICI 0.22, HDFC -0.14, Kotak 0.42, Infosys 0.01, Reliance 0.13 — all five names show relatively low/weak correlation with the broader market over the sample period.

## Files

- `Aditya_Birla_VaR_Optimization.ipynb` — main analysis notebook
- `stock_prices.xlsx` — input data (daily closing prices; see **Data Format** below)

## Data Format

The notebook expects an Excel file named `stock_prices.xlsx` with a sheet called `Closing Prices`, where:
- The first 4 rows are skipped (header/title rows)
- Columns (after skipping) are, in order: an index/day column, `Date`, `ICICI`, `HDFC`, `Kotak`, `Infosys`, `Reliance`, `NIFTY`

If you're adapting this for your own tickers, update the `tickers` list and column selection in the data-loading cell accordingly.

## How to run

```bash
pip install pandas numpy scipy openpyxl
jupyter notebook Aditya_Birla_VaR_Optimization.ipynb
```

Run all cells in order. Place `stock_prices.xlsx` in the same directory as the notebook.

## Methodology notes & limitations

- **Parametric VaR assumes normally distributed returns.** The historical-simulation comparison confirms this matters in practice here: parametric VaR is materially more conservative than the empirical estimate at 95%, driven by Kotak's outsized volatility.
- **Sample period is ~1 year of daily data**, so covariance, beta, and especially mean-return estimates should be treated as indicative rather than stable, long-run parameters — the degenerate max-Sharpe result is a direct consequence of this.
- **The backtest uses a 60-day rolling window** re-estimated daily with no lookahead; 192 out-of-sample days is a reasonably small sample for a Kupiec test, so treat the p-values as suggestive rather than conclusive.
- **The optimizers are constrained** to weights between 5% and 100% per asset with no short-selling.
- NIFTY is used strictly as a market benchmark for beta calculation and is excluded from the portfolio covariance matrix and optimization.
- **Risk-free rate for the Sharpe ratio** is an approximation (6.7% p.a.), based on the general range the Indian 10-year G-Sec yield traded in through 2026 (~6.6-6.9%), not a rate pulled for the exact sample dates.

## Possible extensions

- Monte Carlo VaR, as a third comparison point alongside parametric and historical simulation
- A longer or multi-window backtest (e.g. 30-day and 120-day rolling VaR side by side) to see how window length affects breach rates
- GARCH-based volatility forecasting instead of a flat rolling standard deviation, to capture volatility clustering
- Extending the Sharpe-ratio optimization with a longer sample or shrinkage-adjusted expected returns, to avoid the corner-solution problem seen here

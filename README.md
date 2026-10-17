# Climate Adaptation Portfolio Analysis

A Python project for analysing a manually selected international equity universe around climate adaptation. It combines a liquidity screen, constrained portfolio optimization, Monte Carlo simulations and a rule based ESG adjustment, then exports the results to Excel.

## Setup and usage

Install the dependencies:

```bash
pip install yfinance pandas numpy scipy statsmodels openpyxl
```

Save the Python code as `portfolio_analysis.py` and run:

```bash
python portfolio_analysis.py
```

You can also run it in a Jupyter notebook or Google Colab. An internet connection is required to download prices and trading volumes from Yahoo Finance.

If copying code from Markdown, remove any formatting escapes such as `\#`, `\*` and `\_`, and use ordinary spaces for indentation.

## Configuration

The main settings are defined at the top of the code:

| Setting | Default |
| --- | --- |
| Stock universe | 43 manually selected tickers across multiple markets |
| Benchmark | `ACWI`, an ETF used as the market proxy |
| Start date | `2010-01-01` |
| Annual risk-free rate | 4% |
| Liquidity threshold | 1,000,000 in average daily price times volume |
| Stock weight bounds | 1% to 10% per retained stock |
| Country weight cap | 39.9% per country |
| Monte Carlo simulations | 10,000 paths over 10 years |

No end date is specified, so the data window extends to the latest available observations when the code runs. Country assignments and ESG inputs are maintained in `ticker_country` and `esg_data`.

## Method

### 1. Screen the stock universe

Download daily adjusted closing prices and trading volumes, then estimate average daily trading value as the mean of price multiplied by volume. Retain tickers that meet the numerical liquidity threshold.

Next, download monthly adjusted prices for the retained stocks and benchmark. Keep series with at least 70% nonmissing price observations, calculate simple monthly returns and drop rows with missing returns.

### 2. Optimize portfolio weights

Use historical mean monthly returns and their covariance matrix to maximize the estimated Sharpe ratio with SciPy's SLSQP solver. The monthly risk-free rate is the annual assumption divided by 12.

The base portfolio is fully invested, with weights summing to one. Each retained stock receives a weight between 1% and 10%, and each country is capped at 39.9% using the supplied country mapping.

The code calculates annualized mean return, volatility and Sharpe ratio, along with alpha, beta, information ratio and historical maximum drawdown. Alpha and beta come from an OLS regression of portfolio returns on benchmark returns.

### 3. Simulate portfolio outcomes

Generate 120 monthly return observations per path from a multivariate normal distribution using the historical mean and covariance estimates. Apply the optimized weights and compound the simulated portfolio returns.

Each path produces a terminal wealth multiple and an annualized compounded return. For example, a wealth multiple of `2.0` represents a 100% cumulative gain, even though the workbook labels that column "Cumulative 10Y Return".

### 4. Apply the ESG adjustment

Multiply each optimized weight by the factors below, then normalize the adjusted weights to sum to one.

| Input | Multiplier |
| --- | --- |
| ESG rating: AAA | 1.15 |
| ESG rating: AA | 1.10 |
| ESG rating: A | 1.05 |
| Revenue flag: YES | 1.15 |
| Revenue flag: NO | 1.10 |

Missing ESG entries default to the A and NO factors. The ratings and revenue flags are manually supplied inputs; the code does not retrieve or verify them.

The adjusted allocation is evaluated using the same historical monthly returns. It is a weight adjustment rather than a second constrained optimization.

## Excel output

Results are saved in the working directory as:

```text
Climate_Adaptation_Portfolio_Results_YYYYMMDD_HHMMSS.xlsx
```

| Worksheet | Contents |
| --- | --- |
| Prices | Monthly adjusted prices |
| Monthly Returns | Stock and benchmark returns |
| Optimised Weights | Base portfolio weights |
| Portfolio Summary | Return, risk and benchmark metrics |
| Monte Carlo Data | Simulated wealth multiples and annualized returns |
| Top 5 Stocks | Largest base portfolio positions |
| Top 3 Countries | Largest country allocations |
| Weightage Method | Optimization method and assumptions |
| Country Weightage | Stock contributions within each country |
| Liquidity | Estimated average daily trading values |
| ESG Adjusted Weights | Original and adjusted allocations |
| ESG-Adjusted Portfolio | Adjusted return, volatility and Sharpe ratio |

## Limitations

- **Currency handling:** the threshold is named as USD in the code, but trading values are calculated in each ticker's quoted currency without conversion. Portfolio returns also combine local currency stock returns without modelling exchange rates.
- **Solver checks:** the code uses `result.x` without checking solver success or constraint violations. A reduced stock universe can make the constraints infeasible.
- **ESG constraints:** normalization after the ESG adjustment can breach the original stock or country limits. Those limits are not reapplied.
- **Historical evaluation:** weights are selected and evaluated on the same data. There is no out-of-sample validation, transaction cost model or turnover analysis. Applying fixed weights each month implicitly assumes monthly rebalancing.
- **Return estimates:** annual mean return is monthly mean return multiplied by 12, not historical CAGR. Regression alpha uses raw returns rather than excess returns.
- **Simulation assumptions:** monthly returns are sampled independently with fixed parameters and a normal distribution. Results depend on these assumptions and vary between runs because no random seed is set.


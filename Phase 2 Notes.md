# Phase 2 Notebook Review: Indian Equity Portfolio Construction & Optimization

## Executive Summary
This notebook implements a sophisticated financial analytics pipeline designed to construct an optimal equity portfolio from a selection of 10 major Indian stocks. It moves beyond simple analysis to apply **Modern Portfolio Theory (MPT)** for allocation, **ARIMA** for short-term return forecasting, and **Monte Carlo simulation** for risk assessment. The code is structured as a production-ready script, modularized into distinct logical sections.

The core objective is to allocate a capital of **₹10,000,000** efficiently, balancing risk and reward, rather than speculatively picking winners.

---

## Section-by-Section Narration & Logic Analysis

### 1. Libraries & Setup
The script begins by importing the standard data science and financial stack.
- `yfinance`: The interface to fetch market data.
- `numpy` (`np`) & `pandas` (`pd`): The backbone for vector algebra and time-series manipulation.
- `scipy.optimize` (`minimize`): The mathematical engine used to solve the optimization problem (finding optimal weights).
- `statsmodels` (`ARIMA`): Used for statistical time-series forecasting.

**Why:** We need specialized tools for distinct tasks—`yfinance` for data, `scipy` for the solver, and `statsmodels` for econometrics. Warnings are silenced to maintain a clean output log.

### 2. Parameters & Universe Definition
```python
CAPITAL = 10_000_000
RISK_FREE_RATE = 0.06
STOCKS = ["RELIANCE.NS", "TCS.NS", ..., "BHARTIARTL.NS"]
```
**Why:** Defining constants at the top follows **Clean Code** principles (Single Source of Truth). The universe consists of blue-chip Nifty 50 stocks across sectors (Energy, IT, Banking, FMCG), ensuring diversification potential. The 6% risk-free rate is a proxy for the Indian government bond yield, crucial for calculating the Sharpe Ratio.

### 3. Data Ingestion & Processing
The script fetches Adjusted Close prices for the defined date range (2022-2024).
```python
prices = yf.download(STOCKS, ...)["Close"]
returns = prices.pct_change().dropna()
```
**Logic:** Raw prices aren't stationary; they have trends. We convert them to **log returns** or **percentage returns** (`pct_change`) to make the data stationary and comparable. This is the input for all subsequent volatility calculations.

### 4. Mathematical Foundation (Returns & Risk)
```python
mean_returns = returns.mean() * 252
cov_matrix = returns.cov() * 252
```
**Why:**
- **Annualization:** Financial metrics are standardly expressed in annual terms. We multiply daily mean returns by **252** (approximate trading days in a year).
- **Covariance Matrix:** This is the heart of MPT. It captures not just individual stock volatility but exactly *how* stocks move relative to each other. This allows the optimizer to find "hedges" within the portfolio—stocks that don't crash together.

### 5. Portfolio Optimization Engine
This section defines the objective function and constraints for the solver.

**Objective Function:**
The goal is to maximize the **Sharpe Ratio** (Risk-Adjusted Return). Since `scipy.optimize.minimize` searches for minima, we define a helper function `negative_sharpe` that returns the *negative* of the Sharpe Ratio. Minimizing the negative is mathematically equivalent to maximizing the positive.

**Step-by-Step Optimization Logic:**
1.  **Solver:** `SLSQP` (Sequential Least Squares Programming) is used. It handles constrained non-linear optimization problems well.
2.  **Constraints:** `np.sum(x) - 1`. This enforces that the sum of all weights must equal exactly 1 (100% allocation). No leverage.
3.  **Bounds:** `(0, 0.25)`. Each stock can hold between 0% and 25% of the portfolio.
    *   **Why:** This prevents "corner solutions" where the mathematical optimizer might dump 100% of cash into a single high-performing stock. It forces diversification.

```python
opt = minimize(negative_sharpe, init_weights, method='SLSQP', bounds=bounds, constraints=constraints)
weights = opt.x
```
**Execution:** The solver iteratively adjusts `weights`, checks the covariance matrix to calculate volatility, checks the mean returns, computes the Sharpe ratio, and repeats until it hits the global maximum within the 25% limit.

### 6. Capital Allocation
The optimal weights are mapped back to the capital of ₹1 Cr.
```python
allocation = pd.DataFrame({ ... "Investment_₹": np.round(weights * CAPITAL, 0) })
```
**Outcome:** A clear, actionable trade list. You see exactly how much of Reliance or HDFC Bank to buy.

### 7. Performance Visualization (Equity Curve)
The script calculates the hypothetical growth of this optimized portfolio over the training period.
```python
portfolio_returns = returns.dot(weights)
equity_curve = (1 + portfolio_returns).cumprod()
```
**Dry Run Logic:**
1.  Take the `(10, 1)` array of optimal weights.
2.  Take the `(T, 10)` matrix of daily stock returns.
3.  **Dot Product:** Create a single time series `(T, 1)` representing the daily return of the *aggregate portfolio*.
4.  **Cumprod:** Compounding these daily returns (`(1+r1)*(1+r2)...`) generates the equity curve starting at 1.0.

### 8. Forecasting (ARIMA)
The script shifts from descriptive to predictive analytics using **ARIMA (AutoRegressive Integrated Moving Average)**.
```python
arima_model = ARIMA(portfolio_returns, order=(1, 0, 1))
```
**Why ARIMA(1,0,1)?**
-   **AR(1):** Assumes today's return depends partly on yesterday's return (momentum/mean reversion).
-   **I(0):** Assumes the data is already stationary (returns usually are, prices are not). No differencing needed.
-   **MA(1):** Assumes today's return depends on the error term of yesterday (shock persistence).

**Caution:** As noted in the script's insights, this provides a statistical expectation (mean forecast), not a crystal ball. Financial time series often defy simple low-order linear models (Efficient Market Hypothesis).

### 9. Monte Carlo Simulation
Finally, the script acknowledges uncertainty by simulating 200 alternative future paths.
```python
simulated_daily_returns = np.random.normal(mean, std, 252)
```
**Why:** History is just one realized path. Simulation asks: "Given the portfolio's statistical properties (mean, volatility), what are 200 other ways the next year could play out?"
This visualizes **Tail Risk**—the spread of the lines shows the range of probable outcomes.

---

## Summary of Flow
1.  **Ingest:** Get raw prices, compute returns.
2.  **Model:** Build the covariance matrix to understand risk relationships.
3.  **Optimize:** Use a solver to find the "Efficient Frontier" portfolio that offers the best return per unit of risk, capped at 25% per stock.
4.  **Allocate:** Translate percentages to Rupees.
5.  **Forecast & Stress Test:** Project the portfolio forward using both linear models (ARIMA) and stochastic simulation (Monte Carlo) to set realistic expectations for the user.

**Outcome:** A data-backed investment plan that minimizes uncompensated risk through diversification and robust mathematics.

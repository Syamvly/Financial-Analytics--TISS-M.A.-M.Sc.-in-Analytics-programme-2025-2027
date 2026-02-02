# Python Notebook Review: NIFTY Options Trading System

This document provides a professional, section-by-section review and narration of the Python code found in `file 1.ipynb`. The notebook implements a quantitative backtesting engine for an options trading strategy on the NIFTY index, driven by volatility regimes.

## Overview
The notebook iteratively builds a trading system:
*   **Cell 1**: Establishes the core logic on a small dataset (December 2024).
*   **Cell 2**: Expands the backtest to the full year 2024 and introduces a granular `trade_log` for trade-by-trade tracking.
*   **Cell 3**: The final, production-ready script that includes visualization (charts) and comprehensive performance metrics. **This review focuses on the complete logic presented in Cell 3.**

---

## Detailed Code Narration (Cell 3)

The script is structured into 10 logical sections. Below is the step-by-step logic and "Why" behind it.

### Libraries & Setup
```python
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt
from scipy.stats import norm
from ta.momentum import RSIIndicator
```
*   **Narrative**: The script begins by importing standard financial data science libraries.
    *   `yfinance`: For fetching historical market data (OHLCV).
    *   `scipy.stats.norm`: Crucial for the **Black-Scholes** option pricing model (specifically the cumulative distribution function).
    *   `ta.momentum.RSIIndicator`: Used to calculate RSI on VIX, determining momentum in volatility.

### 1. Data Download
The script fetches daily closing prices for the **NIFTY 50** (`^NSEI`) and **India VIX** (`^INDIAVIX`) for the year 2024.
*   **Logic**: Data is concatenated into a single DataFrame `data` and aligned by date. `dropna()` ensures data integrity.
*   **Why**: Implied volatility (approximated by VIX) is the primary driver for option premium pricing and strategy selection.

### 2. Volatility Regime Identification (Signal Generation)
```python
data['VIX_Change'] = data['INDIA_VIX'].diff()
def classify_vix_regime(vix, change):
    if vix < 15: return "LOW_VOL"
    elif 15 <= vix <= 20 and change > 0: return "RISING_VOL"
    elif vix > 20: return "HIGH_VOL"
    else: return "NEUTRAL"
```
*   **Narrative**: This is the **Brain** of the strategy. It classifies the market environment based on absolute VIX levels and daily change.
*   **Why**: Option selling (Short Strangle) works best in low/stable volatility. Option buying (Long Straddle) profits from expanding volatility (`RISING_VOL`). High volatility (`vix > 20`) is flagged as `HIGH_VOL` (or "Hedge Only") to avoid extreme tail risk.

### 3. Strategy Selection
A functional mapping `select_strategy(regime)` assigns a trading logic to each regime:
*   `LOW_VOL` -> **SHORT_STRANGLE** (Selling OTM Call & Put to collect premium).
*   `RISING_VOL` -> **LONG_STRADDLE** (Buying ATM Call & Put to chase price explosions).
*   `HIGH_VOL` -> **HEDGE_ONLY** or **NO_TRADE** (Capital preservation mode).

### 4. Black-Scholes Option Pricing Model
```python
def bs_price(S, K, T, r, sigma, opt):
    ...
    d1 = (np.log(S/K)+(r+0.5*sigma**2)*T)/(sigma*np.sqrt(T))
    ...
```
*   **Narrative**: A custom implementation of the Black-Scholes-Merton formula.
*   **Variable Breakdown**:
    *   `S`: Spot Price (Nifty).
    *   `K`: Strike Price.
    *   `T`: Time to expiry (annualized).
    *   `sigma`: Volatility (derived from VIX).
*   **Why**: Since historical option premium data is often unavailable or expensive, the script **theoretically prices** options based on the spot price and VIX. This is a standard proxy in quant backtesting.

### 5. Strike Selection & Expiry
*   **Strike Selection**: `select_strikes(spot)` dynamically calculates strikes relative to the ATM (At-The-Money) price.
    *   `ATM`: Rounded to nearest 50.
    *   `OTM Call/Put`: ATM +/- 100 points (for Short Strangle).
    *   `Hedge`: ATM +/- 300 points (to define defined-risk spreads).
*   **Time to Expiry**: `days_to_expiry` calculates days remaining until Friday (assumed weekly expiry cycle).

### 6. Strategy Execution (The Event Loop)
The script iterates through the DataFrame row by row:
1.  **Extract State**: Gets current Spot (`S`), VIX (`sigma`), and Time (`T`).
2.  **Determine Strikes**: Computes the specific strike levels for the day.
3.  **Execute Logic**:
    *   **Short Strangle**: Calculates premium received from selling OTM strikes minus cost of hedging (Iron Condor structure).
    *   **Long Straddle**: Calculates cost paid for ATM options.
4.  **Transaction Costs**: Applies `TCOST = 0.15%` slippage/commission to the PnL.
5.  **Logging**: Appends the daily result to `trade_log`.

**Dry Run Logic**:
> On a `LOW_VOL` day, the system simulates selling a 21600 Call and 21400 Put. It uses Black-Scholes to estimate their value (e.g., ₹150 total). It subtracts the value of further OTM hedges. The net result is the daily theoretical PnL.

### 7. Position Sizing
```python
CAPITAL = 10_000_000  # ₹1 Crore
RISK_PCT = 0.01       # 1% Risk per trade
lots = max(1, int((CAPITAL * RISK_PCT) // MAX_LOSS_PER_LOT))
```
*   **Narrative**: Implements **Fixed Fractional Position Sizing**.
*   **Why**: It limits risk to 1% of equity per trade. By dividing risk capital by `MAX_LOSS_PER_LOT`, it dynamically adjusts the number of lots (contracts) traded, ensuring the account isn't over-leveraged during drawdowns.

### 8. Risk Metrics (VaR & ES)
Calculates **Value at Risk (VaR)** and **Expected Shortfall (ES)** at 95% confidence using historical simulation (`np.percentile`).
*   **Why**: Standard deviation (Sharpe) assumes normal distribution. VaR/ES quantify "Tail Risk"—how bad the worst 5% of trading days can get.

### 9. Final Performance Summary
Calculates aggregate metrics:
*   **Sharpe Ratio**: Risk-adjusted return.
*   **Max Drawdown**: Worst peak-to-valley equity drop.
*   **Equity Curve**: Computed via cumulative product of returns (`(1 + r).cumprod()`).

### 10. Visualization (Matplotlib)
Generates three key charts:
1.  **Cumulative P&L**: Time-series of strategy performance.
2.  **Daily Return Boxplot**: Visualizes volatility of returns for each strategy (Strangle vs. Straddle).
3.  **Return Contribution**: Bar chart showing which strategy drove the bulk of profits.

---

## Summary of Execution Flow
1.  **Ingest**: Load Nifty & VIX data.
2.  **Classify**: Tag every day with a Volatility Regime.
3.  **Simulate**: Loop through days, pricing virtual options based on regime rules.
4.  **Account**: Apply transaction costs and position sizing rules.
5.  **Analyze**: Output risk metrics and visual curves.

**Intended Outcome**: To validate whether a hybrid "Short Volatility in calm / Long Volatility in chaos" approach generates alpha over a buy-and-hold baseline on the Nifty index.

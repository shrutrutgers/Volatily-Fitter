# SPX/SPY Implied Volatility Surface Dashboard

## 1. Project Objective

The objective of this project is to construct, plot, and visualize the SPX/SPY implied volatility surface using live option-chain data, Treasury rates, dividend assumptions, repo bootstrapping, implied volatility inversion, and quadratic smile fitting.

## 2. Inputs and Data Sources

| Input | Purpose |
| --- | --- |
| Yahoo Finance | Option chains, spot prices, and SPY dividends |
| FRED | Treasury rates |
| Optional Excel Upload Mode | Allows the user to upload their own inputs instead of using live market data fetching |
| SPX vs SPY | Uses different modeling assumptions: European options for SPX and American options for SPY |

## 3. Modeling Assumptions

- SPX options are treated as European.
- SPY options are treated as American.
- SPX dividend yield is proxied using SPY's trailing four-quarter cash dividend yield.
- SPY dividends are modeled as discrete cash dividends.
- The risk-free curve is interpolated linearly.
- OTM options are used for visualization.
- A quadratic smile is fit tenor-by-tenor.

## 4. Implementation Walkthrough

### Step 1: Build Rate Curve

We pull daily constant maturity rates from FRED for maturities such as 1 month, 3 months, 6 months, 1 year, 2 years, and 5 years. These are theoretical yields of U.S. Treasury securities constructed daily by the Federal Reserve. For intermediate maturities, we perform linear interpolation to find the corresponding rates. This gives us the risk-free rate used in the exercise.

### Step 2: Pull and Clean Option Chain Data

We pull option chain data for multiple expiries from Yahoo Finance.

#### 2.a Expiry Selection

We select expiries closest to `[14D, 30D, 60D, 90D, 180D, 365D]`. This is done because Yahoo Finance can have many expiries, and we do not want the dashboard to look too dense. Also, many short-dated options are noisy, and IV inversion can become slow and expensive. The selected expiries give a cleaner term structure from short term to medium term to long term.

#### 2.b Strike Sampling

First, we select strikes that have quotes for both calls and puts. Then we select strikes in the range:

```text
0.8 * spot <= K <= 1.25 * spot
```

Finally, we keep strikes closest to target log-moneyness levels such as:

```text
[-0.08, -0.06, -0.04, ..., 0.10, 0.12]
```

This sampling keeps the full surface readable and avoids reading the whole chain.

#### 2.c Compute Mid Price and Spreads

We compute the mid price, spread, and relative spread, where relative spread is spread divided by mid price.

#### 2.d Bid/Ask Cleaning

We filter out options with wide spreads (`relative spread > 30%`), crossed or invalid quotes (`ask <= bid`), zero volume and open interest, very low mid price (`< 0.1`), or zero bids/asks.

#### 2.e Fallback Price Logic

Only if bid or ask is missing or NaN, we use the fallback price, provided that the fallback price is positive.

#### 2.f Pair Selection

For a given strike, we require both the call and put mid prices to pass the filter before using them as a pair. If either side fails the filter, the strike is dropped completely from the quote dictionary.

#### 2.g American Intrinsic Value Check

For American options like SPY, we reject prices below intrinsic value:

```python
intrinsic = max(S_eff - K, 0) for calls
intrinsic = max(K - S_eff, 0) for puts

if price < intrinsic:
    continue
```

#### 2.h IV Sanity Filter

After IV inversion, we drop IVs that are at/below 1% or above 200%. This removes bad results due to failed IV inversion by the numerical solver or other absurd outputs.

#### 2.i OTM Filtering

We compute the forward per tenor and mark options as OTM:

```text
Call is OTM if K >= F
Put is OTM if K <= F
```

The plots use OTM options only, so ITM quotes do not drive the smile/surface visualization.

### Step 3: Calculate Dividends and Dividend Yields

For SPY, this is straightforward with yfinance because SPY has actual cash dividend history. We use the most recent four quarterly dividends as a forward estimate of dividend payments.

For SPX, we use SPY's trailing four-quarter cash dividend yield as a proxy for SPX's continuous dividend yield.

### Step 4: Fit Repo Rate

For repo bootstrapping, we initialize the curve using a dividend/carry prior. For SPX, this prior is the SPY trailing four-quarter dividend yield proxy. For SPY, the same trailing four-quarter SPY dividend yield is used as the carry prior, while SPY cash dividends are still modeled separately as discrete payments.

For each tenor, we estimate a theoretical forward using the previously fitted repo rate. We choose the strike price closest to this theoretical forward because ATM options generally have the most liquidity and the least amount of microstructure noise. That is why IV estimates are usually best around this strike.

For European options, we use put-call parity to find the market-implied forward price:

```text
C - P = exp(-rT) * (F - K)
```

The repo rate can then be estimated from this market-implied forward.

For American options, put-call parity does not hold directly. However, around the ATM strike, call and put implied volatilities should be close when the carry assumption is correct. Therefore, for the ATM strike closest to the theoretical forward, we try different repo values and choose the one where call IV and put IV are closest.

If repo bootstrapping fails for a tenor, the model falls back to the dividend/carry prior. If live dividend data is unavailable, the model falls back to hardcoded dividend assumptions.

### Step 5: Calculate Forward Prices

We calculate forward prices for different tenors using the fitted repo rate.

### Step 6: IV Inversion

We calculate implied volatility for different maturities and log-moneyness levels, where `x = log(K/F)`.

For European options, we use the Newton-Raphson method to find the root of `BS(sigma) - price = 0`. In the Newton-Raphson update, the denominator is vega, which has a closed-form expression under Black-Scholes. If Newton-Raphson is unstable, we use bisection as a fallback.

For American options, we use bisection to find the root of `pricer(sigma) - price = 0`. The pricer solves the Black-Scholes PDE with an early exercise premium. To solve the PDE, we use a Crank-Nicolson finite difference method. To account for early exercise, we use the Brennan-Schwartz approximation during backward substitution, updating values as:

```text
V = max(payoff, continuation value)
```

### Step 7: Fit Quadratic Polynomial

We fit a quadratic polynomial to model IV versus log-moneyness and obtain coefficients `a`, `b`, and `c`:

```text
IV(x) = a + b*x + c*x^2, where x = log(K/F)
```

- `a` = ATM IV level
- `b` = skew
- `c` = curvature

### Step 8: Greeks Computation

For every option that survives cleaning and IV inversion, we compute delta and gamma under two conventions:

- **Plain (fixed-IV)**: the standard Black-Scholes spot sensitivity holding the option's own fitted IV constant. For SPX (European) this uses the closed-form expressions. For SPY (American) there is no closed form, so we use central finite differences through the Crank-Nicolson PDE pricer, with all bumped solves pinned to a single shared grid (re-centering the grid per solve introduces discretization noise that would swamp the second difference used for gamma).
- **Skew-adjusted (sticky-moneyness)**: the total spot sensitivity accounting for the option sliding along the fitted smile as spot moves. When spot is bumped, the forward moves, the option's log-moneyness `x = log(K/F)` changes, and its effective IV is re-read from the already-fitted quadratic `a + b*x + c*x^2` before repricing. Skew-adjusted greeks are then finite differences of these smile-consistent prices.

The bump size is 0.1% of effective spot. Greeks are computed for OTM options only (matching the smile/surface convention). In the dashboard, the Greeks tab shows one expiry at a time (selectable via dropdown, defaulting to the nearest expiry), and put deltas are plotted as call-equivalent delta (`1 + put delta`) so the put and call wings form one continuous curve; the table and CSV retain raw signed deltas.

For performance, the Crank-Nicolson time-stepping and Brennan-Schwartz solver are JIT-compiled with numba (with a pure-Python fallback when numba is unavailable). This makes each PDE solve sub-millisecond, so the full SPY American pipeline — repo bootstrap, IV inversion, and greeks — completes in well under a second.

#### The ATM kink in long tenors

The greek curves show a visible kink at the ATM boundary (`log-moneyness = 0`), and it grows with time to expiry. This is expected behavior, not a numerical artifact, and it is informative in its own right:

- The OTM series switches instrument at the boundary — puts to the left of `x = 0`, calls to the right — so adjacent points come from different quotes with different implied vols.
- The call-equivalent transform `1 + put delta = call delta` is an identity only under **European put-call parity**. American options do not satisfy parity as an equality (only as an inequality): American puts carry extra early-exercise premium, amplified by discrete dividends, so their transformed delta and their gamma sit systematically off the call line. For SPY (American) this is the dominant effect at long tenors — e.g. at the ~1-year tenor, put and call IVs at the same strike diverge by up to a few vol points in the wings.
- A single bootstrapped repo rate per tenor cannot perfectly reconcile noisy put and call quotes on both sides of the forward, and this misalignment also contributes to the jump.
- The pure dividend/carry offset (`1 − e^(−qT)`) is small by comparison (well under one delta point at one year).

SPX (European) shows a much smaller kink at the same spot, which confirms the American early-exercise premium — rather than dividends per se — as the primary driver of the SPY kink. In effect, the size of the ATM kink is a visual measure of how far the American options deviate from European parity at that tenor.

### Step 9: Visualization and Dashboard Output

Finally, we display the fitted repo curve, implied volatility table, quadratic surface coefficients, volatility smiles across tenors, the 3D implied volatility surface, and per-expiry greeks. The plotted smiles and surfaces use OTM options only, so the visualization focuses on the most liquid and commonly quoted side of the volatility surface.

The run summary reports the timestamp of the latest option trade seen in the fetched chains (in ET) and warns when quotes are more than 30 minutes old, which typically means the market is closed and the surface reflects the last session. Live data fetches are cached for 15 minutes, so repeated runs within that window reuse the same snapshot instead of re-hitting Yahoo Finance and FRED.

## 5. Dashboard Features

- Fetch latest market data (cached for 15 minutes; runs automatically on first visit and hourly thereafter, with the last downloaded snapshot persisted to disk so new visitors always see a surface)
- Upload Excel file
- View SPX/SPY surfaces
- View smiles across tenors
- View front-month smile
- View plain and skew-adjusted delta/gamma per expiry (Greeks tab with expiry dropdown)
- See the latest option quote timestamp and a stale-quote warning when the market is closed
- Download IV and greeks data as CSV
- Inspect repo curve, IV table, and coefficients

## 6. Interpretation Guide

- `a` = ATM volatility level
- `b` = skew
- `c` = curvature
- Higher left-wing IV means downside protection is more expensive
- Repo curve instability can signal noisy short-tenor data
- Spikes can come from short-dated OTM put skew
- A kink in the greek curves at the ATM boundary that grows with tenor measures the deviation of American options from European put-call parity (early-exercise premium); see the Greeks section

## 7. Limitations

- Yahoo Finance data may be delayed or stale (the dashboard surfaces the latest trade timestamp to make this visible)
- Bid/ask quality varies; some expiries have thin two-sided coverage, so a tenor can show only a few strikes
- The quadratic fit is simple and not arbitrage-free; this version does not perform arbitrage checks
- Skew-adjusted greeks inherit any misfit of the quadratic smile, since bumped IVs are read off the fitted coefficients
- No SVI/SABR smoothing yet
- Repo bootstrapping can be unstable for short tenors
- This is not intended for trading or risk decisions

## 8. Future Improvements

- SVI fitting
- Static arbitrage checks
- Calendar/butterfly violation flags
- Better dividend curve
- More robust quote cleaning
- Fit-quality metrics and residual plots
- Historical surface snapshots

## 9. Deployment

- Built with Streamlit
- Hosted on Streamlit Cloud
- Uses free data sources
- Public dashboard link / GitHub repository


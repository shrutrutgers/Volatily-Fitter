# Volatility Surface Dashboard

Interactive SPX/SPY implied volatility surface dashboard built with Streamlit and Plotly.

## Run locally

```bash
streamlit run app.py
```

The app can either:

- upload an `OptionData.xlsx` file, or
- fetch latest available SPX/SPY option chains with `yfinance` and Treasury rates from FRED.

## Features

- 3D fitted implied volatility surface and per-tenor volatility smiles (OTM options, quadratic fit in log-moneyness).
- Repo-rate bootstrapping: put-call parity for SPX (European), ATM call/put IV matching for SPY (American, Crank-Nicolson PDE with Brennan-Schwartz early exercise).
- Greeks tab with an expiry dropdown: plain (fixed-IV) and skew-adjusted (sticky-moneyness) delta and gamma; put deltas plot as call-equivalent (1 + delta) so the wings read as one curve.
- Run summary shows the latest option trade timestamp (ET) and flags stale quotes when the market is closed.
- Live fetches are cached for 15 minutes to avoid yfinance/FRED rate limits.
- The app runs automatically on first visit and saves the last downloaded data to disk (`latest_run.pkl`), so new visitors immediately see the most recent surface; data auto-refreshes when it is more than an hour old.
- CSV downloads for implied vols and greeks per asset.

## Project documentation

A detailed explanation of the data pipeline, modeling assumptions, option-chain cleaning, repo bootstrapping, IV inversion, and dashboard outputs is available here:

[Project Documentation](docs/IV_Project_Final_Document.md)

A PDF copy is also available for download:

[PDF Documentation](docs/IV_Project_Final_Document.pdf)

## FRED API key

For local use, add the key to `.streamlit/secrets.toml`.

For deployment, add this secret in Streamlit Community Cloud:

```toml
FRED_API_KEY = "your_key_here"
```

Do not commit `.streamlit/secrets.toml` to GitHub.

## Free hosting

Use Streamlit Community Cloud:

1. Push this folder to a GitHub repository.
2. Go to Streamlit Community Cloud.
3. Create an app from the repository.
4. Select `app.py` as the entrypoint.
5. Add `FRED_API_KEY` in app secrets.

The free deployment gives you a public `streamlit.app` URL that you can add to a resume or portfolio.

# FinAgent: AI-Powered Financial Data Agent

## Project Overview
FinAgent is an end-to-end pipeline that collects live financial data, cleans and engineers features, produces professional visualisations, and delivers LLM-powered natural-language analysis, all in a series of modular Jupyter notebooks.

The project integrates:
- Yahoo Finance market data
- FRED macroeconomic data
- Interactive Plotly dashboards
- AI-generated financial commentary using Groq Llama 3.3 70B

---

## Table of Contents

- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Modules](#modules)
- [Outputs](#outputs)
- [API Keys](#api-keys)
- [Dependencies](#dependencies)

---

## Project Structure

```text
FinAgent/
│
├── notebooks/
│   ├── Data_Collection.ipynb
│   ├── Data_Cleaning.ipynb
│   ├── Visualization.ipynb
│   └── AI.ipynb
│
├── data/
│   ├── raw/
│   └── processed/
│
├── outputs/
│   ├── charts/
│   └── reports/
│
├── .env.example
├── .gitignore
├── README.md
└── requirements.txt

```

---
## Configuration

All runtime settings live in `data/raw/finagent_config.json`, which is written automatically by `Data_Collection.ipynb`.  
You can edit the following variables at the top of that notebook before running:

| Variable | Default | Description |
|----------|---------|-------------|
| `TICKERS` | `['AAPL','MSFT','NVDA','GOOG']` | Equity symbols to analyse |
| `MACRO` | `['GC=F','CL=F']` | Commodity / macro futures |
| `PERIOD` | `5y` | Historical look-back (`1mo`, `1y`, `5y`) |
| `INTERVAL` | `1mo` | Data frequency (`1d`, `1wk`, `1mo`) |

---

## Modules

### Module 1 - Data Collection (`Data_Collection.ipynb`)

- Fetches OHLCV price data via **yfinance** for all tickers
- Downloads news headlines via **NewsAPI** (last 30 days)
- Saves raw CSVs to `data/raw/` and config JSON
- Handles API rate limits with `time.sleep()` between requests
- Gracefully skips symbols with no data

### Module 2 - Data Cleaning & Processing (`Data_Cleaning.ipynb`)

Cleaning steps applied to every symbol:

| Issue | Treatment |
|-------|-----------|
| Missing prices | Forward-fill (limit 2), then time-interpolation |
| Duplicate dates | Keep first occurrence, log count |
| Data type errors | `pd.to_numeric(errors='coerce')` |
| Outliers | Flagged with `Outlier_Flag` column (IQR × 3 or \|return\| > 40%) |

Feature engineering (all symbols):

- `Daily_Return`, `Log_Return`
- `MA_7`, `MA_30` (simple moving averages)
- `EMA_12`, `EMA_26`, `MACD`, `MACD_Signal`
- `BB_Upper`, `BB_Middle`, `BB_Lower` (Bollinger Bands, 20-day)
- `Volatility_30` (annualised rolling 30-day volatility)
- `RSI_14` (Wilder's smoothing method)

Also fetches **fundamental data** (P/E, EPS, margins, debt/equity, beta) via yfinance and **macro indicators** (Fed Funds Rate, CPI, Unemployment, 10Y Treasury Yield) via the **FRED API**.

### Module 3 - Visualization (`Visualization.ipynb`)

Produces a single interactive Plotly dashboard saved as `outputs/reports/dashboard.html`:

| Chart | Description |
|-------|-------------|
| Price + MA + Bollinger Bands | Trend line with overlaid MA-7, MA-30, and BB bands |
| Volume bars | Green/red volume bars colour-coded by daily return |
| Correlation heatmap | Pearson correlation of daily returns across all assets |
| RSI-14 panel | Momentum indicator with overbought (70) / oversold (30) lines |
| Fundamental P/E bars | Trailing vs. forward P/E comparison |
| Macro overlay | Fed rate, CPI YoY, 10Y yield, unemployment over time |

Static PNG exports are also saved (requires `kaleido`).

### Module 4 - AI Analysis (`AI.ipynb`)

Uses **Groq API (Llama 3.3 70B)** to generate five analytical sections, then exports a Markdown and a PDF report:

1. **Market Trend Summary** - direction, momentum, top/laggard with RSI commentary
2. **Anomaly Detection** - flags statistically extreme trading days and hypothesises causes
3. **Risk Commentary & Asset Comparison** - Sharpe-ratio ranking, diversification, investor profiles
4. **Fundamental Valuation Analysis** - P/E, margins, EPS quality, debt/equity ranking
5. **Macro Environment Analysis** - Fed policy, inflation, yield curve, labour market impact

All LLM prompts pass structured JSON data so outputs are grounded in real numbers.

---

## Outputs

After running all notebooks you will find:

```
outputs/
├── charts/
│   ├── chart1_performance.png.{html,png}
│   ├── chart2_correlation_heatmap.{html,png}
│   ├── chart3_return_distributions.{html,png}
│   ├── chart4_<TICKER>_rolling.{html,png}   (one per ticker)
│   ├── chart5_fundamentals.{html,png}
│   ├── chart6_macro_dashboard.{html,png}
└── reports/
    ├── ai_analysis_<DATE>.md
    └── ai_analysis_<DATE>.pdf
```

---

## API Keys

Three external API keys are required. **Never commit real keys to Git.**

| Key | Where to get it | Free tier |
|-----|----------------|-----------|
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) | Free |
| `NEWSAPI_KEY` | [newsapi.org](https://newsapi.org) | Free (100 req/day) |
| `FRED_API_KEY` | [fred.stlouisfed.org/docs/api](https://fred.stlouisfed.org/docs/api/api_key.html) | Free |

Copy `.env.example` to `.env` and fill in your keys. The notebooks read keys from `finagent_config.json` (generated on first run from the values in `.env`).

---

## Dependencies

See `requirements.txt` for the full pinned list. Key libraries:

| Library | Purpose |
|---------|---------|
| `yfinance` | Yahoo Finance price + fundamental data |
| `pandas` / `numpy` | Data manipulation and feature engineering |
| `plotly` | Interactive charts and dashboard |
| `matplotlib` / `seaborn` | Static chart exports |
| `groq` | Groq LLM API client |
| `requests` | NewsAPI and FRED REST calls |
| `reportlab` | PDF report generation |
| `kaleido` | Plotly static PNG export |

---

## Tips

- Run notebooks **in order** (1 → 2 → 3 → 4); each module reads outputs from the previous one.
- To change the asset universe, edit `TICKERS` at the top of `Data_Collection.ipynb` and re-run all notebooks.
- If NewsAPI returns no articles (free tier daily limit), the news JSON will be empty - the other modules continue normally.
- FRED macro data requires a valid `FRED_API_KEY`; without it the module falls back to synthetic demo data.

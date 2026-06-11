# Conflict Risk Forecasting from GDELT — BigQuery Pipeline

A self-contained Jupyter notebook that pulls GDELT event and TV-caption data from Google BigQuery, builds an analytic dashboard (time series, station × time heat map, actor word cloud, CAMEO event-type bar chart, geographic scatter), and trains a SARIMAX time-series model that forecasts daily conflict-event volume using TV mentions as an exogenous regressor.

The shipped example targets **Israel, October 2023 → October 2024**, but the notebook is parameterised — you can repoint it at any country and date window by editing one cell.

---

## What's in the repo

| File | Purpose |
|------|---------|
| `BigQuery_GDELT_Setup.ipynb` | The notebook (clean source, no embedded outputs) |
| `BigQuery_GDELT_Setup.executed.ipynb` | Same notebook with one full run's outputs/figures embedded — open this on GitHub to see what the pipeline produces without running it |
| `viz_0[1-9]_*.png` | Standalone PNGs of the nine figures, for embedding in reports/slides |
| `requirements.txt` | Python dependencies |
| `.env.example` | Template for the environment variables (`GCP_PROJECT`, optional `BQ_TOKEN` / `GOOGLE_APPLICATION_CREDENTIALS`). Copy to `.env` and fill in your own values. |
| `.gitignore` | Excludes `.env`, credential JSONs, checkpoints, etc. — your secrets stay local. |

---

## Prerequisites — one-time setup

You need:

1. A **Google Cloud Platform project** with billing enabled and the **BigQuery API** turned on.
   - Sign in at [console.cloud.google.com](https://console.cloud.google.com).
   - Note your **project ID** (looks like `my-project-id-12345`) or **project number** (a 12-digit integer). Either works.
   - Free tier: BigQuery gives **1 TB of query scans per month** at no charge. A full run of this notebook scans ~37 GB, so well within the free quota.

2. **Python 3.9 or newer** with the libraries in `requirements.txt`:
   ```bash
   pip install -r requirements.txt
   ```

3. **Authentication.** Pick one method:

   ### Option A — Application Default Credentials (recommended for personal use)
   Install the [gcloud CLI](https://cloud.google.com/sdk/docs/install), then:
   ```bash
   gcloud auth application-default login
   gcloud auth application-default set-quota-project YOUR_PROJECT_ID
   ```
   Credentials are stored in your user profile; Python libraries find them automatically.

   ### Option B — Short-lived access token (no install)
   ```bash
   gcloud auth print-access-token
   ```
   Paste the result into the `BQ_TOKEN` environment variable:
   ```bash
   # PowerShell
   $env:BQ_TOKEN = "ya29.a0..."
   # bash / zsh
   export BQ_TOKEN="ya29.a0..."
   ```
   Tokens expire after ~1 hour.

   ### Option C — Service account JSON (for servers / CI)
   In the Cloud Console, create a service account with the **BigQuery User** role, download the JSON key, then:
   ```bash
   # PowerShell
   $env:GOOGLE_APPLICATION_CREDENTIALS = "C:\path\to\key.json"
   # bash / zsh
   export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
   ```

---

## Running it

1. Copy the env template and fill in your values:
   ```bash
   cp .env.example .env
   # Then open .env and set GCP_PROJECT (and optionally BQ_TOKEN
   # or GOOGLE_APPLICATION_CREDENTIALS).
   ```
   `.env` is git-ignored, so your credentials stay on your machine.

2. Load the env vars before launching Jupyter:
   ```bash
   # bash / zsh
   set -a; source .env; set +a
   # PowerShell
   Get-Content .env | Where-Object {$_ -match '^\s*[^#].*='} |
     ForEach-Object { $kv = $_ -split '=', 2; [Environment]::SetEnvironmentVariable($kv[0].Trim(), $kv[1].Trim(), 'Process') }
   ```
   Or, if you prefer, hard-code `GCP_PROJECT` directly in the notebook's first code cell — but for a public fork, please use the `.env` file so nothing personal lands in git.

2. Launch Jupyter and open `BigQuery_GDELT_Setup.ipynb`:
   ```bash
   jupyter notebook
   ```

3. Run all cells. The first cell prints your configuration; the second confirms BigQuery auth.

4. To analyse a different country or window, change the values in the **Configure** cell (section 0):
   - `COUNTRY_CODE` — FIPS 2-letter code (e.g. `EG` for Egypt, `UP` for Ukraine, `IR` for Iran)
   - `COUNTRY_NAME` — display name for chart titles
   - `TV_KEYWORDS` — lowercase keywords to count in TV captions
   - `EVENTS_FROM` / `EVENTS_TO`, `TV_FROM` / `TV_TO` — date ranges
   - `LAT_MIN/MAX`, `LON_MIN/MAX` (in section 9.5) — geographic clip for the scatter plot

   Rerun all cells.

---

## Methodology summary

### Data

- **GDELT Events** (`gdelt-bq.gdeltv2.events_partitioned`) — every socio-political event coded from worldwide news since 1920. We use `AvgTone`, `GoldsteinScale`, `NumMentions`, the CAMEO action codes, and lat/long.
- **GDELT TV** (`gdelt-bq.gdeltv2.iatv_1gramsv2`) — counts of every word spoken in TV captions, per (day, station, show). Stream ended October 2024.

Both tables are partitioned; the notebook always filters on the partition column, which is the main cost lever.

### Visualisations

1. **Daily time series** — events, AvgTone, Goldstein with 7-day rolling means.
2. **Heat map** — top 12 TV stations × weeks, log scale.
3. **Word cloud** — top actors weighted by mention volume.
4. **CAMEO bar chart** — event-type distribution, red for hostile/coercive codes (13-20).
5. **Geographic scatter** — events mapped, sized by mention count, coloured by tone.

### Forecasting model — SARIMAX

| Component | Choice | Why |
|-----------|--------|-----|
| Target | Daily count of CAMEO codes 18/19/20 (Assault, Fight, Unconventional) | A clean "violence intensity" signal |
| Exogenous regressor | log(1 + daily TV mentions) | TV coverage is strongly correlated with on-the-ground event volume |
| Order | SARIMAX(1,1,1)(1,1,1,7) | Differencing handles the trend; (P,D,Q,s=7) captures weekly news cycles |
| Test split | Last 60 days | Out-of-sample evaluation |
| Baseline | 7-day mean | Sanity check; the model must beat this |

On the shipped Israel example: **MAE 587 vs naive 803 → +26.9% skill score**.

### Why SARIMAX and not Prophet / LSTM?

- **Prophet** — needs cmdstanpy / Stan; heavyweight install, especially on Windows.
- **LSTM / Transformer** — overkill for ~365 daily observations; high overfit risk.
- **XGBoost with lag features** — strong forecasts but loses the time-series interpretability and built-in confidence intervals that SARIMAX gives you for free.

---

## Cost notes

A full run scans approximately:

| Query | Scan | Approx. cost |
|-------|------|--------------|
| Events (1 country, 1 year) | ~15 GB | ~$0.09 |
| TV (5 keywords, 1 year) | ~22 GB | ~$0.14 |
| **Total** | **~37 GB** | **~$0.23** |

All under Google's 1 TB/month free tier. The notebook dry-runs every query first and prints the scan estimate, so you can abort before spending anything if filters are too loose.

---

## Limitations & responsible use

- **GDELT is a coding of news, not ground truth.** Reporting bias is real; events outside English-/major-language media are under-counted.
- **Correlation, not causation.** TV mentions and on-the-ground events share underlying news cycles; the model is descriptive, not causal.
- **GDELT TV ended Oct 2024.** For analyses past that date, drop the TV regressor or substitute another signal.
- **Forecasts are probabilistic.** Use confidence intervals, not point predictions, for any decision-grade use.
- **No personal identification.** The pipeline aggregates events and word counts only. Do not attempt to attribute named individuals from this data.

---

## License

MIT. See `LICENSE` if present, otherwise treat the code as MIT-licensed.

GDELT data is provided by the GDELT Project under its own terms — see [gdeltproject.org](https://www.gdeltproject.org/) for usage and attribution requirements.

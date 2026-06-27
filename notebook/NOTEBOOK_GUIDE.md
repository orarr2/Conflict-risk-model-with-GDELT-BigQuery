# Notebook guide - `Conflict risk model with GDELT BigQuery.ipynb`

Cell-by-cell reference for the notebook. Each entry lists what the cell does, the variables it consumes from earlier cells, the variables it produces, the BigQuery scan (if any), an estimated runtime, and known gotchas.

---

## Section 1 - What is GDELT?

Project pitch + side-by-side illustrations of the Events table and the GKG knowledge graph (from `assets/`), the research idea, scope, and method in one paragraph.

## Section 2 - Configure

Defines every module-level constant: imports (`pd / np / plt / sns / display / Path / textwrap`), `OUT_DIR`, BigQuery table refs, `COUNTRIES` dict (8 entities with FIPS + CAMEO + GKG codes), `ANCHOR` (Israel), `TARGETS` (the seven non-Israel entities), `SIGNALS`, `TODAY / W7_FROM / W30_FROM`. Sets `axes.grid = False` globally. Renders a settings table.

## Section 3 - Install + Auth

- **Install cell**: `%pip install -q google-cloud-bigquery google-cloud-bigquery-storage pandas numpy matplotlib seaborn wordcloud scikit-learn plotly networkx kaleido`. Renders a library-version table after install.
- **Auth cell**: prompts for project ID and `gcloud auth print-access-token` token via `getpass` (both hidden). Builds `bq` and `dry_run_mb`. Runs a free `SELECT 1` smoke test.

## Section 4 - Explore the Events table

- Schema dump printed as a horizontal `textwrap.fill` comma-separated list.
- 1-day events sample over the 8 countries.
- Summary stats: dtypes, describe(), top CAMEO root codes printed horizontally.
- 4-panel distribution figure: AvgTone, Goldstein, CAMEO roots (red = hostile), log10(NumMentions). No grid; bar values annotated where useful.

## Section 4.5 - Top actors word cloud

Dedicated 30-day query over the 8 target countries. Aggregates `Actor1Name + Actor2Name` weighted by `NumMentions`, filters generic stop tokens, renders a 1400×700 word cloud, prints the top-10 as a single horizontal line, and saves to `assets/viz_top_actors.png`.

## Section 5 - Explore the GKG table

- GKG schema printed horizontally.
- Top 25 GKG themes over the last 30 days, printed horizontally.

## Section 6 - News samples per country

10 random news rows per target country, columns: date, Actor1Name, Actor2Name, EventCode, AvgTone, GoldsteinScale, SOURCEURL. One DataFrame per country.

## Section 7 - Signal helpers

Defines `fetch_events`, `fetch_gkg`, `daily_internal_signals`, `daily_bilateral_signals`. Bilateral helper uses 3-letter CAMEO actor codes (`ISR`, `EGY`, `JOR`, …, `PSE`) plus an `ActionGeo` fallback for events where one actor's country code is null. Prints a narrator block confirming the four helpers are ready.

## Section 8 - Per-country internal instability

Excludes Israel (anchor). For each of the 7 targets:

```
share_score    = ( Σₛ wₛ · events_7d(s) )  /  ( Σₛ wₛ · events_30d(s) + 1 )
volume_score   = ( Σₛ wₛ · events_7d(s) ) min-max scaled across the 7 countries
internal_score = sqrt(share_score · volume_score)
```

Output table shows all five numbers (composite, share, volume, n_7d, n_30d). Bar chart with stretched x-axis; annotations on each bar show only `score   n_7d   n_30d` (no labels). Saves to `assets/viz_internal_instability.png`.

## Section 9 - Bilateral escalation pressure - Israel vs each X

For each of the 7 dyads:

```
raw(X)   = Σₛ wₛ · events_7d(s)
score(X) = ( raw(X) - minₓ raw ) / ( maxₓ raw - minₓ raw )
```

Min-max scaled across the seven dyads. Bar chart, annotations are `score   n_7d   n_30d`. Saves to `assets/viz_bilateral_pressure.png`. `_pair_data` dict is cached for reuse by sections 10 and 11.

## Section 10 - Correlation, covariance, and daily hostility time series

Builds a daily-hostility matrix (rows = 30 days, columns = 7 dyads). Renders:

- Daily hostile-event time series - one line per dyad. Saves to `assets/viz_daily_hostile_events.png`.
- Side-by-side Pearson correlation + covariance heatmaps in green palette. Saves to `assets/viz_corr_cov.png`.
- Top correlated dyad pairs table; canary identification (highest mean off-diagonal correlation).

## Section 11 - Combined risk dashboard (interactive)

Interactive Plotly scatter: x = internal_score, y = bilateral score, marker size = 7-day hostile event count, marker colour = co-movement. Hover any marker for all supporting numbers. Static PNG export via kaleido to `assets/viz_combined_dashboard.png` for the README. Ranked summary table + headline call-out.

## Section 12 - GKG entity co-occurrence network

Fetches `V2Persons + V2Organizations` from GKG for each country in parallel over the last 30 days. Builds a NetworkX graph (red country nodes, blue entity nodes, edges weighted by co-mention count). Saves to `assets/viz_gkg_entity.png`.

## Section 13 - Per-country word clouds (general)

4×2 grid of word clouds - one per country - built from actor names in that country's events over the last 30 days. Same stop-word filter as section 4.5.

## Section 14 - Per-country word clouds (bilateral vs Israel)

4×2 grid (one slot blank to fit 7 targets in a 4×2 layout). Each cloud is built from actor names appearing in the bilateral event stream `(Israel ↔ X)` only.

## Section 15 - Per-country 7d vs 30d time-series

8 panels - one per country - showing the daily hostile-event count over the last 30 days. The trailing 7-day window is shaded red. Each panel title shows the 7-day total and 30-day total.

## Section 16 - Limitations and responsible use

The disclaimer set: not P(war), no calibration, reporting bias, correlation ≠ causation, bolt-from-the-blue blind spot, Gaza vs West Bank geo-disambiguation caveat, no individual attribution, GDELT acknowledgement.

---

## End-to-end numbers

| Metric | Value |
|---|---|
| Total BigQuery scan per full run | ≈ 600 MB – 1.5 GB |
| Total wall-clock | ≈ 3-5 min |
| Static PNG files exported to `assets/` per run | 6 + 2 illustrations |

## PNGs the notebook writes to `assets/`

| File | Generated by | Content |
|---|---|---|
| `gdelt_events_map.png` | one-time | Section 1 illustration of GDELT Events 2.0 |
| `gdelt_gkg_graph.png` | one-time | Section 1 illustration of GDELT GKG 2.0 |
| `viz_top_actors.png` | section 4.5 | Top actors across all 8 countries |
| `viz_internal_instability.png` | section 8 | Per-country internal instability bar chart |
| `viz_bilateral_pressure.png` | section 9 | Bilateral pressure bar chart |
| `viz_daily_hostile_events.png` | section 10 | Daily hostile-event time series (7 dyads) |
| `viz_corr_cov.png` | section 10 | Pearson correlation + covariance heatmaps |
| `viz_combined_dashboard.png` | section 11 | Static export of the Plotly dashboard |
| `viz_gkg_entity.png` | section 12 | GKG entity co-occurrence network |

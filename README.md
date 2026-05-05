# Epic Games Store Giveaways: Causal Effect on Steam Player Engagement

A full research pipeline investigating whether free game giveaways on the Epic Games Store increase or decrease engagement for the same title on Steam. The pipeline spans raw data acquisition, automated and manual data cleaning, panel construction, staggered difference-in-differences estimation, and a written research paper.

## Research Question

Does a free game giveaway on the Epic Games Store causally affect concurrent player counts for the same game on Steam? Motivated by anecdotal reports — such as the *Blood West* studio's account in January 2026 of an EGS giveaway nearly doubling their Steam revenue — this project uses a staggered DiD event study across 446 treated games from 2018–2026 to estimate the causal effect and characterize its dynamics.

For the motivating anecdote, see [this Reddit thread](https://www.reddit.com/r/Steam/comments/1qi94gk/epic_game_stores_free_giveaways_just_cause_a_huge/).

---

## Pipeline Overview

```
Step 1   Raw data (Kaggle)
            ↓
Step 2   Game ID disambiguation        scripts/game_id_disambiguation.ipynb
            ↓                          → raw_data/steam_matches.json
Step 3   Manual match cleanup          (edit raw_data/steam_matches_manual.json)
            ↓
Step 4   SteamCharts scraping          scripts/steam_charts_data.ipynb
            ↓                          → raw_data/steamcharts/all_monthly.csv
Step 5   Empirical analysis            scripts/empirical_tests.ipynb
            ↓                          → output/fig_*.pdf, summary_stats.csv
Step 6   Research paper                writeups/egs_steam_paper.tex / .pdf
```

---

## Data Sources

| Source | Description | Location |
|--------|-------------|----------|
| [Epic Games Free Giveaway History (Kaggle)](https://www.kaggle.com/datasets/prajwaldongre/epic-games-free-giveaway-history-20182025?resource=download) | 541 giveaway records from Dec 2018–2026, with game name, start/end dates, and genre | `raw_data/epic games data.csv` |
| Steam Store Search API | Used to map Epic game names to Steam App IDs | queried live, cached in `raw_data/steam_search_cache.json` |
| [SteamCharts](https://steamcharts.com) | Monthly average and peak concurrent player counts per app | scraped, cached in `raw_data/steamcharts/` |

---

## Pipeline Steps

### Step 1 — Download Epic giveaway data

Download `epic games data.csv` from the Kaggle link above and place it in `raw_data/`.

---

### Step 2 — Game ID disambiguation ([scripts/game_id_disambiguation.ipynb](scripts/game_id_disambiguation.ipynb))

Matches each Epic game name to its Steam App ID.

**Requires** a Steam API key in `.env.local`:
```
STEAM_API_KEY=your_key_here
```

**What it does:**
1. Loads the Epic CSV (541 rows) and strips `(repeat)` suffixes from 77 duplicate giveaway entries, yielding 476 unique game names.
2. Queries `store.steampowered.com/api/storesearch` for each unique name. Results are cached incrementally to `raw_data/steam_search_cache.json` (~190 seconds on first run; resumable).
3. Scores each result against the query using `rapidfuzz.WRatio`:
   - Score ≥ 88 → single confident match (413 games)
   - Score < 88 → top 5 candidates kept for manual review (14 games)
   - No results → not found on Steam (49 games — F2P titles, delisted, collections, DLCs, or Epic-exclusives)
4. Saves `raw_data/steam_matches.json`:
   ```json
   { "Subnautica": [[264710, "Subnautica"]], "For Honor": [[304390, "FOR HONOR™"], ...], ... }
   ```
   Single-element lists are confident matches; multi-element lists need review; empty lists are not on Steam.

---

### Step 3 — Manual cleanup of `steam_matches_manual.json`

Copy `raw_data/steam_matches.json` → `raw_data/steam_matches_manual.json`, then edit manually:

- **Multiple candidates** (14 games): delete all but the correct `[appid, name]` pair.
- **No match** (49 games): for games that are on Steam under a non-standard name, manually add the correct `[appid, name]` entry. Epic-exclusives or F2P-only titles can stay as `[]` and will be skipped.

`steam_matches_manual.json` is the ground-truth mapping consumed by all downstream steps.

---

### Step 4 — Scrape SteamCharts ([scripts/steam_charts_data.ipynb](scripts/steam_charts_data.ipynb))

Scrapes monthly concurrent player data for every matched game.

**What it does:**
1. Reads `steam_matches_manual.json` and deduplicates by App ID.
2. Fetches `https://steamcharts.com/app/{appid}/chart-data.json` for each App ID. Monthly pre-aggregated points and hourly points (recent ~6 months) are both bucketed to monthly granularity.
3. Caches each game's result to `raw_data/steamcharts/{appid}.json` (safe to interrupt and resume).
4. Assembles all cached files into a single long-format panel: **`raw_data/steamcharts/all_monthly.csv`**.
5. Attaches treatment metadata to the panel (`all_giveaway_dates`, `repeat_event_it`).

| Column | Description |
|--------|-------------|
| `appid` | Steam App ID |
| `steam_name` | Game name on Steam |
| `epic_name` | Game name as listed in the Epic giveaway data |
| `month` | Month (YYYY-MM-01 format) |
| `avg_players` | Average concurrent players that month |
| `peak_players` | Peak concurrent players that month |
| `first_giveaway_date` | Date of the first Epic free giveaway for this game |
| `all_giveaway_dates` | Semicolon-separated list of every Epic giveaway start date for this game |
| `repeat_event_it` | 1 if month *t* falls in `[0, 12]` months after a *non-first* giveaway, else 0 |

> **Bot detection note:** SteamCharts uses Cloudflare. The scraper uses a full Chrome 120 User-Agent with matching `Sec-Fetch-*` headers and a warmed-up persistent session. Rate limit: 2 seconds per request.

---

### Step 5 — Empirical analysis ([scripts/empirical_tests.ipynb](scripts/empirical_tests.ipynb))

Staggered difference-in-differences event study on `all_monthly.csv`, using `first_giveaway_date` as the treatment date. Estimates the dynamic causal effect of an EGS giveaway on Steam player counts, with control games as the comparison group.

---

### Step 6 — Research paper ([writeups/egs_steam_paper.tex](writeups/egs_steam_paper.tex))

Full writeup of the research question, data, empirical strategy, results, and interpretation. Compile with `pdflatex` (run twice) or paste into Overleaf.

---

## Repository Layout

```
raw_data/
  epic games data.csv            # source: Kaggle download
  steam_search_cache.json        # cached Steam Store Search API responses
  steam_matches.json             # auto-generated appid mapping (Step 2 output)
  steam_matches_manual.json      # manually corrected mapping (Step 3 output)
  steamcharts/
    {appid}.json                 # per-game player count cache (Step 4)
    all_monthly.csv              # final panel dataset (Step 4 output)

scripts/
  game_id_disambiguation.ipynb   # Step 2: Epic → Steam ID mapping
  steam_charts_data.ipynb        # Step 4: scrape player counts + build panel
  empirical_tests.ipynb          # Step 5: DiD event study

output/
  fig_*.pdf, fig_*.png           # figures referenced in the paper
  summary_stats.csv              # summary statistics table

writeups/
  egs_steam_paper.tex            # Step 6: research paper (LaTeX source)
  egs_steam_paper.pdf            # compiled paper

.env.local                       # STEAM_API_KEY=... (not committed)
```

---

## Requirements

```
pandas
requests
python-dotenv
rapidfuzz
```

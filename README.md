# Epic Games Free Giveaway → Steam Player Count Pipeline

This project studies the effect of Epic Games Store free game giveaways on Steam concurrent player counts. The pipeline turns a raw giveaway history CSV into a monthly panel dataset of Steam player activity, ready for empirical analysis.

## Motivation

In January 2026, the studio behind *Blood West* publicly shared that the free giveaway of their game on the Epic Games Store led to a dramatic increase in Steam sales, nearly doubling their revenue on the platform. For more details, see [this discussion on Reddit.](https://www.reddit.com/r/Steam/comments/1qi94gk/epic_game_stores_free_giveaways_just_cause_a_huge/)

This project was inspired by such anecdotal reports, aiming to systematically analyze the impact of Epic's free giveaways on Steam player activity—shedding light on how cross-store promotions may drive real economic outcomes for game developers.

---

## Data Sources

| Source | Description | Location |
|--------|-------------|----------|
| [Epic Games Free Giveaway History (Kaggle)](https://www.kaggle.com/datasets/prajwaldongre/epic-games-free-giveaway-history-20182025?resource=download) | 541 giveaway records from Dec 2018 – 2025, with game name, start/end dates, and genre | `raw_data/epic games data.csv` |
| Steam Store Search API | Used to map Epic game names to Steam App IDs | queried live, cached in `raw_data/steam_search_cache.json` |
| [SteamCharts](https://steamcharts.com) | Monthly average and peak concurrent player counts per app | scraped, cached in `raw_data/steamcharts/` |

---

## Pipeline Steps

### Step 1 — Download Epic giveaway data

Download `epic games data.csv` from the Kaggle link above and place it in `raw_data/`.

---

### Step 2 — Game ID disambiguation (`scripts/game_id_disambiguation.ipynb`)

Matches each Epic game name to its Steam App ID.

**Requires** a Steam API key in `.env.local`:
```
STEAM_API_KEY=your_key_here
```

**What it does:**
1. Loads the Epic CSV (541 rows) and strips `(repeat)` suffixes from 77 duplicate giveaway entries, yielding 476 unique game names.
2. Queries `store.steampowered.com/api/storesearch` for each unique name. Results are cached incrementally to `raw_data/steam_search_cache.json` so re-runs are instant and interruptions are safe (~190 seconds on first run).
3. Scores each result against the query using `rapidfuzz.WRatio`:
   - Score ≥ 88 → single confident match (413 games)
   - Score < 88 → top 5 candidates kept for manual review (14 games)
   - No results → not found on Steam (49 games — F2P titles, delisted, collections, DLCs, or Epic-exclusives)
4. Saves `raw_data/steam_matches.json` in the format:
   ```json
   { "Subnautica": [[264710, "Subnautica"]], "For Honor": [[304390, "FOR HONOR™"], ...], ... }
   ```
   Single-element lists are confident matches; multi-element lists need review; empty lists are not on Steam.

---

### Step 3 — Manual cleanup of `steam_matches_manual.json`

Copy `raw_data/steam_matches.json` to `raw_data/steam_matches_manual.json`, then edit manually:

- **Multiple candidates** (14 games): delete all but the correct `[appid, name]` pair.
- **No match** (49 games): for games that are Steam-exclusive collections, bundles, or have non-standard names, manually add the correct `[appid, name]` entry. Games that are Epic-exclusives or F2P-only can be left as `[]` and will be skipped.

`steam_matches_manual.json` is the ground-truth mapping file consumed by all downstream steps.

---

### Step 4 — Scrape SteamCharts (`scripts/steam_charts_data.ipynb`)

Scrapes monthly concurrent player data for every matched game.

**What it does:**
1. Reads `steam_matches_manual.json` and deduplicates by App ID (a game given away multiple times is only scraped once).
2. For each App ID, fetches `https://steamcharts.com/app/{appid}/chart-data.json`. The endpoint returns a mix of monthly pre-aggregated points (older history) and hourly points (recent ~6 months); both are bucketed and averaged to monthly granularity.
3. Caches each game's result to `raw_data/steamcharts/{appid}.json`:
   ```json
   {
     "appid": 264710,
     "steam_name": "Subnautica",
     "epic_name": "Subnautica",
     "months": [
       {"month": "2018-12", "avg_players": 4321.2, "peak_players": 8900},
       ...
     ]
   }
   ```
   Re-running skips already-cached App IDs, so the scrape is safe to interrupt and resume.
4. Assembles all cached files into a single long-format panel:

**Output: `raw_data/steamcharts/all_monthly.csv`**

| Column | Description |
|--------|-------------|
| `appid` | Steam App ID |
| `steam_name` | Game name on Steam |
| `epic_name` | Game name as listed in the Epic giveaway data |
| `month` | Month (YYYY-MM-01 format) |
| `avg_players` | Average concurrent players that month |
| `peak_players` | Peak concurrent players that month |
| `first_giveaway_date` | Date of the first Epic free giveaway for this game |

> **Bot detection note:** SteamCharts uses Cloudflare. The scraper uses a full Chrome 120 User-Agent, matching `Sec-Fetch-*` headers, and warms up a persistent session to establish cookies before scraping. Rate limit is 2 seconds per request.

---

### Step 5 — Empirical analysis (`scripts/empirical_tests.ipynb`)

Run difference-in-differences or event-study regressions on `all_monthly.csv` using the `first_giveaway_date` column as the treatment date. *(In progress.)*

---

## Repository Layout

```
raw_data/
  epic games data.csv          # source: Kaggle download
  steam_search_cache.json      # cached Steam Store Search API responses
  steam_matches.json           # auto-generated appid mapping (Step 2 output)
  steam_matches_manual.json    # manually corrected mapping (Step 3 output)
  steamcharts/
    {appid}.json               # per-game player count cache (Step 4)
    all_monthly.csv            # final panel dataset (Step 4 output)

scripts/
  game_id_disambiguation.ipynb # Step 2
  steam_charts_data.ipynb      # Step 4
  empirical_tests.ipynb        # Step 5

.env.local                     # STEAM_API_KEY=... (not committed)
```

---

## Requirements

```
pandas
requests
python-dotenv
rapidfuzz
```

# Spotify_Playlist_Analysis(PYTHON+SQL(SUPABASE)+POWER-BI)

A complete end-to-end data analytics project: a raw Spotify chart CSV is cleaned, modeled, and
turned into a 6-page interactive Power BI report, backed by a live Postgres database on Supabase.

**Project name:** `Spotify_Playlist_Analysis(PYTHON+SQL(SUPABASE)+POWER-BI)`

---

## 📌 Project Overview

**Pipeline:**

```
Raw CSV → Python/Jupyter (clean & explore) → Supabase (Postgres) →
Star Schema Data Model → 46 DAX Measures → 6-Page Power BI Report
```

The project follows an **ELT approach** (Extract → Load → Transform) rather than cleaning data
before upload. The raw file is loaded into Supabase untouched first, so there is always an
original version to fall back on if a cleaning decision later needs revisiting.

---

## 📂 1. The Raw Dataset

`Spotify_Top_50_World.csv` — a daily snapshot of Spotify's Top 50 (World) chart.

**Grain:** one row = one song, on one chart day. A song charting for 100 days contributes
100 rows — with song-level fields (artist, duration, album info) repeating identically on
every one of those rows.

**Key columns:** `date`, `position` (1–50), `song`, `artist`, `popularity` (0–100),
`duration_ms`, `album_type`, `total_tracks`, `release_date`, `is_explicit`, `album_cover_url`.

### Full Column Reference

| Column | Type | What it holds |
|---|---|---|
| `date` | date | The calendar day this chart snapshot was taken |
| `position` | integer | The song's rank that day — expected range 1 to 50 |
| `song` | text | The track title, as published |
| `artist` | text | The performing artist or artist combination |
| `popularity` | integer | Spotify's own popularity score for the track (0–100 scale) |
| `duration_ms` | integer | Track length, in milliseconds |
| `album_type` | text | Whether the track came from a "single" or a full "album" |
| `total_tracks` | integer | How many tracks are on the parent album or single |
| `release_date` | text | The track's original release date, as published |
| `is_explicit` | boolean | Whether the track is tagged as explicit content |
| `album_cover_url` | text | A link to the track's cover art image |

**Data-quality checks applied:** duplication, denormalization risk, text/encoding corruption,
out-of-range values, unsafe identifiers (song titles alone aren't unique — a `song_key` of
song + artist is used instead), and inconsistent date formats.

---

## 🗄️ 2. Data Pipeline (Jupyter Notebook)

`Spotify_Project_EDA.ipynb` handles:

1. Load the raw CSV, untouched
2. Connect to Supabase via a `credentials.json` file (5 keys: host, port, db name, user, password)
3. Upload the raw data as-is to Supabase (safe original copy)
4. Explore the data for real-world issues (nulls, duplicates, corrupted characters, invalid dates)
5. Apply documented fixes (e.g. cleaning `release_date`, fixing corrupted artist names)
6. Reshape the cleaned data into a normalized star schema and upload the final tables

---

## 🌟 3. Data Model (Star Schema)

Three related tables, live in Supabase and imported into Power BI:

| Table | Purpose |
|---|---|
| **dim_songs** | One row per unique song+artist (`song_key`), with duration, album type, explicit flag, release date |
| **dim_dates** | A complete calendar table (marked as the Power BI Date Table), with year, month, weekday info |
| **fact_chart_entries** | One row per chart-day entry: position, popularity, date, song_key |

**Relationships:**
- `dim_songs` ↔ `fact_chart_entries` on `song_key` (1-to-many)
- `dim_dates` ↔ `fact_chart_entries` on `date` (1-to-many)

---

## 🧮 4. DAX Measures (46 total)

Tables referenced: `dim_songs`, `dim_dates`, `fact_chart_entries`

### Category 1 — Overall Dataset Stats

**1. Total Songs** — total chart entries (rows) in context; counts chart-entry days, not distinct songs
```
Total Songs = COUNTROWS(fact_chart_entries)
```
**2. Distinct Songs** — distinct song titles only, ignoring artist
```
Distinct Songs = DISTINCTCOUNT(fact_chart_entries[song])
```
**3. Distinct Artists** — count of unique artists in context
```
Distinct Artists = DISTINCTCOUNT(fact_chart_entries[artist])
```
**4. Avg Popularity** — average popularity score across chart entries
```
Avg Popularity = AVERAGE(fact_chart_entries[popularity])
```
**5. Max Popularity** — highest popularity score in context
```
Max Popularity = MAX(fact_chart_entries[popularity])
```
**6. Min Popularity** — lowest popularity score in context
```
Min Popularity = MIN(fact_chart_entries[popularity])
```

### Category 2 — Duration

**7. Avg Duration Minutes** — average song length in minutes (song-level attribute)
```
Avg Duration Minutes = AVERAGE(dim_songs[duration_ms]) / 60000
```
**8. Max Duration Minutes**
```
Max Duration Minutes = MAX(dim_songs[duration_ms]) / 60000
```
**9. Min Duration Minutes**
```
Min Duration Minutes = MIN(dim_songs[duration_ms]) / 60000
```

### Category 3 — Explicit Content

**10. Explicit Songs** — chart entries where the song is tagged explicit
```
Explicit Songs = CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[is_explicit] = TRUE())
```
**11. Non-Explicit Songs**
```
Non-Explicit Songs = CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[is_explicit] = FALSE())
```
**12. Pct Explicit Songs** — share of entries that are explicit (0–1)
```
Pct Explicit Songs = DIVIDE([Explicit Songs], [Total Songs], 0)
```
**13. Avg Popularity Explicit**
```
Avg Popularity Explicit = CALCULATE(AVERAGE(fact_chart_entries[popularity]), dim_songs[is_explicit] = TRUE())
```
**14. Avg Popularity NonExplicit**
```
Avg Popularity NonExplicit = CALCULATE(AVERAGE(fact_chart_entries[popularity]), dim_songs[is_explicit] = FALSE())
```

### Category 4 — Chart Position

**15. Avg Position** — average chart position (1–50); lower is better
```
Avg Position = AVERAGE(fact_chart_entries[position])
```
**16. Position 1 Songs** — chart-entry days at #1
```
Position 1 Songs = CALCULATE(COUNTROWS(fact_chart_entries), fact_chart_entries[position] = 1)
```
**17. Position 1 Artists** — distinct artists who ever held #1
```
Position 1 Artists = CALCULATE(DISTINCTCOUNT(fact_chart_entries[artist]), fact_chart_entries[position] = 1)
```

### Category 5 — Album / Release Format

**18. Avg Tracks per Album**
```
Avg Tracks per Album = AVERAGE(dim_songs[total_tracks])
```
**19. Album Type Count** — distinct album_type values (data-quality check, expected = 2)
```
Album Type Count = DISTINCTCOUNT(dim_songs[album_type])
```
**20. Singles Count**
```
Singles Count = CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[album_type] = "single")
```
**21. Albums Count**
```
Albums Count = CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[album_type] = "album")
```
**22. Pct Singles**
```
Pct Singles = DIVIDE([Singles Count], [Total Songs])
```

### Category 6 — Artist-Scoped

**23. Songs per Artist** — identical formula to Total Songs, named for clarity in Artist context
```
Songs per Artist = COUNTROWS(fact_chart_entries)
```
**24. Distinct Songs per Artist** — uses the safe `song_key` identifier
```
Distinct Songs per Artist = DISTINCTCOUNT(fact_chart_entries[song_key])
```
**25. Avg Popularity per Artist**
```
Avg Popularity per Artist = AVERAGE(fact_chart_entries[popularity])
```
**26. Position1 Hits per Artist**
```
Position1 Hits per Artist = CALCULATE(COUNTROWS(fact_chart_entries), fact_chart_entries[position] = 1)
```
**27. Artist Rank by Chart Entries** — RANKX demo, ranks every artist by total entries
```
Artist Rank by Chart Entries = RANKX(ALL(dim_songs[artist]), [Songs per Artist], , DESC)
```
**28. Top 10 Artist Entries** — TOPN demo, combined entries of the top 10 artists
```
Top 10 Artist Entries = CALCULATE([Total Songs], TOPN(10, ALL(dim_songs[artist]), [Songs per Artist], DESC))
```
**29. Pct of All Entries by Artist** — ALL() demo, removes every dim_songs filter at once
```
Pct of All Entries by Artist = DIVIDE([Songs per Artist], CALCULATE([Total Songs], ALL(dim_songs)))
```

### Category 7 — Time-Scoped

**30. Songs per Year**
```
Songs per Year = COUNTROWS(fact_chart_entries)
```
**31. Avg Popularity per Year**
```
Avg Popularity per Year = AVERAGE(fact_chart_entries[popularity])
```
**32. Avg Duration per Year**
```
Avg Duration per Year = AVERAGE(dim_songs[duration_ms]) / 60000
```
**33. Pct Explicit per Year**
```
Pct Explicit per Year = DIVIDE(CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[is_explicit] = TRUE()), [Songs per Year], 0)
```
**34. Pct Explicit per Month**
```
Pct Explicit per Month = DIVIDE(CALCULATE(COUNTROWS(fact_chart_entries), dim_songs[is_explicit] = TRUE()), COUNTROWS(fact_chart_entries))
```
**35. Weekend Entries Count** — FILTER() demo; weekend = weekday_num {6, 7}
```
Weekend Entries Count = CALCULATE(COUNTROWS(fact_chart_entries), FILTER(dim_dates, dim_dates[weekday_num] IN {6, 7}))
```

### Category 8 — Song Lifecycle

**36. Days Charted** — identical formula to Total Songs, named for clarity in Song context
```
Days Charted = COUNTROWS(fact_chart_entries)
```
**37. Days to No1** — returns BLANK() for songs that never hit #1
```
Days to No1 =
    VAR ReleaseDate = MIN(dim_songs[release_date_cleaned])
    VAR First1Date = CALCULATE(MIN(fact_chart_entries[date]), fact_chart_entries[position] = 1)
    RETURN IF(ISBLANK(First1Date), BLANK(), DATEDIFF(ReleaseDate, First1Date, DAY))
```
**38. Peak Position** — best (lowest) position a song ever reached
```
Peak Position = MIN(fact_chart_entries[position])
```
**39. First Chart Date**
```
First Chart Date = MIN(fact_chart_entries[date])
```
**40. Last Chart Date**
```
Last Chart Date = MAX(fact_chart_entries[date])
```
**41. Avg Days Since Release at Charting** — reads the `days_since_release` calculated column
```
Avg Days Since Release at Charting = AVERAGE(fact_chart_entries[days_since_release])
```
**42. New Entries Count** — reads the `is_new_entry` calculated column
```
New Entries Count = CALCULATE(COUNTROWS(fact_chart_entries), fact_chart_entries[is_new_entry] = TRUE())
```
**43. Distinct Song-Artist Combos** — the accurate distinct-song count, using `song_key`
```
Distinct Song-Artist Combos = DISTINCTCOUNT(fact_chart_entries[song_key])
```

### Category 9 — Position Volatility

**44. Position Range** — spread between a song's best and worst position
```
Position Range = MAX(fact_chart_entries[position]) - MIN(fact_chart_entries[position])
```
**45. Position StdDev** — population standard deviation of a song's daily position
```
Position StdDev = STDEV.P(fact_chart_entries[position])
```

### Category 10 — General

**46. Popularity Std Dev**
```
Popularity Std Dev = STDEV.P(fact_chart_entries[popularity])
```

---

## 📊 5. The Power BI Report (6 Pages) — Final Build

Two background designs are reused across all pages — **BG-1** (dark cover screen) and
**BG-2** (a tile grid dashboard) — with invisible nav buttons wired to Page Navigation
actions. Below is what each finished page actually contains.

### Page 0 — Cover
Spotify logo + wordmark, dark album-art collage on the right, and 4 nav buttons
(Executive Overview, Artist Deep Dive, Song Lifecycle & Trends, Content & Release-Format
Trends, Seasonality & Chart Behavior). No data visuals — pure title screen.

### Page 1 — Executive Overview
All Card visuals, five-second-glance dashboard:
- **Pct Explicit Songs** (58.82%), **Total Songs** hero card (28K)
- Distinct Songs (794), Distinct Artists (342), Songs per Artist (28K), Position 1 Artists (31), Position Range (49)
- Avg Popularity (89.62), Position 1 Songs (556)
- Max Popularity (100) / Min Popularity (0)
- Explicit Songs (16K), Avg Duration Minutes (3.40)
- Avg Position (25.50)
- Max Duration Minutes (7.62) / Min Duration Minutes (0.69)

### Page 2 — Artist Deep Dive
- **Artist table** — Artist, Distinct Songs per Artist, Songs per Artist, Avg Position (sortable, scrollable)
- **Top 10 Artist Entries by Artist** — donut chart, legend-labelled by artist
- **Songs per Artist by Year and Artist** — multi-line chart, top artists compared across 2023–2024

### Page 3 — Song Lifecycle & Trends
- **Song/Artist table** — scrollable list of every song and its artist
- Avg / Max / Min Duration Minutes cards (3.40 / 7.62 / 0.69)
- **Days Charted by Song** — horizontal bar chart, longest-running songs at the top
- **Days to No.1, Days Charted and Avg Popularity by song_key** — scatter/bubble chart

### Page 4 — Content & Release-Format Trends
- Explicit Songs (16K) / Non-Explicit Songs (21K) cards
- **Avg Duration Minutes and Avg Popularity by is_explicit** — scatter chart (True vs False)
- Avg Tracks per Album (12.19), Album Type Count (3)
- Avg Popularity Explicit (89.21) / Avg Popularity NonExplicit (89.74)
- **Singles Count and Albums Count by year_month** — line chart
- Pct Singles (0.72), Popularity Std Dev (9.97)

### Page 5 — Seasonality & Chart Behavior
- Weekend Entries Count (8K), Avg Duration per Year (3.40)
- **Songs per Year by Year** — area chart, with year slicer buttons (2023 / 2024)
- **Avg Position and Count of weekday_num by weekday_name** — column chart across all 7 days
- **Pct Explicit per Month by year_month** — line chart, declining trend over time

Unused tile slots on each page were deleted rather than left empty, and every visual was
verified to return real numbers (no blanks/errors) before the report was considered complete.

---

## 🚀 Step-by-Step: How to Build This Project (Start to End)

### Step 1 — Set Up Your Project Folder
- Create one folder for everything: CSV, notebook, `credentials.json`
- Install required Python libraries:
  ```
  pip install pandas sqlalchemy psycopg2-binary
  ```

### Step 2 — Create a Supabase Project
1. Sign up / log in at supabase.com → **New Project**
2. Set a database password and **save it immediately**
3. Once provisioned, click **Connect** → choose the **Session pooler** connection string
4. Note down: Host, Port (5432), Database name (`postgres`), User

### Step 3 — Build `credentials.json`
Create a JSON file with exactly these 5 keys, saved in the project folder:
```json
{
  "db_host": "...",
  "db_port": 5432,
  "db_name": "postgres",
  "db_user": "...",
  "db_password": "..."
}
```
⚠️ Never share or commit this file — it grants direct database access.

### Step 4 — Run the Notebook (`Spotify_Project_EDA.ipynb`)
Run cells top to bottom, in order (use *Restart Kernel and Run All* if anything errors):
1. Load the raw CSV (enter its file path when prompted)
2. Connect to Supabase (enter the `credentials.json` path when prompted)
3. Upload the raw data as-is to Supabase (safe original copy)
4. Explore the data — check nulls, duplicates, encoding issues, out-of-range values
5. Apply documented fixes (clean `release_date`, fix corrupted artist names, etc.)
6. Reshape into the 3-table star schema (`dim_songs`, `dim_dates`, `fact_chart_entries`)
7. Upload the final clean tables back to Supabase

### Step 5 — Connect Power BI to Supabase
1. Power BI Desktop → **Get Data** → PostgreSQL database
2. Enter the Supabase host/port/database, sign in with the pooler user/password
3. Import all 3 tables: `dim_songs`, `dim_dates`, `fact_chart_entries`

### Step 6 — Build the Data Model
- Create relationships: `dim_songs` ↔ `fact_chart_entries` on `song_key`,
  `dim_dates` ↔ `fact_chart_entries` on `date` (both 1-to-many)
- Mark `dim_dates` as the **Date Table**

### Step 7 — Create All 46 DAX Measures
- Type each measure from `Measures_Details.txt` into the Fields pane
- Spot-check a few on a blank card visual to confirm real numbers come back (no blanks/errors)

### Step 8 — Build the 6 Report Pages
- Import the BG-1 (cover) and BG-2 (grid) backgrounds as page backgrounds
- Place each visual into its slot, using the real measures (see Section 5 above for exactly
  what goes on each page)
- Delete any unused tile slots rather than leaving them empty
- Add 4 invisible nav buttons (transparent blank buttons over the background's pills, with
  **Page Navigation** actions) on every page that uses BG-2

### Step 9 — Final Review
- Confirm all 6 pages exist and are reachable
- Confirm every visual shows real data — no blanks or "value cannot be determined" errors
- Polish: consistent number formatting, tooltips, and visual titles

---

## 🔗 Project Links

- **Supabase Dashboard:** https://supabase.com/dashboard/project/hivpjoiseniijfrqsjrs

---

## 🔐 Security Note

The project requires a `credentials.json` file with live Supabase database credentials
(host, port, db name, user, password). **This file must never be committed to version
control, pasted into chats, or shared outside your own machine** — anyone with it has
direct access to the database.

---

## ✅ Final Checklist

- [ ] Star schema built and relationships confirmed (1-to-many, not many-to-many)
- [ ] `dim_dates` marked as Date Table
- [ ] All 46 measures created and spot-checked
- [ ] All 6 report pages assembled with correct visuals in each slot
- [ ] Unused slots deleted, not left empty
- [ ] 4 invisible nav buttons working on every BG-2 page
- [ ] Every visual shows real numbers — no blanks or errors

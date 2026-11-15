# Crime Pattern Analysis and Geographic Hotspot Detection Using DBSCAN

MCA final-year project: a Streamlit web application that analyses historical crime records, detects
geographic hotspots with DBSCAN (haversine distance), finds unusual periods with Isolation Forest, and
produces CSV / PNG / PDF reports. Data is stored in PostgreSQL.

| | |
| --- | --- |
| Language | Python 3.11+ |
| UI | Streamlit (8 pages, login required) |
| Data | Pandas, NumPy |
| Charts and maps | Plotly, Folium + streamlit-folium, Matplotlib/Seaborn (PDF images) |
| Machine learning | scikit-learn: DBSCAN, Isolation Forest; SciPy: Mann-Kendall, chi-square, t-test |
| Database | PostgreSQL 14+ via SQLAlchemy + psycopg2 |
| Security | bcrypt password hashing, parameterised queries, roles, session timeout |
| Tests | pytest (81 tests) + Streamlit AppTest |

---

## 1. Datasets

| File | Type | What it supports |
| --- | --- | --- |
| `data/raw/crime_dataset_india.csv` | 40,160 incident records, 29 Indian cities, Jan 2020 - Jul 2024 | Pages 1-6: overview, hotspots, temporal, category, advanced analytics |
| `data/raw/crime_review_for_month_Feb2026.csv` | Karnataka State Police monthly crime review: 738 category rows x 4 periods | Page 7: Karnataka crime review |

**Important notes**

* The India incident file has **no latitude/longitude**. Each city is placed at its approximate centre from
  `data/reference/india_city_coordinates.csv`, so maps and DBSCAN work at **city level** (regional hotspots
  such as Delhi NCR or the Mumbai region), not street level. A dataset with real coordinates is clustered at
  incident level automatically.
* The India incident file is **near-uniform**: about 1,900 incidents in every one of 21 categories, and the
  occurrence column advances exactly one hour per record. That indicates **simulated data**, so the app loads
  it with a "synthetic" flag and every page shows a warning. Use it to demonstrate the methods; do not present
  its results as real crime statistics.
* `Date of Occurrence` is month-first (`01-13-2020`) while the other date columns are day-first. The app detects
  the format of each column automatically.
* The Karnataka file has 24 unlabelled trailing rows (excluded and logged), 3 stray values in an unnamed column
  (ignored and logged) and 4 rows where year-to-date is not January + February (kept and flagged).

---

## 2. Setup (Windows, macOS or Linux)

### 2.1 Install Python and PostgreSQL

* Python 3.11 or newer: <https://www.python.org/downloads/> (Windows: tick "Add python.exe to PATH").
* PostgreSQL 14 or newer: <https://www.postgresql.org/download/>. Remember the password you set for the
  `postgres` user during installation.

### 2.2 Create the database

Open **SQL Shell (psql)** on Windows, or a terminal on macOS/Linux, log in as `postgres` and run:

```sql
CREATE USER crime_admin WITH PASSWORD 'crime_pass';
CREATE DATABASE crime_analytics OWNER crime_admin;
-- optional, only for running the database tests against PostgreSQL:
CREATE DATABASE crime_analytics_test OWNER crime_admin;
```

### 2.3 Install the project

```bash
cd crime_analytics
python -m venv .venv
# Windows:            .venv\Scripts\activate
# macOS / Linux:      source .venv/bin/activate
pip install -r requirements.txt
```

### 2.4 Configure

Copy `.env.example` to `.env` (Windows: `copy .env.example .env`) and edit it if your database user,
password or port differ. Change `ADMIN_PASSWORD` before showing the app to anyone.

```
DATABASE_URL=postgresql+psycopg2://crime_admin:crime_pass@localhost:5432/crime_analytics
ADMIN_USERNAME=admin
ADMIN_PASSWORD=ChangeMe@123
```

### 2.5 Create tables, admin user and load the data

```bash
python scripts/init_db.py
python scripts/load_sample_data.py
```

Expected output:

```
Database connection OK
Tables ready: users, dataset_uploads, aggregate_crime_stats, analysis_history, crime_records, hotspot_clusters
Created admin user 'admin'. Change the password in .env for real use.
Incidents: upload #1, 40,160 saved, 0 excluded.
  note: No coordinate columns: maps will use approximate city-centre coordinates ...
  note: No Severity column: severity will be derived from the crime type ...
Karnataka aggregates: upload #2, 738 saved, 24 excluded.
```

### 2.6 Run

```bash
streamlit run app.py
```

Open <http://localhost:8501> and sign in with the admin credentials from `.env`.
Maps need an internet connection (Leaflet library and OpenStreetMap tiles load from the web).

---

## 3. Using the app

| Page | What it shows |
| --- | --- |
| Overview | KPI cards, monthly trend, category bar, severity donut, location comparison, recent trend, automatic observations |
| Geographic Hotspots | DBSCAN parameters (eps km, min_samples), cluster map / marker map / heatmap, hotspot ranking, location ranking, k-distance plot, "how DBSCAN works" |
| Temporal Analysis | Hour, weekday, weekday x hour heatmap, calendar month, yearly (per-month average), weekend vs weekday with t-test, time of day, time series, year-over-year |
| Crime Categories | Distribution, severity by category, category x location heatmap, category trends with Mann-Kendall test, share by year, domain / weapon / age group / gender counts |
| Advanced Analytics | Automatic observations, DBSCAN summary and eps sensitivity, Isolation Forest on weekly/monthly counts, trend tests, Spearman correlation, period comparison with chi-square test, saved-analysis history |
| Karnataka Crime Review | Totals by head with MoM / YoY change, major heads, biggest increases and decreases, Isolation Forest on category movement, searchable table, cleaning log |
| Reports & Exports | Filtered CSV, hotspot summary CSV, chart PNGs, PDF analytical report, Karnataka CSV and PDF |
| Data Management (admin) | Upload CSV/Excel, preview, missing values, duplicates, personal-data warning, column mapping, validation and preprocessing log, save; stored datasets; users; system status |

The **sidebar filters** (date range, year, month, category, location, severity, police station, weekday)
apply to every chart, KPI, map and report on pages 1-6.

### Uploading your own dataset

Only **Crime_Type** and **Date** are required. Map the other standard fields when your file has them:
Crime_ID, Time, Latitude, Longitude, Location, Severity, Police_Station, Date_Reported, Crime_Code,
Crime_Domain, Weapon_Used, Victim_Age, Victim_Gender, Police_Deployed, Case_Closed, Date_Case_Closed.
Column names are matched automatically where possible; you can change every mapping before saving.

---

## 4. Tests

```bash
pytest                                   # 81 tests, uses SQLite for database tests
# also run the database tests on PostgreSQL:
# Windows (PowerShell): $env:TEST_DATABASE_URL="postgresql+psycopg2://crime_admin:crime_pass@localhost:5432/crime_analytics_test"; pytest
# macOS / Linux:        TEST_DATABASE_URL=postgresql+psycopg2://crime_admin:crime_pass@localhost:5432/crime_analytics_test pytest
```

The end-to-end tests in `tests/test_app_smoke.py` log in and open every page; they run when the database
from `.env` is reachable and loaded (section 2.5), and are skipped otherwise.

Test data in `tests/sample_data/` is **synthetic** (IDs start with `SYN-`): valid file, missing columns,
missing values, duplicates, invalid coordinates, empty file, wrong file type, city-level file.

---

## 5. Project structure

```
crime_analytics/
├── app.py                      # entry point: login gate + navigation
├── config.py                   # settings from .env
├── requirements.txt, .env.example, pytest.ini, README.md
├── .streamlit/config.toml      # theme, upload limit
├── data/raw/                   # the two datasets
├── data/reference/india_city_coordinates.csv
├── database/connection.py      # pooled engine, transactions
├── database/models.py          # tables + all queries (parameterised)
├── database/schema.sql         # generated from models.py
├── modules/
│   ├── authentication.py       # bcrypt, login, roles, timeout
│   ├── data_upload.py          # file checks, column mapping, profiling
│   ├── preprocessing.py        # cleaning, date detection, features, log
│   ├── karnataka_stats.py      # aggregate review parser + MoM/YoY
│   ├── ingestion.py            # save datasets in one transaction
│   ├── crime_analysis.py       # KPIs, temporal, trends, comparison, observations
│   ├── hotspot_detection.py    # DBSCAN (haversine) + cluster summary
│   ├── anomaly_detection.py    # Isolation Forest on aggregated counts
│   ├── visualization.py        # Plotly charts + Folium maps
│   ├── report_generation.py    # CSV, PNG, PDF
│   ├── filters.py, app_state.py, cached.py
├── views/                      # the 8 pages
├── scripts/init_db.py, scripts/load_sample_data.py
├── notebooks/exploratory_analysis.ipynb
├── tests/                      # pytest suite + synthetic sample data
└── assets/styles.css
```

Pages live in `views/` rather than `pages/` on purpose: Streamlit turns a `pages/` folder into links that
bypass `app.py`, which would skip the login check.

---

## 6. Common errors and fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `Database connection failed: OperationalError ... password authentication failed` | Wrong user/password in `.env` | Fix `DATABASE_URL`, or reset the password in psql: `ALTER USER crime_admin WITH PASSWORD 'crime_pass';` |
| `... Connection refused` / `could not connect to server` | PostgreSQL not running or other port | Start the PostgreSQL service (Windows: Services -> postgresql-x64-NN); check the port in `.env` |
| `database "crime_analytics" does not exist` | Step 2.2 skipped | Run the `CREATE DATABASE` commands |
| `ModuleNotFoundError: No module named 'streamlit'` | Virtual environment not active | Activate `.venv` and `pip install -r requirements.txt` |
| `pg_config executable not found` while installing | Building psycopg2 from source | `requirements.txt` uses `psycopg2-binary`; upgrade pip: `python -m pip install --upgrade pip` |
| Login says "Invalid username or password" | Admin not created or `.env` changed later | Run `python scripts/init_db.py`; it creates the admin once (changing `.env` later does not change an existing password) |
| Map area is blank | No internet (Leaflet/OSM tiles load online) or a firewall | Connect to the internet; charts and tables still work offline |
| "No incident dataset has been loaded yet" | Data not loaded | `python scripts/load_sample_data.py` or upload on Data Management |
| Upload: "Required field 'Date' is not mapped" | No date column chosen | Pick the date column in the mapping, or add one to the file |
| Upload: "... rows would be excluded" | Wrong date column or unusual date format | Check the preprocessing log; map the correct column |
| DBSCAN finds 0 clusters / everything is one cluster | eps too small / too large for the data | Use the k-distance plot; for city-level data try eps 40-120 km and min_samples 1,000-4,000 |

---

## 7. Honest-use notes

* Trend results describe **observed history** only. The app makes no forecasts.
* A DBSCAN hotspot shows where incidents were **recorded**, which also reflects reporting and policing patterns.
* Isolation Forest flags unusual **periods of aggregated counts**; it says nothing about any individual.
* Severity is **derived** from the crime category when the source has no severity column (rule in
  `modules/preprocessing.py`, shown in the preprocessing log).
* No names, addresses or other personal identifiers are stored. Victim age and gender, when present, are
  shown only as aggregate counts.
"# crime_analysis" 

# Getting Started with dbt on Snowflake — ITM327 Template

## Overview

This repository is your dbt project template for the ITM327 BYU-Idaho course.
You will use it to transform the raw news, stocks, and weather data your Airflow
pipelines loaded into Snowflake into a structured data warehouse and two data marts.

**Your data pipeline so far:**

```
Open-Meteo API  ──► Airflow (weather_to_snowflake_dag)  ──► SNOWBEARAIR_DB.RAW.LASTN_FI_WEATHER
Yahoo Finance   ──► Airflow (stocks_to_mongo → mongo_to_snowflake) ──► RAW.LASTN_FI_STOCKS
Finnhub API     ──► Airflow (news_to_sftp → sftp_to_snowflake)     ──► RAW.LASTN_FI_NEWS
```

**What dbt adds (the T in EtLT):**

```
RAW (Airflow writes here)
  └─► staging/    thin views, pass-through from RAW
        └─► warehouse/  dimensional model — fact + dimension tables  ← you build this
              └─► marts/  business questions answered on top of the DW
```

---

## Before You Start — Fork This Repository

1. In the top-right corner of this GitHub page, click **Fork**
2. Accept the defaults and click **Create fork** — this creates your personal copy under your GitHub account
3. Work only in your fork; you'll submit your finished project by opening a Pull Request back to the class repo

---

## Create a GitHub Personal Access Token (PAT)

Snowflake Workspaces connects to GitHub using a PAT instead of your password.

1. Go to [github.com](https://github.com) → click your profile icon → **Settings**
2. In the left sidebar, scroll to **Developer settings** → **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token (classic)**
4. Configure it:
   - **Token name**: something like `dbt-snowflake`
   - **Expiration**: set a date past the end of the semester
   - **Scopes**: check `repo` (full control of repositories)
5. Click **Generate token** — copy it immediately and save it somewhere safe (you cannot view it again)

![GitHub PAT Setup](images/pat.png)

> Never commit your token to version control. If you suspect it was exposed, delete it and generate a new one.

**Troubleshooting:**
- *Token not working* — verify it hasn't expired and has the `repo` scope selected
- *Permission denied* — double-check the scope; re-generate if needed
- *Token compromised* — delete it immediately in GitHub Settings and generate a new one

For more detail, see the [GitHub Personal Access Token documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

---

## Before You Start — Customize Your Table Names

Your raw Airflow tables follow a `LASTN_FI_` prefix naming convention (e.g., `SMITHJ_STOCKS`).
You must update three files before running dbt:

1. **`models/staging/__sources.yml`** — change the three `name:` fields:
   ```yaml
   - name: LASTN_FI_NEWS      # → e.g., SMITHJ_NEWS
   - name: LASTN_FI_STOCKS    # → e.g., SMITHJ_STOCKS
   - name: LASTN_FI_WEATHER   # → e.g., SMITHJ_WEATHER
   ```

2. **`models/staging/raw_news.sql`**, **`raw_stocks.sql`**, **`raw_weather.sql`** — update the
   `source()` call in each to match:
   ```sql
   -- Change LASTN_FI_NEWS → your actual table name
   select * from {{ source('snowbearair', 'SMITHJ_NEWS') }}
   ```

Everything else (`warehouse/`, `marts/`) references staging models via `{{ ref() }}`.
You will need to update the sql or python files so your warehouse and marts are created and updatable via dbt project scheduling.

---

## Following the dbt Demo Video

Video from the dbt Demo (reference as needed)
[Running dbt Projects On Snowflake](https://www.youtube.com/watch?v=w7C7OkmYPFs)

### When you get to the Git Repo Setup
- Use **your fork's URL** instead of the one in the video
- Select **Personal Access Token** and paste the PAT you created above
- Otherwise follow the video step by step

### dbt Commands to Run (in order)

1. **`dbt deps`** — install packages (none required by default, but good habit)
2. **`dbt compile`** — validate SQL and Jinja without running anything
3. **`dbt run`** — build all models in dependency order (can take several minutes)
4. View the compiled SQL in `target/compiled/`
5. Run compiled SQL directly in Snowflake to inspect results
6. **`dbt test`** — run data quality tests against your sources
7. **Deploy** the project to `SNOWBEARAIR_DB.RAW` as the metadata schema
   - Use the naming convention: `dbt_project_lastn_fi`
   - **Why RAW?** When deploying a dbt project in Snowflake, you must choose a schema to store
     dbt's internal metadata (run history, artifact tables). We use `RAW` here because it already
     exists and all students have access to it. This is *not* where your transformed models are
     written — those go to `DEV` or `PROD` as defined in `profiles.yml`. Think of this schema
     selection as "where dbt parks its paperwork," not "where dbt puts your data."
8. Understand how to orchestrate the project with Snowflake Tasks

---

## Project Structure

```
itm327_dbt_template/
├── dbt_project.yml          # Project name, profile, materialization config
├── profiles.yml             # Snowflake connection (SNOWBEARAIR_DB, dev/prod)
├── packages.yml             # External dbt packages (dbt_utils, commented out)
│
├── models/
│   ├── staging/             # Materialized as VIEWS — never edit raw data
│   │   ├── __sources.yml    # Source definitions + data quality tests
│   │   ├── raw_news.sql     # ← TODO: update table name to your prefix
│   │   ├── raw_stocks.sql   # ← TODO: update table name to your prefix
│   │   └── raw_weather.sql  # ← TODO: update table name to your prefix
│   │
│   ├── warehouse/           # Materialized as TABLES — your dimensional model
│   │   ├── fct_daily_stocks.sql      # Fact table: SYMBOL + TRADE_DATE grain
│   │   ├── fct_daily_weather.sql     # Fact table: CITY + DATE grain
│   │   ├── dim_symbols.sql           # Dimension: one row per stock ticker
│   │   └── dim_weather_cities.sql    # Dimension: one row per financial hub city
│   │
│   └── marts/               # Materialized as TABLES — business questions
│       ├── stock_news_daily.sql          # Mart 1: stock performance + news
│       ├── weather_market_conditions.sql  # Mart 2: weather vs. market performance
│       └── market_volatility_metrics.py  # Snowpark Python model: per-symbol volatility
│
├── macros/
│   └── generate_schema_name.sql  # Controls DEV/PROD schema naming
│
├── tests/
│   └── generic/
│       └── test_is_positive_amount.sql  # Custom test: value must be > 0
│
├── examples/
│   └── example_queries.sql   # Sample Snowflake queries to explore your marts
│
└── setup/
    └── snowflake_setup.sql   # Run once in Snowflake before your first dbt run
```

---

## Things to Understand from This Project

### The Three-Layer Architecture

| Layer | Folder | Materialization | Purpose |
|-------|--------|-----------------|---------|
| Raw | `SNOWBEARAIR_DB.RAW` | Tables (Airflow writes) | Source of truth — dbt never writes here |
| Staging | `models/staging/` | Views | Pass-through from RAW; where sources are declared and tested |
| Warehouse | `models/warehouse/` | Tables | Dimensional model — facts and dimensions |
| Marts | `models/marts/` | Tables | Business questions built on top of the warehouse |

### dbt Project Configuration

- **`dbt_project.yml`** — sets materialization per folder (`staging: view`, `warehouse: table`, `marts: table`)
- **`profiles.yml`** — defines `dev` and `prod` Snowflake targets; always develop against `dev`
- Why does the `dev`/`prod` separation matter? What goes wrong if you run directly against `prod`?

### Dimensional Modeling (warehouse/ layer)

The `warehouse/` folder is where you turn normalized staging data into a star schema:

- **Fact tables** (`fct_`) — record events at a specific grain (one row per symbol per trading day).
  They hold measurable metrics: OPEN, HIGH, LOW, CLOSE, VOLUME, PCT_CHANGE.
- **Dimension tables** (`dim_`) — describe the "who/what/where":
  - `dim_symbols` — one row per stock ticker with historical summary stats
  - `dim_weather_cities` — one row per financial hub city with climate norms
- Marts join facts + dimensions to answer specific business questions

> **Best practice — marts always build from the warehouse layer, never from staging.**
> If a mart needs data that isn't in a fact or dimension table yet, the right fix is to add it
> to the warehouse first, then reference it from the mart. Reaching back to a staging model
> (e.g., `ref('raw_weather')`) from a mart bypasses the cleaning and modeling you already did
> and forces each mart to re-implement the same logic independently.
>
> In this project, `weather_market_conditions` uses `{{ ref('fct_daily_weather') }}` (warehouse),
> not `{{ ref('raw_weather') }}` (staging). `stock_news_daily` uses `{{ ref('fct_daily_stocks') }}`
> and `{{ ref('dim_symbols') }}` — also warehouse. Every mart reference should be a `fct_` or
> `dim_` model, never a `raw_` model.

### Jinja Templating

- `{{ ref('model_name') }}` — references another dbt model; builds the dependency graph so dbt runs models in the right order
- `{{ source('snowbearair', 'TABLE_NAME') }}` — references a raw source table declared in `__sources.yml`
- Look at `tests/generic/test_is_positive_amount.sql` — a custom test written as a Jinja macro

### Data Testing

- **Built-in tests** (`not_null`, `unique`, `relationships`) — declared in `__sources.yml` as YAML, compiled to SQL by dbt
- **Custom generic test** (`is_positive_amount`) — returns failing rows if a value is ≤ 0; used on price and volume columns
- Tests run against **source data in RAW**, not the transformed models
- A passing test returns **0 rows**

### Python Models (Snowpark)

- `models/marts/market_volatility_metrics.py` is a dbt Python model
- It uses Snowpark DataFrames instead of SQL — useful for statistical functions like `stddev()` that are verbose in SQL
- Calculates per-symbol volatility (`RETURN_VOLATILITY`), `UP_DAYS`, `DOWN_DAYS`, and period high/low
- Note: Python models cannot be run directly in Snowflake Workspaces — they are executed by dbt via Snowpark

### Deployment & Orchestration

- `dev` target → `SNOWBEARAIR_DB.DEV` schema (for development)
- `prod` target → `SNOWBEARAIR_DB.PROD` schema (for scheduled production runs)
- Snowflake Tasks can schedule dbt project runs for orchestration

### Dev/Prod and CI/CD (Beyond the Demo)

The demo shows you running dbt manually in a `dev` environment, but in a real-world team
this workflow is automated through CI/CD:

- **Dev** — each developer has their own schema (e.g., `DEV_SMITHJ`) so they don't overwrite each other's work
- **Pull Requests** — opening a PR triggers automated CI checks: dbt compiles, runs tests, and validates before human review
- **Merge to main** — once approved, a CD pipeline runs `dbt run --target prod`, promoting changes to `PROD`
- **Prod** — the stable schema that dashboards and reports read from, run on a schedule via Snowflake Tasks

**How data mirrors across layers:**

| Schema | Written by | Contains |
|--------|-----------|----------|
| `SNOWBEARAIR_DB.RAW` | Airflow | Raw source tables — dbt reads here, never writes |
| `SNOWBEARAIR_DB.DEV` | dbt (dev) | Work-in-progress staging views + warehouse/mart tables |
| `SNOWBEARAIR_DB.PROD` | dbt (prod) | Production-quality, tested transformations for reporting |

This layered pattern ensures broken SQL or failed tests never reach production.

---

## Submitting Your Work

When you are ready to submit:

1. Make sure to change `itm327_dbt_template` to 'itm327_dbt_LASTFI`
2. Make sure all your changes are committed and pushed to **your fork** on GitHub
3. Open a Pull Request from your fork back to the class repo:
   - Go to your fork on GitHub
   - Click **Contribute** → **Open pull request**
   - Set the base repository to the class repo and base branch to `main`
   - Title your PR: `Submission — [Your Name]`
   - Click **Create pull request**

Your instructor will review your SQL and Python, and use the PR as your grading artifact.

---

## Resources

- https://www.snowflake.com/en/developers/guides/getting-started-with-dbt-projects-on-snowflake/
- https://www.snowflake.com/en/developers/guides/dbt-projects-on-snowflake/#0

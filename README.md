# Kroger Product Analysis

A Python pipeline that pulls product, pricing, and store-location data from the Kroger Public API, transforms it with dbt and DuckDB, and serves a Dash dashboard of fulfillment KPIs and price trends across Kroger stores.

![Python 3.12](https://img.shields.io/badge/python-3.12-blue)

## Overview

The pipeline gets an OAuth2 token from Kroger, finds stores near ten US zip codes, fetches product catalogs and prices in eight grocery categories, and writes the results to a local DuckDB database.

A dbt project turns the raw tables into a mart layer. The Dash dashboard reads the marts directly and renders five charts: price distributions, stock availability, promotional discounts, and fulfillment channel coverage, plus a store map.

Credentials and paths come from a single `.env` file. The source code has no hardcoded secrets.

## Architecture

### Pipeline DAG

```
get_access_token()
    └─► fetch_locations()          # 10 zip codes × up to 10 stores each
            └─► load_locations()   # upsert → raw.locations
    └─► fetch_products()           # 8 category terms × each discovered location
            └─► load_products_and_prices()
                    ├─► raw.products
                    └─► raw.product_prices
run_dbt_ops()                      # dbt deps → dbt run --full-refresh
```

Category search terms: `produce`, `dairy`, `bakery`, `meat`, `frozen`, `snacks`, `beverages`, `household`

Zip codes searched: `10001` (New York), `60601` (Chicago), `77001` (Houston), `90001` (Los Angeles), `30301` (Atlanta), `85001` (Phoenix), `98101` (Seattle), `78201` (San Antonio), `33101` (Miami), `02101` (Boston)

A full run makes about 165 API calls, far under the 10,000-call daily limit.

### dbt model DAG

```
Source (raw schema)
├── raw.locations
├── raw.products
└── raw.product_prices
        │
        ▼
Staging (views — main_staging schema)
├── stg_locations      ← filters nulls, passes through address + geo fields
├── stg_products       ← extracts primary_category from JSON categories array
└── stg_prices         ← computes discount_amount, discount_pct; casts dates
        │
        ▼
Dimensions (tables — main_marts schema)
├── dim_locations      ← location_id, name, chain, city, state, zip_code, lat/lon
└── dim_products       ← product_id, description, brand, category
        │
        ▼
Fact (table — main_marts schema)
└── fact_prices        ← joined price + product + location grain
        │
        ▼
Mart aggregates (tables — main_marts schema)
├── mart_location_sales         ← per-store: avg_price, total_products, physical/online counts
├── mart_category_distribution  ← per-category: product_count, avg_regular/promo price
└── mart_price_by_category      ← per-category per date: percentiles, min/max, avg discount
```

The dashboard queries `main_marts.fact_prices` and `main_marts.mart_location_sales`.

## Prerequisites

- Python 3.12.x. It has to be 3.12, because 3.13 and later break mashumaro, which dbt depends on.
- uv, the package manager ([install instructions](https://docs.astral.sh/uv/getting-started/installation/)).
- A Kroger Developer account. Register at [https://developer.kroger.com](https://developer.kroger.com) to get `KROGER_CLIENT_ID` and `KROGER_CLIENT_SECRET`. The free tier allows 10,000 API calls a day.

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Brucelee352/Kroger_API_Pipeline.git
cd Kroger_API_Pipeline
```

### 2. Install dependencies

```bash
uv sync
```

This creates `.venv` and installs the locked dependencies from `uv.lock`. Don't use `pip install` directly.

### 3. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and add your Kroger API credentials (see [Configuration](#configuration)).

### 4. Run the pipeline

```bash
uv run run-pipeline
```

Or, with the virtual environment activated:

```bash
run-pipeline
```

The pipeline authenticates, fetches data, loads DuckDB, and runs the dbt models. Logs go to stdout and `logs/pipeline.log`. A full run takes two to five minutes, depending on network latency.

### 5. Launch the dashboard

```bash
uv run dashboard
```

Or, with the virtual environment activated:

```bash
dashboard
```

Open [http://127.0.0.1:8050](http://127.0.0.1:8050). The pipeline must have run at least once so that `dbt_pipeline_demo/databases/kroger_pipeline.duckdb` exists.

## Configuration

Configuration is read from `.env` in the project root. Copy `.env.example` to `.env` and set each value.

| Variable | Required | Default | Description |
|---|---|---|---|
| `KROGER_CLIENT_ID` | Yes | — | OAuth2 client ID from https://developer.kroger.com |
| `KROGER_CLIENT_SECRET` | Yes | — | OAuth2 client secret from https://developer.kroger.com |
| `KROGER_BASE_URL` | No | `https://api.kroger.com` | Kroger API base URL |
| `DB_PATH` | No | `dbt_pipeline_demo/databases/kroger_pipeline.duckdb` | Path to the DuckDB file, relative to the project root |
| `LOG_LEVEL` | No | `INFO` | Python logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `HOME_URL` | No | `https://brucea-lee.com/` | Target of the dashboard's Home link |
| `DBT_DOCS_URL` | No | `https://brucea-lee.com/docs-kro/` | Target of the dashboard's dbt Docs link |

Kroger OAuth2 tokens expire after about 30 minutes. The pipeline gets one token at startup, which lasts the whole run.

## Dashboard

The dashboard is a single page with no filters. All five charts are built from the DuckDB mart tables on each page load. The header has links to the home page and the dbt docs, and an About button that opens a short description of the pipeline.

### Chart 1: Product price distribution (box plot)

Regular prices grouped by product category, sorted by median price, highest first. If the database has Q1 2026 prices, the chart uses only that quarter; otherwise it uses the full date range.

### Chart 2: Store fulfillment KPIs and average price by location (map)

A US map with two layers. The choropleth shades each state by the average catalog price of its stores. On top of it, each store is a bubble sized by the number of unique products in its catalog and colored by average price. Hovering over a bubble shows the store name, city, zip code, average price, product count, and the split between physical and online fulfillment counts.

### Chart 3: Stock availability by category (stacked bar)

Horizontal bars showing the share of items per category in each stock status (`In stock (high)`, `Low stock`, `Out of stock`), normalized to 100%. Categories are ordered by out-of-stock rate, highest first.

### Chart 4: Average promotional discount by category (bar)

Horizontal bars of the average promotional discount per category, for categories with at least five items on promotion, ordered from deepest discount to shallowest. Hovering shows the average regular price and the number of promotional items.

### Chart 5: Category value map (bubble scatter)

Average regular price on the X axis and average promotional discount on the Y axis. Each bubble is a category, sized by its number of distinct products. Median lines split the plot into quadrants, so you can see at a glance which categories are expensive and heavily discounted and which are cheap with small discounts.

## Project structure

```
.
├── .dbt/
│   └── profiles.yml               # dbt connection profile (DuckDB, relative path)
├── .env.example                   # environment variable template
├── dbt_pipeline_demo/
│   ├── databases/
│   │   └── kroger_pipeline.duckdb # generated on first pipeline run
│   ├── models/
│   │   ├── staging/
│   │   │   ├── stg_locations.sql
│   │   │   ├── stg_products.sql
│   │   │   └── stg_prices.sql
│   │   └── marts/
│   │       ├── dim_locations.sql
│   │       ├── dim_products.sql
│   │       ├── fact_prices.sql
│   │       ├── mart_category_distribution.sql
│   │       ├── mart_location_sales.sql
│   │       └── mart_price_by_category.sql
│   ├── dbt_project.yml
│   └── packages.yml
├── logs/
│   └── pipeline.log               # generated at runtime
├── portfolio_app/
│   ├── app.py                     # Dash dashboard entry point
│   └── scripts/
│       └── main_pipeline.py       # ETL entry point
├── pyproject.toml                 # package metadata and dependencies
└── uv.lock                        # locked dependency graph
```

## Troubleshooting

### Wrong Python version

The package requires `requires-python = ">= 3.12, < 3.13"`. If `uv sync` fails or dbt throws import errors, check your Python version:

```bash
python --version
```

Install Python 3.12 from [python.org](https://www.python.org/downloads/) or with `pyenv`, then run `uv sync` again.

### dbt cannot find the database

The dbt profile in `.dbt/profiles.yml` points to the relative path `databases/kroger_pipeline.duckdb`, which only resolves when dbt runs from inside `dbt_pipeline_demo/`. The pipeline changes into that directory with `os.chdir()` before calling dbt. If you run dbt by hand, run it from `dbt_pipeline_demo/`:

```bash
cd dbt_pipeline_demo
dbt run --target dev --full-refresh
cd ..
```

### Missing or invalid Kroger credentials

If the pipeline exits with `KROGER_CLIENT_ID and KROGER_CLIENT_SECRET must be set`, check that:

1. You have a registered application at [https://developer.kroger.com](https://developer.kroger.com).
2. `.env` exists in the project root and has the correct values.
3. The virtual environment is active. If it isn't, the pipeline prints activation instructions and exits.

### Dashboard shows empty charts

The dashboard needs a populated DuckDB file, so run `uv run run-pipeline` first. If one chart is still blank after a successful run, look in `logs/pipeline.log` for dbt errors. A failed model leaves its mart table missing.

### Rate limit errors from the Kroger API

The free tier allows 10,000 API calls a day and a full run uses about 165. If you get HTTP 429 responses, wait for the quota to reset at midnight UTC and run again.

## Author

Bruce Anthony Lee, [brucelee352@gmail.com](mailto:brucelee352@gmail.com)  
Portfolio: [kro.brucea-lee.com](https://kro.brucea-lee.com)  
LinkedIn: [linkedin.com/in/brucealee](https://www.linkedin.com/in/brucealee/)  
Repository: [github.com/Brucelee352/Kroger_API_Pipeline](https://github.com/Brucelee352/Kroger_API_Pipeline)

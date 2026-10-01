# 🏠 Airbnb ELT Pipeline — dbt + Snowflake + S3

## Overview

An end-to-end ELT (Extract, Load, Transform) data pipeline for Airbnb data, built using dbt and Snowflake. Raw CSV data (listings, hosts, bookings) is staged in AWS S3, loaded into Snowflake, and transformed through a bronze → silver → gold medallion architecture into analytics-ready tables.

## Architecture

Raw CSVs → AWS S3 → Snowflake (staging) → dbt transformations
↓
Bronze (raw) → Silver (cleaned) → Gold (analytics-ready)


**Tech stack:** Snowflake · dbt-core · AWS S3 · Jinja/SQL

## Data Model

### 🥉 Bronze — Raw ingestion
Incremental models that load raw source data with minimal transformation:
- `bronze_bookings`
- `bronze_hosts`
- `bronze_listings`

### 🥈 Silver — Cleaned & enriched
Cleaned, typed, and enriched models built on top of bronze:
- `silver_bookings` — computed `TOTAL_AMOUNT` from booking fees
- `silver_hosts` — includes a derived `RESPONSE_RATE_QUALITY` field
- `silver_listings` — includes a derived `PRICE_PER_NIGHT_TAG` (low/medium/high, via a custom macro)

### 🥇 Gold — Analytics-ready
- `obt.sql` — a denormalized One Big Table joining bookings, listings, and hosts, built dynamically using a Jinja-driven join pattern
- `fact.sql` — a fact table for dimensional modeling
- `ephemeral/` — intermediate CTE-style models (`bookings.sql`, `hosts.sql`, `listings.sql`) used as building blocks for the gold layer without materializing extra tables

### Snapshots (SCD Type 2)
Historical change tracking for slowly changing dimensions:
- `dim_bookings.yml`
- `dim_hosts.yml`
- `dim_listings.yml`

## Project Structure

aws_dbt_snowflake_project/
├── dbt_project.yml
├── models/
│ ├── sources/
│ │ └── sources.yml
│ ├── bronze/
│ ├── silver/
│ └── gold/
│ └── ephemeral/
├── macros/
│ ├── generate_schema_name.sql # custom per-layer schema naming
│ ├── multiply.sql # reusable numeric computation
│ ├── tag.sql # categorical bucketing (low/medium/high)
│ └── trimmer.sql # string cleanup utility
├── analyses/ # ad-hoc exploratory queries
├── snapshots/ # SCD Type 2 tracking
├── tests/ # data quality tests
└── seeds/


## Key Features

- **Incremental models** — bronze and silver layers only process new/changed rows, using an `is_incremental()` + `MAX(CREATED_AT)` pattern
- **Custom macros** — reusable Jinja/SQL logic, including a dynamic `tag()` macro for categorizing values and a `multiply()` macro for computed fields
- **Dynamic SQL generation** — the gold-layer `obt` model builds its SELECT/JOIN logic from a Jinja-driven config list rather than hardcoded SQL, making it easy to extend
- **Snapshots** — timestamp-based SCD Type 2 tracking to preserve historical state of bookings, hosts, and listings
- **Schema separation by layer** — bronze, silver, and gold each land in their own Snowflake schema

## Setup

### Prerequisites
- A Snowflake account
- An AWS account with an S3 bucket for raw file staging
- Python 3.12+ and a package manager (uv or pip)

### Steps

1. **Clone the repo**
```bash
   git clone https://github.com/pranavboddapati/airbnb-elt-pipeline.git
   cd airbnb-elt-pipeline
```

2. **Set up your Python environment**
```bash
   uv sync
   source .venv/bin/activate
```

3. **Configure your Snowflake connection**

   Create `~/.dbt/profiles.yml` (this file is intentionally excluded from the repo — never commit credentials):
```yaml
   aws_dbt_snowflake_project:
     outputs:
       dev:
         type: snowflake
         account: <your-account-identifier>
         user: <your-username>
         password: <your-password>
         role: ACCOUNTADMIN
         database: AIRBNB
         warehouse: COMPUTE_WH
         schema: dbt_schema
         threads: 1
     target: dev
```

4. **Stage raw data**

   Upload the source CSVs to an S3 bucket, then load them into Snowflake staging tables (via `COPY INTO` from an external stage pointing at S3).

5. **Validate the connection**
```bash
   dbt debug
```

## Usage

```bash
dbt run                      # build all models
dbt run --select bronze.*    # build one layer at a time
dbt test                     # run data quality tests
dbt snapshot                 # capture SCD snapshots
dbt build                    # models + tests + snapshots together
```

## Security Notes

- `profiles.yml` (which holds Snowflake credentials) is never committed — it lives only in `~/.dbt/`, outside the repo, and is listed in `.gitignore`.
- No credentials or secrets are hardcoded in any model, macro, or config file.

## What I Learned

Building this project deepened my understanding of incremental loading patterns, Jinja-driven dynamic SQL generation, medallion architecture design, and SCD Type 2 historical tracking — core patterns in modern analytics engineering.

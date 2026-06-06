# Architecture — Thailand DE Job Market Pipeline

An end-to-end Data Engineering pipeline that aggregates Data Engineer job postings from
4 platforms in Thailand, models the data through a Medallion architecture
(Bronze → Silver → Gold) in PostgreSQL with dbt, orchestrates the flow with Mage.ai, and
serves a Power BI dashboard over a Kimball star schema.

> The aggregation scripts and dbt models are not published in this repository — see
> [README.md](README.md) for the reason. This file documents the architecture only.

---

## Medallion Architecture (Bronze / Silver / Gold)

```
4 platforms in Thailand   (aggregate — Python + BeautifulSoup / Selenium)
        │
        ▼
   Raw JSON  →  data/raw/*.json
        │
        ▼
bronze.raw_jobs          (PostgreSQL, upsert on company_name + job_title)
        │
        ▼  dbt staging
silver.stg_jobs          (cleaned, filtered, salary/date parsed, locations normalised)
        │
        ▼  dbt marts
gold.fact_jobs            (star-schema fact — incl. type_of_work)
gold.dim_skills           (unique skills — true dimension)
gold.bridge_job_skills    (job ↔ skill M:N)
gold.dim_companies        (descriptive — aggregates computed as DAX measures)
gold.dim_platforms        (one row per source platform)
gold.bridge_job_platforms (job ↔ platform M:N)
        │
        ▼
Power BI Dashboard

Orchestration: Mage.ai  —  DAG:
  aggregate_platform_a ┐
  aggregate_platform_b ├─► load_bronze_platform_* ─► dbt_silver ─► dbt_gold
  aggregate_platform_c │
  aggregate_platform_d ┘
```

| Layer  | Schema   | Content                                        | Materialization  |
|--------|----------|------------------------------------------------|------------------|
| Bronze | `bronze` | Raw ingested data, no transformation           | PostgreSQL table |
| Silver | `silver` | Cleaned, filtered, salary/date/location parsed | dbt table        |
| Gold   | `gold`   | Aggregated, business-ready star schema         | dbt tables       |

---

## Tech Stack

- **Python 3.11** — data aggregation and ingestion (`requests` + `BeautifulSoup`; `Selenium` for JS-rendered platforms)
- **PostgreSQL 15** — data warehouse (bronze + silver + gold)
- **dbt-core** — Silver and Gold transformation layers, plus data-quality tests
- **Mage.ai** — pipeline orchestration (aggregate → bronze → silver → gold)
- **Docker + Docker Compose** — runs the whole stack locally (postgres + mage)
- **Power BI Desktop** — final dashboard, connected to the PostgreSQL gold layer

---

## Data Sources

| Source     | Status  | Notes |
|------------|---------|-------|
| Platform A | Done    | Static HTML, page-by-page |
| Platform B | Done    | Static HTML, Thai Buddhist calendar dates |
| Platform C | Done    | JS-rendered list page (Selenium) + static detail page |
| Platform D | Done    | JS-rendered list page (Selenium, Next.js) + static detail page |
| LinkedIn   | Skipped | Requirements behind a login wall — breaks skill extraction |

**Search keywords (aligned across all 4 platforms):**
`data engineer` · `etl developer` · `data pipeline` · `data warehouse` · `data platform`

All sources load into `bronze.raw_jobs` with a `platform_tags` column tracking which platforms
provided each job. All downstream Silver and Gold layers are source-agnostic.

---

## PostgreSQL Schema Design

### Bronze — `bronze.raw_jobs`

Raw data as ingested, no transformation.

```
id, job_title, company_name, location, salary_text, posted_date, job_url, full_job_url,
job_id, tags, requirements, type_of_work, scraped_date, source_file, source,
platform_tags, inserted_at
```

- Unique constraint on `(company_name, job_title)` drives the upsert logic.
- `platform_tags` accumulates platforms per job (appended only if not already present).
- `type_of_work` — Full-time / Part-time / Contract / Temporary, parsed from the detail page.
- `job_url` / `job_id` are **not** unique, so the gold layer keys jobs on `(company_name, job_title)`.

---

### Silver — `silver.stg_jobs` (dbt table)

**Responsibility: data quality only — no business logic.**

- Filters out non-DE titles (second pass after the Python aggregator filter).
- Parses `salary_text` → `salary_min` / `salary_max` (handles all Unicode dash variants,
  the `k` multiplier, and a length guard against concatenation overflow).
- Parses `posted_date` → `days_ago` (ISO dates and relative phrases).
- Normalises dirty `location` strings to a consistent `district, province` form.
- Passes through `platform_tags` and `type_of_work`.

Does **not** do skill extraction, keyword matching, or aggregation.

---

### Gold — Kimball Star Schema (dbt tables, DB-level constraints)

Conformed dimensions plus bridge tables for the two many-to-many relationships
(jobs ↔ skills, jobs ↔ platforms). Aggregates are **not** stored — they are DAX measures in Power BI.

```
dim_companies          dim_skills              dim_platforms
 PK company_id          PK skill_id             PK platform_id
      │                      │                       │
      ▼                      ▼                       ▼
  fact_jobs ◄── bridge_job_skills      bridge_job_platforms ──► (job ↔ platform)
  PK job_id        FK job_id, skill_id     FK job_id, platform_id
  FK company_id
```

**DB-level constraints (enforced via dbt hooks):**
- Primary keys on every dimension, fact, and bridge table.
- `fact_jobs.company_id` → `dim_companies.company_id` (FK).
- `bridge_job_skills` → `fact_jobs` and `dim_skills` (FKs).
- `bridge_job_platforms` → `fact_jobs` and `dim_platforms` (FKs).
- `pre_hook`s drop child FKs before each parent rebuild; `post_hook`s re-add them.

#### `gold.fact_jobs`
```
job_id (PK), company_id (FK→dim_companies), job_title, location, type_of_work,
salary_min, salary_max, days_ago, platform_tags, job_url, scraped_at, updated_at
```
`job_id = md5(company_name || '|' || job_title)` — the bronze natural key.

#### `gold.dim_companies` — descriptive only
```
company_id (PK), company_name, locations, updated_at
```
`job_count` and `avg_salary` are filter-aware DAX measures, not stored columns.

#### `gold.dim_skills` — one row per unique skill
```
skill_id (PK = md5(skill_name + skill_type)), skill_name, skill_type
```
- `skill_type = 'known'` — matched by regex against ~75 DE tools/technologies.
- `skill_type = 'candidate'` — auto-extracted CamelCase/ALL_CAPS tokens in 2+ jobs.

#### `gold.bridge_job_skills` — job ↔ skill M:N
```
row_id (PK), job_id (FK→fact_jobs), skill_id (FK→dim_skills), found_in
```

#### `gold.dim_platforms` — one row per source platform
```
platform_id (PK), platform_name, display_name
```

#### `gold.bridge_job_platforms` — job ↔ platform M:N
```
row_id (PK), job_id (FK→fact_jobs), platform_id (FK→dim_platforms)
```

#### `models/marts/int_job_skills.sql` — ephemeral
Shared skill-extraction logic inlined into both `dim_skills` and `bridge_job_skills`.

---

## dbt Conventions

- Staging models prefixed `stg_`; mart models prefixed `fact_` / `dim_`.
- Every model has a `schema.yml` with column descriptions; PKs carry `not_null` + `unique` tests.
- `macros/generate_schema_name.sql` enforces exact schema names (`silver` / `gold`, no prefix).
- Silver and Gold are materialised as **tables**.
- Gold PK/FK constraints enforced at DB level via dbt hooks.
- **26 / 26 dbt data-quality tests pass.**

---

## Power BI Model

| From | To | Cardinality | Cross-filter |
|------|----|-------------|--------------|
| `fact_jobs[company_id]` | `dim_companies[company_id]` | Many-to-One | Single |
| `bridge_job_skills[job_id]` | `fact_jobs[job_id]` | Many-to-One | Both |
| `bridge_job_skills[skill_id]` | `dim_skills[skill_id]` | Many-to-One | Single |
| `bridge_job_platforms[job_id]` | `fact_jobs[job_id]` | Many-to-One | Both |
| `bridge_job_platforms[platform_id]` | `dim_platforms[platform_id]` | Many-to-One | Single |

---

## Port Configuration (local development)

- **Docker PostgreSQL:** `localhost:5434` (host) → `5432` (container)
- **Mage.ai UI:** `http://localhost:6789`
- **Database:** `thai_de_skills`, user `postgres`
- Inside containers, use `POSTGRES_HOST=postgres` (internal Docker network, port 5432).

---

## Educational Use Only

This project was built solely for personal skill development and portfolio demonstration.
It has no commercial intent whatsoever and is not affiliated with, sponsored by, or endorsed
by any job platform or third party. All data aggregated through this pipeline is publicly
accessible and is used exclusively for educational purposes. Platform names have been
anonymised in all public-facing materials out of respect for each platform's Terms of Service.

# Thailand Data Engineer Skills Tracker

An end-to-end **Data Engineering pipeline** that aggregates Data Engineer job postings from
**4 platforms in Thailand**, models the data through a **Medallion architecture**
(Bronze → Silver → Gold) in PostgreSQL with dbt, and visualises in-demand skills, salaries,
and hiring companies in **Power BI**.

> **Goal:** demonstrate real, production-style DE skills for a portfolio — data aggregation,
> ingestion, data quality, dbt modelling, a Kimball star schema, orchestration, and BI.

---

> ⚠️ **Note on source code**
> The aggregation and ingestion scripts are intentionally not published in this repository.
> This project aggregates publicly available job-listing data for **educational and portfolio
> purposes only**. Publishing automation code that interacts with third-party platforms
> may conflict with their Terms of Service, so only the architecture, data models, and
> visualisation are shared here.

---

## 🔴 Live Demo

**▶ Interactive project page:** **https://windgabrielx.github.io/thailand-de-skills-tracker/**

[![Live Demo](https://img.shields.io/badge/Live_Demo-Visit_Site-00d4ff?style=for-the-badge&logo=githubpages&logoColor=white)](https://windgabrielx.github.io/thailand-de-skills-tracker/)

---

## 📊 At a Glance

| Metric | Value |
|--------|-------|
| Job postings aggregated | **102** |
| Companies hiring | **90** |
| Skills tracked | **77+** |
| Source platforms | **4 platforms in Thailand** |
| dbt data-quality tests | **26 / 26 passing** |

> A **point-in-time snapshot** of the DE hiring market across all 4 platforms — not a sample.
> The pipeline is built to run on a schedule (e.g. an automated aggregation every morning),
> but this build runs as a single on-demand pass to showcase the full end-to-end flow.

---

## 🏗️ Architecture

```
4 platforms in Thailand   (aggregate — Python + BeautifulSoup / Selenium)
                 │
                 ▼
        Raw JSON  →  data/raw/*.json
                 │
                 ▼
┌──────────────────────────────────────────────┐
│  BRONZE   bronze.raw_jobs                    │  Raw, untouched. Upsert on
│  103 rows · platform_tags · type_of_work     │  (company_name, job_title)
└──────────────────────────────────────────────┘
                 │  dbt staging
                 ▼
┌──────────────────────────────────────────────┐
│  SILVER   silver.stg_jobs                    │  Data quality only — cleaned,
│  94 rows · salary parsed · days_ago · typed  │  filtered, type-cast
└──────────────────────────────────────────────┘
                 │  dbt marts
                 ▼
┌──────────────────────────────────────────────┐
│  GOLD — Kimball star schema (6 tables)       │
│  fact_jobs            (PK job_id, FK company)│
│  dim_companies   dim_skills   dim_platforms  │
│  bridge_job_skills   bridge_job_platforms    │  M:N bridges + DB-level PK/FK
└──────────────────────────────────────────────┘
                 │
                 ▼
        Power BI Dashboard  (interactive, DAX measures over gold)
```

**Orchestration:** Mage.ai runs the complete DAG:

```
aggregate_platform_a ┐
aggregate_platform_b ├─► load_bronze_platform_*  ─► dbt_silver ─► dbt_gold
aggregate_platform_c │
aggregate_platform_d ┘
```

| Layer  | Schema   | Responsibility                              | Materialization  |
|--------|----------|---------------------------------------------|------------------|
| Bronze | `bronze` | Raw ingested data, no transformation        | PostgreSQL table |
| Silver | `silver` | Cleaning, filtering, salary/date parsing    | dbt **table**    |
| Gold   | `gold`   | Star schema, business-ready                 | dbt tables       |

---

## 🧰 Tech Stack

| Tool | Purpose |
|------|---------|
| **Python 3.11** | Data aggregation + ingestion |
| **BeautifulSoup / Selenium** | Static + JS-rendered page parsing |
| **PostgreSQL 15** | Data warehouse (bronze / silver / gold) |
| **dbt-core** | Silver + Gold transformation, data-quality tests |
| **Mage.ai** | Pipeline orchestration UI |
| **Docker + Docker Compose** | Run the full stack locally |
| **Power BI Desktop** | Dashboard over the gold layer |

---

## ⭐ Gold — Kimball Star Schema

```
dim_companies      dim_skills          dim_platforms
     │                  │                    │
     ▼                  ▼                    ▼
  fact_jobs ◄─ bridge_job_skills   bridge_job_platforms ─► (jobs ↔ platforms)
  PK job_id     (jobs ↔ skills M:N)     (jobs ↔ platforms M:N)
  FK→company
```

- **`fact_jobs`** — one row per posting; `job_id = md5(company_name | job_title)`;
  includes `type_of_work`, salary range, `days_ago`.
- **`dim_skills`** — one row per unique skill. `skill_type = 'known'` (regex against ~75 DE
  tools/technologies) or `'candidate'` (auto-extracted CamelCase / ALL_CAPS in 2+ postings).
- **`dim_companies`** — descriptive only; aggregates are filter-aware **DAX measures** in
  Power BI, not stored columns.
- **Bridge tables** carry the two M:N relationships and a `found_in` attribute
  (job title vs requirements text).
- PKs and FKs enforced at **database level** via dbt `post_hook`s; `pre_hook`s drop child
  FKs before each parent rebuild.

For the full schema design see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 📁 What Is in This Repository

This repository publishes only the **architecture documentation and portfolio site**.
The aggregation scripts, dbt models, Mage pipeline blocks, and Docker configuration are
**not included** — see the note at the top for the reason.

```
thailand-de-skills-tracker/
├── README.md               this file
├── ARCHITECTURE.md         detailed schema + pipeline design
├── requirements.txt        Python dependencies (reference only)
└── docs/
    ├── index.html          GitHub Pages portfolio site
    └── assets/             dashboard screenshots + diagrams
```

---

## 🔗 GitHub Pages

```
https://windgabrielx.github.io/thailand-de-skills-tracker/
```

Enable once: **Settings → Pages → Source: `main` / `/docs` → Save**.

---

## 📄 Educational Use Only

This project was built solely for personal skill development and portfolio demonstration.
It has no commercial intent whatsoever and is not affiliated with, sponsored by, or endorsed
by any job platform or third party. All data aggregated through this pipeline is publicly
accessible and is used exclusively for educational purposes. Platform names have been
anonymised in all public-facing materials out of respect for each platform's Terms of Service.

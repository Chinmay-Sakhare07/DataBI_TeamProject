# 🎵 Chinook Data Engineering Pipeline
### *From Raw Transactions to a Star Schema — End to End on Azure Databricks*

> **DAMG7370 · Northeastern University · Spring 2026**  
> Shreya Darban · Darshan Patgar · Chinmay Sakhare

---

<div align="center">

```
Azure SQL  ──▶  RAW  ──▶  BRONZE  ──▶  SILVER  ──▶  GOLD
(Chinook)      Parquet    Delta       DQX-Validated   Star Schema
```

![Status](https://img.shields.io/badge/Pipeline-Operational-brightgreen)
![Layers](https://img.shields.io/badge/Layers-4%20(Medallion)-blue)
![Tables](https://img.shields.io/badge/Gold%20Tables-6-purple)
![Records](https://img.shields.io/badge/Records%20Validated-100%25-success)
![Platform](https://img.shields.io/badge/Platform-Azure%20Databricks-orange)

</div>

---

## 📖 Table of Contents

- [What We Built](#-what-we-built)
- [Architecture](#-architecture)
- [Azure Infrastructure](#️-azure-infrastructure)
- [Repository Structure](#-repository-structure)
- [Pipeline Walkthrough](#-pipeline-walkthrough)
  - [Raw Zone](#1️⃣-raw-zone)
  - [Bronze Layer](#2️⃣-bronze-layer)
  - [Silver Layer — DQX](#3️⃣-silver-layer--dqx-validation)
  - [Gold Layer — Dimensional Model](#4️⃣-gold-layer--dimensional-model)
- [Star Schema](#-star-schema)
- [Data Quality Results](#-data-quality-results)
- [Job Execution](#-job-execution)
- [Technologies](#-technologies)
- [Team](#-team)

---

## 🚀 What We Built

A **production-grade, metadata-driven data engineering pipeline** that ingests all 11 tables from the Chinook music store database, processes them through 4 Medallion Architecture layers, and delivers a fully validated **Star Schema dimensional model** — all orchestrated as a Databricks Job on Serverless compute.

| Metric | Value |
|--------|-------|
| Source tables ingested | 11 |
| Pipeline notebooks | 5 |
| DQX validation rules | 10 |
| Records validated | 6,454 |
| Failed records | 0 |
| Gold layer tables | 6 |
| Total job runtime | ~4 minutes |

---

## 🏛 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEDALLION ARCHITECTURE                        │
├──────────┬────────────┬──────────────┬──────────────────────────┤
│  SOURCE  │    RAW     │    BRONZE    │   SILVER   │    GOLD      │
│          │            │              │            │              │
│ Azure    │  Parquet   │   Delta      │   Delta    │   Delta      │
│ SQL      │  Volume    │   Tables     │  + DQX     │  Star Schema │
│ ChinookDB│            │              │            │              │
│          │ Immutable  │ Exact copy   │ Cleaned &  │ Dimensions + │
│ 11 tables│ snapshots  │ No transforms│ Validated  │ Facts        │
└──────────┴────────────┴──────────────┴────────────┴──────────────┘
         ▲            ▲              ▲            ▲             ▲
    Connection    Parquet        Overwrite     DQX Rules    MD5 Keys
    Manager       /YYYY/MM/DD    Mode          + Quarantine  SCD Type 2
```

---

## ☁️ Azure Infrastructure

All resources provisioned in **East US 2** under `rg-chinook-databricks`:

```
rg-chinook-databricks (Resource Group)
│
├── 🔷 dbw-chinook-team          Azure Databricks Service (Premium + Serverless)
├── 🗄️  sqldb-chinook-team        Azure SQL Server
├── 📦  ChinookDB                 SQL Database (11 Chinook tables)
└── 🔐  kv-chinook                Azure Key Vault (sqlserver-password secret)
```

**Databricks Environment:**

| Component | Value |
|-----------|-------|
| Connection Manager | `chinook_sql_conn` — no credentials in code |
| Secret Scope | `chinook-scope` → linked to Key Vault |
| Unity Catalog | `dbw_chinook_team` |
| Schemas | `raw_zone` · `chinook_bronze` · `chinook_silver` · `chinook_gold` |
| Compute | Serverless |

---

## 📁 Repository Structure

```
DataBI_TeamProject/
│
├── 00_setup_metadata.ipynb       # Creates pipeline control tables
├── 01_extract_to_raw.ipynb       # Azure SQL → Parquet Volume
├── 02_raw_to_bronze.ipynb        # Parquet → Bronze Delta tables
├── 03_bronze_to_silver.ipynb     # DQX validation + Silver transforms
├── 04_silver_to_gold.ipynb       # Dimensions + Facts → Gold
└── Readme.md
```

> All notebooks are parameterized via **Databricks Widgets** — no hardcoded values anywhere.  
> Version controlled on the `main` branch with contributions from all 3 team members.

---

## 🔄 Pipeline Walkthrough

### Metadata-Driven Design

Before any data moves, two control tables govern execution:

```
pipeline_metadata          ← 11 rows, one per source table
  ├── table_name
  ├── file_name
  ├── active_flag           ← toggle tables on/off without code changes
  ├── created_date
  └── modified_date

pipeline_execution_log     ← one row per table per run
  ├── table_name
  ├── execution_time
  ├── status
  ├── source_row_count
  ├── target_row_count
  ├── file_location
  └── created_date
```

---

### 1️⃣ Raw Zone

**Notebook:** `01_extract_to_raw`  
**Purpose:** Pull every active table from Azure SQL and write an immutable Parquet snapshot.

```
/Volumes/dbw_chinook_team/raw_zone/chinook/
└── {table}/
    └── {YYYY}/{MM}/{DD}/
        └── {table}.parquet       ← new file every run, never overwritten
```

| Table | Rows |
|-------|------|
| artist | 275 |
| album | 347 |
| track | 3,503 |
| genre | 25 |
| mediatype | 5 |
| customer | 59 |
| employee | 8 |
| invoice | 412 |
| invoiceline | 2,240 |
| playlist | 18 |
| playlisttrack | 8,715 |

---

### 2️⃣ Bronze Layer

**Notebook:** `02_raw_to_bronze`  
**Purpose:** Load all Parquet snapshots into Delta format with zero transformations.

```python
# Philosophy: Bronze = exact replica, maximum fidelity
df.write.format("delta").mode("overwrite").saveAsTable(f"{BRONZE}.{table_name}")
```

- ✅ 11 Delta tables in `chinook_bronze`
- ✅ Overwrite mode — daily refreshed snapshot
- ✅ Row counts validated against execution log

---

### 3️⃣ Silver Layer — DQX Validation

**Notebook:** `03_bronze_to_silver`  
**Purpose:** Enforce data quality rules, quarantine failures, apply cleaning transforms.

**Validation Rules:**

| Table | Rules |
|-------|-------|
| Customer | `CustomerId`, `Email`, `FirstName`, `LastName` not null |
| Invoice | `InvoiceId`, `CustomerId` not null · `Total > 0` |
| InvoiceLine | `Quantity > 0` · `UnitPrice > 0` · `TrackId` not null |
| Track | `TrackId`, `Name` not null · `Duration > 0` |

**Transforms Applied:**
```
TRIM    → all name fields
LOWER   → all email fields
COALESCE → null handling with safe defaults ('N/A', 'Unknown')
CAST    → data type standardization
```

**Failed records** → written to `chinook_silver.quarantine` (append mode, schema evolution enabled)

---

### 4️⃣ Gold Layer — Dimensional Model

**Notebook:** `04_silver_to_gold`  
**Purpose:** Build a Star Schema from clean Silver data using MD5 surrogate keys.

#### Dimensions

| Table | Rows | Highlights |
|-------|------|-----------|
| `dim_customer` | 59 | **SCD Type 2** — MD5 hash change detection, full history tracking |
| `dim_track` | 3,503 | 5-table JOIN: Track + Album + Artist + Genre + MediaType |
| `dim_date` | 10,950 | Generated 2000-01-01 → 2030-12-31 via Spark sequence |
| `dim_employee` | 8 | Self-referencing hierarchy via `reports_to_nk` |

#### Facts

| Table | Rows | Grain |
|-------|------|-------|
| `fact_sales` | 2,240 | One row per invoice line item |
| `fact_sales_customer_agg` | 59 | One row per customer — aggregated from `fact_sales` |

---

## ⭐ Star Schema

```
                        ┌─────────────┐
                        │  DIM_DATE   │
                        │  date_sk PK │
                        └──────┬──────┘
                               │ fk_date_sk
                               │
┌──────────────┐        ┌──────▼───────┐        ┌───────────────┐
│  DIM_TRACK   │        │  FACT_SALES  │        │ DIM_EMPLOYEE  │
│  track_sk PK │◄───────│              │───────►│ employee_sk PK│
└──────────────┘fk_track│ invoice_     │fk_empl └───────────────┘
                        │  line_id PK  │
                        │ customer_key │
                        │ track_key    │
                        │ date_sk      │
                        │ employee_key │
                        │ quantity     │
                        │ unit_price   │
                        │ line_total   │
                        └──────┬───────┘
                               │ fk_customer_sk
                               │
                        ┌──────▼──────────┐
                        │  DIM_CUSTOMER   │
                        │  customer_sk PK │
                        │  ← SCD Type 2  │
                        │  hash_value     │
                        │  is_current     │
                        │  effective_dates│
                        └─────────────────┘
```

#### SCD Type 2 Logic (dim_customer)

```
Incoming record
       │
       ▼
Compute MD5(all tracked columns)
       │
       ▼
Compare with stored hash_value WHERE is_current = TRUE
       │
   ┌───┴────────────────────┐
   │ MATCH                  │ MISMATCH
   ▼                        ▼
Skip (no write)     UPDATE existing record
Idempotent          effective_end_date = now()
                    is_current = FALSE
                           │
                           ▼
                    INSERT new record
                    effective_start_date = now()
                    is_current = TRUE
```

---

## ✅ Data Quality Results

**DQX Validation — 100% pass rate across all runs:**

| Table | Total Records | Passed | Failed | Status |
|-------|--------------|--------|--------|--------|
| Customer | 59 | 59 | 0 | ✅ |
| Invoice | 412 | 412 | 0 | ✅ |
| InvoiceLine | 2,240 | 2,240 | 0 | ✅ |
| Track | 3,503 | 3,503 | 0 | ✅ |

**Gold Layer FK Validation — zero orphan records:**

| Foreign Key | Null Count |
|-------------|-----------|
| `customer_key` | 0 ✅ |
| `track_key` | 0 ✅ |
| `date_sk` | 0 ✅ |
| `employee_key` | 0 ✅ |

---

## ⚡ Job Execution

The full pipeline runs as a single **Databricks Job** (`chinook_pipeline_job`) on Serverless compute, executing all 5 tasks in sequence:

```
setup_metadata  ──▶  extract_to_raw  ──▶  raw_to_bronze  ──▶  bronze_to_silver  ──▶  silver_to_gold
    43s                  31s                  41s                 1m 34s                  36s

                            Total runtime: ~4 minutes ✅
```

All tasks succeeded on first run with green checkmarks across the board.

---

## 🛠 Technologies

| Technology | Role |
|------------|------|
| **Azure Databricks** | Pipeline execution + Serverless compute + Job orchestration |
| **Unity Catalog** | Schema, table, and Volume management |
| **Delta Lake** | Storage format for Bronze, Silver, and Gold |
| **Databricks Volume** | Immutable Raw Parquet file storage |
| **Connection Manager** | Secure source connection — zero credentials in code |
| **Azure SQL Server** | Source database (Chinook) |
| **Azure Key Vault** | Secret management |
| **DQX** | Data quality profiling, validation, and quarantine |
| **PySpark** | All data transformation and processing |
| **GitHub** | Version control — `DataBI_TeamProject` |

---

## 👥 Team

| Name | GitHub | Role |
|------|--------|------|
| Shreya Darban | darbanshreya | Infrastructure + Raw/Bronze notebooks |
| Darshan Patgar | Darshannn09 | Silver DQX validation |
| Chinmay Sakhare | Chinmay-Sakhare07 | Gold dimensional model + Job orchestration |

---

<div align="center">

*Built with 🎵 data, ☁️ cloud, and a lot of ✅ green checkmarks.*  
**Northeastern University · DAMG7370 · Spring 2026**

</div>

# FinanceCore — Bank Transactions Data Pipeline

> **Sprint 2 · Brief 2** — ETL pipeline that loads cleaned bank transaction data into a PostgreSQL star-schema data warehouse and exposes KPI views for analytics.

---

## Overview

This project transforms a flat CSV of bank transactions into a structured **star schema** inside PostgreSQL. It covers the full pipeline:

1. Schema creation (dimension + fact tables, indexes)
2. Data loading via staging tables
3. Data quality checks
4. SQL analytics queries
5. KPI views

---

## Database Schema

The warehouse follows a classic star schema:

### Dimension Tables

| Table              | Key Columns                                                                   |
| ------------------ | ----------------------------------------------------------------------------- |
| `dim_clients`    | `client_id`,`score_credit_client`,`segment_client`,`categorie_risque` |
| `dim_produits`   | `produit_id`,`produit`                                                    |
| `dim_agences`    | `agence_id`,`agence`                                                      |
| `dim_categories` | `categorie_id`,`categorie`                                                |
| `dim_dates`      | `date_id`,`anne`,`trimestre`,`mois`,`jour_semaine`                  |

### Fact Table

**`fact_transactions`** — one row per bank transaction, referencing all dimension tables via foreign keys.

Key measures: `montant`, `montant_eur`, `taux_change_eur`, `crédits`, `débits`, `solde_net`, `is_anomaly_montant`, `is_conversion_error`.

### Indexes

Performance indexes are created on the most-queried foreign keys: `client_id`, `date_id`, `agence_id`.

---

## Project Structure

```
financecore_project/
├── data/
│   └── financecore_clean.csv   # Cleaned source data
└── main.ipynb                  # ETL pipeline notebook
```

---

## Prerequisites

* Python 3.8+
* PostgreSQL instance (local or remote)
* The following Python packages:

```bash
pip install pandas sqlalchemy psycopg2-binary python-dotenv
```

---

## Configuration

Create a `.env` file in the project root with your database credentials:

```env
DB_USER=your_user
DB_PASSWORD=your_password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_database
```

---

## How to Run

Open and run `main.ipynb` sequentially. The notebook is divided into the following sections:

### 1. Connection

Loads environment variables and creates a SQLAlchemy engine connected to PostgreSQL.

### 2. Create Tables

Creates all dimension tables and the fact table with proper foreign key constraints, using `CREATE TABLE IF NOT EXISTS` for idempotency.

### 3. Create Indexes

Adds indexes on `client_id`, `date_id`, and `agence_id` in `fact_transactions` to speed up analytical queries.

### 4. Data Loading

* Reads `financecore_clean.csv` into a pandas DataFrame.
* Extracts each dimension (clients, products, agencies, categories, dates) into a separate DataFrame.
* Loads each dimension via a staging table, then upserts into the target dimension table using `ON CONFLICT` logic.
* Merges surrogate keys back into the main DataFrame.
* Finally loads `fact_transactions`.

### 5. Check

Validates referential integrity by counting NULL foreign keys in `fact_transactions` (should all be 0).

### 6. SQL Analytics

Sample queries included:

* **Top 10 agency/product/month combinations** by transaction volume, with a global average for comparison.
* **Clients below average net balance** — identifies at-risk clients.
* **Client segment distribution** — percentage share per segment.
* **Full join** across all tables (quick sanity check / exploration).

### 7. Create Views

KPI views for dashboarding:

| View                       | Description                                         |
| -------------------------- | --------------------------------------------------- |
| `kpi_total_transactions` | Total number of transactions                        |
| `kpi_total_amount`       | Total transaction amounts in local currency and EUR |

---

## Data Quality Flags

Two boolean flags are carried through from the source data into `fact_transactions`:

* `is_anomaly_montant` — flags transactions with anomalous amounts.
* `is_conversion_error` — flags transactions with suspected EUR conversion errors.

---

## Notes

* All dimension inserts use `ON CONFLICT DO NOTHING` (or `DO UPDATE`) to make the pipeline safe to re-run.
* The `dim_dates` table is appended directly from the DataFrame; ensure deduplication is handled upstream or add a unique constraint as needed.
* Update the CSV path in the **Data Loading** section if your directory structure differs.

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
FINANCECORE_PROJECT/
├── .venv/                      # Virtual environment (not committed)
├── data/
│   └── financecore_clean.csv   # Cleaned source data
├── docs/
│   └── ERD.png                 # Entity-Relationship Diagram
├── notebooks/
│   └── main.ipynb              # ETL pipeline notebook
├── .env                        # Database credentials (not committed)
├── .gitignore
├── README.md
└── requirements.txt
```

---

## Prerequisites

* Python 3.8+
* PostgreSQL instance (local or remote)

Install all dependencies with:

```bash
pip install -r requirements.txt
```

Key packages (pinned versions from `requirements.txt`):

| Package             | Version |
| ------------------- | ------- |
| `pandas`          | 3.0.2   |
| `sqlalchemy`      | 2.0.49  |
| `psycopg2-binary` | 2.9.11  |
| `python-dotenv`   | 1.2.2   |
| `ipykernel`       | 7.2.0   |
| `numpy`           | 2.4.4   |

---

## Configuration

Create a `.env` file in the project root with your database credentials. Use the following template (matching the expected variable names):

```env
DB_HOST=host_db
DB_PORT=port_db
DB_NAME=name_db
DB_USER=user_db
DB_PASSWORD=password_db
```

> ⚠️ The `.env` file is listed in `.gitignore` and will **not** be committed to version control. Never hardcode credentials in the notebook.

---

## .gitignore

The following are excluded from version control:

```
.venv/
.env
```

---

## How to Run

Open and run `notebooks/main.ipynb` sequentially. The notebook is divided into the following sections:

### 1. Connection

Loads environment variables from `.env` and creates a SQLAlchemy engine connected to PostgreSQL.

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

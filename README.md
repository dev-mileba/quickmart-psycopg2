# QuickMart Data Warehouse ETL

A small ETL pipeline that transforms raw QuickMart retail CSV extracts into a
**star schema** and loads it into a PostgreSQL data warehouse. All logic lives in
[`main.ipynb`](main.ipynb) and uses `pandas` for transformation and `psycopg2`
for loading.

## Data model

Raw source files in [`data/`](data/) (tracked with Git LFS):

| File            | Rows    | Description                                    |
| --------------- | ------- | --------------------------------------------- |
| `customers.csv` | ~25,000 | Customer master (name, email, city, segment)  |
| `products.csv`  | ~1,000  | Product catalogue (name, category, brand, price) |
| `stores.csv`    | ~80     | Store master (name, city, region, type)       |
| `sales.csv`     | ~2,000,000 | Transaction line items                      |

These are loaded into the following warehouse tables:

**Dimensions**

- `dim_date` — one row per distinct sale date, with `date_key` (`YYYYMMDD`), day, month, year, quarter
- `dim_customer` — customers + surrogate `customer_key`
- `dim_products` — products + surrogate `product_key`
- `dim_stores` — stores + surrogate `store_key`
- `dim_sales` — raw sales rows + surrogate `sale_key` (staging / degenerate dimension)

**Fact**

- `fact_sales` — grain: one row per `sale_id`. Foreign keys to every dimension,
  plus `quantity`, `unit_price`, and derived `sale_amount` (`quantity * unit_price`).

## Requirements

- Python 3.13+
- A running PostgreSQL instance with a database named `quickmart`
- Git LFS (to pull the CSV files)

Python packages: `pandas`, `numpy`, `psycopg2-binary`, and Jupyter
(`ipykernel` / `notebook`).

```bash
pip install pandas numpy psycopg2-binary notebook
```

## Setup

1. Pull the LFS-tracked data files:

   ```bash
   git lfs install
   git lfs pull
   ```

2. Create the target database:

   ```bash
   createdb quickmart
   ```

3. The notebook connects with these defaults — adjust the `psycopg2.connect(...)`
   call in [`main.ipynb`](main.ipynb) to match your environment:

   | Setting  | Value       |
   | -------- | ----------- |
   | host     | `localhost` |
   | port     | `5432`      |
   | dbname   | `quickmart` |
   | user     | `postgres`  |
   | password | `postgres`  |

## Running

Open and run all cells in order:

```bash
jupyter notebook main.ipynb
```

The notebook will:

1. Read the four CSVs into DataFrames.
2. Build the date dimension and assign surrogate keys to each dimension.
3. Assemble `fact_sales` by merging dimension keys onto the sales rows and
   computing `sale_amount`.
4. Create the warehouse tables (`CREATE TABLE IF NOT EXISTS`).
5. Bulk-insert each table with `psycopg2.extras.execute_values`, committing per table.
6. Close the connection.

> Note: `numpy` int/float adapters are registered before the `fact_sales` load so
> NumPy scalar types insert cleanly.

## Notes

- Re-running the load cells against an already-populated database will raise
  primary key violations. Drop or truncate the tables first for a clean reload.


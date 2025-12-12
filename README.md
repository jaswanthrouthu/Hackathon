# Hackathon
# Insurance Policy ETL → DWH (Colab-ready)

**Short description**
This repo contains a hackathon-ready ETL pipeline for the Q2 Insurance Policy problem. It reads zipped CSV snapshots (4 region ZIPs), standardizes input, builds SCD Type-2 dimensions (customer & policy), creates surrogate keys (SKs), assembles a star-schema (4 dims + 1 fact), and produces Q2 query outputs (CSV). Two Colab-ready scripts are provided for ingestion, SCD processing and Q2 analytics.

---

## Files in this repo
- `build_insurance_dwh.py`  — full ETL → DWH pipeline (find ZIPs, extract, standardize, build SCD2 dims, create SKs, assemble fact, output CSVs):
  - Outputs: `merged_clean_26cols.csv`, `dim_customer.csv`, `dim_address.csv`, `dim_policy_type.csv`, `dim_policy.csv`, `fact_transactions.csv`
- `q2_colab_singlecell.py`  — alternate compact Colab-ready single cell that:
  - Extracts uploaded ZIPs,
  - Normalizes columns robustly,
  - Produces the Q2-specific outputs required for parts (b,c,d,e,f) and saves them to `q2_outputs/`.
  - Outputs: `q2_b_policy_type_changed.csv`, `q2_c_total_policy_amt_all.csv`, `q2_d_total_policy_amt_auto.csv`, `q2_e_east_west_quarterly_2012.csv`, `q2_f_marital_status_changed.csv`
- `RUN_IN_COLAB.md` — short instructions to run the single-cell Colab script.
- `dwh_schema.sql` — sample Postgres DDL to create tables and load CSVs (provided below in README).
- `README.md` — this file.

> If you used different filenames, update the paths accordingly.

---

## Architecture / Design (high level)
1. **Ingest**: find region ZIPs named like `Insurance_details_US_<REGION>_day.zip`, extract CSVs.
2. **Standardize**: normalize column names, clean whitespace, coerce types, and ensure canonical columns.
3. **Natural keys**:
   - `policy_nk = src_region_file + '|' + policy_id`
   - `address_nk = md5(customer_id + country + region + state + city + postal_code)`
4. **SCD Type-2**:
   - For `customer` and `policy` we detect history by hashing tracked columns across `snapshot_day`.
   - When the hash changes we create a new version row with `start_day` and `end_day`.
5. **Surrogate Keys (SKs)**:
   - Version SKs are generated using `pd.factorize(entity|start_day)` (integer SKs starting at 1).
   - SKs are used as PKs in dims and FKs in the fact table.
6. **Temporal assignment**:
   - For each fact snapshot row we assign `customer_sk` / `policy_sk` by finding the version whose `start_day` ≤ snapshot_day < next start (binary-search via `np.searchsorted`).
7. **Outputs**:
   - DWH CSVs (4 dims + 1 fact)
   - Q2 answer CSVs (parts b–f in `q2_outputs/`)

---

## How SCD Type-2 works (concise)
- For each entity (customer/policy) we build a fingerprint of tracked columns per snapshot. If the fingerprint differs from the previous snapshot for the same entity, a new version is emitted.
- Each version row has:
  - `start_day` = snapshot when change observed
  - `end_day` = next version's `start_day` (or `NULL` if current)
  - `is_current` = `end_day IS NULL`
- Facts are linked to the exact version active at their `snapshot_day` via SK.

---

## How to run (Colab / local)

### Option A — Run in Colab (recommended for hackathon)
1. Open Colab and create a new notebook.
2. Upload the 4 region ZIPs (files) to Colab `/content` (left pane → Files → Upload).
3. Copy the cell content of `q2_colab_singlecell.py` (the single-cell script in the repo) into a Colab cell and run it.
4. Outputs will be in `/content/q2_outputs` (download from the left pane).

_See `RUN_IN_COLAB.md` for quick steps._

### Option B — Run full ETL script locally / Colab
1. Put the 4 ZIPs in one of these: `/content`, `/content/drive/MyDrive` (Colab), or `/mnt/data`.
2. Run `build_insurance_dwh.py` (requires Python 3.8+, pandas, numpy).
```bash
python build_insurance_dwh.py

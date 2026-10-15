# Synapse - Healthcare Operations Analytics Platform

One view of **hospital occupancy, patient flow and ambulance operations** for an 8-hospital network:
a star-schema warehouse (SQL Server and Snowflake), a validated and idempotent Python ETL, KPI views,
a Power BI model with a 40+ measure DAX library and row-level security, and an LLM-written monthly briefing
that cannot quote a number that is not in the data.

> All data is **synthetic** (93,800 rows across 7 tables). No real patient information is used or implied.

![architecture](docs/architecture.svg)

## What's in the box
| Layer | What | Where |
|---|---|---|
| Data | Synthetic generator: seasonality, two flu surges, rural response delays, optional dirty rows | `python/generate_data.py` |
| Warehouse | Star schema DDL for SQL Server, Snowflake and SQLite (local) | `sql/*/01_schema.sql` |
| ETL | Validate, quarantine, stage, MERGE, reconcile, log | `python/etl.py` |
| KPI layer | 3 views (CTEs, window functions: rolling 7-day avg, LAG, RANK) and a pandas KPI step | `sql/*/02_kpi_views.sql`, `python/kpis.py` |
| Narrative | Claude or OpenAI, number-guarded, template fallback | `python/narrative.py` |
| BI | 43 DAX measures, RLS roles, Power Query, build guide | `powerbi/` |
| Docs | Data dictionary, design decisions, interview notes | `docs/` |
| Tests | 12 tests: volume, reconciliation, quarantine, idempotency, FK integrity, view parity, MERGE SQL, narrative guard | `tests/` |

## Quick start (local, no cloud account needed)
```bash
pip install -r requirements.txt
python python/generate_data.py                 # writes data/raw/*.csv (add --inject-dirty to demo validation)
python python/etl.py --init --load             # SQLite warehouse at data/synapse.db
python python/kpis.py                          # outputs/kpis_*.csv
python python/narrative.py                     # outputs/narrative_<month>.md (template unless an API key is set)
python -m pytest -q tests                      # 12 tests
```
Run `python python/etl.py --load` a second time: every table reports **0 rows affected**.

### SQL Server or Snowflake
```bash
cp .env.example .env     # set SQLSERVER_URL or SNOWFLAKE_URL, install the optional drivers in requirements.txt
python python/etl.py --target sqlserver --init --load
python python/kpis.py --target sqlserver
```
`--init` applies `01_schema.sql` then `02_kpi_views.sql` for that target.

### Narrative with an LLM
Set `ANTHROPIC_API_KEY` and/or `OPENAI_API_KEY`. `--provider auto|anthropic|openai|template` selects the source.
The facts JSON (numbers and rule-based flags only, no patient-level data) is the only thing sent. The reply is checked and any
number not in the facts triggers the template fallback, as do API errors and missing SDKs. The source used is written to the output.

## Data model
Dimensions: `dim_date`, `dim_hospital`, `dim_department`, `dim_ambulance`.
Facts: `fact_bed_occupancy_daily` (department x day), `fact_admissions` (per admission), `fact_ambulance_dispatch` (per dispatch).
Full column list, grains and KPI definitions: [`docs/data_dictionary.md`](docs/data_dictionary.md).

## KPIs
Occupancy % (ratio of bed-days), 7-day rolling occupancy, hospital-days over 95%, admissions/discharges/net flow, ALOS,
average and P90 ED wait, % waiting over 4h, 30-day readmission %, in-hospital mortality %, dispatches, dispatch delay,
average response time, % within target by priority (P1 8 min, P2 15, P3 30), job-cycle time.

## Example output (December 2025, from the generated data)
Network occupancy 89.3% (+6.7 pts vs November, flu-surge window), average ED wait 82.6 min, 30-day readmission 11.2%,
78.0% of priority-1 calls reached within 8 minutes. The same figures are the reconciliation targets in
[`powerbi/build_guide.md`](powerbi/build_guide.md).

## Power BI
The report file itself is not included; `powerbi/build_guide.md` is the exact recipe (tables, 11 relationships incl. two inactive,
measures, RLS, five pages, and the numbers to reconcile against).

## Verification status
| Verified in this repo | Written but not run here |
|---|---|
| Whole pipeline on SQLite: generate -> ETL -> views -> KPIs -> narrative (template and fallback paths) | SQL Server and Snowflake execution (DDL, MERGE, views are written to each platform's syntax; MERGE text is unit-tested) |
| 12 automated tests passing | Live Anthropic / OpenAI calls (guard and fallback are tested with stub providers) |
| Idempotent re-run, quarantine of injected bad rows | The `.pbix` and DAX evaluated inside Power BI |

## Project layout
```
python/    generate_data.py  etl.py  kpis.py  narrative.py  common.py
sql/       sqlserver/  snowflake/  sqlite/      (01_schema.sql, 02_kpi_views.sql)
powerbi/   measures.dax  rls_roles.md  power_query.m  build_guide.md  sample_user_access.csv
docs/      architecture.svg/.png  data_dictionary.md  design_decisions.md  interview_notes.md
tests/     test_pipeline.py
data/raw/  generated CSVs        outputs/  KPI CSVs and narrative
```

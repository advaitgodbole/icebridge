# IceBridge Migration Accelerator – Functional Design Document

*Generated on 2025-05-14*

---

## Objectives
1. **End‑to‑end pipeline** for migrating foreign Iceberg tables → managed Iceberg tables.  
2. **Workspace‑native**: UI, API, and compute run in Databricks; compute on ephemeral **Photon** job clusters.  
3. **Governance built‑in**: app exits unless `external‑data access` and `EXTERNAL USE SCHEMA` are enabled.  
4. **Extensible & observable**: Postgres state, Prometheus metrics, hooks for zero‑copy & DV writes.  

---

## Development environment
```bash
# Python
git clone … && cd icebridge
pyenv install 3.11.7 && pyenv local 3.11.7
poetry install

# Node
corepack enable
pnpm dlx create-vite@latest app --template react-ts

# Postgres (Docker)
docker run -e POSTGRES_PASSWORD=dev -p 5432:5432 postgres:15
```

---

## Modular breakdown

### 1. PostgreSQL layer (`/server/db`)
* **schema.sql** – tables: `catalogs`, `tables`, `jobs`, `validations`  
* **schema.py** – apply migrations with `psycopg_pool.ConnectionPool`  
* **models.py** – SQLAlchemy ORM classes  

### 2. FastAPI backend (`/server`)
| Module | Purpose |
|--------|---------|
| `auth.py` | Exchange workspace JWT for PAT (Secrets API) |
| `catalog.py` | Inventory & managed‑table creation |
| `cluster.py` | Build Photon cluster spec & submit |
| `jobs.py` | Wrapper for Jobs API `/runs/*` |
| `migration.py` | Generate Spark script & launch bulk copy |
| `validation.py` | Run GX checkpoint & publish docs |
| `sync.py` | Optional snapshot MERGE loop |
| `audit.py` | Assemble audit bundle |
| `db.py` | Pooled Postgres session |

### 3. Spark notebooks (`/spark`)
* **bulk_copy.py** – CREATE managed table + INSERT SELECT  
* **gx_runner.py** – run GX checkpoint; exit non‑zero on fail  
* **incremental_sync.py** – (optional) snapshot merge  

### 4. React front‑end (`/app/src`)
* **InventoryViewer** – React‑Query grid, CSV export  
* **CatalogMapper** – matrix UI (open Q2)  
* **MigrationWizard** – stepper (select → pre‑flight → plan → run)  
* **JobMonitor** – socket.io streaming  
* **ValidationDashboard** – iframe GX docs  

`app.yaml` for Databricks App:
```yaml
name: IceBridge
static_dir: ./app/dist
backend:
  command: python -m server.main
permissions:
  - principal: users
    allow: ["launch"]
```

---

## Secrets & security
* Secrets stored under `secrets://icebridge/*`  
* Backend fetches via `dbutils.secrets.get`  
* Front‑end receives only workspace JWT  

---

## Predictive Optimization
```python
spark.sql("ALTER CATALOG {dest_catalog} ENABLE PREDICTIVE OPTIMIZATION")
```

---

## Observability
* Prometheus exporter at `/metrics` (`prometheus-fastapi-instrumentator`)  
* Per‑table progress inserted into `system.migration_iceberg_jobs`  

---

## CI/CD (GitHub Actions)
```yaml
jobs:
  test:
    services:
      postgres:
        image: postgres:15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: '3.11'}
      - run: pip install poetry && poetry install
      - run: pytest
  deploy:
    needs: test
    steps:
      - uses: databricks/setup-cli@v1
      - run: databricks apps upload --path .
```

---

## Concurrency tuning (placeholder)
Default:
```toml
max_parallel_writers = 64
max_commits_per_min  = 800
```
Benchmark commit conflicts and expose slider in Settings modal.

---

## Implementation timeline
| Week | Deliverable |
|------|-------------|
| 1 | DB schema & health route |
| 2 | Inventory API + UI |
| 3 | Photon cluster orchestration |
| 4 | Bulk copy notebook + migrations endpoint |
| 5 | GX integration & dashboard |
| 6 | Cut‑over logic + PO toggle |
| 7 | Secrets & metrics |
| 8 | App bundle & staging QA |
| 9 | Resolve open questions |
| 10 | Docs & GA release |

---

*This document is the step‑level blueprint for coding IceBridge in **Python, React, PostgreSQL** inside Databricks.*

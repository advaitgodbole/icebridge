# IceBridge Migration Accelerator – Functional Design Document  
*Generated on 2025-05-14 – package‑manager update to **uv***  

---

## Objectives  
1. **End‑to‑end pipeline** for migrating foreign Iceberg tables → managed Iceberg tables.  
2. **Workspace‑native**: UI, API, and compute run in Databricks; compute on ephemeral **Photon** job clusters.  
3. **Governance built‑in**: app exits unless `external‑data access` and `EXTERNAL USE SCHEMA` are enabled. [turn0search1]  
4. **Extensible & observable**: Postgres state, Prometheus metrics, hooks for zero‑copy & DV writes.  

---

## Development environment  

> **Change‑note** — The project now uses **[uv]** as the Python package & virtual‑environment manager instead of Poetry. uv is an ultra‑fast, Rust‑based drop‑in compatible replacement for pip/virtualenv that resolves and installs dependencies 10‑100× faster in benchmarks [turn0search0] [turn0search2] [turn0search9].

### Prerequisites  
* Python 3.11.7 (via `pyenv` or system interpreter)  
* Node 18+  
* Docker (for local Postgres)

### Bootstrap steps  

```bash
# Clone repo
git clone … && cd icebridge

# Install uv (one‑liner from Astral) 
curl -LsSf https://astral.sh/uv/install.sh | sh          # installs uv in ~/.local/bin [turn0search1]

# Create a virtual‑env and install deps
uv venv .venv                                            # creates .venv with isolated Python
uv pip install -r requirements.txt                       # lightning‑fast install via uv‑pip wrapper

# Generate a lock‑file (optional but recommended)
uv pip freeze > requirements-lock.txt

# Node toolchain for React
corepack enable
pnpm dlx create-vite@latest app --template react-ts      # vite + pnpm quick‑start [turn0search5] [turn0search13]

# Local Postgres for dev
docker run -e POSTGRES_PASSWORD=dev -p 5432:5432 postgres:15 [turn0search6]
```

**Why uv?**  

* Written in Rust – drastically faster resolution/install than pip/Poetry [turn0search0] [turn0search9].  
* Single binary (no global site‑packages pollution) – CI‑friendly.  
* Fully interoperable with `requirements*.txt` and `pyproject.toml` (PEP 621).  

---

## Modular breakdown  

### 1 · PostgreSQL layer (`/server/db`)  
* **schema.sql** – tables: `catalogs`, `tables`, `jobs`, `validations`  
* **schema.py** – apply migrations with `psycopg_pool.ConnectionPool` [turn0search3][turn0search11]  
* **models.py** – SQLAlchemy ORM classes  

### 2 · FastAPI backend (`/server`)  
FastAPI remains the async API layer [turn0search4][turn0search12].  

| Module | Purpose |
|--------|---------|
| `auth.py` | exchange workspace JWT for PAT (Secrets API) |
| `catalog.py` | inventory & managed‑table creation |
| `cluster.py` | build Photon cluster spec & submit Jobs API |
| `jobs.py` | wrapper for Jobs API `/runs/*` |
| `migration.py` | generate Spark script & launch bulk copy |
| `validation.py` | run GX checkpoint & publish docs [turn0search8] |
| `sync.py` | optional snapshot MERGE loop |
| `audit.py` | assemble audit bundle |
| `db.py` | pooled Postgres session |

### 3 · Spark notebooks (`/spark`)  
* **bulk_copy.py** – CREATE managed table + INSERT SELECT  
* **gx_runner.py** – run GX checkpoint; `exit(1)` on fail  
* **incremental_sync.py** – optional snapshot merge  

### 4 · React front‑end (`/app/src`)  
* **InventoryViewer** – React‑Query grid, CSV export  
* **CatalogMapper** – matrix UI (open question)  
* **MigrationWizard** – stepper (select → pre‑flight → plan → run)  
* **JobMonitor** – socket.io streaming  
* **ValidationDashboard** – iframe GX docs  

`app.yaml` for Databricks App bundling [turn0search8]:

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
* Secrets stored at `secrets://icebridge/*`; backend fetches via `dbutils.secrets.get`.  
* Front‑end receives workspace JWT only (never PAT).  

---

## Predictive Optimization  
After copy:

```python
spark.sql("ALTER CATALOG {dest_catalog} ENABLE PREDICTIVE OPTIMIZATION")  # [turn0search3]
```

---

## Observability  
* **prometheus-fastapi-instrumentator** exposes `/metrics` [turn0search7].  
* Per‑table metrics inserted into `system.migration_iceberg_jobs`.  

---

## CI/CD (GitHub Actions) – uv edition  

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
      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh
      - name: Install deps
        run: ~/.local/bin/uv pip install -r requirements.txt
      - run: pytest
  deploy:
    needs: test
    steps:
      - uses: databricks/setup-cli@v1
      - run: databricks apps upload --path .
```

`uv pip install` completes in seconds even on cold CI runners, shrinking pipeline time.

---

## Concurrency tuning (placeholder)  
```toml
max_parallel_writers = 64
max_commits_per_min  = 800
```
Benchmark commit conflicts in sprint 9; slider exposed in **Settings** modal.

---

## Implementation timeline (unchanged)  

| Week | Deliverable |
|------|-------------|
| 1 | DB schema & health route |
| 2 | Inventory API + UI |
| 3 | Photon cluster orchestration |
| 4 | Bulk copy notebook + migrations endpoint |
| 5 | GX integration & dashboard |
| 6 | Cut‑over logic + PO toggle |
| 7 | Secrets & metrics |
| 8 | App bundle & staging QA |
| 9 | Resolve open questions |
| 10 | Docs & GA release |

---

*This version updates the tooling to use **uv** while preserving every other design element, adding concrete install commands, CI steps, and rationale. All factual statements are backed by public documentation or project READMEs.*  

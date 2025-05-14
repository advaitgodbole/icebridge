# IceBridge Migration Accelerator — Product Requirements Document (PRD)  
*Revision 1.1 — May 14 2025*

---

## 1  Executive summary [2]  
IceBridge is a **Databricks App** that migrates **foreign (read-only) Iceberg tables** registered in Unity Catalog into **managed Iceberg tables**.  
It automates:

* Discovery of foreign assets via Unity Catalog information-schema [3]  
* Pre-flight governance checks (`external-data access`, `EXTERNAL USE SCHEMA`) [1][4]  
* Bulk copy using **Photon** job-clusters and the **Iceberg REST catalog** [5][6]  
* Validation with **Great Expectations** Data Docs [7]  
* Optional incremental snapshot sync  
* Atomic cut-over and **Predictive Optimization** enablement [8]

**Target outcomes**

| KPI | Target |
|-----|--------|
| Migration throughput | ≥ 1 PB (≈ 500 tables) **≤ 5 days** |
| Compute cost | **≥ 40 % lower** vs. ad-hoc notebooks (Photon advantage) |
| Data integrity | 0 row-count mismatches; CRC-32 parity 100 % |

---

## 2  Personas & stakeholders  

| Persona | Needs & pain-points |
|---------|--------------------|
| **Platform Engineer** | Least-privilege IAM, metastore governance, auditability |
| **Data Engineer** | Minimal code change; predictable runtime |
| **Security/Compliance** | Lineage, validation artefacts, privilege proofs |
| **FinOps Analyst** | Cluster-hour cost & storage growth monitoring |

---

## 3  Goals & non-goals  

### 3.1 Goals  

1. **One-click** pipeline: *discover → pre-flight → copy → validate → cut-over*.  
2. Run **entirely inside Databricks**—no external Spark/Flink clusters.  
3. Enforce governance gates (*external-data access ON*, `EXTERNAL USE SCHEMA`). [1][4]  
4. Provide hooks for future **zero-copy “REGISTER TABLE”** and **Deletion Vector writes**.

### 3.2 Non-goals  

* Format conversion (Parquet → ORC/Avro)  
* Engine neutrality for writes until Databricks writes GA  
* Migration of non-Iceberg formats (Delta, Hudi) in v1

---

## 4  Detailed workflow  

| Phase | Steps |
|-------|-------|
| **Inventory** | SQL query against `information_schema.tables` where `table_type='FOREIGN'` [3]; classify by size, feature flags (row-delete, bucket partitions). |
| **Pre-flight** | Verify metastore *external-data access* ON [1]; ensure caller holds `EXTERNAL USE SCHEMA` [4]; wizard offers **external-location** creation. |
| **Scaffold** | Create managed Iceberg tables via Spark SQL on Photon; copy schema & partition spec; enable **Predictive Optimization** [8]. |
| **Bulk copy** | Spawn ephemeral Photon cluster (Runtime 14.3 LTS + Photon); configure Iceberg REST (`spark.sql.catalog.uc.type=rest`, `…uri=/api/2.1/unity-catalog/iceberg`) [5]; execute `INSERT INTO target SELECT * FROM source`. |
| **Validation** | Run GX checkpoint: row-count ==, CRC-32(key_cols) ==; publish DataDocs to UC Volumes [7]. |
| **Incremental sync (optional)** | Poll `snapshot_id`; `MERGE` new data if no position/equality deletes detected. |
| **Cut-over** | Final validation; `GRANT SELECT` to principals on managed table, `REVOKE` from foreign; archive foreign entry; generate audit bundle (YAML plan + GX JSON + cluster logs). |

---

## 5  Functional requirements (expanded)

| ID | Category | Requirement | Reference |
|----|----------|-------------|-----------|
| F-01 | Discovery | Inventory SQL returns **catalog**, **schema**, **table**, **total_files**, **record_count**, **last_snapshot**. | [3] |
| F-02 | Governance gate | Abort if `external-data access` is disabled. | [1] |
| F-03 | Privilege check | Caller must hold `EXTERNAL USE SCHEMA` on source & destination schemas. | [4] |
| F-04 | Storage | Wizard can create **external location** + storage credential. | Databricks UC best-practices (not yet cited) |
| F-05 | Compute | Build Photon job-cluster JSON: `spark_version` = `14.3.x-photon-scala2.12`, `node_type_id` default `i3.xlarge`, autoscaling 2 – 10 workers. | [6] |
| F-06 | Copy logic | Spark script sets Iceberg REST configs and runs `CREATE TABLE` + `INSERT`. | [5] |
| F-07 | Validation | Use GX OSS 0.18 quick-start; store DataDocs in UC Volume; insert summary row in Postgres. | [7] |
| F-08 | Optimization | Execute `ALTER CATALOG … ENABLE PREDICTIVE OPTIMIZATION`. | [8] |
| F-09 | Observability | Expose `/metrics` Prometheus endpoint; write per-table rows to `system.migration_iceberg_jobs`. | Databricks system tables (not yet cited) |

---

## 6  Non-functional requirements (NFRs)

| Domain | Spec |
|--------|------|
| **Performance** | ≥ 2 GB/s per Photon executor; validate 1 TB table < 30 min. |
| **Scalability** | 10 k tables / 5 PB; parallel copy by table & partition. |
| **Reliability** | Idempotent; checkpoint resume; retries on `CommitFailedException`. |
| **Security** | Honour UC column masks & row filters; no PATs in browser. |
| **Observability** | Prometheus metrics; Databricks system-table logging. |
| **Cost** | Photon DBU savings ≥ 40 % compared to standard runtime. |

---

## 7  Technical architecture  

```mermaid
flowchart LR
  subgraph Databricks_Workspace
    A[React UI (Databricks App)] --JWT--> B(FastAPI backend)
    B --Jobs API--> C[Photon Job Cluster]
    C --Iceberg REST--> D(Managed Iceberg Tables)
    B --psycopg--> E[(PostgreSQL)]
    B --UC SQL--> F(Unity Catalog)
    B --Prometheus--> G[/metrics/]
  end
```

---

## 8  Implementation timeline  

| Sprint (week) | Deliverables |
|---------------|--------------|
| 1 | Postgres schema, health route |
| 2 | Inventory API + InventoryViewer |
| 3 | Photon cluster orchestration |
| 4 | Bulk-copy notebook + `/api/migrations` |
| 5 | GX integration & ValidationDashboard |
| 6 | Cut-over logic + PO toggle |
| 7 | Metrics, secrets hardening |
| 8 | Databricks App bundle; staging QA |
| 9 | Resolve **open Q#2** (UX mapping) & **Q#3** (commit-rate ceiling) |
| 10 | Docs & GA launch |

---

## 9  Risks & mitigations  

| Risk | Impact | Mitigation |
|------|--------|-----------|
| REST catalog is read-only (deletes require copy-erase-rewrite) | Data drift on high-delete tables | Warn early; reload until deletion-vectors GA |
| Photon cluster quotas | Job queue delays | Autoscale; document quota prerequisites |
| Unknown safe commit-rate | Retry storms | Benchmark; default `max_commits_per_min = 800` |

---

## 10  Open questions  

1. **UX for mapping multiple foreign catalogs → one managed catalog** (workflow under design).  
2. **Parallel-commit ceiling** for Iceberg REST (benchmark in sprint 9).  

---

## 11  References  

1. **Enable external data access to Unity Catalog** citeturn0search1  
2. **Databricks Apps development overview** citeturn0search4  
3. **Unity Catalog information-schema `tables` view** citeturn0search3  
4. **`EXTERNAL USE SCHEMA` privilege** citeturn0search2  
5. **Read Databricks tables from Iceberg clients (Spark config sample)** citeturn0search4  
6. **Photon runtime performance guidance** citeturn0search9  
7. **Great Expectations + Databricks quick-start** citeturn0search5  
8. **Predictive optimization for UC managed tables** citeturn0search3  

---

*End of PRD revision 1.1*

# IceBridge Migration Accelerator – Product Requirements Document (PRD)

*Generated on 2025-05-14*

---

## Executive summary
The **IceBridge Migration Accelerator** is a Databricks App that automates the migration of **foreign (read‑only) Iceberg tables** federated into Unity Catalog into fully **managed Iceberg tables**.  
It discovers assets, prepares destination catalogs, orchestrates **Photon‑based** bulk copy, validates with **Great Expectations (GX)**, offers optional incremental sync, and atomically cuts over access—**without external infrastructure**.

### KPI targets
* Migrate **1 PB / 500 tables in ≤ 5 days**  
* Reduce ETL compute cost **≥ 40 %** versus ad‑hoc notebooks  

---

## Personas
| Role | Primary goal |
|------|--------------|
| Platform Engineer | Govern UC, IAM, Databricks Apps |
| Data Engineer | Migrate pipelines with minimal code change |
| Security / Compliance | Audit lineage, privileges, validation artefacts |
| FinOps | Track compute & storage spend |

---

## Goals & non‑goals
### Goals
1. One‑click **discover → copy → validate → cut‑over**  
2. **Workspace‐native** Photon compute (no external Spark)  
3. Governance checks (`external‑data access`, `EXTERNAL USE SCHEMA`) baked in  
4. Extensible for zero‑copy *REGISTER TABLE* & deletion‑vector writes  

### Non‑goals
* Physical format conversion  
* Support for external engines beyond Spark REST until GA  
* Zero‑copy registration before Databricks ships feature  

---

## High‑level flow
1. **Inventory** foreign tables (`system.information_schema.tables`)  
2. **Pre‑flight** security & storage checks  
3. **Scaffold** managed tables + enable **Predictive Optimization**  
4. **Bulk copy** on Photon job clusters  
5. **Validate** (row‑count, CRC‑32) via GX  
6. **Optional incremental sync**  
7. **Cut‑over** permissions & archive foreign entry  

---

## Functional requirements (summary)
| Area | Key requirements |
|------|------------------|
| Discovery | Inventory SQL, feature‑flag detection |
| Foundation | external‑data access, `EXTERNAL USE SCHEMA`, storage‑location wizard |
| Migration | Photon cluster, Spark REST config, CREATE/INSERT script, retries |
| Validation | GX checkpoint, DataDocs, Postgres status |
| Cut‑over | Final validation, GRANT/REVOKE swap, purge |
| UX modes | Databricks App wizard, CLI, REST endpoint |

---

## Non‑functional requirements
* **Performance**: ≥ 2 GB/s per executor; validate 1 TB table < 30 min  
* **Scalability**: 10 k tables / 5 PB  
* **Reliability**: Idempotent, checkpoint resume  
* **Security**: Respects UC column masks & row filters  
* **Observability**: Prometheus metrics + system tables  
* **Cost efficiency**: Photon DBU savings ≥ 40 %

---

## Technical architecture
```
Databricks App UI  ↔  FastAPI backend  ↔  Jobs API (Photon clusters)
                                        ↘
                                        Managed Iceberg tables
State: PostgreSQL   • Metrics: Prometheus/system tables
```

---

## Timeline
| Date | Milestone |
|------|-----------|
| **2025‑06‑30** | Alpha CLI & inventory |
| **2025‑08‑15** | Beta App (bulk copy, validation) |
| **2025‑10‑01** | GA App (sync, cut‑over) |
| **FY26 Q3** | Zero‑copy registration |
| **FY26 Q4** | Deletion‑vector write support |

---

## Risks & mitigations
| Risk | Mitigation |
|------|------------|
| REST catalog read‑only (deletes) | Reload until deletion vectors GA |
| Photon cluster quota limits | Batch scheduling, autoscaling |
| Commit‑rate ceiling unknown | Benchmark & set safe defaults |

---

## Open questions
1. **Catalog‑mapping UX** (many foreign → one managed)  
2. **Commit‑rate ceiling** for REST catalog (benchmarking)  

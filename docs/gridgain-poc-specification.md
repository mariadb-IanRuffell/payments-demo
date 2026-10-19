# GridGain as Mainframe System-of-Record Replacement — POC Specification

**Version:** 0.1 (Draft)
**Date:** 2026-10-19
**Owner:** Ian Ruffell
**Status:** For review

---

## 1. Executive Summary

This Proof of Concept (POC) evaluates whether **GridGain** (the enterprise distributed database and in-memory computing platform built on Apache Ignite) can serve as the **system of record (SoR)** for a core financial workload currently hosted on a mainframe (e.g., IBM z/OS running CICS/COBOL over DB2 or IMS).

A system of record is not a cache. It must be **durable, strongly consistent, transactional, auditable, and continuously available**. This POC is therefore designed to prove three properties under conditions that mirror production, not just a benchmark:

1. **Performance** — sustained throughput and low, predictable tail latency for OLTP payment operations, at parity with or better than the mainframe baseline.
2. **Scalability** — near-linear horizontal scaling of both throughput and data volume, with elastic scale-out/in and no downtime.
3. **Resilience** — zero data loss (RPO = 0) and rapid recovery (low RTO) under node, rack, and data-center failure, including split-brain protection, rolling upgrades, and verifiable backup/restore.

The POC succeeds if GridGain meets or exceeds the acceptance thresholds in [§9](#9-acceptance-criteria-kpis) while demonstrating the durability and consistency guarantees required of an SoR ([§8](#8-system-of-record-guarantees-durability--consistency)).

> **Assumptions to confirm with stakeholders** (see [§13](#13-assumptions--open-questions)): the reference workload is core payments / account ledger; target scale and SLAs below are illustrative placeholders to be replaced with the customer's actual production figures and mainframe baseline.

---

## 2. Objectives & Non-Goals

### 2.1 Objectives
- Demonstrate GridGain can act as the **authoritative SoR**, holding the golden copy of account and transaction data with full ACID guarantees and native disk persistence.
- Quantify performance, scalability, and resilience against defined, measurable KPIs.
- Establish a **defensible comparison** to the incumbent mainframe on cost-per-transaction, latency, throughput, and availability.
- Validate integration paths: ingest from upstream channels and emit change events to downstream systems (CDC) without weakening SoR guarantees.
- Produce evidence (dashboards, logs, recorded chaos experiments) suitable for an architecture review board and risk/compliance sign-off.

### 2.2 Non-Goals
- Full application migration or COBOL rewrite — the POC uses a representative workload harness, not the entire production application.
- Production security hardening beyond what is needed to prove SoR viability (TLS, auth, and audit are in scope conceptually; full pen-test is out).
- Downstream analytics/warehouse migration.

---

## 3. Reference Workload

A **core payments / demand-deposit account (DDA) ledger** — the workload most commonly resident on the mainframe and hardest to move because it is the SoR.

### 3.1 Entities (illustrative)
| Entity | Approx. rows | Notes |
|---|---|---|
| `ACCOUNT` | 50–200 M | Balance, status, currency, overdraft limit |
| `LEDGER_ENTRY` | 5–20 B (append-only) | Double-entry postings; the audit backbone |
| `TRANSACTION` | 2–10 B | Payment instructions, lifecycle state |
| `CUSTOMER` | 20–80 M | KYC reference data |
| `PARTY_LIMIT` | 50–200 M | Daily/exposure limits |

### 3.2 Operation mix (OLTP)
| Operation | Type | % of load | Consistency requirement |
|---|---|---|---|
| Balance inquiry | Point read (key lookup) | 45% | Read-committed, current balance |
| Post payment (debit/credit) | **Distributed ACID txn** across ≥2 accounts + ledger insert | 30% | Serializable / repeatable-read |
| Authorization + limit check | Read + conditional update | 15% | Atomic check-and-update |
| Statement / mini-statement | Range scan (SQL) | 7% | Read-committed |
| Reversal / correction | Compensating ACID txn | 3% | Serializable |

The **post-payment** operation is the critical path: it must be a genuine distributed transaction (money cannot be created or destroyed), which is precisely where a naive cache fails and where GridGain's transactional grid must prove itself.

### 3.3 Batch dimension (mainframe "batch window")
A key mainframe pain point is the nightly batch window (interest accrual, statement generation, EOD reconciliation). The POC includes a **batch scenario** to show GridGain can run these as distributed compute (colocated with data) concurrently with online traffic, shrinking or eliminating the batch window.

---

## 4. Reference Architecture

```
        Upstream channels (ISO 20022 / cards / RTP)
                        │
                        ▼
        ┌──────────────────────────────────┐
        │   Load / Workload Harness         │  (payment simulator, YCSB, custom)
        └──────────────────────────────────┘
                        │  thin-client / JDBC / REST
                        ▼
   ┌───────────────────────────────────────────────────────┐
   │                 GRIDGAIN CLUSTER (SoR)                  │
   │                                                         │
   │   DC-A (primary)                DC-B (DR)               │
   │   ┌─────┐ ┌─────┐ ┌─────┐       ┌─────┐ ┌─────┐         │
   │   │Node1│ │Node2│ │Node3│  ...  │Node4│ │Node5│  ...    │
   │   └─────┘ └─────┘ └─────┘       └─────┘ └─────┘         │
   │   Partitioned caches, backups=2                         │
   │   Native Persistence (disk) on every node              │
   │   Baseline topology + persistence = durable SoR         │
   └───────────────────────────────────────────────────────┘
                        │  CDC / Kafka
                        ▼
        Downstream: data warehouse, fraud, notifications
```

### 4.1 Key GridGain mechanisms under test
| SoR requirement | GridGain mechanism |
|---|---|
| Durability (survives full cluster restart) | **Native Persistence** — disk-backed, write-ahead log (WAL), checkpointing |
| No data loss on node failure | **Partitioned cache with `backups ≥ 2`** synchronous backups |
| ACID transactions | **Transactional cache mode**, 2-phase commit, pessimistic/optimistic + serializable |
| Strong consistency & durable membership | **Baseline topology** (which nodes hold persistent data) |
| Split-brain protection | **Segmentation resolver / discovery**, majority-based baseline |
| Geo-redundancy | **GridGain Data Center Replication (DR)** — active-passive or active-active between DC-A and DC-B |
| Point-in-time recovery | **Cluster snapshots** (+ incremental / WAL for PITR) |
| Query | **ANSI-99 SQL** with distributed joins, indexes |
| Zero-downtime maintenance | **Rolling upgrade**, online rebalancing |
| Fast batch | **Colocated compute / compute grid** over persistent data |

---

## 5. Test Environment

> Right-size to match the customer's expected production footprint. The table below is a starting reference for a mid-tier core banking workload.

| Component | Spec (per node) | Count | Notes |
|---|---|---|---|
| GridGain data nodes | 16 vCPU, 128 GB RAM, NVMe SSD (≥ 3 GB/s), 10 GbE | 6 (DC-A) + 6 (DC-B) | Persistence data region sized to hot set; disk sized to full dataset ×1.5 |
| Load generators | 16 vCPU, 32 GB | 3–4 | Isolated from cluster nodes |
| Monitoring | GridGain Control Center + Prometheus/Grafana | 1 | Metrics, latency histograms, cluster health |
| Network | Same-DC ≤ 1 ms RTT; cross-DC 5–40 ms RTT (emulate real inter-DC) | — | Cross-DC latency materially affects DR + txn design |

**Environment principles**
- Persistence **enabled** on all data nodes (this is an SoR, not a cache POC).
- `backups = 2`, `writeSynchronizationMode = PRIMARY_SYNC` (or `FULL_SYNC` for the strictest durability test).
- Dedicated disks for WAL vs. data where possible.
- All results reproducible: pinned versions, config committed to repo, scripted deployment.

---

## 6. Data Model & Dataset

- **Volume:** Load to a realistic scale (e.g., 100 M accounts, 5 B ledger entries) so persistence, rebalancing, and query performance are tested at data-heavy conditions — not just in-memory hot sets.
- **Distribution:** Skewed access (Zipfian) to model real hot accounts, plus a uniform variant for comparison.
- **Affinity colocation:** Colocate `LEDGER_ENTRY` and `PARTY_LIMIT` with their `ACCOUNT` by affinity key so debit/credit transactions and limit checks are node-local where possible (measure the effect).
- **Referential integrity:** Enforced in the transactional posting logic (application-level constraints validated in test assertions).

---

## 7. Test Scenarios

Each scenario states **goal → method → measured metric → pass threshold** (thresholds consolidated in [§9](#9-acceptance-criteria-kpis)).

### 7.1 Performance (P)

| ID | Scenario | Method | Primary metric |
|---|---|---|---|
| P1 | Point-read latency | Sustained balance-inquiry load at target TPS | p50 / p99 / p99.9 read latency |
| P2 | Payment-posting latency | Distributed ACID debit/credit at target TPS | p99 / p99.9 txn latency, commit success rate |
| P3 | Peak throughput | Ramp load until latency SLA breached | Max sustained TPS at SLA |
| P4 | Mixed realistic load | Full operation mix (§3.2) at production ratio | End-to-end TPS + tail latency |
| P5 | SQL range/query | Statement generation, ad-hoc SQL | Query latency, throughput |
| P6 | Batch as compute | EOD accrual/statements via compute grid, concurrent with OLTP | Batch wall-clock time; OLTP latency impact |
| P7 | Sustained soak | 24–72 h at 70% peak | Latency/throughput drift, GC pauses, memory stability |

### 7.2 Scalability (S)

| ID | Scenario | Method | Primary metric |
|---|---|---|---|
| S1 | Throughput scale-out | Run identical load at 3, 6, 9, 12 nodes | TPS vs. node count (linearity coefficient) |
| S2 | Data volume scale | Grow dataset 1×→2×→4× at fixed node count, then add nodes | Latency stability, rebalance time |
| S3 | Elastic scale-out | Add nodes to a live, loaded cluster | Rebalance duration, latency impact, zero errors |
| S4 | Elastic scale-in | Gracefully remove nodes under load | No data loss, controlled rebalance, latency impact |
| S5 | Client connection scale | Ramp concurrent thin clients / connections | TPS and latency vs. connection count |

**Linearity target:** doubling nodes yields ≥ 1.7× throughput (≥ 85% scaling efficiency) within the tested range.

### 7.3 Resilience (R) — the core of the SoR case

| ID | Scenario | Method | Primary metric |
|---|---|---|---|
| R1 | Single node kill (hard) | `kill -9` a data node under load | Data loss (must be 0), failover time, error window |
| R2 | Multiple node loss | Kill 2 nodes (= backup count) simultaneously | Data loss = 0, cluster stays available |
| R3 | Rack / AZ failure | Drop a fault domain (subset of nodes) | Availability maintained, RPO = 0 |
| R4 | Full data-center failure (DR) | Sever DC-A; promote DC-B | RTO, RPO, data consistency post-failover |
| R5 | Split-brain / network partition | Partition the cluster network | No divergent writes; minority segment fenced |
| R6 | Full cluster restart (durability) | Stop all nodes, restart from persistence | 100% data recovery, correct balances, WAL replay |
| R7 | Rolling upgrade | Upgrade GridGain version node-by-node under load | Zero downtime, zero data loss, no client errors |
| R8 | Backup & restore | Take cluster snapshot; restore to clean cluster | Restore correctness + time; PITR to a chosen point |
| R9 | Disk/WAL fault injection | Fill disk / fail a WAL volume on one node | Graceful degradation, no corruption, recovery |
| R10 | Slow node / gray failure | Inject latency on one node | Cluster isolates/mitigates; tail latency bounded |

**Resilience validation harness:** a continuous **money-conservation invariant** runs throughout every R-scenario — total system balance across all accounts must remain constant (sum of debits = sum of credits) before, during, and after each fault. Any violation is an automatic fail. This is the single most important SoR proof.

### 7.4 System-of-Record integrity (I)

| ID | Scenario | Method | Primary metric |
|---|---|---|---|
| I1 | ACID under contention | Concurrent transfers on hot accounts | No lost updates, no phantom money; serializable correctness |
| I2 | Crash-mid-transaction | Kill coordinator node mid-2PC | Transaction fully commits or fully rolls back — never partial |
| I3 | Consistency after failover | Reconcile balances after every R-scenario | Balances match expected ledger exactly |
| I4 | Audit trail completeness | Verify append-only ledger vs. issued transactions | Zero missing/duplicate entries |
| I5 | CDC fidelity | Compare downstream CDC stream to committed writes | Exactly-once / no-gap delivery of committed changes |

---

## 8. System-of-Record Guarantees (Durability & Consistency)

This section makes explicit what must be **true**, not just fast, for GridGain to be trusted as SoR:

1. **RPO = 0** — no committed transaction is ever lost, under any single-fault (and defined multi-fault) scenario. Proven by R1–R6 + the money-conservation invariant.
2. **Durability across total outage** — committed data survives a full cluster power-off, proven by R6 (persistence + WAL replay).
3. **Atomicity** — multi-account postings are all-or-nothing across nodes, proven by I1–I2 including coordinator failure mid-commit.
4. **Isolation correctness** — no lost updates / write skew under contention at the chosen isolation level, proven by I1.
5. **Auditability** — append-only, complete, ordered ledger, proven by I4.
6. **Recoverability** — verifiable backup/restore and point-in-time recovery, proven by R8.

If any of guarantees 1–4 fail, the POC fails regardless of performance numbers.

---

## 9. Acceptance Criteria (KPIs)

> Replace placeholder targets with the customer's real mainframe SLAs before execution. These illustrate a demanding-but-realistic core-banking bar.

| Category | KPI | Target (illustrative) | Scenario |
|---|---|---|---|
| Performance | Point-read p99 | ≤ 2 ms | P1 |
| Performance | Payment-posting p99 | ≤ 10 ms | P2 |
| Performance | Payment-posting p99.9 | ≤ 25 ms | P2 |
| Performance | Sustained mixed throughput | ≥ 50,000 TPS (scale to customer peak ×2 headroom) | P4 |
| Performance | Soak stability (72 h) | < 5% latency drift, no memory growth | P7 |
| Performance | Batch window | EOD batch ≤ 30 min with OLTP running (vs. multi-hour mainframe window) | P6 |
| Scalability | Scaling efficiency | ≥ 85% (linear) across 3→12 nodes | S1 |
| Scalability | Online scale-out impact | ≤ 10% latency degradation during rebalance; 0 errors | S3 |
| Resilience | Data loss (any single-fault) | **0 bytes / 0 transactions** | R1–R6, I3 |
| Resilience | Node-failure failover | ≤ 5 s to stable; error window ≤ 3 s | R1 |
| Resilience | DR failover RTO | ≤ 60 s | R4 |
| Resilience | DR failover RPO | 0 (sync DR) or ≤ agreed bound (async DR) | R4 |
| Resilience | Rolling upgrade | 0 downtime, 0 client errors, 0 data loss | R7 |
| Resilience | Full-restart recovery | 100% data, correct balances | R6 |
| Integrity | Money-conservation invariant | Holds at all times | All R + I |
| Integrity | ACID contention correctness | 100% (no anomalies) | I1–I2 |
| Availability | Measured over POC | ≥ 99.999% equivalent | All |

---

## 10. Mainframe Baseline & Comparison

To make the replacement case, compare like-for-like against the incumbent:

| Dimension | Mainframe baseline (to be gathered) | GridGain (measured in POC) |
|---|---|---|
| Peak TPS | e.g., CICS/DB2 measured peak | P3/P4 result |
| Read latency p99 | Current SLA | P1 result |
| Txn latency p99 | Current SLA | P2 result |
| Batch window | e.g., 4–6 h nightly | P6 result |
| Availability | e.g., 99.999% | Measured |
| DR RTO/RPO | Current DR posture | R4 result |
| Cost | MIPS / MSU licensing + hardware + support ($/yr) | Node count × infra cost; **cost-per-transaction** |
| Scaling model | Vertical (buy MIPS) | Horizontal (add commodity nodes) |
| Skills | COBOL/z/OS scarcity | Java/SQL/cloud-native |

**Headline comparison metric:** **cost-per-transaction** and **cost-per-peak-TPS**, plus the qualitative shift from vertical MIPS procurement to elastic horizontal scaling.

---

## 11. Tooling & Measurement

- **Workload harness:** custom payments simulator (double-entry postings + money-conservation checker) as primary; YCSB and TPC-C-style variants for standardized comparison points.
- **Chaos/fault injection:** scripted (`kill -9`, `iptables`/`tc` for partition & latency, disk-fill), optionally Chaos Mesh / Pumba for reproducibility.
- **Latency measurement:** HdrHistogram-based, capturing full distribution (never averages) to p99.99; coordinated-omission–corrected.
- **Observability:** GridGain Control Center (cluster/txn/rebalance state) + Prometheus/Grafana (latency, throughput, GC, WAL, checkpoint, page memory, disk I/O).
- **Correctness assertions:** automated reconciliation job comparing issued transactions vs. persisted ledger vs. account balances after every scenario.
- **Reproducibility:** all cluster config, cache config, test scripts, and dashboards committed to the repo; each run tagged with version + config hash.

---

## 12. Execution Plan (indicative 6–8 weeks)

| Phase | Duration | Activities | Exit gate |
|---|---|---|---|
| 0. Setup & alignment | 1 wk | Confirm workload, SLAs, mainframe baseline, target scale; provision environment | Signed-off KPIs + baseline |
| 1. Build & load | 1 wk | Deploy cluster, schema, data load, harness, dashboards | Cluster durable + dataset loaded |
| 2. Performance | 1–1.5 wk | P1–P7 | Meets/misses P KPIs (documented) |
| 3. Scalability | 1 wk | S1–S5 | Linearity + elasticity proven |
| 4. Resilience & integrity | 1.5–2 wk | R1–R10, I1–I5 (the critical phase) | RPO=0 + invariant holds |
| 5. Comparison & report | 1 wk | Mainframe comparison, cost model, exec readout, evidence pack | Final report + go/no-go |

---

## 13. Assumptions & Open Questions

**Assumptions**
1. Reference workload is core payments / DDA ledger acting as SoR.
2. GridGain runs with Native Persistence enabled and `backups = 2` (not memory-only).
3. Two data centers available (or emulated) for DR testing.
4. A representative production dataset (or realistic synthetic) can be provisioned.

**Open questions for stakeholders** — *these materially shape scale, KPIs, and DR design:*
1. What is the **actual production peak TPS**, current p99 SLAs, and availability target to use as the baseline?
2. Is DR required as **RPO = 0 (synchronous)** or is a small async RPO acceptable? (Drives DC-B topology and cross-DC latency budget.)
3. **Full replacement** of the mainframe SoR, or **strangler-pattern** coexistence (GridGain fronting / offloading the mainframe during migration)?
4. Does GridGain need to be the SoR **standalone**, or integrate with an existing relational store (e.g., MariaDB) for a subset of data or for downstream reporting?
5. Regulatory/compliance constraints (data residency, encryption, audit retention) that must be demonstrated in-POC?
6. Required isolation level for postings (serializable vs. repeatable-read) — affects both correctness tests and latency.

---

## 14. Deliverables & Exit Criteria

**Deliverables**
- This specification (agreed & baselined).
- Reproducible test harness, cluster config, and dashboards (in repo).
- Results pack: latency histograms, throughput/scaling curves, chaos-experiment recordings, reconciliation reports.
- Mainframe-vs-GridGain comparison including cost-per-transaction model.
- Executive readout + architecture-review evidence bundle + go/no-go recommendation.

**Exit criteria (POC is a PASS if):**
1. All **Integrity (I)** and **critical Resilience (R1–R7)** criteria pass with **RPO = 0** and the money-conservation invariant intact.
2. Performance meets the agreed KPIs at the customer's peak ×2 headroom.
3. Scalability demonstrates ≥ 85% linear efficiency and non-disruptive elasticity.
4. A credible cost and operational case vs. the mainframe is established.

---

## Appendix A — Illustrative GridGain configuration highlights

- **Cache mode:** `PARTITIONED`, `backups = 2`, `atomicityMode = TRANSACTIONAL`.
- **Write sync:** `PRIMARY_SYNC` (default test) and `FULL_SYNC` (strict-durability comparison run).
- **Persistence:** Native Persistence on; dedicated WAL disk; checkpoint tuning documented per run.
- **Baseline topology:** explicitly set; auto-adjust behavior tested in R-scenarios.
- **Affinity:** colocate ledger/limit with account via affinity key.
- **DR:** GridGain Data Center Replication configured DC-A → DC-B; active-passive baseline, active-active as stretch test.
- **Security (conceptual):** TLS on client + server, authentication enabled, audit logging on.

## Appendix B — Glossary
- **SoR** — System of Record: the authoritative, durable source of truth.
- **RPO** — Recovery Point Objective: max acceptable data loss (target 0).
- **RTO** — Recovery Time Objective: max acceptable downtime.
- **WAL** — Write-Ahead Log (durability mechanism).
- **2PC** — Two-Phase Commit (distributed transaction protocol).
- **CDC** — Change Data Capture (streaming committed changes downstream).

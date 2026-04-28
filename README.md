
# RDS Multi-AZ vs. Cluster vs. Aurora: The Deep Dive

## 🚀 Preface: The Question

> **"Since both RDS Multi-AZ and Aurora use physical replication, why can Aurora failover in seconds while RDS still takes minutes?"**

Most answers stay at the surface: *"DNS is slow"* or *"Aurora is optimized."*  
But in real systems, failover behavior is determined by something deeper:

> **The core question is not replication speed — it is recovery workload distribution.**

If you don’t understand **LSN gaps**, **recovery mechanics**, and **state readiness**, you cannot accurately predict downtime under failure.

---

## TL;DR

Failover latency is **not primarily a DNS problem**.

> The real difference is **whether recovery work is on the failover path or off the critical path**.

**Engineering view:**

Failover latency =
detection time
promotion time
recovery work
client reconnection time

- **Multi-AZ Instance** → Recovery is *reactive* (during failover) → **Slow (60–120s)**
- **Multi-AZ Cluster** → Recovery is *proactive* (continuous replay) → **Faster (~30s)**
- **Aurora** → Recovery is largely *offloaded* → **Fastest (seconds)**

---

## 1. The "Physical Replication" Paradox

RDS Multi-AZ (Instance) uses physical replication at the storage level.

However:

> It is optimized for **durability (disk consistency)**, not **instant readiness (engine state)**.

The standby has the data safely stored, but not fully materialized into a ready-to-serve state.

---

## 2. Three Levels of State Synchronization

| Model | Sync Target | Standby State | Failover Cost |
|------|------------|--------------|--------------|
| **Multi-AZ Instance** | Disk (block-level) | Warm standby (not fully replayed) | High |
| **Multi-AZ Cluster** | Engine state (redo-applied) | Active reader | Medium |
| **Aurora** | Distributed storage quorum | Active reader | Low |

---

## 3. Why Multi-AZ Instance Is Slow: Recovery on the Critical Path

### Mechanism

1. Primary writes redo logs.
2. Storage layer replicates data blocks to standby.
3. Standby engine **does not continuously apply all changes to its working state**.

### Failover Flow

Primary crash
→ Promote standby
→ Crash Recovery (heavy)
→ DNS update
→ Client reconnect

### What Happens During Crash Recovery

- **Redo**: apply committed changes from logs  
- **Undo**: roll back incomplete transactions  

### Key Insight

> Recovery work is deferred until failure, making failover time **workload-dependent and unpredictable**.

---

## 4. Why Multi-AZ Cluster Is Faster: Recovery Shifted to Runtime

### Mechanism

- Primary continuously streams redo logs
- Standby continuously applies them

Primary → redo logs → Standby → continuous apply

### Resulting State

> The standby keeps its state **closely aligned** with the primary.
> (Not identical, but near-ready)

### Failover Flow

Promote standby
→ minimal recovery work
→ DNS update

### Important Clarification

> Failover still requires some recovery work,  
> but most of it has already been completed during normal operation.

---

## 5. Why Aurora Is the Fastest: Recovery Offloaded to Storage

Aurora re-architects the database by decoupling compute and storage.

### Key Design Principles

#### 1. Decoupled Compute and Storage
- Multiple instances share the same distributed storage

#### 2. Log-Centric Storage Model
- Logs are treated as the primary source of truth

#### 3. Storage-Assisted Recovery
- Storage layer ensures consistency using quorum replication

### Important Clarification

Aurora does **not eliminate recovery entirely**, but:

> It **minimizes traditional checkpoint and recovery overhead** by shifting work into the storage layer.

### Failover Flow

Select reader → promote to writer

### Result

> Most recovery work is removed from the failover critical path.

---

## 6. Deep Dive: The LSN Perspective

### Core Concepts

- **Log LSN**: latest operation recorded in logs  
- **Page LSN**: last update applied to a data page  

### Recovery Algorithm

Scan redo log
→ locate affected pages
→ compare Page LSN vs Log LSN
→ apply redo if needed

### Why This Is Expensive

#### 1. Random I/O
- Logs are sequential
- Data pages are scattered

#### 2. Strict Ordering
- Operations must be replayed in order

#### 3. Unbounded Workload
- High write rates → large recovery backlog

### Key Insight

> The gap between Page LSN and Log LSN determines how much work must be done during failover.

---

## 7. Failover Timeline Comparison

**Instance:**
[Crash] → [Promote] → [Heavy Recovery] → [Ready]

**Cluster:**
[Crash] → [Promote] → [Minimal Recovery] → [Ready]

**Aurora:**
[Crash] → [Promote] → [Ready]

---

## 8. Mental Model

- **Instance** → Do recovery at failure time  
- **Cluster** → Do recovery continuously  
- **Aurora** → Minimize recovery through architecture  

---

## 9. Architectural Recommendations

| Requirement | Recommended | Reason |
|------------|------------|-------|
| ~1–2 min RTO acceptable | Multi-AZ Instance | Simpler, but recovery-heavy |
| ~30s RTO | Multi-AZ Cluster | Pre-applied recovery reduces downtime |
| Seconds-level RTO | Aurora | Recovery largely offloaded |

---

## 10. Final Takeaway

> High availability is not about how fast you switch endpoints —  
> it is about how much recovery work remains when failure happens.

---

## Bonus Insight

> Modern database systems reduce failover latency by shifting recovery from a failure-time operation to an always-on background process.

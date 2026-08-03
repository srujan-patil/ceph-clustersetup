# Day 1 — Introduction to Ceph

**Version:** Ceph Squid (v19.x)

---

## Quick Summary

- Ceph is a distributed storage system providing **Block (RBD)**, **File (CephFS)**, and **Object (RGW/S3)** storage.
- No single point of failure — data is replicated across multiple nodes.
- Core daemons: **MON** (cluster state/quorum), **MGR** (dashboard/APIs), **OSD** (actual data storage).
- Data placement is handled automatically via **Placement Groups (PGs)** and the **CRUSH** algorithm — no central lookup table.
- Authentication is handled via **CephX**.
- This guide will deploy MON, MGR, OSD, Dashboard, cephadm, NGINX (reverse proxy), and Let's Encrypt (HTTPS).

---

## 1.1 What is Ceph?

Ceph is an open-source distributed storage platform designed to provide:

- **Block Storage** (RBD)
- **File Storage** (CephFS)
- **Object Storage** (RGW / S3 Compatible)

Unlike a traditional storage server, Ceph stores data across multiple servers, providing high availability, scalability, and fault tolerance.

---

## 1.2 Why Use Ceph?

**Traditional storage** relies on a single server, which is a single point of failure:

```mermaid
flowchart LR
    VM1(VM 1) --> S{{"Single Storage\nAppliance"}}
    VM2(VM 2) --> S
    VM3(VM 3) --> S
    S -.->|"if this dies,\neverything stops"| X["❌ Outage"]
    style S fill:#f96,stroke:#333
    style X fill:#fdd,stroke:#900
```

**Problems:**

- Single point of failure
- Limited scalability
- Expensive storage appliances
- Difficult expansion

**Ceph** distributes data across multiple nodes instead:

```mermaid
flowchart TB
    subgraph Node1["ceph1"]
        M1["MON"]
        O1["OSD"]
    end
    subgraph Node2["ceph2"]
        M2["MON"]
        O2["OSD"]
    end
    subgraph Node3["ceph3"]
        M3["MON"]
        O3["OSD"]
    end
    Pool(("Shared\nData Pool"))
    O1 --> Pool
    O2 --> Pool
    O3 --> Pool
    M1 -.-> M2 -.-> M3 -.-> M1
```

**Advantages:**

- No single point of failure
- Automatic replication
- Horizontal scaling
- Self-healing
- Open source

---

## 1.3 Ceph Architecture

A production Ceph cluster consists of multiple services:

```mermaid
sequenceDiagram
    participant C as Client (ceph/RBD/CephFS)
    participant MON as Monitor
    participant MGR as Manager
    participant OSD as OSD Layer
    C->>MON: 1. Request cluster map
    MON-->>C: 2. Return map + auth
    C->>OSD: 3. Read/write object directly
    MGR->>MON: (background) collect metrics
    MGR->>OSD: (background) collect stats
```

Each service has a specific role.

---

## 1.4 Ceph Components

### Monitor (MON)

The Monitor maintains the cluster state.

**Responsibilities:**

- Cluster membership
- Quorum
- Authentication
- Cluster map

Without a healthy Monitor quorum, the cluster cannot operate normally.

```mermaid
flowchart TD
    A["3 MONs deployed"] --> B{"How many alive?"}
    B -->|"3/3 or 2/3"| Healthy["✅ Quorum OK\nCluster operational"]
    B -->|"1/3 or 0/3"| Down["❌ Quorum lost\nCluster halts writes"]
```

> **Rule of thumb:** with 3 MONs, at least 2 must be available to maintain quorum.

### Manager (MGR)

The Manager provides:

- Dashboard
- REST APIs
- Monitoring
- Metrics
- Orchestration support

**Without the Manager:**

- Storage continues to work.
- Dashboard and management features are unavailable.

### OSD (Object Storage Daemon)

OSDs store the actual data. Each storage disk usually corresponds to one OSD.

```mermaid
flowchart LR
    subgraph Physical Disks
        d0["/dev/sdb"]
        d1["/dev/sdb"]
        d2["/dev/sdb"]
    end
    subgraph Logical OSDs
        o0["osd.0 (ceph1)"]
        o1["osd.1 (ceph2)"]
        o2["osd.2 (ceph3)"]
    end
    d0 --> o0
    d1 --> o1
    d2 --> o2
```

### Placement Groups (PGs)

Ceph does not place data directly onto OSDs. Instead:

```mermaid
flowchart TD
    F["File"] -->|"split into chunks"| Obj1["Object 1"]
    F -->|"split into chunks"| Obj2["Object 2"]
    Obj1 -->|"CRUSH hash"| PG1["PG 1.a"]
    Obj2 -->|"CRUSH hash"| PG2["PG 1.b"]
    PG1 --> OSDx["osd.0 / osd.1 / osd.2"]
    PG2 --> OSDy["osd.1 / osd.2 / osd.0"]
```

Placement Groups help distribute and rebalance data efficiently.

### CRUSH

CRUSH stands for:

**Controlled Replication Under Scalable Hashing**

CRUSH determines:

- Which OSD stores an object
- Where replicas are placed

No central lookup table is required — the location is computed algorithmically.

---

## 1.5 Data Flow

Suppose an application writes a file:

```mermaid
flowchart TD
    A(("① App writes file")) --> B["② Kubernetes intercepts I/O"]
    B --> C["③ PVC forwards to StorageClass"]
    C --> D["④ Ceph RBD image receives write"]
    D --> E["⑤ Data split into objects"]
    E --> F["⑥ CRUSH computes placement"]
    F --> G[("⑦ Written to OSDs")]
```

---

## 1.6 Replication

Assume a replication size of 3:

```mermaid
flowchart LR
    W["Client write: 1 GB"] --> P["Primary copy\n(osd.0)"]
    P ==>|"sync replicate"| R1["Copy 2\n(osd.1)"]
    P ==>|"sync replicate"| R2["Copy 3\n(osd.2)"]
    P --- Total["Total raw usage: 3 GB"]
```

**Example:**

| OSD  | Data Stored |
|------|-------------|
| OSD1 | 1 GB        |
| OSD2 | 1 GB        |
| OSD3 | 1 GB        |

> The application writes 1 GB, but Ceph stores **3 GB** of raw data (replication factor 3).

---

## 1.7 CephX Authentication

Ceph uses **CephX** for authentication.

**Example users:**

```
client.admin
client.kubernetes
```

Each user has:

- Username
- Secret key
- Permissions

**Example capabilities:**

```
mon:
  allow r

osd:
  allow rwx pool=kubernetes
```

---

## 1.8 Ceph Services Used in This Guide

In this guide, we'll configure:

| Service | Purpose |
|---|---|
| **MON** | Cluster monitoring and quorum |
| **MGR** | Dashboard and management |
| **OSD** | Data storage |
| **Dashboard** | Web UI |
| **cephadm** | Cluster deployment and orchestration |
| **NGINX** | Reverse proxy |
| **Let's Encrypt** | HTTPS for the dashboard |

---

## What's Next

Continue to [`cluster-setup`](../setup-cluster/) to begin preparing the 3 Ubuntu machines for the Ceph cluster.

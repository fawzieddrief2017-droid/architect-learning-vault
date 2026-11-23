# Control Plane Brain: `etcd` Raft Consensus, Storage Engine & Day-2 Disaster Recovery

## 1. Architectural Mental Model (The "Why" & "What")

Every Pod, Deployment, Service, Secret, and `EndpointSlice` in Kubernetes is a serialized Protobuf object stored inside **`etcd`**. While the `kube-apiserver` is a stateless horizontal gateway, `etcd` is the **sole stateful distributed brain** of the cluster.

If `etcd` loses majority quorum or violates its strict **10ms disk `fsync()` write budget**, the entire Kubernetes control plane locks up: no Pods can be scheduled, no `EndpointSlices` update, and no autoscaling events commit.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                 KUBE-APISERVER -> ETCD RAFT CONSENSUS WRITE PATH                            |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  CLIENT (kubectl / kubelet / Controller)
        │  HTTPS: PATCH /api/v1/namespaces/prod/pods/web-1/status
        ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ KUBE-APISERVER (Stateless Gateway)                                                                      │
  │   1. AuthN -> RBAC -> Admission Webhooks                                                                │
  │   2. API Priority & Fairness (APF): Queues request by priority level                                    │
  │   3. Writes Protobuf object over gRPC to etcd Leader                                                    │
  └─────────────────────────┬───────────────────────────────────────────────┘
                            │ gRPC Put("/registry/pods/prod/web-1")
                            ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ ETCD LEADER (Node 1)                                                                                    │
  │   1. Appends entry to local Write-Ahead Log (WAL) + calls fsync() (< 10ms budget!)                      │
  │   2. Sends AppendEntries RPC in parallel to Follower Node 2 and Follower Node 3                         │
  └──────────────────────────┬──────────────────────────────────────────────┬───────────────────────────────┘
                             │ AppendEntries + fsync()                      │ AppendEntries + fsync()
                             ▼                                              ▼
               [ ETCD FOLLOWER (Node 2) ]                     [ ETCD FOLLOWER (Node 3) ]
                             │                                              │
                             └──────────► Majority Quorum (2/3) ACK! ◄──────┘
                                                            │
                                                            ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ COMMIT & WATCH FAN-OUT                                                                                  │
  │   1. Leader commits entry to BoltDB (bbolt) B+Tree & increments global MVCC Revision                    │
  │   2. Pushes event to kube-apiserver Watch Cache                                                         │
  │   3. kube-apiserver fans out Watch event to 500 Informers (KCM, Scheduler, Cilium, kubelet)             │
  └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Under the Hood: Mechanics & Internals

### A. Raft Consensus & Quorum Math ($\lfloor N/2 \rfloor + 1$)
`etcd` does **not** scale write throughput by adding nodes—every write must be replicated to a strict majority before committing, so adding nodes *increases* network consensus latency. Always run an **odd** number of control plane members:

| Cluster Size ($N$) | Majority Quorum ($\lfloor N/2 \rfloor + 1$) | Max Tolerated Node Failures | Architectural Verdict |
| :--- | :--- | :--- | :--- |
| **1** | 1 | **0** | Dev/Test only. Zero high availability. |
| **2** | 2 | **0** | **Worse than 1!** Losing either node breaks quorum. |
| **3** | 2 | **1** | **Standard Production HA** (survives 1 node or 1 AZ failure). |
| **4** | 3 | **1** | **Anti-pattern:** Higher Raft latency & cost than 3 nodes, still survives only **1** failure! |
| **5** | 3 | **2** | **Mission-Critical HA** (survives 1 planned maintenance + 1 sudden crash simultaneously). |

---

### B. The 10ms WAL `fsync()` Barrier & MVCC `NOSPACE` Lockup
1. **Disk I/O Sensitivity:** `etcd` uses `heartbeat-interval: 100ms` and `election-timeout: 1000ms`. Every mutation requires a synchronous `fsync()` to the Write-Ahead Log (WAL). If a shared root disk or AWS `gp2` volume exhausts burst IOPS credits, `fsync()` latency spikes past 100ms–500ms. Heartbeats stall, followers trigger leader elections, and the API server fails with `etcdserver: request timed out`.
   * **Production Rule:** Isolate `/var/lib/etcd` onto a dedicated **NVMe SSD or AWS `gp3`/`io2` volume** ($\ge 3,000$ IOPS, `< 2ms` `fdatasync` p99).
2. **MVCC Revisions & `alarm:NOSPACE`:**
   - `etcd` defaults to `--quota-backend-bytes=2147483648` (**2 GiB**, max recommended **8 GiB**).
   - High Secret/Event churn fills BoltDB with historical revisions. At 2 GiB, `etcd` raises `alarm:NOSPACE` and forces the entire cluster into **read-only mode**.
   - **Compaction vs. Defragmentation:** `etcdctl compact` marks old revisions as free *logically* inside BoltDB, but only **`etcdctl defrag`** reclaims physical disk space—and `defrag` holds an **exclusive stop-the-world write lock**!

---

### C. `kube-apiserver` Watch Cache & Version Skew Policy
* **Watch Cache:** Shields `etcd` from hundreds of `kubelet`, `cilium-agent`, and controller `LIST/WATCH` storms by serving `resourceVersion` queries from an in-memory ring buffer inside `kube-apiserver`.
* **API Priority & Fairness (APF):** Classifies and queues incoming API requests into isolated priority lanes so a runaway custom controller cannot starve `kubelet` `NodeLease` heartbeats.
* **Kubernetes Upgrade Skew Policy:**
  1. Upgrade **`kube-apiserver`** first.
  2. Upgrade **`kube-controller-manager`** and **`kube-scheduler`** (supported up to **1 minor version** older than `kube-apiserver`).
  3. Upgrade **`kubelet`** on worker nodes via rolling cordon/drain (supported up to **3 minor versions** older than `kube-apiserver`).

---

## 3. Production Implementation: Backup, Restore & Safe Defragmentation

```bash
# 1. Inspect live etcd cluster health, DB size, and Raft Leader
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# 2. Take an atomic point-in-time snapshot backup (encrypt and ship to S3)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd-snapshot-$(date +%F).db

# 3. Restore snapshot to a new data-dir during catastrophic control plane loss
ETCDCTL_API=3 etcdutl snapshot restore /var/backups/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master-0 \
  --initial-cluster=master-0=https://10.0.1.10:2380 \
  --initial-cluster-token=etcd-cluster-recovered \
  --initial-advertise-peer-urls=https://10.0.1.10:2380

# 4. Safe rolling defragmentation (NEVER run --cluster in production!)
# Step A: Defragment Follower 1, then Follower 2
etcdctl defrag --endpoints=https://10.0.1.11:2379
etcdctl defrag --endpoints=https://10.0.1.12:2379
# Step B: Step down Leader, then defragment former Leader last
etcdctl move-leader <FOLLOWER_HEX_ID>
etcdctl defrag --endpoints=https://10.0.1.10:2379
# Step C: Disarm NOSPACE alarm
etcdctl alarm disarm
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

| Failure Mode | Symptom | Root Cause | Remediation |
| :--- | :--- | :--- | :--- |
| **Leader Election Storm** | `etcdserver: leader changed`, `took too long` in logs | Shared disk I/O (`fsync` > 10ms) starving WAL writes | Isolate `/var/lib/etcd` on dedicated NVMe / `io2` EBS |
| **Read-Only Freeze (`NOSPACE`)** | `mvcc: database space exceeded` on all writes | BoltDB hit 2GiB quota without compaction/defrag | `compact` $\rightarrow$ rolling 1-by-1 `defrag` $\rightarrow$ `alarm disarm` |
| **Total Quorum Loss** | `etcdserver: no leader` (500/504 from `kube-apiserver`) | $\ge \lceil N/2 \rceil$ control plane nodes permanently lost | Restore latest S3 snapshot via `etcdutl snapshot restore` |

---

## 5. Socratic Architectural Challenge

### Scenario:
A 3-node self-managed Kubernetes control plane (`master-1`, `master-2`, `master-3`) hits `alarm:NOSPACE` at `2.0 GiB`, locking the cluster into read-only mode. To recover quickly, an engineer runs `etcdctl defrag --cluster` during peak traffic.

### Question:
What happens to the cluster during `etcdctl defrag --cluster`, and what is the safe SRE procedure?

### Answer:
`etcdctl defrag` acquires an **exclusive stop-the-world BoltDB lock** to rebuild the `bbolt` file. Running `--cluster` freezes all 3 members simultaneously, breaking Raft quorum and causing a total control plane outage. The safe procedure is to compact old revisions, defragment **followers one by one** (preserving 2/3 quorum), transfer leadership via `etcdctl move-leader`, defragment the former leader last, and then run `etcdctl alarm disarm`.

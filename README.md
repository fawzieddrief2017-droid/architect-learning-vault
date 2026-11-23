# Cloud Systems Architecture & Kubernetes Internals

> **Author:** Fawzi Eddrief  
> **Contact / GitHub:** [fawzieddrief2017-droid](https://github.com/fawzieddrief2017-droid)  
> **Methodology:** First-Principles Engineering, Kernel Data Paths & Production Failure-Mode Analysis  
> **Target Specialization:** Senior Linux Systems Engineering, Cloud Architecture (AWS), and Production Kubernetes Core Internals  

---

## 📕 Downloadable Architecture Compendium (104-Page Field Manual)

The core systems curriculum (Pillars 1–4, 15 modules) is compiled into a publication-grade offline reference book:

* **[Download: Kubernetes-Systems-Architecture-Field-Manual.pdf](Kubernetes-Systems-Architecture-Field-Manual.pdf)**  
  *(104 Pages | Formatted for Print & High-DPI Readers | Updated regularly as Pillars 5 & 6 complete)*

---

## 1. Engineering Philosophy

This repository is a rigorous, publication-grade technical curriculum and operational reference for deep cloud infrastructure and container orchestration.

Rather than treating systems as high-level abstractions or memorizing `kubectl` syntax, every topic in this masterclass is systematically dissected down to:
- **What happens on the wire, in memory, and in the Linux kernel**.
- **Deterministic control plane reconciliation loops and state machines**.
- **Kernel isolation boundaries (`cgroups v2`, namespaces, and OCI runtimes)**.
- **Production failure modes under load, partition, or misconfiguration**.
- **Concrete, battle-hardened manifests, kernel sysctls, and CLI diagnostic pipelines**.

---

## 2. The 6 Pillars of Kubernetes Architecture

### Pillar 0: Distributed Tooling & Environment Sync

| Module | Core Architecture Covered | Low-Level Systems Reality |
| :--- | :--- | :--- |
| **[00. Multi-Host Synchronization Architecture](00-systems-and-tooling/cross-machine-sync-architecture.md)** | Distributed state sync between independent workstations | Decoupling deterministic content from ephemeral runtimes, SSH over port 443 multiplexing, atomic repo syncing. |

---

### Pillar 1: Compute Isolation & Container Runtime Engine (`01-pillar-compute-and-runtime/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **1.1 Pod Anatomy & Linux Namespaces** | [01-pod-anatomy-and-namespaces.md](01-pillar-compute-and-runtime/01-pod-anatomy-and-namespaces.md) | ✅ Completed | `/pause` subreaper, `CLONE_NEW*` namespaces, `nsenter` |
| **1.2 Linux IPC & Shared Memory** | [02-linux-ipc-and-shared-memory.md](01-pillar-compute-and-runtime/02-linux-ipc-and-shared-memory.md) | ✅ Completed | `/dev/shm` 64MiB ceiling, SysV vs. POSIX IPC, `emptyDir: Memory` |
| **1.3 Container Runtime Engine (Field Manual)** | [03-container-runtime-engine-architect-guide.md](01-pillar-compute-and-runtime/03-container-runtime-engine-architect-guide.md) | ✅ Completed | CRI gRPC, `containerd-shim`, `runc`, `SIGNAL_UNKILLABLE`, `inotify` math, CVE-2024-21626 |

---

### Pillar 2: Cluster Networking, DNS, Ingress & eBPF (`02-pillar-networking-and-ebpf/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **2.1 ClusterIP Service Packet Walk** | [01-service-clusterip-packet-walk.md](02-pillar-networking-and-ebpf/01-service-clusterip-packet-walk.md) | ✅ Completed | `KUBE-SERVICES` Netfilter chains, `nf_conntrack`, VXLAN |
| **2.2 CoreDNS, Conntrack Race & `ndots:5`** | [02-coredns-conntrack-race-and-ndots.md](02-pillar-networking-and-ebpf/02-coredns-conntrack-race-and-ndots.md) | ✅ Completed | `__nf_conntrack_confirm()` UDP races, 5s DNS spikes, `NodeLocal DNSCache` |
| **2.3 eBPF Socket-Level Bypass vs. CNI** | [03-ebpf-socket-bypass-vs-cni.md](02-pillar-networking-and-ebpf/03-ebpf-socket-bypass-vs-cni.md) | ✅ Completed | `BPF_PROG_TYPE_SOCK_OPS`, `SK_MSG`, `bpf_msg_redirect_hash()` |
| **2.4 End-to-End AWS ALB Ingress Walk** | [04-end-to-end-ingress-packet-walk.md](02-pillar-networking-and-ebpf/04-end-to-end-ingress-packet-walk.md) | ✅ Completed | ALB Instance vs. IP Mode (Secondary ENI IPs), Target Group deregistration |
| **2.5 Kernel Execution Hierarchy & BPF Masquerade** | [05-kernel-execution-hierarchy-and-bpf-masquerade.md](02-pillar-networking-and-ebpf/05-kernel-execution-hierarchy-and-bpf-masquerade.md) | ✅ Completed | `cgroup/connect4` vs. Netfilter vs. TC/XDP, lock-free per-CPU SNAT |
| **2.6 Service Discovery vs. In-Kernel Routing** | [06-service-routing-and-internal-dns.md](02-pillar-networking-and-ebpf/06-service-routing-and-internal-dns.md) | ✅ Completed | CoreDNS L7 discovery vs. `kube-proxy`/Cilium L4 `EndpointSlice` watch |
| **2.7 AWS VPC CNI, `/28` Prefix Delegation & BGP** | `07-aws-vpc-cni-prefix-delegation-and-bgp.md` | ⏳ Planned | Nitro ENI math, `/28` prefixes, `externalTrafficPolicy: Local` vs. `Cluster`, BGP |

---

### Pillar 3: Workloads, Pod Lifecycle & CSI Storage (`03-pillar-workloads-and-storage/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **3.1 ConfigMaps & Atomic Symlinks** | [01-configmaps-and-atomic-symlinks.md](03-pillar-workloads-and-storage/01-configmaps-and-atomic-symlinks.md) | ✅ Completed | `execve()` env immutability vs. atomic `rename()` symlink swap on `tmpfs` |
| **3.2 The 5 Workload Controllers Architecture** | [02-workload-controllers-architecture.md](03-pillar-workloads-and-storage/02-workload-controllers-architecture.md) | ✅ Completed | `Informer`/`WorkQueue`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob` |
| **3.3 Pod Lifecycle, Probes & Zero-Downtime Draining** | [03-pod-lifecycle-probes-and-zero-downtime-draining.md](03-pillar-workloads-and-storage/03-pod-lifecycle-probes-and-zero-downtime-draining.md) | ✅ Completed | Orthogonal state axes, Prober CRI math, `preStop.sleep`, L4/L7 drain |
| **3.4 CSI Storage & Two-Phase Mount Engine** | [04-csi-storage-and-volume-lifecycle.md](03-pillar-workloads-and-storage/04-csi-storage-and-volume-lifecycle.md) | ✅ Completed | `AttachDetachController`, `VolumeAttachment`, `NodeStage`/`NodePublish`, RWOP, STONITH |

---

### Pillar 4: Scheduling, QoS & Kernel Resource Sizing (`04-pillar-scheduling-and-qos/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **4.1 QoS, `cgroup v2`, CFS Throttling & OOM** | [01-qos-cgroups-cfs-throttling-and-oom.md](04-pillar-scheduling-and-qos/01-qos-cgroups-cfs-throttling-and-oom.md) | ✅ Completed | `cpu.weight`, `cpu.max`, `memory.max`, PSI, `oom_score_adj` (-997 to 1000) |
| **4.2 Scheduler Framework, Topology Spread & PDBs** | [02-scheduler-affinity-taints-and-pdb.md](04-pillar-scheduling-and-qos/02-scheduler-affinity-taints-and-pdb.md) | ✅ Completed | 11-stage Filter/Score/Bind pipeline, `maxSkew` math, `NoExecute` taints, PDBs |

---

### Pillar 5: Security, Cloud Identity & Micro-Segmentation (`05-pillar-security-and-identity/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **5.1 RBAC, IRSA vs. EKS Pod Identity, Webhooks & PSA** | [01-rbac-irsa-webhooks-and-psa.md](05-pillar-security-and-identity/01-rbac-irsa-webhooks-and-psa.md) | ✅ Completed | Bound JWTs, OIDC STS exchange, `169.254.170.23` agent, IMDSv2 hop-limit=1, Webhooks |
| **5.2 NetworkPolicies & eBPF/Cilium Micro-Segmentation** | `02-network-policies-and-microsegmentation.md` | 🎯 Next Target | Default-deny ingress/egress, CoreDNS port 53 rule, Cilium identity, `hubble observe` |

---

### Pillar 6: Control Plane Brain (`etcd`) & Day-2 Disaster Recovery (`06-pillar-control-plane-and-etcd/`)

| Module | Vault File | Status | Core Primitives & Systems Reality |
| :--- | :--- | :--- | :--- |
| **6.1 `etcd` Raft Consensus, Upgrades & Day-2 Triage** | `01-etcd-consensus-upgrades-and-day2-triage.md` | ⏳ Planned | Raft quorum math, WAL `fsync` $<10\text{ms}$, MVCC compaction/defrag, `etcdctl snapshot` |

---

## 3. Repository Structure

```
architect-learning-vault/
├── README.md                                          # Master 6-Pillar index & curriculum tracker
├── Kubernetes-Systems-Architecture-Field-Manual.pdf   # Compiled offline PDF compendium
├── .gitignore                                         # System filter (blocks sockets, locks, temp files)
│
├── 00-systems-and-tooling/
│   └── cross-machine-sync-architecture.md
│
├── 01-pillar-compute-and-runtime/                     # Pillar 1: Isolation, IPC & Runtime Engine
│   ├── 01-pod-anatomy-and-namespaces.md
│   ├── 02-linux-ipc-and-shared-memory.md
│   └── 03-container-runtime-engine-architect-guide.md
│
├── 02-pillar-networking-and-ebpf/                     # Pillar 2: ClusterIP, DNS, Ingress & eBPF
│   ├── 01-service-clusterip-packet-walk.md
│   ├── 02-coredns-conntrack-race-and-ndots.md
│   ├── 03-ebpf-socket-bypass-vs-cni.md
│   ├── 04-end-to-end-ingress-packet-walk.md
│   ├── 05-kernel-execution-hierarchy-and-bpf-masquerade.md
│   └── 06-service-routing-and-internal-dns.md
│
├── 03-pillar-workloads-and-storage/                   # Pillar 3: ConfigMaps, Controllers, Draining & CSI
│   ├── 01-configmaps-and-atomic-symlinks.md
│   ├── 02-workload-controllers-architecture.md
│   ├── 03-pod-lifecycle-probes-and-zero-downtime-draining.md
│   └── 04-csi-storage-and-volume-lifecycle.md
│
├── 04-pillar-scheduling-and-qos/                      # Pillar 4: cgroups v2, CFS, OOM, Scheduler & PDBs
│   ├── 01-qos-cgroups-cfs-throttling-and-oom.md
│   └── 02-scheduler-affinity-taints-and-pdb.md
│
├── 05-pillar-security-and-identity/                   # Pillar 5: RBAC, IRSA, Pod Identity & Webhooks
│   └── 01-rbac-irsa-webhooks-and-psa.md
│
└── 06-pillar-control-plane-and-etcd/                  # Pillar 6: etcd Raft, Watch Cache & Day-2 DR
    └── .gitkeep
```

---

## 4. Multi-Host Development Workflow

To sync updates across engineering workstations:

```bash
# Push updates from current host
git add -A
git commit -m "docs: add deep-dive systems analysis module"
git push origin main

# Pull updates on secondary host
git pull --rebase origin main
```

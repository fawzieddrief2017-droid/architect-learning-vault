# Cloud Systems Architecture & Kubernetes Internals

> **Author:** Fawzi Eddrief  
> **Contact / GitHub:** [fawzieddrief2017-droid](https://github.com/fawzieddrief2017-droid)  
> **Methodology:** First-Principles Engineering, Kernel Data Paths & Production Failure-Mode Analysis  
> **Target Specialization:** Senior Linux Systems Engineering, Cloud Architecture (AWS), and Production Kubernetes Core Internals  

---

## 1. Engineering Philosophy

This repository is a rigorous, publication-grade technical curriculum and operational reference for deep cloud infrastructure and container orchestration. 

Rather than treating systems as high-level abstractions or memorizing YAML syntax, every topic in this masterclass is systematically dissected down to:
- **What happens on the wire, in memory, and in the Linux kernel**.
- **Deterministic control plane reconciliation loops and state machines**.
- **Kernel isolation boundaries (`cgroups v2`, namespaces, and OCI runtimes)**.
- **Production failure modes under load, partition, or misconfiguration**.
- **Concrete, battle-hardened manifests, kernel sysctls, and CLI diagnostic pipelines**.

---

## 2. Curriculum Overview & Master Modules

### Part 1: Distributed Infrastructure & Environment Architecture

| Module | Core Architecture Covered | Low-Level Systems Reality |
| :--- | :--- | :--- |
| **[01. Multi-Host Synchronization Architecture](01-systems-and-tools/cross-machine-sync-architecture.md)** | Distributed state sync between independent physical machines | Decoupling deterministic technical content from ephemeral host runtimes, SSH over port 443 multiplexing, atomic repository syncing. |

### Part 2: Kubernetes Core Internals & The Linux Kernel

| Phase | Module ID & Title | Vault File (`02-kubernetes-internals/`) | Status | Cross-References & Core Primitives |
| :--- | :--- | :--- | :--- | :--- |
| **Phase 1: Linux Isolation & Runtime** | **1.1** Pod Anatomy & Namespaces | [01-pod-anatomy-and-namespaces.md](02-kubernetes-internals/01-pod-anatomy-and-namespaces.md) | Completed | `/pause` subreaper, `CLONE_NEW*`, `nsenter` |
| **Phase 1** | **1.2** Linux IPC & Shared Memory | [05-linux-ipc-and-shared-memory.md](02-kubernetes-internals/05-linux-ipc-and-shared-memory.md) | Completed | `/dev/shm` 64MiB ceiling, `emptyDir: Memory` |
| **Phase 1** | **1.3** Container Runtime Engine | [09-container-runtime-engine-architect-guide.md](02-kubernetes-internals/09-container-runtime-engine-architect-guide.md) | Completed | CRI gRPC, `containerd-shim`, `runc`, `SIGNAL_UNKILLABLE`, `inotify` math |
| **Phase 2: Cluster Networking & DNS** | **2.1** ClusterIP Packet Walk | [02-service-clusterip-packet-walk.md](02-kubernetes-internals/02-service-clusterip-packet-walk.md) | Completed | `KUBE-SERVICES`, `nf_conntrack`, VXLAN |
| **Phase 2** | **2.2** CoreDNS & `ndots:5` | [04-coredns-conntrack-race-and-ndots.md](02-kubernetes-internals/04-coredns-conntrack-race-and-ndots.md) | Completed | UDP conntrack races, `NodeLocal DNSCache` |
| **Phase 2** | **2.3** eBPF Socket Bypass | [06-ebpf-socket-bypass-vs-cni.md](02-kubernetes-internals/06-ebpf-socket-bypass-vs-cni.md) | Completed | `BPF_PROG_TYPE_SOCK_OPS`, `SK_MSG` |
| **Phase 2** | **2.4** AWS ALB Ingress Walk | [07-end-to-end-ingress-packet-walk.md](02-kubernetes-internals/07-end-to-end-ingress-packet-walk.md) | Completed | ALB Instance vs. IP mode, Target Group deregistration |
| **Phase 2** | **2.5** Kernel Execution Hierarchy | [08-kernel-execution-hierarchy-and-bpf-masquerade.md](02-kubernetes-internals/08-kernel-execution-hierarchy-and-bpf-masquerade.md) | Completed | `connect4` vs. Netfilter vs. TC/XDP |
| **Phase 2** | **2.6** Service Discovery vs. Routing | [10-service-routing-and-internal-dns.md](02-kubernetes-internals/10-service-routing-and-internal-dns.md) | Completed | CoreDNS L7 vs. `kube-proxy` L4 `EndpointSlice` watch |
| **Phase 2** | **2.7** AWS VPC CNI, `/28` Prefix Delegation & BGP | `15-aws-vpc-cni-prefix-delegation-and-bgp.md` | Planned | Nitro ENI math, `/28` prefixes, `externalTrafficPolicy: Local` vs. `Cluster`, BGP |
| **Phase 3: Workloads, Lifecycle & Storage** | **3.1A** ConfigMaps & Atomic Symlinks | [03-configmaps-and-atomic-symlinks.md](02-kubernetes-internals/03-configmaps-and-atomic-symlinks.md) | Completed | `execve()` env immutability vs. `rename()` symlink swap |
| **Phase 3** | **3.1B** The 5 Workload Controllers | [11-workload-controllers-architecture.md](02-kubernetes-internals/11-workload-controllers-architecture.md) | Completed | `Informer`/`WorkQueue`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob` |
| **Phase 3** | **3.2** Pod Lifecycle, Probes & Zero-Downtime Draining | [12-pod-lifecycle-probes-and-zero-downtime-draining.md](02-kubernetes-internals/12-pod-lifecycle-probes-and-zero-downtime-draining.md) | Completed | Orthogonal state axes, Prober CRI math, `preStop.sleep`, L4/L7 drain |
| **Phase 3** | **3.3** CSI Storage & Volume Lifecycle | [13-csi-storage-and-volume-lifecycle.md](02-kubernetes-internals/13-csi-storage-and-volume-lifecycle.md) | Completed | `AttachDetachController`, `VolumeAttachment`, `NodeStage`/`NodePublish`, RWO Multi-Attach |
| **Phase 4: Scheduling & Resource Sizing** | **4.1** QoS, `cgroup v2`, CFS Throttling & OOM | [14-qos-cgroups-cfs-throttling-and-oom.md](02-kubernetes-internals/14-qos-cgroups-cfs-throttling-and-oom.md) | Completed | `cpu.max`, `memory.max`, PSI (`cpu.pressure`), `oom_score_adj`, Kubelet eviction |
| **Phase 4** | **4.2** Scheduler Internals, Affinity, Taints & PDBs | [16-scheduler-affinity-taints-and-pdb.md](02-kubernetes-internals/16-scheduler-affinity-taints-and-pdb.md) | Completed | Filter/Score plugins, `topologySpreadConstraints`, `NoExecute` tolerations, PDB |
| **Phase 5: Security & Micro-Segmentation** | **5.1** RBAC, IRSA / Pod Identity, Webhooks & PSA | `17-rbac-irsa-webhooks-and-psa.md` | Planned | Bound JWTs, OIDC exchange, Admission Webhooks, Audit Logs |
| **Phase 5** | **5.2** NetworkPolicies & eBPF/IPSet Enforcement | `18-network-policies-and-microsegmentation.md` | Planned | Default-deny ingress/egress, CoreDNS port 53 rule, `hubble observe` |
| **Phase 6: Control Plane & Day-2 Operations** | **6.1** `etcd` Raft Consensus, Upgrades & Triage | `19-etcd-consensus-upgrades-and-day2-triage.md` | Planned | Raft quorum, WAL `fsync` $<10\text{ms}$, MVCC compaction, `etcdctl snapshot`, `kubeadm` |

---

## 3. Repository Structure

```
architect-learning-vault/
├── README.md                                  # Master technical index, architecture overview & navigation
├── .gitignore                                 # System filter (blocks sockets, locks, temp files)
│
├── 01-systems-and-tools/                      # Systems architecture, diagnostics, sync pipelines
│   └── cross-machine-sync-architecture.md
│
└── 02-kubernetes-internals/                   # Step-by-step kernel and systems engineering masterclass modules
    ├── 01-pod-anatomy-and-namespaces.md
    ├── 02-service-clusterip-packet-walk.md
    ├── 03-configmaps-and-atomic-symlinks.md
    ├── 04-coredns-conntrack-race-and-ndots.md
    ├── 05-linux-ipc-and-shared-memory.md
    ├── 06-ebpf-socket-bypass-vs-cni.md
    ├── 07-end-to-end-ingress-packet-walk.md
    ├── 08-kernel-execution-hierarchy-and-bpf-masquerade.md
    ├── 09-container-runtime-engine-architect-guide.md
    ├── 10-service-routing-and-internal-dns.md
    ├── 11-workload-controllers-architecture.md
    ├── 12-pod-lifecycle-probes-and-zero-downtime-draining.md
    ├── 13-csi-storage-and-volume-lifecycle.md
    ├── 14-qos-cgroups-cfs-throttling-and-oom.md
    └── 16-scheduler-affinity-taints-and-pdb.md
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

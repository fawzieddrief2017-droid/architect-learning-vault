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

| Module | Core Architecture Covered | Low-Level Systems Reality |
| :--- | :--- | :--- |
| **[02. Pod Anatomy & Linux Namespaces](02-kubernetes-internals/01-pod-anatomy-and-namespaces.md)** | The Pod Sandbox Pattern, `/pause` container, multi-container communication | Namespace matrix (`MNT`, `NET`, `IPC`, `UTS`, `PID`, `USER`), zombie process reaping via `pause()`, host-level `nsenter` debugging. |
| **[03. ClusterIP Service Packet Walk](02-kubernetes-internals/02-service-clusterip-packet-walk.md)** | Service IP routing across nodes without physical interface binding | Netfilter `KUBE-SERVICES` chains, `statistic --mode random` probabilistic selection, `nf_conntrack` state tables, and VXLAN encapsulation. |
| **[04. ConfigMaps & Dynamic Volumes](02-kubernetes-internals/03-configmaps-and-atomic-symlinks.md)** | Decoupling configuration from immutable containers | Process stack `execve()` immutability (env vars) vs. atomic `rename()` symlink swapping on in-memory `tmpfs` volumes. |
| **[05. CoreDNS, Conntrack Race & ndots:5](02-kubernetes-internals/04-coredns-conntrack-race-and-ndots.md)** | Intermittent 5-second DNS latency spikes | `__nf_conntrack_confirm()` lock contention, parallel A/AAAA UDP socket collisions, `ndots:5` query multiplication, and NodeLocal DNSCache. |
| **[06. Linux IPC & Shared Memory](02-kubernetes-internals/05-linux-ipc-and-shared-memory.md)** | Shared memory between Pod containers | Virtual Memory Manager page table mapping, SysV vs. POSIX IPC, `/dev/shm` 64MiB crash limit, and `emptyDir` tmpfs scaling. |
| **[07. eBPF Socket-Level Bypass vs. CNI](02-kubernetes-internals/06-ebpf-socket-bypass-vs-cni.md)** | Direct kernel socket routing | `BPF_PROG_TYPE_SOCK_OPS`, `BPF_PROG_TYPE_SK_MSG`, `bpf_msg_redirect_hash()`, bypassing the TCP/IP stack, Netfilter, and veth pairs. |
| **[08. End-to-End Ingress Packet Walk](02-kubernetes-internals/07-end-to-end-ingress-packet-walk.md)** | Ingress from public client to container socket | Route 53 -> AWS ALB -> AWS Load Balancer Controller discovery loop -> Target Group (Instance vs. IP Mode) -> Node driver -> Netfilter / eBPF -> Pod socket. |
| **[09. Kernel Execution Hierarchy & BPF Masquerade](02-kubernetes-internals/08-kernel-execution-hierarchy-and-bpf-masquerade.md)** | Order of precedence across the kernel stack & eBPF SNAT | Userspace syscall boundary (`connect4`) vs Netfilter Layer 3 vs driver Layer 2 (XDP/TC); `bpf.masquerade` lock-free per-CPU hash maps vs `iptables MASQUERADE`. |
| **[10. The Container Runtime Engine (Architect Field Manual)](02-kubernetes-internals/09-container-runtime-engine-architect-guide.md)** | High-signal executive distillation of container runtime essentials | Signal vs. noise filtration, CRI v1 architecture, containerd-shim subreaper decoupling, capacity planning math (inotify, NOFILE, PTYs), container security threat models (CVE-2024-21626, CVE-2019-5736), PID 1 signal masking, and Terraform/EKS UserData decision matrix. |
| **[11. Service Discovery vs. In-Kernel Packet Routing](02-kubernetes-internals/10-service-routing-and-internal-dns.md)** | Decoupling CoreDNS discovery (L7) from kube-proxy routing (L4) | In-memory kubelet Pod Manager Go structs, EndpointSliceController HTTP/2 Watch streams, service process debunking, and Headless Service peer-to-peer resolution. |

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
    └── 10-service-routing-and-internal-dns.md
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

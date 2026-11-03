# Kubernetes Internals: Kernel Execution Hierarchy & BPF Masquerade

## 1. Architectural Mental Model (The "Why" & "What")

A central debate in Linux systems engineering is understanding the boundary between **User Space Application Logic**, **Kernel System Calls**, **eBPF Acceleration**, and **Netfilter Firewalling**.

The key principles are:
1. **OSI Layer 6 (Presentation) is a Userspace Myth:** Data formatting, JSON serialization, and TLS encryption happen entirely in User Space (CPU Ring 3) before a single syscall is issued.
2. **The Syscall Boundary:** When an application calls `connect()` or `sendmsg()`, it hands the destination address to the kernel in memory.
3. **eBPF Preemption:** eBPF hooks attached to `cgroup/connect4` intercept the destination address **before an IP packet or Ethernet frame is constructed**.
4. **Masquerading (SNAT):** When a Pod with a private overlay IP (`10.244.1.25`) speaks to an external server, the node must rewrite the packet's source IP to the node's public/routable IP (`192.168.1.50`). Doing this via traditional `iptables MASQUERADE` creates severe spinlock contention (`xtables.lock`); doing this via **`bpf.masquerade`** uses lock-free per-CPU BPF hash maps.

```
+---------------------------------------------------------------------------------------------------------------+
|                                    KERNEL EXECUTION HIERARCHY: SOCKET TO WIRE                                 |
+---------------------------------------------------------------------------------------------------------------+

  APPLICATION (User Space: CPU Ring 3)
    │
    │  1. Application formats payload & sets target: struct sockaddr_in { IP: 10.96.0.100, Port: 80 }
    │  2. Syscall: connect(fd, &target, sizeof(target))
    ▼
=================================================================================================================
  KERNEL SPACE (CPU Ring 0: Privileged Mode)
=================================================================================================================
  [ LAYER 1: BSD SOCKET LAYER ]
    │
    ├───► [ eBPF: BPF_PROG_TYPE_CGROUP_SOCK_ADDR (connect4) ]
    │        ├── Rewrites target IP in memory before connection is established!
    │        └── Netfilter never needed for DNAT; IP already points to backend Pod!
    ▼
  [ LAYER 2: TRANSPORT LAYER (TCP State Machine) ]
    │
    ├───► [ eBPF: sock_ops / sk_msg ]
    │        ├── Is destination Pod on the SAME NODE?
    │        │    ├── YES: bpf_msg_redirect_hash() -> Divert data directly to Pod B socket buffer!
    │        │    └── NO:  Fall through to Layer 3...
    ▼
  [ LAYER 3: NETWORK / IP LAYER (ip_output / ip_forward) ]
    │
    ├───► [ NETFILTER: PREROUTING / OUTPUT / FORWARD / POSTROUTING ]
    │        ├── Traditional iptables / nftables chains evaluated here
    │        └── Global connection tracking table: nf_conntrack
    ▼
  [ LAYER 4: TRAFFIC CONTROL & DRIVER LAYER (TC / XDP) ]
    │
    ├───► [ eBPF: TC Ingress/Egress & XDP ]
    │        ├── BPF Masquerade (eBPF SNAT via cilium_snat_v4_external)
    │        └── XDP wire-speed packet filtering on NIC driver ring buffer
    ▼
  PHYSICAL ETHERNET WIRE (NIC DMA Transfer)
```

---

## 2. Under the Hood: Traditional Iptables Masquerade vs. BPF Masquerade

```
+---------------------------------------------------------------------------------------------------------------+
|                               TRADITIONAL IPTABLES MASQUERADE vs. BPF MASQUERADE                              |
+---------------------------------------------------------------------------------------------------------------+

  OUTBOUND PACKET LEAVING NODE (From Pod 10.244.1.25 to External DB 192.168.1.200)
       │
       ├───────────────────────────────────────────────────────┐
       ▼                                                       ▼
 [ TRADITIONAL IPTABLES MASQUERADE ]            [ CILIUM BPF MASQUERADE ]
       │                                                       │
       ▼                                                       ▼
  1. Packet hits Netfilter POSTROUTING hook.              1. Packet hits eBPF TC hook at physical interface.
       │                                                       │
  2. Evaluates iptables rule:                             2. eBPF program runs custom C code:
     -j MASQUERADE                                           cilium_nat_ipv4()
       │                                                       │
  3. Writes to Linux nf_conntrack table:                  3. Allocates ephemeral port from node's port range.
     * Takes a kernel global mutex lock!                       │
     * Risk: nf_conntrack table exhaustion                4. Overwrites Source IP: 10.244.1.25 -> 192.168.1.50
       │                                                       │
  4. Rewrites Source IP and recalculates IP/TCP           5. Recalculates TCP/IP Checksum using BPF helper:
     checksums.                                              bpf_l4_csum_replace()
       │                                                       │
       │                                                  6. Saves translation tuple in an in-kernel BPF Map:
       │                                                     BPF_MAP: `cilium_snat_v4_external`
       │                                                     (Lock-free per-CPU hash map! O(1) speed!)
       ▼                                                       ▼
 [ TRANSMITTED ON PHYSICAL ETHERNET ]                    [ TRANSMITTED ON PHYSICAL ETHERNET ]
```

---

## 3. Production Configuration: Enabling BPF Masquerade in Cilium

```yaml
# Helm configuration parameters for Cilium BPF Masquerade
bpf:
  masquerade: true
  hostRouting: true

# Specify outbound physical device to attach TC masquerade programs
devices:
  - eth0
  - ens5

# Enable full kube-proxy replacement to bypass iptables completely
kubeProxyReplacement: true
```

---

## 4. Diagnostics & Live Verification

```bash
# 1. Inspect live active BPF NAT mappings in kernel memory
cilium-dbg bpf nat list

# 2. Inspect active eBPF programs attached to TC hook
sudo tc filter show dev eth0 egress

# 3. Check for dropped packets in Cilium eBPF pipeline
cilium-dbg monitor --type drop
```

---

## 5. Kubernetes Networking Mastery: The Roadmap to 10/10

| Stage | Competency Area | Core Technical Test |
| :--- | :--- | :--- |
| **8.0 (Current)** | **Kernel Data Plane & Control Loop** | Netfilter hooks, eBPF socket bypass, pause container namespaces, conntrack race conditions, ALB controller discovery loop. |
| **9.0** | **AWS VPC CNI & Nitro Hardware Density** | ENI allocation limits per instance type, secondary IP math, Prefix Delegation (`/28` blocks), and custom pod subnets. |
| **9.5** | **Ingress Policies & Client IP Preservation** | `externalTrafficPolicy: Local` vs `Cluster`, hair-pinning traffic, asymmetric routing mitigation on NLBs. |
| **10.0** | **Wire-Speed BGP & Low-Level Driver Tuning** | Cilium native routing via BGP peering with Top-of-Rack (ToR) switches, XDP driver-level DDoS drops, NIC ring buffer tuning. |

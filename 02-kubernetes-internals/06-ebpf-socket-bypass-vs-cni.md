# Kubernetes Internals: eBPF Socket-Level Bypass vs. Traditional CNI Networking

## 1. Architectural Mental Model (The "Why" & "What")

In traditional Kubernetes CNI networking (Flannel, Calico in iptables mode, AWS VPC CNI), communication between two Pods on the same worker node must traverse the Linux TCP/IP network stack twice: building packets, traversing `veth` pairs, evaluating Netfilter chains (`iptables`), and allocating connection tracking (`nf_conntrack`) records.

**eBPF (Extended Berkeley Packet Filter) socket-level bypass** attaches kernel programs directly to the socket layer (`sock_ops` and `sk_msg`). For Pods co-located on the same node, eBPF redirects data payloads directly between socket queues in kernel memory, completely **bypassing the TCP/IP stack, Netfilter, and veth pairs**.

```
+---------------------------------------------------------------------------------------------------------------+
|                               TRADITIONAL CNI vs. eBPF SOCKET-LEVEL BYPASS                                    |
+---------------------------------------------------------------------------------------------------------------+

  POD A (Process 1)                                                     POD B (Process 2)
  Socket: fd=3                                                          Socket: fd=4
       |                                                                     ^
       | sendmsg()                                                           | recvmsg()
       v                                                                     |
=================================================================================================================
  PATH 1: TRADITIONAL LINUX CNI (Flannel / Calico iptables / AWS VPC CNI)                                        
=================================================================================================================
       |                                                                     ^
       v                                                                     |
 [ Kernel Socket Buffer: wmem ]                                        [ Kernel Socket Buffer: rmem ]
       |                                                                     ^
       v (TCP/IP Encapsulation: IP/TCP headers, Checksums)                   | (TCP/IP Decapsulation)
 [ Netfilter: OUTPUT / POSTROUTING ]                                   [ Netfilter: PREROUTING / INPUT ]
       |                                                                     ^
       v                                                                     |
 [ vethA (Pod namespace) ]                                             [ vethB (Pod namespace) ]
       |                                                                     ^
       v                                                                     |
 [ Host Linux Bridge / Routing Engine / conntrack ] -------------------------+
 (Traverses kernel networking stack TWICE - High CPU cache thrashing & latency)

=================================================================================================================
  PATH 2: eBPF SOCKET BYPASS (Cilium sock_ops + sk_msg)                                                         
=================================================================================================================
  POD A Socket (fd=3)                                                   POD B Socket (fd=4)
       |                                                                     ^
       | sendmsg()                                                           |
       v                                                                     |
  [ BPF: sock_ops ] ---> Intercepts connect() -> Maps: (SrcTuple <-> DstTuple) in `sock_hash` map
       |                                                                     ^
       |                                                                     |
       v                                                                     |
  [ BPF: sk_msg ] ------> bpf_msg_redirect_hash() ---------------------------+
  
  * ZERO TCP/IP header construction!
  * ZERO Netfilter / iptables evaluation!
  * ZERO veth pair traversal!
  * Data payload copied DIRECTLY from Pod A write buffer to Pod B read buffer!
```

---

## 2. Under the Hood (Mechanics & Internals)

### A. The Hook Points: `sock_ops` and `sk_msg`
1. **`BPF_PROG_TYPE_SOCK_OPS`:** Hooks into TCP state transitions (`BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB`).
   - When Pod A establishes a connection to Pod B, the BPF program intercepts the handshake.
   - It records the socket pointers (`struct sock *`) in an in-kernel hash map (**`BPF_MAP_TYPE_SOCKHASH`**).
2. **`BPF_PROG_TYPE_SK_MSG`:** Hooks into `sendmsg()` syscalls.
   - When Pod A writes to its socket, the BPF program matches the destination tuple against `sock_hash`.
   - It invokes the kernel helper `bpf_msg_redirect_hash()`, moving data directly into Pod B's receive buffer (`rmem`).

### B. Same-Node vs. Cross-Node Behavior
- **Same-Node Traffic:** Bypasses TCP/IP stack completely via socket memory redirect.
- **Cross-Node Traffic:** Cannot bypass the physical network. eBPF attaches at the driver/TC (Traffic Control) or XDP layer, performing direct routing or VXLAN/Geneve encapsulation without Netfilter/iptables.

---

## 3. Production Implementation: Cilium Helm Configuration

```yaml
# Production Cilium Helm values for eBPF Kube-Proxy Replacement & Socket LB
kubeProxyReplacement: true
k8sServiceHost: 10.0.1.10
k8sServicePort: 6443

bpf:
  masquerade: true
  hostRouting: true

socketLB:
  enabled: true
  hostNamespaceOnly: false # Enforces socket-level load balancing inside Pods

sockops:
  enabled: true # Enables sock_ops / sk_msg zero-stack socket bypass
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode: Sockmap Saturation
- **Symptom:** Microservices experience connection resets (`ECONNRESET`) or fall back to high-latency routing under massive concurrency.
- **Root Cause:** The `sock_hash` BPF map exceeds its maximum entry capacity.
- **Diagnostics:**
  ```bash
  # Check Cilium BPF drop metrics and program state
  cilium-dbg bpf metrics list
  cilium-dbg monitor --type drop

  # Inspect active sock_ops and sk_msg programs with bpftool
  sudo bpftool prog show --type sock_ops
  sudo bpftool prog show --type sk_msg
  sudo bpftool map list | grep -E 'sock_hash|sock_ops'
  ```

# Kubernetes Internals: CoreDNS, UDP Conntrack Race Conditions & The 5-Second DNS Penalty

## 1. Architectural Mental Model (The "Why" & "What")

In high-concurrency Kubernetes clusters, applications frequently encounter an elusive, intermittent issue: **outbound HTTP requests or database calls experience an unexplained, exact 5-second latency spike**, while CPU, memory, and CoreDNS pods show low utilization.

This anomaly is not caused by application code, CoreDNS saturation, or network congestion. It is an **in-kernel race condition inside the Linux Netfilter Connection Tracking (`nf_conntrack`) engine** triggered by parallel UDP DNS queries (`A` and `AAAA` records) multiplexed from the exact same socket and source port.

```
+---------------------------------------------------------------------------------------------------------------+
|                               THE LINUX KERNEL UDP CONNTRACK RACE CONDITION                                   |
+---------------------------------------------------------------------------------------------------------------+

  POD WORKLOAD (glibc getaddrinfo)
     Socket: UDP Source Port: 42150
     ├── Packet 1 (A Record query)    -----> Dst: 10.96.0.10:53 (CoreDNS ClusterIP)
     └── Packet 2 (AAAA Record query) -----> Dst: 10.96.0.10:53 (CoreDNS ClusterIP) [Concurrent]
                  |
                  | Both packets hit Host Kernel Netfilter PREROUTING at the exact same microsecond
                  v
+---------------------------------------------------------------------------------------------------------------+
| LINUX KERNEL: NETFILTER & NF_CONNTRACK RACE CONDITION                                                         |
|                                                                                                               |
|  Packet 1: Traverses KUBE-SERVICES -> DNAT applied to CoreDNS Pod A (10.244.0.5:53)                         |
|            Kernel attempts to insert into conntrack hash table:                                               |
|            [ORIG: 10.244.1.25:42150 -> 10.96.0.10:53 | REPL: 10.244.0.5:53 -> 10.244.1.25:42150]           |
|            -> Insertion Succeeded                                                                             |
|                                                                                                               |
|  Packet 2: Simultaneously traverses KUBE-SERVICES -> DNAT applied to CoreDNS Pod B (10.244.0.6:53)           |
|            Kernel attempts to insert conntrack entry for the exact same source tuple:                         |
|            Tuple conflict detected! conntrack hash lock collision:                                           |
|            -> KERNEL DROPS PACKET 2 (nf_conntrack: table collision / confirm failed)                          |
+---------------------------------------------------------------------------------------------------------------+
                  |
                  | Packet 2 never reaches CoreDNS!
                  v
  glibc DNS Resolver: Times out waiting for dropped AAAA response...
  Timer: Exactly 5.000 seconds (glibc default RES_TIMEOUT)
  Retransmission sent at t = 5.001s -> Finally resolves -> 5-Second Latency Spike on your API!
```

---

## 2. Under the Hood (Mechanics & Internals)

### A. The Lock Contention in `__nf_conntrack_confirm()`
1. Standard C libraries (`glibc`) issue `A` (IPv4) and `AAAA` (IPv6) lookups concurrently using the same UDP client socket.
2. In Kubernetes, the destination is a virtual Service IP (`10.96.0.10:53`).
3. If CoreDNS has multiple replicas, kube-proxy's probabilistic `statistic` rules can DNAT Packet 1 to CoreDNS Pod A and Packet 2 to CoreDNS Pod B.
4. When both packets exit via `POSTROUTING`, the kernel attempts to insert them into the connection tracking table via `__nf_conntrack_confirm()`.
5. Because both packets have the identical original 5-tuple (`Src IP`, `Src Port`, `Dst IP`, `Dst Port`, `Protocol UDP`), the kernel detects a tuple collision. The second packet is marked invalid and dropped silently in kernel space.

### B. The `ndots:5` Resolution Explosion
By default, Kubernetes injects `ndots:5` into `/etc/resolv.conf`:
```ini
nameserver 10.96.0.10
search production.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

If an application queries an external domain with fewer than 5 dots (e.g., `api.stripe.com` has 2 dots), the resolver first searches all suffix paths:
1. `api.stripe.com.production.svc.cluster.local` (A + AAAA = 2 queries)
2. `api.stripe.com.svc.cluster.local` (A + AAAA = 2 queries)
3. `api.stripe.com.cluster.local` (A + AAAA = 2 queries)
4. `api.stripe.com` (A + AAAA = 2 queries)

This generates **8 UDP packets in microsecond bursts from the same client port**, drastically increasing the probability of a conntrack race condition drop.

---

## 3. Production Solutions & Workarounds

### Solution 1: Deploy NodeLocal DNSCache (Architectural Fix)
NodeLocal DNSCache runs a DNS caching agent as a `DaemonSet` on every node on a link-local IP (`169.254.20.10`).
- Pods query their local node instance.
- NodeLocal DNSCache queries upstream CoreDNS over **TCP**, completely bypassing UDP conntrack race collisions.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    metadata:
      labels:
        k8s-app: node-local-dns
    spec:
      hostNetwork: true
      priorityClassName: system-node-critical
      containers:
        - name: node-cache
          image: registry.k8s.io/dns/k8s-dns-node-cache:1.22.28
          args:
            - -localip
            - 169.254.20.10
            - -conf
            - /etc/Corefile
            - -upstreamsvc
            - kube-dns-upstream
```

### Solution 2: Pod-Level `single-request-reopen` (Workload Fix)
Forces `glibc` to close the socket and open a fresh ephemeral port between the `A` and `AAAA` queries:

```yaml
spec:
  dnsConfig:
    options:
      - name: single-request-reopen
      - name: ndots
        value: "2"
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode: Identifying Conntrack Insert Collisions
- **Symptom:** Application traces show sporadic 5000ms latency on external API calls.
- **Verification Commands:**
  ```bash
  # Check kernel dropped conntrack packets due to collisions
  sudo conntrack -S | grep -E 'insert_failed|drop'

  # Trace dropped UDP packets on host interface
  sudo tcpdump -i any udp port 53 -nn -vv | grep -i "5.00"
  ```

# Kubernetes NetworkPolicies & eBPF Identity-Based Micro-Segmentation

## 1. Architectural Mental Model (The "Why" & "What")

By default, Kubernetes cluster networking is **completely flat and open**: every Pod in every Namespace can open a TCP or UDP socket to every other Pod in the cluster. Without micro-segmentation, a single RCE vulnerability in a low-privilege frontend Pod in `marketing` grants an attacker direct network reachability to scan and exploit databases in `payments`.

Kubernetes **`NetworkPolicy`** provides declarative L3/L4 isolation, while modern eBPF CNIs (such as **Cilium**) replace slow IP-based `iptables` chains with **$O(1)$ cryptographic Security Identities** enforced directly in kernel RAM.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                     IP-BASED FILTERING (LEGACY) vs. IDENTITY-BASED FILTERING (CILIUM eBPF)                  |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  LEGACY (Calico iptables / IPSet):
  • Every Pod IP (10.244.1.25) is written into kernel ipset/iptables chains on every node.
  • When 500 Pods autoscale or crash: Control plane must rewrite iptables/ipsets across ALL nodes!
  • IP Reuse Race: Pod-A dies, Pod-B gets 10.244.1.25; stale iptables rules briefly leak access!

  MODERN eBPF (Cilium Security Identity):
  ┌──────────────────────────────────────────────────────────┐
  │ 1. Cilium Control Plane assigns a 32-bit Numeric ID      │
  │    to each unique Pod Label Set:                         │
  │      labels: {app=frontend} ──► Identity ID: 10421       │
  │      labels: {app=payments} ──► Identity ID: 20884       │
  └──────────────────────────┬───────────────────────────────┘
                             │ Written ONCE into Kernel BPF Map (`cilium_policy_v2`)
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │ 2. In-Kernel BPF Policy Map (O(1) Hash Lookup)           │
  │    KEY: (SrcIdentity=10421, DstIdentity=20884, Port=8080)│
  │    VAL: ALLOW                                            │
  │                                                          │
  │ • If frontend scales from 3 to 300 Pods:                 │
  │   ZERO policy map updates needed! All 300 share ID 10421!│
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Under the Hood: Mechanics & Internals

### A. Zero-Trust Semantics: Default-Open Until Selected
1. **Purely Additive Allow-Lists:** Standard Kubernetes `NetworkPolicy` has **no `Deny` rules**. Multiple policies selecting the same Pod are unioned (OR-ed) together.
2. **The State Flip:** A Pod with zero matching `NetworkPolicy` allows 100% of traffic. The instant **one** `NetworkPolicy` selects that Pod for `Ingress` (or `Egress`), that direction flips to **Default Deny**—dropping all packets except those explicitly matched by an allow rule.
3. **CNI Enforcement Dependency:** `kube-apiserver` only stores `NetworkPolicy` YAML in `etcd`; it enforces zero packets. Your CNI plugin (Cilium, Calico, or AWS VPC CNI Network Policy Agent) must implement the dataplane rules.

### B. The 3 Classic `NetworkPolicy` Architectural Traps

#### Trap 1: The Egress DNS Blackout (CoreDNS Port 53)
Applying a `default-deny-egress` policy to a namespace immediately breaks every microservice's outbound calls—even to allowed targets—because DNS queries (`UDP/TCP 53`) to `kube-dns` in `kube-system` are blocked. Every `default-deny-egress` baseline **must** include an explicit scoped DNS allow rule.

#### Trap 2: The YAML Dash (`AND` vs. `OR` Logic)
In `ingress.from` and `egress.to`, a leading `-` starts a **new OR-ed list item**:
```yaml
# ❌ TWO DASHES = OR! Allows ANY pod in 'frontend' OR any pod with 'app: checkout' in the local namespace!
from:
- namespaceSelector:
    matchLabels: {kubernetes.io/metadata.name: frontend}
- podSelector:
    matchLabels: {app: checkout}

# ✅ ONE DASH = AND! Allows ONLY pods with 'app: checkout' INSIDE namespace 'frontend'!
from:
- namespaceSelector:
    matchLabels: {kubernetes.io/metadata.name: frontend}
  podSelector:
    matchLabels: {app: checkout}
```

#### Trap 3: Service `port` vs. Container `targetPort`
`NetworkPolicy` is enforced at the Pod's `veth`/eBPF endpoint **after** `cgroup/connect4` or `kube-proxy` DNAT has already translated `ClusterIP:80` to `PodIP:8080`. Specifying the Service port (`80`) instead of the container `targetPort` (`8080`) causes 100% packet drops.

---

## 3. Production Implementation: Zero-Trust Baseline + Scoped DNS & DB Access

```yaml
# 1. Default-Deny ALL Ingress & Egress in the 'payments' namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}                          # ──► Selects EVERY Pod in 'payments'
  policyTypes:
  - Ingress
  - Egress
---
# 2. Explicitly allow ONLY CoreDNS (UDP/TCP 53) in kube-system
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns                # ──► Tightly scoped to CoreDNS Pods only!
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# 3. Allow 'checkout-api' in 'frontend' namespace -> 'postgres' in 'payments' on 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-checkout-to-postgres
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: checkout-api                # ──► Single dash = AND condition!
    ports:
    - protocol: TCP
      port: 5432                           # ──► Must match container targetPort!
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: Silent DNS Failure After Applying Default-Deny Egress
* **Symptom:** Pods log `dial tcp: lookup orders-api.payments.svc.cluster.local: i/o timeout` immediately after deploying Egress policies.
* **Diagnostics & Fix:**
  ```bash
  # 1. Watch real-time eBPF policy drops in Cilium Hubble
  hubble observe --namespace payments --verdict DROPPED -f

  # 2. Inspect exact eBPF drop reason and numeric Security Identities on the node
  cilium-dbg monitor --type drop
  cilium-dbg identity list | grep "k8s-app=kube-dns"
  ```

### Failure Mode 2: `ipBlock` CIDR Matching Broken for Cluster-Internal IPs
* **Symptom:** An `egress.to.ipBlock` rule works for external APIs, but fails when targeting an AWS RDS endpoint over VPC Peering or a Pod IP because `kube-proxy` / eBPF SNAT or DNAT rewrites the address before or after policy evaluation.
* **Rule:** Use `podSelector` + `namespaceSelector` for all intra-cluster Pod-to-Pod traffic, and reserve `ipBlock` strictly for external VPC/Internet CIDRs (`except: ["10.0.0.0/8"]`).

---

## 5. Socratic Architectural Challenge

### Scenario:
You apply `default-deny-all` in `payments`, then write an `Ingress` `NetworkPolicy` allowing `frontend` Pods to reach `payments-api`. The Kubernetes `Service` for `payments-api` exposes `port: 80` and routes to `targetPort: 8080`. You configure `ports: [{ protocol: TCP, port: 80 }]` in the `NetworkPolicy`. Every single request from `frontend` times out.

### Question:
Why did allowing port `80` in `NetworkPolicy` drop all traffic sent to the Service on port `80`?

### Answer:
`NetworkPolicy` is enforced at the destination Pod's virtual interface (`veth`/eBPF endpoint) **after** `cgroup/connect4` or `kube-proxy` DNAT has already rewritten `ClusterIP:80` to `PodIP:8080`. The packet arriving at `payments-api` carries `Dst Port: 8080`, never `80`. `NetworkPolicy` must always specify the container's **`targetPort` (`8080`)**.

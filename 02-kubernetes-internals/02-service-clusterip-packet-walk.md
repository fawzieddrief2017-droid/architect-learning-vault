# Kubernetes Internals: Service ClusterIP Packet Walk & Netfilter Mechanics

## 1. Architectural Mental Model

A Kubernetes `ClusterIP` Service does not exist on any network card, has no MAC address, and does not respond to ARP requests. It is a **kernel-programmable abstraction**.

When a Pod initiates a connection to a Service IP (e.g., `10.96.0.100:80`):
1. Packets traverse the pod's `veth` pair onto the host node's network stack.
2. The host kernel's **Netfilter** framework intercepts the packet at the `PREROUTING` or `OUTPUT` hook.
3. Kube-proxy rules execute **Destination NAT (DNAT)**, rewriting `10.96.0.100:80` to a concrete target Pod IP (`10.244.2.14:8080`).
4. The packet is routed across the physical underlay fabric (via VXLAN overlay encapsulation or direct AWS VPC CNI routing).
5. On the return path, connection tracking (`nf_conntrack`) performs **Un-DNAT**, restoring the apparent source address back to the ClusterIP.

```
+---------------------------------------------------------------------------------------------------------------+
|                                    SERVICE PACKET WALK: CLIENT TO ENDPOINT                                    |
+---------------------------------------------------------------------------------------------------------------+

  CLIENT POD (10.244.1.5)
    Socket write() -> Dst: 10.96.0.100:80 (ClusterIP)
       |
       | veth0 -> host veth
       v
+---------------------------------------------------------------------------------------------------------------+
| HOST KERNEL: NETFILTER (Node 1)                                                                               |
|   1. PREROUTING / OUTPUT -> KUBE-SERVICES -> KUBE-SVC-XXX                                                     |
|   2. Statistic Module (Probabilistic Endpoint Selection):                                                    |
|      - Probability 0.50 -> KUBE-SEP-1 (10.244.1.9:8080 - Local Pod)                                          |
|      - Fallback       -> KUBE-SEP-2 (10.244.2.14:8080 - Remote Pod on Node 2)                                |
|   3. DNAT Applied: Dst IP rewritten to 10.244.2.14:8080                                                      |
|   4. nf_conntrack entry created for reverse translation                                                      |
+---------------------------------------------------------------------------------------------------------------+
       |
       | Routing Table -> Route via VXLAN / ENI interface
       v
+---------------------------------------------------------------------------------------------------------------+
| UNDERLAY NETWORK ENCAPSULATION                                                                                |
|   Outer UDP Packet: Src Node 1 (192.168.1.10) -> Dst Node 2 (192.168.1.20) [UDP Port 4789]                  |
|   Inner IP Packet:  Src Pod (10.244.1.5)      -> Dst Pod (10.244.2.14)                                       |
+---------------------------------------------------------------------------------------------------------------+
       |
       | Physical Wire Traversal
       v
+---------------------------------------------------------------------------------------------------------------+
| DESTINATION NODE (Node 2)                                                                                     |
|   1. Outer UDP decapsulation                                                                                  |
|   2. Inner packet delivered to target Pod veth interface                                                      |
|   3. Target application receives TCP SYN from 10.244.1.5                                                      |
+---------------------------------------------------------------------------------------------------------------+
```

---

## 2. Under the Hood: Netfilter Probabilistic Selection & Conntrack

### A. How Kube-Proxy Balances Traffic Without a Daemon Load Balancer
Kube-proxy injects rules using the Netfilter **`statistic --mode random`** module into the `nat` table:

```text
Chain KUBE-SVC-PAYMENT (1 references)
pkts bytes target          prot opt in out source    destination
   0     0 KUBE-SEP-AAAA   all  --  *  *   0.0.0.0/0 0.0.0.0/0  statistic mode random probability 0.3333333333
   0     0 KUBE-SEP-BBBB   all  --  *  *   0.0.0.0/0 0.0.0.0/0  statistic mode random probability 0.5000000000
   0     0 KUBE-SEP-CCCC   all  --  *  *   0.0.0.0/0 0.0.0.0/0  /* Fallback to 3rd endpoint */
```

### B. Connection Tracking (`nf_conntrack`) Lifecycle
Only the initial `TCP SYN` packet traverses the `KUBE-SVC` chain:
1. `TCP SYN`: DNAT occurs; entry recorded in kernel conntrack table.
2. `TCP ACK / DATA`: Netfilter identifies state `ESTABLISHED` and executes immediate address translation without evaluating rule chains.
3. `TCP SYN-ACK` (Return Path): Destination node delivers reply. Origin node kernel reads conntrack entry and applies **Un-DNAT**, rewriting source IP from real Pod IP back to the virtual `ClusterIP`.

---

## 3. Production Verification & Diagnostics

```bash
# 1. View active Kube-Proxy iptables service chains
sudo iptables -t nat -L KUBE-SERVICES -n -v | head -n 30

# 2. Inspect conntrack table utilization
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max

# 3. Dump active connection tracking entries for a specific Pod
sudo conntrack -L -s 10.244.1.5
```

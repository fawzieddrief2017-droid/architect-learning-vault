# Kubernetes Internals: Service Discovery vs. In-Kernel Packet Routing

## 1. Architectural Mental Model (The "Why" & "What")

A core architectural puzzle in Kubernetes systems engineering is understanding the division of responsibilities between **CoreDNS (Layer 7 Service Discovery)** and **kube-proxy (Layer 4 In-Kernel Packet Routing)**:

*Why does a cluster require internal DNS if `kube-proxy` already dynamically synchronizes and routes all active Pod IPs inside the Linux kernel?*

The distinction is clear:
1. **Application Sockets Require Numeric Addresses:** Application runtimes (Python, Go, Java) cannot open a raw socket to an ASCII name like `payment-service`. The operating system's network stack requires a concrete Layer 3 IP address to construct the initial IP header and TCP `SYN` packet.
2. **`kube-proxy` is a Packet Router, Not a Directory:** `kube-proxy` programs Netfilter (`iptables`), `IPVS`, or eBPF to rewrite virtual `ClusterIP`s to backend `PodIP`s. It has no mechanism to inform the application *which* virtual IP to target.
3. **CoreDNS is the Directory:** CoreDNS resolves the symbolic service name (`payment-service.default.svc.cluster.local`) into its corresponding virtual `ClusterIP` or direct Pod IPs (for Headless Services).

```
+---------------------------------------------------------------------------------------------------+
| Application Code (Layer 7)                                                                        |
| Needs an address to initiate a network connection:                                                |
| "Where is payment-service?"                                                                       |
+---------------------------------+-----------------------------------------------------------------+
                                  |
                   [DNS Lookup / CoreDNS (Port 53 UDP/TCP)]
                                  |
                                  v
+---------------------------------------------------------------------------------------------------+
| Operating System Kernel / Network Socket (Layer 4)                                                |
| Receives IP address (e.g., Virtual ClusterIP: 10.96.0.45:80)                                       |
+---------------------------------+-----------------------------------------------------------------+
                                  |
                     [kube-proxy (Netfilter / IPVS / eBPF)]
                                  |
                                  v
+---------------------------------------------------------------------------------------------------+
| Packet Destination Rewritten in Kernel RAM (DNAT)                                                 |
| Dst: 10.96.0.45:80  ──>  Dst: 10.244.1.15:8080 (Target Backend Pod IP)                            |
+---------------------------------------------------------------------------------------------------+
```

---

## 2. Under the Hood: The Control Plane Watch Graph

Neither CoreDNS nor `kube-proxy` actively polls worker nodes or scans the physical network. They participate in an event-driven **HTTP/2 Long-Polling Watch Graph** centered on `kube-apiserver`.

```
                    +------------------------------------+
                    |        kubelet (Worker Node)       |
                    +-----------------+------------------+
                                      |
                                      | 1. Reports Pod IP & Readiness Probe status
                                      v
                    +------------------------------------+
                    |           kube-apiserver           | <==== etcd (Cluster State Store)
                    +-----------------+------------------+
                                      |
                                      | 2. Pushes Pod/Service state transitions
                                      v
                    +------------------------------------+
                    |      EndpointSliceController       |
                    |   (kube-controller-manager loop)   |
                    +-----------------+------------------+
                                      |
                                      | 3. Generates chunked EndpointSlice objects (100 endpoints/slice)
                                      v
+-------------------------------------+-------------------------------------+
|                                                                           |
| 4. Streaming HTTP/2 Watch                                                 | 4. Streaming HTTP/2 Watch
v                                                                           v
+----------------------------------+       +----------------------------------+
|           CoreDNS Pod            |       |     kube-proxy (Worker Node)     |
|                                  |       |                                  |
| Maintains in-memory radix tree   |       | Programs host Netfilter/IPVS     |
| Maps: svc name -> ClusterIP      |       | Chains: ClusterIP -> Pod IPs     |
| Maps: headless -> Pod IPs        |       | Evaluates: statistic random DNAT |
+----------------------------------+       +----------------------------------+
```

### The Step-by-Step Lifecycle
1. **Pod Readiness Detection:** When a container passes its readiness probe, the local node's `kubelet` updates the Pod status (`status.conditions[Type=Ready, Status=True]`) and sends a `PATCH` request to `kube-apiserver`.
2. **EndpointSlice Assembly:** The `EndpointSliceController` detects the status mutation via its informer cache. It packs the healthy `PodIP:Port` pair into an `EndpointSlice` object.
3. **CoreDNS Cache Mutation:** The CoreDNS `kubernetes` plugin receives the event over its persistent Watch connection and atomically updates its internal lookup tables in memory. Live DNS resolutions read purely from RAM without querying `etcd`.
4. **Data Plane Programming:** Simultaneously, `kube-proxy` on every worker node receives the `EndpointSlice` update and modifies the kernel's `iptables` chains or IPVS sets.

---

## 3. When Internal DNS Is Bypassed: North-South Ingress

A common architectural oversight is assuming CoreDNS is involved in every Kubernetes request. **For external inbound (North-South) traffic, internal cluster DNS is completely bypassed:**

### Scenario A: External Load Balancer in `ip` Mode (AWS ALB via AWS Load Balancer Controller)
1. The cloud controller watches `EndpointSlices` directly from `kube-apiserver`.
2. The Load Balancer Target Group directly stores the current Pod IPs (`10.0.x.x`).
3. External client packets travel from the ALB directly to the Pod's secondary ENI over the VPC underlay.
4. **CoreDNS Involvement:** **Zero.**
5. **kube-proxy Involvement:** **Zero.**

### Scenario B: External Load Balancer in `instance` (NodePort) Mode
1. The Load Balancer targets the worker node's physical IP on a high static port (`30000–32767`).
2. The node's Netfilter stack intercepts the packet at the `PREROUTING` hook and executes DNAT to the backend Pod IP.
3. **CoreDNS Involvement:** **Zero.**
4. **kube-proxy Involvement:** **Mandatory** (maintains the `NodePort` DNAT rules).

---

## 4. When Internal DNS Is Mandatory: East-West Workloads

Internal DNS is strictly required for workloads initiating connections from within the cluster:

1. **Standard ClusterIP Discovery:** Decouples clients from dynamic IP reassignments. If a Service is recreated or redeployed, it receives a new `ClusterIP`. DNS name resolution (`http://payment.production.svc.cluster.local`) isolates client code from this churn.
2. **Headless Services for Distributed State (`clusterIP: None`):** 
   Stateful applications (Kafka, Cassandra, ZooKeeper, MongoDB replica sets) require point-to-point communication with specific cluster peers (e.g., `kafka-0` replicating data to `kafka-1`), rather than hitting an anonymous random load balancer.
   - For Headless Services, `kube-proxy` does **nothing** because no virtual ClusterIP exists.
   - CoreDNS resolves direct individual `A`/`AAAA` records:
     ```text
     kafka-1.kafka-service.default.svc.cluster.local -> 10.244.2.14
     ```
3. **Namespace Isolation:** CoreDNS enables multi-tenant environment scoping: a query to `database` from the `staging` namespace automatically resolves to the local `database.staging.svc.cluster.local`, while the same query in `production` resolves to `database.production.svc.cluster.local`.
4. **Dynamic Port Discovery (RFC 2782 SRV Records):** Applications querying `_http._tcp.backend.default.svc.cluster.local` discover both the active port and target hostname dynamically.

---

## 5. Architectural Summary: Discovery vs. Routing

| Component | Network Layer | Core Responsibility | Failure Mode When Stalled |
| :--- | :--- | :--- | :--- |
| **CoreDNS** | Layer 7 (Application / Discovery) | Resolves human-readable service names to `ClusterIP`s or individual Pod IPs. | `i/o timeout` during DNS lookups; application cannot resolve destination address. |
| **`kube-proxy`** | Layer 4 (Transport / Kernel Routing) | Programs in-kernel Netfilter / IPVS / eBPF rules to translate virtual `ClusterIP`s to active `PodIP`s. | `No route to host` or connection timed out after DNS resolution succeeds. |
| **AWS LB Controller** | Cloud Control Plane | Watches `EndpointSlices` and syncs concrete Pod IPs into AWS Target Groups. | `502 Bad Gateway` or `504 Gateway Timeout` on ingress while internal traffic works normally. |

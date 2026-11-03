# Kubernetes Internals: Complete End-to-End Ingress Packet Walk

## 1. Architectural Mental Model (The "Why" & "What")

When an internet user accesses a containerized application running in an enterprise Kubernetes cluster (such as AWS EKS), the request does not hit the container directly. It traverses a coordinated multi-tier data plane bridging **Cloud Edge Load Balancing**, **Container Orchestrator Control Planes**, and **Linux Kernel Host Networking**.

The critical architectural insight is understanding the difference between **Legacy Instance Mode** (double-hop routing through `NodePort` and `kube-proxy`) and **Modern IP Mode** (direct Pod targeting via cloud hypervisor ENI routing).

```
+---------------------------------------------------------------------------------------------------------------+
|                                    COMPLETE END-TO-END INGRESS PACKET WALK                                    |
+---------------------------------------------------------------------------------------------------------------+

  CLIENT BROWSER (Internet: 203.0.113.50)
    │
    │  1. DNS Resolution (Amazon Route 53) -> Resolves to public ALB Anycast IPs
    │  2. TLS Handshake & HTTP Request (e.g., GET /api/v1/checkout)
    ▼
  [ AWS APPLICATION LOAD BALANCER (ALB) ]
    │
    │  * Terminates TLS; evaluates Layer 7 routing rules (host/path)
    │  * Queries active Target Group populated by AWS Load Balancer Controller
    │
    ├───► [ MODE A: Instance Mode (Legacy NodePort) ]
    │        ├── Sends to Worker Node Private IP on static NodePort (192.168.1.50:31245)
    │        └── Node Netfilter / kube-proxy executes DNAT to Pod IP (Risk: Inter-node extra hop!)
    │
    └───► [ MODE B: IP Mode (AWS VPC CNI - Production Recommended) ]
             └── Sends DIRECTLY to Pod Secondary Private IP (10.0.2.85:8080)
                 (Bypasses NodePort, kube-proxy, and inter-node network hops completely!)
    ▼
=================================================================================================================
  AWS VPC NETWORK FABRIC (AWS Nitro System: Hardware offloaded via PCIe)
=================================================================================================================
    │
    │  Packet delivered directly to Worker Node's Elastic Network Interface (ENI)
    ▼
  [ WORKER NODE: PHYSICAL NIC (ENA DRIVER) ]
    │
    ├───► [ 1. eBPF: XDP Hook ] 
    │        └── (If DDoS filter active -> Evaluates here at driver speed. Else -> Pass)
    ▼
  [ Linux Kernel: alloc_skb() & Driver Ring Buffer ]
    │
    ├───► [ 2. eBPF: TC Ingress Hook (cls_bpf) ]
    │        └── (If Cilium active -> Evaluates BPF security identity & NetworkPolicies)
    ▼
  [ Linux Kernel: Netfilter PREROUTING ]
    │
    │  * In "IP Mode": Destination is ALREADY 10.0.2.85. No DNAT required!
    │  * conntrack records translation state: (ALB Private IP -> Pod IP)
    ▼
  [ Kernel Routing Decision (fib_lookup) ]
    │
    │  * Destination 10.0.2.85 matches host route table: "10.0.2.85 dev vethXXXX"
    ▼
  [ Container veth Pair (vethXXXX -> eth0 inside Pod Sandbox) ]
    │
    ▼
  [ POD NETWORK NAMESPACE: eth0 (10.0.2.85) ]
    │
    ▼
  [ CONTAINER APPLICATION SOCKET (e.g., Python / Go Web Server) ]
    │
    └── read(fd, buffer) -> Application executes business logic -> 200 OK Response generated!
```

---

## 2. Under the Hood: Target Group Modes Compared

### A. Instance Mode vs. IP Mode Data Paths

| Architectural Dimension | Target Group `instance` Mode | Target Group `ip` Mode |
| :--- | :--- | :--- |
| **Ingress Controller** | Ingress NGINX / Cloud Controller | AWS Load Balancer Controller |
| **Target Address** | Worker Node IP + `NodePort` (30000-32767) | Concrete Pod Secondary ENI IP |
| **Kube-Proxy Role** | **Mandatory**. `iptables`/IPVS DNAT translates NodePort to Pod IP. | **Bypassed completely** for inbound ingress. |
| **Network Hop Efficiency** | **Double Hop**: ALB -> Node 1 -> (if pod not on Node 1) -> Node 2. | **Single Hop**: ALB -> Target Node containing the Pod. |
| **Client IP Visibility** | Requires Proxy Protocol or `externalTrafficPolicy: Local`. | Preserved in `X-Forwarded-For` HTTP header by ALB. |

### B. The Dynamic Discovery Control Loop
How does the ALB know the Pod's dynamic private IP?
1. The **AWS Load Balancer Controller** runs as a Deployment inside the cluster.
2. It registers a `SharedInformer` watching `EndpointSlices` from `kube-apiserver`.
3. When a Deployment scales up or down, the controller detects the IP delta.
4. The controller executes AWS API calls:
   - `elasticloadbalancing:RegisterTargets`
   - `elasticloadbalancing:DeregisterTargets`
5. The ALB Target Group is updated dynamically in AWS VPC Hyperplane without restarting or reconfiguring the Load Balancer.

---

## 3. Production Implementation: Hardened AWS Ingress in IP Mode

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ingress
  namespace: production
  annotations:
    # Enforces ALB provisioning via AWS Load Balancer Controller
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    # CRITICAL: "ip" mode routes directly to Pod secondary ENIs (bypasses kube-proxy)
    alb.ingress.kubernetes.io/target-type: ip
    # Enforce TLS 1.3 modern cipher suite
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    # Health check path tuning for fast failover
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "10"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
spec:
  rules:
    - host: api.prod.internal.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 8080
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: Target Registration Delay (`504 Gateway Timeout`)
- **Symptom:** During rolling deployments, clients experience intermittent `502 Bad Gateway` or `504 Gateway Timeout` for 30–60 seconds.
- **Root Cause:** A terminating Pod is killed before the AWS Load Balancer finishes deregistering it from the Target Group. The ALB continues sending live requests to a dead Pod IP.
- **Remediation:** Configure `deregistration_delay.timeout_seconds` on the Target Group and add a `preStop` hook inside the container lifecycle to delay `SIGTERM` until deregistration completes:
  ```yaml
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 20"]
  ```

---

## 5. Socratic Architectural Challenge

> **Scenario:**
> You deploy an AWS Network Load Balancer (NLB) in front of an EKS cluster running with default CNI in `instance` mode. You observe that backend application audit logs record every incoming request as originating from the **Worker Node's private IP (`192.168.1.50`)**, completely obscuring the real client public IP addresses (`203.0.113.x`).
>
> 1. Which Kubernetes Service parameter causes this SNAT behavior inside the node's Netfilter stack?
> 2. What architectural trade-off occurs if you modify that parameter to preserve client IP addresses?

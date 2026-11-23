# Pod Lifecycle, Health Probes & Zero-Downtime Draining

## 1. Architectural Mental Model (The "Why" & "What")

During Kubernetes rolling updates, uncoordinated pod terminations trigger transient HTTP 502 (Bad Gateway), HTTP 504 (Gateway Timeout), and TCP connection resets (`ECONNRESET`). This occurs because pod provisioning and decommissioning are **asynchronous, multi-path distributed state reconciliations**.

Two decoupled control loops run concurrently across distinct cluster subsystems:
1. **The Ingress/Routing Path (Control Plane -> Data Plane):** EndpointSlice controller detects deletion, updates `EndpointSlice` resources, triggers `kube-proxy` iptables/IPVS updates on worker nodes, and signals the AWS Load Balancer Controller to deregister targets from the AWS Application Load Balancer (ALB).
2. **The Execution Path (Node Engine):** The node-local `kubelet` observes deletion, executes the `preStop` hook, delivers `SIGTERM` to the container process (PID 1), waits for `terminationGracePeriodSeconds`, and issues `SIGKILL`.

Because the node execution path operates independently from routing reconciliation, a race condition emerges: the application receives `SIGTERM` and closes its listening socket **before** external proxies and host routing tables purge the pod IP. In-flight requests and new TCP connections directed to the dying pod hit a closed socket, resulting in dropped packets.

```
+----------------------------------------------------------------------------------------------------+
| Part 1: Startup & Readiness Gate Timeline                                                          |
+----------------------------------------------------------------------------------------------------+
 Container:     [OCI Create/Start] ──> [InitContainers] ──> [App Process (PID 1)]
                                                                    │
 StartupProbe:  [Fail] ──> [Fail] ──> [Success] ────────────────────┼─────────────────> (Disabled)
                                             │                      │
 ReadinessProbe:                             └──────────> [Fail] ──> [Success] ───────> (Continuous)
                                                                          │
 Conditions:                                                 ContainersReady: True
                                                                          │
 ReadinessGate: [target-health.elbv2.k8s.aws: Initializing] ─────────> [Target Healthy]
                                                                                  │
 Pod Status:                                                              PodReady: True
                                                                                  │
 EndpointSlice: [Excluded from Endpoints] ────────────────────────────────> [IP:Port Added]
                                                                                  │
 Data Plane:    [ALB / kube-proxy: NO TRAFFIC] ───────────────────────────> [TRAFFIC DIRECTED]

+----------------------------------------------------------------------------------------------------+
| Part 2: Asynchronous Termination Race (Path A vs Path B)                                          |
+----------------------------------------------------------------------------------------------------+
                               API SERVER: metadata.deletionTimestamp set
                                              │
         ┌────────────────────────────────────┴────────────────────────────────────┐
         ▼                                                                         ▼
  [ PATH A: Execution Path (Kubelet) ]                      [ PATH B: Routing Path (Controllers) ]
  Kubelet begins local teardown                             EndpointSlice controller observes deletion
         │                                                                         │
         ├─> 1. preStop Hook Executed (e.g. sleep 15s)                             ├─> 1. EndpointSlice updated
         │      App continues servicing in-flight traffic                          │
         │                                                                         ├─> 2. kube-proxy updates iptables/IPVS
         │      ◄───────────── CRITICAL DRAIN WINDOW ─────────────►                │
         │      (Time required for Path B to propagate)                            ├─> 3. AWS LB Controller calls
         │                                                                         │      DeregisterTargets API
         │                                                                         │
         │                                                                         ├─> 4. AWS ALB marks target draining
         │                                                                         ▼
         ├─> 2. preStop hook completes                                     [ Routing Purged ]
         │                                                                 No new connections routed to Pod
         ├─> 3. SIGTERM sent to PID 1
         │      App stops accept(), drains in-flight sockets
         │
         ├─> 4. terminationGracePeriodSeconds expires
         ▼
     [ SIGKILL ] (Hard process abort)

  RACE FAILURE WITHOUT preStop:
  Without preStop, Step A3 (SIGTERM / socket closure) occurs in ~50ms.
  Path B requires 2s to 20s across kube-proxy and AWS ALB Target Groups.
  Result: In-flight and new packets hit a closed socket -> TCP RST / HTTP 502 Bad Gateway.
```

---

## 2. Under the Hood (Mechanics & Internals)

### 2.1 The Three Orthogonal Pod State Axes

A Pod's operational state is governed by three independent API axes:
1. **`.status.phase` (High-Level State Machine):** `Pending`, `Running`, `Succeeded`, `Failed`, or `Unknown`. Summarizes scheduling and container existence, not traffic readiness. A `Running` Pod may be unready to accept requests.
2. **`.status.conditions[]` (Fine-Grained Capabilities):** Array of conditions: `PodScheduled`, `Initialized`, `ContainersReady`, and `Ready`. The `Ready` condition controls traffic dispatch. External controllers inject custom gates via `spec.readinessGates` (e.g., `target-health.elbv2.k8s.aws`). The Pod's `Ready` condition remains `False` until all declared gates evaluate to `True`.
3. **`.metadata.deletionTimestamp` (Teardown Marker):** When deletion is initiated, the API server populates this RFC3339 timestamp and sets `metadata.deletionGracePeriodSeconds`.

> **Architectural Reality:** `Terminating` is **not** a Pod phase. In the core Kubernetes API, `.status.phase` remains `Running`. `kubectl` synthesizes the `Terminating` string client-side when `.metadata.deletionTimestamp != nil`.

### 2.2 Kubelet Prober Engine Architecture

The `kubelet` prober engine (`pkg/kubelet/prober/prober.go`) runs dedicated goroutines per probe type:

- **Startup vs. Readiness Dependency Split:**
  `initContainers` run sequentially to completion before app containers start. For slow-booting applications, `startupProbe` absorbs initialization delay. While `startupProbe` runs, `livenessProbe` and `readinessProbe` execution is **suspended**. Once `startupProbe` succeeds, `readinessProbe` takes over evaluation of the `Ready` condition.
- **Fail-Open Endpoint Cushioning:**
  If all replicas fail their `readinessProbe`, core `kube-proxy` empties the iptables/IPVS destination chains, causing connection drops. However, ingress controllers and service meshes (e.g., Envoy, ALB) can be configured with fail-open policies to cushion the outage by routing across unready pods rather than blackholing 100% of traffic.
- **Native gRPC Probes:**
  Configured via `spec.containers[*].readinessProbe.grpc` (Kubernetes v1.24+ GA). The `kubelet` establishes an in-process TCP connection speaking the gRPC Health Checking Protocol (`grpc.health.v1.Health/Check`) directly, avoiding container runtime process execution overhead.
- **Exec Probe Math & PLEG Saturation:**
  When `exec` probes are configured (`exec: command: ["/bin/healthcheck"]`), `kubelet` invokes the Container Runtime Interface (CRI) `ExecSync` RPC:
  $$\text{Kubelet} \xrightarrow{\text{ExecSync}} \text{containerd} \xrightarrow{\text{shim}} \text{runc exec} \xrightarrow{\text{kernel}} \text{clone()} \to \text{setns()} \to \text{execve()}$$
  If a worker node hosts 50 pods with 1-second exec probes:
  $$\text{Fork Velocity} = \frac{50 \text{ pods}}{1 \text{ second}} = 50 \text{ ExecSync calls / second}$$
  Process table contention and cgroup synchronization stall the Pod Lifecycle Event Generator (PLEG) relist loop. When PLEG relist latency exceeds 3 minutes, `kubelet` reports `NodeNotReady` (`PLEG is not healthy`), triggering cluster-wide node evictions.
  
  *(For container namespace isolation, PID 1 `SIGNAL_UNKILLABLE`, and zombie process reaping, see [Container Runtime Engine Architect Guide](./09-container-runtime-engine-architect-guide.md)).*

### 2.3 Graceful Teardown Timing & Race Resolution

To ensure zero downtime, termination timing must satisfy:

$$T_{\text{grace}} > T_{\text{preStop}} + T_{\text{in-flight}} + T_{\text{buffer}}$$

Where:
- $T_{\text{grace}}$: `spec.terminationGracePeriodSeconds` (default: 30s).
- $T_{\text{preStop}}$: Duration of the `preStop` hook or native sleep.
- $T_{\text{in-flight}}$: Maximum duration of active requests.
- $T_{\text{buffer}}$: Kernel socket teardown and process exit headroom ($\ge 5\text{s}$).

#### East-West vs. North-South Propagation Realities
- **East-West (ClusterIP via `kube-proxy`):**
  Updating host iptables/IPVS tables across nodes takes **1.5s to 5s**. A $T_{\text{preStop}}$ of **5s** prevents dropped East-West packets *(see [Service Routing and Internal DNS](./10-service-routing-and-internal-dns.md))*.
- **North-South (AWS ALB via Target Groups):**
  AWS Load Balancer Controller watches `EndpointSlice` deletions and calls `DeregisterTargets`. ALB infrastructure requires **15s to 20s** to propagate state across availability zones and transition targets to `draining`. A $T_{\text{preStop}}$ of **15s to 20s** is required for North-South ingress paths *(see [End-to-End Ingress Packet Walk](./07-end-to-end-ingress-packet-walk.md))*.

#### Native `lifecycle.preStop.sleep.seconds` in v1.30+ GA
Previously, operators used shell hooks (`/bin/sh -c "sleep 15"`). In distroless or scratch containers lacking `/bin/sh`, this causes `FailedPreStopHook` and triggers instant `SIGTERM`. Kubernetes v1.30 introduced native sleeping in `preStop` (GA):
```yaml
lifecycle:
  preStop:
    sleep:
      seconds: 15
```
The `kubelet` pauses internally without spawning an in-container process.

#### Layer 4 vs. Layer 7 Connection State Handling
- **L4 `ctstate NEW`**: New TCP `SYN` packets evaluate against Netfilter/iptables rules. Once the pod IP is removed from the DNAT chain, new connections redirect to remaining healthy pods.
- **L7 `nf_conntrack ctstate ESTABLISHED` (Keep-Alive & HTTP/2)**:
  Existing persistent TCP connections match entries in Linux `nf_conntrack` and bypass DNAT rules. Netfilter continues routing packets on established connections to the terminating pod *(see [Service ClusterIP Packet Walk](./02-service-clusterip-packet-walk.md))*.
  
  Applications serving HTTP/1.1 keep-alive or HTTP/2 must handle `SIGTERM` by:
  1. Stopping `accept()` on listening sockets.
  2. Setting `Connection: close` on HTTP/1.1 responses.
  3. Emitting HTTP/2 `GOAWAY` frames (`NO_ERROR`) to redirect clients to alternate pods while draining active streams.

---

## 3. Production Implementation (Code / CLI / Manifests)

### 3.1 CLI Discovery: Schema Inspection

Verify resource fields and API parameters directly:

```bash
# Inspect native lifecycle sleep and exec options
kubectl explain pod.spec.containers.lifecycle.preStop

# Inspect readiness gates syntax for external load balancers
kubectl explain pod.spec.readinessGates

# Inspect rolling update deployment parameters
kubectl explain deployment.spec.strategy.rollingUpdate
```

### 3.2 Annotated Zero-Downtime Deployment Manifest

This production manifest is configured for an EKS cluster with AWS ALB in TargetGroup IP mode and a distroless runtime:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-gateway
  namespace: production
  labels:
    app.kubernetes.io/name: order-gateway
spec:
  replicas: 6
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # Create replacement pods before terminating old ones
      maxSurge: 25%      # Bound surge to prevent node and IP pool exhaustion
  minReadySeconds: 15    # Pod must stay healthy 15s before rollout advances
  progressDeadlineSeconds: 600
  selector:
    matchLabels:
      app.kubernetes.io/name: order-gateway
  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-gateway
    spec:
      # Satisfies: 60s > 15s (preStop) + 30s (HTTP drain) + 15s (buffer)
      terminationGracePeriodSeconds: 60

      # Blocks PodReady until AWS ALB target health checks pass
      readinessGates:
        - conditionType: "target-health.elbv2.k8s.aws"

      containers:
        - name: gateway
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/order-gw:v3.2.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          # Native sleep hook (Kubernetes v1.30+ GA)
          # Delays SIGTERM by 15s so EndpointSlice & ALB deregistration finish
          lifecycle:
            preStop:
              sleep:
                seconds: 15

          # Startup Probe: Handles runtime JIT warmup and cache priming
          startupProbe:
            httpGet:
              path: /healthz/startup
              port: http
            initialDelaySeconds: 2
            periodSeconds: 3
            timeoutSeconds: 2
            successThreshold: 1
            failureThreshold: 20  # 20 * 3s = 60s startup ceiling

          # Readiness Probe: Governs EndpointSlice inclusion (local health ONLY)
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            successThreshold: 1
            failureThreshold: 2   # 2 failures (10s) to withdraw from endpoints

          # Liveness Probe: Detects unrecoverable deadlocks
          livenessProbe:
            httpGet:
              path: /healthz/liveness
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3   # 3 failures (30s) before container restart

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1024Mi"
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### 4.1 Downstream Dependency Checking in Readiness Probes (Cascading Outage)

#### The Failure Mechanism
Wiring downstream database (`SELECT 1`) or cache health into `/healthz/ready` creates an architectural vulnerability:
1. Database latency spikes or connection pool exhausts.
2. Every replica fails its readiness probe simultaneously.
3. The EndpointSlice controller removes all pod IPs from Service endpoints.
4. The service returns 100% HTTP 503 / connection errors to clients.
5. When the database recovers, all pods become ready at once, hitting the database with a thundering herd that re-triggers the outage.

#### SRE Rule
Readiness probes must evaluate **only node-local readiness** (listening socket bound, local caches initialized). Downstream dependencies must be isolated via application-level circuit breakers, timeouts, and connection pools.

### 4.2 `FailedPreStopHook` on Distroless Containers

When using distroless images without a shell, executing `/bin/sh -c "sleep 15"` fails immediately:

```bash
$ kubectl describe pod order-gateway-7484df7c4-v8b2q
Events:
  Warning  FailedPreStopHook  10s  kubelet  Exec lifecycle hook ([/bin/sh -c sleep 15]) in container "gateway" failed - error: container not running or exec failed: exec: "/bin/sh": executable file not found in $PATH
```

**Impact:** `kubelet` aborts the `preStop` hook immediately and sends `SIGTERM` at $T=0$, leaving the 15–20s ALB target deregistration window unbuffered.

**Remediation:** Use native `lifecycle.preStop.sleep.seconds: 15` (Kubernetes v1.30+), or compile a static binary (`/sleep`) on older versions.

### 4.3 Grace-Period Exhaustion: Exit 137 vs. Exit 143

Inspect container termination status:

$$\text{Exit Code} = 128 + \text{Signal Number}$$

- **Exit Code 143 ($128 + 15 = \text{SIGTERM}$):**
  The container handled `SIGTERM`, drained connections, and exited cleanly before the grace period expired.
- **Exit Code 137 ($128 + 9 = \text{SIGKILL}$):**
  The container exceeded `terminationGracePeriodSeconds` and was forcefully killed by the kernel, or was terminated by the Linux OOM killer.

```bash
# Inspect container exit codes and termination reasons across pods
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\tExitCode:"}{.status.containerStatuses[*].state.terminated.exitCode}{"\tReason:"}{.status.containerStatuses[*].state.terminated.reason}{"\n"}{end}'
```

If pods exit with code 137 and reason `Completed`, $T_{\text{grace}} \le T_{\text{preStop}} + T_{\text{in-flight}}$. Increase `terminationGracePeriodSeconds`.

### 4.4 Exec Probe Fork Storms & PLEG Saturation

Frequent `exec` probes cause runtime saturation on high-density nodes.

```bash
# Check kubelet journal for PLEG latency warnings
journalctl -u kubelet -n 100 --no-pager | grep -E "PLEG is not healthy|skipping pod synchronization"

# Query Prometheus for PLEG relist duration metric (Threshold: > 10s)
kubelet_pleg_relist_duration_seconds_bucket{le="10"}

# Correlate with CloudWatch ALB Target Group drop metrics:
# HTTPCode_Target_5XX_Count, HTTPCode_ELB_502_Count
```

Replace `exec` probes with native `httpGet` or `grpc` health checks.

### 4.5 Host Socket Backlog & Network Triage

When pods drop connections during rollouts, determine whether the Linux kernel listen queue or application worker pool is saturated:

```bash
# Inspect listen socket backlog states (-t: TCP, -l: Listening, -p: Process, -n: Numeric)
ss -tlpn
```

On sockets in `LISTEN` state:
- **`Send-Q`**: Maximum listen backlog queue depth (`backlog` parameter in `listen(2)`, capped by `net.core.somaxconn`).
- **`Recv-Q`**: Number of established TCP connections waiting for application `accept4(2)`.
- If $\text{Recv-Q} > \text{Send-Q}$, the application cannot accept connections fast enough. Incoming TCP `SYN` packets are dropped.

```bash
# Check cumulative listen drops and overflows via netstat counters
nstat -az TcpExtListenDrops TcpExtListenOverflows

# Apply production kernel socket tuning via /etc/sysctl.d/99-k8s-networking.conf:
# sysctl -w net.core.somaxconn=4096
# sysctl -w net.ipv4.tcp_max_syn_backlog=8192
```

---

## 5. Socratic Architectural Challenge

### The Scenario
You are auditing an EKS deployment for a financial transaction service written in Go. The service communicates over **HTTP/2** and runs on **Distroless** container images (`gcr.io/distroless/static-debian12:nonroot`). Traffic is routed through an AWS Application Load Balancer using **TargetGroup IP mode** (ALB routes directly to Pod ENI secondary IPs via AWS VPC CNI).

During rolling deployments, clients experience sporadic **HTTP 502 Bad Gateway** errors and dropped connections. The deployment manifest includes:

```yaml
spec:
  terminationGracePeriodSeconds: 20
  containers:
    - name: ledger
      image: gcr.io/distroless/static-debian12:nonroot
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sleep", "20"]
```

The team explains: *"We added a 20-second preStop sleep and set terminationGracePeriodSeconds to 20s. AWS ALB target deregistration takes ~15 seconds, so a 20-second sleep should prevent any race conditions. Why are we still seeing HTTP 502 errors during rollouts?"*

### Root Cause Analysis & Architectural Audit

Four cascading flaws cause the HTTP 502 errors:

#### 1. Distroless `FailedPreStopHook` Bypass
`gcr.io/distroless/static-debian12` contains no shell or coreutils. The binary `/bin/sleep` does not exist.
- When `kubelet` attempts to invoke `["/bin/sleep", "20"]`, the CRI `ExecSync` call fails immediately with `executable file not found in $PATH`.
- The `kubelet` emits a `FailedPreStopHook` event, aborts the hook, and sends `SIGTERM` to the Go process at $T = 0\text{s}$.
- Because ALB target deregistration requires 15–20s, the ALB continues sending new HTTP requests to the Pod for up to 20s after the Go server has already received `SIGTERM` and closed its listener.

#### 2. Grace Period Mathematical Inequality Violation
Even if `/bin/sleep` existed, setting `terminationGracePeriodSeconds: 20` with a 20s sleep creates a zero-headroom boundary condition:
$$T_{\text{grace}} (20\text{s}) = T_{\text{preStop}} (20\text{s}) + T_{\text{in-flight}} (0\text{s}) + T_{\text{buffer}} (0\text{s})$$
At $T = 20\text{s}$, the sleep completes and `kubelet` immediately sends `SIGKILL` (`kill -9`). The Go process gets 0ms to drain active requests or close TCP sockets cleanly.

#### 3. HTTP/2 Multiplexing & Connection Teardown
HTTP/2 multiplexes multiple logical streams over a single persistent TCP connection. 
- When the Go application process is abruptly killed, its kernel TCP socket closes, transmitting a TCP `RST` to the ALB. All active streams on that connection terminate with HTTP 502.
- **Architectural Requirement:** The Go HTTP/2 server must implement a graceful drain sequence upon receiving `SIGTERM`:
  1. Call `http.Server.Shutdown(ctx)`.
  2. The server sends an HTTP/2 `GOAWAY` frame with `ErrCodeNo` to instruct the ALB and clients not to allocate new streams on this connection while permitting active streams to complete.
  3. Wait for active streams to drain before exiting.

#### 4. Missing AWS ALB Target Health Readiness Gate
The deployment lacks `readinessGates` for `target-health.elbv2.k8s.aws`. During rolling updates, the Deployment controller considers new pods available as soon as the local pod readiness check passes. It immediately begins terminating old pods **before** the AWS ALB has registered and confirmed the health of the new pods in the Target Group, causing total available target capacity to drop below required levels.

---

### Corrected Production Architecture

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ledger-service
  namespace: finance
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 2
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: ledger-service
    spec:
      # Satisfies inequality: 60s > 20s (preStop) + 30s (HTTP/2 drain) + 10s (buffer)
      terminationGracePeriodSeconds: 60

      # Synchronize rollout progression with AWS ALB Target Group health
      readinessGates:
        - conditionType: "target-health.elbv2.k8s.aws"

      containers:
        - name: ledger
          image: gcr.io/distroless/static-debian12:nonroot
          lifecycle:
            preStop:
              # Native sleep eliminates shell dependency in distroless images (v1.30+ GA)
              sleep:
                seconds: 20
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            periodSeconds: 5
            failureThreshold: 2
```

Configure the ALB Target Group deregistration delay to complete before `preStop` finishes:
```yaml
# In TargetGroupBinding or AWS Ingress annotation:
deregistration_delay.timeout_seconds: "15"
```

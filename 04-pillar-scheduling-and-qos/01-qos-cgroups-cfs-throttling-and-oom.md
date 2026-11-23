# QoS Classes, cgroup v2, CFS Throttling & Linux OOM Architecture

## 1. Architectural Mental Model (The "Why" & "What")

In cloud-native infrastructure, resource management is not an abstract Kubernetes concept; it is **direct Linux kernel accounting**. When a developer declares `resources.requests` and `resources.limits` in a Pod manifest, Kubernetes does not monitor CPU cycles or terminate memory-hungry containers in real time. Instead, `systemd` and `kubelet` translate those declarative values into **Linux `cgroup v2` controllers** (`cpu`, `memory`, `io`, and `pids`) and configure the kernel's **Out-Of-Memory (`oom_score_adj`) priority table**.

Two fundamental rules govern this architecture:
1. **CPU is a Compressible Resource (Throttling):** If a container attempts to consume more CPU than allowed by its limit, the Completely Fair Scheduler (CFS) bandwidth controller deschedules its processes. The threads freeze in the runqueue; execution stalls and tail latency spikes, but the process remains alive.
2. **Memory is a Non-Compressible Resource (Termination):** If a container demands more memory than allowed by its limit and the kernel cannot reclaim sufficient memory from clean file page caches, the kernel cannot pause the process. It invokes the **Linux Out-Of-Memory (OOM) Killer** to immediately terminate the process with an uncatchable `SIGKILL` (exit code `137`).

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    cgroup v2 UNIFIED HIERARCHY TREE ON LINUX                                |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  /sys/fs/cgroup/
  ├── system.slice/                  <── Host OS systemd daemons (sshd, systemd-journald, chrony)
  │
  └── kubepods.slice/                <── Root slice for ALL Kubernetes containers
      │
      ├── kubepods-burstable.slice/  <── Burstable QoS tier (Has requests != limits)
      │   │
      │   └── kubepods-burstable-pod<uuid>.slice/
      │       ├── cgroup.procs       <── PIDs belonging to this pod
      │       ├── cpu.weight         <── CPU Shares (relative priority from requests: 1 to 10000)
      │       ├── cpu.max            <── Hard CFS Bandwidth Quota: $QUOTA $PERIOD (from limits)
      │       ├── memory.min         <── Hard memory protection (never reclaimed by kernel)
      │       ├── memory.high        <── Memory throttle threshold (triggers reclaim/throttling)
      │       └── memory.max         <── Hard OOM ceiling (triggers Ring 0 OOM Killer!)
      │
      ├── kubepods-besteffort.slice/ <── BestEffort QoS tier (No requests, no limits)
      │   └── cpu.weight = 1         <── Lowest possible scheduling priority (cpu.max = max)
      │
      └── (Root kubepods.slice)      <── Guaranteed QoS pods sit directly at the root!
```

---

## 2. Under the Hood: Mechanics & Internals

### A. The Three Kubernetes Quality of Service (QoS) Classes

Kubernetes dynamically assigns one of three QoS classes to every Pod based strictly on its compute resource declarations:

| QoS Tier | Declaration Criteria | cgroup Location | Default `oom_score_adj` | Scheduling & Eviction Priority |
| :--- | :--- | :--- | :--- | :--- |
| **`Guaranteed`** | `requests == limits` for **both** CPU & Memory across all containers. | Directly under `/sys/fs/cgroup/kubepods.slice/` | **`-997`** (Protected) | Top scheduling priority; lowest eviction risk; spared by OOM killer. |
| **`Burstable`** | `requests < limits`, or only requests/limits defined for one resource. | `/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/` | **`2` to `999`** (Dynamic) | Scheduled based on requests; throttled at limits; killed based on usage ratio. |
| **`BestEffort`** | **No** requests and **no** limits specified for any container. | `/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/` | **`1000`** (Highest target) | Lowest priority; first to be evicted by kubelet or killed by the kernel. |

---

### B. Translating YAML Declarations to `cgroup v2` Filesystem Values

When `kubelet` configures container isolation, it evaluates exact mathematical formulas to populate the `cgroup v2` controllers:

#### 1. CPU Requests $\rightarrow$ `cpu.weight` (Proportional Share)
`cpu.weight` dictates relative CPU share among competing runnable processes when all CPU cores on the node are 100% saturated.
* **Kernel Range:** `1` (lowest priority) to `10,000` (highest priority). Baseline default is `100`.
* **Formula:**
  $$\text{cpu.weight} = 1 + \left\lfloor \frac{(\text{cpu\_request\_in\_millicores} - 1) \times 9999}{9999} \right\rfloor$$
  * *1000m (1 vCPU):* $\longrightarrow \text{cpu.weight} = 100$
  * *4000m (4 vCPUs):* $\longrightarrow \text{cpu.weight} = 400$
  * *BestEffort Pod:* $\longrightarrow \text{cpu.weight} = 1$

#### 2. CPU Limits $\rightarrow$ `cpu.max` (CFS Bandwidth Hard Cap)
`cpu.max` enforces an unyielding ceiling on total CPU runtime within a recurring enforcement window.
* **Format:** `$QUOTA_US $PERIOD_US`
* **Default Period:** **100,000 microseconds (100ms)**.
* **Formula:**
  $$\text{quota\_us} = \text{cpu\_limit} \times 100,000\,\mu\text{s}$$
  * *`cpu: 500m` (0.5 vCPU):* $\longrightarrow$ `50000 100000` (50ms of CPU time allowed per 100ms period)
  * *`cpu: 2` (2 full vCPUs):* $\longrightarrow$ `200000 100000` (200ms of CPU time allowed per 100ms period)
  * *No Limit Set:* $\longrightarrow$ `max 100000` (Uncapped)

#### 3. Memory Limits $\rightarrow$ `memory.max` (Hard OOM Boundary)
* Directly converts bytes into an integer value.
* *Example:* `memory: 4Gi` $\longrightarrow$ `4294967296` written directly to `/sys/fs/cgroup/.../memory.max`.

---

### C. The Multi-Threaded CFS Quota Trap (Tail-Latency Disasters)

A prevalent production bottleneck in multi-core cloud environments (e.g., 32-core or 64-core AWS EC2 instances) is **synthetic CFS throttling**:

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    THE MULTI-THREADED CFS QUOTA COLLAPSE                                    |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  INFRASTRUCTURE CONTEXT:
  - Worker Node: AWS c5.16xlarge (64 vCPUs)
  - Pod Spec: cpu: "2" (Quota = 200,000µs runtime per 100,000µs period)
  - Workload: Go / Java / Node.js with 32 active concurrent worker threads

  TIMELINE OF A SINGLE 100ms CFS PERIOD:
  ├──────────────────────────────────────────────────────────────────────────────────────────┤
  0ms                                      6.25ms                                           100ms
  
  [ 32 Threads execute concurrently across 32 physical cores ]
  │
  ├── 32 threads × 6.25ms of wall-clock time = 200ms TOTAL CPU QUOTA CONSUMED!
  │
  ▼
  AT 6.25ms: QUOTA BURST EXHAUSTED!
  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │ THE LINUX KERNEL FREEZES ALL 32 THREADS FOR THE REMAINING 93.75ms!                       │
  │ - All processes in the cgroup are descheduled.                                           │
  │ - Inbound TCP SYN & HTTP requests queue in kernel socket buffers.                        │
  │ - Prometheus shows low average CPU usage (e.g., 1.2 vCPUs), but P99 latency spikes by    │
  │   93.75ms on every single period boundary!                                               │
  └──────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                             │
  AT 100ms: PERIOD RESETS! ──────────────────────────────────────────────────────────────────┘
  - The kernel unfreezes threads... until the quota is exhausted again in the next 6ms!
```

#### Diagnostic Metric:
Check `/sys/fs/cgroup/.../cpu.stat`. If $\frac{\text{nr\_throttled}}{\text{nr\_periods}} > 0.05$ (5%), the workload is suffering from active CFS starvation.

---

### D. The Linux Out-Of-Memory (OOM) Execution Hierarchy

When physical RAM on a worker node is completely exhausted, the kernel invokes the OOM Killer. It calculates an OOM score between `0` and `1000` for every process:

$$\text{OOM Score} = \text{Normalized Process Memory Usage} + \text{oom\_score\_adj}$$

The kernel terminates the process with the highest score first.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                      THE KERNEL OOM SACRIFICE HIERARCHY                                     |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  TIER 1: SYSTEM CRITICAL DAEMONS
  systemd, kubelet, containerd, sshd
  oom_score_adj: -998 to -1000  ──► IMMUNE (Never killed)

  TIER 2: GUARANTEED QoS PODS (Requests == Limits)
  oom_score_adj: -997           ──► PROTECTED (Last pods to be evaluated)

  TIER 3: BURSTABLE QoS PODS (Requests != Limits)
  Formula: oom_score_adj = 1000 - max( floor( 1000 × memory_request / node_capacity ), 2 )
  Range: 2 to 999               ──► PROPORTIONAL (Larger memory requests are spared longer)

  TIER 4: BESTEFFORT QoS PODS (No requests, no limits)
  oom_score_adj: 1000           ──► FIRST TO BE SLAUGHTERED BY THE KERNEL!
```

---

### E. Kernel OOM Killer vs. Kubelet Eviction

Engineers must cleanly separate **Kernel Space (Ring 0)** termination from **User Space (Ring 3)** eviction:

| Attribute | Kernel OOM Killer (`exit code 137`) | Kubelet Node Eviction |
| :--- | :--- | :--- |
| **Execution Layer** | **Kernel Space (CPU Ring 0)** | **User Space (CPU Ring 3 - Kubelet daemon)** |
| **Trigger Point** | Container hits `cgroup` `memory.max`, OR host RAM hits absolute zero. | Node available memory drops below threshold (e.g., `memory.available < 100Mi`). |
| **Evaluation Speed** | **Microseconds (Synchronous, blocking allocation)** | **Seconds (Asynchronous 10s syncLoop polling)** |
| **Termination Signal** | Instantaneous uncatchable **`SIGKILL`** | Graceful **`SIGTERM`** $\rightarrow$ `gracePeriod` $\rightarrow$ `SIGKILL`. |
| **Pod Final State** | Pod stays on node; container restarts or enters `CrashLoopBackOff`. | Pod transitions to `Failed: Evicted` and is rescheduled elsewhere. |

---

### F. Pressure Stall Information (PSI)

Modern Linux distributions expose real-time resource starvation metrics under `/proc/pressure/`:
* `/proc/pressure/cpu`
* `/proc/pressure/memory`
* `/proc/pressure/io`

PSI tracks two critical dimensions:
1. **`some`:** The percentage of time in which *at least one process* was stalled waiting for the resource.
2. **`full`:** The percentage of time in which *ALL processes* in the cgroup were completely stalled (total productive standstill).

If `full` memory pressure climbs above `0.00%`, the system is actively thrashing swap or frozen in direct kernel page cache reclaim loops.

---

## 3. Production Implementation: Hardened Resource Profiles

### A. Latency-Sensitive Web Microservice (The "No CPU Limits" Pattern)
For HTTP APIs and gRPC services where P99 tail latency is critical:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
spec:
  replicas: 4
  template:
    spec:
      containers:
      - name: api
        image: payment-api:v2.1
        resources:
          requests:
            cpu: "2"                  # ──► Guarantees 200 cpu.weight shares in runqueue
            memory: "4Gi"             # ──► Bases scheduling on 4Gi capacity
          limits:
            memory: "4Gi"             # ──► Strict OOM ceiling to prevent memory leaks
            # CPU Limit OMITTED!      # ──► Eliminates CFS bandwidth throttling completely!
```

### B. Managed Runtime Memory Sizing Formula (Java / JVM)
Never set JVM `-Xmx` equal to the container memory limit. Off-heap allocations, Metaspace, thread stacks, and JNI buffers will trigger kernel OOM kills.

$$\text{Container Memory Limit} \ge \frac{\text{Heap Maximum (-Xmx)}}{0.70}$$

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-java-service
spec:
  template:
    spec:
      containers:
      - name: jvm-app
        image: openjdk:21-slim
        resources:
          requests:
            cpu: "4"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "8Gi"             # ──► Guaranteed QoS (oom_score_adj: -997)
        env:
        - name: JAVA_TOOL_OPTIONS
          # 70% of 8Gi = 5.6Gi allocated to Heap; 2.4Gi reserved for off-heap buffers:
          value: "-XX:MaxRAMPercentage=70.0 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: The Multi-Threaded Latency Collapse
* **Symptom:** Microservice P99 latency spikes by 100ms+ during peak traffic. Node CPU utilization is under 30%.
* **Root Cause:** Multi-threaded runtime (Go/Java/Node) exhausts CFS bandwidth quota in the first 10ms of each 100ms period. The kernel freezes all threads for the remaining 90ms.
* **Diagnostics:**
  ```bash
  # Check throttling statistics on host cgroup
  cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/.../cpu.stat | grep nr_throttled
  ```
* **Remedy:** Remove `resources.limits.cpu`, or increase the limit to accommodate concurrency.

### Failure Mode 2: Silent Exit Code `137` (OOMKilled)
* **Symptom:** Pod restarts intermittently. `kubectl describe pod` shows:
  ```text
  State:          Terminated
    Reason:       OOMKilled
    Exit Code:    137
  ```
* **Root Cause:** Application anonymous memory (RSS) exceeded `cgroup` `memory.max`. The kernel issued an instantaneous `SIGKILL`.
* **Diagnostics:**
  ```bash
  # 1. Check kernel dmesg for OOM killer invocations
  dmesg -T | grep -E -i "oom[-_]killer|killed process"

  # 2. Inspect memory usage vs max limit in cgroup v2
  cat /sys/fs/cgroup/.../memory.current
  cat /sys/fs/cgroup/.../memory.max

  # 3. Inspect real-time memory pressure
  cat /proc/pressure/memory
  ```

---

## 5. Socratic Architectural Challenge

### Scenario:
You are the Lead SRE supporting an automated high-frequency trading platform on AWS EKS.

A Go-based trade execution engine runs as a `Burstable` pod on an AWS `c5.18xlarge` instance (72 vCPUs, 144 GiB RAM):
```yaml
resources:
  requests:
    cpu: "4"
    memory: "16Gi"
  limits:
    cpu: "4"
    memory: "16Gi"
```
*(Notice: CPU request equals limit, and Memory request equals limit, classifying this as a `Guaranteed` QoS pod with `oom_score_adj: -997`).*

During a market volatility event:
1. The Go application uses a pool of 64 concurrent goroutines to execute order books.
2. Order execution latency spikes from **2ms to 98ms**.
3. Prometheus reports pod CPU usage at **3.1 vCPUs** (under the 4-core limit).
4. Simultaneously, a rogue `BestEffort` logging agent on the same node enters an infinite loop, driving total node CPU usage to 100%.

### Architectural Questions:
1. Why did the trading engine experience 98ms latency spikes when its CPU utilization was only 3.1 vCPUs?
2. Did the rogue logging agent steal CPU cycles from the trading engine? Why or why not?
3. How do you re-architect the trading engine's compute profile to guarantee sub-millisecond execution even during market volatility?

### Architectural Answers:
1. **Multi-Threaded CFS Quota Exhaustion:** The Go application ran 64 goroutines concurrently across 64 physical cores on the `c5.18xlarge` instance. With a limit of `4` vCPUs, its CFS bandwidth quota was $4 \times 100\text{ms} = 400\text{ms}$ of total runtime per 100ms window. 64 threads consumed 400ms of CPU time in just $\frac{400\text{ms}}{64} = \mathbf{6.25\text{ms}}$ of wall-clock time! The kernel's CFS scheduler froze all 64 threads for the remaining **93.75ms** of the period.
2. **No, the logging agent did not steal cycles:** Because the trading engine is a `Guaranteed` pod with `cpu: "4"`, its `cpu.weight` was set to `400`, whereas the `BestEffort` logging agent had a `cpu.weight` of `1`. The trading engine had overwhelming scheduling priority; its latency was self-inflicted by CFS bandwidth quota capping, not node-level CPU starvation.
3. **The Production Fix:**
   - **Omit `resources.limits.cpu` completely** to eliminate CFS bandwidth quota checks (`cpu.max = max`).
   - Increase `resources.requests.cpu` to `16` or `32` to guarantee dedicated proportional shares (`cpu.weight: 1600` or `3200`).
   - Enable the **Kubernetes CPU Manager with `static` policy** (`cpumanager: static`), pinning the trading engine's threads to dedicated, isolated physical CPU cores via `cpuset.cpus`, completely bypassing the CFS scheduler runqueue!

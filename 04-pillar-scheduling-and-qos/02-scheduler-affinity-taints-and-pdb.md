# Kubernetes Scheduler Framework, Topology Spread, Taints & PDB Architecture

## 1. Architectural Mental Model (The "Why" & "What")

In cloud-native systems, scheduling is the engine of **high availability, fault tolerance, and multi-AZ resilience**. Deploying workloads blindly allows `kube-scheduler` to pack all replicas of a critical database or API onto a single worker node or an isolated AWS Availability Zone (AZ). If that AZ suffers a power cut or the underlying EC2 instance undergoes hypervisor maintenance, the entire service experiences an outage.

To enforce high availability mathematically, Kubernetes provides four interrelated architectural primitives:
1. **The Scheduling Framework:** A plugin-based, 11-stage extension pipeline dividing work into a serialized **Scheduling Cycle** (evaluating nodes in memory) and a concurrent **Binding Cycle** (writing the assignment to `etcd`).
2. **Topology Spread Constraints:** Distributes replicas evenly across topological fault domains (e.g., AWS AZs, physical server racks) using strict variance math (`maxSkew`), avoiding the exponential CPU scaling penalties of legacy `podAntiAffinity`.
3. **Taints & Tolerations:** Node-level access boundaries used to repel unauthorized pods, dedicating specialized hardware (GPUs, high-memory database nodes, AWS Spot instances) to specific workloads.
4. **PodDisruptionBudgets (PDB):** An operational SLA contract enforced by the Kubernetes **Eviction API** that limits voluntary disruptions during `kubectl drain`, Karpenter autoscaling, and automated EKS AMI rollouts.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    KUBERNETES SCHEDULING FRAMEWORK & TOPOLOGY PIPELINE                      |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  UNSCHEDULED POD (Pending in activeQ)
        │
  =============================================================================================================
  1. THE SCHEDULING CYCLE (Single-Threaded, Serialized In-Memory Evaluation)
  =============================================================================================================
        │
        ├──► [ PreFilter ]  ──► Validates prerequisites (e.g., Pod matches basic requirements)
        │
        ├──► [ Filter ]     ──► ELIMINATES INELIGIBLE NODES:
        │                       ├── NodeResourcesFit: CPU & Memory allocatable vs requests
        │                       ├── NodeAffinity: Checks requiredDuringSchedulingIgnoredDuringExecution
        │                       ├── NodeTaints: Discards nodes with untolerated NoSchedule taints
        │                       ├── VolumeRestrictions: Checks max EBS attachments per instance type
        │                       └── TopologySpread: Filters domains exceeding maxSkew (DoNotSchedule)
        │                       * IF NO NODES PASS: Calls PostFilter (Preemption)!
        │
        ├──► [ PreScore ]   ──► Prepares state for scoring plugins
        │
        ├──► [ Score ]      ──► RANKS SURVIVING NODES (0 to 100 Points):
        │                       ├── NodeResourcesBalancedAllocation: Balances CPU-to-memory ratio
        │                       ├── ImageLocality: Favors nodes with container image cached in containerd
        │                       ├── NodeAffinity: Evaluates preferredDuringScheduling weights
        │                       └── TopologySpread: Prioritizes domains with fewest existing replicas
        │
        ├──► [ Reserve ]    ──► ATOMIC IN-MEMORY RESERVATION:
        │                       Temporarily locks node resources in scheduler cache before etcd commit,
        │                       preventing concurrent scheduling race conditions!
        ▼
  =============================================================================================================
  2. THE BINDING CYCLE (Asynchronous, Multi-Threaded Execution)
  =============================================================================================================
        │
        ├──► [ Permit ]     ──► Approval gate (Holds pods up to N seconds for gang/batch scheduling)
        │
        ├──► [ PreBind ]    ──► Network/storage attachments (e.g., dynamic CSI volume operations)
        │
        ├──► [ Bind ]       ──► WRITES TO ETCD:
        │                       Issues HTTP POST /api/v1/namespaces/{ns}/pods/{name}/binding
        │                       Commits: spec.nodeName = "ip-10-0-1-50.ec2.internal"
        │
        └──► [ PostBind ]   ──► Emits scheduler metrics and clears internal reservation locks
```

---

## 2. Under the Hood: Mechanics & Internals

### A. The Three Scheduling Queues
Inside `kube-scheduler`, pending pods reside in one of three internal data structures:
1. **`activeQ` (Active Priority Queue):** Holds ready-to-schedule pods sorted by `PriorityClass` (`value` integer from $-2{,}147{,}483{,}648$ to $1{,}000{,}000{,}000$).
2. **`backoffQ` (Exponential Backoff Queue):** Pods that fail scheduling move here temporarily (delaying 1s, 2s, 4s, up to 10s) to avoid pegging control plane CPU in tight retry loops.
3. **`unschedulableQ` (Unschedulable Heap):** If a pod cannot be placed (e.g., `0/20 nodes available: Insufficient memory`), it is parked here until a cluster event (node addition, pod termination, PVC bound) moves it back to `activeQ`.

---

### B. High Availability: `podAntiAffinity` vs. `topologySpreadConstraints`

#### The Legacy Problem: `podAntiAffinity` ($O(N \times M)$ Scaling Collapse)
* **Algorithmic Overhead:** To evaluate `podAntiAffinity`, the scheduler compares every incoming pod against every existing pod on every node. In large clusters (500+ nodes, 10,000+ pods), this causes extreme scheduler latency and high control plane CPU usage.
* **The Binary Failure Mode:** `podAntiAffinity` is binary: a node/AZ either has 0 matching pods or 1. If you run 6 replicas across 3 Availability Zones, `podAntiAffinity` with `requiredDuringScheduling` marks replicas 4, 5, and 6 **permanently unschedulable**.

#### The Modern Solution: `topologySpreadConstraints` (Domain Variance Math)
`topologySpreadConstraints` (GA in Kubernetes 1.19+) evaluates domain balance using the **Skew Formula**:

$$\text{Skew} = (\text{Matching Pods in Candidate Domain}) - (\text{Global Minimum Pods across Any Domain}) \le \text{maxSkew}$$

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    TOPOLOGY SPREAD BALANCING (maxSkew = 1)                                  |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  AVAILABLE TOPOLOGY DOMAINS (AWS Availability Zones)
  ┌─────────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────────┐
  │ ZONE: us-east-1a        │    │ ZONE: us-east-1b        │    │ ZONE: us-east-1c        │
  │                         │    │                         │    │                         │
  │   [ Pod 1 ]  [ Pod 2 ]  │    │   [ Pod 3 ]             │    │   [ Pod 4 ]             │
  │                         │    │                         │    │                         │
  │ Current Count: 2        │    │ Current Count: 1        │    │ Current Count: 1        │
  └─────────────────────────┘    └─────────────────────────┘    └─────────────────────────┘
  
  NOW SCHEDULING: [ Pod 5 ]
  
  Evaluating Zone us-east-1a:
  - If Pod 5 is placed in us-east-1a -> Count becomes 3.
  - Global Minimum Count across all zones = 1 (in us-east-1b and us-east-1c).
  - Skew = 3 - 1 = 2.
  - SKEW EXCEEDS maxSkew (1)! ──► Filter plugin REJECTS us-east-1a!
  
  Evaluating Zone us-east-1b or us-east-1c:
  - If Pod 5 is placed in us-east-1b -> Count becomes 2.
  - Global Minimum Count across all zones = 1.
  - Skew = 2 - 1 = 1 (Within maxSkew <= 1).
  - RESULT: Scheduler assigns Pod 5 to us-east-1b or us-east-1c!
```

---

### C. Node Partitioning: Taints vs. Tolerations

Taints are key-value pairs applied to **Nodes** that repel pods. Tolerations are applied to **Pods** allowing them to schedule on tainted nodes:

| Taint Effect | Evaluation Phase | System Behavior |
| :--- | :--- | :--- |
| **`NoSchedule`** | Scheduling (Filter Plugin) | Blocks pods lacking a matching toleration from scheduling. Running pods are unaffected. |
| **`PreferNoSchedule`** | Scheduling (Score Plugin) | Soft repulsion. The scheduler avoids the node, but places the pod there if no other nodes have capacity. |
| **`NoExecute`** | **Continuous Runtime & Eviction** | Hard eviction. If applied to a node, any running pod lacking a matching toleration is **immediately evicted and terminated**. |

#### Spot Interruption & Network Partition Resilience:
When an AWS Spot instance receives a 2-minute termination notice, or a node becomes unreachable, the node controller taints it with `node.kubernetes.io/unreachable:NoExecute`. Pods can define `tolerationSeconds` to avoid instant eviction during transient network flaps:
```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300    # ──► Holds pod alive for 5 minutes before evicting
```

---

### D. Production Protection: PodDisruptionBudgets (PDB)

A **`PodDisruptionBudget`** protects workloads from **voluntary disruptions** (e.g., `kubectl drain`, Karpenter node consolidation, AWS EKS automated AMI rollouts):

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+
|                                    THE EVICTION API & PDB ENFORCEMENT LOOP                                  |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────────+

  Karpenter / SRE executes: kubectl drain ip-10-0-1-50.ec2.internal
        │
        ▼
  Client invokes Kubernetes EVICTION API:
  POST /api/v1/namespaces/{ns}/pods/{name}/eviction
        │
        ▼
  kube-apiserver checks active PodDisruptionBudgets:
        │
        ├── Current Healthy Pods: 3
        ├── PDB minAvailable: 2
        ├── Allowed Disruptions: 3 - 2 = 1
        │
        ├──► ALLOWED DISRUPTIONS > 0:
        │    - Eviction approved! Pod receives SIGTERM.
        │    - Allowed Disruptions decremented to 0.
        │
        └──► IF SECOND POD ON NODE IS CALLED FOR EVICTION:
             - Allowed Disruptions == 0!
             - API REJECTS request: HTTP 429 Too Many Requests (Cannot evict pod: PDB violated!)
             - kubectl drain blocks and waits until replacement pod becomes Ready on another node!
```

---

## 3. Production Implementation: Hardened Multi-AZ & PDB Blueprint

### A. Multi-Tier Topology Spread Constraint
This production pattern guarantees:
1. Strict equal distribution across **AWS Availability Zones** (`DoNotSchedule`).
2. Best-effort distribution across **individual EC2 instances** within each zone (`ScheduleAnyway`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processing-api
  namespace: core-banking
spec:
  replicas: 6
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
    spec:
      topologySpreadConstraints:
      # Tier 1: Strict AZ distribution across AWS us-east-1a, 1b, 1c
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: order-api
      # Tier 2: Soft host-level distribution to avoid single EC2 hardware failure
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: order-api
      containers:
      - name: api
        image: order-api:v3.4
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            memory: "4Gi"
```

### B. Production `PodDisruptionBudget`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-api-pdb
  namespace: core-banking
spec:
  # Guarantees that at most 1 replica can be evicted simultaneously during node maintenance
  maxUnavailable: 1
  selector:
    matchLabels:
      app: order-api
```

### C. Dedicated GPU Node Pool (Taints & Tolerations)

#### 1. Worker Node Taint (Applied via Terraform / AWS EKS Managed Node Group):
```bash
kubectl taint nodes ip-10-0-5-20.ec2.internal accelerator=nvidia-gpu:NoSchedule
```

#### 2. Workload Manifest (Dedicated ML Model):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fraud-detection-ml
spec:
  replicas: 2
  template:
    spec:
      tolerations:
      - key: "accelerator"
        operator: "Equal"
        value: "nvidia-gpu"
        effect: "NoSchedule"
      nodeSelector:
        node.kubernetes.io/instance-type: "g5.4xlarge"
      containers:
      - name: inference
        image: fraud-ml:v1.0
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### Failure Mode 1: The "PDB Deadlock" Freezing EKS Cluster Upgrades
* **Symptom:** Automated EKS managed node group upgrades or Karpenter node consolidation hang indefinitely. The upgrade pipeline times out after 60 minutes.
* **Root Cause:** A single-replica service (`replicas: 1`) configured a PDB with `minAvailable: 1` or `maxUnavailable: 0%`. The Eviction API calculates $\text{Allowed Disruptions} = 1 - 1 = \mathbf{0}$. The Eviction API permanently returns `429 Too Many Requests`, preventing the node from ever draining.
* **Diagnostics:**
  ```bash
  # Check which PDB is blocking disruptions:
  kubectl get pdb -A
  # Look for PDBs where ALLOWED DISRUPTIONS is 0 and CURRENT < MIN
  ```
* **Remedy:** Never configure `minAvailable: 1` or `maxUnavailable: 0` on single-replica workloads. Use `maxUnavailable: 1` or delete the PDB during maintenance.

### Failure Mode 2: Topology Spread "Unschedulable" Trap on Node Failure
* **Symptom:** After an AWS AZ failure, replacement pods remain stuck in `Pending` with:
  ```text
  Warning FailedScheduling 0/15 nodes available: 5 node(s) didn't match pod topology spread constraints
  ```
* **Root Cause:** The Deployment configured `whenUnsatisfiable: DoNotSchedule` with `maxSkew: 1`. When AZ `us-east-1a` went offline, its pod count became `0`. Because `maxSkew: 1` required that no zone exceed $\text{Global Minimum} + 1 = 0 + 1 = 1$, pods in `us-east-1b` (already at 2) could not accept any new replicas!
* **Remedy:** Set `whenUnsatisfiable: ScheduleAnyway` for auto-recovery, or ensure `nodeAffinity` restricts evaluation to active healthy zones.

---

## 5. Socratic Architectural Challenge

### Scenario:
You are the Lead Cloud Architect designing a mission-critical Payment Processing Service for a banking platform deployed on an AWS EKS cluster running across 3 Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`).

The operational requirements are:
1. The service runs **6 replicas**.
2. If any single AWS Availability Zone suffers a catastrophic blackout, the application must maintain at least **4 active, healthy replicas** without dropping transactions.
3. During scheduled EKS node upgrades or AMI patching, automated node drains must **never take down more than 1 pod at a time**.
4. The pods must be strictly balanced across the 3 zones (maximum difference of 1 pod between any two zones).

### Architectural Solution & Verification:
1. **Topology Spread:** Configure `topologySpreadConstraints` with:
   - `maxSkew: 1`
   - `topologyKey: topology.kubernetes.io/zone`
   - `whenUnsatisfiable: DoNotSchedule`
   - *Resulting Allocation:* Exactly **2 pods per AZ** ($2 + 2 + 2 = 6$).
2. **AZ Failure Math:** If `us-east-1a` suffers a blackout, its 2 pods die. The remaining 4 pods continue operating in `us-east-1b` (2 pods) and `us-east-1c` (2 pods), satisfying Requirement #2 without dropping traffic!
3. **Disruption Budget:** Apply a `PodDisruptionBudget` with **`maxUnavailable: 1`** (or `minAvailable: 5`). During an automated rolling node drain, the Eviction API permits exactly 1 pod to be terminated at a time, requiring the replacement pod to pass its readiness probe before the next node can be drained.
4. **Why `podAntiAffinity` Fails Here:** Setting `podAntiAffinity` on `topology.kubernetes.io/zone` allows only 1 pod per zone, making pods 4, 5, and 6 permanently unschedulable. Modern `topologySpreadConstraints` is the only mechanism that satisfies all four criteria.

# Workload Controllers Architecture: State Reconciliation & Lifecycle Engines

## 1. Architectural Mental Model (The "Why" & "What")

In Kubernetes, state reconciliation is decoupled from execution. Controllers in `kube-controller-manager` execute continuous control loops that observe cluster state via the API server, compare actual state in `etcd` against desired state, and execute mutations to converge them. 

Rather than polling `kube-apiserver` via expensive HTTP GETs, controllers implement the **client-go Informer** architecture. A persistent HTTP/2 `ListWatch` stream maintains a local, thread-safe memory cache (`Indexer`), isolating `etcd` from high-frequency queries.

```
+-------------------------------------------------------------------------------------------------------+
|                                  KUBE-CONTROLLER-MANAGER CONTROL LOOP                                 |
+-------------------------------------------------------------------------------------------------------+

   +----------------+
   | kube-apiserver | <==== etcd (Raft MVCC Store)
   +-------+--------+
           | 1. ListWatch (HTTP/2 Chunked Stream with ResourceVersion)
           v
   +-------+--------+
   |   Reflector    | (cache/reflector.go)
   +-------+--------+
           | 2. Push Delta (Added / Updated / Deleted / Sync)
           v
   +-------+--------+
   |   DeltaFIFO    | (Bounded delta queue)
   +-------+--------+
           | Pop()
           +-----------------------------+
           | 3. OnUpdate/OnAdd           | 4. Update In-Memory Cache
           v                             v
   +-------+--------+            +-------+--------+
   | Resource Event |            |    Indexer     | <==== Thread-Safe Local RAM Store
   |    Handler     |            |  (Local Cache) |       (cache.ThreadSafeStore)
   +-------+--------+            +-------+--------+
           | 5. Add key: "ns/name"       |
           v                             | 7. Read object
   +-------+--------+                    |
   |   WorkQueue    |                    v
   | (Rate Limiting)|             +------+---------+
   +-------+--------+             |  syncHandler   | (Reconciliation Logic)
           | 6. Get()             |  Worker Pool   |
           +--------------------->+------+---------+
                                         | 8. Mutate (POST/PUT/PATCH via CAS)
                                         v
                                  +------+---------+
                                  | kube-apiserver | ====> etcd Commit
                                  +----------------+
```

### Workload Controller Comparison Matrix

| Controller | Primary Invariant | Identity Model | Storage Semantics | Partition Failover | Target Workload |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Deployment** | $N$ identical, fungible replicas | Ephemeral: `pod-template-hash` + random string | Ephemeral (`emptyDir`) or `ReadWriteMany` (NFS/EFS) | Immediate recreation on healthy nodes | Stateless HTTP APIs, web frontends |
| **StatefulSet** | Stable identity, strict ordered rollout | Deterministic: `$(name)-$(ordinal)` (e.g. `db-0`) | Dedicated `volumeClaimTemplates` (`ReadWriteOnce`) | Blocked failover during partition to prevent split-brain | Clustered DBs (ZooKeeper, Kafka, etcd, Redis) |
| **DaemonSet** | Exactly one pod per eligible node | Node-bound: `$(name)-$(hash)` | Host mounts (`hostPath`), host net/PID namespaces | Auto-terminates on node drain, runs on new nodes | Log forwarders, CNI plugins, monitoring daemons |
| **Job** | Run to completion ($K$ successful runs) | Non-indexed random or Indexed batch ordinal | Scratch storage or shared network volumes | Retries failed pods with exponential backoff | DB migrations, batch ETL pipelines |
| **CronJob** | Time-series schedule creating Jobs | Delegated to downstream Job definitions | Delegated to downstream Job definitions | Policy-dependent (`Allow`, `Forbid`, `Replace`) | Scheduled backups, nightly report rollups |

---

## 2. Under the Hood (Mechanics & Internals)

### A. Level-Triggered vs. Edge-Triggered Reconciliation
Edge-triggered architectures emit signals on state changes ($A \to B$). If a message is lost, state desynchronizes permanently. 

Kubernetes controllers are **level-triggered**:
- The `WorkQueue` stores only the **key** (`<namespace>/<name>`), not the diff payload.
- Rapid successive events collapse into a single key (event coalescing).
- The `syncHandler` queries the local `Indexer` for the complete current state and reconciles against the actual world.
- A periodic **resync** (default 10–12 hours) forces all objects back through `syncHandler`, correcting silent drift or missed watch events.

### B. Optimistic Concurrency Control (OCC) & 409 Conflict Mechanics
Kubernetes rejects row-level locking. It uses OCC governed by `metadata.resourceVersion` (etcd 64-bit revision):
```
Worker 1: GET Pod (rv: 1042) -------------------> [etcd: rv 1042]
Worker 2: GET Pod (rv: 1042) -------------------> [etcd: rv 1042]
Worker 1: PUT Pod (rv: 1042) -> CAS Matches! ---> [etcd: rv 1043] -> 200 OK
Worker 2: PUT Pod (rv: 1042) -> CAS Fails! (rv 1042 != 1043) ---> 409 Conflict
```
On receiving `HTTP 409 Conflict` (`apierrors.IsConflict()`), the controller drops the mutation, re-reads the fresh object from `Indexer`, re-applies its business logic, and retries the commit.

### C. Garbage Collection & `ownerReferences`
Child resources point to parents via `metadata.ownerReferences` containing `uid`, `controller: true`, and `blockOwnerDeletion: true`.
- **Foreground Cascading**: Parent gets `deletionTimestamp` and `foregroundDeletion` finalizer. Parent remains visible until GC deletes all child objects with `blockOwnerDeletion: true`.
- **Background Cascading**: API server deletes parent immediately; GC sweeps orphaned children in the background.
- **Orphan Deletion**: Parent is deleted; children survive with `ownerReferences` stripped.

### D. Deployment to ReplicaSet Rollout Mechanics

#### 1. FNV-32a Hashing and `pod-template-hash`
The Deployment controller hashes `spec.template` using FNV-32a:
$$\text{pod-template-hash} = \text{FNV32a}(\text{CanonicalJSON}(\text{PodTemplateSpec}))$$
The hash string (e.g. `5968b6b8b5`) is stamped on ReplicaSets and Pod labels. Any change to `spec.template` (image, env, resources) alters the hash, creating a new ReplicaSet.

#### 2. Rollout Sizing Math (`maxSurge` & `maxUnavailable`)
During a `RollingUpdate`, boundaries are calculated from desired replicas $R$:
$$\text{Max Allowed Pods} = R + \lceil \text{maxSurge} \rceil$$
$$\text{Min Available Pods} = R - \lfloor \text{maxUnavailable} \rfloor$$
*Example*: $R = 8$, $\text{maxSurge} = 25\%$ (2), $\text{maxUnavailable} = 25\%$ (2).
- Max pods = $8 + 2 = 10$. Min available = $8 - 2 = 6$.
- Controller creates 2 new ReplicaSet pods ($RS_{new} \to 2$). Once healthy, it scales old ReplicaSet ($RS_{old} \to 6$), stepping until $RS_{new} = 8$ and $RS_{old} = 0$.

### E. StatefulSet Mechanics: Ordinal Identity & Partitioned Canaries
- **Identity & DNS**: Pods follow `$(name)-$(ordinal)`. Paired with a Headless Service (`clusterIP: None`), CoreDNS creates direct A/SRV records (`pod-0.svc.ns.svc.cluster.local`). Cross-reference [`./10-service-routing-and-internal-dns.md`](./10-service-routing-and-internal-dns.md).
- **`volumeClaimTemplates`**: Provisions unique PVCs: `$(claimName)-$(name)-$(ordinal)`. Scaling down **never** deletes PVCs, preventing catastrophic data loss.
- **Canary via `partition`**: Setting `updateStrategy.rollingUpdate.partition = K` restricts rollout:
  - Pods with ordinal $i \ge K$ are updated to the new template.
  - Pods with ordinal $i < K$ remain on the old template.

### F. DaemonSet Mechanics: Node-Affinity & Priority
- **Node-Affinity Injection**: Modern `daemonset-controller` injects `nodeAffinity` targeting `metadata.name` of candidate nodes, allowing `kube-scheduler` to evaluate standard taints/tolerations.
- **`system-node-critical`**: DaemonSets running critical node services specify `priorityClassName: system-node-critical` (priority `2000001000`). Under resource exhaustion, the scheduler preempts (evicts) lower-priority pods to seat the DaemonSet.

### G. Job & CronJob Mechanics
- **Completions, Parallelism & Backoff**: Job succeeds when `completions` pods exit `0`. Up to `parallelism` run concurrently. Pod failures trigger exponential backoff:
  $$T_{\text{delay}} = 10\text{s} \times 2^{R-1} \quad (\text{capped at 6 minutes})$$
- **Concurrency Policies**: `Allow` (parallel runs), `Forbid` (skips if prior run active), `Replace` (aborts current run with `SIGKILL` and starts new).
- **100-Missed-Schedule Rule**: If a CronJob misses $>100$ executions without `startingDeadlineSeconds`, it stalls permanently. Setting `startingDeadlineSeconds: 300` limits historical lookback to 300s, preventing lockup.

### H. ConfigMap Atomic Symlinks and Rollout Triggers
Mounted ConfigMaps update in-place via atomic `rename()` symlink swapping in `tmpfs`. Cross-reference [`./03-configmaps-and-atomic-symlinks.md`](./03-configmaps-and-atomic-symlinks.md).
**Crucial Architectural Invariant**: Mutating a ConfigMap does **not** trigger a Deployment rollout because `spec.template` does not change. To force automated rollouts on config mutation, inject a SHA256 digest into annotations:
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
```

---

## 3. Production Implementation (Code, CLI & Manifests)

### A. Zero-Memorization CLI Workflow
```bash
# 1. Generate syntactically valid skeletons via dry-run
kubectl create deployment api-server --image=app:v1.2 --replicas=4 --dry-run=client -o yaml > dep.yaml
kubectl create job migrator --image=flyway:10.8 --dry-run=client -o yaml -- migrate > job.yaml
kubectl create cronjob backup --image=pg:16 --schedule="0 1 * * *" --dry-run=client -o yaml > cron.yaml

# 2. Inspect nested API schemas recursively
kubectl explain deployment.spec.strategy.rollingUpdate
kubectl explain statefulset.spec.updateStrategy.rollingUpdate
kubectl explain cronjob.spec.jobTemplate.spec.template.spec
```

---

### B. Production Manifests

#### 1. Production Deployment (Hardened Stateless Workload)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: finance
spec:
  replicas: 4
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%       # 4 * 0.25 = 1 extra pod
      maxUnavailable: 0   # Never drop below 4 ready pods during rollout
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
      annotations:
        checksum/config: "b8a5f33a82e9114b7e8d641148e64c23f2ec49e49666c8b1bb7123456789abcd"
    spec:
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels: { app: payment-api }
              topologyKey: "topology.kubernetes.io/zone"
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: api
        image: internal-registry.enterprise.io/finance/api:v2.1.0
        ports:
        - name: http
          containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities: { drop: ["ALL"] }
        resources:
          requests: { cpu: "500m", memory: "1Gi" }
          limits: { cpu: "2000m", memory: "2Gi" }
        readinessProbe:
          httpGet: { path: /ready, port: http }
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /live, port: http }
          initialDelaySeconds: 15
          periodSeconds: 10
```

#### 2. Production StatefulSet (Headless Service, Partition Canary & PVC Retention)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  namespace: datastore
spec:
  clusterIP: None
  ports:
  - { port: 6379, name: redis }
  selector:
    app: redis-cluster
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: datastore
spec:
  serviceName: "redis-cluster"
  replicas: 3
  podManagementPolicy: OrderedReady
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2 # Canary: updates only redis-cluster-2
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: internal-registry.enterprise.io/cache/redis:7.2-alpine
        ports:
        - { containerPort: 6379, name: redis }
        volumeMounts:
        - { name: data, mountPath: /data }
        resources:
          requests: { cpu: "1000m", memory: "2Gi" }
          limits: { cpu: "2000m", memory: "2Gi" }
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ebs-gp3-sc"
      resources:
        requests: { storage: 50Gi }
```

#### 3. Production DaemonSet (Critical Node Agent)
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-collector
  namespace: kube-system
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10%
  selector:
    matchLabels:
      app: node-collector
  template:
    metadata:
      labels:
        app: node-collector
    spec:
      priorityClassName: system-node-critical # High priority; avoids eviction
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node.kubernetes.io/not-ready
        operator: Exists
        effect: NoExecute
      containers:
      - name: agent
        image: internal-registry.enterprise.io/ops/agent:v3.4
        securityContext:
          readOnlyRootFilesystem: true
          capabilities: { drop: ["ALL"] }
        volumeMounts:
        - { name: varlog, mountPath: /var/log, readOnly: true }
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits: { cpu: "500m", memory: "512Mi" }
      volumes:
      - name: varlog
        hostPath: { path: /var/log }
```

#### 4. Production Job (Finite Batch Execution)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-schema-upgrade-v4
  namespace: finance
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3 # Retries: 10s, 20s, 40s
  activeDeadlineSeconds: 900 # Hard 15-minute execution ceiling
  ttlSecondsAfterFinished: 86400 # Purge completed Job after 24h
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: flyway
        image: internal-registry.enterprise.io/db/flyway:10.8
        command: ["flyway", "migrate"]
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
          limits: { cpu: "1000m", memory: "1Gi" }
```

#### 5. Production CronJob (Guaranteed Scheduled Run)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: table-vacuum
  namespace: finance
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid       # Skip new run if prior run is still active
  startingDeadlineSeconds: 300   # Prevent >100 missed runs lockup after outage
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: vacuum
            image: internal-registry.enterprise.io/db/pg-client:16
            command: ["vacuumdb", "-a", "-z"]
            resources:
              requests: { cpu: "500m", memory: "512Mi" }
              limits: { cpu: "1000m", memory: "1Gi" }
```

---

## 4. Failure Modes, Bottlenecks & Troubleshooting

### A. Failure Mode 1: StatefulSet RWO Multi-Attach Deadlock
**Root Cause**: When a worker node experiences a network partition, the node controller marks it `NotReady`. After 5 minutes, pods on that node enter `Terminating`.
- Because the partitioned node cannot acknowledge unmounting, the volume remains locked.
- The `attachdetach-controller` will not detach the AWS EBS RWO volume from the partitioned EC2 instance to protect against split-brain data corruption.
- A replacement pod scheduled on another node hangs permanently in `ContainerCreating` with:
  `Multi-Attach error for volume ... Volume is already exclusively attached to one node`.

**SRE Diagnostics & Resolution**:
```bash
# 1. Identify locked VolumeAttachments
kubectl get volumeattachment | grep -E "Attached.*false|Attached.*true"

# 2. Inspect attachment error details
kubectl describe volumeattachment <attachment-id>

# 3. Filter controller mount/attach events
kubectl get events -n datastore --sort-by='.metadata.creationTimestamp' | grep -E "FailedAttach|FailedMount"

# 4. Safe remediation: Terminate partitioned EC2 instance FIRST to force cloud detach
aws ec2 terminate-instances --instance-ids <partitioned-node-instance-id>

# 5. Only after EC2 instance termination is confirmed out-of-band:
kubectl delete pod redis-cluster-0 -n datastore --force --grace-period=0
```

### B. Failure Mode 2: CronJob >100 Missed Schedule Stall
**Root Cause**: If the control plane or CronJob controller is unavailable for $>100$ schedule intervals without `startingDeadlineSeconds`, the controller permanently halts future executions.
**Diagnostics & Recovery**:
```bash
# 1. Detect missed schedule failure event
kubectl get events -A --field-selector reason=TooManyMissedTimes

# 2. Inspect CronJob status and last schedule timestamp
kubectl get cronjob table-vacuum -n finance -o jsonpath='{.status}'

# 3. Remediate by patching startingDeadlineSeconds
kubectl patch cronjob table-vacuum -n finance --type=merge -p '{"spec":{"startingDeadlineSeconds":600}}'
```

### C. Diagnostic Command Suite & Exit Code Triage
```bash
# Rollout lifecycle inspection
kubectl rollout status deployment/payment-api
kubectl rollout history deployment/payment-api
kubectl rollout undo deployment/payment-api --to-revision=2

# Inspect Job container termination exit codes
kubectl get pods -n finance -l job-name=db-schema-upgrade-v4 \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\tExitCode:"}{.status.containerStatuses[*].state.terminated.exitCode}{"\tReason:"}{.status.containerStatuses[*].state.terminated.reason}{"\n"}{end}'
```

| Exit Code | Reason | Architectural Root Cause | Action Item |
| :--- | :--- | :--- | :--- |
| **137** | `OOMKilled` | Process exceeded cgroup v2 `memory.max`. Killed via kernel `SIGKILL`. | Increase `resources.limits.memory` or profile application heap. |
| **143** | `Error` | Process received `SIGTERM` ($128 + 15$) due to `activeDeadlineSeconds` expiry. | Increase job timeout ceiling or optimize query execution plan. |
| **1** | `Error` | Application runtime exception or assertion failure. | Inspect application stdout via `kubectl logs <pod>`. |
| **127** | `ContainerCannotRun` | Specified command/binary not found in container rootfs `$PATH`. | Correct `command: [...]` entrypoint in manifest. |

---

## 5. Socratic Architectural Challenge

### Architectural Context
In an Amazon EKS cluster spanning 3 Availability Zones, a network partition isolates worker node `ip-10-0-12-45.ec2.internal` (Node A) from the control plane. Node A remains powered on; its local processes continue running, but its kubelet cannot communicate with `kube-apiserver`.

Node A hosts two workloads:
1. `payment-api-5968b6b8b5-x8k9m`: A pod belonging to a stateless **Deployment** (6 replicas).
2. `db-datastore-0`: A pod belonging to a **StatefulSet** (3 replicas), backed by an AWS EBS `gp3` RWO volume (`vol-0abc`) mounted to Node A.

After 5 minutes, the control plane applies `node.kubernetes.io/unreachable:NoExecute` to Node A.

```
                  +------------------------------------------+
                  |         EKS Control Plane                |
                  +------------------------------------------+
                             |                    |
             Network Partition! (X)               | Healthy Link
                             |                    v
             +---------------+----------+   +-----+--------------------+
             | Node A (us-east-1a)      |   | Node B (us-east-1b)      |
             | Status: NotReady         |   | Status: Ready            |
             |                          |   |                          |
             | Pod: payment-api-xxx     |   |                          |
             | Pod: db-datastore-0      |   |                          |
             |   [EBS vol-0abc mounted] |   |                          |
             +--------------------------+   +--------------------------+
```

### The Architectural Problem
1. **Controller Divergence Under Node Eviction**:
   Trace the distinct control loops of `deployment-controller` and `statefulset-controller`. Why does the Deployment controller immediately spin up a replacement pod on Node B while the old pod is still terminating on Node A, whereas the StatefulSet controller strictly refuses to create `db-datastore-0` on Node B, leaving the database degraded?
2. **Filesystem Corruption & Split-Brain Mechanics**:
   Suppose the `statefulset-controller` behaved identically to the Deployment controller and instantiated `db-datastore-0` on Node B while Node A remained partitioned. What would happen at the block storage (EBS / NVMe) and filesystem layer (`ext4`/`XFS`) if Node A's network recovered while both instances wrote to `vol-0abc`?
3. **Staff SRE Triage Protocol**:
   A junior engineer suggests running `kubectl delete pod db-datastore-0 --force --grace-period=0`. Why is this dangerous? What exact AWS infrastructure assertion must be validated before executing this command?
